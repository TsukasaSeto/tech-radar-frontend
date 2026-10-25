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

**根拠**:
- `generateStaticParams` が返したパラメータごとに、ビルド時にページが生成される。返さなかったパスはリクエスト時に生成される
- `params` は Promise で渡されるので `await` して読む（v16 で同期アクセスは削除された）
- 生成対象外のパスを 404 にしたい場合、Cache Components 無効時は `export const dynamicParams = false` を使える。Cache Components 有効時は `dynamicParams` を export するとビルドが失敗するので、データが見つからないときにページ側で `notFound()` を呼ぶ

**コード例**:
```tsx
// app/blog/[slug]/page.tsx
import { notFound } from 'next/navigation';

export async function generateStaticParams() {
  const posts = await fetchAllPosts();
  return posts.map(post => ({ slug: post.slug }));
}

// Cache Components 無効時のみ: 生成対象外のパスを 404 にする（有効時はビルドエラーになるので書かない）
export const dynamicParams = false;

export default async function BlogPost({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await fetchPost(slug);
  if (!post) notFound(); // Cache Components 有効時は、生成対象外のパスもここで 404 にする
  return <article>{post.content}</article>;
}
```

**出典**:
- [Next.js Docs: generateStaticParams](https://nextjs.org/docs/app/api-reference/functions/generate-static-params) (Next.js公式)
- [Next.js Docs: Migrating to Cache Components — `dynamicParams` is not supported](https://nextjs.org/docs/app/guides/migrating-to-cache-components#dynamicparams-is-not-supported) (Next.js 公式、Cache Components 有効時に `dynamicParams` を export するとビルドが失敗し、代わりに `notFound()` を使う) ※2026-10-25 確認

**バージョン**: Next.js 15+（`params` は Promise。v14 以前は同期オブジェクト、v16 で同期アクセスは削除）
**確信度**: 高
**最終更新**: 2026-10-25

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

Intercepting Routes は、ルートセグメントの相対的な位置に基づいたマッチング記法を持つ。基準はファイルシステムのフォルダの深さではなくルートセグメントで、Parallel Routes の `@slot` フォルダはセグメントに数えない。モーダルや先読みプレビューなどの高度なUIパターンを正確に実装するために記法を理解する。

**根拠**:
- 記法を誤るとルートがインターセプトされず、単純なページ遷移になってしまう
- クライアントナビゲーションとURLの直接アクセスで異なる表示を提供できる（URLシェア可能なモーダル等）
- ハードリフレッシュ（フルSSR）では、クライアントナビゲーション時のスロット状態を引き継げないため、各スロットについて現在のURLに一致するルートを個別に解決し直す。`default.tsx` はこの「一致なし」時に描画するフォールバックであり、`@modal` のようなオプショナルなスロットに限らず**全てのスロットに用意する**
- v16 では、`default.js` がないスロットがあるとビルドが失敗する。v15 以前はビルドは通り、一致するルートを持たないスロットがあるとハードリフレッシュ時だけ 404 になっていた

**コード例**:
```
# マッチング規則（基準はルートセグメント。@slot フォルダは数えない）
# (.)  — 同じ階層のセグメントと一致
# (..) — 一つ上の階層のセグメントと一致
# (...) — app ルートからと一致

app/
├── feed/
│   └── page.tsx                      # /feed
├── @modal/
│   └── (.)photos/[id]/page.tsx       # /feed などからの遷移で /photos/[id] をインターセプト
│                                     # （@modal はセグメントに数えないので、app 直下の photos と同じ階層として (.) を使う）
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

// Bad: 一部のスロットにだけ default.tsx を用意する
// v16 では app/@analytics/ に default.tsx がないとビルドが失敗する
// （v15 以前は、/dashboard 以外の URL へのハードリフレッシュで @analytics スロットが解決できず、ページ全体が 404 になっていた）

// Good: 全スロットに default.tsx を用意する
// app/@analytics/default.tsx
export default function DefaultAnalytics() {
  return null; // または直近の妥当な状態を返す
}
```

**出典引用**:
> "Next.js hits a wall...Lacking a defined path for that specific URL within that specific slot, the framework defaults to the safest option: throwing a 404."
> ([Fix Next.js Parallel Routes 404 on Refresh | App Router Guide](https://dev.to/nainikmehta/fix-nextjs-parallel-routes-404-on-refresh-app-router-guide-11o2), セクション本文) ※2026-09-10に実際にfetch成功

**出典**:
- [Next.js Docs: Intercepting Routes](https://nextjs.org/docs/app/api-reference/file-conventions/intercepting-routes) (Next.js公式、`(..)` はルートセグメント基準で `@slot` フォルダを数えない) ※2026-10-25 確認
  > "The `(..)` convention is based on *route segments*, not the file-system. For example, it does not consider `@slot` folders in Parallel Routes."
- [Next.js Docs: Parallel Routes](https://nextjs.org/docs/app/api-reference/file-conventions/parallel-routes) (Next.js公式 / 2024)
- [Next.js Docs: How to upgrade to version 16 — Parallel Routes `default.js` requirement](https://nextjs.org/docs/app/guides/upgrading/version-16#parallel-routes-defaultjs-requirement) (Next.js 公式、全スロットに `default.js` が必須でないとビルドが失敗する) ※2026-10-25 確認
- [Fix Next.js Parallel Routes 404 on Refresh | App Router Guide](https://dev.to/nainikmehta/fix-nextjs-parallel-routes-404-on-refresh-app-router-guide-11o2) (dev.to、ハードリフレッシュで一部スロットのみ 404 になる具体的な再現条件と `default.js` による修正例) ※2026-09-10 fetch

**バージョン**: Next.js 15+（`default.js` が全スロットで必須になりビルドが失敗するのは 16+）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 7. `generateMetadata` + `sitemap.ts` + `robots.ts` の SEO 三点セットを配置する

App Router の SEO 対策は3つのファイルで構成する。
`generateMetadata` でページごとのメタデータ（title / description / OG / canonical）を動的に生成し、
`app/sitemap.ts` で全 URL を登録し、`app/robots.ts` でクローラーアクセスを制御する。
`metadata` / `generateMetadata` は Server Component でしか export できない。`'use client'` が必要な画面は、page.tsx を Server Component のまま保ち、クライアント部分を別ファイルの Client Component に切り出す。

**根拠**:
- App Router は静的 `<head>` タグよりも `generateMetadata` が推奨（ページごとに動的に設定できる）
- `sitemap.ts` / `robots.ts` はルートに置くだけで Next.js が自動的に `/sitemap.xml` / `/robots.txt` として配信
- `metadata` と `generateMetadata` は Server Component でのみサポートされる（公式）。公式は page.tsx を Server Component のまま保ち、Client Component を別ファイルに分ける方法を示している。親の layout.tsx に metadata を置く方法もあるが、その layout 配下のすべてのページに効く
- `params` は Promise で渡されるので `await` して読む（v16 で同期アクセスは削除された）

**コード例**:
```tsx
// app/blog/[slug]/page.tsx — 動的メタデータ
import type { Metadata } from 'next';

export async function generateMetadata({
  params,
}: {
  params: Promise<{ slug: string }>;
}): Promise<Metadata> {
  const { slug } = await params;
  const post = await fetchPost(slug);
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      url: `https://example.com/blog/${slug}`,
    },
    alternates: { canonical: `https://example.com/blog/${slug}` },
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

// 'use client' が必要な画面: page.tsx は Server Component のまま metadata を export し、
// クライアント部分を別ファイルに切り出す
// app/dashboard/page.tsx（Server Component）
import type { Metadata } from 'next';
import { DashboardClient } from './dashboard-client';

export const metadata: Metadata = { title: 'Dashboard', description: '...' };

export default function Page() {
  return <DashboardClient />;
}

// app/dashboard/dashboard-client.tsx
'use client'; // このファイルには metadata を書けない
export function DashboardClient() {
  // hooks やイベントハンドラーを使う部分
  return null;
}
```

**出典**:
- [Next.js App RouterでSSG+SEOページを100枚量産する設計パターン](https://zenn.dev/keisuke58/articles/nextjs-ssg-seo-100pages) (Zenn / 2026-05) ※2026-05-06に実際にfetch成功
- [Next.js Docs: generateMetadata](https://nextjs.org/docs/app/api-reference/functions/generate-metadata) (Next.js 公式、Server Component 限定である点と、page を Server Component のまま Client Component を分ける方法) ※2026-10-25 確認
  > "The `metadata` object and `generateMetadata` function exports are **only supported in Server Components**."
- [Next.js Docs: sitemap.xml](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/sitemap) (Next.js 公式) ※2026-10-25 確認

**バージョン**: Next.js 15+（`params` は Promise。v14 以前は同期オブジェクト）
**確信度**: 高（Metadata API の仕様は公式ドキュメントで確認）
**最終更新**: 2026-10-25

---

#### 追加根拠 (2026-05-06) — ルール2「動的ルートと generateStaticParams でSSGを活用する」

新たに以下の記事/ドキュメントで同じプラクティスが推奨された:
- [Next.js App RouterでSSG+SEOページを100枚量産する設計パターン](https://zenn.dev/keisuke58/articles/nextjs-ssg-seo-100pages) (Zenn / 2026-05) ※2026-05-06に実際にfetch成功

実際に300ページ以上のサイトを `generateStaticParams` でビルドした事例報告。「generateStaticParams が返す配列の要素ごとにビルド時にページが生成され、DBアクセスが不要になる」という効果を実証。更新頻度が低くSEO重要なコンテンツはデータをTypeScript定数で管理しSSGと組み合わせるパターン（Gitでバージョン管理可能）が効果的と報告。`dynamicParams = false` でパラメーター外のパスを404にする方法も確認。

**確信度**: 既存（高）→ 高（実プロダクション事例で実証済み）

---

### 8. `layout.tsx` は共有 UI と Provider の配置に限定し、認証・認可のガードは置かない

`layout.tsx` の役割は、配下のルートで共有する UI（ヘッダー・サイドバーなど）と、配下で使う Context / Provider の配置に限る。
認証・認可のチェックは `page.tsx` の冒頭と DAL（最後の砦）で行い、Proxy は Cookie の有無を見る楽観的チェックにとどめる。
layout はクライアント遷移のたびに再レンダリングされないため、layout に書いたチェックはルートを移るたびには実行されない。layout は子のルートセグメントや parallel route のレンダリングを止めることもできないので、layout のガードでは子ページも Server Action も守れない。

**根拠**:
- 公式の Authentication ガイドは、layout は遷移時に再レンダリングされず、ルートが変わるたびにはセッションが確認されないため、layout でのチェックに注意するよう求めている。チェックはデータソースの近く（DAL）か、条件付きで描画するコンポーネントで行う
- layout が子を隠したり差し替えたりしても、子のルートセグメントと parallel route のスロットはルーターが描画するので、実行も RSC Payload への出力も止まらない
- Server Action は layout とは別の POST エンドポイントとして呼ばれるので、layout のガードを通らない。Action 内でも認証・認可が必要（`server-actions.md` #5）
- `layout.tsx` はファイル数を増やすためではなく、「共有 UI が変わる」または「Provider のスコープを分けたい」場合にだけ追加する
- 既存コードで layout に `redirect()` のゲートを置いている場合、redirect 先を同じ layout の配下に置くと、クライアント遷移で遷移先の `?_rsc=` リクエストが繰り返し発行され、画面が使えなくなる無限ループになったという報告がある（Qiita の記事と、Next.js 15.1.7・16.3.8・16.4.0-canary.57 で再現したとする GitHub issue #99568。遷移先の URL を直接開くと起きない。2026-10-25 時点で issue は open で、原因は Next.js チームによる確認前）。ページ単体のテストでは検出しづらい。ゲートは page と DAL に移す
- 優先順位 A（`README.md`）と `practices/architecture/data-access-layer.md` #4 の「layout では認可しない」に合わせる

**コード例**:
```
app/
├── layout.tsx              # ルートレイアウト + 全体の Provider
├── (auth)/
│   ├── layout.tsx          # 認証フロー用の共有 UI（ロゴだけのヘッダーなど）
│   └── login/page.tsx
└── (app)/
    ├── layout.tsx          # サイドバー付きの共有 UI（認証・認可は置かない）
    └── dashboard/page.tsx  # page 冒頭で verifySession()、個別データの認可は DAL
```

```tsx
// Good: app/(app)/layout.tsx — 共有 UI と Provider だけを置く
import { Sidebar } from '@/components/Sidebar';
import { ThemeProvider } from '@/components/ThemeProvider';

export default function AppLayout({ children }: { children: React.ReactNode }) {
  return (
    <ThemeProvider>
      <div className="flex">
        <Sidebar />
        <main>{children}</main>
      </div>
    </ThemeProvider>
  );
}
```

```tsx
// Good: app/(app)/dashboard/page.tsx — 認証は page 冒頭、個別データの認可は DAL
import { redirect } from 'next/navigation';
import { verifySession } from '@/app/_lib/dal/auth';
import { getMyDashboard } from '@/app/_lib/dal/dashboard';
import { Dashboard } from '@/components/Dashboard';

export default async function DashboardPage() {
  const session = await verifySession(); // page 冒頭の粗いガード
  if (!session) redirect('/login');

  const dashboard = await getMyDashboard(); // DAL の中でも verifySession() で認可する
  return <Dashboard data={dashboard} />;
}
```

```tsx
// Bad: layout で認証ガードする
// クライアント遷移では再実行されず、子ページのレンダリングや Server Action の呼び出しも止められない
import { redirect } from 'next/navigation';
import { getSession } from '@/lib/auth';

export default async function AppLayout({ children }: { children: React.ReactNode }) {
  const session = await getSession();
  if (!session) redirect('/login');
  return <main>{children}</main>;
}
```

**アンチパターン**:
- layout.tsx を「ファイル整理のため」だけに追加する（共有 UI と Provider に変化がなければ不要）
- layout で `return null` や redirect をして配下を守れたと考える（子のセグメント・スロット・Server Action は別に実行される）

**出典**:
- [Next.js Docs: Authentication — Layouts and auth checks](https://nextjs.org/docs/app/guides/authentication#layouts-and-auth-checks) (Next.js 公式、layout は遷移時に再レンダリングされず、子セグメントの描画も止めない) ※2026-10-25 確認
  > "Due to Partial Rendering, be cautious when doing checks in Layouts as these don't re-render on navigation, meaning the user session won't be checked on every route change."
- [Expo Router / Next.js App Router のレイアウト設計を責務で整理する](https://zenn.dev/kaji_kaji/articles/file-based-routing-layout-design) (Zenn kaji_kaji / 2026-05-07、layout を追加する条件の整理) ※2026-05-07に実際にfetch成功
- [Next.js App Router：layout で redirect する先をその layout の配下に置くと、クライアント遷移で無限ループする](https://qiita.com/20051215kaito/items/3ea9e7765fe60d98f244) (Qiita 20051215kaito、layout ゲートの redirect 先を route group で layout 外へ出す。単一記事) ※2026-10-15 fetch
  > "App Router は、共通の layout をクライアント遷移のたびに作り直しません。"
  > (セクション "原因")
- [vercel/next.js#99568: Layout redirect loops on Link navigation despite returned middleware response](https://github.com/vercel/next.js/issues/99568) (GitHub issue、open。layout の redirect 先が同じ layout の配下にあるとき、Link での遷移で `?_rsc=` リクエストが繰り返される。前身の #76308 はアプリ側の不備として close されたが、その修正後も再現するという報告) ※2026-10-25 確認
  > "Link navigation changes the URL to `/test/1/1`, but the page remains blank and repeatedly requests its RSC payload. Direct destination navigation succeeds."

**バージョン**: Next.js 14+, App Router
**確信度**: 高（layout でガードしない理由は公式ドキュメントで確認。`_rsc` の無限ループは記事 1 本と未解決の GitHub issue の報告で、Next.js 側の原因は未確認）
**最終更新**: 2026-10-25

---

### 9. `loading.tsx` があると `notFound()` の HTTP ステータスが 200 になるケースを把握する

`loading.tsx` がルートに存在すると Next.js はページを Suspense でラップし、レスポンスヘッダー（HTTP 200）をストリーミング開始時点で送信する。
その後のレンダリングフェーズで `notFound()` を呼んでも、**ヘッダーは既に送信済みのため HTTP 404 に変えられない**。
SEO クローラー・BFF・監視ツールが HTTP ステータスコードに依存している場合に影響が出る。

**根拠**:
- 公式の `loading.js` リファレンスによると、ストリーミングするレスポンスは 200 で始まり、ストリームを開始した時点でレスポンスヘッダーが送られてステータスコードが確定する。`loading.tsx` のフォールバックが描画された時点、または `<Suspense>` 配下の Server Component が suspend した時点でストリームが始まる
- それより前に `notFound()` / `forbidden()` / `unauthorized()` を呼べば、404 / 403 / 401 のステータスが保たれる
- Next.js は `loading.tsx` の存在をもって Suspense 境界を自動挿入し、ローディング UI のチャンクを先行送信する
- 親ディレクトリの `loading.tsx` も子ルートを Suspense でラップするため、直接配置した覚えがなくても発生する
- ストリーム中に 404 ページを返した場合、Next.js は `<meta name="robots" content="noindex">` を付けるので検索結果には載らない。ただしステータスは 200 のままなので、ステータスコードで判定する監視・分析・BFF には 404 として伝わらない

**コード例**:
```tsx
// Bad: loading.tsx が存在するセグメントで notFound() を呼ぶと 200 が返る
// app/blog/[slug]/loading.tsx が存在する状態で:
import { notFound } from 'next/navigation';

export default async function BlogPost({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await fetchPost(slug);
  if (!post) notFound(); // ← HTTP は 200 のまま（ヘッダー送信済み）
  return <article>{post.content}</article>;
}
```

```tsx
// Good: loading.tsx を置かず、存在確認（notFound）を Suspense 境界の外で先に済ませる
// app/blog/[slug]/page.tsx（このセグメントと親に loading.tsx がない状態）
import { Suspense } from 'react';
import { notFound } from 'next/navigation';

export default async function BlogPost({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await fetchPost(slug);
  if (!post) notFound(); // ストリーミング開始前なので 404 になる

  return (
    <article>
      <h1>{post.title}</h1>
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments postId={post.id} /> {/* 遅い部分だけを Suspense でストリーミングする */}
      </Suspense>
    </article>
  );
}
```

**アンチパターン**:
- `loading.tsx` を使いつつ同セグメントで `notFound()` によるエラーハンドリングをしている（HTTP ステータスが 200 になる）
- 親ディレクトリに `loading.tsx` があることに気づかず `notFound()` が効いていると思い込む
- 404 判定のために Proxy で毎リクエスト DB から本文を取得する（公式は Proxy で slug を確認して not-found へ rewrite する方法も挙げているが、Proxy は全リクエストで走るため、本文は取得せず軽量な確認に限る。`middleware.md` #1）

**出典引用**:
> "ストリーミング時は200が返る。`redirect` や `notFound` によるエラーはストリームのコンテンツ内で伝達される。レスポンスヘッダーはすでに送信されているため、ステータスコードを更新することはできない。"
> ([Next.js: loading.tsx があると notFound() が 200 を返すことがある（仕様です）](https://zenn.dev/av_shiritai/articles/66f9bcaa7df414), セクション "原因") ※2026-06-12に実際にfetch成功

> "Starting the stream sends the response headers and fixes the status code."
> ([Next.js Docs: loading.js](https://nextjs.org/docs/app/api-reference/file-conventions/loading#status-codes), セクション "Status codes") ※2026-10-25 確認

**出典**:
- [Next.js: loading.tsx があると notFound() が 200 を返すことがある（仕様です）](https://zenn.dev/av_shiritai/articles/66f9bcaa7df414) (Zenn、streaming + notFound() の挙動解説と3つの対策) ※2026-06-12に実際にfetch成功
- [Next.js Docs: loading.js — Status codes](https://nextjs.org/docs/app/api-reference/file-conventions/loading#status-codes) (Next.js 公式、ストリーム開始でステータスが確定する点と、開始前に notFound() を呼べば 404 が保たれる点) ※2026-10-25 確認

**バージョン**: Next.js 15+ (App Router)（コード例の `params` は Promise。v14 以前は同期オブジェクト）
**確信度**: 高（公式ドキュメントで確認）
**最終更新**: 2026-10-25

---

### 10. `cacheComponents` + `partialPrefetching` でインスタントナビゲーション（SPA並み応答）を実現する

Next.js 16.3 Preview で導入された「Instant Navigations」を使うと、サーバー駆動モデルを維持しながら SPA と同等のクリック即応性を得られる。
ルートが `<Suspense>` でストリームするか `"use cache"` でキャッシュしていれば、クリック後にサーバーレスポンスを待たずにシェルが即座に表示される。
`partialPrefetching: true` を有効化すると、ビューポート内の全リンクに対して個別プリフェッチする代わりに、**ルートごとに再利用可能なシェルを 1 回だけプリフェッチ**するため、プリフェッチリクエスト数が大幅に削減される。
Next.js 16.4 以降は `create-next-app` が両フラグを既定で有効化する。大きなリスト・グリッドでの prefetch の昇格は Rule #13、低優先度コンテンツを遷移後に回す方法は Rule #12 を参照。

**根拠**:
- サーバー駆動アプリのナビゲーションが「遅く感じる」根本原因は、クリック後のネットワーク往復とサーバー処理の 2 つ。`<Suspense>`（ストリーム）か `"use cache"`（キャッシュ）でサーバー処理ブロックを排除し、シェルプリフェッチでネットワーク往復を隠蔽することで両方を解消する
- 旧来の per-link プリフェッチは同一ルートに 20 個のリンクがあれば 20 リクエストを発行していた。`partialPrefetching` は同じルートを 1 リクエスト（シェル）で済ませ、キャッシュされたシェルをすべてのリンクで再利用する
- `export const instant = false` は、そのセグメント（page / layout / parallel route のスロット）をインスタントナビゲーションの検証から外し、サーバーでのブロッキングを許可する設定。ルートを動的にするわけではなく（プリレンダーできるルートは静的シェルを出す）、既存の `<Suspense>` やローディング UI を消すわけでもない。Cache Components へ段階的に移行する間、まだ対応していないルートに付けておく使い方が公式の想定（codemod `cache-components-instant-false` で一括付与できる）。`new Date()` などの同期 IO によるビルドエラーは `instant = false` でも解消しない
- `instant()` Playwright テストヘルパーでナビゲーション後に「何が即座に表示されるか」を Assertion できる。CI でインスタントナビゲーションのリグレッションを防止できる

**コード例**:
```ts
// next.config.ts — 両フラグを有効化（Next.js 16.4 以降の create-next-app では既定で有効）
import type { NextConfig } from 'next';
const nextConfig: NextConfig = {
  cacheComponents: true,
  partialPrefetching: true,
};
export default nextConfig;
```

```tsx
// Good: params の Promise を Suspense の内側に渡して await する（シェル表示 → データ流入）
// app/products/[slug]/page.tsx
import { Suspense } from 'react';

export default function ProductPage({ params }: { params: Promise<{ slug: string }> }) {
  return (
    <Suspense fallback={<ProductSkeleton />}>
      <ProductDetails params={params} />
    </Suspense>
  );
}

async function ProductDetails({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const product = await getProduct(slug);
  return <h1>{product.name}</h1>;
}
```

```tsx
// 移行中: まだ対応していないルートを検証から外し、サーバーでのブロッキングを許可する
// app/blog/[slug]/page.tsx
export const instant = false; // ルートを動的にはしない。Suspense やローディング UI もそのまま
```

```ts
// Playwright: instant() ヘルパーでナビゲーション直後の即時表示を Assertion
import { expect, test } from '@playwright/test';
import { instant } from '@next/playwright';

test('product title is immediately visible', async ({ page }) => {
  await page.goto('/products/shoes');
  await instant(page, async () => {
    await page.click('a[href="/products/hats"]');
    await expect(page.locator('h1')).toContainText('Baseball Cap');
    await expect(page.getByText('Checking inventory...')).toBeVisible(); // ローディング中
  });
  await expect(page.getByText('12 in stock')).toBeVisible(); // データ流入後
});
```

**アンチパターン**:
- `cacheComponents: true` だけ有効化して `partialPrefetching` を忘れる（プリフェッチの最適化が効かない）
- `export const instant = false` を「ローディング UI を消す設定」と誤解して付ける（検証から外れるだけで、Suspense のフォールバックは表示される）
- `instant = false` を付けたまま放置し、ルートごとの移行（`use cache` / `<Suspense>` での対応）を終えない
- `instant()` テストを書かずにインスタント性をリリース後に壊す

**出典引用**:
> "You get the full benefits of a server, but navigations are instant like in a single-page app."
> ([Next.js 16.3: Instant Navigations](https://nextjs.org/blog/next-16-3-instant-navigations), セクション "We're fixing this") ※2026-06-25に実際にfetch成功

> "Instead of prefetching a page *per link*, Next.js will now prefetch a reusable shell *per route*."
> ([Next.js 16.3: Instant Navigations](https://nextjs.org/blog/next-16-3-instant-navigations), セクション "Rethinking prefetching") ※2026-06-25に実際にfetch成功

**出典**:
- [Next.js 16.3: Instant Navigations](https://nextjs.org/blog/next-16-3-instant-navigations) (Next.js 公式ブログ)
- [Improving navigation experience in v0 with Next.js 16.4](https://nextjs.org/blog/improving-navigation-experience-in-v0-with-nextjs-16-4) (Next.js 公式ブログ、16.4 で両フラグが `create-next-app` の既定になった点) ※2026-10-17 fetch
- [Next.js 16.4](https://nextjs.org/blog/next-16-4) (Next.js 公式ブログ、新規アプリは Cache Components が既定で有効。Cache Components を使うときは `partialPrefetching` も併せて有効にする) ※2026-10-25 確認
  > "As of today, all new apps created with `create-next-app` will have Cache Components enabled by default."
- [vercel/next.js `packages/create-next-app/templates/index.ts`（v16.4.0）](https://github.com/vercel/next.js/blob/v16.4.0/packages/create-next-app/templates/index.ts) (Next.js 公式リポジトリ、Cache Components の質問の既定値は Yes で、有効時に `cacheComponents: true` と `partialPrefetching: true` の両方を next.config に書き込む。`--no-cache-components` で無効にできる) ※2026-10-25 確認
- [Next.js Docs: instant](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config/instant) (Next.js 公式、`instant` は v16.3.0 で安定化した route segment config で、`true` / `false` / `{ level }` を受け付ける。Cache Components 有効時のみ) ※2026-10-25 確認
- [Next.js Docs: Migrating to Cache Components — Opting out of validation](https://nextjs.org/docs/app/guides/migrating-to-cache-components#opting-out-of-validation) (Next.js 公式、`instant = false` の意味と段階移行の手順) ※2026-10-25 確認
  > "`instant = false` marks a segment as *allowed to block*. It does not force the route to be dynamic, so a genuinely prerenderable route still ships a static shell."

**バージョン**: Next.js 16.3 Preview+（`cacheComponents: true` + `partialPrefetching: true` 必須。16.4+ は `create-next-app` の既定）
**確信度**: 高（Next.js 公式ブログ・公式ドキュメント）
**最終更新**: 2026-10-25

---

### 11. `scroll-behavior: smooth` を使うなら `<html>` に `data-scroll-behavior="smooth"` を明示する（Next.js 16 のルート遷移スクロール変更）

Next.js 16 は、`<html>` に `scroll-behavior: smooth` が当たっているのに `data-scroll-behavior` 属性が無い場合に開発時警告を出す。
Next.js 15 以前はルート遷移時にフレームワーク側がスムーススクロールを暗黙に打ち消していたが、16 ではその上書きをデフォルトで行わなくなった。
`data-scroll-behavior="smooth"` は「スムーススクロールを有効にする属性」ではなく、「ルート遷移中だけ Next.js に上書きを許可する opt-in マーカー」である点を取り違えないこと。
`<Link scroll={false}>` は個別ナビゲーションの制御であり、`data-scroll-behavior` のグローバルな役割とは別物なので置き換えにならない。

**根拠**:
- 警告文自体が対処法を指定している: `Detected \`scroll-behavior: smooth\` on the \`<html>\` element. To disable smooth scrolling during route transitions, add \`data-scroll-behavior="smooth"\` to your <html> element.`
- 属性を付けないまま放置すると、ルート遷移時にページ先頭までスムーススクロールが走り、遷移が体感的に遅くなる（Next.js 15 以前の暗黙の打ち消しが無くなったため）
- `prefers-reduced-motion: reduce` での `scroll-behavior: auto` へのフォールバックは CSS 側の責務であり、`data-scroll-behavior` では代替できない（アクセシビリティ要件は別途担保する）
- 検証環境は Next.js 16.3.1 / React 19.2.8 / App Router

**コード例**:
```tsx
// Good: app/layout.tsx — グローバルにスムーススクロールを使うなら opt-in を明示
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja" data-scroll-behavior="smooth">
      <body>{children}</body>
    </html>
  );
}

// Bad: CSS だけ smooth にして属性を付けない（Next.js 16 で警告 + 遷移がスムーススクロールに巻き込まれる）
// <html lang="ja">
```
```css
/* globals.css — アクセシビリティのフォールバックは CSS 側で担保する */
html {
  scroll-behavior: smooth;
}

@media (prefers-reduced-motion: reduce) {
  html {
    scroll-behavior: auto;
  }
}
```

**アンチパターン**:
- `data-scroll-behavior="smooth"` を「スムーススクロールを有効化する属性」と誤解し、CSS の `scroll-behavior` を削除してしまう
- 警告を消す目的で `<Link scroll={false}>` を全リンクに付けて回る（個別ナビゲーション制御であり、警告の原因は解消しない）

**出典引用**:
> "重要なのは、`data-scroll-behavior`がスムーズスクロールを有効にする属性ではないことです。"
> ([Next.js 16のscroll-behavior警告をdata-scroll-behaviorで整理する](https://zenn.dev/nadarakainc/articles/0b3681191ac051), セクション "警告が示していること") ※2026-08-28に実際にfetch成功

> "Next.js 15以前は...SPAのルート遷移で...処理を自動的に行っていました。Next.js 16では、この上書きをデフォルトで行わなくなりました。"
> ([Next.js 16のscroll-behavior警告をdata-scroll-behaviorで整理する](https://zenn.dev/nadarakainc/articles/0b3681191ac051), セクション "Next.js 16で何が変わったか") ※2026-08-28に実際にfetch成功

**バージョン**: Next.js 16+（16.3.1 で確認）
**確信度**: 中（Next.js 公式の警告文・属性仕様を実機で検証した単独記事、パターン1c採用）
**最終更新**: 2026-08-28

---

### 12. `<Link prefetch>` が重いルートは `await navigation()` / `await prefetch()` で後段に処理を遅延する

`<Link prefetch>` は読み込み状態を消せるが、可視リンクすべてがキャッシュ済みデータ全体を先読みするためサーバー負荷が過大になりうる。プリフェッチ自体を止めるのではなく、Next.js 16.4 の `navigation()`（実際のナビゲーションまで遅延）と `prefetch()`（明示プリフェッチまで遅延）を `await` し、重い部分だけをプリフェッチ対象から外す。

**根拠**:
- メール一覧のようにリンクが多数並ぶ画面で、全メッセージのスレッドを先読みするとコストが過大になる。最初のメッセージだけ先読みし、残りは遷移後に描画できる
- `navigation()` はプリフェッチ時に対象を除外しナビゲーション時に描画、`prefetch()` はルートのシェルに入るはずのキャッシュ内容を除外し明示プリフェッチまで遅延する
- 遅延する部分は `<Suspense>` で囲み、フォールバックを用意する
- 先読みの粒度を「全部」か「なし」の二択にせず、段階ごとに調整できる

**コード例**:
```tsx
import { navigation } from 'next/cache';

async function Message({ id }: { id: string }) {
  const message = await getMessage(id);
  return (
    <>
      <p>{message.subject}</p>
      <Suspense fallback={<Spinner />}>
        <Thread id={id} />
      </Suspense>
    </>
  );
}

// Good: スレッドはプリフェッチから除外し、実ナビゲーションで取得する
async function Thread({ id }: { id: string }) {
  await navigation();
  const thread = await getThread(id);
  return thread.map((m) => <div key={m.id}>{m.body}</div>);
}
```

**アンチパターン**:
- 重いデータを含むページへのリンクすべてに `prefetch` を付け、使われない先読みでサーバー負荷を増やす
- 負荷を理由に `prefetch` を外して読み込み状態を復活させる（段階的な遅延で両立できる）

**出典引用**:
> "The new `navigation` API lets you exclude loading cached content during a prefetch, effectively deferring it until an actual navigation takes place."
> ([Next.js 16.4](https://nextjs.org/blog/next-16-4), セクション "Excluding content from a prefetch") ※2026-10-06に実際にfetch成功

**出典**:
- [Next.js 16.4](https://nextjs.org/blog/next-16-4) (Next.js 公式ブログ、2026-10-06公開) ※2026-10-06に実際にfetch成功

**バージョン**: Next.js 16.4+（`cacheComponents: true` + `partialPrefetching: true`）
**確信度**: 高（Next.js 公式ブログ）
**最終更新**: 2026-10-06

---

### 13. 大きなリスト・グリッドは「ユーザー意図（hover / focus / touch）」を検知してから `prefetch={true}` に昇格させる

Partial Prefetching 環境では `prefetch={true}` が動的ルート（`/products/[id]` 等）の「キャッシュ済みコンテンツ」までプリフェッチ対象に含める。
ビューポートに入ったカード全件に `prefetch={true}` を付けると、開かれないページまで取得してリクエスト数が跳ね上がるため、大きなグリッドでは hover / focus / touch を検知してから昇格させる（デフォルトは `'auto'`）。
ルート内の「クリック時には不要だが戻り時には再利用したい」低優先度コンテンツを遷移後に回す方法は Rule #12（`await navigation()`）を参照。

**根拠**:
- v0 では全カードに richer prefetch を入れた結果、`/templates/[id]` へのプリフェッチリクエストが **2.9 倍** に増えた。意図ベースに切り替えて抑制した
- 件数が少なく固定のリンク群は、ビューポート進入時プリフェッチのままで問題ない（グリッド・長いリストだけが対象）
- v0 は、共有可能な公開データを `'use cache: remote'` + `cacheTag` + `cacheLife`、ユーザー固有データを `'use cache: private'`（ブラウザ内保持）に分けた。失効は、公開データは `revalidateTag()`、ユーザーが更新するデータの即時反映は `updateTag()` と使い分ける。ユーザー固有データの扱いは、まず検証済みの userId を引数に渡す `'use cache'` を検討し、`'use cache: private'` はそれが成り立たない場合に選ぶ（選び方は `caching.md` #8）
- 効果は Runtime Cache 費用・リクエスト数・Active CPU・プリフェッチのデータ転送量の 4 指標で PR ごとに検証する。v0 の事例では Runtime Cache 費用が 4.3 倍（約 $3.44/日増）、リクエスト処理コストが 14.4% 減だった。全体の日次コストは 2.9% 減だが、公式ブログは同期間にトラフィック等も変動したため「悪化しなかったことの確認」として扱うと明記しており、削減効果の根拠にはしない

**コード例**:
```tsx
'use client';
import Link from 'next/link';
import { useState } from 'react';

// Good: 意図を検知してから richer prefetch に昇格する（大きなグリッド向け）
export function TemplateCard({ href, children }: { href: string; children: React.ReactNode }) {
  const [intent, setIntent] = useState(false);
  return (
    <Link
      href={href}
      prefetch={intent ? true : 'auto'}
      onMouseEnter={() => setIntent(true)}
      onFocus={() => setIntent(true)}
      onTouchStart={() => setIntent(true)}
    >
      {children} {/* カードのタイトルなど、リンクのアクセシブルな名前になる内容 */}
    </Link>
  );
}

// Bad: 全カードを一律 prefetch={true} にする（開かれないページまで取得する）
// <Link href={`/templates/${t.id}`} prefetch={true} />
```

**アンチパターン**:
- 大きなグリッドの全リンクに `prefetch={true}` を付け、プリフェッチ量とキャッシュ費用の増加を計測しない
- ユーザー固有コンテンツを、ユーザーを区別しないキーで `'use cache'` / `'use cache: remote'` に載せる（検証済みの userId を引数に渡すか、`'use cache: private'` を使う。`caching.md` #8）

**出典引用**:
> "The agent changed the richer prefetch to wait for user intent, starting on hover, focus, or touch rather than when the card entered the viewport."
> ([Improving navigation experience in v0 with Next.js 16.4](https://nextjs.org/blog/improving-navigation-experience-in-v0-with-nextjs-16-4), セクション "Avoiding unnecessary prefetches in large grids") ※2026-10-16に実際にfetch成功

**バージョン**: Next.js 16.4+（`cacheComponents: true` + `partialPrefetching: true`）
**確信度**: 高（Next.js 公式ブログ）
**最終更新**: 2026-10-25

---
