# API レスポンスキャッシュ戦略のベストプラクティス

## ルール

### 1. TanStack Query と SWR の使い分けを理解する

クライアントサイドのデータフェッチライブラリとして TanStack Query と SWR は
どちらも優れているが、プロジェクトの要件に応じて使い分ける。

**根拠**:
- TanStack Query は高度なキャッシュ制御、無限スクロール、楽観的更新、DevTools が充実しており、データ量の多いアプリに向いている
- SWR はシンプルな API と小さなバンドルサイズが特徴で、シンプルなユースケースに向いている
- どちらも Next.js App Router と Server Components に対応している
- RTK Query は既に Redux を使っているプロジェクトでの採用が合理的

**コード例**:
```tsx
// TanStack Query（推奨: 複雑なキャッシュ要件がある場合）
// app/providers.tsx
'use client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { useState } from 'react';

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() =>
    new QueryClient({
      defaultOptions: {
        queries: {
          staleTime: 60 * 1000,     // 1分間はfreshとみなす
          gcTime: 5 * 60 * 1000,   // 5分間キャッシュを保持（旧 cacheTime）
          retry: 1,
          refetchOnWindowFocus: false,
        },
      },
    }),
  );

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}

// SWR（推奨: シンプルなユースケース）
import useSWR from 'swr';

const fetcher = (url: string) => fetch(url).then(r => r.json());

function useUser(id: string) {
  const { data, error, isLoading } = useSWR<User>(
    `/api/users/${id}`,
    fetcher,
    {
      revalidateOnFocus: false,
      dedupingInterval: 60_000,  // 60秒以内の重複リクエストをデデュープ
    },
  );
  return { user: data, error, isLoading };
}
```

