# Next.js キャッシュのベストプラクティス

> **層の役割**: このファイルは **Server 層**（Request Memoization / Data Cache / Full Route Cache / Router Cache / `"use cache"`）を扱う。
> **Client 層**のキャッシュ（TanStack Query / SWR / Hydration Boundary）は [`api-client/caching.md`](../api-client/caching.md) を参照。
> Server → Client の境界（RSC で取得 → Client にハイドレート）は両ファイルにまたがるが、**取得側ロジックは Server 層、再フェッチ / 楽観更新は Client 層** という分担を基本にする。

## ルール

### 1. Next.js の4層キャッシュを理解して使い分ける

Next.js 14+ には4つのキャッシュ層がある。それぞれの役割と無効化方法を理解する。

**根拠**:
- 各キャッシュ層は独立しており、適切な層を無効化しないと古いデータが表示される
- デフォルトが変わったバージョン（13→15）があるため明示的な指定が安全
- **Vercel にデプロイする場合、Data Cache は Runtime Cache に移行する（2026-10-30 告知）**: `fetch` キャッシュと `unstable_cache` はコード変更なしで Runtime Cache に切り替わる。リージョンごとに独立したキャッシュで、ミス時に他リージョンは参照されない。移行後はこれまで無課金だった Data Cache の読み書きが Runtime Cache 料金で課金されるため、移行前に Observability で読み書き量を確認し、不要な読み書きを減らす
- **開発モードでの検証は Data Cache の挙動を隠す**: `fetch` リクエストに `cache-control: no-cache` ヘッダーが付く場合（開発モードのハードリロード等）、`options.cache` / `options.next.revalidate` / `options.next.tags` は無視される。そのためローカル開発では「DB更新後すぐ最新が表示される」ように見えても、本番相当の環境では Data Cache が効いたまま古いデータが表示され続けるバグに気づけない

**コード例**:
```
1. Request Memoization - 同一レンダリング内の重複 fetch を自動デデュープ
2. Data Cache - fetch の cache オプションで制御
3. Full Route Cache - 静的ルートの HTML をサーバーにキャッシュ
4. Router Cache - ブラウザのセッション中に RSC ペイロードをメモリにキャッシュ
```

