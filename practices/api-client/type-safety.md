# API 型安全性のベストプラクティス

## ルール

### 1. スキーマ駆動開発でAPIの型をコードから分離する

APIの型定義を手書きせず、OpenAPI スキーマや GraphQL スキーマ、
Protocol Buffers 定義から自動生成する。

**根拠**:
- 手書きの型定義はバックエンドのAPI仕様と乖離しやすく、ランタイムエラーの原因になる
- スキーマ駆動ならCI でフロントエンドとバックエンドの整合性を検証できる
- 型生成ツールはスキーマ変更時に TypeScript エラーで変更箇所を教えてくれる
- チーム間での「型の正」が単一のスキーマファイルに集約される

**コード例**:
```bash
# OpenAPI スキーマから型を生成
npx openapi-typescript https://api.example.com/openapi.json -o src/types/api.d.ts

# CI でスキーマのドリフトを検出する例
# .github/workflows/type-check.yml
# - run: npx openapi-typescript $API_URL -o /tmp/api-new.d.ts
# - run: diff src/types/api.d.ts /tmp/api-new.d.ts

# GraphQL スキーマから型を生成
npx graphql-codegen --config codegen.ts

# Protocol Buffers から型を生成
npx buf generate
```

```ts
// 生成された型を使用（手書きしない）
import type { paths } from '@/types/api';  // openapi-typescript が生成

type User = paths['/users/{id}']['get']['responses']['200']['content']['application/json'];
type CreateUserBody = paths['/users']['post']['requestBody']['content']['application/json'];

// createClient で全てのAPIコールが型付きになる
import createClient from 'openapi-fetch';
const client = createClient<paths>({ baseUrl: '/api' });

const { data } = await client.GET('/users/{id}', {
  params: { path: { id: '123' } },
});
// data は自動的に User 型 | undefined
```

