# Next.js Proxy（旧 Middleware）のベストプラクティス

Next.js 16 で `middleware.ts` / `export function middleware` は非推奨になり、`proxy.ts` / `export function proxy` に改名された。このファイルは `proxy.ts` を基準に書く。`proxy.ts` のランタイムは Node.js 固定で、`runtime` を指定するとエラーになる。Edge ランタイムを使い続ける場合は `middleware.ts` を残す（v15 以前も `middleware.ts`）。

## ルール

### 1. Proxy（旧 Middleware）は楽観的な認証チェックとリダイレクトに限定する

`proxy.ts` は Cookie の有無を見る楽観的な認証チェック、リダイレクト、ヘッダー付与など軽量な処理のみに使う。最終的な認可は page と DAL で行う。

**根拠**:
- Proxy はプリフェッチを含む全ルートで実行されるので、DB アクセスや重い処理を入れると全リクエストが遅くなる。公式は Cookie からセッションを読むだけの楽観的チェックにとどめ、DB でのチェックを避けるよう求めている
- Server Function（Server Action）は独立したルートではなく、使われているルートへの POST として処理される。matcher で除外したパスでは Server Function の呼び出しも Proxy を通らないので、Proxy だけに頼らず各 Server Function の中で認証・認可を確認する（公式）
- Proxy は Node.js ランタイム固定（v15 以前の `middleware.ts` は Edge が既定）。Edge 前提のコードを移すときは後述の注意点を確認する

**コード例**:
```ts
// proxy.ts（v15 以前は middleware.ts / export function middleware）
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

const protectedPrefixes = ['/dashboard'];

export function proxy(req: NextRequest) {
  const { pathname } = req.nextUrl;
  const isProtectedPage = protectedPrefixes.some((prefix) => pathname.startsWith(prefix));
  const hasSession = req.cookies.has('session'); // 楽観的チェック: Cookie の有無だけを見る

  if (isProtectedPage && !hasSession) {
    const loginUrl = new URL('/login', req.url);
    loginUrl.searchParams.set('from', pathname);
    return NextResponse.redirect(loginUrl);
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|api).*)'],
};
```

