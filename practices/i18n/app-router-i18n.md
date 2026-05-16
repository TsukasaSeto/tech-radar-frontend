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
```tsx
export async function generateMetadata({ params }): Promise<Metadata> {
  const { locale } = await params;
  return {
    alternates: {
      canonical: `https://example.com/${locale}`,
      languages: {
        'ja': 'https://example.com/ja',
        'en': 'https://example.com/en',
        'x-default': 'https://example.com/en',
      },
    },
  };
}
```

**出典**:
- [Next.js Docs: Internationalization](https://nextjs.org/docs/app/building-your-application/routing/internationalization) (Next.js 公式)
- [Google: Locale-aware crawling](https://developers.google.com/search/docs/specialty/international/managing-multi-regional-sites) (Google Search Central)
- [web.dev: Tell Google about localized versions of your page](https://web.dev/articles/hreflang) (web.dev)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 2. Middleware で Accept-Language を検出し、未指定なら正規 URL にリダイレクトする

`middleware.ts` で `/` 等のロケールなし URL を `Accept-Language` 解析でリダイレクトする。
`/en/...` のような有効ロケールはそのまま通過させる。

**根拠**:
- Accept-Language ヘッダーはブラウザの言語設定をサーバーが知る標準的手段
- 初回訪問時に「正しい言語にリダイレクト」+「明示パスでブックマーク可能」の両立
- BCP 47 形式のパースは `@formatjs/intl-localematcher` 等のライブラリに任せる
- `middleware` でリダイレクトすれば Server Component 側の処理は単純化できる

**Middleware の実装**:
```ts
// middleware.ts
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

export function middleware(request: NextRequest) {
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
    // _next/static, _next/image, api, favicon は除外
    '/((?!api|_next|favicon.ico|sitemap.xml|robots.txt).*)',
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

**出典**:
- [Next.js Docs: Internationalization - Routing](https://nextjs.org/docs/app/building-your-application/routing/internationalization#routing-overview) (Next.js 公式)
- [@formatjs/intl-localematcher](https://formatjs.io/docs/polyfills/intl-localematcher/) (FormatJS)
- [BCP 47](https://www.rfc-editor.org/info/bcp47) (IETF)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 3. `next-intl` を i18n ライブラリの第一選択にする

App Router での i18n には `next-intl` を採用する。
ICU MessageFormat サポート・型安全・Server Component 統合の 3 点で優位。

**根拠**:
- `next-intl` は App Router を最初から想定して設計されている（Server Component / Client Component 両対応）
- ICU MessageFormat（複数形・性別・選択）を標準サポート
- TypeScript 型生成でメッセージキーを型レベルで強制できる
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
- [next-intl Docs](https://next-intl-docs.vercel.app/) (next-intl)
- [react-i18next](https://react.i18next.com/) (i18next)
- [FormatJS](https://formatjs.io/) (FormatJS)
- [paraglide-js](https://inlang.com/m/gerre34r/library-inlang-paraglideJs) (inlang)

**バージョン**: next-intl 3+, Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-16

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
- **デメリット**: hreflang の `x-default` 指定が必須、URL 構造が非対称

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
```tsx
export async function generateMetadata({ params }): Promise<Metadata> {
  return {
    alternates: {
      canonical: `https://example.com/${locale}`,
      languages: {
        'en': 'https://example.com/en',
        'ja': 'https://example.com/ja',
        'x-default': 'https://example.com/en',  // 検索エンジンが「どれにも該当しない」時に使う
      },
    },
  };
}
```

**移行時の注意**:
- 既存サイトを後から多言語化 → パターン B（既存 URL を維持）
- 新規多言語サイト → パターン A（一貫性優先）
- 既存 URL を破壊する場合は 301 リダイレクトを `next.config.ts` の `redirects()` で永続化

**出典**:
- [Google: URL structures for multilingual sites](https://developers.google.com/search/docs/specialty/international/managing-multi-regional-sites#url-structures) (Google Search Central)
- [next-intl: Routing](https://next-intl-docs.vercel.app/docs/routing) (next-intl)

**バージョン**: Next.js 13+, next-intl 3+
**確信度**: 高
**最終更新**: 2026-05-16
