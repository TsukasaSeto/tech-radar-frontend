# ビルドキャッシュのベストプラクティス

依存インストール・型チェック・bundling のすべてに増分キャッシュを効かせて、CI ビルドを 5-10 倍速くする。

## ルール

### 1. Turborepo のリモートキャッシュを CI で共有する

Turborepo を採用し、リモートキャッシュ（Vercel Remote Cache / 自前 S3）を CI 間で共有する。
ローカルで build した成果物がそのまま CI で使われる、または別 PR の build が再利用される。

**根拠**:
- 同じ input（ソース + 依存）から生成される output は一意なので、再計算する必要がない
- リモートキャッシュ共有により、CI run 全体が 30 秒程度に短縮されるケースもある
- モノレポでは「変更があったパッケージのみ build / test」を可能にする
- Turborepo は Next.js / Remix / Vite で同等に使える（特定ツール依存ではない）
- ビルド結果を変える環境変数は `env`（タスク単位）/ `globalEnv`（全タスク）に列挙する。列挙した変数の値だけがハッシュに入る。既定の Strict Mode では未宣言の変数はタスクに渡らず、Loose Mode では渡るがハッシュに入らないため、環境違いの成果物をリモートキャッシュから誤って復元しうる。Next.js の `NEXT_PUBLIC_*` は framework inference で自動的に含まれる
- Turborepo は `.env` ファイルをタスクに読み込まない。`.env*` は `inputs` に入れてハッシュ対象にし、CI が注入する変数は `env` で扱う

**`turbo.json` 設定**:
```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": [".env"],
  // 全タスクのハッシュに入れる変数
  "globalEnv": ["NODE_ENV"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "!.next/cache/**", "dist/**"],
      "inputs": ["$TURBO_DEFAULT$", ".env*"],
      // Bad: env を書かない（API_URL が違う環境の成果物がキャッシュから復元される）
      // Good: ビルド出力に影響する変数を列挙する（NEXT_PUBLIC_* は自動）
      "env": ["API_URL", "SENTRY_*"]
    },
    "typecheck": {
      "dependsOn": ["^typecheck"],
      "outputs": [".tsbuildinfo"]
    },
    "lint": {
      "outputs": [".eslintcache"]
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    }
  }
}
```

**Vercel Remote Cache（推奨デフォルト）**:
```yaml
# .github/workflows/ci.yml
env:
  TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}      # vercel.com から取得
  TURBO_TEAM: ${{ secrets.TURBO_TEAM }}        # チーム名

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with: { persist-credentials: false }
      - uses: actions/setup-node@v7
        with: { node-version: 24 }
      # 実運用ではサードパーティ action を SHA でピン留めする（`practices/ci-cd/github-actions.md` #9）
      - uses: pnpm/setup@v3
        with: { node-version-file: false, cache: true, install: false }
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo run build typecheck lint test
        # ↑ TURBO_TOKEN / TURBO_TEAM があれば自動でリモートキャッシュ使用
```

**self-hosted リモートキャッシュ**:
```jsonc
// turbo.json に remoteCache を設定
{
  "remoteCache": {
    "signature": true,        // キャッシュ改ざんを検知
    "preflight": true         // 大きな成果物の PUT 前に check
  }
}
```

```bash
# 環境変数でリモートキャッシュサーバー（S3 互換ストレージ等）を指定
TURBO_API=https://my-turbo-server.example.com
```