**出典**:
- [TanStack Query Docs](https://tanstack.com/query/latest/docs/framework/react/overview) (TanStack公式)
- [SWR Docs](https://swr.vercel.app/docs/getting-started) (Vercel公式)

**バージョン**: TanStack Query 5+, SWR 2+
**確信度**: 高
**最終更新**: 2026-05-05

---

#### 追加根拠 (2026-05-15) — ルール1「TanStack Query と SWR の使い分けを理解する」

新たに以下の記事で `QueryClient` を `useState` で初期化することの必要性が実例で示された:
- [自前キャッシュをTanStack Queryに移行したらコードが半分になった](https://zenn.dev/hiroki0304/articles/tanstack-query-migration-custom-cache) (Zenn / 2026-05-15) ※2026-05-15に実際にfetch成功

**出典引用**:
> "サーバーサイドではこのインスタンスが全リクエストで共有されるため、ユーザーAのキャッシュがユーザーBのレスポンスに混入するデータ漏えいが起こります"
> ([自前キャッシュをTanStack Queryに移行したらコードが半分になった], セクション "QueryClient の初期化")

Next.js App Router（SSR）では `QueryClient` をモジュールスコープ（ファイルトップレベル）で定義すると全リクエスト間でインスタンスが共有され、ユーザー間でキャッシュが漏洩する。`useState(() => new QueryClient(...))` でリクエストごとに新しいインスタンスを生成することで漏洩を防ぐ（既存のコード例はこの正しいパターンを採用済み）。

また、`staleTime` はデータの揮発性に応じて設定する。ホテル詳細のような比較的安定したデータでは5分（300,000ms）が実用的とされており、`staleTime: 0`（デフォルト）のまま放置すると不要なリフェッチが多発する。この記事の移行事例では、手動キャッシュ管理コード176行がTanStack Query採用により82行に削減された（53%削減）。

**確信度**: 既存（高）→ 高（Next.js SSR でのデータ漏洩リスクを具体例で明示化）

---

### 2. staleTime と gcTime（cacheTime）を意図的に設計する

TanStack Query のデフォルト値はほとんどのユースケースに適していないため、
データの性質に合わせて `staleTime` と `gcTime` を明示的に設定する。

**根拠**:
- `staleTime: 0`（デフォルト）はウィンドウフォーカス時に毎回リフェッチする（多くの場合過剰）
- ユーザープロフィールや設定データは数分〜数時間 stale でも問題ないケースが多い
- `gcTime`（旧 `cacheTime`）はキャッシュのメモリ保持時間であり、画面を戻った時のスピード感に影響する

**コード例**:
```ts
// データ種別ごとの staleTime 設計例

// ユーザープロフィール（更新頻度低）: 5分
const { data: profile } = useQuery({
  queryKey: ['profile', userId],
  queryFn: () => fetchProfile(userId),
  staleTime: 5 * 60 * 1000,   // 5分間はリフェッチしない
  gcTime: 30 * 60 * 1000,     // 30分キャッシュ保持
});

// ダッシュボード統計（更新頻度中）: 1分
const { data: stats } = useQuery({
  queryKey: ['dashboard', 'stats'],
  queryFn: fetchDashboardStats,
  staleTime: 60 * 1000,        // 1分
  refetchInterval: 60 * 1000,  // バックグラウンドで1分毎にリフェッチ
});

// 在庫数（リアルタイム性が高い）: キャッシュしない
const { data: inventory } = useQuery({
  queryKey: ['inventory', productId],
  queryFn: () => fetchInventory(productId),
  staleTime: 0,                // 常にfetchするが、ローディング中はキャッシュを表示
  refetchOnWindowFocus: true,  // フォーカス時にリフェッチ
});

// 静的なマスターデータ（都道府県リスト等）: Infinity
const { data: prefectures } = useQuery({
  queryKey: ['prefectures'],
  queryFn: fetchPrefectures,
  staleTime: Infinity,  // 絶対にリフェッチしない
  gcTime: Infinity,     // メモリに永続
});
```

**出典**:
- [TanStack Query: Important Defaults](https://tanstack.com/query/latest/docs/framework/react/guides/important-defaults) (TanStack公式)
- [TanStack Query: Caching](https://tanstack.com/query/latest/docs/framework/react/guides/caching) (TanStack公式)

**バージョン**: TanStack Query 5+（注: v4 では `cacheTime`、v5 では `gcTime`）
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. クエリキーはファクトリ関数で構造化する

クエリキーを文字列リテラルで散在させず、
クエリキーファクトリ関数でプロジェクト全体の命名を統一する。

**根拠**:
- クエリキーはキャッシュの識別子であり、命名が重要
- ファクトリ関数で型安全にクエリキーを生成できる
- `queryClient.invalidateQueries` での無効化範囲を正確に制御できる
- キー構造の変更が1箇所に集約される

**コード例**:
```ts
// lib/query-keys.ts
export const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  list: (filters: UserFilters) => [...userKeys.lists(), { filters }] as const,
  details: () => [...userKeys.all, 'detail'] as const,
  detail: (id: string) => [...userKeys.details(), id] as const,
};

// 使用例
const { data: users } = useQuery({
  queryKey: userKeys.list({ role: 'admin' }),
  queryFn: () => fetchUsers({ role: 'admin' }),
});

const { data: user } = useQuery({
  queryKey: userKeys.detail(userId),
  queryFn: () => fetchUser(userId),
});

// Mutation 後の無効化（user.list 系を全て無効化）
const queryClient = useQueryClient();
await queryClient.invalidateQueries({ queryKey: userKeys.lists() });

// 特定ユーザーのキャッシュだけ無効化
await queryClient.invalidateQueries({ queryKey: userKeys.detail(userId) });

// Bad: 文字列リテラルをコード全体に散在させる
useQuery({ queryKey: ['user', id] });  // どこかで ['users', id] と書いてキャッシュが分離
```

**出典**:
- [TanStack Query: Query Keys](https://tanstack.com/query/latest/docs/framework/react/guides/query-keys) (TanStack公式)

**バージョン**: TanStack Query 5+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. 楽観的更新（Optimistic Updates）でUIの体感速度を向上させる

ユーザーの操作結果をサーバーレスポンス前に先行反映し、
エラー時にロールバックする楽観的更新パターンを採用する。

**根拠**:
- いいね・チェックアウト・タスク完了など即時フィードバックが重要な操作に有効
- ネットワーク遅延をユーザーが体感しにくくなる
- エラー時のロールバックで一貫性を保つ
- TanStack Query の `onMutate` / `onError` / `onSettled` で実装できる

**コード例**:
```tsx
// TanStack Query での楽観的更新
function TodoItem({ todo }: { todo: Todo }) {
  const queryClient = useQueryClient();

  const { mutate: toggleTodo } = useMutation({
    mutationFn: (id: string) => apiClient.PATCH('/todos/{id}', {
      params: { path: { id } },
      body: { completed: !todo.completed },
    }),

    onMutate: async (id) => {
      // 進行中のクエリをキャンセル（競合を防ぐ）
      await queryClient.cancelQueries({ queryKey: ['todos'] });

      // 現在のキャッシュをスナップショット
      const previousTodos = queryClient.getQueryData<Todo[]>(['todos']);

      // 楽観的にキャッシュを更新
      queryClient.setQueryData<Todo[]>(['todos'], old =>
        old?.map(t => t.id === id ? { ...t, completed: !t.completed } : t) ?? [],
      );

      // ロールバック用にスナップショットを返す
      return { previousTodos };
    },

    onError: (err, id, context) => {
      // エラー時にスナップショットに戻す
      queryClient.setQueryData(['todos'], context?.previousTodos);
      showErrorToast('更新に失敗しました');
    },

    onSettled: () => {
      // 成功・失敗に関わらず最新データを取得
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });

  return (
    <li onClick={() => toggleTodo(todo.id)}>
      {todo.completed ? '✅' : '⬜'} {todo.title}
    </li>
  );
}
```

**出典**:
- [TanStack Query: Optimistic Updates](https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates) (TanStack公式)

**バージョン**: TanStack Query 5+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 5. Next.js App Router では HydrationBoundary でサーバーサイドプリフェッチを実装する

Server Component でデータをプリフェッチし、`HydrationBoundary` と `dehydrate()` でクライアントに
キャッシュを引き渡す。初期ローディング状態を排除し、サーバーサイドのデータをクライアントに継承できる。

**根拠**:
- Server Component でプリフェッチすることで、クライアントに届いた段階でデータがキャッシュ済みになり初期ローディングを排除できる
- `React.cache()` で `QueryClient` の再生成を防ぎ、同一リクエスト内で1インスタンスを保証する
- `useSuspenseQuery` と組み合わせるとローディング/エラー管理を `<Suspense>` / `<ErrorBoundary>` に委譲できる（v5 では `useQuery` の `suspense: true` は廃止済み）

**コード例**:
```tsx
// app/posts/page.tsx（Server Component）
import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query';
import { cache } from 'react';
import { PostList } from '@/features/posts';

// React.cache() で同一リクエスト内で QueryClient を再利用（新規生成を防ぐ）
const getQueryClient = cache(
  () => new QueryClient({ defaultOptions: { queries: { staleTime: 60 * 1000 } } })
);

export default async function PostsPage() {
  const queryClient = getQueryClient();

  // サーバーサイドでプリフェッチ
  await queryClient.prefetchQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  });

  // dehydrate() でシリアライズ → HydrationBoundary でクライアントに注入
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PostList />
    </HydrationBoundary>
  );
}

// features/posts/components/PostList.tsx（Client Component）
'use client';
import { useSuspenseQuery } from '@tanstack/react-query';  // v5: suspense: true は廃止

function PostList() {
  // HydrationBoundary のキャッシュが存在するので初期ローディングが発生しない
  const { data: posts } = useSuspenseQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  });

  return <ul>{posts.map(post => <li key={post.id}>{post.title}</li>)}</ul>;
}
// → ローディング管理は親の <Suspense fallback={<Skeleton />}> に委譲
```

**出典**:
- [TanStack Query実践パターン ： キャッシュ・楽観的更新・無限スクロール](https://zenn.dev/correlate_dev/articles/react-query-patterns) (Zenn correlate_dev / 2026) ※2026-05-06に実際にfetch成功

**バージョン**: TanStack Query 5+, Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-06

---

## 関連プラクティス

- [`api-client/error-handling.md`](./error-handling.md) - キャッシュ更新失敗時のエラー処理
- [`nextjs/caching.md`](../nextjs/caching.md) - Next.js のサーバーサイドキャッシュ

---

#### 追加根拠 (2026-05-06) — ルール1「TanStack Query と SWR の使い分けを理解する」

新たに以下の記事でTanStack Query v5の重要な変更点が示された:
- [TanStack Query実践パターン ： キャッシュ・楽観的更新・無限スクロール](https://zenn.dev/correlate_dev/articles/react-query-patterns) (Zenn correlate_dev / 2026) ※2026-05-06に実際にfetch成功

TanStack Query v5 の破壊的変更: `useQuery` の `suspense: true` オプションが廃止され、`useSuspenseQuery` フックが正式な Suspense 統合手段になった。`useSuspenseQuery` を使うとローディング/エラー状態の管理が `<Suspense>` と `<ErrorBoundary>` に委譲され、コンポーネント内でのステータスチェックが不要になる。また `useInfiniteQuery` v5 では `initialPageParam` が必須になった（v4 からの破壊的変更）。gcTime は staleTime の約2倍を目安に設定すると、stale中のキャッシュが消える問題を防ぐ。

**確信度**: 既存（高）→ 高（v5破壊的変更を実例で確認）

---

#### 追加根拠 (2026-05-06) — ルール3「クエリキーはファクトリ関数で構造化する」

新たに以下のドキュメントでクエリキーの挙動の細則が公式確認された:
- [TanStack Query: Query Keys](https://raw.githubusercontent.com/TanStack/query/main/docs/framework/react/guides/query-keys.md) (TanStack / mainブランチ) ※2026-05-06に実際にfetch成功

公式ドキュメントが2点を明示: (1) **オブジェクト内のプロパティ順序は関係ない** — `{ status, page }` と `{ page, status }` は同一キーとして扱われる（決定論的ハッシュ）。(2) **配列内の要素順序は関係ある** — `['todos', status, page]` と `['todos', page, status]` は別のキーになる。クエリキーファクトリパターンはこれらの挙動を意識したうえで一貫した構造を定義することで、予期しないキャッシュ分離を防ぐ。

**確信度**: 既存（高）→ 高（公式ドキュメントで細則確認済み）