**出典**:
- [openapi-typescript Docs](https://openapi-ts.dev/introduction) (openapi-ts.dev)
- [GraphQL Code Generator Docs](https://the-guild.dev/graphql/codegen/docs/getting-started) (The Guild公式)

**バージョン**: openapi-typescript 7+, @graphql-codegen/cli 5+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. Zod でAPIレスポンスをランタイム検証する

TypeScript の型は実行時に消えるため、外部APIのレスポンスを信用しない。
Zod でランタイム検証を行い、期待と異なるデータを早期に検出する。

**根拠**:
- TypeScript は型のランタイム保証を与えない（`as User` はコンパイル時のみ）
- APIのバックエンドが予期せずデータ形式を変更した場合にランタイムエラーが起きる
- Zod は型推論と組み合わさり、定義が1箇所で済む（TypeScript型 + ランタイム検証）
- `safeParse` はエラーをスローせずに Result 型で返し、エラー処理が明示的になる

**コード例**:
```ts
// lib/schemas/user.ts
import { z } from 'zod';

export const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(100),
  email: z.string().email(),
  role: z.enum(['admin', 'member', 'viewer']),
  createdAt: z.string().datetime(),
  profile: z.object({
    avatarUrl: z.string().url().nullable(),
    bio: z.string().optional(),
  }).optional(),
});

// Zod スキーマから TypeScript 型を自動生成（重複定義不要）
export type User = z.infer<typeof UserSchema>;

// APIレスポンスの検証
async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);

  const json = await res.json();

  // safeParse でエラーハンドリング
  const result = UserSchema.safeParse(json);
  if (!result.success) {
    // スキーマ不一致をエラー監視サービスに送信
    captureError(new Error('API schema mismatch'), {
      validationErrors: result.error.errors,
      received: json,
    });
    throw new Error('API レスポンスの形式が想定と異なります');
  }

  return result.data;
}

// Bad: レスポンスを型アサーションで素通し
const user = await res.json() as User;  // ランタイムで壊れる可能性がある
```

**出典**:
- [Zod Docs](https://zod.dev) (Zod公式)

**バージョン**: Zod 3+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. Valibot を軽量代替として採用できる

バンドルサイズが重要なプロジェクトでは Zod の代わりに Valibot を検討する。
モジュラーな設計によりツリーシェイキングが効き、使用した分だけバンドルされる。

**根拠**:
- Valibot はモジュラー設計で使用したバリデーターのみがバンドルに含まれる（tree-shakable）
- Zod は全体が1モジュールのため、使用量に関わらず一定のバンドルサイズがかかる
- API は Zod に類似しているため移行のコストが低い
- クライアントバンドルへの影響が大きいプロジェクトで特に有効

**コード例**:
```ts
// Valibot の使用例
import * as v from 'valibot';

const UserSchema = v.object({
  id: v.string([v.uuid()]),
  name: v.string([v.minLength(1), v.maxLength(100)]),
  email: v.string([v.email()]),
  role: v.picklist(['admin', 'member', 'viewer']),
  createdAt: v.string([v.isoTimestamp()]),
});

type User = v.Output<typeof UserSchema>;

// safeParse（Zod と同じ使い勝手）
const result = v.safeParse(UserSchema, json);
if (!result.success) {
  console.error(result.issues);
  throw new Error('Invalid user data');
}
const user = result.output;

// Zod との比較
// Zod:    ~14KB gzip（全機能）
// Valibot: ~2KB gzip（使用部分のみ）
```

**出典**:
- [Valibot Docs](https://valibot.dev) (Valibot公式)

**バージョン**: Valibot 0.31+
**確信度**: 中
**最終更新**: 2026-05-05

---

### 4. 型生成を CI に組み込み、スキーマドリフトを防ぐ

型生成ファイルをコミットし、CI でスキーマとの一致を検証する。
生成ファイルの手動編集は行わない。

**根拠**:
- 生成ファイルへの手動変更は次の型生成で上書きされ、意図せず削除される
- CI でスキーマと生成ファイルの差分をチェックすることでドリフトを自動検出できる
- コミットに生成ファイルを含めることで PR レビューでの変化が見える

**コード例**:
```yaml
# .github/workflows/type-check.yml
name: Type Check

on: [push, pull_request]

jobs:
  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      # OpenAPI の型生成とドリフトチェック
      - name: Generate OpenAPI types
        run: npx openapi-typescript ${{ secrets.API_URL }}/openapi.json -o /tmp/api-new.d.ts

      - name: Check for schema drift
        run: |
          if ! diff -q src/types/api.d.ts /tmp/api-new.d.ts; then
            echo "::error::API schema has changed. Run 'npm run generate:types' and commit the result."
            diff src/types/api.d.ts /tmp/api-new.d.ts
            exit 1
          fi

      # GraphQL の型生成とドリフトチェック
      - name: Generate GraphQL types
        run: npx graphql-codegen --config codegen.ts

      - name: Check GraphQL drift
        run: |
          git diff --exit-code src/gql/ || {
            echo "::error::GraphQL types are out of sync. Run 'npm run codegen' and commit."
            exit 1
          }

      - name: TypeScript check
        run: npx tsc --noEmit
```

```json
// package.json
{
  "scripts": {
    "generate:types": "openapi-typescript $npm_config_api_url -o src/types/api.d.ts",
    "codegen": "graphql-codegen --config codegen.ts",
    "typecheck": "tsc --noEmit"
  }
}
```

**出典**:
- [openapi-typescript: CI Integration](https://openapi-ts.dev/cli) (openapi-ts.dev)

**バージョン**: openapi-typescript 7+, @graphql-codegen/cli 5+
**確信度**: 高
**最終更新**: 2026-05-05

---

## 関連プラクティス

- [`api-client/rest.md`](./rest.md) - openapi-fetch による型安全な REST クライアント
- [`api-client/graphql.md`](./graphql.md) - GraphQL Code Generator
- [`api-client/grpc.md`](./grpc.md) - Protocol Buffers 型生成（Buf）
- [`typescript/generics.md`](../typescript/generics.md) - TypeScript ジェネリクス活用
