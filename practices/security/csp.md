# Content Security Policy のベストプラクティス

XSS の最後の防衛線として CSP を運用する。「設定して終わり」ではなく、段階導入・違反監視・ポリシー更新のサイクルで運用する。

## ルール

### 1. CSP は nonce ベースで設計し、`'unsafe-inline'` を避ける

`script-src` には `'nonce-xxx'` を使い、サーバーが生成したワンタイム nonce を一致するインラインスクリプトのみ許可する。
`'unsafe-inline'` は CSP を実質無効化するため使わない。

**根拠**:
- `'unsafe-inline'` を許可すると XSS で注入されたスクリプトも実行可能になり、CSP の意味がなくなる
- nonce は HTTP レスポンスごとにランダム生成され、攻撃者は予測できない
- Google・MDN は `'strict-dynamic'` + nonce を CSP Level 3 のベストプラクティスとして推奨
- `'unsafe-eval'` も同様に避ける（Vue.js テンプレート等で必要なら明示的に許可）

**コード例（Next.js Middleware）**:
```ts
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const nonce = Buffer.from(crypto.randomUUID()).toString('base64');

  const cspHeader = `
    default-src 'self';
    script-src 'self' 'nonce-${nonce}' 'strict-dynamic' https:;
    style-src 'self' 'nonce-${nonce}';
    img-src 'self' blob: data: https:;
    font-src 'self' https://fonts.gstatic.com;
    object-src 'none';
    base-uri 'self';
    form-action 'self';
    frame-ancestors 'none';
    upgrade-insecure-requests;
  `.replace(/\s{2,}/g, ' ').trim();

  const requestHeaders = new Headers(request.headers);
  requestHeaders.set('x-nonce', nonce);
  requestHeaders.set('Content-Security-Policy', cspHeader);

  const response = NextResponse.next({ request: { headers: requestHeaders } });
  response.headers.set('Content-Security-Policy', cspHeader);
  return response;
}

export const config = {
  matcher: [
    { source: '/((?!api|_next/static|_next/image|favicon.ico).*)', missing: [
      { type: 'header', key: 'next-router-prefetch' },
      { type: 'header', key: 'purpose', value: 'prefetch' },
    ]},
  ],
};
```

**Bad パターン**:
```
# Bad: unsafe-inline は CSP を無効化する
Content-Security-Policy: script-src 'self' 'unsafe-inline';

# Bad: ホスト allowlist は迂回されやすい（CDN や JSONP 経由）
Content-Security-Policy: script-src 'self' https://cdn.example.com https://cdnjs.cloudflare.com;
```

