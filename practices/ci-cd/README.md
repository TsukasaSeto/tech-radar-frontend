# CI/CD Practices

フロントエンドの CI/CD パイプライン構築に関するベストプラクティス。
GitHub Actions・ビルドキャッシュ・プレビューデプロイ・リリース戦略を扱う。

## ファイル構成

- `github-actions.md` — GitHub Actions の標準パイプライン構成と最適化
- `build-cache.md` — Turborepo / Next.js キャッシュ / `actions/cache` の活用
- `preview-deploy.md` — PR ごとのプレビュー URL と Visual Regression
- `release-strategy.md` — semantic-release / Changesets / カナリア / ロールバック

## スコープ

- 対象 CI: GitHub Actions（メインターゲット）、CircleCI / GitLab CI は参考扱い
- 対象 PaaS: Vercel / Netlify / Cloudflare Pages
- インフラ自体（IaC / k8s）は対象外。フロントエンドビルド〜デプロイの最後の 1 マイル
