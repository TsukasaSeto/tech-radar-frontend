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

## 関連プラクティス

- [`api-client/error-handling.md`](./error-handling.md) - キャッシュ更新失敗時のエラー処理
- [`nextjs/caching.md`](../nextjs/caching.md) - Next.js のサーバーサイドキャッシュ
