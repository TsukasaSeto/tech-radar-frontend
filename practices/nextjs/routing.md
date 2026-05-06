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
├── (marketing)/          # URL: /about, /blog（マーケティングページ群）
│   ├── layout.tsx        # マーケティング専用レイアウト
│   ├── about/page.tsx    # /about
│   └── blog/page.tsx     # /blog
├── (auth)/               # URL: /login, /register（未認証ページ群）
│   ├── layout.tsx        # 認証フロー専用レイアウト
│   ├── login/page.tsx    # /login
│   └── register/page.tsx # /register
└── (dashboard)/          # URL: /dashboard（認証済みページ群）
    ├── layout.tsx        # サイドバー付きダッシュボードレイアウト
    └── dashboard/page.tsx

// Bad: グループを使わず全レイアウトをルートlayout.tsxに押し込む
// app/layout.tsx で条件分岐によりレイアウトを切り替えようとする（複雑で保守困難）
```

**出典**:
- [Next.js Docs: Route Groups](https://nextjs.org/docs/app/building-your-application/routing/route-groups) (Next.js公式 / 2024)

**バージョン**: Next.js 15+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 6. Intercepting Routes のマッチング規則（`(.)` / `(..)` / `(...)`）を正しく使う

Intercepting Routes はフォルダの相対的な深さに基づいたマッチング記法を持つ。モーダルや先読みプレビューなどの高度なUIパターンを正確に実装するために記法を理解する。

**根拠**:
- 記法を誤るとルートがインターセプトされず、単純なページ遷移になってしまう
- クライアントナビゲーションとURLの直接アクセスで異なる表示を提供できる（URLシェア可能なモーダル等）

**コード例**:
```
# マッチング規法
# (.)  — 同じ階層のセグメントと一致
# (..) — 一つ上の階層と一致
# (...) — app ルートからと一致

app/
├── feed/
│   └── page.tsx                      # /feed
├── @modal/
│   └── (.)photos/[id]/page.tsx       # /feed からナビゲート時にインターセプト
├── photos/
│   └── [id]/page.tsx                 # URL直接アクセス時はフルページ表示
└── layout.tsx                        # @modal スロットを受け取る
```

```tsx
// app/layout.tsx — @modal スロットを受け取る
export default function RootLayout({
  children,
  modal,
}: {
  children: React.ReactNode;
  modal: React.ReactNode;
}) {
  return (
    <html>
      <body>
        {children}
        {modal}  {/* モーダルはここに描画 */}
      </body>
    </html>
  );
}

// app/@modal/default.tsx — モーダルなし時のデフォルト
export default function Default() {
  return null;
}
```

**出典**:
- [Next.js Docs: Intercepting Routes](https://nextjs.org/docs/app/api-reference/file-conventions/intercepting-routes) (Next.js公式 / 2024)
- [Next.js Docs: Parallel Routes](https://nextjs.org/docs/app/api-reference/file-conventions/parallel-routes) (Next.js公式 / 2024)

**バージョン**: Next.js 15+
**確信度**: 高
**最終更新**: 2026-05-06

---
