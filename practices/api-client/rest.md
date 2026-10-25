# REST API クライアントのベストプラクティス

## ルール

### 1. fetch は AbortController でキャンセル可能にする

コンポーネントのアンマウントやタイムアウト時にリクエストをキャンセルするため、
AbortController を必ず組み合わせる。
キャンセルしない fetch はメモリリークやゾンビ状態更新の原因となる。

**根拠**:
- ユーザーが画面を離れた後もリクエストが完了し、アンマウント済みコンポーネントの state を更新しようとする
- Next.js の Route Handler / Server Actions では `request.signal` を fetch に渡すことで上流キャンセルに対応できる
- ブラウザの AbortController は全モダンブラウザでサポートされている

**コード例**:
```tsx
// Good: AbortController でキャンセル対応
async function fetchUser(id: string, signal?: AbortSignal): Promise<User> {
  const res = await fetch(`/api/users/${id}`, { signal });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json() as Promise<User>;
}

// React hook での使用例
function useUser(id: string) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    fetchUser(id, controller.signal)
      .then(setUser)
      .catch(err => {
        if (err.name !== 'AbortError') console.error(err);
      });

    return () => controller.abort();
  }, [id]);

  return user;
}

// Next.js Route Handler での上流キャンセル伝播
export async function GET(request: Request) {
  const data = await fetch('https://api.example.com/data', {
    signal: request.signal,  // クライアント切断時に上流も中止
  });
  return Response.json(await data.json());
}

// Bad: キャンセルなし
useEffect(() => {
  fetchUser(id).then(setUser);  // アンマウント後も実行される
}, [id]);
```

**出典**:
- [MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) (MDN Web Docs)
- [Next.js Docs: Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers) (Next.js公式)

**バージョン**: 全モダンブラウザ, Node.js 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. 素の fetch より ky を使い、タイムアウト・リトライ・エラー処理を統一する

素の fetch にはタイムアウトも非2xx エラーの自動スロー機能もない。
ky を使うことでこれらを設定1回で統一できる。

**根拠**:
- fetch は HTTP 4xx/5xx を例外として扱わず、`ok` フラグの手動チェックが必要
- タイムアウト実装は AbortController + setTimeout の組み合わせが必要で複雑
- ky はフック機構（beforeRequest/afterResponse/beforeRetry）でインターセプト処理を統一できる
- `.json()` の戻り値が `any` ではなく `unknown` でありより型安全

**コード例**:
```ts
// lib/http.ts
import ky, { isHTTPError } from 'ky';
import { getAuthToken } from '@/lib/auth';

export const http = ky.create({
  prefix: process.env.NEXT_PUBLIC_API_BASE_URL,  // ky 1.x では prefixUrl
  timeout: 10_000,        // 10秒タイムアウト（per-attempt）
  retry: {
    limit: 2,             // 最大2回リトライ
    methods: ['get'],     // GETのみリトライ
    statusCodes: [408, 429, 500, 502, 503, 504],
  },
  hooks: {
    beforeRequest: [
      ({ request }) => {
        const token = getAuthToken();
        if (token) request.headers.set('Authorization', `Bearer ${token}`);
      },
    ],
    // 401 のトークンリフレッシュは #4 の afterResponse で行う
    beforeError: [
      ({ error }) => {
        // HTTPError なら、解析済みのレスポンスボディが error.data に入っている
        if (
          isHTTPError(error)
          && typeof error.data === 'object'
          && error.data !== null
          && 'message' in error.data
        ) {
          error.message = String(error.data.message);
        }
        return error;
      },
    ],
  },
});

// 使用例
const user = await http.get('users/123').json<User>();
const created = await http.post('users', { json: { name: 'Alice' } }).json<User>();

// Bad: 素の fetch での冗長な実装
const res = await fetch('/api/users/123');
if (!res.ok) throw new Error('Request failed');  // 手動チェック必須
const user = await res.json();  // 型が any
```

