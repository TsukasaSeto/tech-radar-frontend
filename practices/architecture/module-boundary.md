# モジュール境界のベストプラクティス

## ルール

### 1. 依存の方向を一方向に保つ（Dependency Rule）

上位レイヤーが下位レイヤーに依存し、下位レイヤーは上位レイヤーに依存しない。
フィーチャー間の直接依存を避け、共有レイヤー経由でやりとりする。

**根拠**:
- 循環依存（A が B に依存し、B が A に依存）はバンドルの問題・テストの困難さを招く
- 一方向の依存によってレイヤーを独立してテスト・交換できる
- Feature-Sliced Design の「レイヤーのルール」に基づく

**コード例**:
```
依存の方向（上位 → 下位のみ許可）:

app/          ← Next.js ルーティング層
  ↓
features/     ← 機能層（features 間の直接依存を禁止）
  ↓
shared/       ← 共有層（UI、フック、ライブラリ）
  ↓
types/        ← 型定義層

// Bad: feature 間の直接依存
// features/posts/components/PostCard.tsx
import { useAuth } from '@/features/auth/hooks/useAuth';  // feature → feature は禁止

// Good: 共有レイヤーを経由
// features/auth/index.ts で公開し、app 層で組み合わせる
// app/dashboard/page.tsx (上位レイヤーで組み合わせる)
import { useAuth } from '@/features/auth';
import { PostList } from '@/features/posts';

export default async function DashboardPage() {
  return (
    <AuthGuard>
      <PostList />
    </AuthGuard>
  );
}
```

