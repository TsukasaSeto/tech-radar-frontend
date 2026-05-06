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

#### 追加根拠 (2026-05-06)

新たに以下の記事/ドキュメントで同じプラクティスが推奨された:
- [Next.js App RouterでSSG+SEOページを100枚量産する設計パターン](https://zenn.dev/keisuke58/articles/nextjs-ssg-seo-100pages) (Zenn / 2026-05-05) ※2026-05-06に実際にfetch成功

`generateStaticParams` のデータソースは必ずしもDBフェッチである必要はなく、TypeScript 定数配列を直接使う方法が有効なケースがあることが実証された。
更新頻度が低くSEOが重要なデータ（例：再開発プロジェクト一覧等）は TypeScript 定数で管理することで、DBアクセスなしのSSGと相性がよくなり、エッジ配信でのキャッシュ効率も高まる。

```typescript
// TypeScript定数をソースにするパターン
export async function generateStaticParams() {
  return PROJECTS.map((p) => ({ id: p.id }));
}
```

SEO対応の3点セットとして **Metadata API + sitemap.ts + robots.ts** の組み合わせが有効であることも確認された。
また、`"use client"` ディレクティブと `metadata` エクスポートは同一ファイルに共存できないため、メタデータは `layout.tsx` に分離する必要があることも実践上の重要な制約として確認された。

**確信度**: 既存（高）→ 高（コミュニティ実証付き）

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

---

### 5. Route Groups でURLに影響しないディレクトリ整理を行う

`(folderName)` 構文のRoute GroupsはURLパスに含まれないため、コードの論理的なグルーピングとレイアウト分割に活用する。

**根拠**:
- 認証済み/未認証、公開/管理画面など、URLを変えずにレイアウトを分けられる
- チームやドメインごとにルートをグルーピングでき、大規模アプリの可読性が上がる
- 複数のルートレイアウト（root layout）を定義してセクションごとに `<html>` / `<body>` を使い分けられる

**コード例**:
```
app/
├── (marketing)/          # URL: /about, /blog
│   ├── layout.tsx
│   ├── about/page.tsx    # /about
│   └── blog/page.tsx     # /blog
├── (auth)/               # URL: /login, /register
│   ├── layout.tsx
│   ├── login/page.tsx    # /login
│   └── register/page.tsx
└── (dashboard)/
    ├── layout.tsx
    └── dashboard/page.tsx
```

**出典**:
- [Next.js Docs: Route Groups](https://nextjs.org/docs/app/building-your-application/routing/route-groups) (Next.js公式 / 2024)

**バージョン**: Next.js 15+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 6. Intercepting Routes のマッチング規則（`(.)` / `(..)` / `(...)`）を正しく使う

Intercepting Routes はフォルダの相対的な深さに基づいたマッチング記法を持つ。

**根拠**:
- 記法を誤るとルートがインターセプトされず、単純なページ遷移になってしまう
- URL直接アクセスとクライアントナビゲーションで異なる表示を提供できる

**コード例**:
```
# (.)  — 同じ階層のセグメントと一致
# (..) — 一つ上の階層と一致
# (...) — app ルートからと一致

app/
├── @modal/
│   └── (.)photos/[id]/page.tsx
├── photos/[id]/page.tsx
└── layout.tsx
```

**出典**:
- [Next.js Docs: Intercepting Routes](https://nextjs.org/docs/app/api-reference/file-conventions/intercepting-routes) (Next.js公式 / 2024)

**バージョン**: Next.js 15+
**確信度**: 高
**最終更新**: 2026-05-06

---