**出典**:
- [ky README](https://github.com/sindresorhus/ky) (sindresorhus/ky / GitHub、`prefix` オプション・フックが受け取る state オブジェクト・`isHTTPError` と `error.data`) ※2026-10-25 確認（ky 2.2.0 の README）

**バージョン**: ky 2.0+（1.x では `prefixUrl` で、`beforeRequest` / `afterResponse` / `beforeError` は state オブジェクトではなく位置引数を受け取る）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 3. openapi-fetch でスキーマ駆動の型安全な REST クライアントを構築する

OpenAPI スキーマから `openapi-typescript` で型を生成し、
`openapi-fetch` で型安全なクライアントを作る。型による保証はコンパイル時のもので、実行時の検証は別に行う。

**根拠**:
- API 仕様とクライアントコードの乖離をコンパイル時に検出できる
- パスパラメータ・クエリパラメータ・リクエストボディ・レスポンスがすべて型付けされる
- 生成した型が保証するのはコンパイル時だけで、openapi-fetch は実行時にレスポンスを検証しない。スキーマとずれたレスポンスや外部 API のレスポンスを信頼できない場合は、Zod などで実行時にも検証する（`practices/api-client/type-safety.md` #2）
- バックエンドの型変更をフロントエンドが即座に検知できる
- openapi-fetch は 4xx / 5xx でも reject せず、`{ data, error, response }` を返す。TanStack Query の `queryFn` / `mutationFn` に渡すときは `error` を throw に変換する。変換しないと失敗が成功として扱われ、`retry` や `onError` のロールバックが動かない

**コード例**:
```bash
# スキーマから型を生成
npx openapi-typescript https://api.example.com/openapi.json -o src/types/api.d.ts
```

```ts
// lib/api-client.ts
import createClient from 'openapi-fetch';
import { ApiError } from '@/lib/api-error';  // error-handling.md #1
import type { paths } from '@/types/api';

export const apiClient = createClient<paths>({
  baseUrl: process.env.NEXT_PUBLIC_API_BASE_URL,
  headers: { 'Content-Type': 'application/json' },
});

// TanStack Query の queryFn / mutationFn に渡す関数では、error を throw に変換する
export async function getUser(id: string) {
  const { data, error, response } = await apiClient.GET('/users/{id}', {
    params: { path: { id } },
  });
  if (error) throw new ApiError(response.status, 'API_ERROR', `HTTP ${response.status}`);
  return data;
}

// 使用例: パス・パラメータ・レスポンスが全て型付き
const { data, error } = await apiClient.GET('/users/{id}', {
  params: { path: { id: '123' } },
});

if (error) {
  // error の型は API スキーマのエラーレスポンス
  console.error(error.message);
} else {
  // data の型は API スキーマの成功レスポンス
  console.log(data.name);
}

// POST も型安全
const { data: newUser } = await apiClient.POST('/users', {
  body: { name: 'Alice', email: 'alice@example.com' },
  // body の型が OpenAPI スキーマの requestBody と一致しないとコンパイルエラー
});

// Bad: 型なし fetch
const res = await fetch(`/api/users/${id}`);
const user = await res.json();  // any 型
```

**出典**:
- [openapi-fetch Docs](https://openapi-ts.dev/openapi-fetch/) (openapi-ts.dev、各メソッドは `data` / `error` / `response` を返し、`error` は 4xx / 5xx のときに入る) ※2026-10-25 確認
- [openapi-typescript Docs](https://openapi-ts.dev/introduction) (openapi-ts.dev)

**バージョン**: openapi-fetch 0.12+, openapi-typescript 7+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 4. 認証ヘッダーの付与はインターセプターで一元化する

各 fetch 呼び出しに認証ヘッダーを手動で付与するのではなく、
インターセプター（ky の beforeRequest フック等）で一元管理する。

**根拠**:
- 認証ロジックの変更が1箇所で完結する
- トークンリフレッシュは `afterResponse` フックに集約し、401 を検知したらリフレッシュして新しいトークンで 1 回だけ再送する（`ky.retry()` を返す）
- `beforeRetry` ではリフレッシュできない。401 は既定の `retry.statusCodes` に含まれないので再試行自体が起きず、`beforeRetry` の `retryCount` は常に 1 以上なので `retryCount === 0` の分岐も成立しない
- 個々のAPIコールがアプリケーションロジックに専念できる

**コード例**:
```ts
// lib/http.ts
import ky from 'ky';
import { getSession } from 'next-auth/react';
import { refreshAccessToken } from '@/lib/auth';

export const authenticatedHttp = ky.create({
  prefix: process.env.NEXT_PUBLIC_API_BASE_URL,  // ky 1.x では prefixUrl
  hooks: {
    beforeRequest: [
      async ({ request }) => {
        const session = await getSession();
        if (session?.accessToken) {
          request.headers.set('Authorization', `Bearer ${session.accessToken}`);
        }
      },
    ],
    afterResponse: [
      // 最初の 401 だけリフレッシュし、新しいトークンで 1 回だけ再送する
      async ({ request, response, retryCount }) => {
        if (response.status !== 401 || retryCount > 0) return;
        const newToken = await refreshAccessToken();
        if (!newToken) return;  // リフレッシュに失敗したら 401 のまま HTTPError になる

        const headers = new Headers(request.headers);
        headers.set('Authorization', `Bearer ${newToken}`);
        return ky.retry({ request: new Request(request, { headers }), code: 'TOKEN_REFRESHED' });
      },
    ],
  },
});

// Bad: 各呼び出しで手動付与
const res = await fetch('/api/users', {
  headers: { Authorization: `Bearer ${token}` },  // 毎回書く必要がある
});
```

**出典**:
- [ky README: Hooks](https://github.com/sindresorhus/ky#hooks) (sindresorhus/ky / GitHub、`afterResponse` から `ky.retry()` で 401 を再送する例・`beforeRetry` の `retryCount` は常に 1 以上) ※2026-10-25 確認（ky 2.2.0 の README）

**バージョン**: ky 2.0+（1.x では `prefixUrl` で、`afterResponse` は `(request, options, response, state)` の位置引数）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 5. Server Components / Route Handlers での fetch は next の拡張オプションを活用する

Next.js の fetch 拡張オプション（`cache`, `next.revalidate`, `next.tags`）を使い、
サーバーサイドのデータ取得を最適化する。

**根拠**:
- Next.js は fetch を拡張しており、ISR・キャッシュ制御がオプション1つで設定できる
- `next.tags` でタグベースの revalidation が可能（Server Action では `updateTag(tag)`、Route Handler では `revalidateTag(tag, 'max')` と組み合わせる。1 引数の `revalidateTag(tag)` は v16 で非推奨）
- データ取得関数を複数の Server Component で呼んでも重複リクエストは自動で排除される（Request Memoization）
- 更新側の Server Action は公開エンドポイントとして扱い、認証・認可・入力検証を Action ごとに行う（`practices/nextjs/server-actions.md` #5）。`'use server'` はファイルか関数本体の先頭に書かないと効かないので、Action は `'use server'` 専用のファイルに分ける

**コード例**:
```ts
// app/_lib/dal/products.ts（DAL の配置は practices/architecture/data-access-layer.md #1）

// 60秒ごとに再検証する（ISR）
export async function getProducts() {
  const res = await fetch('https://api.example.com/products', {
    next: { revalidate: 60, tags: ['products'] },
  });
  return res.json() as Promise<Product[]>;
}

// キャッシュしない（常に最新データ）
export async function getDashboardData() {
  const res = await fetch('https://api.example.com/dashboard', {
    cache: 'no-store',
  });
  return res.json();
}
```

```ts
// app/actions/products.ts - Server Action でキャッシュを無効化
'use server';

import { updateTag } from 'next/cache';
import { z } from 'zod';
// 未認証・権限不足ならスローする自前ヘルパー（unauthorized() / forbidden() を使うなら experimental.authInterrupts: true が前提）
import { requireAuth, requirePermission } from '@/lib/auth';

const ProductUpdateInput = z.object({
  id: z.uuid(),
  name: z.string().min(1).max(100),
  price: z.number().int().nonnegative(),
});

export async function updateProduct(raw: unknown): Promise<void> {
  const user = await requireAuth();                        // ① 認証
  await requirePermission(user, 'product:update');         // ② 認可（最終判定は API / DAL 側でも行う）
  const { id, ...data } = ProductUpdateInput.parse(raw);   // ③ 許可したフィールドだけを受け付ける

  const res = await fetch(`https://api.example.com/products/${id}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${user.accessToken}`,
    },
    body: JSON.stringify(data),
    cache: 'no-store',
  });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);

  updateTag('products');  // 'products' タグのキャッシュをすぐに失効させ、自分の更新を次の表示に反映する
}
```

**出典**:
- [Next.js Docs: fetch](https://nextjs.org/docs/app/api-reference/functions/fetch) (Next.js公式)
- [Next.js Docs: Caching](https://nextjs.org/docs/app/building-your-application/caching) (Next.js公式)
- [Next.js Docs: updateTag](https://nextjs.org/docs/app/api-reference/functions/updateTag) (Next.js公式、Server Action 専用で read-your-own-writes 向け) ※2026-10-25 確認
- [Next.js Docs: revalidateTag](https://nextjs.org/docs/app/api-reference/functions/revalidateTag) (Next.js公式、1 引数の形式は非推奨・推奨プロファイルは `'max'`) ※2026-10-25 確認
- [Next.js Docs: use server](https://nextjs.org/docs/app/api-reference/directives/use-server) (Next.js公式、ファイル先頭か関数本体の先頭に置く・Server Function 内で認証と認可を行う) ※2026-10-25 確認

**バージョン**: Next.js 16+（`updateTag` と `revalidateTag(tag, profile)`。v15 以前は `revalidateTag(tag)` の 1 引数）、Zod 4+（`z.uuid()`）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 6. TanStack Query の Query Key をファクトリパターンで管理する

クエリキーのファクトリの詳細（構造・使い方・無効化の範囲）は `practices/api-client/caching.md` #3 を参照。
ここでは REST API で使うときの観点だけを扱う。

**根拠**:
- REST のリソース（`/users`、`/posts` など）ごとにファクトリを 1 つ作り（`userKeys` / `postKeys`）、`all → lists → list(filters) → details → detail(id)` の階層をすべてのリソースでそろえる。どのリソースでも「一覧だけ」「1 件だけ」の無効化を同じ書き方にできる

**出典**:
- [Effective React Query Keys](https://tkdodo.eu/blog/effective-react-query-keys) (TkDodo's Blog、TanStack Query のメンテナー。機能ごとのファクトリと `all` / `lists` / `list` / `details` / `detail` の階層) ※2026-10-25 確認

**バージョン**: TanStack Query 5+
**確信度**: 中
**最終更新**: 2026-10-25

---

### 7. Optimistic Update は useMutation の onMutate / onError / onSettled で実装する

3 つのコールバック（`onMutate` / `onError` / `onSettled`）の役割と実装例の詳細は `practices/api-client/caching.md` #4 を参照。
ここでは REST の 1 件のリソースを更新するときの観点だけを扱う。

**根拠**:
- 1 件のリソースを更新するときは、クエリキーのファクトリ（`practices/api-client/caching.md` #3）の `detail(id)` キーだけを `cancelQueries` / `setQueryData` / `invalidateQueries` の対象にし、関係のないキャッシュを巻き込まない
- openapi-fetch で `mutationFn` を書くときは、`error` を throw に変換してから返す（#3）。変換しないと 4xx / 5xx でも `onError` のロールバックが走らない
- 送信中はボタンを `isPending` で無効にし、楽観的更新が重なる連打を防ぐ
- `onSuccess` で `invalidateQueries` するだけの実装は、サーバーの応答まで UI が変わらない。応答を待ってよい操作ならそれで十分で、楽観的更新が必要なのは即時のフィードバックが重要な操作に限る

**コード例**:
```tsx
// features/posts/components/LikeButton.tsx
'use client';
import { useLikePost } from '@/features/posts/hooks/use-like-post'; // caching.md #4 と同じ構成で、キーに postKeys.detail(postId) を使う
import type { Post } from '@/features/posts/types';

export function LikeButton({ post }: { post: Post }) {
  const { mutate: likePost, isPending } = useLikePost();
  return (
    <button onClick={() => likePost(post.id)} disabled={isPending}>
      {post.isLiked ? '❤️' : '🤍'} {post.likeCount}
    </button>
  );
}
```

**出典**:
- [TanStack Query: Optimistic Updates](https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates) (TanStack公式)

**バージョン**: TanStack Query 5+
**確信度**: 高
**最終更新**: 2026-10-25

---

## 関連プラクティス

- [`api-client/error-handling.md`](./error-handling.md) - HTTPエラーの処理戦略
- [`api-client/caching.md`](./caching.md) - クライアントサイドキャッシュ
- [`api-client/type-safety.md`](./type-safety.md) - スキーマ駆動型安全性
- [`architecture/error-handling.md`](../architecture/error-handling.md) - アプリ全体のエラー処理
