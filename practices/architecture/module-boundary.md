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
- [TypeScript Docs: Declaration Merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html) (TypeScript公式)

**バージョン**: TypeScript 5+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. 環境変数はサーバー/クライアントを明示的に分けて管理する

`NEXT_PUBLIC_` プレフィックスの有無でサーバー専用変数とクライアント公開変数を区別し、
型安全な環境変数アクセスを実装する。

**根拠**:
- `NEXT_PUBLIC_` のない変数がクライアントコードに混入するとビルドエラー/機密漏洩になる
- 型なしの `process.env.XXX` アクセスは未定義時のランタイムエラーを招く
- Zod などでスキーマ検証することで起動時に設定ミスを検出できる

**コード例**:
```ts
// src/lib/env.ts - Zod で環境変数を型安全に管理
import { z } from 'zod';

// サーバーサイドのみの変数（クライアントバンドルに含まれない）
const serverSchema = z.object({
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  SENDGRID_API_KEY: z.string(),
  NODE_ENV: z.enum(['development', 'test', 'production']),
});

// クライアントに公開される変数（NEXT_PUBLIC_ プレフィックス必須）
const clientSchema = z.object({
  NEXT_PUBLIC_APP_URL: z.string().url(),
  NEXT_PUBLIC_GA_MEASUREMENT_ID: z.string().optional(),
});

// サーバー変数（サーバーコンポーネント・Server Actions でのみ使用可）
export const serverEnv = serverSchema.parse(process.env);

// クライアント変数（サーバー・クライアント両方で使用可）
export const clientEnv = clientSchema.parse({
  NEXT_PUBLIC_APP_URL: process.env.NEXT_PUBLIC_APP_URL,
  NEXT_PUBLIC_GA_MEASUREMENT_ID: process.env.NEXT_PUBLIC_GA_MEASUREMENT_ID,
});

// 使用例
// Server Component / Server Action
import { serverEnv } from '@/lib/env';
const db = createDatabaseClient(serverEnv.DATABASE_URL);

// Client Component
import { clientEnv } from '@/lib/env';
const appUrl = clientEnv.NEXT_PUBLIC_APP_URL;
```

**出典**:
- [Next.js Docs: Environment Variables](https://nextjs.org/docs/app/building-your-application/configuring/environment-variables) (Next.js公式)
- [t3-env](https://env.t3.gg/) (T3 Stack)

**バージョン**: Next.js 13+, Zod 3+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. `eslint-plugin-import` でモジュール境界違反を自動検出する

`eslint-plugin-import` と `eslint-import-resolver-typescript` を使い、
レイヤー間の不正な依存方向・循環インポートを CI で自動検出する。
`no-restricted-imports` ルールと組み合わせることで、feature 間の直接依存も Lint エラーにできる。

**根拠**:
- コードレビューで依存方向のチェックを人手で行うのはスケールしない
- `import/no-cycle` ルールが循環依存を早期に検出し、バンドルサイズの肥大化を防ぐ
- `no-restricted-imports` でプロジェクト固有のレイヤー規則を Lint ルールとして明文化できる

**コード例**:
```jsonc
// .eslintrc.json
{
  "plugins": ["import"],
  "settings": {
    "import/resolver": {
      "typescript": { "alwaysTryTypes": true }
    }
  },
  "rules": {
    // 循環インポートを禁止
    "import/no-cycle": ["error", { "maxDepth": 3 }],
    // デフォルトエクスポートより名前付きエクスポートを推奨
    "import/prefer-default-export": "off",
    "import/no-default-export": "off",
    // インポート順の統一
    "import/order": [
      "error",
      {
        "groups": ["builtin", "external", "internal", "parent", "sibling", "index"],
        "pathGroups": [
          { "pattern": "@/app/**", "group": "internal", "position": "before" },
          { "pattern": "@/features/**", "group": "internal" },
          { "pattern": "@/shared/**", "group": "internal", "position": "after" }
        ],
        "newlines-between": "always",
        "alphabetize": { "order": "asc" }
      }
    ]
  },
  "overrides": [
    {
      // entities 層: shared より上位への依存を禁止
      "files": ["src/entities/**/*.ts", "src/entities/**/*.tsx"],
      "rules": {
        "no-restricted-imports": [
          "error",
          {
            "patterns": [
              { "group": ["@/features/*"], "message": "entities から features への依存は禁止です" },
              { "group": ["@/pages/*"], "message": "entities から pages への依存は禁止です" }
            ]
          }
        ]
      }
    },
    {
      // features 層: 他 feature への直接依存を禁止
      "files": ["src/features/**/*.ts", "src/features/**/*.tsx"],
      "rules": {
        "no-restricted-imports": [
          "error",
          {
            "patterns": [
              { "group": ["@/pages/*"], "message": "features から pages への依存は禁止です" }
            ]
          }
        ]
      }
    }
  ]
}
```

**出典**:
- [eslint-plugin-import](https://github.com/import-js/eslint-plugin-import) (import-js / GitHub)
- [Feature-Sliced Design: Linting](https://feature-sliced.design/docs/guides/linting) (Feature-Sliced Design公式 / 2023)

**バージョン**: eslint-plugin-import 2.29+, ESLint 8+
**確信度**: 高
**最終更新**: 2026-05-06

---
