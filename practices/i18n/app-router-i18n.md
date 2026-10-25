# Next.js App Router での i18n のベストプラクティス

ロケールをルーティングでどう表現するか、Middleware でどう検出するか、ライブラリ選定をどうするか。

## ルール

### 1. ロケールはパスベース（`/en/`, `/ja/`）で表現する

ロケールは URL の最初のセグメントに含める。Cookie や Accept-Language ヘッダーだけに依存しない。

**根拠**:
- パスベースなら検索エンジンが各言語ページを個別の URL としてインデックスできる
- URL を共有した時、開いた相手にも同じ言語で表示される（Cookie 依存だと違う言語になる）
- `hreflang` で言語ごとの URL を明示でき、SEO 効果が高い
- Cookie ベースは UX シンプルだがアクセシビリティ・SEO で劣る

**ルーティング戦略の選択**:

| 戦略 | 例 | 適用 |
|---|---|---|
| **サブパス** | `/ja/about` `/en/about` | 最も一般的、Next.js App Router の `[locale]` で実装 |
| **サブドメイン** | `ja.example.com` `en.example.com` | 言語ごとに独立したインフラ、大規模サイト |
| **ドメイン分離** | `example.jp` `example.com` | 法的・運用上の理由（GDPR / 法人分離） |
| **Cookie のみ** | `/about`（Cookie で言語切替） | 推奨しない（SEO 不可・共有困難） |

**App Router での `[locale]` 配置**:
```
app/
├── [locale]/
│   ├── layout.tsx
│   ├── page.tsx                # /ja/, /en/
│   ├── about/
│   │   └── page.tsx            # /ja/about, /en/about
│   └── products/
│       └── [id]/
│           └── page.tsx        # /ja/products/123, /en/products/123
└── api/                        # API は locale 共通
```

**`generateStaticParams` で全ロケールを静的生成**:
```tsx
// app/[locale]/layout.tsx
import { setRequestLocale } from 'next-intl/server';
import { notFound } from 'next/navigation';

const locales = ['ja', 'en'] as const;
type Locale = typeof locales[number];

export function generateStaticParams() {
  return locales.map((locale) => ({ locale }));
}

export default async function LocaleLayout({
  children,
  params,
}: {
  children: React.ReactNode;
  params: Promise<{ locale: Locale }>;
}) {
  const { locale } = await params;
  if (!locales.includes(locale)) notFound();

  setRequestLocale(locale);
  return (
    <html lang={locale}>
      <body>{children}</body>
    </html>
  );
}
```

**`<head>` への `hreflang` 注入**:

canonical と `languages` はページごとに、パスを含めて作る。locale の layout に固定の URL で置くと、`/ja/about` や記事ページなど子ページすべての canonical がロケールのトップになってしまう。

```ts
// i18n/alternates.ts
import type { Metadata } from 'next';

const BASE_URL = 'https://example.com';
const locales = ['ja', 'en'] as const;

export function localeAlternates(locale: string, path: string): Metadata['alternates'] {
  return {
    canonical: `${BASE_URL}/${locale}${path}`,
    languages: {
      ...Object.fromEntries(locales.map((l) => [l, `${BASE_URL}/${l}${path}`])),
      'x-default': `${BASE_URL}/en${path}`,
    },
  };
}
```

```tsx
// app/[locale]/about/page.tsx
import type { Metadata } from 'next';
import { localeAlternates } from '@/i18n/alternates';

export async function generateMetadata({
  params,
}: {
  params: Promise<{ locale: string }>;
}): Promise<Metadata> {
  const { locale } = await params;
  return { alternates: localeAlternates(locale, '/about') };
}
```

