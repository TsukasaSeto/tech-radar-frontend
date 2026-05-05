# Next.js ルーティングのベストプラクティス

## ルール

### 1. App Router のファイルシステムルーティングを活用する

`app/` ディレクトリのファイル構造がそのままルーティングになる。特殊ファイルを適切に配置する。

**根拠**:
- `layout.tsx` でネストしたUIを共有でき、ページ遷移時に再レンダリングされない
- `loading.tsx` と `error.tsx` で自動的に Suspense / ErrorBoundary がラップされる

**コード例**:
```
app/
├── layout.tsx          # ルートレイアウト
├── page.tsx            # /
├── loading.tsx         # Suspense fallback
├── error.tsx           # ErrorBoundary
├── dashboard/
│   ├── layout.tsx      # /dashboard/* 共通
│   └── page.tsx        # /dashboard
└── blog/
    └── [slug]/
        └── page.tsx    # /blog/:slug
```

**出典**:
- [Next.js Docs: Routing Fundamentals](https://nextjs.org/docs/app/building-your-application/routing) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. 動的ルートと generateStaticParams でSSGを活用する

`generateStaticParams` でビルド時に静的生成するパスを指定する。

**コード例**:
```tsx
export async function generateStaticParams() {
  const posts = await fetchAllPosts();
  return posts.map(post => ({ slug: post.slug }));
}

export const dynamicParams = false;

export default async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await fetchPost(params.slug);
  if (!post) notFound();
  return <article>{post.content}</article>;
}
```

**出典**:
- [Next.js Docs: generateStaticParams](https://nextjs.org/docs/app/api-reference/functions/generate-static-params) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. Parallel Routes と Intercepting Routes でモーダルUIを実装する

`@folder` 構文と `(.)folder` 構文を組み合わせてディープリンク可能なモーダルを実現する。

**根拠**:
- URL を直接開くとフルページ表示、クライアントナビゲーション時はモーダルとして機能する

**コード例**:
```
app/
├── layout.tsx
├── @modal/
│   ├── default.tsx
│   └── (.)photos/[id]/page.tsx
├── page.tsx
└── photos/[id]/page.tsx
```

**出典**:
- [Next.js Docs: Parallel Routes](https://nextjs.org/docs/app/building-your-application/routing/parallel-routes) (Next.js公式)
- [Next.js Docs: Intercepting Routes](https://nextjs.org/docs/app/building-your-application/routing/intercepting-routes) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. `useRouter` より `<Link>` を優先し、プログラムナビゲーションは最小限に

通常のリンクは `<Link>` を使い、フォーム送信後など真にプログラム的な制御が必要な場合のみ `useRouter` を使う。

**根拠**:
- `<Link>` はビューポートに入ったリンク先を自動的にプリフェッチする
- `<Link>` はアクセシビリティ（`<a>` タグ）を自動的に保証する

**コード例**:
```tsx
import Link from 'next/link';

function NavMenu() {
  return (
    <nav>
      <Link href="/dashboard">Dashboard</Link>
    </nav>
  );
}
```

**出典**:
- [Next.js Docs: Link Component](https://nextjs.org/docs/app/api-reference/components/link) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05
