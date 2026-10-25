# 依存管理のベストプラクティス

> **役割分担**: このファイルは **環境再現性 / 開発体験**（lockfile 戦略、packageManager の固定、未使用検出、パッチ管理、range の決め方と major 更新の進め方）を扱う。lockfile / `packageManager` / Corepack は #1 が正本。
> **脆弱性スキャン / supply chain attack 対策 / 自動更新 PR（Renovate / Dependabot）の設定と自動マージの範囲**は [`security/dependency-security.md`](../security/dependency-security.md) を参照（更新ポリシーの正本は同 #2）。
> lockfile 周りは両ファイルで触れるが、**「正しく固定する」は dx 側、「壊れた依存を検知・封じ込める」は security 側** という分担を基本にする。

`package.json` の `dependencies` は本番に乗る代物。雑な追加・放置がそのまま負債とセキュリティリスクになる。

## ルール

### 1. lockfile を必ずコミット、`packageManager` で CLI バージョンを固定する

`pnpm-lock.yaml` 等の lockfile はリポジトリに含め、`package.json` の `packageManager` フィールドでパッケージマネージャ自体のバージョンも固定する。
CI / 本番 / ローカルで完全に同じ依存ツリーが構築されることを保証する。

**根拠**:
- 「ローカルで動くが CI で壊れる」「先週は動いたが今日は動かない」の主因は依存バージョンの非決定性
- lockfile は依存パッケージ・transitive deps・パッチを完全固定する
- `packageManager` フィールドで pnpm 自体のバージョンも固定できる。pnpm 11 以降は、宣言された版を pnpm 自身が取得して実行する（`pmOnFail: download` が既定）ので、Corepack は必須ではない
- Corepack は Node.js 16.9.0（14 系は 14.19.0）から 24.x まで同梱されているが、Node.js 25 以降は同梱されない。使う場合は `npm install -g corepack` で別途入れる
- 同じ commit から make したビルドが常に同じ結果になる reproducibility が CI / 監査の前提
- `package.json` だけ編集して `pnpm install` を実行し忘れると、lockfile と不整合を起こし `--frozen-lockfile` の CI が全ジョブ同時に fail する実例が報告されている（1パッケージの削除忘れで 7 ジョブ同時 fail）
- lockfile がもたらすセキュリティ上の効果（dependency confusion への耐性など）は [`security/dependency-security.md`](../security/dependency-security.md) #3 を参照

**`package.json`**:
```json
{
  "name": "myapp",
  "packageManager": "pnpm@11.28.8",
  "engines": {
    "node": ">=24.0.0",
    "pnpm": ">=11.0.0"
  },
  "scripts": {
    "preinstall": "npx only-allow pnpm"
  }
}
```

**pnpm の版の切り替え**:
```bash
# pnpm 11 以降: packageManager の版を pnpm 自身が取得して実行する
pnpm install  # → 11.28.8 で実行される

# Corepack を使う場合（Node.js 25 以降は同梱されないので先に入れる）
npm install -g corepack
corepack enable
```

**コマンド対応表**:
| ツール | ローカル install | CI install |
|---|---|---|
| npm | `npm install` | `npm ci` |
| pnpm | `pnpm install` | `pnpm install --frozen-lockfile` |
| yarn | `yarn install` | `yarn install --immutable`（Berry） |
| bun | `bun install` | `bun install --frozen-lockfile` |

pnpm は CI 環境を検出すると、lockfile がある場合は `--frozen-lockfile` を既定で有効にするが、意図を明示するために書いておく。

**CI で完全一致を強制**:
```yaml
# .github/workflows/ci.yml
- uses: actions/checkout@v7
- uses: actions/setup-node@v7
  with: { node-version-file: '.nvmrc' }
# 実運用ではサードパーティ action を SHA でピン留めする（`practices/ci-cd/github-actions.md` #9）
- uses: pnpm/setup@v3          # pnpm v11+ を package.json の packageManager のバージョンで導入
  with: { node-version-file: false, cache: true, install: false }   # Node は setup-node に任せ、install は次のステップで明示する

- run: pnpm install --frozen-lockfile
  # lockfile と package.json が不一致なら fail
```

