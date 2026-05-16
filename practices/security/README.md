# Security Practices

フロントエンドセキュリティに関するベストプラクティス。
CSP・XSS・認証トークン管理・シークレット管理・依存セキュリティを扱う。

## ファイル構成

- `csp.md` — Content Security Policy（CSP）の設計と段階導入
- `xss.md` — クロスサイトスクリプティング対策（DOMPurify / URL スキーム / JSON 埋め込み）
- `auth-token-storage.md` — トークン保存（HttpOnly Cookie / Refresh Rotation / CSRF）
- `secret-management.md` — 環境変数・Secret Manager・バンドル漏洩防止
- `dependency-security.md` — npm audit / Renovate / lockfile / SBOM

## スコープ

フロントエンド単独で実装・設定できる対策に絞る。WAF / IDS / インフラレイヤの対策は対象外。
バックエンドとの境界（認証フロー・CORS）は両側に関わるため重複して記載することがある。
