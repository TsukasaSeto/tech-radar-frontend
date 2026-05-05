# データフェッチのベストプラクティス

## ルール

### 1. Server Components でデータを取得し、props で渡す（ウォーターフォール回避）

可能な限りサーバー側でデータを取得する。並列で取得できるデータは `Promise.all` を使う。

**根拠**:
- サーバー側のデータ取得はウォーターフォールを削減する
- `fetch` に自動的なキャッシュ・重複除去が適用される

**コード例**:
```tsx
async function Dashboard() {
  const [user, posts, stats] = await Promise.all([
    fetchUser(),
    fetchPosts(),
    fetchStats(),
  ]);
  return (
    <>
      <UserProfile user={user} />
      <PostList posts={posts} />
      <StatsPanel stats={stats} />
    </>
  );
}
```

**出典**:
- [Next.js Docs: Data Fetching Patterns](https://nextjs.org/docs/app/building-your-application/data-fetching/patterns) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. `fetch` のキャッシュオプションを明示的に指定する

Next.js の `fetch` のデフォルト挙動はバージョンで変わったため、意図を明示的に指定する。

**コード例**:
```tsx
// 静的データ（ビルド時にフェッチ、長期キャッシュ）
const staticData = await fetch('https://api.example.com/products', {
  cache: 'force-cache',
});

// 動的データ（キャッシュなし）
const dynamicData = await fetch('https://api.example.com/live-data', {
  cache: 'no-store',
});

// 一定時間で再検証（ISR）
const revalidatedData = await fetch('https://api.example.com/posts', {
  next: { revalidate: 3600, tags: ['posts'] },
});
```

**出典**:
- [Next.js Docs: Fetching Data on the Server](https://nextjs.org/docs/app/building-your-application/data-fetching/fetching) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. コンポーネント境界でデータフェッチを分散させる（並列フェッチ + Streaming）

各コンポーネントが独立してデータを取得し、Suspense と組み合わせてストリーミングレスポンスを活用する。

**根拠**:
- 最も遅いデータがページ全体の表示を妨げなくなる
- ユーザーは部分的なコンテンツを先に見られる

**コード例**:
```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react';

export default function DashboardPage() {
  return (
    <div>
      <PageHeader />
      <Suspense fallback={<StatsSkeleton />}>
        <StatsPanel />
      </Suspense>
      <Suspense fallback={<FeedSkeleton />}>
        <ActivityFeed />
      </Suspense>
    </div>
  );
}
```

**出典**:
- [Next.js Docs: Streaming with Suspense](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05
