# データフェッチのベストプラクティス

## ルール

### 1. Server Components でデータを取得し、依存のない取得は並列化する（ウォーターフォール回避）

データはサーバー側の Server Components で取得し、必要なコンポーネントの近くで取得する（コロケーション。ページで全部取って props で配るより、各コンポーネントで取得して `<Suspense>` で分ける形を主にする。#3）。
1 つのコンポーネントが互いに依存しない複数のデータを必要とする場合は、逐次 `await` せず `Promise.all` で並列に取得する。

**根拠**:
- サーバー側のデータ取得は、クライアントからの往復によるウォーターフォールを削減する
- 依存のない取得を逐次 `await` すると、前の取得の完了を待ってから次が始まるので、合計時間が足し算になる。`Promise.all` なら最も遅い取得の時間で済む
- 同一 URL・同一オプションの `fetch` は Request Memoization で同一レンダー内の重複が 1 回にまとまるので、コロケーションしても実 fetch は増えない。v15 以降、`fetch` はデフォルトではキャッシュされない（Data Cache への保存は明示が必要。`caching.md`）

**コード例**:
```tsx
// Good: 1 つのコンポーネントが必要とする、互いに依存しないデータを並列に取得する
async function ProfileHeader({ userId }: { userId: string }) {
  const [user, stats] = await Promise.all([getUser(userId), getUserStats(userId)]);
  return <Header user={user} stats={stats} />;
}

// Bad: 依存関係がないのに逐次 await してウォーターフォールにする
async function ProfileHeaderSlow({ userId }: { userId: string }) {
  const user = await getUser(userId);
  const stats = await getUserStats(userId); // user の取得完了を待ってから始まる
  return <Header user={user} stats={stats} />;
}
```

**出典**:
- [Next.js Docs: Data Fetching Patterns](https://nextjs.org/docs/app/building-your-application/data-fetching/patterns) (Next.js公式)
- [Next.js Docs: Caching and Revalidating (Previous Model)](https://nextjs.org/docs/app/guides/caching-without-cache-components) (Next.js 公式、`fetch` はデフォルトではキャッシュされない。`fetch` は自動でメモ化される) ※2026-10-25 確認

**バージョン**: Next.js 13+（`fetch` がデフォルトでキャッシュされないのは 15+）
**確信度**: 高
**最終更新**: 2026-10-25

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
- ユーザーは部分的なコンテンツを先に見られる。サーバーは HTML を段階的にストリーミングし、ブラウザは受け取り次第レンダリングできる
- `loading.tsx` はルート全体に Suspense を張るショートカット。コンポーネント単位で `<Suspense>` を張れば、どこまでを先に出すかを細かく制御できる
- ページで全データを `Promise.all` してから描画すると、すべての取得が終わるまで何も表示されない（依存のない取得を 1 コンポーネント内で並列化するのは #1）
- **LCP 要素（ヒーロー画像・主要見出し等）は Suspense 境界の外側（初期シェル）に置く**。スローデータを待つ Suspense 内に LCP 要素があると、シェルが届いてもレンダリングが遅延し LCP が最大 400ms 悪化するケースが観測されている

**コード例**:
```tsx
// app/dashboard/page.tsx — LCP要素を初期シェルに、スローデータをSuspense境界内に分離
import { Suspense } from 'react';

export default function DashboardPage() {
  return (
    <div>
      {/* 初期シェル: LCP対象要素（ヒーロー画像等）はここに置く */}
      <HeroImage />  {/* ← LCP要素: Suspenseの外側 */}
      <PageHeader />
      {/* スローデータ: Suspenseで遅延 */}
      <Suspense fallback={<StatsSkeleton />}>
        <StatsPanel />
      </Suspense>
      <Suspense fallback={<FeedSkeleton />}>
        <ActivityFeed />
      </Suspense>
    </div>
  );
}

// components/StatsPanel.tsx (Server Component) — 自分が必要なデータを自分で取得する
export async function StatsPanel() {
  const stats = await getStats(); // DAL 経由
  return <div>{/* stats を表示 */}</div>;
}

// Bad: ページ全体を一つの async Server Component にまとめて全データを待つ
export default async function DashboardPage() {
  const [stats, feed] = await Promise.all([getStats(), getActivityFeed()]);
  // 両方の取得が完了するまでユーザーは何も見えない
  return <div>...</div>;
}
```

**出典**:
- [Next.js Docs: Streaming with Suspense](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming) (Next.js公式)
- [Next.js Docs: Data Fetching Patterns](https://nextjs.org/docs/app/building-your-application/data-fetching/patterns) (Next.js公式 / 2024)
- [Streaming SSR Is Not a Free LCP Win](https://dev.to/nosyos/streaming-ssr-is-not-a-free-lcp-win-5b8l) (dev.to、LCP配置の実測データ) ※2026-05-26に実際にfetch成功

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-10-25

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
**確信度**: 中（Zenn book 1 冊が根拠。DataLoader と `React.cache()` の組み合わせは Next.js 公式ドキュメントの記載ではない）
**最終更新**: 2026-10-25

---

### 5. 接続断時は `useOffline` フックで自動リトライ中の UI を出す（実験的機能）

Next.js 16.3 の実験的機能 `experimental.useOffline` を有効にすると、RSC フェッチ・prefetch・Server Actions がオフライン/接続断で失敗した際に自動的にリトライされる。アプリ側は `useOffline` フックの戻り値を見て「再接続中」等のフィードバック UI を出す責務だけを持てばよい。本番運用は非推奨の実験的機能である点に注意する。

**根拠**:
- 接続断のハンドリングを各 fetch 呼び出し側で個別に実装せず、フレームワーク側の自動リトライに委譲できる
- `useOffline` はリトライの実施有無ではなく「現在オフライン状態かどうか」の可視化に専念させる設計になっている。UI 側の責務を再接続中インジケータの表示だけに絞れる

**コード例**:
```ts
// next.config.ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  experimental: {
    useOffline: true,
  },
};

export default nextConfig;
```
```tsx
// Good: useOffline で再接続中のフィードバックのみを担当する
'use client';
import { useOffline } from 'next/navigation';

export function ReconnectingBanner() {
  const isOffline = useOffline();
  if (!isOffline) return null;
  return <div role="status">再接続しています…</div>;
}
```

**出典引用**:
> "Offline support is currently experimental and subject to change, and is not recommended for production."
> ([Building App-like Experiences with Next.js 16.3](https://nextjs.org/blog/building-app-like-experiences-with-nextjs-16-3), セクション "Handling connection drops") ※2026-08-18に実際にfetch成功

**出典**:
- [Building App-like Experiences with Next.js 16.3](https://nextjs.org/blog/building-app-like-experiences-with-nextjs-16-3) (Next.js 公式ブログ、`useOffline` フックと `experimental.useOffline` 設定の解説) ※2026-08-18 fetch

**バージョン**: Next.js 16.3+（`experimental.useOffline`、本番非推奨）
**確信度**: 高
**最終更新**: 2026-08-18
