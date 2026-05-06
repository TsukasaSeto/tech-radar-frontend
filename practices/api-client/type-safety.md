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
- [openapi-typescript: CLI Integration](https://openapi-ts.dev/cli) (openapi-ts.dev)

**バージョン**: openapi-typescript 7+, @graphql-codegen/cli 5+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 5. openapi-typescript で OpenAPI spec から型を自動生成し、手動型定義を排除する

OpenAPI スペックファイル（JSON/YAML）から `openapi-typescript` を使って型を生成し、
手書きの interface/type 定義をゼロにする。
CI でスペックとの整合を確認し、型の陳腐化を防ぐ。

**根拠**:
- 手書き型定義はバックエンドの変更に追随できず、型と実データの乖離がサイレントに発生する
- `openapi-typescript` はパス・パラメータ・リクエストボディ・レスポンスの型を網羅的に生成する
- 生成型は `paths` オブジェクトに集約されており、`openapi-fetch` の `createClient<paths>` と組み合わせると全エンドポイントが型付きになる
- `$defs` や `allOf` / `oneOf` も正確に TypeScript の型へ変換される

**コード例**:
```bash
# インストール
npm install -D openapi-typescript

# ローカルの OpenAPI ファイルから型を生成
npx openapi-typescript ./openapi.yaml -o src/types/api.d.ts

# リモート URL から直接生成
npx openapi-typescript https://api.example.com/openapi.json -o src/types/api.d.ts

# package.json スクリプト化
# "generate:types": "openapi-typescript ./openapi.yaml -o src/types/api.d.ts"
```

```ts
// src/types/api.d.ts（生成物 - 手動編集禁止）
export interface paths {
  '/users': {
    get: { responses: { 200: { content: { 'application/json': { users: User[] } } } } };
    post: { requestBody: { content: { 'application/json': CreateUserRequest } }; responses: { 201: { content: { 'application/json': User } } } };
  };
  '/users/{id}': {
    get: { parameters: { path: { id: string } }; responses: { 200: { content: { 'application/json': User } }; 404: never } };
  };
}

// lib/api-client.ts - 生成型を使って型安全なクライアントを構築
import createClient from 'openapi-fetch';
import type { paths } from '@/types/api';

export const api = createClient<paths>({ baseUrl: '/api' });

// 全エンドポイントが型付き
const { data: users } = await api.GET('/users');
//            ^^^^^ User[] | undefined

const { data: newUser } = await api.POST('/users', {
  body: { name: 'Alice', email: 'alice@example.com' },
  //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  //     CreateUserRequest に一致しないとコンパイルエラー
});

// Bad: 手書き型定義（バックエンド変更時に陳腐化する）
interface User {        // APIと乖離しても TypeScript は気づかない
  id: string;
  name: string;
  // email フィールドが追加されたがここに反映されていない
}
```

**出典**:
- [openapi-typescript Docs](https://openapi-ts.dev/introduction) (openapi-ts.dev)
- [openapi-fetch Docs](https://openapi-ts.dev/openapi-fetch/) (openapi-ts.dev)

**バージョン**: openapi-typescript 7+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 6. Zod によるランタイムバリデーションと型推論を統合する（z.infer）

Zod スキーマを「型定義の唯一の正」として扱い、
`z.infer<typeof Schema>` で TypeScript 型を派生させる。
型定義とバリデーションロジックの二重管理を解消する。

**根拠**:
- `z.infer` を使うことでスキーマと型定義が常に同期され、手動での型更新が不要になる
- フォームバリデーション（React Hook Form + Zod）やAPIレスポンス検証に同一スキーマを再利用できる
- `z.discriminatedUnion` や `z.transform` を活用してレスポンスの正規化も型安全に行える
- スキーマをファイル分割してフロント・バック（tRPC 等）で共有できる

**コード例**:
```ts
// lib/schemas/post.ts
import { z } from 'zod';

// スキーマがすべての「型の正」
export const PostSchema = z.object({
  id: z.string().uuid(),
  title: z.string().min(1).max(200),
  status: z.enum(['draft', 'published', 'archived']),
  publishedAt: z.string().datetime().nullable(),
  tags: z.array(z.string()).default([]),
  // APIが snake_case を返す場合、transform で camelCase に変換
  author_id: z.string().uuid(),
}).transform(data => ({
  ...data,
  authorId: data.author_id,  // キー名を正規化
}));

// z.infer で型を自動導出（手書き interface 不要）
export type Post = z.infer<typeof PostSchema>;
// => { id: string; title: string; status: 'draft' | 'published' | 'archived';
//      publishedAt: string | null; tags: string[]; author_id: string; authorId: string }

// フォームバリデーション用に部分スキーマを派生
export const CreatePostSchema = PostSchema.innerType().pick({
  title: true,
  status: true,
  tags: true,
});
export type CreatePostInput = z.infer<typeof CreatePostSchema>;

// React Hook Form との統合
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';

function CreatePostForm() {
  const form = useForm<CreatePostInput>({
    resolver: zodResolver(CreatePostSchema),
    defaultValues: { status: 'draft', tags: [] },
  });
  // フォームの型とバリデーションが CreatePostSchema から自動導出
}

// APIレスポンス検証でも同じスキーマを再利用
async function fetchPost(id: string): Promise<Post> {
  const json = await api.GET('/posts/{id}', { params: { path: { id } } });
  return PostSchema.parse(json.data);  // ランタイム検証 + transform 適用
}

// Bad: 型定義とバリデーションを別々に管理
interface Post { title: string; status: string }  // Zod スキーマとズレが生じやすい
const titleSchema = z.string().min(1);              // 型と分離して管理
```

**出典**:
- [Zod: Type Inference](https://zod.dev/?id=type-inference) (Zod公式)
- [React Hook Form: Zod Resolver](https://react-hook-form.com/docs/useform#resolver) (React Hook Form公式)

**バージョン**: Zod 3+
**確信度**: 高
**最終更新**: 2026-05-06

---

## 関連プラクティス

- [`api-client/rest.md`](./rest.md) - openapi-fetch による型安全な REST クライアント
- [`api-client/graphql.md`](./graphql.md) - GraphQL Code Generator
- [`api-client/grpc.md`](./grpc.md) - Protocol Buffers 型生成（Buf）
- [`typescript/generics.md`](../typescript/generics.md) - TypeScript ジェネリクス活用