pnpm 11 以降の GitHub Actions では、`pnpm/action-setup` の後継の `pnpm/setup` を使う（`pnpm/action-setup` は pnpm 10 以前向け）。`actions/setup-node` の `cache: pnpm` は、pnpm が先に PATH にないと失敗するので使わず、キャッシュは `pnpm/setup` の `cache: true` で行う。

**`pnpm-workspace.yaml` で挙動を強制**:
```yaml
# pnpm-workspace.yaml（pnpm 11 は認証・レジストリ以外の設定を .npmrc から読まない）
autoInstallPeers: true
strictPeerDependencies: true
shamefullyHoist: false
nodeLinker: isolated  # symlink ベース（デフォルト）
preferFrozenLockfile: true

# engines.node を満たさない依存パッケージのインストールを拒否する
engineStrict: true
```

`.npmrc` には認証・レジストリの設定だけを書く。pnpm 10 以前は上記を `.npmrc` に kebab-case（`auto-install-peers=true` など）で書いていた。設定場所の変更で起きる CI 専用の失敗は [`security/dependency-security.md`](../security/dependency-security.md) #11 を参照。

**`only-allow` で他のパッケージマネージャを禁止**:
```bash
# preinstall で npm / yarn を弾く
npx only-allow pnpm
```

**`.nvmrc` でも Node.js バージョン管理**:
```
# .nvmrc
24.22.1
```

Volta を使う場合:
```json
// package.json
{
  "volta": {
    "node": "24.22.1",
    "pnpm": "11.28.8"
  }
}
```

**lockfile の競合解消**:
```bash
# 手動で編集しない
# 一方を採用 → pnpm install で再生成
git checkout --theirs pnpm-lock.yaml
pnpm install
```

pnpm は競合マーカーの入った `pnpm-lock.yaml` も `pnpm install` で自動解消できるが、どちらの版が採られたかは commit 前に diff で確認する。壊れたエントリは `pnpm install --fix-lockfile` で直す。

**pre-commit hook で unsync を検知する**（`package.json` の編集忘れ対策）:
```bash
# .husky/pre-commit
# package.json が staged のときだけ、lockfile が依存定義と一致しているかを検証する
if git diff --cached --name-only | grep -qE '(^|/)package\.json$'; then
  if ! pnpm install --frozen-lockfile > /dev/null; then
    echo "pnpm-lock.yaml が package.json の依存定義と一致しません。'pnpm install' を実行してください。"
    exit 1
  fi
  if ! git diff --quiet -- pnpm-lock.yaml; then
    echo "pnpm-lock.yaml の変更が stage されていません。'git add pnpm-lock.yaml' を実行してください。"
    exit 1
  fi
fi
```
staged のファイル名だけで判定すると、`scripts` や `description` だけを変えた正常な commit まで止めてしまう。依存定義との整合は `--frozen-lockfile` に判定させる。
CI が fail してから気づくより、ローカルの commit 時点で検知した方が手戻りが小さい。CI ログを読む際も「Install dependencies のステップ」から遡って原因を特定する。

**判断軸**:
- 新規プロジェクト → pnpm + `packageManager` フィールド（pnpm 11 以降は Corepack なしで版が切り替わる）
- 既存プロジェクト（npm / yarn 採用）→ 既存を維持、ただし lockfile + `packageManager` は徹底
- Bun を使うなら `bun install --frozen-lockfile` も同様にサポート

**どのパッケージマネージャーを選ぶか**:
- pnpm（推奨デフォルト）: disk 使用量最小、リンク経由のインストールで速い、モノレポ対応良好
- npm: Node.js 同梱、追加インストール不要、CI で安定
- yarn (Berry): Plug'n'Play で `node_modules` を使わない設計、上級者向け
- bun: 速度最強だが、まだ実戦投入のリスク残る

