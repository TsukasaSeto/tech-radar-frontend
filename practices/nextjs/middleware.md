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

### 2. Middleware でのトークン検証はエッジ互掰の方法で行う

JWT 検証には Node.js 標準 API に依存しない `jose` などを使う。

**根拠**:
- Middleware はエッジランタイムで動作するため `jsonwebtoken` などは使えない
- `jose` は Web Crypto API を使用しエッジランタイムと互掰

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
