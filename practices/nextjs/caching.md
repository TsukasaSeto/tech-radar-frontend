# Next.js キャッシュのベストプラクティス

## ルール

### 1. Next.js の4層キャッシュを理解して使い分ける

Next.js 14+ には4つのキャッシュ層がある。それぞれの役割と無効化方法を理解する。

**根拠**:
- 各キャッシュ層は独立しており、適切な層を無効化しないと古いデータが表示される
- デフォルトが変わったバージョン（13→15）があるため明示的な指定が安全

**コード例**:
```
1. Request Memoization - 同一レンダリング内の重複 fetch を自動デデュープ
2. Data Cache - fetch の cache オプションで制御
3. Full Route Cache - 静的ルートの HTML をサーバーにキャッシュ
4. Router Cache - ブラウザのセッション中に RSC ペイロードをメモリにキャッシュ
```

**出典**:
- [Next.js Docs: Caching Overview](https://nextjs.org/docs/app/building-your-application/caching) (Next.js公式)

**バージョン**: Next.js 14+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. `revalidateTag` と `revalidatePath` でオンデマンド再検証する

データ更新時はタグベースまたはパスベースのオンデマンド再検証を使う。

**根拠**:
- タグベースはデータの種類に対応して複数ページを一括無効化できる
- Server Actions と組み合わせることでミューテーション後の自動更新が可能

**コード例**:
```tsx
'use server';
import { revalidateTag, revalidatePath } from 'next/cache';

export async function createUser(formData: FormData) {
  await db.user.create({ data: Object.fromEntries(formData) });
  revalidateTag('users');
  revalidatePath('/users');
}
```

**出典**:
- [Next.js Docs: On-demand Revalidation](https://nextjs.org/docs/app/building-your-application/data-fetching/incremental-static-regeneration#on-demand-revalidation-with-revalidatetag) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. `unstable_cache` で ORM クエリ等の非 fetch 呼び出しをキャッシュする

`fetch` を使わない ORM クエリや外部 SDK の呼び出しは `unstable_cache` でキャッシュする。

**コード例**:
```tsx
import { unstable_cache } from 'next/cache';

const getCachedUsers = unstable_cache(
  async () => db.user.findMany(),
  ['users-list'],
  { tags: ['users'], revalidate: 3600 }
);
```

**出典**:
- [Next.js Docs: unstable_cache](https://nextjs.org/docs/app/api-reference/functions/unstable_cache) (Next.js公式)

**バージョン**: Next.js 14+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. `dynamic` エクスポートで静的/動的レンダリングを明示制御する

`export const dynamic` でルートの動的レンダリングを明示的に制御する。

**コード例**:
```tsx
// 常に動的レンダリング
export const dynamic = 'force-dynamic';

// cookies() や headers() の使用で自動的に動的になる
export default async function Page() {
  const cookieStore = cookies();
  const token = cookieStore.get('auth-token');
  return <Dashboard />;
}
```

**出典**:
- [Next.js Docs: Route Segment Config](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 5. `unstable_cache` と React `cache()` を目的に応じて使い分ける

`unstable_cache` はリクエストをまたいだ永続的なキャッシュに、React の `cache()` は同一レンダリングパス内のリクエスト重複除去（メモ化）に使う。用途が異なるため、混同せず適切に使い分ける。

**根拠**:
- `cache()` はリクエストのライフタイム内のみ有効で、再検証制御はできない
- `unstable_cache` は Data Cache に書き込み、`revalidateTag` 等で無効化できる
- Next.js 15 では `use cache` ディレクティブが `unstable_cache` の後継として提供されている

**コード例**:
```tsx
// Good: React cache() — 同一レンダリング内でDBクエリを重複実行しない
import { cache } from 'react';

export const getUser = cache(async (id: string) => {
  return db.user.findUnique({ where: { id } });
});

// Good: unstable_cache — リクエスト間でもキャッシュを保持し、タグで無効化
import { unstable_cache } from 'next/cache';

export const getCachedPosts = unstable_cache(
  async () => db.post.findMany({ orderBy: { createdAt: 'desc' } }),
  ['posts-list'],
  { tags: ['posts'], revalidate: 60 }
);

// Bad: fetch を使わない処理に何もキャッシュをかけない
export async function getPostsUncached() {
  // リクエストごとに毎回DBを叩く
  return db.post.findMany();
}
```

**出典**:
- [Next.js Docs: unstable_cache](https://nextjs.org/docs/app/api-reference/functions/unstable_cache) (Next.js公式 / 2024)
- [Next.js Docs: Caching](https://nextjs.org/docs/app/building-your-application/caching) (Next.js公式 / 2024)
- [React Docs: cache](https://react.dev/reference/react/cache) (React公式 / 2024)

**バージョン**: Next.js 15+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 6. `use cache` ディレクティブ（Next.js 15 実験的機能）を将来の標準として把握する

Next.js 15 で導入された実験的な `use cache` ディレクティブは、`unstable_cache` の後継として設計されている。コンポーネント・関数・ファイルレベルでキャッシュを宣言でき、`cacheLife` / `cacheTag` API と組み合わせて細粒度な制御が可能。

**根拠**:
- `unstable_cache` より宣言的で直感的な構文を提供する
- コンポーネント単位でのキャッシュが可能になり、Partial Prerendering との親和性が高い
- Next.js 16 では `unstable_cache` の公式後継として位置づけられている

**コード例**:
```tsx
// Good: use cache ディレクティブでコンポーネントをキャッシュ
// next.config.ts で experimental.dynamicIO: true を有効化した場合
import { unstable_cacheTag as cacheTag, unstable_cacheLife as cacheLife } from 'next/cache';

export async function CachedProductList() {
  'use cache';
  cacheLife('hours');
  cacheTag('products');

  const products = await db.product.findMany();
  return <ProductGrid products={products} />;
}

// Bad: unstable_cache のラッパーをコンポーネントごとに都度定義する（冗長）
const getCachedProductList = unstable_cache(
  async () => db.product.findMany(),
  ['product-list'],
  { tags: ['products'], revalidate: 3600 }
);
```

**出典**:
- [Next.js Docs: use cache directive](https://nextjs.org/docs/app/api-reference/directives/use-cache) (Next.js公式 / 2025)
- [Vercel Blog: Our Journey with Caching](https://nextjs.org/blog/our-journey-with-caching) (Next.js公式ブログ / 2024)

**バージョン**: Next.js 15+
**確信度**: 中
**最終更新**: 2026-05-06

---
