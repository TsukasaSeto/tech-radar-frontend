# Next.js セキュリティヘッダーのベストプラクティス

`next.config.ts` の `headers()` API を使った静的セキュリティヘッダーの設定と、実レスポンスでの検証方法。CSP・HSTS のような個別設計が必要なヘッダーはこのファイルの対象外（`../security/csp.md` を参照）。

## ルール

### 1. `next.config.ts` の `headers()` で全ルートに静的セキュリティヘッダーを設定し、実レスポンスで検証する

Next.js では `next.config.ts` の `async headers()` に `source: "/:path*"` を指定することで、CSP を除く静的セキュリティヘッダー（X-Content-Type-Options / X-Frame-Options / Referrer-Policy / Permissions-Policy）を全ルート（404 含む）に一括適用できる。設定を書いただけで終わらせず、`curl -sSI` で 200 / 404 それぞれの実レスポンスにヘッダーが付与されているかを必ず確認する。

**根拠**:
- `next.config.ts` の `headers` 配列 + `async headers()` の `source: "/:path*"` で全ルートに適用する具体構成が公式 API として提供されている
- 設定して終わりではなく「プロダクションビルドで確認する」「`curl` で 200 と 404 を確認する」という検証まで含めることで、設定ミスや意図しないルートの除外を防げる
- `X-XSS-Protection` は MDN で非推奨・非標準とされているため、意図的に対象から除外する
- CSP と HSTS は「値をコピーするだけでは危険」であり個別の設計判断が必要なため、このヘッダー一括設定には含めない（CSP の設計は `../security/csp.md` を参照）
- 実装時のチェックリスト: 未使用ブラウザ機能の確認（`Permissions-Policy` の値）、CDN によるヘッダー上書きの確認、複数ステータスコード・本番URLでのテスト

**コード例**:
```typescript
const securityHeaders = [
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "X-Frame-Options", value: "SAMEORIGIN" },
  { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
  {
    key: "Permissions-Policy",
    value: "camera=(), microphone=(), geolocation=()",
  },
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

**出典**:
- [Next.jsの全ルートに4つのセキュリティヘッダーを設定し、実レスポンスで確認する](https://zenn.dev/nadarakainc/articles/666ae312dc336e) (Zenn nadarakainc、next.config.ts の headers() 実装と curl による実レスポンス検証) ※2026-08-24 fetch

**バージョン**: Next.js 16.2.1（React 19.2.3, Node.js 26.7.0 で検証）
**確信度**: 中（単一の非公式記事だが、公式 API の具体的な設定コードと実レスポンス検証を伴うためパターン1c扱い）
**最終更新**: 2026-08-24

---
