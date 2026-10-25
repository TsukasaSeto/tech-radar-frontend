# Content Security Policy のベストプラクティス

XSS の最後の防衛線として CSP を運用する。「設定して終わり」ではなく、段階導入・違反監視・ポリシー更新のサイクルで運用する。

## ルール

### 1. CSP は nonce ベースで設計し、`'unsafe-inline'` を避ける

`script-src` には `'nonce-xxx'` を使い、サーバーが生成したワンタイム nonce を一致するインラインスクリプトのみ許可する。
`'unsafe-inline'` は CSP を実質無効化するため使わない（#5 の、`'strict-dynamic'` 非対応ブラウザ向けフォールバックとして nonce と併記する場合だけは例外。対応ブラウザでは無視される）。

**根拠**:
- `'unsafe-inline'` を許可すると XSS で注入されたスクリプトも実行可能になり、CSP の意味がなくなる
- nonce は HTTP レスポンスごとにランダム生成され、攻撃者は予測できない
- Google・MDN は `'strict-dynamic'` + nonce を CSP Level 3 のベストプラクティスとして推奨
- `'unsafe-eval'` も同様に避ける（Vue.js テンプレート等で必要なら明示的に許可）
- `prefetch-src` は実験的ディレクティブとして提案されたが CSP Level 3 仕様から削除されており、最新ブラウザでは無視される。防御手段として `prefetch-src` に依存せず、`script-src` / `default-src` 等の標準 fetch ディレクティブでリソース取得を制御する

