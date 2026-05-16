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

**`turbo.json` 設定**:
```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": [".env"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "!.next/cache/**", "dist/**"],
      "inputs": ["$TURBO_DEFAULT$", ".env*"]
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
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo run build typecheck lint test
        # ↑ TURBO_TOKEN / TURBO_TEAM があれば自動でリモートキャッシュ使用
```

**self-hosted リモートキャッシュ**:
```yaml
# turbo.json に remoteCache を設定
{
  "remoteCache": {
    "signature": true,        // キャッシュ改ざんを検知
    "preflight": true         // 大きな成果物の PUT 前に check
  }
}

# 環境変数で S3 互換ストレージを指定
# TURBO_API: https://my-turbo-server.example.com
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

**バージョン**: Turborepo 2+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 2. Next.js の `.next/cache` を `actions/cache` で永続化する

Next.js は `.next/cache` 配下に webpack / SWC / Babel / TypeScript / Image Optimization のキャッシュを保存する。
これを CI run 間で永続化することで、増分ビルドが効く。

**根拠**:
- Next.js のフルビルドは 1-3 分、増分ビルドは 10-30 秒程度
- `.next/cache` には大量の中間ファイルが入る（30-300MB 程度）
- Turborepo を使っていても、Next.js 内部のキャッシュは別管理。両方併用が最も速い
- イメージ最適化キャッシュ（`.next/cache/images`）も恩恵が大きい

**設定例**:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: pnpm }

      # Next.js cache を復元
      - uses: actions/cache@v4
        with:
          path: |
            ${{ github.workspace }}/.next/cache
            ${{ github.workspace }}/apps/*/.next/cache  # モノレポ対応
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

**Image Optimization キャッシュ**:
```yaml
# `next/image` が生成した最適化済み画像も別途キャッシュ
- uses: actions/cache@v4
  with:
    path: ${{ github.workspace }}/.next/cache/images
    key: ${{ runner.os }}-nextjs-images-${{ hashFiles('public/**/*.{jpg,png,webp,avif}') }}
```

**出典**:
- [Next.js Docs: CI build caching](https://nextjs.org/docs/app/building-your-application/deploying/ci-build-caching) (Next.js 公式)
- [GitHub: actions/cache](https://github.com/actions/cache)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 3. キャッシュキーは lockfile ハッシュ + ソースハッシュで構成する

キャッシュキーは「いつ invalidate すべきか」の判断に直結する。
依存ハッシュ（lockfile）とソースハッシュ（src/）の両方を組み合わせて、過剰 invalidate / 過小 invalidate を回避する。

**根拠**:
- キーが粗すぎると（lockfile だけ）、ソース変更でキャッシュが再利用されない
- キーが細かすぎると（全ファイル hash）、毎回 cache miss してキャッシュの意味がない
- `restore-keys` で複数の fallback を指定し、最も近い古いキャッシュから差分ビルド
- 異なる用途（lint / test / build）にはそれぞれ別キャッシュを切る

**設計パターン**:
```yaml
# パターン 1: 依存キャッシュ（最も粗い、変更頻度低）
- uses: actions/cache@v4
  with:
    path: ~/.local/share/pnpm/store
    key: ${{ runner.os }}-pnpm-${{ hashFiles('pnpm-lock.yaml') }}

# パターン 2: ビルドキャッシュ（中程度、ソース変更で更新）
- uses: actions/cache@v4
  with:
    path: .next/cache
    key: ${{ runner.os }}-nextjs-${{ hashFiles('pnpm-lock.yaml') }}-${{ hashFiles('src/**/*.{ts,tsx,css}') }}
    restore-keys: |
      ${{ runner.os }}-nextjs-${{ hashFiles('pnpm-lock.yaml') }}-

# パターン 3: テストキャッシュ（細かい、各 PR ごと）
- uses: actions/cache@v4
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
restore-keys: |
  app-v1-${{ hashFiles('lockfile') }}-    # ← lockfile 同じ・ソース違いでも復元
  app-v1-                                  # ← lockfile 違いでもいいから何か復元
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
| 依存（node_modules / pnpm store） | os + lockfile hash | os | 部分復元不可、完全一致が必要 |
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

**出典**:
- [GitHub Actions: Cache best practices](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#matching-a-cache-key) (GitHub Docs)
- [hashFiles function](https://docs.github.com/en/actions/learn-github-actions/expressions#hashfiles) (GitHub Docs)

**バージョン**: actions/cache v4
**確信度**: 高
**最終更新**: 2026-05-16

---

### 4. インクリメンタル TypeScript / ESLint で fast feedback

TypeScript の `incremental: true` と ESLint の `--cache` を有効にし、変更ファイルのみ処理する。
CI でも `.tsbuildinfo` と `.eslintcache` を保存して、run 間で再利用する。

**根拠**:
- TypeScript の `tsc --noEmit` も大規模プロジェクトでは 30-60 秒かかる
- インクリメンタル ビルドで 5-10 秒に短縮
- ESLint は `--cache` で 2-3 倍高速化
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
# CI
- uses: actions/cache@v4
  with:
    path: |
      .tsbuildinfo
      **/.tsbuildinfo  # モノレポ
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
    "lint": "eslint . --cache --cache-location ./node_modules/.cache/eslint/.eslintcache"
  }
}
```

```yaml
- uses: actions/cache@v4
  with:
    path: node_modules/.cache/eslint
    key: ${{ runner.os }}-eslint-${{ hashFiles('.eslintrc.*', 'eslint.config.*') }}-${{ hashFiles('**/*.{ts,tsx}') }}
    restore-keys: |
      ${{ runner.os }}-eslint-${{ hashFiles('.eslintrc.*', 'eslint.config.*') }}-

- run: pnpm lint
```

**Prettier の高速化（--check で対象限定）**:
```bash
# 全ファイル prettier (遅い)
prettier --check .

# 変更ファイルのみ
git diff --name-only --diff-filter=ACM HEAD~1 | xargs prettier --check
```

**Vitest のキャッシュ**:
```ts
// vitest.config.ts
export default defineConfig({
  test: {
    cache: { dir: './node_modules/.cache/vitest' },
  },
});
```

```yaml
- uses: actions/cache@v4
  with:
    path: node_modules/.cache/vitest
    key: ${{ runner.os }}-vitest-${{ hashFiles('**/*.test.{ts,tsx}') }}
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
| ESLint | `--cache` + `.eslintcache` | ✅ |
| Vitest | `cache.dir` | ✅ |
| Jest | `--cache` (デフォルト有効) | ✅ |
| Prettier | キャッシュなし、対象を絞る | △（lint-staged） |
| Stylelint | `--cache` + `.stylelintcache` | ✅ |

**出典**:
- [TypeScript: --incremental](https://www.typescriptlang.org/tsconfig#incremental) (TypeScript 公式)
- [ESLint: --cache](https://eslint.org/docs/latest/use/command-line-interface#--cache) (ESLint 公式)
- [Vitest: Caching](https://vitest.dev/config/#cache) (Vitest 公式)

**バージョン**: TypeScript 5+, ESLint 8+, Vitest 1+
**確信度**: 高
**最終更新**: 2026-05-16
