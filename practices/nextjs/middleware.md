# Next.js Middleware のベストプラクティス

## ルール

### 1. Middleware は認証チェックとリダイレクトに限定する

`middleware.ts` は認証・認可チェック、リダイレクト、ヘッダー付与など軽量な処理のみに使う。

**根拠**:
- Middleware はエッジランタイムで動作し、Node.js APIが制限される
- DBアクセスや重い処理を入れると全リクエストが遅くなる

**コード例**:
```ts
// middleware.ts
import { NextRequest, NextResponse } from 'next/server';

export function middleware(req: NextRequest) {
  const token = req.cookies.get('auth-token')?.value;
  const isProtectedPage = req.nextUrl.pathname.startsWith('/dashboard');

  if (isProtectedPage && !token) {
    const loginUrl = new URL('/login', req.url);
    loginUrl.searchParams.set('from', req.nextUrl.pathname);
    return NextResponse.redirect(loginUrl);
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|api).*)'],
};
```

**出典**:
- [Next.js Docs: Middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. Middleware でのトークン検証はエッジ互掴の方法で行う

JWT 検証には Node.js 標準 API に依存しない `jose` などを使う。

**根拠**:
- Middleware はエッジランタイムで動作するため `jsonwebtoken` などは使えない
- `jose` は Web Crypto API を使用しエッジランタイムと互掴

**コード例**:
```ts
import { jwtVerify } from 'jose';

const JWT_SECRET = new TextEncoder().encode(process.env.JWT_SECRET!);

export async function middleware(req: NextRequest) {
  const token = req.cookies.get('auth-token')?.value;
  if (!token) return NextResponse.redirect(new URL('/login', req.url));

  try {
    const { payload } = await jwtVerify(token, JWT_SECRET);
    const requestHeaders = new Headers(req.headers);
    requestHeaders.set('x-user-id', payload.sub as string);
    return NextResponse.next({ request: { headers: requestHeaders } });
  } catch {
    const response = NextResponse.redirect(new URL('/login', req.url));
    response.cookies.delete('auth-token');
    return response;
  }
}
```

**出典**:
- [Next.js Docs: Edge Runtime](https://nextjs.org/docs/app/api-reference/edge) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. Middleware でのレスポンスヘッダー付与でセキュリティを強化する

CSPや HSTS などのセキュリティヘッダーを Middleware で付与する。

**コード例**:
```ts
export function middleware(req: NextRequest) {
  const nonce = crypto.randomUUID().replace(/-/g, '');
  const csp = [
    `default-src 'self'`,
    `script-src 'self' 'nonce-${nonce}' 'strict-dynamic'`,
    `object-src 'none'`,
  ].join('; ');

  const requestHeaders = new Headers(req.headers);
  requestHeaders.set('x-nonce', nonce);

  const response = NextResponse.next({ request: { headers: requestHeaders } });
  response.headers.set('Content-Security-Policy', csp);
  response.headers.set('X-Frame-Options', 'DENY');
  return response;
}
```

**出典**:
- [Next.js Docs: Content Security Policy](https://nextjs.org/docs/app/building-your-application/configuring/content-security-policy) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. `next-intl` で App Router の多言語対応（i18n）を実装する

`next-intl` v4 を使い、App Router の Middleware でロケール検出・リダイレクトを行い、
型安全な翻訳関数 `getTranslations`（Server Component）・`useTranslations`（Client Component）で翻訳文字列を取得する。

**根拠**:
- App Router では `next/navigation` のナビゲーション関数をそのまま使うとロケールパスを意識した実装が複雑になる
- `next-intl` の `createNavigation()` で生成した `Link`・`redirect`・`useRouter` はロケールを自動付与するため、実装時にロケールを意識する必要がなくなる
- `IntlMessages` インターフェースを拡張することで、翻訳キーが TypeScript で型チェックされ、存在しないキーのアクセスがコンパイルエラーになる
- Server Component と Client Component で異なる API（`getTranslations` / `useTranslations`）を使い分けることでサーバー/クライアントの責務が明確になる

**コード例**:
```ts
// i18n/routing.ts
import { defineRouting } from 'next-intl/routing';

export const routing = defineRouting({
  locales: ['ja', 'en'],
  defaultLocale: 'ja',
});

// i18n/navigation.ts
import { createNavigation } from 'next-intl/navigation';
import { routing } from './routing';

// next/navigation の代わりにこちらを使う（ロケールを自動付与）
export const { Link, redirect, usePathname, useRouter } = createNavigation(routing);
```

```ts
// middleware.ts
import createMiddleware from 'next-intl/middleware';
import { routing } from '@/i18n/routing';

export default createMiddleware(routing);

export const config = {
  matcher: ['/((?!_next|_vercel|.*\\..*).*)'],
};
```

```tsx
// app/[locale]/layout.tsx
import { NextIntlClientProvider } from 'next-intl';
import { getMessages } from 'next-intl/server';

export default async function LocaleLayout({
  children,
  params: { locale },
}: {
  children: React.ReactNode;
  params: { locale: string };
}) {
  const messages = await getMessages();
  return (
    <NextIntlClientProvider messages={messages}>
      {children}
    </NextIntlClientProvider>
  );
}

// app/[locale]/page.tsx - Server Component での翻訳
import { getTranslations } from 'next-intl/server';

export default async function HomePage() {
  const t = await getTranslations('home');
  return <h1>{t('title')}</h1>;
}

// components/NavBar.tsx - Client Component での翻訳
'use client';
import { useTranslations } from 'next-intl';

export function NavBar() {
  const t = useTranslations('common');
  return <nav>{t('home')}</nav>;
}
```

```ts
// types/global.d.ts - 翻訳キーの型安全化
import ja from '@/messages/ja.json';

declare global {
  interface IntlMessages extends Messages {}
}

type Messages = typeof ja;
```

**出典**:
- [next-intl を使ってNext.js App RouterのI18n対応をする](https://qiita.com/nhatcaofedev/items/387cda18f797938acb6b) (Qiita nhatcaofedev / 2026-05) ※2026-05-06に実際にfetch成功
- [next-intl Docs: App Router Setup](https://next-intl-docs.vercel.app/docs/getting-started/app-router) (next-intl公式)

**バージョン**: next-intl 4+, Next.js 15+
**確信度**: 高
**最終更新**: 2026-05-06
