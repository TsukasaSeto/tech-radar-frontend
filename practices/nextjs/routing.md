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

### 7. `generateMetadata` + `sitemap.ts` + `robots.ts` の SEO 三点セットを配置する

App Router の SEO 対策は3つのファイルで構成する。
`generateMetadata` でページごとのメタデータ（title / description / OG / canonical）を動的に生成し、
`app/sitemap.ts` で全 URL を登録し、`app/robots.ts` でクローラーアクセスを制御する。
`'use client'` が必要なページは metadata を export できないため、親の layout.tsx（Server Component）に `generateMetadata` を置く。

**根拠**:
- App Router は静的 `<head>` タグよりも `generateMetadata` が推奨（ページごとに動的に設定できる）
- `sitemap.ts` / `robots.ts` はルートに置くだけで Next.js が自動的に `/sitemap.xml` / `/robots.txt` として配信
- `'use client'` のページでは metadata export が不可のため、layout.tsx への委譲パターンが必要（Zenn SSG記事で実証）

**コード例**:
```tsx
// app/blog/[slug]/page.tsx — 動的メタデータ
export async function generateMetadata({ params }: { params: { slug: string } }) {
  const post = await fetchPost(params.slug);
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      url: `https://example.com/blog/${params.slug}`,
    },
    alternates: { canonical: `https://example.com/blog/${params.slug}` },
  };
}

// app/sitemap.ts — 全URL登録
import type { MetadataRoute } from 'next';
export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const posts = await fetchAllPosts();
  return [
    { url: 'https://example.com', priority: 1.0 },
    ...posts.map(p => ({
      url: `https://example.com/blog/${p.slug}`,
      lastModified: p.updatedAt,
      priority: 0.8,
    })),
  ];
}

// app/robots.ts — クローラー制御
import type { MetadataRoute } from 'next';
export default function robots(): MetadataRoute.Robots {
  return {
    rules: { userAgent: '*', allow: '/', disallow: '/api/' },
    sitemap: 'https://example.com/sitemap.xml',
  };
}

// 'use client' ページは layout.tsx に metadata を委譲
// app/dashboard/layout.tsx（Server Component）
export const metadata = { title: 'Dashboard', description: '...' };
// app/dashboard/page.tsx
'use client'; // ここに metadata export は書けない
```

**出典**:
- [Next.js App RouterでSSG+SEOページを100枚量産する設計パターン](https://zenn.dev/keisuke58/articles/nextjs-ssg-seo-100pages) (Zenn / 2026-05) ※2026-05-06に実際にfetch成功

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-06

---

#### 追加根拠 (2026-05-06) — ルール2「動的ルートと generateStaticParams でSSGを活用する」

新たに以下の記事/ドキュメントで同じプラクティスが推奨された:
- [Next.js App RouterでSSG+SEOページを100枚量産する設計パターン](https://zenn.dev/keisuke58/articles/nextjs-ssg-seo-100pages) (Zenn / 2026-05) ※2026-05-06に実際にfetch成功

実際に300ページ以上のサイトを `generateStaticParams` でビルドした事例報告。「generateStaticParams が返す配列の要素ごとにビルド時にページが生成され、DBアクセスが不要になる」という効果を実証。更新頻度が低くSEO重要なコンテンツはデータをTypeScript定数で管理しSSGと組み合わせるパターン（Gitでバージョン管理可能）が効果的と報告。`dynamicParams = false` でパラメーター外のパスを404にする方法も確認。

**確信度**: 既存（高）→ 高（実プロダクション事例で実証済み）

---

### 8. `layout.tsx` の3つの責務（ナビゲーション宣言・Provider配置・共通ロジック）を明確に分離する

`layout.tsx` が担う責務を「ナビゲーション種別の宣言」「Provider 配置」「共通ロジック（認証ガード・リダイレクト等）」の3つとして意識し、1ファイルに責務が混在しすぎないよう整理する。

**根拠**:
- `layout.tsx` はファイル数を増やすためではなく、「ナビゲーション形式が変わる」または「新しい共通ロジックが必要」な場合にのみ追加する
- 3つの責務を意識することで、認証ガードの位置・Provider のスコープ・ネストされたヘッダー表示などの設計判断が明確になる
- Expo Router でも同様の設計が適用でき、クロスプラットフォームでのレイアウト設計知識が共通化できる

**3つの責務**:
1. **ナビゲーション種別の宣言**: Stack / Tabs / Drawer / Slot のどれを使うかを明示
2. **Provider 配置**: 配下の画面で使う Context や状態を提供
3. **共通ロジック**: 認証ガード、リダイレクト、ログ送信などの横断的関心事

**コード例**:
```
app/
├── layout.tsx              # 責務①: ルートレイアウト + 責務②: 全体Provider
├── (auth)/
│   ├── layout.tsx          # 責務③: 認証チェック + リダイレクト / 責務①: headerShown=false Stack
│   └── login/page.tsx
└── (app)/
    ├── layout.tsx          # 責務③: 認証必須ガード / 責務①: Tabs ナビゲーション
    └── dashboard/page.tsx
```

```tsx
// app/(app)/layout.tsx — 責務が明確
import { redirect } from 'next/navigation';
import { getSession } from '@/lib/auth';

export default async function AppLayout({ children }: { children: React.ReactNode }) {
  // 責務③: 認証ガード（共通ロジック）
  const session = await getSession();
  if (!session) {
    redirect('/login');
  }

  // 責務②: 認証済みユーザー向けのProvider
  return (
    <UserProvider user={session.user}>
      {/* 責務①: 認証済みページのナビゲーション構造 */}
      <div className="flex">
        <Sidebar />
        <main>{children}</main>
      </div>
    </UserProvider>
  );
}

// アンチパターン: layout.tsx を「ファイル整理のため」だけに追加しない
// ナビゲーション変更・Provider追加・共通ロジックの追加がなければ不要
```

**出典**:
- [Expo Router / Next.js App Router のレイアウト設計を責務で整理する](https://zenn.dev/kaji_kaji/articles/file-based-routing-layout-design) (Zenn kaji_kaji / 2026-05-07) ※2026-05-07に実際にfetch成功

**バージョン**: Next.js 14+, App Router
**確信度**: 高
**最終更新**: 2026-05-07

---
