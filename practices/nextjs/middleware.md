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

#### 追加根拠 (2026-05-16) — 手動取り込み

新たに sstf-5461-admin-app チームドキュメント（原典: akfm_sato 氏 Zenn book）から以下の知見を追加:

**v16 で `middleware.ts` は非推奨化され、`proxy.ts` に名称変更された**:
v16 では `middleware.ts` が非推奨になり、新規プロジェクトでは `proxy.ts` を使う。ファイル名・関数名・ランタイムが変わる。`middleware.ts` は当面残せるが将来削除予定なので、可能なら設計を見直す。

| 項目 | v15 まで（middleware） | v16+（proxy） |
|---|---|---|
| ファイル名 | `middleware.ts` | `proxy.ts` |
| 関数名 | `middleware` | `proxy` |
| ランタイム | Edge（デフォルト）/ Node.js | **Node.js 固定**（Edge 非サポート） |
| 配置の単位 | プロジェクトに 1 つ | プロジェクトに 1 つ |

ランタイムが Node.js 固定になることで、`jsonwebtoken` など Node.js API に依存するライブラリも使えるようになる一方、Edge ランタイム前提の最適化（地理ベースのリダイレクトなど）は失われる。

**Proxy（旧 middleware）は楽観的チェックに留め、最終認可は DAL で行う（CVE-2025-29927 の教訓）**:
2025 年 3 月に Next.js の middleware で **CVE-2025-29927**（CVSS 9.1）が公表され、`x-middleware-subrequest` ヘッダを送るだけで middleware の認証チェックを完全にバイパスできる脆弱性が判明した。教訓は「middleware（v16 では proxy）単独で守られていると考えない」「認可は DAL に必ず置く（defense-in-depth）」。Proxy は楽観的チェック（速い拒否）、`page.tsx` は粗いガード、**DAL は最終的な砦**という三層を守る。

**コード例**:
```ts
// proxy.ts (v16+)
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function proxy(request: NextRequest) {
  // ルーティング前に走る。重いロジックや DB アクセスは置かない。
  // 楽観的チェック（クッキー存在確認・リダイレクト・A/B 振り分け・ヘッダー付与など）に留める
  return NextResponse.next();
}

// 公式 codemod でファイル名と関数名を一括変更
// npx @next/codemod@canary upgrade latest
```

**出典**:
- [Next.jsの考え方 / 5.2 Proxy（旧 middleware）・6.4 認可チェックを多段で組む](https://zenn.dev/akfm/books/nextjs-basic-principle)
- [Next.js Docs: middleware → proxy 移行](https://nextjs.org/docs/messages/middleware-to-proxy)

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**確信度**: 既存（高）→ 高

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

---

### 5. Next.js のセキュリティアドバイザリを追跡し、patch バージョンを即座に適用する

Next.js のセキュリティリリースには Middleware 認可バイパス・DoS・XSS 等、WAF では防御できない脆弱性が含まれるため、公式発表後すぐにバージョンを更新する。patch 更新のみが唯一の完全な対策であり、設定変更や WAF ルールでは代替不可能な点に注意する。

**根拠**:
- 2026年5月リリースで13件のCVEが修正: Middleware・プロキシバイパス（5件）、DoS（3件）、SSRF（1件）、キャッシュポイズニング（2件）、XSS（2件）
- 公式発表: "patching represents the only complete mitigation, with no WAF-level workarounds available"
- Next.js 13.x / 14.x を使用中の場合は 15.5.18 または 16.2.6 への移行が必要
- `vercel.com/changelog` と GitHub Security Advisories を購読してアドバイザリを追跡する

**コード例**:
```sh
# 現在のバージョン確認
npm list next

# 最新 patch へ更新
npm install next@latest

# または特定バージョンを指定
npm install next@15.5.18   # 15.x 系
npm install next@16.2.6    # 16.x 系
```

**出典引用**:
> "upgrade immediately" / "patching represents 'the only complete mitigation,' with no WAF-level workarounds available"
> ([Next.js May 2026 security release](https://vercel.com/changelog/next-js-may-2026-security-release), セクション "Recommended Actions") ※2026-05-08に実際にfetch成功

**バージョン**: Next.js 13.x〜16.x（影響）→ 15.5.18 / 16.2.6 以上（修正済み）
**確信度**: 高
**最終更新**: 2026-05-08

#### 追加根拠 (2026-05-09)

新たに以下の記事で、2026年5月の脆弱性の根本原因と長期的な防止設計が解説された:
- [RSC Is Not the Input Boundary](https://dev.to/lazarv/rsc-is-not-the-input-boundary-2aao) (dev.to / lazarv, 2026-05-09) ※2026-05-09に実際にfetch成功

**出典引用**:
> "Server Function input payloads need protocol-level validation"
> (セクション "Core Argument")

この記事では、脆弱性の根本原因は「Server Function のハンドラ内でバリデーションを行っていた（デシリアライズ完了後）」にあると分析されている。
Server Functions は任意のペイロードを受け取れるパブリックエンドポイントであるため、次のアーキテクチャ原則が重要になる:

- **Server Action はハンドラ冒頭で即座に Zod 検証を行う**: ペイロードを処理する前に型・サイズ・値域を検証する
- **大きなペイロードを想定する場合はサイズ上限を設ける**: `Content-Length` ヘッダーや Middleware でのボディサイズ制限を組み合わせる
- **ビジネスロジックより先に認証・認可・バリデーションを行う**: アーリーリターンで後続処理を保護する

```typescript
// ✅ Good: ハンドラ先頭でスキーマ検証（入口保護）
'use server'
import { z } from 'zod'

const UploadSchema = z.object({
  displayName: z.string().min(1).max(80),
})

export async function uploadAvatar(formData: FormData) {
  const result = UploadSchema.safeParse({
    displayName: formData.get('displayName'),
  })
  if (!result.success) return { error: '入力値が不正です' }
  // 以降は検証済みデータのみ処理
  await processUpload(result.data)
}
```

**確信度**: 既存（高）→ 高（根本原因の理解が深まった）
