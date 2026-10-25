# Next.js セキュリティヘッダーのベストプラクティス

`next.config.ts` の `headers()` API を使った静的セキュリティヘッダーの設定と、実レスポンスでの検証方法。静的セキュリティヘッダーの値はこのファイルの #1 を正本とする。リクエストごとに nonce が変わる CSP はこのファイルの対象外（`proxy.ts` で付与する。`../security/csp.md` #6 を参照）。

## ルール

### 1. `next.config.ts` の `headers()` で全ルートに静的セキュリティヘッダーを設定し、実レスポンスで検証する

Next.js では `next.config.ts` の `async headers()` に `source: "/:path*"` を指定することで、CSP を除く静的セキュリティヘッダー（X-Content-Type-Options / X-Frame-Options / Referrer-Policy / Permissions-Policy / Strict-Transport-Security）を全ルート（404 含む）に一括適用できる。設定を書いただけで終わらせず、`curl -sSI` で 200 / 404 それぞれの実レスポンスにヘッダーが付与されているかを必ず確認する。

**根拠**:
- `next.config.ts` の `headers` 配列 + `async headers()` の `source: "/:path*"` で全ルートに適用する具体構成が公式 API として提供されている
- 設定して終わりではなく「プロダクションビルドで確認する」「`curl` で 200 と 404 を確認する」という検証まで含めることで、設定ミスや意図しないルートの除外を防げる
- `X-XSS-Protection` は MDN で非推奨・非標準とされているため、意図的に対象から除外する
- CSP は nonce をリクエストごとに生成する必要があるので `headers()` では設定せず、`proxy.ts` で付与する（`../security/csp.md` #6）。CSP を両方に書くと nonce が食い違ってポリシーが壊れる
- `X-Frame-Options` は CSP の `frame-ancestors` に置き換えられつつあり、対応ブラウザでは `frame-ancestors` が優先される（公式）。値は CSP 側とそろえる。このリポジトリの CSP（`csp.md` #6）は `frame-ancestors 'none'` なので、`X-Frame-Options` は `DENY` にする。自サイト内で iframe 埋め込みが必要な場合だけ、`SAMEORIGIN` と `frame-ancestors 'self'` の組み合わせにする
- HSTS は値をコピーするだけでは危険なので、影響を確認してから入れる。`max-age=63072000; includeSubDomains`（2 年・サブドメイン含む）を基本値にする。`includeSubDomains` は HTTP でしか配信できないサブドメインを開けなくするので、全サブドメインが HTTPS で配信できることを先に確認する。`preload` はブラウザの preload list に登録する場合だけ付ける。登録は簡単には取り消せず、削除がユーザーに届くまで数か月かかる（hstspreload.org）
- `Referrer-Policy` はモダンブラウザの既定値（`strict-origin-when-cross-origin`）と同じ値でも、既定に頼らず明示的に設定する（OWASP HTTP Headers Cheat Sheet）
- 実装時のチェックリスト: 未使用ブラウザ機能の確認（`Permissions-Policy` の値）、CDN によるヘッダー上書きの確認、複数ステータスコード・本番URLでのテスト

**コード例**:
```typescript
// next.config.ts
import type { NextConfig } from "next";

const securityHeaders = [
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "X-Frame-Options", value: "DENY" }, // CSP の frame-ancestors 'none' とそろえる
  { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
  {
    key: "Permissions-Policy",
    value: "camera=(), microphone=(), geolocation=()",
  },
  // 全サブドメインを HTTPS で配信できることを確認してから入れる。preload list に登録する場合だけ "; preload" を足す
  { key: "Strict-Transport-Security", value: "max-age=63072000; includeSubDomains" },
  // Content-Security-Policy はここに書かない（proxy.ts で nonce 付きで付与する。security/csp.md #6）
];

const nextConfig: NextConfig = {
  async headers() {
    return [
      {
        source: "/:path*",
        headers: securityHeaders,
      },
    ];
  },
};

export default nextConfig;
```

```bash
# プロダクションビルドで実レスポンスを検証する（200 と 404 の両方）
curl -sSI https://example.com/            # 200 ルート
curl -sSI https://example.com/not-found   # 404 ルートでもヘッダーが付与されているか確認
```

**出典引用**:
> "ビルドが通り、想定した各レスポンスにヘッダーが付き、既存機能を壊していないところまで確認して完了です。"
> ([Next.jsの全ルートに4つのセキュリティヘッダーを設定し、実レスポンスで確認する](https://zenn.dev/nadarakainc/articles/666ae312dc336e), セクション "`curl`で200と404を確認する") ※2026-08-24に実際にfetch成功

> "MDNでは`X-XSS-Protection`を非推奨かつ非標準としており、使用を推奨していません。"
> ([Next.jsの全ルートに4つのセキュリティヘッダーを設定し、実レスポンスで確認する](https://zenn.dev/nadarakainc/articles/666ae312dc336e), セクション本文) ※2026-08-24に実際にfetch成功

> "CSPとHSTSも重要ですが、値をコピーするだけでは危険です。"
> ([Next.jsの全ルートに4つのセキュリティヘッダーを設定し、実レスポンスで確認する](https://zenn.dev/nadarakainc/articles/666ae312dc336e), セクション本文) ※2026-08-24に実際にfetch成功

> "This header has been superseded by CSP's `frame-ancestors` option, which has better support in modern browsers"
> ([Next.js Docs: headers](https://nextjs.org/docs/app/api-reference/config/next-config-js/headers), セクション "X-Frame-Options") ※2026-10-25 確認

> "Be aware that inclusion in the preload list cannot easily be undone."
> ([HSTS Preload List Submission](https://hstspreload.org/)) ※2026-10-25 確認

> "Modern browsers default to `strict-origin-when-cross-origin`: same-origin requests include the origin, path, and query string in the `Referer` header; cross-origin requests include only the origin, except HTTPS-to-HTTP requests, which omit the header. Set the header explicitly rather than relying on browser defaults."
> ([HTTP Headers Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/HTTP_Headers_Cheat_Sheet.md), セクション "Referrer-Policy" > "Recommendation") ※2026-10-03に実際にfetch成功（`security/xss.md` #5 から移した引用）

**出典**:
- [Next.jsの全ルートに4つのセキュリティヘッダーを設定し、実レスポンスで確認する](https://zenn.dev/nadarakainc/articles/666ae312dc336e) (Zenn nadarakainc、next.config.ts の headers() 実装と curl による実レスポンス検証) ※2026-08-24 fetch
- [Next.js Docs: headers](https://nextjs.org/docs/app/api-reference/config/next-config-js/headers) (Next.js 公式、`headers()` の設定方法と各セキュリティヘッダーの説明) ※2026-10-25 確認
- [HSTS Preload List Submission](https://hstspreload.org/) (preload list の登録要件と、取り消しに時間がかかる点・`preload` を既定では付けない推奨) ※2026-10-25 確認
- [OWASP HTTP Headers Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/HTTP_Headers_Cheat_Sheet.md) (OWASP CheatSheetSeries 公式、Referrer-Policy を明示設定する推奨) ※2026-10-03 fetch

**バージョン**: Next.js 16.2.1（React 19.2.3, Node.js 26.7.0 で検証）
**確信度**: 中（ヘッダーの一括設定と curl での検証は単一の非公式記事。`headers()` の API と各ヘッダーの説明は公式ドキュメントで確認）
**最終更新**: 2026-10-25

---