**コード例（Next.js Proxy）**:
```ts
// proxy.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function proxy(request: NextRequest) {
  const nonce = Buffer.from(crypto.randomUUID()).toString('base64');
  // 開発時は React がデバッグ情報の復元に eval を使うため 'unsafe-eval' が要る（本番では不要）
  const isDev = process.env.NODE_ENV === 'development';

  const cspHeader = `
    default-src 'self';
    script-src 'self' 'nonce-${nonce}' 'strict-dynamic' https:${isDev ? " 'unsafe-eval'" : ''};
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
- [OWASP Content Security Policy Cheat Sheet: prefetch-src の記述修正](https://github.com/OWASP/CheatSheetSeries/commit/1eacf6cb9bfcba006ca972a804c5faace8c2758a) (OWASP CheatSheetSeries、`prefetch-src` が CSP Level 3 仕様から削除され最新ブラウザでは無視される旨の公式ドキュメント修正) ※2026-09-05に実際にfetch成功
- [Next.js Docs: Content Security Policy](https://nextjs.org/docs/app/guides/content-security-policy) (Next.js 公式、proxy.ts の nonce 例と開発時の `'unsafe-eval'`) ※2026-10-25 確認

> "It was removed from the CSP Level 3 specification and is ignored by modern browsers — do not rely on it for defense."
> ([OWASP Content Security Policy Cheat Sheet commit](https://github.com/OWASP/CheatSheetSeries/commit/1eacf6cb9bfcba006ca972a804c5faace8c2758a), セクション "prefetch-src") ※2026-09-05に実際にfetch成功

**バージョン**: CSP Level 3 / Next.js 16+（v15 以前は `middleware.ts` / `export function middleware` で同じ内容を書く）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 2. Next.js の Proxy（v15 以前は Middleware）で nonce を生成し、Script / Style に注入する

Next.js では Proxy（`proxy.ts`）で nonce を生成し、`headers()` 経由で Server Components から取得して `<Script nonce={nonce}>` に渡す。Proxy 側の書き方（リクエストとレスポンスの両方に CSP を設定する）は #6 を参照。
クライアント側で nonce を生成しない（攻撃者が読み取れる）。

**根拠**:
- Proxy はリクエスト単位で実行されるため、リクエストごとに新しい nonce を生成できる
- Server Component は `headers()` API で Proxy が設定した値を読み取れる
- Next.js の `<Script>` コンポーネントは `nonce` prop をサポートし、自動で CSP 準拠になる
- インライン `<script>` 内で `dangerouslySetInnerHTML` を使う場合も nonce 必須
- nonce は仕様上「リクエストごとに一意」でなければ意味を持たない。静的プリレンダリング等でビルド時に固定された nonce を焼き込んでしまうと、値としては動いていても CSP のセキュリティ効果は実質ゼロになる

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
- static 化を保ちたい場合は、nonce ではなく hash ベースの CSP に切り替える（Next.js では実験的な Subresource Integrity: `experimental.sri` が、ビルド時に JS ファイルの hash を `integrity` 属性として付与する。App Router のみ）。`'strict-dynamic'` は nonce か hash で信頼の起点を作らないと成立しないため、単独では static 化の代替にならない（#5）
- Pages Router では `_document.tsx` の `getInitialProps` で nonce を取得する

**出典**:
- [Next.js Docs: Content Security Policy](https://nextjs.org/docs/app/guides/content-security-policy) (Next.js 公式、nonce には動的レンダリングが必須であることと、代替の `experimental.sri`) ※2026-10-25 確認
- [Next.js Docs: headers()](https://nextjs.org/docs/app/api-reference/functions/headers) (Next.js 公式)
- [script-srcのunsafe-inlineを消すために、Next.js 16のmiddlewareでリクエスト毎のnonceを配った話](https://zenn.dev/sen_web3/articles/vouch-csp-nonce-middleware) (Zenn、静的プリレンダリングでのnonce焼き込み問題を実例で確認) ※2026-07-25に実際にfetch成功

**出典引用**:
> "nonceは仕様上「リクエストごとに一意」でなければ意味がないため、ビルド時に固定された値は本質的にCSPのセキュリティ効果を持たなくなります。"
> ([script-srcのunsafe-inlineを消すために、Next.js 16のmiddlewareでリクエスト毎のnonceを配った話](https://zenn.dev/sen_web3/articles/vouch-csp-nonce-middleware), セクション "nonceをリクエスト毎に生成する") ※2026-07-25に実際にfetch成功

**バージョン**: Next.js 13+ (App Router)、Next.js 16 で追加確認（v16 は `proxy.ts`、v15 以前は `middleware.ts`）
**確信度**: 高
**最終更新**: 2026-10-25

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

type JsonObject = Record<string, unknown>;

type Violation = {
  documentUrl: unknown;
  effectiveDirective: unknown;
  blockedUrl: unknown;
  sourceFile: unknown;
  lineNumber: unknown;
};

const isObject = (value: unknown): value is JsonObject =>
  typeof value === 'object' && value !== null;

// report-to（application/reports+json）: レポートの配列で、違反の詳細は各要素の body にある
function parseReportingApi(payload: unknown): Violation[] {
  if (!Array.isArray(payload)) return [];
  return payload.flatMap((report) => {
    if (!isObject(report) || report.type !== 'csp-violation') return [];
    const body = report.body;
    if (!isObject(body)) return [];
    return [{
      documentUrl: body.documentURL,
      effectiveDirective: body.effectiveDirective,
      blockedUrl: body.blockedURL,
      sourceFile: body.sourceFile,
      lineNumber: body.lineNumber,
    }];
  });
}

// report-uri（application/csp-report）: { "csp-report": { ... } } で、キーはハイフン区切り
function parseLegacyReport(payload: unknown): Violation[] {
  if (!isObject(payload)) return [];
  const report = payload['csp-report'];
  if (!isObject(report)) return [];
  return [{
    documentUrl: report['document-uri'],
    effectiveDirective: report['effective-directive'] ?? report['violated-directive'],
    blockedUrl: report['blocked-uri'],
    sourceFile: report['source-file'],
    lineNumber: report['line-number'],
  }];
}

export async function POST(req: NextRequest) {
  let payload: unknown;
  try {
    payload = await req.json();
  } catch {
    return new Response(null, { status: 400 });
  }

  const contentType = req.headers.get('content-type') ?? '';
  const violations = contentType.startsWith('application/reports+json')
    ? parseReportingApi(payload)
    : parseLegacyReport(payload);

  for (const violation of violations) {
    logger.warn('CSP violation', {
      ...violation,
      userAgent: req.headers.get('user-agent'),
    });
  }

  return new Response(null, { status: 204 });
}
```

**観測の運用**:
- `documentUrl` でページ単位、`effectiveDirective` でディレクティブ単位に集計
- `blockedUrl: 'inline'` が多い → コードに残った `'unsafe-inline'` 依存
- ブラウザ拡張機能由来（`chrome-extension://` 等）は除外フィルタを入れる
- 急増アラートを設定（XSS 攻撃の兆候）

**出典**:
- [MDN: CSP report-to](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/report-to) (MDN Web Docs)
- [MDN: Reporting API](https://developer.mozilla.org/en-US/docs/Web/API/Reporting_API) (MDN Web Docs)
- [web.dev: Reporting API](https://web.dev/articles/reporting-api) (web.dev)
- [MDN: CSPViolationReportBody](https://developer.mozilla.org/en-US/docs/Web/API/CSPViolationReportBody) (MDN Web Docs、report-to で届くレポートの配列形式と body のフィールド) ※2026-10-25 確認
- [MDN: CSP report-uri](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/report-uri) (MDN Web Docs、`application/csp-report` の `csp-report` 形式。report-to 対応ブラウザでは report-uri は無視される) ※2026-10-25 確認

**バージョン**: CSP Level 3, Reporting API Level 1
**確信度**: 高
**最終更新**: 2026-10-25

---

### 4. CSP は段階導入する（Report-Only → Enforce）

新規 CSP は `Content-Security-Policy-Report-Only` ヘッダーから始め、違反データを蓄積してから enforce に切り替える。
既存サイトでいきなり enforce すると広範囲が壊れる。

**根拠**:
- `Report-Only` は違反をレポートするだけで実際にはブロックしない
- 2-4 週間のレポート収集で「期待されない違反」と「正規ソースの allow 漏れ」を区別できる
- 段階導入は OWASP / Google の CSP 導入ガイドで推奨される
- enforce 後も `Report-Only` を別ヘッダーで併用して、より厳しいポリシーを試験できる
- `NEXT_PUBLIC_` 付きの環境変数や、静的生成時に評価されるコードの `process.env` はビルド時に値が埋め込まれるため、そこで CSP の allowlist を `process.env` から動的に組み立てると、ビルド後に環境変数が変わっても反映されない。未設定時に `undefined` 文字列がそのまま allowlist に混入し（例: `connect-src 'self' undefined`）、スクリプトブロックのように派手に壊れず analytics のビーコンだけが黙って失敗する部分的な障害になりやすい。既知の API オリジンは環境ごとにハードコードした配列を fallback として持たせる

**段階導入プロセス**:
1. **Phase 1（1-2 週間）**: `Report-Only` で広めの暫定ポリシー、レポート収集
2. **Phase 2**: レポートに基づいてポリシーを調整、allowlist を確定
3. **Phase 3（1-2 週間）**: 調整したポリシーで再度 `Report-Only` 検証
4. **Phase 4**: enforce 切り替え（`Content-Security-Policy` ヘッダーに変更）
5. **Phase 5**: より厳しいポリシーを `Report-Only` で追加検証（永続的に）

**コード例**:
```ts
// proxy.ts — Phase 1-3: Report-Only
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

```javascript
// 既知の API オリジンをハードコードした fallback（ビルド時 env var 欠落対策）
const KNOWN_API_ORIGINS = [
  "https://api.example.com",
  "https://api-staging.example.com",
];
const BUILD_API_ORIGIN = (process.env.NEXT_PUBLIC_API_URL ?? "").trim();
const API_ORIGINS = Array.from(
  new Set([...KNOWN_API_ORIGINS, BUILD_API_ORIGIN].filter(Boolean)),
);
// → env var が空でも "undefined" が connect-src に混入しない
```

**よくある失敗**:
- いきなり enforce → 広告 / Analytics / フィードバックフォーム等が壊れる
- レポート収集だけして調整しない → 永遠に `Report-Only` のままになる
- production だけで試験 → staging で再現できない事象を見逃す
- ビルド時に埋め込まれた `process.env` の allowlist が実行時の環境変数変更に追従しない → staging でだけ `connect-src 'self' undefined` のような欠落が起き、analytics のビーコンだけが静かに落ちる
- allowlist にコメントを残さない → 数ヶ月後に「なぜこのドメインが許可されているか」が分からなくなり、誰かが「整理」した拍子に analytics 等を壊す

**出典**:
- [MDN: Content-Security-Policy-Report-Only](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy-Report-Only) (MDN Web Docs)
- [OWASP: Content Security Policy Cheat Sheet - Refactoring inline code](https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html#refactoring-inline-code) (OWASP)
- [I Shipped an Enforcing Content-Security-Policy on a Real Next.js](https://medium.com/@virendrapatil2902/i-shipped-an-enforcing-content-security-policy-on-a-real-next-js-4e12f83f2c28) (Medium、ビルド時 env var 埋め込みによるサイレント障害の実例) ※2026-07-24に実際にfetch成功

**出典引用**:
> "CSP is baked into the bundle. A missing env var doesn't throw; it silently produces `connect-src 'self' undefined`"
> ([I Shipped an Enforcing Content-Security-Policy on a Real Next.js](https://medium.com/@virendrapatil2902/i-shipped-an-enforcing-content-security-policy-on-a-real-next-js-4e12f83f2c28), セクション "Failure #3: the API works locally and dies in staging") ※2026-07-24に実際にfetch成功

**バージョン**: CSP Level 2+
**確信度**: 高
**最終更新**: 2026-10-25

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

リクエスト毎に値が変わらないセキュリティヘッダ（`X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy` 等）は `next.config.ts` の `async headers()` で静的配信し、リクエスト毎に nonce が変わる `Content-Security-Policy` のみを `proxy.ts`（v15 以前は `middleware.ts`）で生成・付与する。proxy.ts では CSP をリクエストヘッダとレスポンスヘッダの両方に設定する。
両者で CSP を二重に書くと、片方の古い nonce が新しい nonce を上書きしてポリシーが壊れるため、**CSP は proxy.ts のみで配信**する。

**根拠**:
- 静的ヘッダは `next.config.ts` で宣言した方が記述が一元化され、proxy の実行コストもかからない。値の選び方と実レスポンスでの検証は `practices/nextjs/security-headers.md` #1 を正本とする
- 一方 CSP の nonce はリクエスト毎にランダム生成する必要があり、静的 `headers()` では実現できない（リクエストオブジェクトに触れない）
- 両所で CSP を書くと「`next.config.ts` の固定 CSP が proxy で生成した nonce 付き CSP を上書き／逆」のような順序事故が起きる。責務を分けると衝突しない
- Next.js はレンダリング時にリクエストの `Content-Security-Policy` ヘッダを解析して nonce を取り出し、フレームワークのスクリプト・ページの JS バンドル・`nonce` prop 付きの `<Script>` に自動で付与する。リクエスト側に CSP を設定せず `x-nonce` だけを渡すと nonce が付かず、`'strict-dynamic'` 下では `'self'` も無視されるため全スクリプトがブロックされる
- nonce を使うページは動的レンダリングが必須。ビルド時に生成される静的ページには nonce を注入できず、ISR や CDN キャッシュも使えない。Partial Prerendering（PPR）の静的シェルとも併用できない。静的化を保ちたい場合は #2 の hash ベース（`experimental.sri`）を検討する
- Next.js 16 の proxy.ts は Node.js ランタイムで動く（`runtime` の指定はエラーになる）ため、`Buffer` を使った nonce 生成をそのまま書ける

**静的ヘッダ（next.config.ts）**: 設定例と値は `practices/nextjs/security-headers.md` #1 を参照。`Content-Security-Policy` はここに書かない。

**コード例（proxy.ts: 動的 CSP + nonce）**:
```ts
// proxy.ts
import { NextResponse, type NextRequest } from "next/server";

export function proxy(request: NextRequest) {
  const nonce = Buffer.from(crypto.randomUUID()).toString("base64");
  const isDev = process.env.NODE_ENV === "development";

  const csp = `
    default-src 'self';
    script-src 'self' 'nonce-${nonce}' 'strict-dynamic'${isDev ? " 'unsafe-eval'" : ""};
    style-src 'self' 'nonce-${nonce}';
    img-src 'self' blob: data:;
    font-src 'self';
    object-src 'none';
    base-uri 'self';
    form-action 'self';
    frame-ancestors 'none';
    upgrade-insecure-requests;
  `.replace(/\s{2,}/g, " ").trim();

  const requestHeaders = new Headers(request.headers);
  requestHeaders.set("x-nonce", nonce);
  // Next.js はリクエスト側の CSP から nonce を取り出してフレームワークのスクリプトに付与する
  requestHeaders.set("Content-Security-Policy", csp);

  const response = NextResponse.next({ request: { headers: requestHeaders } });
  response.headers.set("Content-Security-Policy", csp);
  return response;
}

export const config = {
  matcher: [
    {
      source: "/((?!api|_next/static|_next/image|favicon.ico).*)",
      missing: [
        { type: "header", key: "next-router-prefetch" },
        { type: "header", key: "purpose", value: "prefetch" },
      ],
    },
  ],
};
```

```tsx
// app/page.tsx — 静的化されうるページは明示的に動的レンダリングへ切り替える
import { connection } from "next/server";

export default async function Page() {
  await connection();
  return <main>...</main>;
}
```

**よくある失敗**:
- `next.config.ts` の `headers()` に `Content-Security-Policy` の固定値（nonce 無し）を書いてしまい、proxy.ts で生成した nonce 付き CSP と衝突する
- 逆に静的ヘッダまで proxy.ts で全部付けようとして proxy の実行コストが上がり、また静的アセットの matcher 除外漏れで余計な処理が走る
- レスポンスにだけ CSP を付け、リクエスト側には `x-nonce` しか渡さない → フレームワークのスクリプトに nonce が付かずアプリが動かなくなり、回避のために `'unsafe-inline'` を足して CSP を弱める

**出典**:
- [Next.jsの考え方 / セキュリティヘッダ・CSP](https://zenn.dev/akfm/books/nextjs-basic-principle)
- [Next.js Docs: Content Security Policy](https://nextjs.org/docs/app/guides/content-security-policy) (Next.js 公式、リクエストとレスポンスの両方への CSP 設定、"How nonces work in Next.js"、動的レンダリング必須と PPR 非互換) ※2026-10-25 確認
- [Next.js Docs: proxy.js](https://nextjs.org/docs/app/api-reference/file-conventions/proxy) (Next.js 公式、"Setting the `runtime` config option in Proxy will throw an error.") ※2026-10-25 確認

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**バージョン**: Next.js 16+（v15 以前は `middleware.ts` / `export function middleware`）
**確信度**: 高
**最終更新**: 2026-10-25
