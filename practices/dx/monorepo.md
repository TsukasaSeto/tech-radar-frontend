# モノレポ運用のベストプラクティス

pnpm workspace + Turborepo を中心としたモノレポ構成。
パッケージ境界・共通設定の集約・ビルド順序の管理がポイント。

## ルール

### 1. pnpm workspace + Turborepo を推奨デフォルト構成にする

モノレポは pnpm workspace（依存管理）+ Turborepo（ビルドオーケストレーション）の組み合わせを採用する。
両者は補完関係にあり、それぞれ別の役割を担う。

**根拠**:
- pnpm workspace は依存解決とパッケージリンクに特化、ディスク使用量が圧倒的に小さい
- Turborepo は task graph / リモートキャッシュ / 増分ビルドに特化
- npm workspaces / yarn workspaces でも動くが、pnpm が最も成熟しており速い
- Nx は強力だが学習コストが高い。Turborepo は学習コストが低く、後から Nx に移行も可能

**標準構成**:
```
my-monorepo/
├── apps/
│   ├── web/                 # Next.js (例: ユーザー向けアプリ)
│   ├── admin/               # Next.js (例: 管理画面)
│   └── docs/                # Astro / Docusaurus (例: ドキュメント)
├── packages/
│   ├── ui/                  # 共通 UI コンポーネント
│   ├── types/               # 共有型定義
│   ├── api-client/          # API クライアントラッパー
│   ├── eslint-config/       # 共通 ESLint 設定
│   ├── tsconfig/            # 共通 tsconfig
│   └── tailwind-config/     # 共通 Tailwind 設定
├── package.json
├── pnpm-workspace.yaml
├── turbo.json
└── tsconfig.base.json
```

