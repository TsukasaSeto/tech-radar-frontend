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
import ky from 'ky';

export const http = ky.create({
  prefixUrl: process.env.NEXT_PUBLIC_API_BASE_URL,
  timeout: 10_000,        // 10秒タイムアウト（per-attempt）
  retry: {
    limit: 2,             // 最大2回リトライ
    methods: ['get'],     // GETのみリトライ
    statusCodes: [408, 429, 500, 502, 503, 504],
  },
  hooks: {
    beforeRequest: [
      request => {
        const token = getAuthToken();
        if (token) request.headers.set('Authorization', `Bearer ${token}`);
      },
    ],
    beforeRetry: [
      async ({ request, error, retryCount }) => {
        if (retryCount === 1) await refreshToken();
      },
    ],
    beforeError: [
      async error => {
        const { response } = error;
        if (response) {
          const body = await response.json().catch(() => ({}));
          error.message = (body as { message?: string }).message ?? error.message;
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
- [ky README](https://github.com/sindresorhus/ky) (sindresorhus/ky / GitHub)

**バージョン**: ky 1.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. openapi-fetch でスキーマ駆動の型安全な REST クライアントを構築する

OpenAPI スキーマから `openapi-typescript` で型を生成し、
`openapi-fetch` でランタイムの型安全なクライアントを作る。

**根拠**:
- API 仕様とクライアントコードの乖離をコンパイル時に検出できる
- パスパラメータ・クエリパラメータ・リクエストボディ・レスポンスがすべて型付けされる
- OpenAPI スキーマがあれば追加のランタイム検証なしで型安全性を確保できる
- バックエンドの型変更をフロントエンドが即座に検知できる

**コード例**:
```bash
# スキーマから型を生成
npx openapi-typescript https://api.example.com/openapi.json -o src/types/api.d.ts
```

```ts
// lib/api-client.ts
import createClient from 'openapi-fetch';
import type { paths } from '@/types/api';

export const apiClient = createClient<paths>({
  baseUrl: process.env.NEXT_PUBLIC_API_BASE_URL,
  headers: { 'Content-Type': 'application/json' },
});

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
- [openapi-fetch Docs](https://openapi-ts.dev/openapi-fetch/) (openapi-ts.dev)
- [openapi-typescript Docs](https://openapi-ts.dev/introduction) (openapi-ts.dev)

**バージョン**: openapi-fetch 0.12+, openapi-typescript 7+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. 認証ヘッダーの付与はインターセプターで一元化する

各 fetch 呼び出しに認証ヘッダーを手動で付与するのではなく、
インターセプター（ky の beforeRequest フック等）で一元管理する。

**根拠**:
- 認証ロジックの変更が1箇所で完結する
- トークンリフレッシュロジックを beforeRetry フックに集約できる
- 個々のAPIコールがアプリケーションロジックに専念できる

**コード例**:
```ts
// lib/http.ts
import ky from 'ky';
import { getSession } from 'next-auth/react';

export const authenticatedHttp = ky.create({
  prefixUrl: process.env.NEXT_PUBLIC_API_BASE_URL,
  hooks: {
    beforeRequest: [
      async request => {
        const session = await getSession();
        if (session?.accessToken) {
          request.headers.set('Authorization', `Bearer ${session.accessToken}`);
        }
      },
    ],
    beforeRetry: [
      async ({ request, retryCount }) => {
        if (retryCount === 0) {
          // 401 時にトークンをリフレッシュして再試行
          const newToken = await refreshAccessToken();
          if (newToken) request.headers.set('Authorization', `Bearer ${newToken}`);
        }
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
- [ky README: Hooks](https://github.com/sindresorhus/ky#hooks) (sindresorhus/ky / GitHub)

**バージョン**: ky 1.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 5. Server Components / Route Handlers での fetch は next の拡張オプションを活用する

Next.js の fetch 拡張オプション（`cache`, `next.revalidate`, `next.tags`）を使い、
サーバーサイドのデータ取得を最適化する。

**根拠**:
- Next.js は fetch を拡張しており、ISR・キャッシュ制御がオプション1つで設定できる
- `next.tags` でタグベースの revalidation が可能（`revalidateTag()` と組み合わせ）
- データ取得関数を複数の Server Component で呼んでも重複リクエストは自動で排除される（Request Memoization）

**コード例**:
```ts
// lib/data.ts

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

// Server Action でキャッシュを無効化
'use server';
import { revalidateTag } from 'next/cache';

export async function updateProduct(id: string, data: ProductUpdate) {
  await fetch(`https://api.example.com/products/${id}`, {
    method: 'PUT',
    body: JSON.stringify(data),
    cache: 'no-store',
  });
  revalidateTag('products');  // 'products' タグのキャッシュを無効化
}
```

**出典**:
- [Next.js Docs: fetch](https://nextjs.org/docs/app/api-reference/functions/fetch) (Next.js公式)
- [Next.js Docs: Caching](https://nextjs.org/docs/app/building-your-application/caching) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

## 関連プラクティス

- [`api-client/error-handling.md`](./error-handling.md) - HTTPエラーの処理戦略
- [`api-client/caching.md`](./caching.md) - クライアントサイドキャッシュ
- [`api-client/type-safety.md`](./type-safety.md) - スキーマ駆動型安全性
- [`architecture/error-handling.md`](../architecture/error-handling.md) - アプリ全体のエラー処理
