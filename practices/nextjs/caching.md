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