**`pnpm-workspace.yaml`**:
```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

**ルート `package.json`**:
```json
{
  "name": "my-monorepo",
  "private": true,
  "packageManager": "pnpm@9.12.0",
  "engines": {
    "node": ">=20.0.0",
    "pnpm": ">=9.0.0"
  },
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev --parallel",
    "lint": "turbo run lint",
    "test": "turbo run test",
    "typecheck": "turbo run typecheck",
    "clean": "turbo run clean && rm -rf node_modules"
  },
  "devDependencies": {
    "turbo": "^2.0.0",
    "typescript": "^5.5.0"
  }
}
```

**`turbo.json`**:
```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": [".env"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "dist/**", "!.next/cache/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": { "outputs": [".eslintcache"] },
    "typecheck": { "dependsOn": ["^build"] },
    "test": { "dependsOn": ["^build"], "outputs": ["coverage/**"] }
  }
}
```

**`apps/web/package.json`**:
```json
{
  "name": "@my-monorepo/web",
  "private": true,
  "dependencies": {
    "@my-monorepo/ui": "workspace:*",
    "@my-monorepo/api-client": "workspace:*",
    "@my-monorepo/types": "workspace:*",
    "next": "^15.0.0"
  }
}
```

**workspace: プロトコル**:
- `"@my-monorepo/ui": "workspace:*"` — モノレポ内のパッケージへの参照
- `"workspace:^1.0.0"` — semver 制約付き
- `pnpm publish` 時に自動で実バージョンに置換される

**選択肢の比較**:

| ツール | 強み | 弱み |
|---|---|---|
| **pnpm + Turborepo** | 軽量、Vercel との統合、学習コスト低 | 大規模では Nx ほどの機能はない |
| **Nx** | グラフ可視化・generators・大規模対応 | 学習コスト高 |
| **Bazel** | 言語横断・超大規模 | フロントエンド単独では過剰 |
| **Lerna** | 古参・歴史 | 主流から外れている（Lerna v7+ は Nx 統合） |
| **Rush** | Microsoft 製・厳格 | 学習コスト・コミュニティ小 |

**出典**:
- [pnpm: Workspaces](https://pnpm.io/workspaces) (pnpm)
- [Turborepo Docs](https://turborepo.com/docs) (Vercel)
- [Nx Docs](https://nx.dev/) (Nrwl)

**バージョン**: pnpm 9+, Turborepo 2+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 2. パッケージ境界を `apps/` / `packages/` の 2 層で明確化する

`apps/` にはデプロイ単位（Web アプリ・ドキュメントサイト等）、`packages/` には apps 間で共有する内部ライブラリを置く。
パッケージは「公開しない（private: true）」または「npm 公開する（access: public）」を最初に決める。

**根拠**:
- フォルダ構造で「これは何のためにあるか」が即座に判別できる
- 依存方向のルール化が容易（`apps/` は `packages/` に依存可、逆は禁止）
- ESLint の `no-restricted-imports` で境界違反を機械的に検出できる
- 「全パッケージが互いに依存する」 spaghetti モノレポを防ぐ

**`packages/` の分類**:
```
packages/
├── ui/              # コンポーネントライブラリ
├── api-client/      # API スキーマ / クライアント
├── types/           # 共有型定義
├── utils/           # 汎用ユーティリティ
├── icons/           # アイコンセット
├── eslint-config/   # 共通 ESLint 設定（config パッケージ）
├── tsconfig/        # 共通 tsconfig（config パッケージ）
└── tailwind-config/ # 共通 Tailwind 設定（config パッケージ）
```

**設計原則**:
- パッケージは **単一責任**（責任が膨らんだら分割を検討）
- `apps/` 間は **依存しない**（共通コードは `packages/` に移動）
- `packages/` は他の `packages/` に依存可、ただし循環依存禁止

**循環依存の禁止**:
```bash
# 循環依存を検出するツール
pnpm dlx madge --circular --extensions ts,tsx packages/
```

ESLint で強制:
```js
// .eslintrc.js
'import/no-cycle': ['error', { maxDepth: 5 }]
```

**ESLint で依存方向を強制**:
```js
// packages/ui/.eslintrc.js
'no-restricted-imports': ['error', {
  patterns: [
    '@my-monorepo/web/*',      // ui は web に依存禁止
    '@my-monorepo/admin/*',
  ]
}]
```

**npm 公開する場合**:
```json
// packages/ui/package.json
{
  "name": "@my-monorepo/ui",
  "version": "1.0.0",
  "access": "public",
  "files": ["dist"],
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.js"
    },
    "./styles": "./dist/styles.css"
  }
}
```

**internal only（公開しない）の場合**:
```json
{
  "name": "@my-monorepo/types",
  "private": true,
  "main": "./src/index.ts",
  "types": "./src/index.ts"
}
```

`private: true` + `main: src/index.ts` でビルド不要・TypeScript 直接読み込み構成（モノレポ内のみ使用時の最適化）。

**判断軸**:
| パッケージタイプ | 配置 | private |
|---|---|---|
| ユーザー向け Web アプリ | `apps/web/` | true |
| 管理画面 | `apps/admin/` | true |
| 共通 UI コンポーネント（社内のみ） | `packages/ui/` | true |
| 共通 UI コンポーネント（OSS 公開） | `packages/ui/` | false |
| 設定ファイル（eslint-config 等） | `packages/<name>-config/` | true |

**出典**:
- [Turborepo: Filtering](https://turborepo.com/docs/reference/run#--filter-string) (Vercel)
- [Monorepo.tools](https://monorepo.tools/) (Nrwl)

**バージョン**: パターン
**確信度**: 高
**最終更新**: 2026-05-16

---

### 3. 共通設定（tsconfig / eslint / prettier）を config パッケージに集約する

`packages/tsconfig/` / `packages/eslint-config/` を作成し、各 app / package はこれを extends する。
個別ファイルに設定をコピーすると保守地獄になる。

**根拠**:
- モノレポでは「tsconfig が 20 個ある」状態になりがち
- 共通設定を 1 箇所で管理できれば、ルール変更が全パッケージに即時反映
- TypeScript の `extends`、ESLint の `extends`、Prettier の `prettierrc` 経由で集中管理可能
- ベース設定 + パッケージ別 override の 2 層構造が現実的

**`packages/tsconfig/` 構成**:
```
packages/tsconfig/
├── package.json
├── base.json           # 全パッケージ共通
├── nextjs.json         # Next.js apps 向け
├── react-library.json  # React コンポーネントライブラリ向け
└── node.json           # Node.js スクリプト向け
```

```json
// packages/tsconfig/base.json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "useUnknownInCatchVariables": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true
  }
}
```

```json
// packages/tsconfig/nextjs.json
{
  "extends": "./base.json",
  "compilerOptions": {
    "lib": ["dom", "dom.iterable", "esnext"],
    "jsx": "preserve",
    "allowJs": true,
    "incremental": true,
    "plugins": [{ "name": "next" }]
  }
}
```

```json
// apps/web/tsconfig.json
{
  "extends": "@my-monorepo/tsconfig/nextjs.json",
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["next-env.d.ts", "src/**/*"]
}
```

**`packages/eslint-config/` 構成**:
```js
// packages/eslint-config/index.js (Flat config)
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import reactPlugin from 'eslint-plugin-react';
import hooksPlugin from 'eslint-plugin-react-hooks';
import jsxA11y from 'eslint-plugin-jsx-a11y';

export default [
  js.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked,
  {
    plugins: { react: reactPlugin, 'react-hooks': hooksPlugin, 'jsx-a11y': jsxA11y },
    rules: {
      'react/react-in-jsx-scope': 'off',
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'warn',
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      '@typescript-eslint/consistent-type-imports': 'error',
    },
  },
];
```

```js
// apps/web/eslint.config.js
import baseConfig from '@my-monorepo/eslint-config';

export default [
  ...baseConfig,
  {
    files: ['src/**/*.{ts,tsx}'],
    rules: {
      // 個別 override
    },
  },
];
```

**Prettier の共有**:
```json
// packages/prettier-config/index.json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "plugins": ["prettier-plugin-tailwindcss"]
}
```

```json
// apps/web/package.json
{
  "prettier": "@my-monorepo/prettier-config"
}
```

**Tailwind の共有**:
```ts
// packages/tailwind-config/index.ts
import type { Config } from 'tailwindcss';