**出典**:
- [Node.js: Corepack](https://nodejs.org/docs/latest-v24.x/api/corepack.html) (Node.js、「Corepack will no longer be distributed starting with Node.js v25」) ※2026-10-25 確認
- [nodejs/corepack](https://github.com/nodejs/corepack) (Node.js、同梱範囲と `npm install -g corepack` による導入) ※2026-10-25 確認
- [pnpm: Settings (pnpm-workspace.yaml)](https://pnpm.io/11.x/settings) (pnpm、「Only auth and registry settings are read from `.npmrc` files.」) ※2026-10-25 確認
- [pnpm: pmOnFail](https://pnpm.io/11.x/settings/cli#pmonfail) (pnpm、v11.0.0 で追加、既定 `download`) ※2026-10-25 確認
- [pnpm install](https://pnpm.io/11.x/cli/install) (pnpm、`--frozen-lockfile` は CI で lockfile があれば既定 true、`--fix-lockfile`) ※2026-10-25 確認
- [pnpm: Continuous Integration](https://pnpm.io/11.x/continuous-integration) (pnpm、GitHub Actions は `pnpm/setup` を使う) ※2026-10-25 確認
- [pnpm/setup](https://github.com/pnpm/setup) (pnpm、pnpm v11+ 専用。版は `packageManager` から読む。`cache` / `install` / `node-version-file` 入力) ※2026-10-25 確認
- [actions/setup-node](https://github.com/actions/setup-node) (GitHub、`cache` を使うときは「Package manager should be pre-installed」) ※2026-10-25 確認
- [pnpm: Working with Git](https://pnpm.io/11.x/git) (pnpm、lockfile の競合の自動解消) ※2026-10-25 確認
- [packageManager 仕様](https://nodejs.org/docs/latest/api/packages.html#packagemanager) (Node.js)
- [package.json 編集後の lockfile 同期忘れで CI 7 ジョブ同時 fail した話](https://qiita.com/teppei19980914/items/3be544670087e8cc0b33) (Qiita、Husky pre-commit での unsync 検知パターン) ※2026-07-05に実際にfetch成功

**出典引用**:
> "package.json が staging にあるが、pnpm-lock.yaml がない → エラー"
> ([package.json 編集後の lockfile 同期忘れで CI 7 ジョブ同時 fail した話](https://qiita.com/teppei19980914/items/3be544670087e8cc0b33), セクション "pre-commit hook で防ぐ") ※2026-07-05に実際にfetch成功

**バージョン**: pnpm 11+（pnpm 10 以前は設定を `.npmrc` に書く）, Node.js 24+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 2. メジャー / マイナー / パッチで更新ポリシーを区別する

すべての更新を同じ重みで扱わない。
パッチ（バグ修正）は自動マージ、メジャー（破壊的変更）は手動レビュー、マイナーは依存の種類で分ける。

**根拠**:
- semver では `patch` は破壊的変更なし、`minor` は機能追加（後方互換）、`major` は破壊的変更
- すべて自動マージすると大規模 breaking change を見逃す
- すべて手動レビューだと CI / レビュー疲労
- 自動化と慎重さのバランスをポリシーで明文化する

**更新ポリシー**:
更新タイプごとの自動マージの範囲（patch は公開から 7 日たった版を CI green で自動マージ、本番依存の minor と major は人がレビュー、セキュリティ修正は待機期間・スケジュールの外で即時 PR）と、Renovate / Dependabot の設定例は [`security/dependency-security.md`](../security/dependency-security.md) #2 を正本とする。ここでは range の決め方と、major 更新の進め方を扱う。

**バージョン pinning vs ranged**:
```json
{
  "dependencies": {
    "next": "15.0.0",           // pin（厳密に 15.0.0）
    "react": "^18.3.0",          // ^ minor 範囲
    "typescript": "~5.5.0"       // ~ patch 範囲
  }
}
```

**判断軸**:
| パッケージ種別 | range | 理由 |
|---|---|---|
| 本体（Next.js / React / TypeScript） | pin or `~` | 想定外の動作差を避ける |
| 安定ライブラリ（lodash / date-fns） | `^` | minor は安全 |
| ESLint plugin / 開発ツール | `^` | 軽い更新は許容 |
| `@types/*` | `^` | バンドルされないので自由 |

**Renovate の `rangeStrategy`**:
- `bump`: lockfile も含めて更新（推奨）
- `pin`: 厳密バージョンに固定
- `update-lockfile`: range 内の更新は lockfile だけを更新する。range 外の更新は `replace` として扱い、package.json も書き換える
- `in-range-only`: range 内の更新だけを lockfile に反映し、package.json は変えない（range 外の更新は捨てるので、気づかないまま古い版に留まりやすい）

**Major upgrade の手順**:
1. CHANGELOG / Migration Guide を読む
2. preview deploy で動作確認
3. Visual Regression / E2E が green
4. PR description に migration ノート
5. staging 環境でしばらく観測
6. main にマージ → production デプロイ
7. release notes に明記

**複数メジャーバージョンを一気に飛ばす場合の追加手順**（例: v75 → v96 のような大幅ジャンプ）:
- バージョン単位ではなく「破壊的変更のパターン」単位でグルーピングし、同じ種類の修正をバッチでまとめて適用する（コンポーネント単位・バージョン単位の逐次対応は重複作業を生む）
- パターンごとに「どの画面で何を確認するか」の動作確認項目を洗い出し、確認漏れと重複確認の両方を防ぐ
- 各バッチ適用後、次のバージョン群に破壊的変更が無いか確認してから、まとめて先に進められるか判断する（段階的検証）
- 再現可能な移行フローとして手順を明文化しておくと、次回以降の大幅ジャンプ移行を再利用できる

**出典引用**:
> "変更パターンごとに、どの画面で何を確認するかの動作確認項目を洗い出す"
> ([内定者インターンがUIライブラリを20バージョン分アップデートするまで](https://tech.smarthr.jp/entry/2026/07/10/103000), セクション "何度もやるうちに「うまくいく流れ」が見えてきた") ※2026-07-10に実際にfetch成功

**出典**:
- [semver](https://semver.org/) (Semantic Versioning)
- [Renovate Docs](https://docs.renovatebot.com/) (Mend)
- [Renovate: Configuration Options](https://docs.renovatebot.com/configuration-options/) (Mend、`rangeStrategy` の `update-lockfile` / `in-range-only`) ※2026-10-25 確認
- [Dependabot Docs](https://docs.github.com/en/code-security/dependabot) (GitHub)
- [内定者インターンがUIライブラリを20バージョン分アップデートするまで](https://tech.smarthr.jp/entry/2026/07/10/103000) (SmartHR Tech Blog、v75.4.1→v96・約950ファイル・13PRの実例) ※2026-07-10に実際にfetch成功

**バージョン**: パターン
**確信度**: 高
**最終更新**: 2026-10-25

---

### 3. `peerDependencies` を明示し、duplicate 検出する

React / TypeScript のような「他パッケージが既に持っていること期待」する依存は `peerDependencies` に置く。
モノレポでも各パッケージで明示する。

**根拠**:
- ライブラリが内部で React を `dependencies` に持つと、利用側に react が 2 つ存在することになり Hook が壊れる
- `peerDependencies` で「ホスト側で同じバージョンを使ってね」と宣言する
- pnpm は `peerDependencies` の不一致を検出して警告する（npm はやや甘い）
- ホスト側のバージョンと違うとランタイムエラーになる微妙なバグの予防

**コード例**:
```json
// packages/ui/package.json
{
  "name": "@my-monorepo/ui",
  "peerDependencies": {
    "react": "^18.0.0 || ^19.0.0",
    "react-dom": "^18.0.0 || ^19.0.0"
  },
  "devDependencies": {
    "react": "^18.3.0",         // 開発時用（テストで使う）
    "react-dom": "^18.3.0"
  }
  // dependencies に react を入れない
}
```

**Optional peer dependencies**:
```json
{
  "peerDependencies": {
    "next": "^14.0.0 || ^15.0.0"
  },
  "peerDependenciesMeta": {
    "next": {
      "optional": true  // Next.js なしでも使える
    }
  }
}
```

**重複依存の検出**:
```bash
# pnpm で重複検出
pnpm why react

# 出力例:
# my-app
# ├── react 18.3.1
# └─┬ some-lib
#   └── react 17.0.2  ← 重複！

# 解消方法
pnpm dedupe

# 強制 override
# pnpm は pnpm-workspace.yaml の overrides、yarn は package.json の resolutions を設定
```

**`overrides` で強制統一**:
```yaml
# pnpm-workspace.yaml（pnpm 11 は package.json の "pnpm" フィールドを読まない）
overrides:
  react: ^18.3.0
  react-dom: ^18.3.0
```

これで transitive deps も含めて react のバージョンが統一される。pnpm 10 以前は package.json の `pnpm.overrides` に書く。

**npm の `overrides`** / yarn の `resolutions` も同等機能を持つ。

**`peerDependencies` の判断軸**:
| パッケージ種別 | 配置 |
|---|---|
| React コンポーネントライブラリの `react` / `react-dom` | peerDependencies |
| TypeScript の compiler API を使うツール（ESLint プラグイン等）の `typescript` | peerDependencies |
| 型定義付きの JS を配布する通常のライブラリの `typescript` | devDependencies |
| Next.js 専用プラグインの `next` | peerDependencies (optional 可) |
| Tailwind プラグインの `tailwindcss` | peerDependencies |
| Internal な型・ユーティリティ | dependencies |
| 完全に self-contained なライブラリ | dependencies |

**peer dep 不一致時のエラー例**:
```
ERR_PNPM_PEER_DEP_ISSUES Unmet peer dependencies

✗ react: found 17.0.2 in lockfile
       want ^18.0.0 (from @my-monorepo/ui)
```

**自動修正フロー**:
```yaml
# pnpm-workspace.yaml
# autoInstallPeers は既定で有効（pnpm 8+）。必要に応じて明示する
autoInstallPeers: true
strictPeerDependencies: true  # 不一致を error にする
```

**出典**:
- [npm: peerDependencies](https://docs.npmjs.com/cli/v10/configuring-npm/package-json#peerdependencies) (npm)
- [pnpm: Peer Dependencies](https://pnpm.io/package_json#peerdependencies) (pnpm)
- [pnpm: overrides](https://pnpm.io/11.x/settings/dependency-resolution#overrides) (pnpm、pnpm 11 は `pnpm-workspace.yaml` に書く) ※2026-10-25 確認
- [pnpm v11.0.0 リリースノート](https://github.com/pnpm/pnpm/releases/tag/v11.0.0) (pnpm、「pnpm no longer reads settings from the `pnpm` field of `package.json`」) ※2026-10-25 確認

**バージョン**: pnpm 11+（pnpm 10 以前は package.json の `pnpm` フィールドと `.npmrc` に書く）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 4. 未使用依存を `knip` / `depcheck` で定期検出する

`package.json` に書かれているが実際にはコードで使われていない依存パッケージを定期的に検出して削除する。
未使用 deps は bundle size 増・脆弱性経路・install 時間増の原因。

**根拠**:
- 機能削除した時に dependency を消し忘れがち（コードは消したが import 削除前）
- bundle に乗らなくても install 時間とリスクは残る
- `knip` / `depcheck` は静的解析で未使用 export / 未使用 deps を検出
- CI に組み込めば自動防御

**`knip` セットアップ**:
```bash
pnpm add -D knip
```

```json
// knip.json
{
  "$schema": "https://unpkg.com/knip@5/schema.json",
  "entry": ["src/index.ts", "src/app/**/page.tsx", "src/app/**/route.ts"],
  "project": ["src/**/*.{ts,tsx}"],
  "ignore": [
    "src/types/global.d.ts"
  ],
  "ignoreDependencies": [
    "tailwindcss",     // postcss config から間接利用
    "@types/node"      // tsconfig types で使用
  ],
  "next": {
    "config": ["next.config.{js,ts}"]
  }
}
```

```bash
# 実行
pnpm knip
# 出力:
# Unused dependencies (3)
#   lodash    package.json:15
#   axios     package.json:18
#   moment    package.json:21
# Unused devDependencies (1)
#   @types/lodash    package.json:42
# Unused exports (5)
#   formatPrice    src/utils/format.ts:12
# Unused types (2)
#   ApiResponse    src/types.ts:8
```

**CI に組み込む**:
```yaml
# .github/workflows/ci.yml
- name: Check unused dependencies and exports
  run: pnpm knip --reporter compact
```

**`depcheck`（より古典的）**:
```bash
pnpm dlx depcheck

# 出力:
# Unused dependencies
# * axios
# Unused devDependencies
# * @types/old-pkg
# Missing dependencies
# * lodash (used in: src/utils.ts)
```

**`knip` vs `depcheck`**:
- `knip` は未使用 dep + 未使用 export + 未使用ファイル + 未使用型を一括検出（推奨）
- `depcheck` は dep の使用有無のみ

**よくある誤検知**:
- TypeScript 用の型パッケージ（`@types/*`）
- ESLint plugin / config 用パッケージ
- `next.config.ts` から動的に require されるパッケージ（postcss / tailwind）
- CLI ツール（`packageJson.scripts` から実行）

`knip.ignoreDependencies` または `depcheck` の `ignores` で除外する。

**未使用ファイルの検出**:
```bash
pnpm knip --include files
# 未使用 .ts / .tsx / .css ファイルを列挙
```

**未使用 export の検出**:
```bash
pnpm knip --include exports
# 他で import されていない関数・型・コンポーネントを列挙
```

これにより「barrel index.ts に再エクスポートしているが、誰も使っていない型」を発見できる。

**運用ルール**:
- 月次で `knip` を実行 → PR で deps cleanup
- CI で未使用 dep が増えていないか chec（差分ベース）
- 大規模削除は別 PR にしてレビューしやすく

**ライセンスチェック（補助）**:
```bash
pnpm dlx license-checker --production --excludePrivatePackages

# 禁止ライセンスを設定
pnpm dlx license-checker --failOn 'GPL;AGPL'
```

**判定マトリクス**:
| 検出 | アクション |
|---|---|
| dependency が package.json にあるが import なし | 削除 |
| dependency が import されているが package.json になし | 追加（CI で fail） |
| devDependency を production code で import | dependencies に移動 |
| dependencies を test ファイルだけで使用 | devDependencies に移動 |
| barrel export が誰も import していない | export 削除 or ファイル削除 |

**出典**:
- [knip](https://knip.dev/) (knip)
- [depcheck](https://github.com/depcheck/depcheck) (depcheck)
- [license-checker](https://github.com/davglass/license-checker)

**バージョン**: knip 5+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 5. ローカル修正（patch-package / pnpm patch）を仕組み化する

依存パッケージにバグがあって upstream に PR を出している間など、ローカルパッチを当てる仕組みを用意する。
`patch-package`（npm/yarn）または `pnpm patch` を使い、ad-hoc な `node_modules` 編集を避ける。

**根拠**:
- 上流に PR を出してもマージまで数週間〜数ヶ月かかることがある
- 一方で本番に influence があるバグは即修正したい
- 手動で `node_modules` を編集すると `pnpm install` 時に消える
- `patch-package` / `pnpm patch` は差分を `.patch` ファイルとして保存し、install 時に自動適用

**`pnpm patch` 使用例**:
```bash
# パッチを当てたいパッケージのコピーを作成
pnpm patch react@18.3.1
# 出力: /tmp/d8a32...

# そのディレクトリでファイルを編集
# /tmp/d8a32.../cjs/react.production.min.js 等

# 編集が終わったら commit
pnpm patch-commit /tmp/d8a32...
# → patches/react@18.3.1.patch が作成される
# → pnpm-workspace.yaml に patchedDependencies が追加される
```

```yaml
# pnpm-workspace.yaml（pnpm 10 以前は package.json の pnpm.patchedDependencies）
patchedDependencies:
  react@18.3.1: patches/react@18.3.1.patch
```

**`patches/react@18.3.1.patch` の中身**:
```diff
diff --git a/cjs/react.production.min.js b/cjs/react.production.min.js
index ...
--- a/cjs/react.production.min.js
+++ b/cjs/react.production.min.js
@@ -123,5 +123,8 @@
-  if (someCondition) throw new Error('...');
+  if (someCondition) {
+    console.warn('Patched: ignoring error');
+    return null;
+  }
```

**install 時に自動適用**:
```bash
pnpm install
# → patches/ にあるパッチが該当パッケージに自動適用される
```

**運用ルール**:
1. パッチを当てたら **上流に Issue / PR を作成**（永続的なパッチは技術負債）
2. パッチファイル内に「なぜパッチが必要か」「上流 PR の URL」をコメント
3. パッケージ更新時にパッチが当たらなくなる → 検証して更新
4. パッチは最小限に（行数が多いほどメンテコスト上がる）

**npm / pnpm の `overrides` も同じ「技術負債」として扱う**:
`overrides` は依存グラフ内のバージョン番号を強制的に書き換えるだけで、
上書き対象パッケージのコードが実際にそのバージョンと互換かは検証しない。
`ERESOLVE` エラーを消すためだけにコピペで追加すると、非互換が黙って残ったまま
CI やプロダクションで後から顕在化する。可能な限りスコープを絞り（下記 Bad/Good 参照）、
一時しのぎとして上流修正後に削除する運用ルールを patch-package と同様に適用する。

```json
// Bad: 影響範囲が広い（全ての依存が対象になる）
"overrides": {
  "react": "$react"
}

// Good: 影響範囲を特定パッケージ配下のみに限定
"overrides": {
  "@testing-library/react-hooks": {
    "react": "$react"
  }
}
```

**判断軸（パッチを当てるべきか）**:
| 状況 | 対応 |
|---|---|
| 緊急 + 上流 PR にも出した | patch-package で一時しのぎ |
| upstream が応答しない | パッチ継続 or fork 維持 |
| 自社専用の挙動変更 | fork して private publish |
| バンドラの transform で済む | patch せず resolve.alias 等 |
| 簡単な monkey patch | コード側で wrap して回避 |

**Yarn / npm の場合（patch-package）**:
```bash
# patch-package インストール
pnpm add -D patch-package postinstall-postinstall

# scripts に追加
"postinstall": "patch-package"

# パッケージを直接編集後
npx patch-package <package-name>
# → patches/<package-name>+<version>.patch が生成
```

**長期的な対応**:
- パッチは技術負債として記録
- 月次で「現在当てているパッチ一覧」をレビュー
- 上流マージされたパッチは即削除

**出典**:
- [pnpm patch](https://pnpm.io/11.x/cli/patch) (pnpm、pnpm 11 では `patchedDependencies` を `pnpm-workspace.yaml` に書く) ※2026-10-25 確認
- [patch-package](https://github.com/ds300/patch-package) (ds300)
- [The npm `overrides` Snippet Everyone Copy-Pastes — And What It Actually Does](https://dev.to/mayank7924/the-npm-overrides-snippet-everyone-copy-pastes-and-what-it-actually-does-3flg) (dev.to mayank7924、`overrides` のスコープ限定パターンと「設定ではなく負債として扱う」原則) ※2026-08-03に実際にfetch成功
  > "If you do add an override, treat it like debt, not like settings."
  > (セクション "A temporary patch. Not permanent config")

**バージョン**: pnpm 8+（`patchedDependencies` の置き場所は pnpm 11 から `pnpm-workspace.yaml`）
**確信度**: 高
**最終更新**: 2026-10-25