**出典**:
- [MDN: Content-Security-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy) (MDN Web Docs)
- [W3C CSP Level 3](https://www.w3.org/TR/CSP3/) (W3C)
- [Google Web Fundamentals: Strict CSP](https://web.dev/articles/strict-csp) (web.dev)
- [OWASP: Content Security Policy Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html) (OWASP)

**バージョン**: CSP Level 3 / Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 2. Next.js Middleware で nonce を生成し、Script / Style に注入する

Next.js では Middleware で nonce を生成し、`headers()` 経由で Server Components から取得して `<Script nonce={nonce}>` に渡す。
クライアント側で nonce を生成しない（攻撃者が読み取れる）。

**根拠**:
- App Router の Middleware はリクエスト単位で実行されるため、リクエストごとに新しい nonce を生成できる
- Server Component は `headers()` API で Middleware が設定した値を読み取れる
- Next.js の `<Script>` コンポーネントは `nonce` prop をサポートし、自動で CSP 準拠になる
- インライン `<script>` 内で `dangerouslySetInnerHTML` を使う場合も nonce 必須

**コード例**:
```tsx
// app/layout.tsx
import { headers } from 'next/headers';
import Script from 'next/script';

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const nonce = (await headers()).get('x-nonce') ?? '';

  return (
    <html lang="ja">
      <head>
        {/* 外部スクリプトに nonce */}
        <Script
          src="https://www.googletagmanager.com/gtag/js?id=G-XXXX"
          strategy="afterInteractive"
          nonce={nonce}
        />
        {/* インラインスクリプトにも nonce */}
        <Script id="ga-init" nonce={nonce}>
          {`
            window.dataLayer = window.dataLayer || [];
            function gtag(){dataLayer.push(arguments);}
            gtag('js', new Date());
            gtag('config', 'G-XXXX');
          `}
        </Script>
      </head>
      <body>{children}</body>
    </html>
  );
}
```

**注意点**:
- `headers()` を使うと自動的に dynamic rendering になる。静的最適化は失われる
- Server Component の static 化を保ちたい場合は CSP を nonce ではなく hash ベースに切り替えるか、外部スクリプトを `'strict-dynamic'` で扱う
- Pages Router では `_document.tsx` の `getInitialProps` で nonce を取得する

**出典**:
- [Next.js Docs: Content Security Policy](https://nextjs.org/docs/app/building-your-application/configuring/content-security-policy) (Next.js 公式)
- [Next.js Docs: headers()](https://nextjs.org/docs/app/api-reference/functions/headers) (Next.js 公式)

**バージョン**: Next.js 13+ (App Router)
**確信度**: 高
**最終更新**: 2026-05-16

---

### 3. CSP 違反を `report-to` / `report-uri` で収集する

CSP の `report-to` ディレクティブで違反レポートを収集し、本番環境で予期せぬブロックを観測する。
新規 CSP ルールは収集データを見てから締める。

**根拠**:
- CSP は厳しすぎると正規のスクリプトもブロックして UI が壊れる
- 違反レポートは攻撃の兆候だけでなく、自社の許可漏れも検出する（CDN の URL 変更等）
- `report-to` は CSP Level 3 の標準、`report-uri` は後方互換のため両方併用が推奨
- Sentry / Datadog / Reporting API 受信エンドポイントを自前実装で受ける

**コード例（CSP ヘッダー）**:
```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-{nonce}' 'strict-dynamic';
  report-uri /api/csp-report;
  report-to csp-endpoint;

Reporting-Endpoints: csp-endpoint="/api/csp-report"
```

**Next.js での実装**:
```ts
// app/api/csp-report/route.ts
import { type NextRequest } from 'next/server';
import { logger } from '@/lib/logger';

export async function POST(req: NextRequest) {
  // Content-Type は application/csp-report または application/reports+json
  const body = await req.json();

  // CSP Level 2 形式
  const report = body['csp-report'] ?? body;

  logger.warn('CSP violation', {
    documentUri: report['document-uri'] ?? report.documentURL,
    violatedDirective: report['violated-directive'] ?? report.effectiveDirective,
    blockedUri: report['blocked-uri'] ?? report.blockedURL,
    sourceFile: report['source-file'],
    lineNumber: report['line-number'] ?? report.lineNumber,
    userAgent: req.headers.get('user-agent'),
  });

  return new Response(null, { status: 204 });
}
```

**観測の運用**:
- `documentUri` でページ単位、`violatedDirective` でディレクティブ単位に集計
- `blockedUri: 'inline'` が多い → コードに残った `'unsafe-inline'` 依存
- ブラウザ拡張機能由来（`chrome-extension://` 等）は除外フィルタを入れる
- 急増アラートを設定（XSS 攻撃の兆候）

**出典**:
- [MDN: CSP report-to](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/report-to) (MDN Web Docs)
- [MDN: Reporting API](https://developer.mozilla.org/en-US/docs/Web/API/Reporting_API) (MDN Web Docs)
- [web.dev: Reporting API](https://web.dev/articles/reporting-api) (web.dev)

**バージョン**: CSP Level 3, Reporting API Level 1
**確信度**: 高
**最終更新**: 2026-05-16

---

### 4. CSP は段階導入する（Report-Only → Enforce）

新規 CSP は `Content-Security-Policy-Report-Only` ヘッダーから始め、違反データを蓄積してから enforce に切り替える。
既存サイトでいきなり enforce すると広範囲が壊れる。

**根拠**:
- `Report-Only` は違反をレポートするだけで実際にはブロックしない
- 2-4 週間のレポート収集で「期待されない違反」と「正規ソースの allow 漏れ」を区別できる
- 段階導入は OWASP / Google の CSP 導入ガイドで推奨される
- enforce 後も `Report-Only` を別ヘッダーで併用して、より厳しいポリシーを試験できる

**段階導入プロセス**:
1. **Phase 1（1-2 週間）**: `Report-Only` で広めの暫定ポリシー、レポート収集
2. **Phase 2**: レポートに基づいてポリシーを調整、allowlist を確定
3. **Phase 3（1-2 週間）**: 調整したポリシーで再度 `Report-Only` 検証
4. **Phase 4**: enforce 切り替え（`Content-Security-Policy` ヘッダーに変更）
5. **Phase 5**: より厳しいポリシーを `Report-Only` で追加検証（永続的に）

**コード例**:
```ts
// middleware.ts — Phase 1-3: Report-Only
const cspHeader = `
  default-src 'self';
  script-src 'self' 'nonce-${nonce}' 'strict-dynamic';
  style-src 'self' 'nonce-${nonce}';
  report-uri /api/csp-report;
`.replace(/\s{2,}/g, ' ').trim();

response.headers.set('Content-Security-Policy-Report-Only', cspHeader);

// Phase 4: enforce へ切り替え（ヘッダー名を変更）
response.headers.set('Content-Security-Policy', cspHeader);

// Phase 5: enforce + さらに厳しい Report-Only を併存
response.headers.set('Content-Security-Policy', currentCsp);
response.headers.set('Content-Security-Policy-Report-Only', tighterCsp);
```

**よくある失敗**:
- いきなり enforce → 広告 / Analytics / フィードバックフォーム等が壊れる
- レポート収集だけして調整しない → 永遠に `Report-Only` のままになる
- production だけで試験 → staging で再現できない事象を見逃す

**出典**:
- [MDN: Content-Security-Policy-Report-Only](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy-Report-Only) (MDN Web Docs)
- [OWASP: Content Security Policy Cheat Sheet - Refactoring inline code](https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html#refactoring-inline-code) (OWASP)

**バージョン**: CSP Level 2+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 5. `'strict-dynamic'` で第三者スクリプト allow list を最小化する

`'strict-dynamic'` を nonce / hash と組み合わせて使うと、nonce 付きで読み込まれた最初のスクリプトが動的に読み込む追加スクリプトも自動的に許可される。
URL allowlist を維持し続ける運用負荷を解消できる。

**根拠**:
- 大規模サイトでは Google Tag Manager・Segment・Sentry 等がさらに第三者スクリプトを動的に読み込む。allowlist で全部追跡するのは現実的に不可能
- `'strict-dynamic'` は nonce / hash の信頼を「動的に挿入されたスクリプト」に伝播させる
- 結果として `script-src` を `'self' 'nonce-xxx' 'strict-dynamic'` の 3 つだけで管理できる
- CSP Level 3 で標準化、Chrome / Firefox / Safari でサポート

**コード例**:
```
# Good (CSP Level 3 推奨形)
Content-Security-Policy:
  script-src 'self' 'nonce-{nonce}' 'strict-dynamic' https: 'unsafe-inline';
  object-src 'none';
  base-uri 'self';

# 解説:
# - 'nonce-xxx' : サーバー生成のスクリプトを許可
# - 'strict-dynamic' : nonce 付きスクリプトが動的に読み込むものも許可
# - https: : Level 3 をサポートしないブラウザのフォールバック（無視される）
# - 'unsafe-inline' : 同上のフォールバック（無視される）
```

**ブラウザ動作**:
- CSP Level 3 対応ブラウザ: `'strict-dynamic'` が有効になり、`https:` と `'unsafe-inline'` は **無視される**
- 古いブラウザ: `'strict-dynamic'` を無視し、`https:` + `'unsafe-inline'` でフォールバック動作
- 結果として全ブラウザで「動作はする / 新しいブラウザでは強固な保護」が両立する

**注意点**:
- ホスト allowlist（`https://cdn.example.com`）は Level 3 では無視されるため、運用としては「nonce を付けて読み込む」が前提
- 動的に DOM 挿入されるスクリプトは `document.createElement('script')` 経由なら nonce 不要（伝播）。`document.write` は不可
- `eval` は別途 `'unsafe-eval'` が必要（できれば避ける）

**出典**:
- [W3C CSP Level 3: strict-dynamic](https://www.w3.org/TR/CSP3/#strict-dynamic-usage) (W3C)
- [web.dev: Strict CSP](https://web.dev/articles/strict-csp) (web.dev)
- [csp-evaluator](https://csp-evaluator.withgoogle.com/) (Google)

**バージョン**: CSP Level 3 / Chrome 52+, Firefox 52+, Safari 15.4+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 6. 静的セキュリティヘッダと動的 CSP/nonce は配信レイヤを分離する

リクエスト毎に値が変わらないセキュリティヘッダ（`Strict-Transport-Security`, `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy` 等）は `next.config.ts` の `async headers()` で静的配信し、リクエスト毎に nonce が変わる `Content-Security-Policy` のみを `proxy.ts`（旧 middleware）で生成・付与する。
両者で CSP を二重に書くと、片方の古い nonce が新しい nonce を上書きしてポリシーが壊れるため、**CSP は proxy.ts のみで配信**する。

**根拠**:
- HSTS や `X-Frame-Options` のような静的ヘッダは `next.config.ts` で宣言した方が記述が一元化され、middleware 実行コストもかからない（全リクエストで middleware を通す必要が薄れる）
- 一方 CSP の nonce はリクエスト毎にランダム生成する必要があり、静的 `headers()` では実現できない（リクエストオブジェクトに触れない）
- 両所で CSP を書くと「`next.config.ts` の固定 CSP が proxy で生成した nonce 付き CSP を上書き／逆」のような順序事故が起きる。責務を分けると衝突しない
- Next.js 16 では middleware が `proxy.ts` にリネームされ Node.js runtime デフォルトに変わったため、Web Crypto API ベースの nonce 生成も自然に書ける

**コード例（next.config.ts: 静的ヘッダ）**:
```ts
// next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  async headers() {
    return [
      {
        source: "/(.*)",
        headers: [
          { key: "Strict-Transport-Security", value: "max-age=63072000; includeSubDomains; preload" },
          { key: "X-Frame-Options", value: "DENY" },
          { key: "X-Content-Type-Options", value: "nosniff" },
          { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
          { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=()" },
          // 注意: Content-Security-Policy はここに書かない（proxy.ts で動的に付与）
        ],
      },
    ];
  },
};

export default nextConfig;
```

**コード例（proxy.ts: 動的 CSP + nonce）**:
```ts
// proxy.ts
import { NextResponse, type NextRequest } from "next/server";

export const runtime = "nodejs"; // v16 デフォルト

export function proxy(request: NextRequest) {
  // リクエスト毎に新しい nonce を生成
  const nonce = Buffer.from(crypto.randomUUID()).toString("base64");

  const csp = `
    default-src 'self';
    script-src 'self' 'nonce-${nonce}' 'strict-dynamic';
    style-src 'self' 'nonce-${nonce}';
    img-src 'self' blob: data:;
    object-src 'none';
    base-uri 'self';
    frame-ancestors 'none';
    upgrade-insecure-requests;
  `.replace(/\s{2,}/g, " ").trim();

  // request headers に nonce を載せて downstream RSC で headers() 経由で読めるようにする
  const requestHeaders = new Headers(request.headers);
  requestHeaders.set("x-nonce", nonce);

  const response = NextResponse.next({ request: { headers: requestHeaders } });
  // CSP は proxy.ts のみで設定（next.config.ts には書かない）
  response.headers.set("Content-Security-Policy", csp);
  return response;
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico).*)"],
};
```

**よくある失敗**:
- `next.config.ts` の `headers()` に `Content-Security-Policy` の固定値（nonce 無し）を書いてしまい、proxy.ts で生成した nonce 付き CSP と衝突する
- 逆に静的ヘッダまで proxy.ts で全部付けようとして middleware 実行コストが上がり、また静的アセットの matcher 除外漏れで余計な処理が走る
- proxy.ts を Edge runtime のまま `Buffer` を使い、v16 への移行時にハマる（v16 では Node.js runtime がデフォルト）

**出典**:
- [Next.jsの考え方 / セキュリティヘッダ・CSP](https://zenn.dev/akfm/books/nextjs-basic-principle)

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**バージョン**: Next.js 16+
**確信度**: 高（v16 公式相当の知見）
**最終更新**: 2026-05-16