const config: Config = {
  content: ['./src/**/*.{ts,tsx}'],
  theme: {
    extend: {
      colors: {
        primary: '#3b82f6',
        secondary: '#10b981',
      },
    },
  },
};

export default config;
```

```ts
// apps/web/tailwind.config.ts
import baseConfig from '@my-monorepo/tailwind-config';

export default {
  ...baseConfig,
  content: [
    './src/**/*.{ts,tsx}',
    '../../packages/ui/src/**/*.{ts,tsx}',  // UI パッケージのファイルも対象に
  ],
};
```

**判断軸**:
- 2 パッケージ以上で同じ設定をコピペしている → config パッケージに抽出
- ベース + override の 2 層が原則（3 層は複雑になりすぎ）
- 「設定の集中管理」と「カスタマイズの柔軟性」のバランス

**出典**:
- [Turborepo: Sharing TypeScript config](https://turborepo.com/docs/handbook/linting/typescript) (Vercel)
- [ESLint: Configuration Files](https://eslint.org/docs/latest/use/configure/configuration-files) (ESLint)

**バージョン**: TypeScript 5+, ESLint 9+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 4. `package.json` の `exports` フィールドで public API を制御する

パッケージのエクスポートは `exports` フィールドで明示し、内部実装ファイルへの直接アクセスを禁止する。
import パスから「これは public API か内部実装か」が読み手に伝わる。

**根拠**:
- `main` だけだと内部ファイル（`@my-pkg/ui/src/internal/private.ts`）まで import 可能で、依存先が暗黙に広がる
- `exports` で明示すれば、TypeScript / Node.js / bundler すべてが respect する
- Public API の意図が package.json で文書化される
- 内部リファクタの自由度が上がる（exports に出ていないファイルは自由に変更可）

**標準的な `exports` 構成**:
```json
{
  "name": "@my-monorepo/ui",
  "version": "1.0.0",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    },
    "./button": {
      "types": "./dist/button.d.ts",
      "import": "./dist/button.mjs",
      "require": "./dist/button.cjs"
    },
    "./styles.css": "./dist/styles.css",
    "./package.json": "./package.json"
  },
  "files": ["dist"]
}
```

**使う側のインポート**:
```tsx
// Good: exports に定義されたパス
import { Button } from '@my-monorepo/ui';
import { Card } from '@my-monorepo/ui/card';
import '@my-monorepo/ui/styles.css';

// Bad: 内部ファイル直接アクセス（exports で禁止）
import { Button } from '@my-monorepo/ui/dist/button.mjs';
import { internalUtil } from '@my-monorepo/ui/src/internal/util.ts';
// → Error: Cannot find module '@my-monorepo/ui/src/internal/util.ts'
```

**TypeScript で `exports` を respect させる**:
```json
// tsconfig.json
{
  "compilerOptions": {
    "moduleResolution": "bundler",  // または "node16" / "nodenext"
    // ↑ "node" だと exports を無視するため
  }
}
```

**条件付きエクスポート（環境別）**:
```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "react-server": "./dist/index.rsc.mjs",  // Server Components 用
      "browser": "./dist/index.browser.mjs",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

**Sub-path patterns（ワイルドカード）**:
```json
{
  "exports": {
    "./icons/*": {
      "types": "./dist/icons/*.d.ts",
      "import": "./dist/icons/*.mjs"
    }
  }
}
```

```tsx
// 使う側
import { ChevronDown } from '@my-monorepo/ui/icons/chevron-down';
```

**Tree-shake friendly な構成**:
```json
// barrel ファイル（index.mjs）の sideEffects 設定
{
  "sideEffects": false,         // 全モジュールが副作用なし
  // または特定ファイルだけ副作用あり
  "sideEffects": [
    "*.css",
    "**/style.{ts,tsx}"
  ]
}
```

**Internal-only な package（モノレポ内のみ使用）**:
```json
{
  "name": "@my-monorepo/types",
  "private": true,
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "exports": {
    ".": "./src/index.ts"
  }
}
```

ビルド不要、TypeScript ファイルを直接読み込ませる構成。

**Verify ツール**:
```bash
# exports の整合性検証
pnpm dlx @arethetypeswrong/cli @my-monorepo/ui
# ✓ Are the types wrong? の評価

pnpm dlx publint @my-monorepo/ui
# ✓ npm publish 前の package.json 検証
```

**`files` フィールドとの関係**:
- `exports`: どのパスを公開するか
- `files`: npm publish 時に含めるファイル

両方設定するのが安全（`exports` だけだと、依存先で予期せぬファイルが除外されることがある）。

**出典**:
- [Node.js Docs: Package exports](https://nodejs.org/api/packages.html#package-entry-points) (Node.js)
- [TypeScript Docs: Module resolution](https://www.typescriptlang.org/docs/handbook/modules/reference.html#nodenext-node16) (TypeScript)
- [arethetypeswrong/arethetypeswrong](https://github.com/arethetypeswrong/arethetypeswrong.github.io) (arethetypeswrong)
- [publint](https://publint.dev/) (publint)

**バージョン**: Node.js 14.13+, TypeScript 4.7+
**確信度**: 高
**最終更新**: 2026-05-16
