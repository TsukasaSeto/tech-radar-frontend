# 依存管理のベストプラクティス

> **役割分担**: このファイルは **環境再現性 / 開発体験**（lockfile 戦略、packageManager の固定、未使用検出、パッチ管理、更新フロー）を扱う。
> **脆弱性スキャン / supply chain attack 対策 / 自動緊急アップデート**は [`security/dependency-security.md`](../security/dependency-security.md) を参照。
> lockfile 周りは両ファイルで触れるが、**「正しく固定する」は dx 側、「壊れた依存を検知・封じ込める」は security 側** という分担を基本にする。

`package.json` の `dependencies` は本番に乗る代物。雑な追加・放置がそのまま負債とセキュリティリスクになる。

## ルール

### 1. lockfile を必ずコミット、`packageManager` で CLI バージョンを固定する

`pnpm-lock.yaml` 等の lockfile はリポジトリに含め、`package.json` の `packageManager` フィールドでパッケージマネージャ自体のバージョンも固定する。
CI / 本番 / ローカルで完全に同じ依存ツリーが構築されることを保証する。

**根拠**:
- 「ローカルで動くが CI で壊れる」「先週は動いたが今日は動かない」の主因は依存バージョンの非決定性
- lockfile は依存パッケージ・transitive deps・パッチを完全固定する
- `packageManager` フィールド + Corepack で pnpm 自体のバージョンも固定可能
- 同じ commit から make したビルドが常に同じ結果になる reproducibility が CI / 監査の前提

**`package.json`**:
```json
{
  "name": "myapp",
  "packageManager": "pnpm@9.12.0",
  "engines": {
    "node": ">=20.0.0",
    "pnpm": ">=9.0.0"
  },
  "scripts": {
    "preinstall": "npx only-allow pnpm"
  }
}
```

**Corepack で CLI 自動切替**:
```bash
# Corepack 有効化（Node.js 16.9+ 同梱）
corepack enable

# packageManager フィールドに従って自動で pnpm を実行
pnpm install  # → 9.12.0 で実行される
```

**CI で完全一致を強制**:
```yaml
# .github/workflows/ci.yml
- uses: pnpm/action-setup@v3
  # version 指定不要、packageManager フィールドから読む

- uses: actions/setup-node@v4
  with:
    node-version-file: '.nvmrc'  # or 'package.json' で engines.node を読む
    cache: pnpm

- run: pnpm install --frozen-lockfile
  # lockfile と package.json が不一致なら fail
```

**`.npmrc` で挙動を強制**:
```ini
# .npmrc
auto-install-peers=true
strict-peer-dependencies=true
shamefully-hoist=false
node-linker=isolated  # symlink ベース（デフォルト）
prefer-frozen-lockfile=true

# pnpm でなければエラー
engine-strict=true
```

**`only-allow` で他のパッケージマネージャを禁止**:
```bash
# preinstall で npm / yarn を弾く
npx only-allow pnpm
```

**`.nvmrc` でも Node.js バージョン管理**:
```
# .nvmrc
20.18.0
```