**出典**:
- [Next.js Docs: Internationalization](https://nextjs.org/docs/app/building-your-application/routing/internationalization) (Next.js 公式)
- [Google: Locale-aware crawling](https://developers.google.com/search/docs/specialty/international/managing-multi-regional-sites) (Google Search Central)
- [web.dev: Tell Google about localized versions of your page](https://web.dev/articles/hreflang) (web.dev)

**バージョン**: Next.js 15+（`params` が Promise。v14 以前は同期オブジェクトで、`await` せずに読む）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 2. Proxy（v15 以前は Middleware）で Accept-Language を検出し、未指定なら正規 URL にリダイレクトする

Next.js 16 以降は `proxy.ts`（`export function proxy`）、v15 以前は `middleware.ts`（`export function middleware`）で、`/` 等のロケールなし URL を `Accept-Language` 解析でリダイレクトする。
`/en/...` のような有効ロケールはそのまま通過させる。next-intl を使う場合は自前で書かず `createMiddleware`（#3）に任せる。

**根拠**:
- Accept-Language ヘッダーはブラウザの言語設定をサーバーが知る標準的手段
- 初回訪問時に「正しい言語にリダイレクト」+「明示パスでブックマーク可能」の両立
- BCP 47 形式のパースは `@formatjs/intl-localematcher` 等のライブラリに任せる
- Proxy でリダイレクトすれば Server Component 側の処理は単純化できる
- Proxy は Next.js 16 で Middleware から改名された（ファイル名と関数名が変わる。codemod: `npx @next/codemod@canary middleware-to-proxy .`）。Proxy は Node.js ランタイム固定で、`runtime` を書くとエラーになる

**Proxy の実装**:
```ts
// proxy.ts（Next.js 16+。v15 以前はファイル名を middleware.ts、関数名を middleware にする）
import { NextRequest, NextResponse } from 'next/server';
import Negotiator from 'negotiator';
import { match as matchLocale } from '@formatjs/intl-localematcher';

const locales = ['ja', 'en'] as const;
const defaultLocale = 'ja';

function detectLocale(request: NextRequest): string {
  const negotiator = new Negotiator({
    headers: { 'accept-language': request.headers.get('accept-language') ?? '' },
  });
  const requested = negotiator.languages();
  try {
    return matchLocale(requested, locales as readonly string[], defaultLocale);
  } catch {
    return defaultLocale;
  }
}

export function proxy(request: NextRequest) {
  const pathname = request.nextUrl.pathname;

  // すでにロケール付きならスキップ
  const hasLocale = locales.some(
    (locale) => pathname.startsWith(`/${locale}/`) || pathname === `/${locale}`,
  );
  if (hasLocale) return NextResponse.next();

  // Cookie で過去の選択を尊重
  const cookieLocale = request.cookies.get('NEXT_LOCALE')?.value;
  const locale = cookieLocale && (locales as readonly string[]).includes(cookieLocale)
    ? cookieLocale
    : detectLocale(request);

  const url = request.nextUrl.clone();
  url.pathname = `/${locale}${pathname === '/' ? '' : pathname}`;
  return NextResponse.redirect(url);
}

export const config = {
  matcher: [
    // api / _next / _vercel と、拡張子付きのパス（public の画像・favicon.ico・sitemap.xml 等）を除外する。
    // 静的ファイルまで /ja/logo.svg にリダイレクトされて 404 になるのを防ぐ。
    // /users/jane.doe のようにドットを含むページがある場合は、そのパスを別途 matcher に足す
    '/((?!api|_next|_vercel|.*\\..*).*)',
  ],
};
```

**Cookie でユーザー選択を記憶**:
```tsx
'use client';
import { useRouter, usePathname } from 'next/navigation';

function LanguageSwitcher({ current }: { current: string }) {
  const router = useRouter();
  const pathname = usePathname();

  function switchLocale(next: string) {
    document.cookie = `NEXT_LOCALE=${next}; path=/; max-age=${60 * 60 * 24 * 365}; SameSite=Lax`;
    const newPath = pathname.replace(/^\/(ja|en)/, `/${next}`);
    router.push(newPath);
  }

  return (
    <select value={current} onChange={(e) => switchLocale(e.target.value)}>
      <option value="ja">日本語</option>
      <option value="en">English</option>
    </select>
  );
}
```

**ボット・SEO クローラーの扱い**:
- Googlebot / Bingbot は Accept-Language を送らない（または en-US 固定）
- bot 検出は不確実なので、リダイレクトロジックを botに優しく:
  - bot がアクセス → デフォルトロケールへ 302 リダイレクト（301 は避ける）
  - 302 にすると検索エンジンは元 URL を保持する

**よくある落とし穴**:
- `/api/` を matcher から除外し忘れる → API リクエストもリダイレクトされて壊れる
- 静的ファイル（画像・OGP）を matcher に含めてリダイレクト → 不要な処理
- `301`（permanent）でリダイレクト → ユーザーが言語切替しても古いリダイレクトがキャッシュされる

**例外: `output: 'export'`（静的エクスポート）では Proxy（Middleware）が動作しない**:
静的エクスポートにはサーバーが存在しないため、Proxy による `Accept-Language` 検出・リダイレクトは使えない。ロケール判定を完全に URL パスへ寄せ、`generateStaticParams()` で全ロケール × 全ルートを事前レンダリングする設計に切り替える。

```ts
// routing.ts（サーバーなし・パスのみでロケールを決定）
export function generateStaticParams() {
  return locales.flatMap((locale) =>
    routes.map((route) => ({ locale, ...route.params })),
  );
}
```

> "On a server you'd typically read `Accept-Language` and pick a locale per request. With static export there's no request to inspect."
> ([Eight locales and no server: internationalizing a static Next.js site](https://dev.to/coolnico/eight-locales-and-no-server-internationalizing-a-static-nextjs-site-54jh), セクション "Locale from the URL, not the request") ※2026-08-15に実際にfetch成功

**出典**:
- [Next.js Docs: Internationalization - Routing](https://nextjs.org/docs/app/building-your-application/routing/internationalization#routing-overview) (Next.js 公式)
- [Next.js Docs: proxy.js](https://nextjs.org/docs/app/api-reference/file-conventions/proxy) (Next.js 公式、v16 での Middleware → Proxy の改名、`export function proxy`、Node.js ランタイム固定、matcher の書き方) ※2026-10-25 確認
- [next-intl: Proxy / middleware](https://next-intl.dev/docs/routing/middleware) (next-intl 公式、推奨 matcher `'/((?!api|trpc|_next|_vercel|.*\\..*).*)'` とドットを含むパスの扱い) ※2026-10-25 確認
- [@formatjs/intl-localematcher](https://formatjs.io/docs/polyfills/intl-localematcher/) (FormatJS)
- [BCP 47](https://www.rfc-editor.org/info/bcp47) (IETF)
- [Eight locales and no server: internationalizing a static Next.js site](https://dev.to/coolnico/eight-locales-and-no-server-internationalizing-a-static-nextjs-site-54jh) (dev.to、`output: 'export'` 配下で Middleware が使えない例外の実装パターン) ※2026-08-15に実際にfetch成功

**バージョン**: Next.js 16+（`proxy.ts`）/ v15 以前は `middleware.ts`（`output: 'export'` の例外は Next.js 15 で検証）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 3. `next-intl` を i18n ライブラリの第一選択にする

App Router での i18n には `next-intl` を採用する。
ICU MessageFormat サポート・型安全・Server Component 統合の 3 点で優位。

**根拠**:
- `next-intl` は App Router を最初から想定して設計されている（Server Component / Client Component 両対応）
- ICU MessageFormat（複数形・性別・選択）を標準サポート
- `declare module 'next-intl'` の `AppConfig` を拡張すると、メッセージキーを型レベルで強制できる（設定は `practices/i18n/message-catalog.md` #4 を参照）
- `createNavigation()` が返す `Link` / `redirect` / `useRouter` / `usePathname` はロケールを自動で付与するので、`next/navigation` を直接使うより実装でロケールを意識せずに済む
- ロケールの検出・リダイレクトは `createMiddleware(routing)` を `proxy.ts`（v15 以前は `middleware.ts`）から default export するだけで済む
- `react-i18next` / `react-intl` も使えるが、App Router 統合は `next-intl` が最も滑らか

**ライブラリ比較**:

| ライブラリ | App Router 統合 | ICU 対応 | 型安全 | バンドルサイズ |
|---|---|---|---|---|
| **next-intl** | ★★★ | ✅ | ✅ | 中 |
| **react-i18next** | ★★ | ⚠️ (plugin) | ⚠️ | 大 |
| **react-intl** (FormatJS) | ★★ | ✅ | ✅ | 中〜大 |
| **paraglide-js** | ★★★ | ❌ (代替) | ✅ | 極小（tree-shake） |
| **lingui** | ★★ | ✅ | ✅ | 中 |

**`next-intl` セットアップ**:
```bash
pnpm add next-intl
```

```ts
// i18n/routing.ts
import { defineRouting } from 'next-intl/routing';

export const routing = defineRouting({
  locales: ['ja', 'en'],
  defaultLocale: 'ja',
});
```

```ts
// proxy.ts（Next.js 16+。v15 以前は middleware.ts。どちらも default export でよい）
import createMiddleware from 'next-intl/middleware';
import { routing } from './i18n/routing';

export default createMiddleware(routing);

export const config = {
  // api / _next / _vercel と、拡張子付きのパス（静的ファイル）を除外する
  matcher: '/((?!api|trpc|_next|_vercel|.*\\..*).*)',
};
```

```ts
// i18n/navigation.ts
import { createNavigation } from 'next-intl/navigation';
import { routing } from './routing';

// next/navigation の代わりにこちらを使う（ロケールとローカライズ済みのパスを自動付与）
export const { Link, redirect, usePathname, useRouter, getPathname } = createNavigation(routing);
```

```ts
// global.ts - メッセージキーとロケールの型安全化（next-intl 4 の AppConfig。3 系の IntlMessages ではない）
import { routing } from './i18n/routing';
import messages from './i18n/messages/ja.json';  // 基準にするロケールの JSON を default import する

declare module 'next-intl' {
  interface AppConfig {
    Locale: (typeof routing.locales)[number];
    Messages: typeof messages;
  }
}
```

型安全化の詳細（next-intl 3 系での書き方、未使用キーの検出）は `practices/i18n/message-catalog.md` #4 を参照。

```ts
// i18n/request.ts
import { getRequestConfig } from 'next-intl/server';
import { notFound } from 'next/navigation';

const locales = ['ja', 'en'] as const;

export default getRequestConfig(async ({ requestLocale }) => {
  const locale = await requestLocale;
  if (!locale || !locales.includes(locale as any)) notFound();

  return {
    locale,
    messages: (await import(`./messages/${locale}.json`)).default,
    timeZone: 'Asia/Tokyo',
  };
});
```

```ts
// next.config.ts
import createNextIntlPlugin from 'next-intl/plugin';

const withNextIntl = createNextIntlPlugin('./i18n/request.ts');
export default withNextIntl({});
```

**Server Component での使用**:
```tsx
// app/[locale]/page.tsx
import { getTranslations } from 'next-intl/server';

export default async function HomePage() {
  const t = await getTranslations('home');
  return (
    <main>
      <h1>{t('title')}</h1>
      <p>{t('description', { count: 42 })}</p>
    </main>
  );
}
```

**Client Component での使用**:
```tsx
'use client';
import { useTranslations } from 'next-intl';

function SubmitButton() {
  const t = useTranslations('common');
  return <button>{t('submit')}</button>;
}
```

**Provider 配置（Client 用）**:
```tsx
// app/[locale]/layout.tsx
import { NextIntlClientProvider } from 'next-intl';
import { getMessages } from 'next-intl/server';

export default async function LocaleLayout({ children, params }) {
  const { locale } = await params;
  const messages = await getMessages();

  return (
    <html lang={locale}>
      <body>
        <NextIntlClientProvider messages={messages}>
          {children}
        </NextIntlClientProvider>
      </body>
    </html>
  );
}
```

**選定の判断軸**:
- すでに `react-i18next` を使っている既存プロジェクト → 無理に乗り換えない
- 新規プロジェクト + App Router → `next-intl`
- バンドルサイズ最優先 + i18n が単純 → `paraglide-js`
- 翻訳作業を翻訳者に外注、CAT ツール統合重視 → FormatJS（広く採用）

**出典**:
- [next-intl Docs](https://next-intl.dev/) (next-intl)
- [next-intl: Proxy / middleware](https://next-intl.dev/docs/routing/middleware) (next-intl 公式、`proxy.ts` は Next.js 16 より前は `middleware.ts` だったこと、`createMiddleware(routing)` の default export) ※2026-10-25 確認
- [next-intl: Navigation APIs](https://next-intl.dev/docs/routing/navigation) (next-intl 公式、`createNavigation(routing)` が返す `Link` / `redirect` / `usePathname` / `useRouter` / `getPathname`) ※2026-10-25 確認
- [next-intl: TypeScript augmentation](https://next-intl.dev/docs/workflows/typescript) (next-intl 公式、`declare module 'next-intl'` の `AppConfig` に `Locale` / `Messages` を宣言する) ※2026-10-25 確認
- [react-i18next](https://react.i18next.com/) (i18next)
- [FormatJS](https://formatjs.io/) (FormatJS)
- [paraglide-js](https://inlang.com/m/gerre34r/library-inlang-paraglideJs) (inlang)

**バージョン**: next-intl 4+, Next.js 15+（`proxy.ts` は Next.js 16+）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 4. デフォルトロケールも明示パス（`/en/`）で扱うか、リダイレクトを明示する

「デフォルトロケールはパスを付けない」（`/about`）か「全ロケールにパスを付ける」（`/en/about`）を最初に決める。
中途半端な混在は SEO と運用を壊す。

**根拠**:
- `/about` と `/en/about` が両方存在する設計だと canonical / hreflang の指定が複雑化し、検索エンジンが正規版を誤認する
- どちらかに統一すれば実装も SEO 設定もシンプル
- Google は「ロケールごとに固有 URL」を推奨。デフォルト言語にもパスを付けるのが SEO 上は無難

**3 つのパターンと比較**:

#### パターン A: 全ロケールにパス付き（推奨）
```
/en/about
/ja/about
/  → /en/about にリダイレクト
```
- **メリット**: 一貫した URL 設計、hreflang 設定が単純
- **デメリット**: ルート URL が直接コンテンツを返さない（リダイレクトが必要）

#### パターン B: デフォルトのみパスなし
```
/about       ← デフォルトロケール（en）
/ja/about    ← 非デフォルト
```
- **メリット**: 既存サイトの URL を維持しやすい、デフォルトロケールの URL が短い
- **デメリット**: hreflang に `x-default` を付けておくことが推奨される（Google のガイドでは必須ではない）、URL 構造が非対称

#### パターン C: ドメイン分離
```
example.com/about      → 英語
example.jp/about       → 日本語
```
- **メリット**: 完全な独立、CDN・SEO・法務すべて分離可能
- **デメリット**: ドメイン管理コスト、SSO・認証共有が複雑

**next-intl での設定**:
```ts
// パターン A（推奨）: 全ロケールにパス
// i18n/routing.ts
import { defineRouting } from 'next-intl/routing';

export const routing = defineRouting({
  locales: ['ja', 'en'],
  defaultLocale: 'en',
  localePrefix: 'always',  // 全ロケールにパス、デフォルトでも /en/ になる
});

// パターン B: デフォルトはパスなし
export const routing = defineRouting({
  locales: ['ja', 'en'],
  defaultLocale: 'en',
  localePrefix: 'as-needed',  // デフォルトロケールのみパスなし
});
```

**hreflang の設定（パターン A）**:

canonical と `languages` は #1 の `localeAlternates` でページごとに作る（`x-default` は、検索エンジンがどのロケールにも該当しないユーザーに出す URL）。

```tsx
// app/[locale]/about/page.tsx
import type { Metadata } from 'next';
import { localeAlternates } from '@/i18n/alternates';

export async function generateMetadata({
  params,
}: {
  params: Promise<{ locale: string }>;
}): Promise<Metadata> {
  const { locale } = await params;
  // → canonical: https://example.com/{locale}/about、languages: ja / en / x-default（/en/about）
  return { alternates: localeAlternates(locale, '/about') };
}
```

**移行時の注意**:
- 既存サイトを後から多言語化 → パターン B（既存 URL を維持）
- 新規多言語サイト → パターン A（一貫性優先）
- 既存 URL を破壊する場合は 301 リダイレクトを `next.config.ts` の `redirects()` で永続化

**出典**:
- [Google: URL structures for multilingual sites](https://developers.google.com/search/docs/specialty/international/managing-multi-regional-sites#url-structures) (Google Search Central)
- [next-intl: Routing](https://next-intl-docs.vercel.app/docs/routing) (next-intl)

**バージョン**: Next.js 15+, next-intl 4+
**確信度**: 高
**最終更新**: 2026-10-25