OSS のリモートキャッシュサーバーには [`ducktors/turborepo-remote-cache`](https://github.com/ducktors/turborepo-remote-cache) があり、S3 / GCS / Azure Blob と接続できる。

**input / output の正確な指定**:
```json
{
  "tasks": {
    "build": {
      "inputs": [
        "$TURBO_DEFAULT$",        // src/**, package.json 等のデフォルト
        ".env.production",         // 環境変数も input に
        "!**/*.test.{ts,tsx}"     // テストファイルは除外
      ],
      "outputs": [
        ".next/**",
        "!.next/cache/**"         // Next.js キャッシュは別管理（actions/cache）
      ]
    }
  }
}
```

**Turbo の output が正しいことを検証**:
```bash
# dry-run でどの input から output が生成されるかを表示
pnpm turbo run build --dry-run=json | jq '.tasks[0]'
```

**キャッシュヒット率の確認**:
```bash
pnpm turbo run build --summarize
# >>> Tasks:    5 successful, 5 total
# >>> Cached:    4 cached, 5 total
# >>> Time:    2.1s >>> FULL TURBO
```

**出典**:
- [Turborepo Docs](https://turborepo.com/docs) (Vercel)
- [Turborepo: Remote Caching](https://turborepo.com/docs/core-concepts/remote-caching) (Vercel)
- [ducktors/turborepo-remote-cache](https://github.com/ducktors/turborepo-remote-cache)
- [Turborepo: Using environment variables](https://turborepo.dev/docs/crafting-your-repository/using-environment-variables) (Turborepo 公式、`env` / `globalEnv` とハッシュ・Strict / Loose Mode・framework inference・`.env` を読み込まないこと) ※2026-10-25 確認

**バージョン**: Turborepo 2+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 2. Next.js の `.next/cache` を `actions/cache` で永続化する

Next.js は `.next/cache` 配下に webpack / SWC / Babel / TypeScript / Image Optimization のキャッシュを保存する。
これを CI run 間で永続化することで、増分ビルドが効く。

**根拠**:
- Next.js のフルビルドは 1-3 分、増分ビルドは 10-30 秒程度
- `.next/cache` には大量の中間ファイルが入る（30-300MB 程度）
- Turborepo を使っていても、Next.js 内部のキャッシュは別管理。両方併用が最も速い
- 画像最適化のキャッシュ（`.next/cache/images`）は `next start` の実行中にリクエストに応じて作られ、`next build` では作られない。CI のキャッシュ対象にしても効果はない

**設定例**:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with: { persist-credentials: false }

      - uses: actions/setup-node@v7
        with: { node-version: 24 }
      # 実運用ではサードパーティ action を SHA でピン留めする（`practices/ci-cd/github-actions.md` #9）
      - uses: pnpm/setup@v3
        with: { node-version-file: false, cache: true, install: false }

      # Next.js cache を復元
      # path / restore-keys は | のブロックスカラーなので、行末に # を書くとコメントではなくパスやキーの一部になる
      # （モノレポでは apps/*/.next/cache も保存する）
      - uses: actions/cache@v6
        with:
          path: |
            ${{ github.workspace }}/.next/cache
            ${{ github.workspace }}/apps/*/.next/cache
          # ソース変更で部分更新できるよう、複合キーを使う
          key: ${{ runner.os }}-nextjs-${{ hashFiles('**/pnpm-lock.yaml') }}-${{ hashFiles('**/*.[jt]s', '**/*.[jt]sx', '**/*.css') }}
          restore-keys: |
            ${{ runner.os }}-nextjs-${{ hashFiles('**/pnpm-lock.yaml') }}-
            ${{ runner.os }}-nextjs-

      - run: pnpm install --frozen-lockfile
      - run: pnpm build
```

**キャッシュ効果の検証**:
```bash
# ローカルでキャッシュ削除前後で時間比較
rm -rf .next
time pnpm build  # 1m 30s
time pnpm build  # 25s（キャッシュあり）
```

**モノレポでの `.next/cache` 配置**:
```
apps/
├── web/
│   └── .next/cache    ← キャッシュ
├── admin/
│   └── .next/cache    ← 別キャッシュ（独立）
```

`actions/cache` の `path:` に複数を指定して両方保存。

**Vercel デプロイの場合**:
- Vercel 自体が内部で `.next/cache` を保持する（明示的に何もしなくて良い）
- ただし dev / CI のローカル build 速度向上には `actions/cache` が必要

**キャッシュ無効化のタイミング**:
- `next.config.ts` 変更 → 自動的に全 cache 無効
- Next.js のメジャー更新 → 自動で破棄
- 環境変数変更 → `inputs` に環境変数を含めれば検知

**Turbopack のビルド用ファイルシステム永続キャッシュ（Next.js 16.3+）**:
Webpack ビルドの `.next/cache` 永続化（このルール本文）と同じ発想を、Turbopack ビルドにも `experimental.turbopackFileSystemCacheForBuild` で適用できる。CI では生成された `.next` ディレクトリを次回実行へコピーすることでビルド時間を短縮する。

```ts
// next.config.ts
const nextConfig = {
  experimental: {
    turbopackFileSystemCacheForBuild: true,
  },
};
```

> "Persistent cache for builds can be enabled with the `turbopackFileSystemCacheForBuild` config flag... CI setups can take advantage of this by copying the generated `.next` directory from one run to the next."
> ([Turbopack: What's New in Next.js 16.3](https://nextjs.org/blog/next-16-3-turbopack), セクション "Persistent caching") ※2026-06-30に実際にfetch成功

**出典**:
- [Next.js Docs: CI build caching](https://nextjs.org/docs/app/building-your-application/deploying/ci-build-caching) (Next.js 公式)
- [GitHub: actions/cache](https://github.com/actions/cache)
- [Next.js Docs: Image Component](https://nextjs.org/docs/app/api-reference/components/image) (Next.js 公式、最適化済み画像は初回のリクエスト時に作られ `<distDir>/cache/images` に保存される) ※2026-10-25 確認
- [Turbopack: What's New in Next.js 16.3](https://nextjs.org/blog/next-16-3-turbopack) (Next.js 公式ブログ、`turbopackFileSystemCacheForBuild` による永続ビルドキャッシュ) ※2026-06-30に実際にfetch成功

**バージョン**: Next.js 13+（Turbopack 永続キャッシュは Next.js 16.3+）/ actions/cache v6
**確信度**: 高
**最終更新**: 2026-10-25

---

### 3. キャッシュキーは lockfile ハッシュ + ソースハッシュで構成する

キャッシュキーは「いつ invalidate すべきか」の判断に直結する。
依存ハッシュ（lockfile）とソースハッシュ（src/）の両方を組み合わせて、過剰 invalidate / 過小 invalidate を回避する。

**根拠**:
- `actions/cache` は「復元」と「保存」が別に動く。ジョブ開始時に `key` の完全一致（なければ `restore-keys` の前方一致）を復元し、ジョブ終了時の保存は `key` が完全一致しなかったときだけ行う。同じキーへの上書き保存はできない
- キーが粗すぎると（lockfile だけ）、ソースを変えても古いキャッシュが完全一致で復元され、保存は行われない。更新後の中身が保存されないまま、lockfile が変わるまで古いキャッシュを使い続ける
- キーが細かすぎると（全ファイル hash）、毎回 cache miss してキャッシュの意味がない
- `restore-keys` で複数の fallback を指定し、最も近い古いキャッシュから差分ビルド
- `hashFiles` のパターンが使えるのは `*` / `?` / `[...]` / `**` と、除外の `!` だけで、`{ts,tsx}` のようなブレース展開は使えない。どのファイルにも一致しないと空文字になり、キーのその部分が固定される。拡張子ごとに `hashFiles('**/*.ts', '**/*.tsx')` のように分けて渡す
- 異なる用途（lint / test / build）にはそれぞれ別キャッシュを切る — GitHub Actions のキャッシュは同じキーへ上書き保存できないため、複数ジョブが同じキーを共有すると先に保存したジョブのキャッシュだけが残り続け、後続ジョブのビルド結果は永遠に保存されないまま毎回フルビルドになる（実例: lint と test が同じキーを共有し、test 側のコンパイル結果が一度も永続化されていなかったケース）

**設計パターン**:
```yaml
# パターン 1: 依存キャッシュ（最も粗い、変更頻度低）
- uses: actions/cache@v6
  with:
    path: ~/.local/share/pnpm/store
    key: ${{ runner.os }}-pnpm-${{ hashFiles('pnpm-lock.yaml') }}

# パターン 2: ビルドキャッシュ（中程度、ソース変更で更新）
# Bad: hashFiles('src/**/*.{ts,tsx,css}')（ブレース展開は使えず、一致なしの空文字になってキーが固定される）
# Good: パターンを拡張子ごとに分けて渡す
- uses: actions/cache@v6
  with:
    path: .next/cache
    key: ${{ runner.os }}-nextjs-${{ hashFiles('pnpm-lock.yaml') }}-${{ hashFiles('src/**/*.ts', 'src/**/*.tsx', 'src/**/*.css') }}
    restore-keys: |
      ${{ runner.os }}-nextjs-${{ hashFiles('pnpm-lock.yaml') }}-

# パターン 3: テストキャッシュ（細かい、各 PR ごと）
- uses: actions/cache@v6
  with:
    path: .vitest/cache
    key: ${{ runner.os }}-vitest-${{ github.sha }}
    restore-keys: |
      ${{ runner.os }}-vitest-${{ github.event.pull_request.base.sha }}-
      ${{ runner.os }}-vitest-
```

**`restore-keys` の動作**:
1. 完全一致のキャッシュを探す
2. 見つからなければ `restore-keys` を順に試す（前方一致）
3. それでも無ければ cache miss として新規ビルド

```yaml
key: app-v1-${{ hashFiles('lockfile') }}-${{ hashFiles('src/*') }}
# restore-keys: | の中に書いた # はコメントにならずキーの一部になるので、説明はブロックの外に書く
# 1 行目: lockfile 同じ・ソース違いでも復元 / 2 行目: lockfile 違いでもいいから何か復元
restore-keys: |
  app-v1-${{ hashFiles('lockfile') }}-
  app-v1-
```

**意図的に invalidate するテクニック**:
```yaml
# version 番号をキーに含めて手動制御
key: app-v2-${{ hashFiles('pnpm-lock.yaml') }}
#         ↑ v1 → v2 にするだけで全キャッシュを破棄できる
```

**OS / アーキテクチャの差を考慮**:
```yaml
# Linux / macOS / Windows でバイナリが異なるパッケージがある（sharp / esbuild 等）
key: ${{ runner.os }}-${{ runner.arch }}-build-${{ hashFiles('lockfile') }}
```

**判定マトリクス**:

| 用途 | key の構成 | restore-keys | 理由 |
|---|---|---|---|
| 依存（pnpm store） | os + lockfile hash | os | store はパッケージ単位で保存されるので、古い store を復元しても install が不足分だけ取得する（部分復元が効く） |
| 依存（node_modules） | os + Node version + lockfile hash | なし（完全一致のみ） | 古い node_modules は lockfile と食い違った状態から始まるため、復元するなら完全一致に限る |
| Next.js / Webpack cache | os + lockfile + source hash | os + lockfile, os | 部分復元で増分ビルド |
| TypeScript .tsbuildinfo | os + ts version + source hash | os + ts version | TypeScript バージョンで invalidate |
| Test snapshot | branch | base branch | base からの差分のみ確認 |

**チェック方法**:
```bash
# GitHub UI: Actions > Caches で現在のキャッシュ一覧と最終アクセス時刻を確認
# CLI:
gh cache list -L 50
gh cache delete <cache-id>  # 不要なキャッシュを削除
```

**出典引用**:
> "GitHub Actionsのキャッシュは同じキーへ上書き保存できないため、`lint`ジョブが先に保存すると、`test`ジョブのコンパイル結果はいつまでもキャッシュに保存されません"
> ([GitHub ActionsのGoテストを8分→3分弱に短縮 —— setup-goのキャッシュ共有を見直す](https://zenn.dev/remitaid/articles/fa94cb567f5d1d), セクション "なぜ毎回フルコンパイルになっていたのか") ※2026-08-21に実際にfetch成功

**出典**:
- [GitHub Actions: Cache best practices](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#matching-a-cache-key) (GitHub Docs)
- [hashFiles function](https://docs.github.com/en/actions/learn-github-actions/expressions#hashfiles) (GitHub Docs)
- [GitHub ActionsのGoテストを8分→3分弱に短縮 —— setup-goのキャッシュ共有を見直す](https://zenn.dev/remitaid/articles/fa94cb567f5d1d) (Zenn、lint/testジョブ間のキャッシュキー衝突の実例) ※2026-08-21 fetch
- [GitHub Docs: Evaluate expressions（hashFiles）](https://docs.github.com/en/actions/reference/workflows-and-actions/expressions) / [@actions/glob README](https://github.com/actions/toolkit/blob/main/packages/glob/README.md) (GitHub 公式、対応するパターンは `*` / `?` / `[...]` / `**`、一致なしは空文字) ※2026-10-25 確認

**バージョン**: actions/cache v6
**確信度**: 高
**最終更新**: 2026-10-25

---

### 4. インクリメンタル TypeScript / ESLint で fast feedback

TypeScript の `incremental: true` と ESLint の `--cache` を有効にし、変更ファイルのみ処理する。
CI でも `.tsbuildinfo` と `.eslintcache` を保存して、run 間で再利用する。

**根拠**:
- TypeScript の `tsc --noEmit` も大規模プロジェクトでは 30-60 秒かかる
- インクリメンタル ビルドで 5-10 秒に短縮
- ESLint は `--cache` で 2-3 倍高速化。ただし既定の `--cache-strategy metadata` はファイルの更新時刻で変更を判定し、git は更新時刻を保存しないため、checkout のたびに全ファイルが変更扱いになる。CI では `--cache-strategy content`（内容で判定）を指定する
- Prettier も `--cache` を持つ（既定の判定方式は `content`）
- developer experience としても「すべての PR で 5 分待つ」より「30 秒で結果」のほうが圧倒的に良い

**TypeScript インクリメンタル**:
```json
// tsconfig.json
{
  "compilerOptions": {
    "incremental": true,
    "tsBuildInfoFile": "./.tsbuildinfo",
    "composite": false  // モノレポなら true
  }
}
```

```yaml
# CI（path: | の中に # を書くとパスの一部になるので、説明はブロックの外に書く。2 行目はモノレポ用）
- uses: actions/cache@v6
  with:
    path: |
      .tsbuildinfo
      **/.tsbuildinfo
    key: ${{ runner.os }}-tsc-${{ hashFiles('**/pnpm-lock.yaml') }}-${{ hashFiles('**/*.ts', '**/*.tsx') }}
    restore-keys: |
      ${{ runner.os }}-tsc-${{ hashFiles('**/pnpm-lock.yaml') }}-

- run: pnpm typecheck  # tsc -b でインクリメンタル動作
```

**ESLint キャッシュ**:
```json
// package.json
{
  "scripts": {
    "lint": "eslint . --cache --cache-strategy content --cache-location ./node_modules/.cache/eslint/.eslintcache"
  }
}
```

```yaml
- uses: actions/cache@v6
  with:
    path: node_modules/.cache/eslint
    key: ${{ runner.os }}-eslint-${{ hashFiles('.eslintrc.*', 'eslint.config.*') }}-${{ hashFiles('**/*.ts', '**/*.tsx') }}
    restore-keys: |
      ${{ runner.os }}-eslint-${{ hashFiles('.eslintrc.*', 'eslint.config.*') }}-

- run: pnpm lint
```

**Prettier の高速化（`--cache` か、--check の対象限定）**:
```bash
# 全ファイル prettier (遅い)
prettier --check .

# キャッシュを使う（既定の保存先は node_modules/.cache/prettier/.prettier-cache。CI ではこのディレクトリを actions/cache で保存する）
prettier --check . --cache

# 変更ファイルのみ
git diff --name-only --diff-filter=ACM HEAD~1 | xargs prettier --check
```

**Vitest のキャッシュ**（失敗したテストと時間のかかるテストを先に実行するための結果キャッシュ）:
```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  // Bad: test.cache.dir（非推奨で、現行の Vitest では効果がない）
  // Good: 保存先は Vite の cacheDir で指定する
  cacheDir: './node_modules/.cache/vitest',
});
```

```yaml
- uses: actions/cache@v6
  with:
    path: node_modules/.cache/vitest
    key: ${{ runner.os }}-vitest-${{ hashFiles('**/*.test.ts', '**/*.test.tsx') }}
```

**増分実行（lint-staged）**:
```json
// package.json
{
  "scripts": {
    "lint:staged": "lint-staged"
  },
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix --cache", "prettier --write"]
  }
}
```

pre-commit hook で変更ファイルのみチェック。CI でも push 時に「base ブランチからの差分ファイルだけ」を対象にする最適化が可能。

**判断軸**:
| ツール | キャッシュ機構 | CI で有効化推奨 |
|---|---|---|
| TypeScript | `incremental: true` + `.tsbuildinfo` | ✅ |
| ESLint | `--cache --cache-strategy content` + `.eslintcache` | ✅ |
| Vitest | `cacheDir`（Vite の設定） | ✅ |
| Jest | `--cache` (デフォルト有効) | ✅ |
| Prettier | `--cache`（または対象を絞る） | ✅ |
| Stylelint | `--cache` + `.stylelintcache` | ✅ |

**出典**:
- [TypeScript: --incremental](https://www.typescriptlang.org/tsconfig#incremental) (TypeScript 公式)
- [ESLint: --cache](https://eslint.org/docs/latest/use/command-line-interface#--cache) (ESLint 公式、`--cache-strategy` の既定は `metadata`。git は更新時刻を保存しないため `content` が有効) ※2026-10-25 確認
- [Vitest: cache](https://vitest.dev/config/cache) (Vitest 公式、`cache.dir` は効果がなく `cacheDir` を使う) ※2026-10-25 確認
- [Prettier: CLI](https://prettier.io/docs/cli) (Prettier 公式、`--cache` / `--cache-location` / `--cache-strategy`) ※2026-10-25 確認

**バージョン**: TypeScript 5+, ESLint 8+, Vitest（`cacheDir`）, Prettier 3+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 5. TypeScript 7.0 の `tsc` を CI の型チェックに先行導入し、並列度は実測で決める

TypeScript 7.0（2026-07-08 リリース、Go へのネイティブ移植）は従来の `tsc` より型チェックが速い。CI の型チェックで先に 7.0 の `tsc` を使い、Compiler API に依存するツール（typescript-eslint・ts-jest・ts-node 等）は API が入る 7.1 まで 6.0 に残すことで、リスクを抑えて fast feedback を得る。
削除されたオプション・既定値の変更・6.0 と 7.0 を併存させる npm エイリアスの書き方は `practices/typescript/strict-mode.md` #9 を正本とし、ここでは CI での並列度（`--checkers` / `--builders`）だけを扱う。

**根拠**:
- 速度の公称値と実測値は情報源で幅がある（`practices/typescript/strict-mode.md` #9）。CI で効果を確認するには、自プロジェクトの型チェック時間を移行前後で測る
- 並列度は OS・CPU 環境に依存する。引用記事の測定環境では、Linux の CI は `--builders 4 --checkers 2` が最速、Windows は `--builders 4` のみが最適（`--checkers` を足すとかえって遅くなった）。他の環境ではそのまま当てはまるとは限らない
- `--checkers` の既定は 4（型チェックのワーカー数）。`--builders` は Project References のビルド並列度を制御する
- 同じ記事では `--builders 8` が undeclared build dependency エラーを起こしたと報告されている。並列度は実測値なしで上げない
- `typescript` を 6.0 の互換パッケージ（`@typescript/typescript6`）にエイリアスすると、`require('typescript')` は 6.0 の API を返し、`tsc` コマンドは別名で入れた 7.0 になる。逆向き（`typescript` を 7.0 のまま、6.0 を別名で入れる）だと、ツールが `require('typescript')` で 7.0 を読み込み、切り替わらない

**移行シーケンス**:
```bash
# Step 1: TypeScript 6.0 で非推奨オプションを解消する（strict-mode.md #9）
npx tsc --noEmit  # エラーゼロを確認

# Step 2: package.json を TypeScript 公式の併存構成にする（バージョンは固定し、Renovate 等で上げる）
#   "typescript": "npm:@typescript/typescript6@^6.0.2"   ← require('typescript') は 6.0（tsc6 コマンドも入る）
#   "@typescript/native": "npm:typescript@^7.0.2"        ← tsc コマンドは 7.0
npm install --save-dev typescript@npm:@typescript/typescript6@^6.0.2 @typescript/native@npm:typescript@^7.0.2

# Step 3: 型チェックだけ 7.0 の tsc で実行する
npx tsc --noEmit
```

```yaml
# GitHub Actions — OS 別パラメータの使い分け（自環境の実測値なしで変更しない）
jobs:
  typecheck:
    runs-on: ubuntu-latest
    steps:
      # checkout と setup は `practices/ci-cd/github-actions.md` #1 の Composite Action
      - run: npx tsc --noEmit --builders 4 --checkers 2
      # 引用記事の Windows runner の測定では --builders 4 のみが最適（--checkers を足すと遅くなった）
```

**アンチパターン**:
- CSS `@import` などの side-effect import をそのまま移行する（`src/types/css.d.ts` で型宣言を先に用意する）
- `.js + JSDoc` ファイルを速度最適化と同時に扱う（別フェーズで対応）
- プログラマティック API ツールも同時に v7 へ移行しようとする（API が入る 7.1 まで待つ）

**出典引用**:
> "並列度は環境依存です"
> ([TypeScript 7.0 RCは速いが、見るべきはtsc移行とCIの並列度だった](https://zenn.dev/53able/articles/fa3d26db6f9fba), セクション "CI 別パラメータ調査") ※2026-06-22に実際にfetch成功

> "約21倍の高速化を実測しました"
> ([TypeScript 7.0を実プロジェクトで検証：型チェックが高速化](https://zenn.dev/nenene01/articles/typescript-7-benchmark-musclelog), セクション "ベンチマーク結果") ※2026-06-22に実際にfetch成功

**出典**:
- [Announcing TypeScript 7.0](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/) (TypeScript 公式ブログ、2026-07-08 リリース・7.0 は API を含まず 7.1 で追加予定・`@typescript/typescript6` と `tsc6`・併存用の npm エイリアス・`--checkers` の既定 4) ※2026-10-25 確認

**バージョン**: TypeScript 7.0+（Compiler API は 7.1 以降。併存用の 6.0 は `@typescript/typescript6`）
**確信度**: 中（併存構成と `--checkers` の既定は公式。OS 別の並列度の最適値は 1 記事の測定）
**最終更新**: 2026-10-25