Volta を使う場合:
```json
// package.json
{
  "volta": {
    "node": "20.18.0",
    "pnpm": "9.12.0"
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

**判断軸**:
- 新規プロジェクト → pnpm + Corepack + `packageManager` フィールド
- 既存プロジェクト（npm / yarn 採用）→ 既存を維持、ただし lockfile + `packageManager` は徹底
- Bun を使うなら `bun install --frozen-lockfile` も同様にサポート

**出典**:
- [Node.js: Corepack](https://nodejs.org/api/corepack.html) (Node.js)
- [pnpm: Configuration](https://pnpm.io/configuration) (pnpm)
- [packageManager 仕様](https://nodejs.org/docs/latest/api/packages.html#packagemanager) (Node.js)

**バージョン**: pnpm 9+, Node.js 20+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 2. メジャー / マイナー / パッチで更新ポリシーを区別する

すべての更新を同じ重みで扱わない。
パッチ（バグ修正）は自動マージ、メジャー（破壊的変更）は手動レビュー、マイナーは間。

**根拠**:
- semver では `patch` は破壊的変更なし、`minor` は機能追加（後方互換）、`major` は破壊的変更
- すべて自動マージすると大規模 breaking change を見逃す
- すべて手動レビューだと CI / レビュー疲労
- 自動化と慎重さのバランスをポリシーで明文化する

**更新ポリシーの 3 層**:

| 更新タイプ | ポリシー | 例 |
|---|---|---|
| **patch** (1.2.3 → 1.2.4) | CI green なら自動マージ | bug fix |
| **minor** (1.2.3 → 1.3.0) | 1 営業日後に自動マージ（人間レビュー機会あり） | 新機能追加 |
| **major** (1.2.3 → 2.0.0) | 手動マージ必須、migration guide 確認 | breaking change |
| **security** (任意) | 即時マージ（patch 扱いより優先） | 脆弱性修正 |

**Renovate での実装**:
```json
{
  "extends": ["config:recommended"],
  "schedule": ["before 9am on monday"],

  "packageRules": [
    {
      "matchUpdateTypes": ["patch", "pin", "digest"],
      "automerge": true,
      "automergeType": "branch"
    },
    {
      "matchUpdateTypes": ["minor"],
      "matchDepTypes": ["devDependencies"],
      "automerge": true,
      "minimumReleaseAge": "1 day"
    },
    {
      "matchUpdateTypes": ["major"],
      "automerge": false,
      "labels": ["breaking-change"],
      "reviewers": ["team:frontend-leads"]
    },
    {
      "matchPackageNames": ["next", "react", "react-dom", "typescript"],
      "matchUpdateTypes": ["major"],
      "addLabels": ["framework-update"],
      "minimumReleaseAge": "7 days"
    },
    {
      "vulnerabilityAlerts": {
        "enabled": true,
        "schedule": ["at any time"],
        "labels": ["security"],
        "automerge": true
      }
    }
  ]
}
```

**Dependabot での実装**:
```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      patch-and-minor-devdeps:
        dependency-type: "development"
        update-types: ["patch", "minor"]
    labels: ["dependencies"]
    ignore:
      - dependency-name: "next"
        update-types: ["version-update:semver-major"]
        # → major は別途手動 PR
```

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
- `update-lockfile`: lockfile のみ更新（package.json は維持）

**Major upgrade の手順**:
1. CHANGELOG / Migration Guide を読む
2. preview deploy で動作確認
3. Visual Regression / E2E が green
4. PR description に migration ノート
5. staging 環境でしばらく観測
6. main にマージ → production デプロイ
7. release notes に明記

**出典**:
- [semver](https://semver.org/) (Semantic Versioning)
- [Renovate Docs](https://docs.renovatebot.com/) (Mend)
- [Dependabot Docs](https://docs.github.com/en/code-security/dependabot) (GitHub)

**バージョン**: パターン
**確信度**: 高
**最終更新**: 2026-05-16

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
# package.json の "pnpm" / "resolutions" を設定
```

**`pnpm.overrides` で強制統一**:
```json
{
  "pnpm": {
    "overrides": {
      "react": "^18.3.0",
      "react-dom": "^18.3.0"
    }
  }
}
```

これで transitive deps も含めて react のバージョンが統一される。

**npm の `overrides`** / yarn の `resolutions` も同等機能を持つ。

**`peerDependencies` の判断軸**:
| パッケージ種別 | 配置 |
|---|---|
| React コンポーネントライブラリの `react` / `react-dom` | peerDependencies |
| TypeScript 用ライブラリの `typescript` | peerDependencies |
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
```bash
# pnpm は auto-install-peers をデフォルトで有効（pnpm 8+）
# 必要に応じて .npmrc で明示
auto-install-peers=true
strict-peer-dependencies=true  # 不一致を error にする
```

**出典**:
- [npm: peerDependencies](https://docs.npmjs.com/cli/v10/configuring-npm/package-json#peerdependencies) (npm)
- [pnpm: Peer Dependencies](https://pnpm.io/package_json#peerdependencies) (pnpm)
- [pnpm: Overrides](https://pnpm.io/package_json#pnpmoverrides) (pnpm)

**バージョン**: pnpm 8+
**確信度**: 高
**最終更新**: 2026-05-16

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
# → package.json に pnpm.patchedDependencies が追加される
```

```json
// package.json
{
  "pnpm": {
    "patchedDependencies": {
      "react@18.3.1": "patches/react@18.3.1.patch"
    }
  }
}
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
- [pnpm patch](https://pnpm.io/cli/patch) (pnpm)
- [patch-package](https://github.com/ds300/patch-package) (ds300)

**バージョン**: pnpm 8+
**確信度**: 高
**最終更新**: 2026-05-16