**出典**:
- [Next.js Docs: Caching Overview](https://nextjs.org/docs/app/building-your-application/caching) (Next.js公式)
- [Next.js App Router で DB を更新しても表示データが古いまま — 原因はサーバー側の Data Cache だった](https://zenn.dev/miyuyu/articles/nextjs-data-cache-not-updated) (Zenn yuyami-ya、開発モードで `no-cache` ヘッダーが付くと cache オプションが無視され本番との挙動差でバグを見逃す原因を解説) ※2026-07-08に実際にfetch成功
- [Data Cache is moving to Runtime Cache](https://vercel.com/changelog/data-cache-is-moving-to-runtime-cache) (Vercel Changelog、移行時期・課金変更) ※2026-10-30に実際にfetch成功
- [Simplifying runtime caching on Vercel](https://vercel.com/blog/simplifying-runtime-caching-on-vercel) (Vercel Blog、リージョン別キャッシュとタグ無効化の仕組み) ※2026-10-30に実際にfetch成功

> "Next.js fetch caching and unstable_cache will switch over on every version without code changes."
> ([Data Cache is moving to Runtime Cache](https://vercel.com/changelog/data-cache-is-moving-to-runtime-cache), 冒頭) ※2026-10-30に実際にfetch成功

> "Each region has its own cache."
> ([Simplifying runtime caching on Vercel](https://vercel.com/blog/simplifying-runtime-caching-on-vercel), セクション "How Runtime Cache works") ※2026-10-30に実際にfetch成功

> "開発モードでは、`fetch` リクエストに `cache-control: no-cache` ヘッダーが含まれる場合、`options.cache` / `options.next.revalidate` / `options.next.tags` は無視されます"
> ([Next.js App Router で DB を更新しても表示データが古いまま](https://zenn.dev/miyuyu/articles/nextjs-data-cache-not-updated), セクション "なぜローカルでは最新が出たのか") ※2026-07-08に実際にfetch成功

**バージョン**: Next.js 14+
**確信度**: 高
**最終更新**: 2026-10-30

---

### 2. `revalidateTag` と `revalidatePath` でオンデマンド再検証する

データ更新時はタグベースまたはパスベースのオンデマンド再検証を使う。
v16 では `revalidateTag` に第 2 引数（`'max'` 推奨）が必須で、Server Action で更新を即時に反映したい場合は `updateTag` を使う。

**根拠**:
- タグベースはデータの種類に対応して複数ページを一括無効化できる。`revalidateTag` はそのタグを使うすべてのページのデータを、`revalidatePath` は指定したページ / layout のパスを無効化する
- Server Actions と組み合わせることでミューテーション後の自動更新が可能。ミューテーションを行う Action では、認証・入力検証を先に行い、許可したフィールドだけを DAL に渡す（`server-actions.md` #5）
- v16 では `revalidateTag(tag)` の 1 引数版は非推奨（TypeScript の型エラーになる）。`revalidateTag(tag, 'max')` は stale-while-revalidate、`updateTag(tag)` は Server Action 専用で、期限切れにしてすぐ取り直す（read-your-writes）
- タグ付きのデータを経由しないルート（`next.tags` / `cacheTag` を付けずに取得したデータで描画するルート）は、`revalidateTag` を呼んでも更新されない。この場合は取得にタグを付けるか、`revalidatePath` でパスを直接無効化するか、Cache Components 無効時はルートに `export const revalidate = <秒数>` を指定して時間ベースで再生成させる（Cache Components 有効時は `revalidate` を export できないので、`'use cache'` + `cacheLife` を使う）

**コード例**:
```tsx
// app/_actions/user.ts
'use server';
import { z } from 'zod';
import { revalidatePath, updateTag } from 'next/cache';
import { unauthorized } from 'next/navigation';
import { verifySession } from '@/app/_lib/dal/auth';
import { createUser } from '@/app/_lib/dal/user.mutations'; // 権限の確認と書き込みは DAL で行う

// 許可したフィールドだけを受け付ける（role などの想定外フィールドは捨てる）
const CreateUserInput = z.object({
  name: z.string().min(1).max(50),
  email: z.email(),
});

export async function createUserAction(formData: FormData): Promise<void> {
  const session = await verifySession();
  if (!session) unauthorized(); // experimental.authInterrupts: true が前提

  const parsed = CreateUserInput.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
  });
  if (!parsed.success) return;

  await createUser(parsed.data);
  updateTag('users');        // 作成した本人にすぐ反映する（多少遅れてよければ revalidateTag('users', 'max')）
  revalidatePath('/users');
}

// Bad: フォームを丸ごと DB に渡す（マスアサインメント。role なども書き込める）
// await db.user.create({ data: Object.fromEntries(formData) });
```

```tsx
// Bad（Cache Components 無効時）: タグを付けずに取得したデータで描画するルートは、revalidateTag('posts', 'max') を呼んでも更新されない
// app/sitemap.ts（Request-time API や dynamic 設定を使わない sitemap はデフォルトでキャッシュされる）
// Cache Components 有効時は、キャッシュしていない DB 取得はリクエストごとに実行される
import type { MetadataRoute } from 'next';
import { db } from '@/lib/db';

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const posts = await db.post.findMany({ select: { slug: true, updatedAt: true } }); // タグなし
  return posts.map((p) => ({ url: `https://example.com/blog/${p.slug}`, lastModified: p.updatedAt }));
}
```

```tsx
// Good（Cache Components 無効時）: ルートに再検証秒数を指定する
// app/sitemap.ts
export const revalidate = 3600; // 1時間ごとにルートの描画結果を再生成

// Good（Cache Components 有効時）: 取得を "use cache" + cacheTag に寄せ、タグで無効化できるようにする
// app/sitemap.ts
import type { MetadataRoute } from 'next';
import { cacheLife, cacheTag } from 'next/cache';
import { db } from '@/lib/db';

async function getPostsForSitemap() {
  'use cache';
  cacheLife('hours');
  cacheTag('posts'); // revalidateTag('posts', 'max') / updateTag('posts') で更新される
  return db.post.findMany({ select: { slug: true, updatedAt: true } });
}

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const posts = await getPostsForSitemap();
  return posts.map((p) => ({ url: `https://example.com/blog/${p.slug}`, lastModified: p.updatedAt }));
}
```

**出典**:
- [Next.js Docs: On-demand Revalidation](https://nextjs.org/docs/app/building-your-application/data-fetching/incremental-static-regeneration#on-demand-revalidation-with-revalidatetag) (Next.js公式)
- [Next.js Docs: revalidateTag](https://nextjs.org/docs/app/api-reference/functions/revalidateTag) (Next.js 公式、タグを使うすべてのページが対象・第 2 引数・1 引数版の非推奨) ※2026-10-25 確認
  > "`revalidateTag` invalidates data with specific tags across all pages that use those tags, while `revalidatePath` invalidates specific page or layout paths."
- [Next.js Docs: How to upgrade to version 16 — revalidateTag / updateTag](https://nextjs.org/docs/app/guides/upgrading/version-16#revalidatetag) (Next.js 公式、1 引数版は非推奨で TypeScript エラー、即時反映は `updateTag`) ※2026-10-25 確認
- [Next.js Docs: sitemap.xml](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/sitemap) (Next.js 公式、sitemap は Request-time API などを使わない限りデフォルトでキャッシュされる) ※2026-10-25 確認
- [Next.js sitemap not updating? Here's the real fix](https://dev.to/techpotions/nextjs-sitemap-not-updating-heres-the-real-fix-2257) (dev.to techpotions、サイトマップが更新されない事例と `export const revalidate` による ISR 対応) ※2026-07-29に実際にfetch成功

**バージョン**: Next.js 13+（`revalidateTag` の第 2 引数と `updateTag` は 16+。`export const revalidate` は Cache Components 無効時のみ）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 3. `unstable_cache` で ORM クエリ等の非 fetch 呼び出しをキャッシュする（Cache Components 無効時・既存コード向け）

Cache Components を使わない構成（v15 以前の書き方を含む）では、`fetch` を使わない ORM クエリや外部 SDK の呼び出しを `unstable_cache` でキャッシュできる。
v16 の Cache Components では `'use cache'` + `cacheLife` / `cacheTag` に置き換えられる。`unstable_cache` は引き続き動くので、移行は任意。React `cache()` との使い分けと移行の詳細は #5 を参照。

**根拠**:
- `fetch` 以外のデータ取得は `fetch` のキャッシュオプションを使えないので、`unstable_cache` でラップしてキャッシュとタグを付ける
- 公式の Cache Components 移行ガイドは、既存の `unstable_cache` はそのまま動き、移行は後回しにできるとしている

**コード例**:
```tsx
// Cache Components 無効時 / 既存コード
import { unstable_cache } from 'next/cache';

const getCachedUsers = unstable_cache(
  async () => db.user.findMany(),
  ['users-list'],
  { tags: ['users'], revalidate: 3600 }
);
```

**出典**:
- [Next.js Docs: unstable_cache](https://nextjs.org/docs/app/api-reference/functions/unstable_cache) (Next.js公式)
- [Next.js Docs: Migrating to Cache Components — `unstable_cache`](https://nextjs.org/docs/app/guides/migrating-to-cache-components#unstable_cache) (Next.js 公式、そのまま動き、移行は任意) ※2026-10-25 確認

**バージョン**: Next.js 14+（Cache Components 無効時。有効時も動くが、新規コードは `'use cache'`）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 4. `dynamic` エクスポートで静的/動的レンダリングを明示制御する（Cache Components 無効時のみ）

Cache Components 無効時は、`export const dynamic` でルートの動的レンダリングを明示的に制御する。
Cache Components 有効時は `dynamic` / `revalidate` / `fetchCache` を export するとエラーになる。`dynamic` を削除し、リクエスト時に読むデータは `<Suspense>` の内側に、再利用するデータは `'use cache'` に置く（#7）。

**根拠**:
- `'force-dynamic'` はルート全体をリクエストごとにレンダリングし、`fetch` をキャッシュしない設定になる
- `cookies()` / `headers()` などの Request-time API を使うと、`dynamic` を書かなくても動的レンダリングになる。v16 ではこれらは `await` して使う（同期アクセスは削除された）
- Cache Components 有効時は、キャッシュしていないデータと Request-time API が最初からリクエスト時に実行されるので、`dynamic` の指定は不要になり、export するとエラーになる（公式の移行ガイド）

**コード例**:
```tsx
// Cache Components 無効時のみ（有効時は dynamic を export するとエラーになる）
import { cookies } from 'next/headers';

// 常に動的レンダリング
export const dynamic = 'force-dynamic';

export default async function Page() {
  // cookies() / headers() を使うと、dynamic を書かなくても動的レンダリングになる
  const theme = (await cookies()).get('theme')?.value;
  return <Dashboard theme={theme} />;
}
```

**出典**:
- [Next.js Docs: Route Segment Config](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config) (Next.js公式)
- [Next.js Docs: Migrating to Cache Components — `dynamic = "force-dynamic"`](https://nextjs.org/docs/app/guides/migrating-to-cache-components#dynamic--force-dynamic) (Next.js 公式、有効時は `dynamic` / `revalidate` / `fetchCache` を export するとエラーになり、削除して `<Suspense>` / `use cache` に置き換える) ※2026-10-25 確認
  > "After enabling the flag, route segments that still export `dynamic`, `revalidate`, or `fetchCache` will error."

**バージョン**: Next.js 13+（`dynamic` の export。コード例の `await cookies()` は 15+、16 で必須）。Cache Components 有効時は使えない
**確信度**: 高
**最終更新**: 2026-10-25

---

### 5. `unstable_cache` と React `cache()` を目的に応じて使い分ける

`unstable_cache` はリクエストをまたいだ永続的なキャッシュに、React の `cache()` は同一レンダリングパス内のリクエスト重複除去（メモ化）に使う。用途が異なるため、混同せず適切に使い分ける。
`unstable_cache` は Cache Components 無効時と既存コード向けの書き方で、v16 の Cache Components では `'use cache'` + `cacheLife` / `cacheTag` が後継になる（#7）。

**根拠**:
- `cache()` はリクエストのライフタイム内のみ有効で、再検証制御はできない
- `unstable_cache` は Data Cache に書き込み、`revalidateTag` 等で無効化できる
- v16 の Cache Components では `'use cache'` が `unstable_cache` の後継。ただし `unstable_cache` は引き続き動き、移行は任意（公式の移行ガイド）。移行するときは、キー配列が不要になり（引数から自動でキーが作られる）、`tags` / `revalidate` は `cacheTag` / `cacheLife` に対応する
- `'use cache'` のエントリはデプロイをまたいで残らない（キャッシュキーにビルド ID が入る）。デプロイをまたいで保持したい非 `fetch` のデータは、公式も `unstable_cache` を挙げている
- `fetch()` ベースの同一パラメータリクエストは App Router が同一レンダリング内で自動メモ化するため、`cache()` を追加する前にリクエストのパラメータ統一を先に検討する
- キャッシュするかどうかは、データの鮮度と認可の要件で決める。ユーザー固有のデータや常に最新が必要なデータは、キャッシュしないのが正しい場合がある（#8）

**コード例**:
```tsx
// Good: React cache() — 同一レンダリング内でDBクエリを重複実行しない
import { cache } from 'react';

export const getUser = cache(async (id: string) => {
  return db.user.findUnique({ where: { id } });
});

// Good（Cache Components 無効時 / 既存コード）: unstable_cache — リクエスト間でもキャッシュを保持し、タグで無効化
import { unstable_cache } from 'next/cache';

export const getCachedPosts = unstable_cache(
  async () => db.post.findMany({ orderBy: { createdAt: 'desc' } }),
  ['posts-list'],
  { tags: ['posts'], revalidate: 60 }
);

// Good（Cache Components 有効時）: 'use cache' + cacheLife / cacheTag
import { cacheLife, cacheTag } from 'next/cache';

export async function getPosts() {
  'use cache';
  cacheLife('minutes');
  cacheTag('posts');
  return db.post.findMany({ orderBy: { createdAt: 'desc' } });
}

// Bad（全ユーザーで共有してよく、多少古くてもよいデータの場合）: キャッシュをかけず、毎リクエスト DB を叩く
// ユーザー固有のデータや常に最新が必要なデータなら、キャッシュしないこの形が正しい
export async function getPostsUncached() {
  return db.post.findMany();
}
```

**出典**:
- [Next.js Docs: unstable_cache](https://nextjs.org/docs/app/api-reference/functions/unstable_cache) (Next.js公式 / 2024)
- [Next.js Docs: Caching](https://nextjs.org/docs/app/building-your-application/caching) (Next.js公式 / 2024)
- [React Docs: cache](https://react.dev/reference/react/cache) (React公式 / 2024)
- [Next.js Docs: Migrating to Cache Components — `unstable_cache`](https://nextjs.org/docs/app/guides/migrating-to-cache-components#unstable_cache) (Next.js 公式、`unstable_cache` は引き続き動き、移行は任意) ※2026-10-25 確認
  > "`unstable_cache` continues to work with Cache Components. You can leave it in place and migrate it later."
- [Next.js Docs: use cache — Runtime caching considerations](https://nextjs.org/docs/app/api-reference/directives/use-cache#runtime-caching-considerations) (Next.js 公式、`use cache` はデプロイをまたがない) ※2026-10-25 確認
- [Next.js App Router のキャッシュを完全理解する：fetch メモ化・Data Cache・Full Route Cache の違い](https://zenn.dev/fuelda/articles/nextjs-app-router-cache-guide) (Zenn、App Router の fetch 自動メモ化と cache() の使い分け) ※2026-05-20に実際にfetch成功
- [Next.js App Routerで`cache()`を足す前に、requestを揃える](https://zenn.dev/fuelda/articles/nextjs-app-router-request-memoization-vs-cache) (Zenn、generateMetadata とページコンポーネントでパラメータを統一した実例) ※2026-05-20に実際にfetch成功

**バージョン**: Next.js 15+（`'use cache'` の例は 16+ で `cacheComponents: true`）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 6. `use cache` ディレクティブを `unstable_cache` の後継として把握する（v16 で安定化、16.4 で推奨化）

`use cache` ディレクティブは `unstable_cache` の後継として設計され、v15 の実験的機能を経て v16 で安定化した（`cacheComponents: true` で有効になる）。コンポーネント・関数・ファイルレベルでキャッシュを宣言でき、`cacheLife` / `cacheTag` と組み合わせて細粒度な制御が可能。
配置（末端のデータ取得に貼る）・寿命・無効化の詳細は #7 を参照。

**根拠**:
- `unstable_cache` より宣言的で直感的な構文を提供する。キーは引数から自動で作られる
- コンポーネント単位でのキャッシュが可能になり、Partial Prerendering との親和性が高い
- v16 で `"use cache"` は安定化し、`cacheLife` / `cacheTag` の `unstable_` 接頭辞は不要になった。v15 の `experimental.dynamicIO` / `experimental.useCache` は削除され、トップレベルの `cacheComponents` に置き換わった
- Next.js 16.4 で Cache Components（`'use cache'`）が全 Next.js アプリに推奨され、`create-next-app` の新規アプリでデフォルト有効、Next.js 17 でデフォルト化される予定。「将来の標準」から「現行の推奨」へ位置づけが進んだ

**コード例**:
```tsx
// Good: use cache ディレクティブでコンポーネントをキャッシュ（next.config.ts で cacheComponents: true）
import { cacheLife, cacheTag } from 'next/cache';

export async function CachedProductList() {
  'use cache';
  cacheLife('hours');
  cacheTag('products');

  const products = await db.product.findMany();
  return <ProductGrid products={products} />;
}

// v15 以前 / Cache Components 無効時の書き方（引き続き動く。移行は任意）
import { unstable_cache } from 'next/cache';

const getCachedProductList = unstable_cache(
  async () => db.product.findMany(),
  ['product-list'],
  { tags: ['products'], revalidate: 3600 }
);
```

**出典**:
- [Next.js Docs: use cache directive](https://nextjs.org/docs/app/api-reference/directives/use-cache) (Next.js公式、v16.0.0 で安定化し Cache Components で有効になる) ※2026-10-25 確認
- [Next.js Docs: How to upgrade to version 16 — cacheLife and cacheTag](https://nextjs.org/docs/app/guides/upgrading/version-16#cachelife-and-cachetag) (Next.js 公式、`unstable_` 接頭辞の削除と `experimental.dynamicIO` / `experimental.useCache` の削除) ※2026-10-25 確認
- [Vercel Blog: Our Journey with Caching](https://nextjs.org/blog/our-journey-with-caching) (Next.js公式ブログ / 2024)
- [Next.js 16.4](https://nextjs.org/blog/next-16-4) (Next.js 公式ブログ、全アプリへの推奨化・新規アプリのデフォルト有効化) ※2026-10-06 fetch
  > "Starting with Next.js 16.4, we're excited to recommend Cache Components as the best choice for every Next.js app."

**バージョン**: Next.js 16+（`cacheComponents: true`）。v15 では `experimental.dynamicIO` / `experimental.useCache` と `unstable_cacheLife` / `unstable_cacheTag` だった
**確信度**: 高（公式ドキュメント・公式ブログ）
**最終更新**: 2026-10-25

---

#### 追加根拠 (2026-05-12)

新たに以下の記事で同じプラクティスが言及され、本番運用での具体的知見が追加された:
- [Next.js 16 Cache Components: A Real Migration From a 4.2M-MAU Site](https://dev.to/nilesh_kasar_2b00e7247dd5/nextjs-16-cache-components-a-real-migration-from-a-42m-mau-site-49d8) (dev.to / 2026-05-12) ※2026-05-12に実際にfetch成功

**出典引用**:
> "You explicitly mark a component as cacheable with the new 'use cache' directive."
> ([Next.js 16 Cache Components: A Real Migration From a 4.2M-MAU Site], セクション "What Cache Components Are (and Why They Replaced Fetch-Based Caching)")

4.2M MAU の本番サービスで Next.js 15.3 → 16 へ移行した実例。TTFB p50 が 70% 改善、月次費用が 64% 削減（$4,800→$1,720）。`use cache` 運用での重要な注意点が明らかになった:

1. **キャッシュタグは意図を表す粒度で命名する**: `articles` のような汎用的な単語タグは避け、`articles:latest`・`articles:by-author:{id}` のように用途が分かる名前にする。汎用タグへの `revalidateTag` 1回でキャッシュの 80% が失われた事例あり。

2. **キャッシュキーはユーザー単位ではなく論理セグメント単位に集約する**: ログイン済み/未ログイン・地域バリアントなどを論理セグメント（例: `"member:jp"`）に畳み込み、その値を `"use cache"` 関数の引数として渡すと、ユーザーごとの個別キャッシュを作らずに済む。記事の事例では Edge Function の呼び出しが 96% 減った。記事のコード例は `import { cacheKey } from "next/cache"` と書いているが、`next/cache` に `cacheKey` という export はない（v16.0.0 / v16.4.3 のソースで確認）。キャッシュキーは引数とクロージャで捕捉した値から自動で作られるので、正規化したセグメントを引数にすればよい。公式も、ユーザー ID ではなく言語やカテゴリのように値の種類が少ない次元でキャッシュするよう勧めている。

3. **`use cache` 境界の内部で `cookies()` を呼ばない**: キャッシュ関数の中（呼び出し先のヘルパーを含む）で `cookies()` を読むと `next-request-in-use-cache` エラーになる。動的にレンダリングされるルートでは `next build` を通り、`next start` で実行したときに失敗することがあるので、記事のチームは lint で検出している。Cookie の値は外部コンポーネントで取得し、props として渡す（渡すのはセッションを検証したうえでの userId などの識別子にする。#8）。

**出典（追加）**:
- [Next.js Docs: use cache — Cache keys / Request-time APIs](https://nextjs.org/docs/app/api-reference/directives/use-cache#cache-keys) (Next.js 公式、キャッシュキーの構成要素と、キャッシュ関数内で `cookies()` を読んだときのエラー) ※2026-10-25 確認
  > "Cached functions and components cannot access runtime APIs like `cookies()`, `headers()`, or `searchParams`, and the restriction follows the call stack"
- [Next.js Docs: use cache: remote — Cache key considerations](https://nextjs.org/docs/app/api-reference/directives/use-cache-remote#cache-key-considerations) (Next.js 公式、値の種類が少ない次元でキャッシュする) ※2026-10-25 確認
  > "find the dimension with fewer unique values (category vs. price, language vs. user ID), cache on that dimension, and filter or select the rest in memory."
- [vercel/next.js `packages/next/cache.js`（v16.4.3）](https://github.com/vercel/next.js/blob/v16.4.3/packages/next/cache.js) (Next.js 公式リポジトリ、`next/cache` の export 一覧に `cacheKey` はない) ※2026-10-25 確認

**確信度**: 既存（中）→ 高（4.2M MAU 本番事例で実証済み）

---

### 7. `"use cache"` ディレクティブは末端のデータフェッチ単位で `cacheLife` / `cacheTag` と組み合わせて貼る

v16 の Cache Components 環境では、関数・コンポーネント単位で `"use cache"` を貼って「キャッシュ可能」を明示する。キーは Build ID + Function ID + シリアライズ可能な引数 から自動算出され、`children` などのスロットはキーに入らずパススルーされる（Composition で除外できる）。そのため layout に貼っても配下の page（`children`）はキャッシュされないが、layout 自身が取得するデータと UI は 1 つのエントリにまとまり、寿命とタグを個別に分けられなくなる。配置はページ全体や layout には貼らず、**できるだけデータフェッチに近い末端**に貼る。寿命は `cacheLife`、無効化のためのタグは `cacheTag`、無効化は `updateTag`（即時反映）または `revalidateTag(tag, "max")`（stale-while-revalidate）で行う。

**根拠**:
- v16 ではデフォルトで暗黙キャッシュしない方針に変わったため、再利用可能なデータは明示的に `"use cache"` でオプトインする必要がある
- `cacheTag` は複数引数で複数タグを一度に登録できる（例: `cacheTag("posts", `post:${slug}`)` で 2 つのタグを付与）
- v16 では `revalidateTag(tag)` の 1 引数版は非推奨。profile（`"max"` 推奨）を渡すか `updateTag` に置き換える必要がある。`updateTag` は更新を即時反映、`revalidateTag(..., "max")` は stale-while-revalidate セマンティクス
- `cacheLife('seconds')` や `expire` が 5 分未満の短命キャッシュは **自動的に prerender から除外され「dynamic hole」として扱われる**ため、`<Suspense>` で囲んでフォールバックを提供する設計が必要
- layout に `"use cache"` を貼っても、`children` として渡る配下の page はキャッシュされない（公式）。キャッシュされるのは layout 自身が生成する部分で、layout 内で取得するナビゲーションやバナーなどが同じ寿命・同じ無効化単位にまとまる。データ取得ごとに寿命とタグを分けたい場合は、取得関数の側に貼る
- **`cacheComponents` は全ルートに効くビルド時のゲートである**。`cacheComponents: true` を有効にすると、ルート内の各データは「ストリームする」「キャッシュする」「blocking にする」のいずれかを明示しないと**ビルド自体が失敗する**ことがある。特に `params` を `<Suspense>` で囲まずに参照する最小構成のルートでもこのゲートに引っかかる。また `dynamic` / `revalidate` / `fetchCache` を export しているルートはエラーになるので、有効化した時点で全ルートの設定の置き換えが必要になる
- ただし、全ルートを一度に直す必要はない。まだ対応していないルートには `export const instant = false` を付けて検証から外し（codemod `cache-components-instant-false` で一括付与できる）、アプリ全体をビルド・稼働させたうえで、ルートごとに `instant = false` を外して `use cache` / `<Suspense>` で対応する段階移行が公式の手順。`new Date()` / `Math.random()` などの同期 IO によるビルドエラーは `instant = false` では解消しないので、先に対応する

**cacheComponents 有効化時のビルド失敗例**:
```typescript
// next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  cacheComponents: true,
  partialPrefetching: true,
};

export default nextConfig;
```

```tsx
// app/products/[id]/page.tsx
import { Suspense } from "react";

// Bad: params を Suspense で囲まずに参照 → ビルドエラー
// "Next.js encountered uncached or runtime data during prerendering.
//  `params`...accessed outside of `<Suspense>`"
export default async function ProductPage({
  params,
}: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const product = await getProduct(id);
  return <h1>{product.name}</h1>;
}

// Good: Suspense で囲み、Partial Prerendering に委ねる
export default function ProductPage({
  params,
}: { params: Promise<{ id: string }> }) {
  return (
    <Suspense fallback={<ProductSkeleton />}>
      <ProductDetail paramsPromise={params} />
    </Suspense>
  );
}

async function ProductDetail({ paramsPromise }: { paramsPromise: Promise<{ id: string }> }) {
  const { id } = await paramsPromise; // Suspense の内側で await する
  const product = await getProduct(id);
  return <h1>{product.name}</h1>;
}

// 移行中: まだ対応できないルートは検証から外す（ルートを動的にはしない）
// export const instant = false;
```

**コード例**:
```typescript
// Good: app/_lib/dal/post.ts
// 末端のデータフェッチ関数に貼り、cacheLife / cacheTag を併用
import "server-only";
import { cacheLife, cacheTag } from "next/cache";
import { notFound } from "next/navigation";

export async function getPublicPost(slug: string) {
  "use cache";
  cacheLife("hours");
  cacheTag("posts", `post:${slug}`); // 複数タグを 1 回で登録

  const res = await fetch(`${API}/posts/${slug}`);
  if (res.status === 404) notFound();
  if (!res.ok) throw new Error("failed");
  return toPostDTO(await res.json());
}

// Server Action からの無効化（v16: 第 2 引数必須）
"use server";
import { revalidateTag, updateTag } from "next/cache";
import { unauthorized } from "next/navigation";
import { verifySession } from "@/app/_lib/dal/auth";
import { publishPost } from "@/app/_lib/dal/post.mutations"; // 権限の確認は DAL で行う

export async function publishPostAction(slug: string) {
  const session = await verifySession();
  if (!session) unauthorized(); // experimental.authInterrupts: true が前提
  await publishPost(slug);

  // 即時反映したい場合
  updateTag(`post:${slug}`);
  // 多少遅延しても良い場合
  revalidateTag("posts", "max");
}

// Bad: layout.tsx 全体に "use cache" を貼り、layout 内のデータ取得をまとめて 1 つの寿命にする
// （children の page はパススルーされキャッシュされないが、ナビゲーションとバナーを別々に無効化できない）
export default async function Layout({ children }: { children: React.ReactNode }) {
  "use cache";
  const nav = await getNavigation();
  const banner = await getCampaignBanner();
  return (
    <div>
      <Nav items={nav} />
      <Banner data={banner} />
      {children}
    </div>
  );
}

// Good: 取得関数ごとに "use cache" を貼り、寿命とタグを分ける
async function getCampaignBanner() {
  "use cache";
  cacheLife("minutes");
  cacheTag("campaign-banner");
  return toBannerDTO(await db.banner.findFirst({ where: { active: true } }));
}

// Bad: revalidateTag の 1 引数版（v16 で非推奨）
revalidateTag("posts"); // 将来削除予定
```

**アンチパターン**:
- `layout.tsx` 全体を `"use cache"` にして、layout 内のデータ取得の寿命・タグを分けられなくする（`children` はキャッシュされないので、配下の page をキャッシュする目的では効果もない）
- `cacheComponents: true` を「全ルートを直すまで有効化できない」と考えて導入を先送りする（`instant = false` で段階移行できる）
- ユーザー固有データに `"use cache"` を貼って他ユーザーに漏れる（→ Rule #8 / `"use cache: private"` か Suspense + 引数渡しを使う）
- `revalidateTag(tag)` の 1 引数版を使い続ける（v16 で非推奨）
- 短命キャッシュ（5 分未満の `expire`）を `<Suspense>` で囲まずに配置し、ポップコーン UI / CLS を引き起こす

**出典引用**:
> "Next.js encountered uncached or runtime data during prerendering. `params`...accessed outside of `<Suspense>`"
> ([I Turned On Cache Components in Next.js 16.3. It Refused to Build My Simplest Page.](https://dev.to/shubhradev/i-turned-on-cache-components-in-nextjs-163-it-refused-to-build-my-simplest-page-3ak0), セクション本文) ※2026-08-19に実際にfetch成功

> "It's not gradual. It's a build gate."
> ([I Turned On Cache Components in Next.js 16.3. It Refused to Build My Simplest Page.](https://dev.to/shubhradev/i-turned-on-cache-components-in-nextjs-163-it-refused-to-build-my-simplest-page-3ak0), セクション本文) ※2026-08-19に実際にfetch成功

**出典**:
- [Next.jsの考え方 / 3.4 "use cache" ディレクティブ](https://zenn.dev/akfm/books/nextjs-basic-principle)
- [Next.js Docs: use cache — Caching a route segment's output](https://nextjs.org/docs/app/api-reference/directives/use-cache#caching-a-route-segments-output-with-use-cache) (Next.js 公式、layout に貼っても children はキャッシュされない) ※2026-10-25 確認
  > "A cached `layout` does not cache the `children` it renders, since slots pass through without affecting its entry."
- [Next.js Docs: Migrating to Cache Components — Adopting incrementally](https://nextjs.org/docs/app/guides/migrating-to-cache-components#adopting-incrementally) (Next.js 公式、`instant = false` と codemod によるルート単位の段階移行) ※2026-10-25 確認
- [I Turned On Cache Components in Next.js 16.3. It Refused to Build My Simplest Page.](https://dev.to/shubhradev/i-turned-on-cache-components-in-nextjs-163-it-refused-to-build-my-simplest-page-3ak0) (dev.to、`cacheComponents: true` + `partialPrefetching: true` 有効化時に `params` を Suspense 外で参照するとビルドが失敗する実機検証) ※2026-08-19 fetch

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**バージョン**: Next.js 16+（Cache Components 前提・`cacheComponents: true`）
**確信度**: 高（公式ドキュメントで確認）
**最終更新**: 2026-10-25

---

### 8. ユーザー固有データは「Suspense 境界 + 引数渡し」を優先し、`"use cache: private"` は最終手段

ユーザー固有データ（セッション、自分のダッシュボードなど）は原則「キャッシュしない」のが基本。まず `<Suspense>` で包んで動的にレンダリングする。`"use cache"` 配下に動的値を持ち込みたい場合は、`"use cache"` 関数の外側、かつ `<Suspense>` の内側で `cookies()` / `headers()` を読み、セッションを検証したうえで、検証済みの userId などの識別子だけを引数として `"use cache"` 関数に渡してキャッシュキーに含める。生のセッション Cookie やトークンは引数に渡さない。
`"use cache"` を付けた関数は export せず、セッションを検証する入口の関数（DAL）からだけ呼ぶ。`"use cache: private"`（クライアント保存）はコンプライアンス要件や引数渡しへのリファクタが現実的でない場合のみ採用する最終手段。

**根拠**:
- `"use cache"` 配下で `cookies()` / `headers()` を直接呼ぶとエラー（Cache Components 環境）。また Cache Components では `<Suspense>` の外で `cookies()` を読むとビルドエラーになるので、読む場所は「`"use cache"` の外側、かつ `<Suspense>` の内側」になる
- 動的値を引数として渡せばその値がキャッシュキーになるため、ユーザー横断で共有可能なものは横断で再利用され、ユーザーごとに分かれるものはユーザーごとに分かれる
- キャッシュ関数の引数と `cacheTag` の値はハッシュ化されず、平文でキャッシュキー・タグとして保存される（リモートキャッシュにもそのまま送られる）。公式は、キーとタグには userId のような安定した識別子を使い、トークン・パスワード・生のメールアドレスなどの秘密や個人情報を入れないよう求めている
- キャッシュ関数を export すると、別の userId を渡して他人のデータを取得できてしまう。公式の例も、キャッシュ関数は export せず、セッションを検証する入口の関数から呼んでいる
- 保存先は 4 種類（ストレージ選択の基準）。レイテンシは Medium 記事の著者が `docker compose up` で起動するローカルのデモ（SQLite + 500 ms の遅延を入れたバックエンド）で測った、ヒット時のデータ取得時間:
  - `"use cache"`（既定のハンドラーはインメモリ LRU）: Next.js のプロセスごとに分かれ、複数のサーバーやコンテナの間では共有されず、再起動で消える。サーバーレスではリクエスト間でほとんど再利用されない。記事のデモでは約 1〜2 ms
  - `"use cache: remote"`: `cacheHandlers.remote` に設定したハンドラー（Redis 等。ホスティング事業者が用意する場合もある）に保存し、全インスタンスで共有する。ハンドラーを設定しないと、こちらもインメモリ LRU になる。Redis のような外部ストアなら再起動では消えないが、キャッシュキーにビルド ID（または `deploymentId`）が入るのでデプロイをまたいでは使えない。記事のデモでは約 15〜40 ms（Redis への往復 1 回分）。公式は、レート制限のある API、遅いバックエンドの保護、高コストな処理などで、コストとレイテンシに見合う場合に使うとしている
  - `"use cache: <name>"`: `cacheHandlers` に追加した名前付きハンドラーを使う
  - `"use cache: private"`: サーバーには保存せず、ブラウザのメモリにだけ置く（`cacheHandlers` ではカスタマイズできない）。最終手段
- フルルート ISR（HTML ごとキャッシュ）は、記事のデモでは、データをキャッシュしたまま重い詳細ページを再レンダリングすると約 1.5〜2 秒かかり、キャッシュ済みの HTML を返すとその 20〜35 倍速かった。レンダリングとシリアライズがボトルネックの重いページの場合の値
- `"use cache: private"` を選ぶ前に、引数渡しパターンが本当に成立しないかを必ず検討する。多くのケースは引数渡しで足りる
- `"use cache: private"` の結果は `cacheLife` の stale 時間（5 分以上など）の間ブラウザのメモリに残りうる（Partial Prefetching 有効時）。共有端末でログアウト後に前ユーザーの値が見えないよう、ログアウト後は `window.location.href` でフルリロードする

**コード例**:
```tsx
// Good: app/_lib/dal/profile.ts
import "server-only";
import { cacheLife, cacheTag } from "next/cache";
import { unauthorized } from "next/navigation";
import { verifySession } from "./auth";

// export する入口: "use cache" の外でセッションを検証し、検証済みの userId だけを渡す
export async function getMyProfile() {
  const session = await verifySession(); // 内部で cookies() を読む
  if (!session) unauthorized(); // experimental.authInterrupts: true が前提
  return getProfileByUserId(session.userId);
}

// キャッシュ本体: export しない（任意の userId を渡して呼ばれないようにする）
async function getProfileByUserId(userId: string) {
  "use cache";
  cacheLife("minutes");
  cacheTag(`profile:${userId}`); // キーとタグは平文で保存されるので、秘密ではない識別子だけを使う
  const profile = await db.profile.findUnique({ where: { userId } });
  return toProfileDTO(profile);
}
```

```tsx
// Good: app/profile/page.tsx — Suspense の内側で読む（Suspense の外で cookies() を読むとビルドエラー）
import { Suspense } from "react";
import { getMyProfile } from "@/app/_lib/dal/profile";

export default function Page() {
  return (
    <Suspense fallback={<Skeleton />}>
      <ProfileContent />
    </Suspense>
  );
}

async function ProfileContent() {
  const profile = await getMyProfile();
  return <div>{profile.displayName}</div>;
}
```

```tsx
import { cookies } from "next/headers";

// Bad: 生のセッション Cookie を "use cache" の引数にする
// → 引数は平文でキャッシュキーに保存される。セッションの検証もしていない
async function CachedContent({ sessionId }: { sessionId?: string }) {
  "use cache";
  const data = await fetchUserData(sessionId);
  return <div>{data}</div>;
}

// Bad: "use cache" 配下で cookies() を直接呼ぶ → エラー
async function BadContent() {
  "use cache";
  const sessionId = (await cookies()).get("session")?.value; // NG
  // ...
}

// Bad: ユーザー固有データに普通の "use cache" を貼る → 他ユーザーに漏れる
async function MyDashboard() {
  "use cache"; // 個人データなのに横断キャッシュされる
  const me = await getCurrentUser();
  return <Dashboard data={me} />;
}
```

**アンチパターン**:
- ユーザー固有データに引数なしの `"use cache"` を貼り、別ユーザーにキャッシュ内容が漏れる
- 引数渡しパターンを検討せずに `"use cache: private"` を最初の選択肢にする
- `"use cache"` 配下で `cookies()` / `headers()` を直接呼ぶ
- セッション Cookie・トークン・メールアドレスなどをそのまま `"use cache"` の引数や `cacheTag` に使う（平文で保存される）
- `"use cache"` を付けた関数を export し、任意の userId で呼べるようにする
- ユーザーごとの個別キャッシュを作って Edge Function 呼び出し回数を爆発させる（横断で集約可能なものは引数の正規化で集約する）

**出典引用**:
> "the cache is **shared by every instance** and survives restarts"
> ([The Caching Playbook: SSR, use cache, Redis and ISR in Next.js 16](https://medium.com/@rezamoosavi.kntu/the-caching-playbook-ssr-use-cache-redis-and-isr-in-next-js-16-30ea8d540d13), セクション "use cache: remote — shared between instances") ※2026-06-05に実際にfetch成功

**出典**:
- [Next.jsの考え方 / 3.5 ユーザー固有データの扱いと保存先の使い分け](https://zenn.dev/akfm/books/nextjs-basic-principle)
- [The Caching Playbook: SSR, use cache, Redis and ISR in Next.js 16](https://medium.com/@rezamoosavi.kntu/the-caching-playbook-ssr-use-cache-redis-and-isr-in-next-js-16-30ea8d540d13) (Medium、`use cache: remote` vs インメモリのレイテンシ比較・ISR の効果測定。数値は著者のローカルのデモでの実測) ※2026-06-05fetch、2026-10-25 に本文の数値と測定条件を再確認
- [Next.js Docs: use cache: remote](https://nextjs.org/docs/app/api-reference/directives/use-cache-remote) (Next.js 公式、インメモリの限界、全インスタンスでの共有、使うべき場面、デプロイをまたいで保持されない点) ※2026-10-25 確認
  > "The `'use cache: remote'` directive lets you declaratively specify that a cached output should be stored in a remote cache instead of in-memory, providing durable caching shared across all server instances."
- [Next.js Docs: cacheHandlers](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheHandlers) (Next.js 公式、既定のインメモリキャッシュはプロセスごと、未設定時は `remote` もインメモリ LRU、名前付きハンドラー、`use cache: private` はハンドラーを使わない) ※2026-10-25 確認
  > "The default in-memory cache is isolated to each Next.js process. If you're running multiple servers or containers, each instance will have its own cache that isn't shared with others and is lost on restart."
- [`use cache: private` のキャッシュはブラウザに残ることもある](https://zenn.dev/chot/articles/362b7a2420ef1a) (Zenn、stale 時間中のブラウザ保持とログアウト時のフルリロード) ※2026-10-02に実際にfetch成功
  > "The client router can keep the rendered output in browser memory for the stale time configured with cacheLife."
- [Next.js Docs: How to implement authentication with Cache Components](https://nextjs.org/docs/app/guides/authentication-with-cache-components) (Next.js 公式、Suspense の内側でセッションを読み、userId だけを export しないキャッシュ関数に渡す。キーとタグは平文で保存される) ※2026-10-25 確認
  > "A cached function's arguments and captured variables are serialized into its cache key, and `cacheTag` values are stored as written. Neither is hashed"
- [Next.js Docs: use cache](https://nextjs.org/docs/app/api-reference/directives/use-cache) (Next.js 公式、cookies / headers はキャッシュの外で読んで引数で渡すのが推奨。`use cache: private` はリファクタできない場合の手段) ※2026-10-25 確認

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**バージョン**: Next.js 16+（Cache Components 前提）
**確信度**: 高（中核の主張と保存先ごとの性質は公式ドキュメントで確認。レイテンシの数値は単一記事のローカル実測）
**最終更新**: 2026-10-25

---

### 9. 大量ページ数のサイトは全ページ事前ビルドをやめ、ISR + 空の `generateStaticParams` で「初回アクセス時に生成」に倒す（Cache Components 無効時）

Cache Components 無効時の構成を前提にする。数十万ページ規模の静的サイトでは、ビルド時に全ページを生成すると使われないページの生成コストを毎回払うことになる。`generateStaticParams` を空配列にして事前ビルド対象を0件にし、ISR（`revalidate` 付き）で初回リクエスト時にオンデマンド生成・以降はキャッシュから返す構成に倒す。ビルドの並列度（`experimental.cpus`）を上げても、ボトルネックが計算ではなくメモリ帯域・データロードの場合は速くならず、メモリ使用量だけが増えて悪化することがある。
Cache Components 有効時は、空配列を返すと `empty-generate-static-params` のビルドエラーになり、`export const revalidate` も使えない。実在するパラメータを 1 件以上返し（返さなかったパスはリクエスト時に生成される）、寿命は `'use cache'` + `cacheLife` で指定する。

**根拠**:
- 24万ページのうち実際にアクセスされるのはごく一部で、全件事前ビルドは「使われないページのために毎回コストを払う」構成になる
- 並列化はボトルネックが計算（CPU）のときにのみ有効。メモリ帯域・データロードがボトルネックの構成では並列度を上げるほどメモリ使用量が増え、OOM やビルド遅延を招く
- Full Route Cache のオンメモリ上限は `cacheMaxMemorySize` で調整でき、ISR 方式でもキャッシュヒット率を制御できる
- 全件を一度に読み込むデータ取得は、集計クエリ1本に置き換えることでメモリ使用量を抑えられる
- Cache Components 有効時は、`generateStaticParams` が 1 件以上返さないとビルドが失敗する（空配列は `empty-generate-static-params`）。`generateStaticParams` を削除するとそのルートは ISR の対象から外れ、`use cache` でデータをキャッシュしていてもリクエストごとにレンダリングされる（公式の移行ガイド）
- **例外**: `output: "export"`（静的エクスポート）ではサーバーがないため、空配列を返すと `Page "/x/[id]" is missing "generateStaticParams()"` でビルドが失敗する（関数は存在するのに「無い」と報告される誤解を招くメッセージ。Next.js 16.2.6 で確認）。ISR + 空配列は静的エクスポートでは使えず、データが0件になり得る場合はプレースホルダーのパスを1件返して `notFound()` に倒し、0件データでのビルドを一度は通しておく
- `generateStaticParams` を**定義しないまま省略する**のは、空配列を明示的に返すのとは異なる挙動になりうる。ビルド時エラーも警告も出ないまま ISR が効かず全体が動的レンダリングにフォールバックすることがあり、`searchParams` を読むコンポーネントを含む場合は特に気づきにくい。`curl -sI <url> | grep -i cache-control` でレスポンスヘッダを見て ISR が実際に効いているか確認する

**コード例**:
```js
// next.config.js
module.exports = {
  experimental: {
    cpus: 1, // メモリ律速のビルドでは並列度を上げるとむしろ悪化する
  },
  cacheMaxMemorySize: 50 * 1024 * 1024, // Full Route Cache のオンメモリ上限（バイト）
};
```

```tsx
// Cache Components 無効時: app/[slug]/page.tsx
export const revalidate = 3600; // ISR: 1時間ごとに再検証

export async function generateStaticParams() {
  // Bad: 全24万件をビルド時に生成 → メモリ枯渇・ビルド時間増大
  // return await getAllSlugs();

  // Good: 空配列を返し、ビルド時は生成しない。初回リクエスト時にオンデマンド生成してキャッシュ
  return [];
}
```

```tsx
// Cache Components 有効時: app/[slug]/page.tsx — 空配列はビルドエラー、revalidate は export できない
import { cacheLife } from 'next/cache';

export async function generateStaticParams() {
  const slugs = await getPopularSlugs(1); // 実在するパラメータを 1 件以上返す。残りはリクエスト時に生成される
  return slugs.map((slug) => ({ slug }));
}

async function getPage(slug: string) {
  'use cache';
  cacheLife('hours'); // 旧モデルの revalidate = 3600 に相当する寿命
  return db.page.findUnique({ where: { slug } });
}
```

**アンチパターン**:
- ページ数が数万〜数十万件規模のサイトで `generateStaticParams` に全件を渡し、ビルド時に全ページを事前生成する
- ビルドが遅い原因を計算コストと決めつけて `experimental.cpus` を安易に増やす（メモリ律速の場合は逆効果）
- `generateStaticParams` を「空配列を返す」のではなく丸ごと省略し、ISR が効いていないことに気づかないまま運用する

**出典引用**:
> "24万ページのうち、実際に人やクローラーが見るのはごく一部です。全部を先に作るのは、使われないページのために毎回1時間を払っているのと同じでした。"
> ([24万ページのNext.jsを、RAM 4GBのVPS 1台で動かす](https://zenn.dev/shiorisuimoku/articles/nextjs-240k-pages-on-4gb-vps), セクション "全ページ事前ビルドをやめる") ※2026-08-02に実際にfetch成功

> "並列化で速くなるのは計算がボトルネックのときで、ここはメモリ帯域とデータロードがボトルネックでした。"
> ([24万ページのNext.jsを、RAM 4GBのVPS 1台で動かす](https://zenn.dev/shiorisuimoku/articles/nextjs-240k-pages-on-4gb-vps), セクション "ビルドの並列度を上げたら、遅くなった") ※2026-08-02に実際にfetch成功

**出典**:
- [24万ページのNext.jsを、RAM 4GBのVPS 1台で動かす](https://zenn.dev/shiorisuimoku/articles/nextjs-240k-pages-on-4gb-vps) (Zenn、`generateStaticParams`/`experimental.cpus`/`cacheMaxMemorySize` の実測検証) ※2026-08-02に実際にfetch成功
- [revalidate を書いてもISRされていなかった](https://zenn.dev/figurehub/articles/nextjs-isr-never-worked) (Zenn figurehub、`generateStaticParams` 省略時の黙った動的化フォールバックと `curl` でのキャッシュ検証手順) ※2026-08-23 fetch
- [Next.jsのoutput: exportは、generateStaticParamsが空配列だとビルドを落とす](https://zenn.dev/gzer0/articles/nextjs-export-empty-generatestaticparams) (Zenn gzer0、静的エクスポートでは空配列がビルドエラーになる例外とプレースホルダー回避策) ※2026-10-13 fetch
  > "空配列を返したときのエラーは「missing」、つまり関数が無いという言い方です。" (セクション "空配列でも「関数が無い」と言われる")
- [Next.js Docs: Migrating to Cache Components — `generateStaticParams` must return at least one param](https://nextjs.org/docs/app/guides/migrating-to-cache-components#generatestaticparams-must-return-at-least-one-param) (Next.js 公式、Cache Components 有効時は空配列が `empty-generate-static-params` エラー。`revalidate` は `cacheLife` に置き換える) ※2026-10-25 確認
  > "Returning an empty array now errors."

**バージョン**: Next.js 15+ App Router（空配列 + `revalidate` は Cache Components 無効時のみ）
**確信度**: 中（公式APIの検証記事、パターン1c：単独ソース。Cache Components 有効時の挙動は公式ドキュメントで確認）
**最終更新**: 2026-10-25

---

### 10. `instant()` Playwright ヘルパーでナビゲーションの体感速度を決定的にテストし、Suspense 境界の配置を検証する

「ナビゲーションが速く感じる」というあいまいな性質は、手動確認だけでは回帰検知できない。`@next/playwright` の `instant()` ヘルパーはナビゲーションを一時停止し、ネットワークリクエストなしで表示されているUI部分をアサートできるため、体感速度を CI で回帰検知可能な決定的テストに変換できる。個人化された動的データ取得を `<Suspense>` 境界の内側に移すことで、境界の外側の UI をナビゲーション前にプリレンダリングしてブラウザにキャッシュでき、テストが green になる。

**根拠**:
- 全ページ静的プリレンダリングは個人化データを持つ動的アプリには非現実的、フルプリフェッチはサーバー負荷が高い。Next.js 16.3 はこの中間として「ユーザーがアプリを操作している間に動的コンテンツを事前レンダリングし、ブラウザ側に丸ごとキャッシュする」方式（Partial Prerendering の実行時版）を提供する
- `instant()` はナビゲーションを一時停止し、ネットワークリクエストが発生せずに見えている部分をアサートするため、「速い」という主観的な体感を CI 上のテストとして固定できる
- Vercel 社内の v0（AI コーディングプラットフォーム）への適用では、「検証可能なゴール（失敗するテスト）→ ガードレール（実証済みパターンを持つ Skill）→ 実フィードバック（本番メトリクス）」の3要素でコーディングエージェントによる自動修正ループが成立した
- 多くの修正は「動的データ取得を Suspense 境界の下に移動するだけ」で完了しており、既存の「Suspense 境界 = Dynamic Content の境界」という設計判断（本ファイル Rule #8 参照）の延長線上にある

**コード例**:
```ts
// e2e/navigation.spec.ts - ネットワークなしで表示される部分をアサート
import { expect, test } from '@playwright/test';
import { instant } from '@next/playwright';

test('navigating to the chats page is instant', async ({ page }) => {
  await page.goto('/');

  await instant(page, async () => {
    await page.getByRole('link', { name: 'Chats' }).click();
    await expect(page.getByTestId('app-title')).toBeVisible();
  });
});
```

```tsx
// Good: 動的データ取得を Suspense 境界の下に移動し、外側のシェルは即座に表示
export default function WorkspacePage() {
  return (
    <SettingsPageLayout>
      <SettingsHeader title="Workspace" />
      <Suspense fallback={<WorkspaceSkeleton />}>
        <WorkspaceContent />
      </Suspense>
    </SettingsPageLayout>
  );
}

async function WorkspaceContent() {
  const session = await getServerSession();
  const team = await fetchTeam(session);
  return <TeamSettings team={team} />;
}

// Bad: ページ全体を async にし、動的データ取得がナビゲーション全体をブロックする
export default async function WorkspacePage() {
  const session = await getServerSession();
  const team = await fetchTeam(session);
  return (
    <SettingsPageLayout>
      <SettingsHeader title="Workspace" />
      <TeamSettings team={team} />
    </SettingsPageLayout>
  );
}
```

**アンチパターン**:
- 「速くなった気がする」を手動確認だけで済ませ、リグレッションテストとして CI に残さない
- 動的データ取得を Suspense 境界の外側に置いたまま、キャッシュ機構だけでナビゲーション高速化を狙う
- テストなしでエージェントに「ナビゲーションを速くして」と依頼し、達成判定を主観に委ねる

**出典引用**:
> "This helper lets you write tests that pause a navigation and assert which parts of the UI are instantly visible to the user, without any network request."
> ([Making navigations instant in v0](https://nextjs.org/blog/making-v0-navigations-instant), セクション "A test for \"fast\"") ※2026-08-07に実際にfetch成功

> "A verifiable goal. ... Guardrails. ... Real feedback. Production metrics that confirm a successful loop had the intended effect."
> ([Making navigations instant in v0](https://nextjs.org/blog/making-v0-navigations-instant), セクション "Using loops to make v0 instant") ※2026-08-07に実際にfetch成功

**出典**:
- [Making navigations instant in v0](https://nextjs.org/blog/making-v0-navigations-instant) (Next.js 公式ブログ、2026-08-06公開) ※2026-08-07に実際にfetch成功

**バージョン**: Next.js 16.3+, `@next/playwright`
**確信度**: 高（Next.js 公式ブログ）
**最終更新**: 2026-08-07

---

### 11. 静的であるべきルートには `ensureStatic` を宣言し、動的コンポーネントの混入をビルドで失敗させる

`'use cache'` は静的シェルと動的コンテンツを 1 レスポンスにストリームできる反面、ECサイト・マーケティングページ・ブログのように「常に静的」であるべきページでも、動的コンポーネントが 1 つ紛れ込むだけでコストと性能特性が崩れる。Next.js 16.4 の `export const ensureStatic` をルートまたはレイアウトに宣言し、意図しないリクエスト時レンダリングをビルド失敗として検知する。

**根拠**:
- `"navigation"`（最も厳格: ナビゲーション全体が静的）/ `"prefetch"`（明示プリフェッチで静的コンテンツのみ取得）/ `"shell"`（初回発見時に静的コンテンツのみ取得）の 3 段階で保証範囲を選べる
- ルートレイアウトに `"navigation"` を置けばサイト全体のナビゲーションが静的であることを保証でき、動的コンテンツが必要になったページだけ入れ子レイアウト側へ宣言を押し下げられる
- 回帰を実行時の性能劣化ではなくビルド時に検知できるため、サーバーコストの意図しない増加を防げる
- 動的ページが中心のアプリでは不要。静的を維持したいルートにだけ使う

**コード例**:
```tsx
// Good: ブログ記事ルートを常に静的として保証する（静的なコンポーネントだけで構成する）
export const ensureStatic = 'navigation';

export default function Page() {
  return <Content />;
}

// Bad（ビルドで検知される）: ensureStatic を宣言したルートに動的コンポーネントが混入している
export default function Page() {
  return (
    <>
      <UserAvatar /> {/* cookies() 等を読む動的コンポーネント → ensureStatic によりビルドが失敗する */}
      <Content />
    </>
  );
}

// Good: ルートレイアウトでサイト全体のナビゲーションを静的に保証する
// app/layout.tsx
export const ensureStatic = 'navigation';
```

**アンチパターン**:
- 静的を前提としたマーケティング/ブログ/EC 一覧ページで宣言を置かず、動的コンポーネント混入に本番のコスト増で気づく
- 動的コンテンツが必要なページを含むレイアウトに `"navigation"` を置いてビルドを壊す（必要なページ配下のレイアウトへ押し下げる）

**出典引用**:
> "In Next.js 16.4, `ensureStatic` offers a simple way to guarantee that a route's shell, prefetch, or full navigation is static."
> ([Next.js 16.4](https://nextjs.org/blog/next-16-4), セクション "Ensuring shells, prefetches, or pages are static") ※2026-10-06に実際にfetch成功

**出典**:
- [Next.js 16.4](https://nextjs.org/blog/next-16-4) (Next.js 公式ブログ、2026-10-06公開) ※2026-10-06に実際にfetch成功

**バージョン**: Next.js 16.4+（`cacheComponents: true` 前提）
**確信度**: 高（Next.js 公式ブログ）
**最終更新**: 2026-10-06

---