**出典**:
- [Feature-Sliced Design: Layers](https://feature-sliced.design/docs/reference/layers) (Feature-Sliced Design公式)
- [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) (Robert C. Martin)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

#### 追加根拠 (2026-05-06) — ルール1「依存の方向を一方向に保つ（Dependency Rule）」

新たに以下のドキュメントで同じプラクティスが推奨された:
- [Bulletproof React: Project Structure](https://raw.githubusercontent.com/alan2207/bulletproof-react/master/docs/project-structure.md) (alan2207 / masterブランチ) ※2026-05-06に実際にfetch成功

Bulletproof React公式ドキュメントが「Unidirectional Architecture（一方向アーキテクチャ）」として同じ原則を明示: shared modules → features → app layer の方向にのみコードが流れ、features は app 層から import せず shared modules は features から独立を保つ。また "Avoid cross-feature imports; instead compose different features at the application level"（feature 間の直接依存を避け、アプリ層で組み合わせる）という具体的な指針を示している。ESLint の `import/no-restricted-paths` によるレイヤー境界違反の自動検出も推奨されており、本ルール4の実装パターンと一致する。

**確信度**: 既存（高）→ 高（Bulletproof React公式ガイドで実証済み）

---

### 2. 型定義はドメインロジックと一緒に置く

共通の `types/` ディレクトリに型をまとめるのではなく、
型を使う機能・モジュールの近くに置く。

**根拠**:
- 型と実装が近くにあることで、変更時の更新漏れが減る
- 巨大な `types/` ファイルはアンチパターン（何でもまとめると機能境界が曖昧になる）
- TypeScript の `interface` と実装を同じファイルに置くことで凝集度が高まる

**コード例**:
```ts
// Bad: 何でも入る types/index.ts
// src/types/index.ts
export type User = { id: string; name: string; email: string };
export type Post = { id: string; title: string; authorId: string };
export type Cart = { items: CartItem[]; total: number };
export type CartItem = { productId: string; quantity: number };

// Good: 機能と同じディレクトリに型を置く
// features/users/types.ts
export type User = {
  id: string;
  name: string;
  email: string;
  createdAt: Date;
};

// features/posts/types.ts
export type Post = {
  id: string;
  title: string;
  content: string;
  authorId: string;
  publishedAt: Date | null;
};

// 共有型（本当に複数機能にまたがるもの）のみ shared/types/ に
// shared/types/api.ts
export type PaginatedResponse<T> = {
  data: T[];
  total: number;
  page: number;
  pageSize: number;
};

export type ApiError = {
  code: string;
  message: string;
  details?: Record<string, string[]>;
};
```

**出典**:
- [TypeScript Docs: Declaration Merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html) (TypeScript公式。型をドメインの近くに置く主張そのものの根拠ではない)

**バージョン**: TypeScript 5+
**確信度**: 中（主張を直接支える出典がない）
**最終更新**: 2026-10-25

---

### 3. 環境変数はサーバー/クライアントを明示的に分けて管理する

サーバー専用の変数とクライアント公開の変数をモジュール単位で分け、サーバー側のモジュールには `import 'server-only'` を付ける。
`NEXT_PUBLIC_` の扱い、Zod / t3-env での検証、サーバー用スキーマの parse を置く場所は [`security/secret-management.md`](../security/secret-management.md) #1 を正本とする。

**根拠**:
- モジュール境界の観点では、環境変数の読み取りもサーバー専用モジュールとクライアントから読めるモジュールの境界に従わせる（詳細は [`security/secret-management.md`](../security/secret-management.md) #1・#2）

**コード例**: [`security/secret-management.md`](../security/secret-management.md) #1 を参照

**出典**:
- [Next.js Docs: Environment Variables](https://nextjs.org/docs/app/building-your-application/configuring/environment-variables) (Next.js公式)
- [t3-env](https://env.t3.gg/) (T3 Stack)

**バージョン**: Next.js 13.4+（`server-only`）, Zod 3+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 4. `eslint-plugin-import` でモジュール境界違反を自動検出する

`eslint-plugin-import` と `eslint-import-resolver-typescript` を使い、
レイヤー間の不正な依存方向・循環インポートを CI で自動検出する。
レイヤー単位の禁止（entities → features など）は `no-restricted-imports` で書けるが、「自分以外の feature への依存」はパターン 1 つでは表せないため、feature 間の直接依存は後半の dependency-cruiser を必須の検査として CI で止める。
設定は ESLint 9 以降と Next.js 16 の既定である flat config（`eslint.config.mjs`）で書く。

**根拠**:
- コードレビューで依存方向のチェックを人手で行うのはスケールしない
- `import/no-cycle` ルールが循環依存を早期に検出し、バンドルサイズの肥大化を防ぐ
- `no-restricted-imports` でプロジェクト固有のレイヤー規則を Lint ルールとして明文化できる
- ESLint の AST 解析が届かない「依存グラフ全体の検査」には dependency-cruiser を追加する。feature 間クロス依存の防止にはパスの後方参照（`$1`）を使うルールが有効で、例外はベースライン allowlist に明示させることで audit trail をコードに残せる。"アーキテクチャ規約は書いた瞬間から腐り始める——レビューの目視だけでは守りきれない"
- 循環依存検出（reactive）と依存方向ルール（proactive）の2層で守る。Stable Dependencies Principle（SDP）では不安定度 `I = Fan-Out / (Fan-In + Fan-Out)` を指標とし、依存は常に I が小さい（安定した）モジュールへ向かうべきとされる。utils/types（I≈0）← features（I≈0.5）← pages（I≈1.0）の方向を強制することで、変更が波及しにくい構造になる

**コード例**:
```js
// eslint.config.mjs
import importPlugin from 'eslint-plugin-import';
import tsParser from '@typescript-eslint/parser';

export default [
  {
    files: ['src/**/*.{ts,tsx}'],
    languageOptions: { parser: tsParser },
    plugins: { import: importPlugin },
    settings: {
      // eslint-import-resolver-typescript が必要
      'import/resolver': {
        typescript: { alwaysTryTypes: true },
      },
    },
    rules: {
      // 循環インポートを禁止
      'import/no-cycle': ['error', { maxDepth: 3 }],
      // デフォルトエクスポートより名前付きエクスポートを推奨
      'import/prefer-default-export': 'off',
      'import/no-default-export': 'off',
      // インポート順の統一
      'import/order': [
        'error',
        {
          groups: ['builtin', 'external', 'internal', 'parent', 'sibling', 'index'],
          pathGroups: [
            { pattern: '@/app/**', group: 'internal', position: 'before' },
            { pattern: '@/features/**', group: 'internal' },
            { pattern: '@/shared/**', group: 'internal', position: 'after' },
          ],
          'newlines-between': 'always',
          alphabetize: { order: 'asc' },
        },
      ],
    },
  },
  {
    // entities 層: shared より上位への依存を禁止
    files: ['src/entities/**/*.{ts,tsx}'],
    rules: {
      'no-restricted-imports': [
        'error',
        {
          patterns: [
            { group: ['@/features/*'], message: 'entities から features への依存は禁止です' },
            { group: ['@/pages/*'], message: 'entities から pages への依存は禁止です' },
          ],
        },
      ],
    },
  },
  {
    // features 層: pages への依存を禁止（feature 間の依存は下の dependency-cruiser で検査する）
    files: ['src/features/**/*.{ts,tsx}'],
    rules: {
      'no-restricted-imports': [
        'error',
        {
          patterns: [
            { group: ['@/pages/*'], message: 'features から pages への依存は禁止です' },
          ],
        },
      ],
    },
  },
];
```

**コード例（dependency-cruiser — feature 間クロス依存を CI で禁止。必須の検査として CI で実行する）**:
```javascript
// .dependency-cruiser.cjs
module.exports = {
  forbidden: [
    {
      name: 'no-cross-feature',
      severity: 'error',
      from: { path: '^src/features/([^/]+)/' },
      to: {
        path: '^src/features/[^/]+/',
        pathNot: '^src/features/$1/'  // 後方参照で自 feature への依存は許可
      }
    }
  ]
};
```

**出典**:
- [eslint-plugin-import](https://github.com/import-js/eslint-plugin-import) (import-js / GitHub、flat config での設定例) ※2026-10-25 確認
- [Next.js Docs: How to upgrade to version 16 — ESLint Flat Config](https://nextjs.org/docs/app/guides/upgrading/version-16#eslint-flat-config) (Next.js公式、v16 で flat config が既定) ※2026-10-25 確認
- [Feature-Sliced Design: Linting](https://feature-sliced.design/docs/guides/linting) (Feature-Sliced Design公式 / 2023)
- [アーキテクチャを"規約"ではなく"CI"で守る — 依存方向を機械強制するフロントエンドテンプレートの設計](https://zenn.dev/hacobu/articles/5717458ca99f9b) (Zenn hacobu、dependency-cruiser と check-structure.mjs の2層CI設計) ※2026-06-18 fetch
- [Stop circular dependencies before they stop you — dependency-cruiser & the Stable Dependencies Principle](https://dev.to/wojciech_kot_b82f5d7cbfc6/stop-circular-dependencies-before-they-stop-you-dependency-cruiser-the-stable-dependencies-34ho) (dev.to、SDP 不安定度公式・2層防御・3ティア設計) ※2026-06-23 fetch

**出典引用**:
> "アーキテクチャ規約は書いた瞬間から腐りはじめる。レビューの目視だけでは守りきれない。"
> ([アーキテクチャを"規約"ではなく"CI"で守る](https://zenn.dev/hacobu/articles/5717458ca99f9b), セクション "① 依存グラフを検査する：dependency-cruiser") ※2026-06-18に実際にfetch成功

> "Circular dependencies aren't a tooling problem; they are a structural failure. A cycle is simply what happens when dependency arrows point the wrong way for long enough."
> ([Stop circular dependencies before they stop you](https://dev.to/wojciech_kot_b82f5d7cbfc6/stop-circular-dependencies-before-they-stop-you-dependency-cruiser-the-stable-dependencies-34ho), セクション "Core Principle") ※2026-06-23に実際にfetch成功

**バージョン**: eslint-plugin-import 2.31+, ESLint 9+（flat config）, dependency-cruiser 16+
**確信度**: 高
**最終更新**: 2026-10-25

---
