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
- OpenAPI codegen ツールの選択は「何を生成できるか」より「生成物をチームがどこまでメンテできるか」で決める。選定指針: (1) まず `openapi-typescript` で型のみ生成（~1.5 秒/75k 行スキーマ）→ (2) SDK 関数・統合引数・インターセプタ認証が必要になったら `hey-api`（~8.0 秒）→ (3) TanStack Query フック・Zod スキーマ・MSW モックまで欲しければ `Orval`（~5.5 秒）→ (4) ファイル単位の生成哲学が合う場合のみ `Kubb`（~18.1 秒）

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
- [OpenAPI codegen どれ選ぶん？ openapi-typescript / hey-api / Orval / Kubb 比較](https://zenn.dev/nyaomaru/articles/openapi-generator) (Zenn、75k行スキーマでの生成速度実測とツール選定フレームワーク) ※2026-05-20に実際にfetch成功

**バージョン**: openapi-typescript 7+, @graphql-codegen/cli 5+
**確信度**: 高
**最終更新**: 2026-05-20

---

### 2. Zod でAPIレスポンスをランタイム検証する（v4 はトップレベルバリデータ + 高速パース）

TypeScript の型は実行時に消えるため、外部APIのレスポンスを信用しない。
Zod でランタイム検証を行い、期待と異なるデータを早期に検出する。

**根拠**:
- TypeScript は型のランタイム保証を与えない（`as User` はコンパイル時のみ）
- APIのバックエンドが予期せずデータ形式を変更した場合にランタイムエラーが起きる
- Zod は型推論と組み合わさり、定義が1箇所で済む（TypeScript型 + ランタイム検証）
- `safeParse` はエラーをスローせずに Result 型で返し、エラー処理が明示的になる
- Zod v4 は文字列パースが約14倍、配列パースが約7倍高速化し、バンドルサイズも約2.3倍小さくなった。`z.string().email()` → `z.email()` のようにバリデータがトップレベル関数に昇格し、`error.flatten()` → `z.flattenError(result.error)` もグローバル関数に変更された
- `z.uuid()` は v4 で RFC 9562/4122 準拠の厳格な検証になった（非準拠UUIDを拒否する場合があるため、ゆるい検証が必要なら `z.guid()` を使う）
- `safeParse` の結果を型ガード関数でラップすると二重parseと `transform` 効果の損失が起きるため、結果はラップせず直接使う。外部データの検証はAPIクライアント層（アプリ境界）で1回だけ行い、以降は型安全として扱う
- v4 では `ZodError` の `.errors`（v3 の `.issues` の別名）が削除されたので `.issues` を使う。`z.string().uuid()` / `.datetime()` / `.url()` などのメソッド形式は非推奨で、`z.uuid()` / `z.iso.datetime()` / `z.url()` を使う
- 検証に失敗したレスポンスは未検証のデータであり、PII やトークンを含みうる。監視サービスには受信データ全文を送らず、issue のパス・コード・期待型だけを送る（PII スクラブの方針は `practices/observability/error-tracking.md` #3）

**コード例**:
```ts
// lib/schemas/user.ts
import { z } from 'zod';

export const UserSchema = z.object({
  id: z.uuid(),
  name: z.string().min(1).max(100),
  email: z.email(), // v4: z.string().email() から昇格
  role: z.enum(['admin', 'member', 'viewer']),
  createdAt: z.iso.datetime(),
  profile: z.object({
    avatarUrl: z.url().nullable(),
    bio: z.string().optional(),
  }).optional(),
});

// Zod スキーマから TypeScript 型を自動生成（重複定義不要）
export type User = z.infer<typeof UserSchema>;
```

```ts
// lib/api/users.ts
import * as Sentry from '@sentry/nextjs';
import { UserSchema, type User } from '@/lib/schemas/user';

export async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);

  const json: unknown = await res.json();

  // safeParse でエラーハンドリング
  const result = UserSchema.safeParse(json);
  if (!result.success) {
    // 受信データ（json）は送らず、どこが・何を期待して外れたかだけを送る
    Sentry.captureException(new Error('API schema mismatch'), {
      extra: {
        issues: result.error.issues.map((issue) => ({
          path: issue.path.map(String).join('.'),
          code: issue.code,
          expected: 'expected' in issue ? issue.expected : undefined,
        })),
      },
    });
    throw new Error('API レスポンスの形式が想定と異なります');
  }

  return result.data;
}

// Bad: レスポンスを型アサーションで素通し
export async function fetchUserUnsafe(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  return (await res.json()) as User;  // ランタイムで壊れる可能性がある
}
```

**出典**:
- [Zod Docs](https://zod.dev) (Zod公式)
- [Zod 4: Migration guide](https://zod.dev/v4/changelog) (Zod公式、`.errors` の削除と文字列フォーマットのトップレベル化) ※2026-10-25 確認
- [Zod: Defining schemas](https://zod.dev/api) (Zod公式、`z.uuid()` / `z.iso.datetime()` / `z.url()`) ※2026-10-25 確認
- [TypeScript 5.5の型ガードとZod v4で、APIレスポンスを安全に扱おう](https://zenn.dev/lv/articles/214f22bbc6df17) (Zenn lv、v4破壊的変更と推奨パターン) ※2026-05-06 fetch
- [ゼロから構築！Zod v4で叶える型安全なスキーマ駆動開発](https://zenn.dev/fd_ai_teacher/articles/tech-20260816020131-1) (Zenn fd_ai_teacher、パース速度・バンドルサイズの改善幅) ※2026-08-16 fetch

**出典引用**:
> "TypeScriptの型はコンパイル時にのみ存在し、ランタイムでは消滅します。"
> ([ゼロから構築！Zod v4で叶える型安全なスキーマ駆動開発](https://zenn.dev/fd_ai_teacher/articles/tech-20260816020131-1), セクション "型安全性の限界") ※2026-08-16に実際にfetch成功

**バージョン**: Zod 4+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 3. Valibot を軽量代替として採用できる

バンドルサイズが重要なプロジェクトでは Zod の代わりに Valibot を検討する。
モジュラーな設計によりツリーシェイキングが効き、使用した分だけバンドルされる。

**根拠**:
- Valibot はモジュラー設計で使用したバリデーターのみがバンドルに含まれる（tree-shakable）
- Zod（通常の `zod`）はメソッドチェーン設計のため、使っていないメソッドもバンドラーで除去しにくい。Zod 4 の `zod/mini`（関数型 API）を使うと差は縮む。Valibot 公式の比較（ログインフォームのスキーマ、esbuild）では Zod 約 17.7 kB、Zod Mini 約 6.88 kB、Valibot 1.37 kB
- API は Zod に類似しているため移行のコストが低い
- クライアントバンドルへの影響が大きいプロジェクトで特に有効

**コード例**:
```ts
// Valibot の使用例（v1: 検証は v.pipe() でつなぐ）
import * as v from 'valibot';

const UserSchema = v.object({
  id: v.pipe(v.string(), v.uuid()),
  name: v.pipe(v.string(), v.minLength(1), v.maxLength(100)),
  email: v.pipe(v.string(), v.email()),
  role: v.picklist(['admin', 'member', 'viewer']),
  createdAt: v.pipe(v.string(), v.isoTimestamp()),
});

type User = v.InferOutput<typeof UserSchema>;

export function parseUser(json: unknown): User {
  // safeParse（Zod と同じ使い勝手）
  const result = v.safeParse(UserSchema, json);
  if (!result.success) {
    console.error(result.issues);
    throw new Error('Invalid user data');
  }
  return result.output;
}
```

**出典**:
- [Valibot Docs](https://valibot.dev) (Valibot公式)
- [Valibot: Introduction](https://valibot.dev/guides/introduction/) (Valibot公式、`v.pipe()` と `v.InferOutput`) ※2026-10-25 確認
- [Valibot: Comparison](https://valibot.dev/guides/comparison/) (Valibot公式、Zod / Zod Mini とのバンドルサイズ比較) ※2026-10-25 確認

**バージョン**: Valibot 1.0+
**確信度**: 中
**最終更新**: 2026-10-25

---

### 4. 型生成を CI に組み込み、スキーマドリフトを防ぐ

型生成ファイルをコミットし、CI でスキーマとの一致を検証する。
生成ファイルの手動編集は行わない。

**根拠**:
- 生成ファイルへの手動変更は次の型生成で上書きされ、意図せず削除される
- CI でスキーマと生成ファイルの差分をチェックすることでドリフトを自動検出できる
- コミットに生成ファイルを含めることで PR レビューでの変化が見える
- 型生成パイプラインを持たない/持てないエンドポイントでは、`curl` + `jq` による軽量なレスポンス形状アサーション（HTTPステータス + フィールド型のみ）を CI の最前段に置くと、生成系に依存せず低コストで契約崩れを検知できる。GitHub Step Summary に checked endpoint / status / fixture を明記すると原因調査が速くなる

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
- [API Contract Drift: A Fast GitHub Actions Check](https://dev.to/pong1965/api-contract-drift-a-fast-github-actions-check-1glo) (dev.to、curl + jq による軽量レスポンス形状アサーションを CI 最前段に置く手法) ※2026-09-19 fetch
  > "A signup test may report an invalid email, while the actual regression is that the API stopped returning `verification_id`."

**バージョン**: openapi-typescript 7+, @graphql-codegen/cli 5+
**確信度**: 高
**最終更新**: 2026-09-19

---

### 5. openapi-typescript で OpenAPI spec から型を自動生成し、手動型定義を排除する

OpenAPI スペックファイル（JSON/YAML）から `openapi-typescript` を使って型を生成し、
手書きの interface/type 定義をゼロにする。
スキーマ駆動の方針とツール選定は #1、CI でのドリフト検出は #4、生成型を使うクライアントの作り方は `practices/api-client/rest.md` #3 を参照。

**根拠**:
- 手書き型定義はバックエンドの変更に追随できず、型と実データの乖離がサイレントに発生する
- `openapi-typescript` はパス・パラメータ・リクエストボディ・レスポンスの型を網羅的に生成する
- 生成型は `paths` オブジェクトに集約されており、`openapi-fetch` の `createClient<paths>` と組み合わせると全エンドポイントが型付きになる
- `$defs` や `allOf` / `oneOf` も正確に TypeScript の型へ変換される
- 生成型はコンパイル時の保証であり、実行時にレスポンスを検証するものではない。信頼できない API のレスポンスは #2 のスキーマ検証も併用する

**コード例**:
```bash
# インストール（devDependency）
npm install -D openapi-typescript

# リポジトリ内の OpenAPI ファイルから型を生成
npx openapi-typescript ./openapi.yaml -o src/types/api.d.ts

# package.json スクリプト化
# "generate:types": "openapi-typescript ./openapi.yaml -o src/types/api.d.ts"
```

```ts
// src/types/api.d.ts（生成物 - 手動編集禁止。説明用に簡略化）
export interface paths {
  '/users': {
    get: { responses: { 200: { content: { 'application/json': { users: User[] } } } } };
    post: { requestBody: { content: { 'application/json': CreateUserRequest } }; responses: { 201: { content: { 'application/json': User } } } };
  };
  '/users/{id}': {
    get: { parameters: { path: { id: string } }; responses: { 200: { content: { 'application/json': User } }; 404: never } };
  };
}
```

```ts
// lib/api-client.ts（クライアントの設定とエラー処理は rest.md #3）
import createClient from 'openapi-fetch';
import type { paths } from '@/types/api';

export const api = createClient<paths>({ baseUrl: '/api' });

// レスポンス型はスキーマの 200 レスポンスそのもの
const { data } = await api.GET('/users');
//      ^^^^ { users: User[] } | undefined（エラー時は undefined）
```

```ts
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
**最終更新**: 2026-10-25

---

### 6. Zod によるランタイムバリデーションと型推論を統合する（z.infer）

Zod スキーマを「型定義の唯一の正」として扱い、
`z.infer<typeof Schema>` で TypeScript 型を派生させる。
型定義とバリデーションロジックの二重管理を解消する。
`z.infer` による型の派生の基本は #2 で扱い、ここでは変換・部分スキーマ・フォームと API 境界での再利用を扱う。

**根拠**:
- `z.infer` を使うことでスキーマと型定義が常に同期され、手動での型更新が不要になる
- フォームバリデーション（React Hook Form + Zod）やAPIレスポンス検証に同一スキーマを再利用できる
- `z.discriminatedUnion` や `z.transform` を活用してレスポンスの正規化も型安全に行える
- Zod 4 の `.transform()` は `ZodPipe` を返し、object スキーマのメソッド（`.pick()` / `.omit()` など）を持たない。部分スキーマを派生させるときは、変換前の object スキーマを別に定義してそこから派生させる
- スキーマをファイル分割してフロント・バック（tRPC 等）で共有できる
- 「型が付いている」と「実行時に検証されている」を混同しない： `@hono/zod-openapi` の `responses[...].content['application/json'].schema` は OpenAPI ドキュメント生成と TypeScript の型推論にしか使われず、実行時には一切検証されない。`c.json(dbRow)` のように DB の生データをそのまま返すと、型上は正しく見えても余分なカラム（パスワードハッシュなど）が漏洩しうる
- 漏洩を防ぐ手段は、返すフィールドだけで DTO を明示的に組み立てるか、`schema.parse(data)` で未知のキーを取り除く（`z.object` は既定で未知のキーを strip する）かのどちらかに限る。`satisfies` は実行時に何も取り除かず、変数を渡したときは余剰プロパティチェックも働かないので、型の整合確認にしか使えない

**コード例**:
```ts
// lib/schemas/post.ts
import { z } from 'zod';

// 変換前の object スキーマを「型の正」として定義する（.pick() などはこちらから派生させる）
export const PostBaseSchema = z.object({
  id: z.uuid(),
  title: z.string().min(1).max(200),
  status: z.enum(['draft', 'published', 'archived']),
  publishedAt: z.iso.datetime().nullable(),
  tags: z.array(z.string()).default([]),
  author_id: z.uuid(),
});

// APIが snake_case を返す場合、transform で camelCase に変換
export const PostSchema = PostBaseSchema.transform(({ author_id, ...rest }) => ({
  ...rest,
  authorId: author_id,  // キー名を正規化
}));

// z.infer で型を自動導出（手書き interface 不要）
export type Post = z.infer<typeof PostSchema>;
// => { id: string; title: string; status: 'draft' | 'published' | 'archived';
//      publishedAt: string | null; tags: string[]; authorId: string }

// フォームバリデーション用に部分スキーマを派生（変換前のスキーマから）
export const CreatePostSchema = PostBaseSchema.pick({
  title: true,
  status: true,
  tags: true,
});
export type CreatePostInput = z.infer<typeof CreatePostSchema>;
```

```tsx
// components/create-post-form.tsx
'use client';

import type { z } from 'zod';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { CreatePostSchema } from '@/lib/schemas/post';

// .default() があると入力型と出力型が異なるため、両方を渡す
type CreatePostFormInput = z.input<typeof CreatePostSchema>;
type CreatePostFormOutput = z.output<typeof CreatePostSchema>;

export function CreatePostForm() {
  const form = useForm<CreatePostFormInput, unknown, CreatePostFormOutput>({
    resolver: zodResolver(CreatePostSchema),
    defaultValues: { title: '', status: 'draft', tags: [] },
  });

  return (
    <form onSubmit={form.handleSubmit((values) => console.log(values))}>
      <input {...form.register('title')} />
      <button type="submit">作成</button>
    </form>
  );
}
```

```ts
// lib/api/posts.ts
import { api } from '@/lib/api-client';
import { PostSchema, type Post } from '@/lib/schemas/post';

// APIレスポンス検証でも同じスキーマを再利用
export async function fetchPost(id: string): Promise<Post> {
  const { data, error } = await api.GET('/posts/{id}', { params: { path: { id } } });
  if (error) throw new Error('記事を取得できませんでした');
  return PostSchema.parse(data);  // ランタイム検証 + transform 適用
}
```

```ts
// lib/dto/user.ts — DB の行をレスポンスに出すときの漏洩対策
import { z } from 'zod';

const PublicUserSchema = z.object({ id: z.uuid(), name: z.string() });
type PublicUser = z.infer<typeof PublicUserSchema>;

type UserRow = { id: string; name: string; passwordHash: string };

// Good: 返すフィールドだけで DTO を明示的に組み立てる
export function toPublicUser(row: UserRow): PublicUser {
  return { id: row.id, name: row.name };
}

// Good: z.object は未知のキーを既定で取り除く（passwordHash は結果に残らない）
export function toPublicUserByParse(row: UserRow): PublicUser {
  return PublicUserSchema.parse(row);
}

// Bad: satisfies は型を確かめるだけ。変数には余剰プロパティチェックも働かず、passwordHash もそのまま返る
export function toPublicUserUnsafe(row: UserRow) {
  return row satisfies PublicUser;
}
```

```ts
// Bad: 型定義とバリデーションを別々に管理
import { z } from 'zod';

interface Post { title: string; status: string }  // Zod スキーマとズレが生じやすい
const titleSchema = z.string().min(1);              // 型と分離して管理
```

**出典引用**:
> "`responses`スキーマは**OpenAPIドキュメント生成とTypeScriptの型推論にしか使われておらず、ランタイムでは一切検証されていない**"
> ([hono/zod-openapiでレスポンススキーマを静的に検証する方法](https://qiita.com/natsugure/items/4b592c94752a473160e7), セクション "なぜ防げなかったか") ※2026-08-21に実際にfetch成功

**出典**:
- [Zod: Type Inference](https://zod.dev/?id=type-inference) (Zod公式)
- [Zod: Defining schemas](https://zod.dev/api) (Zod公式、`z.object` は既定で未知のキーを strip する) ※2026-10-25 確認
- [Zod 4: Migration guide](https://zod.dev/v4/changelog) (Zod公式、`.transform()` が `ZodPipe` を返す) ※2026-10-25 確認
- [React Hook Form: Zod Resolver](https://react-hook-form.com/docs/useform#resolver) (React Hook Form公式)
- [hono/zod-openapiでレスポンススキーマを静的に検証する方法](https://qiita.com/natsugure/items/4b592c94752a473160e7) (Qiita、`@hono/zod-openapi` のレスポンススキーマがランタイム未検証である実例と、静的な型チェックとしての `satisfies` / `typedJson()`) ※2026-08-21 fetch

**バージョン**: Zod 4+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 7. RPC スタイルの直接型共有（oRPC）でコード生成の間接コストを排除する

OpenAPI スペックや GraphQL スキーマを経由したコード生成ではなく、
バックエンドのルーター定義を TypeScript 型として直接フロントエンドに共有する
**oRPC** を採用することで、型の「ソース・オブ・トゥルース」を1ファイルに集約できる。

**根拠**:
- OpenAPI コード生成は「スキーマ更新 → 生成 → コミット → レビュー」のサイクルが必要で間接コストが高い
- 型の共有・配布方法で選ぶ。tRPC はサーバーのルーター型をクライアントが import する方式で、モノレポは必須ではないが、別リポジトリではバックエンドの型をプライベートパッケージとして配布することになり、クライアントとサーバーの整合の保証も弱まる（tRPC 公式 FAQ）。oRPC の contract-first では、スキーマ付きの contract をサーバー実装から独立した共有パッケージにでき、サーバーは `implement(contract)` で実装し、クライアントは contract の型だけを参照する
- バックエンドの `contract` オブジェクトをそのまま型として参照するため、型の二重管理がなくなる
- TanStack Query 向けの `@orpc/tanstack-query` が `queryOptions` / `mutationOptions` を生成するので、既存の `useQuery` / `useMutation` にそのまま渡せる

**コード例**:
```ts
// === 共有パッケージ ===
// shared/contract.ts
import { oc } from '@orpc/contract';
import { z } from 'zod';

const UserSchema = z.object({
  id: z.string(),
  name: z.string(),
  email: z.email(),
});

// contract は普通のオブジェクトでネストする
export const contract = {
  user: {
    find: oc
      .input(UserSchema.pick({ id: true }))
      .output(UserSchema),
    list: oc
      .output(z.array(UserSchema)),
  },
};
// サーバー側は `import { implement } from '@orpc/server'` の implement(contract) で実装する
```

```ts
// === フロントエンド ===
// lib/orpc-client.ts
import { createORPCClient } from '@orpc/client';
import { RPCLink } from '@orpc/client/fetch';
import type { ContractRouterClient } from '@orpc/contract';
import { createTanstackQueryUtils } from '@orpc/tanstack-query';
import type { contract } from 'shared/contract';

const link = new RPCLink({
  url: `${typeof window !== 'undefined' ? window.location.origin : 'http://localhost:3000'}/rpc`,
});

const client: ContractRouterClient<typeof contract> = createORPCClient(link);

export const orpc = createTanstackQueryUtils(client);
```

```tsx
// components/user-profile.tsx（QueryClientProvider の内側で使う）
'use client';

import { useQuery } from '@tanstack/react-query';
import { orpc } from '@/lib/orpc-client';

export function UserProfile({ id }: { id: string }) {
  const { data: user } = useQuery(orpc.user.find.queryOptions({ input: { id } }));

  return <div>{user?.name}</div>;
}
```

**出典**:
- [oRPC でコード生成なしの型安全API通信を実現する](https://zenn.dev/dress_code/articles/9040b2e3532693) (Zenn dress_code / 2026-05)
- [oRPC v1: Define Contract](https://v1.orpc.dev/docs/contract-first/define-contract) (oRPC公式) ※2026-10-25 確認
- [oRPC v1: Implement Contract](https://v1.orpc.dev/docs/contract-first/implement-contract) (oRPC公式、`implement(contract)`) ※2026-10-25 確認
- [oRPC v1: Client-Side Clients](https://v1.orpc.dev/docs/client/client-side) (oRPC公式、`createORPCClient` / `RPCLink` / `ContractRouterClient`) ※2026-10-25 確認
- [oRPC v1: TanStack Query](https://v1.orpc.dev/docs/integrations/tanstack-query) (oRPC公式、`createTanstackQueryUtils` / `queryOptions`) ※2026-10-25 確認
- [tRPC FAQ](https://trpc.io/docs/faq) (tRPC公式、"a monorepo is not mandatory") ※2026-10-25 確認

**バージョン**: @orpc/contract 1.x, @orpc/client 1.x, @orpc/tanstack-query 1.x（2026-10 時点の npm latest は 1.16.1。2.0 は beta）
**確信度**: 中
**最終更新**: 2026-10-25

---

## 関連プラクティス

- [`api-client/rest.md`](./rest.md) - openapi-fetch による型安全な REST クライアント
- [`api-client/graphql.md`](./graphql.md) - GraphQL Code Generator
- [`api-client/grpc.md`](./grpc.md) - Protocol Buffers 型生成（Buf）
- [`typescript/generics.md`](../typescript/generics.md) - TypeScript ジェネリクス活用
