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

#### 追加根拠 (2026-05-16) — 手動取り込み

新たに sstf-5461-admin-app チームドキュメント（原典: akfm_sato 氏 Zenn book）から以下の知見を追加:

**Request Memoization が効く条件は「同一 URL・同一オプションの `fetch`」のみ**:
コロケーション（必要なデータを必要なコンポーネントで取る）は Request Memoization により実 fetch が 1 回に集約される前提で成立する。ただし Memoization は同一 URL・同一オプションの `fetch` 呼び出しでしか効かない。引数やオプションを微妙に変えた呼び出しを各コンポーネントで散在させると Memoization が無効化されるため、**データアクセスは DAL（Data Access Layer）の関数として共通化**しておくことが前提条件となる。

**preload パターンによる並行化の補助**:
依存のないデータフェッチを並行化する際、データフェッチ単位でのコンポーネント分割（兄弟並行レンダリング）と `Promise.all` に加えて、親で `void getX()` を先に呼んでおく **preload パターン**が有効。Memoization により実 fetch は 1 回に集約される。使われなくなった preload は無駄な fetch が残るため必ず削除する。

**コード例**:
```ts
// app/_lib/dal/user.ts
import "server-only";

// 同一 URL・同一オプションで呼ぶことで Memoization が効く
export async function getCurrentUser() {
  const res = await fetch(`${API}/me`, { headers: authHeaders() });
  return toUserDTO(await res.json());
}

// preload: 親で fire-and-forget → 子で await
export const preloadCurrentUser = () => {
  void getCurrentUser();
};
```

**出典**:
- [Next.jsの考え方 / 1.2 コロケーション・1.3 並行データフェッチ](https://zenn.dev/akfm/books/nextjs-basic-principle)

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**確信度**: 既存（高）→ 高

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

---

### 4. N+1 を避けるため DataLoader でバッチ化し、`React.cache()` でリクエスト単位に閉じる

配列要素ごとに fetch する設計（一覧 → 各要素の詳細取得など）は N+1 を引き起こすため、**DataLoader** でバッチ化する。インスタンスは **`React.cache()` でリクエスト単位**に閉じて、ユーザー間でローダー内のキャッシュが漏れないようにする。DataLoader の配置は DAL の中。

**根拠**:
- 配列の各要素ごとに個別 fetch を投げると N+1 になり、サーバー間通信であってもレイテンシが累積する
- DataLoader は同一 tick 内のキーを集約し、バッチエンドポイント（`?id=1&id=2&id=3` 等）に 1 回でリクエストする
- DataLoader 自体のキャッシュをモジュールスコープ singleton にすると、別ユーザーのリクエストにキャッシュ済みデータが漏れる。`React.cache()` でラップすることで「同一リクエスト内では同一インスタンス、リクエストを跨ぐと別インスタンス」になり、安全に共有できる
- Lazy Loading（DataLoader でのバッチ集約）が辛いケース（関連が深いツリー構造など）では Eager Loading（最初の 1 回で関連を全取得）も検討する

**コード例**:
```ts
// Good: app/_lib/dal/user.ts
import "server-only";
import DataLoader from "dataloader";
import * as React from "react";

// React.cache() でリクエスト単位に閉じる（ユーザー間で漏れない）
const getUserLoader = React.cache(
  () => new DataLoader((keys: readonly number[]) => batchGetUser(keys)),
);

export async function getUser(id: number) {
  return getUserLoader().load(id);
}

// Bad: モジュールスコープ singleton はリクエスト間で共有されてしまう
const userLoader = new DataLoader((keys: readonly number[]) => batchGetUser(keys));
export async function getUserUnsafe(id: number) {
  return userLoader.load(id); // 別ユーザーにキャッシュ済み結果が漏れるリスク
}
```

**アンチパターン**:
- `posts.map(p => fetch(`/api/users/${p.authorId}`))` のように配列内で個別 fetch を投げる
- DataLoader をモジュールトップレベルで `new` して singleton として共有する（リクエスト間でキャッシュが漏れる）
- DataLoader をコンポーネント内に直接置く（DAL を経由しないと認可・DTO 変換が抜ける）

**出典**:
- [Next.jsの考え方 / 1.4 N+1 と DataLoader](https://zenn.dev/akfm/books/nextjs-basic-principle)

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**バージョン**: Next.js 16+
**確信度**: 高（v16 公式相当の知見）
**最終更新**: 2026-05-16