**出典**:
- [Next.js Docs: proxy.js](https://nextjs.org/docs/app/api-reference/file-conventions/proxy) (Next.js 公式、Node.js ランタイム固定・`runtime` 指定はエラー・Server Function は matcher の除外の影響を受ける) ※2026-10-25 確認
  > "Always verify authentication and authorization inside each Server Function rather than relying on Proxy alone."
- [Next.js Docs: Authentication — Optimistic checks with Proxy](https://nextjs.org/docs/app/guides/authentication#optimistic-checks-with-proxy-optional) (Next.js 公式、Proxy では Cookie からのセッション読み取りだけを行い DB チェックを避ける) ※2026-10-25 確認

**バージョン**: Next.js 16+（`proxy.ts`）。v15 以前は `middleware.ts`
**確信度**: 高
**最終更新**: 2026-10-25

#### 追加根拠 (2026-05-16) — 手動取り込み

新たに sstf-5461-admin-app チームドキュメント（原典: akfm_sato 氏 Zenn book）から以下の知見を追加:

**v16 で `middleware.ts` は非推奨化され、`proxy.ts` に名称変更された**:
v16 では `middleware.ts` が非推奨になり、新規プロジェクトでは `proxy.ts` を使う。ファイル名・関数名・ランタイムが変わる。`middleware.ts` は当面残せるが将来削除予定なので、可能なら設計を見直す。

| 項目 | v15 まで（middleware） | v16+（proxy） |
|---|---|---|
| ファイル名 | `middleware.ts` | `proxy.ts` |
| 関数名 | `middleware` | `proxy` |
| ランタイム | Edge（デフォルト）/ Node.js | **Node.js 固定**（Edge 非サポート。`runtime` を書くとエラー） |
| 配置の単位 | プロジェクトに 1 つ | プロジェクトに 1 つ |

ランタイムが Node.js 固定になることで、`jsonwebtoken` など Node.js API に依存するライブラリも使えるようになる一方、Edge ランタイム前提の最適化（地理ベースのリダイレクトなど）は失われる。Edge ランタイムを使い続けたい場合は `proxy.ts` に改名せず `middleware.ts` を残す（公式）。移行時は `request.geo` / `request.ip`、明示的な `runtime: 'edge'` 指定、ストリーミング `Response`、WASM import に依存したコードが Node.js ランタイム下でも動くか個別に確認する（これらは移行を即座に壊すわけではないが、出荷前に必ずテストすべき挙動として報告されている）。

**Proxy（旧 middleware）は楽観的チェックに留め、最終認可は DAL で行う（CVE-2025-29927 の教訓）**:
2025 年 3 月に Next.js の middleware で **CVE-2025-29927**（CVSS 9.1）が公表され、`x-middleware-subrequest` ヘッダを送るだけで middleware の認証チェックを完全にバイパスできる脆弱性が判明した。教訓は「middleware（v16 では proxy）単独で守られていると考えない」「認可は DAL に必ず置く（defense-in-depth）」。Proxy は楽観的チェック（速い拒否）、`page.tsx` は粗いガード、**DAL は最終的な砦**という三層を守る。

**楽観的チェック（proxy）と厳密チェック（layout/DAL）が「別の真実」を参照すると無限リダイレクトを起こす**:
三層防御はそれぞれ異なる判定ロジックを持つが、判定に使うデータソースが食い違うと、各層が互いを打ち消し合う無限リダイレクトループに陥ることがある。実例では proxy がクッキーの有無だけを見る楽観的チェックで通過させ、`layout.tsx` が `getServerSession()` でDB上のセッション実体を検証する厳密チェックで弾く構成になっていた。クッキーは残っているがDBセッションが失効している状態では、proxy が `/login` → `/portal` へ通し、layout が `/portal` → `/login` へ差し戻す、という往復が無限に続く。対策は「楽観的チェックと厳密チェックに、同じ質問（＝同じ真偽の答え）をさせない」こと — 各層が判定できる範囲を明確に分離し（例: proxy は「クッキーが存在するか」だけを見て存在しなければ確実に弾き、セッションの有効性判定は必ず一箇所〈DAL〉に一本化する）、複数層が同じ観点で矛盾した判定を出さないようにする。なお、厳密チェックはそもそも layout に置かず page と DAL で行う（`routing.md` #8）。

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
- [Next.js Docs: How to upgrade to version 16 — `middleware` to `proxy`](https://nextjs.org/docs/app/guides/upgrading/version-16#middleware-to-proxy) (Next.js 公式、Edge を使い続けるなら `middleware` を残す) ※2026-10-25 確認
  > "The `edge` runtime is **NOT** supported in `proxy`. The `proxy` runtime is `nodejs`, and it cannot be configured. If you want to continue using the `edge` runtime, keep using `middleware`."
- [Migrating middleware.ts to proxy.ts in Next.js 16](https://dev.to/dev_encyclopedia/migrating-middlewarets-to-proxyts-in-nextjs-16-heres-what-actually-changes-3gbd) (dev.to、Node.js ランタイム下で個別確認が必要な挙動を追加) ※2026-07-26 fetch
  > "None of these break the migration outright, but each one is worth testing under Node.js before you ship."
  > (セクション "Patterns Requiring Testing")
- [Next.js 16で無限リダイレクト｜認証チェック二重化の罠](https://zenn.dev/miyoki_labs/articles/optimistic-vs-strict-auth-redirect-loop) (Zenn、楽観的チェックと厳密チェックの不一致による無限リダイレクトの実例) ※2026-08-11に実際にfetch成功
  > "この2つが永遠に押し問答を続けます"
  > (セクション "真因：楽観チェックと実チェックが違う答えを出していた")
  > "楽観チェックと実チェックに、同じ質問をさせない"
  > (セクション "どう直したか：判定の一本化")

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**確信度**: 既存（高）→ 高
**最終更新**: 2026-10-25

---

### 2. Proxy でのトークン検証はランタイムを問わない `jose` で行い、検証結果は下流で認可の根拠にしない

JWT 検証には Node.js 標準 API に依存しない `jose` などを使う。v16 の `proxy.ts` は Node.js ランタイムなので `jsonwebtoken` も動くが、`middleware.ts` を Edge で使い続ける構成や、Edge と Node.js の両方で動かしたいコードでは `jose` が必要になる。
Proxy で検証した結果をリクエストヘッダー（`x-user-id` など）で下流に渡しても、下流はそれを認可の根拠にしない。page / Server Action / DAL で `verifySession()` により再検証する。

**根拠**:
- `jose` は Web Crypto API を使用し、Edge ランタイムと Node.js ランタイムの両方で動く。公式の Authentication ガイドもセッション管理ライブラリとして Jose を挙げている
- matcher の対象外パスや、Proxy がバイパスされた経路（CVE-2025-29927 のようなフレームワークの脆弱性を含む）では、クライアントが同名のリクエストヘッダーを送れる。Proxy が付けたヘッダーを信用すると、Proxy が動かなかったときに認可が破られる
- 認証用ページ（`/login` など）を保護対象から外さないと、トークンがないときに `/login` 自身が `/login` へリダイレクトされ、無限リダイレクトになる

**コード例**:
```ts
// proxy.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { jwtVerify } from 'jose';

const secret = process.env.JWT_SECRET;
if (!secret) throw new Error('JWT_SECRET is not set');
const JWT_SECRET = new TextEncoder().encode(secret);

const publicPaths = ['/login', '/signup'];

export async function proxy(req: NextRequest) {
  const { pathname } = req.nextUrl;
  // 認証用ページ自身は対象外にする（含めると /login → /login の無限リダイレクトになる）
  if (publicPaths.some((path) => pathname.startsWith(path))) return NextResponse.next();

  const token = req.cookies.get('auth-token')?.value;
  if (!token) return NextResponse.redirect(new URL('/login', req.url));

  try {
    await jwtVerify(token, JWT_SECRET, { algorithms: ['HS256'] });
    // 検証できたら通すだけにする。下流は page / Server Action / DAL で verifySession() により再検証する
    return NextResponse.next();
  } catch {
    const response = NextResponse.redirect(new URL('/login', req.url));
    response.cookies.delete('auth-token');
    return response;
  }
}

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
};

// Bad: 検証結果をリクエストヘッダーで渡し、下流がそのヘッダーだけで本人と見なす
// requestHeaders.set('x-user-id', payload.sub as string);
// → matcher の対象外パスや Proxy がバイパスされた経路では、クライアントが x-user-id を偽装できる
```

**出典**:
- [Next.js Docs: Edge Runtime](https://nextjs.org/docs/app/api-reference/edge) (Next.js公式)
- [Next.js Docs: Authentication](https://nextjs.org/docs/app/guides/authentication) (Next.js 公式、Proxy は楽観的チェックのみ・Proxy は Node.js ランタイムで動く・セッション管理ライブラリとして Jose) ※2026-10-25 確認
  > "While Proxy can be useful for initial checks, it should not be your only line of defense in protecting your data."

**バージョン**: Next.js 16+（`proxy.ts`）。`middleware.ts` を Edge で使う場合も同じ
**確信度**: 高
**最終更新**: 2026-10-25

---

### 3. Proxy では nonce 付きの CSP だけを付与し、リクエストとレスポンスの両方に設定する

Proxy で付与するセキュリティヘッダーは、リクエストごとに nonce が変わる `Content-Security-Policy` に限る。`X-Frame-Options` や HSTS などの静的ヘッダーは `next.config.ts` の `headers()` で配信する（正本は `security-headers.md` #1）。
CSP はリクエストヘッダーとレスポンスヘッダーの両方に設定する。nonce を使うページは動的レンダリングが必須で、Partial Prerendering（PPR）の静的シェルとは併用できない。ディレクティブの設計・コード例・よくある失敗は `practices/security/csp.md` #6 を正本とする。

**根拠**:
- Next.js はレンダリング時にリクエストの `Content-Security-Policy` ヘッダーから nonce を取り出し、フレームワークのスクリプトや `<Script>` に自動で付与する。レスポンスにだけ CSP を付け、リクエストには `x-nonce` しか渡さないと、`'strict-dynamic'` の下でフレームワークのスクリプトがブロックされる（公式）
- 静的ヘッダーまで Proxy で付けると値が `next.config.ts` 側と食い違いやすく（`X-Frame-Options` の `DENY` と `SAMEORIGIN` など）、Proxy の実行コストも増える

**コード例**:
```ts
// proxy.ts の要点（全体は security/csp.md #6）
const requestHeaders = new Headers(request.headers);
requestHeaders.set('x-nonce', nonce);
requestHeaders.set('Content-Security-Policy', csp); // リクエスト側にも設定する（Next.js がここから nonce を読む）

const response = NextResponse.next({ request: { headers: requestHeaders } });
response.headers.set('Content-Security-Policy', csp);
return response;
```

**出典**:
- [Next.js Docs: Content Security Policy](https://nextjs.org/docs/app/guides/content-security-policy) (Next.js 公式、リクエストとレスポンスの両方への CSP 設定・動的レンダリング必須・PPR 非互換) ※2026-10-25 確認
  > "Next.js applies nonces during **server-side rendering**, based on the CSP header present in the request."

**バージョン**: Next.js 16+（`proxy.ts`）。v15 以前は `middleware.ts` で同じ書き方
**確信度**: 高
**最終更新**: 2026-10-25

---

### 4. `next-intl` で App Router の多言語対応（i18n）を実装する

`next-intl` v4 のロケール検出・リダイレクトは、`proxy.ts`（v15 以前は `middleware.ts`）で `createMiddleware(routing)` を default export して行う。
`next-intl` を選ぶ理由、`defineRouting` / `createNavigation()` / `AppConfig` の型拡張、`i18n/request.ts` とプラグインの設定、Server / Client Component での翻訳関数の使い方は `practices/i18n/app-router-i18n.md` #3 を参照（このルールでは Proxy ファイルに関わる部分だけを扱う）。

**根拠**:
- next-intl の公式ドキュメントは、Next.js 16 では `proxy.ts` に `createMiddleware(routing)` を default export する形を示している
- matcher で `api` / `_next` / `_vercel` と拡張子付きのパスを除外しないと、静的ファイルや API までロケール付きのパスへリダイレクトされる（matcher の詳細は `practices/i18n/app-router-i18n.md` #2 を参照）

**コード例**:
```ts
// proxy.ts（v15 以前は middleware.ts）
import createMiddleware from 'next-intl/middleware';
import { routing } from '@/i18n/routing';

export default createMiddleware(routing);

export const config = {
  matcher: '/((?!api|trpc|_next|_vercel|.*\\..*).*)',
};
```

**出典**:
- [next-intl を使ってNext.js App RouterのI18n対応をする](https://qiita.com/nhatcaofedev/items/387cda18f797938acb6b) (Qiita nhatcaofedev / 2026-05) ※2026-05-06に実際にfetch成功
- [next-intl Docs: App Router Setup](https://next-intl-docs.vercel.app/docs/getting-started/app-router) (next-intl公式)
- [next-intl Docs: TypeScript augmentation](https://next-intl.dev/docs/workflows/typescript) (next-intl 公式、`AppConfig` の拡張) ※2026-10-25 確認
- [next-intl Docs: Middleware](https://next-intl.dev/docs/routing/middleware) (next-intl 公式、`proxy.ts` での設定と matcher) ※2026-10-25 確認
  > "`proxy.ts` was called `middleware.ts` up until Next.js 16."

**バージョン**: next-intl 4+, Next.js 15+（ファイル名は 16+ で `proxy.ts`、15 以前は `middleware.ts`）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 5. Next.js のセキュリティアドバイザリを追跡し、patch バージョンを即座に適用する

Next.js のセキュリティリリースには Proxy（旧 Middleware）の認可バイパス・DoS・XSS 等、WAF では防御できない脆弱性が含まれるため、公式発表後すぐにバージョンを更新する。告知の追跡方法・最新の修正版・適用計画の立て方は `practices/security/dependency-security.md` #10 を正本とする。
このルールの独自の観点は、(1) 更新は使用中のメジャー系列の中で行う（`next@latest` はメジャーを跨ぐことがある）、(2) フレームワークの更新とアプリ側の入力検証は別々の防御として両方行う、の 2 点。

**根拠**:
- 2026年5月リリースの公式発表は「patch 適用が唯一の完全な対策で、WAF レベルの回避策はない」としていた。脆弱性の修正はフレームワークの更新でしか得られない
- `npm install next@latest` は、15 系を使っていても 16 系へのメジャー更新になりうる。セキュリティ修正の適用と、メジャー移行（upgrade ガイドに沿った作業）は分けて行う
- Server Function の入力ペイロードは、プロトコルレベル（デシリアライズ前）での検証が必要という指摘がある（下記の追加根拠）。デシリアライズ処理自体の脆弱性はアプリのハンドラ内の検証では防げないので、フレームワークの更新で塞ぐ。デシリアライズ後のアプリ側の入力検証（Zod など）は別の防御として行う（`server-actions.md` #5）

**コード例**:
```sh
# 現在のバージョン確認
npm list next

# 使用中のメジャー系列の最新版へ更新する（next@latest はメジャーを跨ぐことがあるので使わない）
npm install next@16   # 16.x 系
npm install next@15   # 15.x 系
# 修正版の番号は告知で確認する（security/dependency-security.md #10）。メジャー移行は upgrade ガイドの手順で別に行う
```

**出典引用**:
> "upgrade immediately" / "patching represents 'the only complete mitigation,' with no WAF-level workarounds available"
> ([Next.js May 2026 security release](https://vercel.com/changelog/next-js-may-2026-security-release), セクション "Recommended Actions") ※2026-05-08に実際にfetch成功

**バージョン**: Next.js 13.x〜16.x（2026-05 リリースの修正版は 15.5.18 / 16.2.6。以降の修正版は `practices/security/dependency-security.md` #10 を参照）
**確信度**: 高
**最終更新**: 2026-10-25

#### 追加根拠 (2026-05-09)

新たに以下の記事で、2026年5月の脆弱性の根本原因と長期的な防止設計が解説された:
- [RSC Is Not the Input Boundary](https://dev.to/lazarv/rsc-is-not-the-input-boundary-2aao) (dev.to / lazarv, 2026-05-09) ※2026-05-09に実際にfetch成功

**出典引用**:
> "Server Function input payloads need protocol-level validation"
> (セクション "Core Argument")

この記事では、脆弱性の根本原因は「Server Function のハンドラ内でバリデーションを行っていた（デシリアライズ完了後）」にあると分析されている。デシリアライズ前の検証はフレームワーク側の責務なので、アプリ側では更新を速やかに適用する。ハンドラ冒頭での入力検証・サイズ上限・ビジネスロジックより先に認証・認可を行う、というアプリ側の対策とコード例は `server-actions.md` #5 に移した。

**確信度**: 既存（高）→ 高（根本原因の理解が深まった）

---

### 6. Vercel では Middleware が本文を読まない場合 `skipMiddlewareRequestBody` で転送を省く

Vercel の Routing Middleware（Next.js の `proxy.ts`、v15 以前の `middleware.ts` を含む）は、既定ではリクエストボディを Middleware に転送する。
Middleware が認証・リダイレクト・ヘッダー付与のみでボディを読まないなら、`vercel.json`（または `vercel.ts`）で `skipMiddlewareRequestBody` を有効にしてボディ転送を省く。ボディは Vercel Functions と rewrite 先には従来どおり届く。

**根拠**:
- Middleware 呼び出しへのボディ転送が減り、Fast Origin Transfer の使用量が下がる
- 大きなボディを持つリクエストで TTFB が改善しうる
- Rule #1（Proxy は楽観的な認証チェックとリダイレクトに限定）に従っていればボディを読む必要は通常なく、設計方針と整合する
- 既定値は `false` のため、有効化するまで既存プロジェクトには影響しない

**注意**:
- Middleware 内で `request.json()` / `request.formData()` / `request.text()` 等を呼ぶ場合は有効化しない
- 設定変更後は再デプロイが必要

**コード例**:
```json
// Good: vercel.json（Middleware がボディを読まない場合のみ）
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "skipMiddlewareRequestBody": true
}
```

```typescript
// Good: vercel.ts（@vercel/config 導入済みの場合）
import type { VercelConfig } from '@vercel/config/v1';

export const config: VercelConfig = {
  skipMiddlewareRequestBody: true,
};
```

**出典引用**:
> "Enable this option only if your Routing Middleware doesn't read request bodies."
> ([Skip sending request bodies to Routing Middleware](https://vercel.com/changelog/skip-sending-request-bodies-to-routing-middleware), Vercel Changelog, セクション "Requirement") ※2026-10-08に実際にfetch成功

**出典**:
- [Skip sending request bodies to Routing Middleware](https://vercel.com/changelog/skip-sending-request-bodies-to-routing-middleware) (Vercel Changelog 2026-10-08) ※2026-10-08 fetch

**バージョン**: Vercel デプロイ限定（セルフホストには適用されない）
**確信度**: 高
**最終更新**: 2026-10-25
