# API Client Practices

REST / GraphQL / gRPC など、フロントエンドからのAPI通信に関するベストプラクティス。
クライアントライブラリの選定、型安全性、エラー処理、キャッシュ戦略を扱う。

## トピック

- `rest.md` - REST API クライアント（fetch, ky, axios, openapi-fetch等）
- `graphql.md` - GraphQL クライアント（Apollo, urql, GraphQL Code Generator等）
- `grpc.md` - gRPC / gRPC-Web / Connect クライアント
- `error-handling.md` - APIエラーの分類と処理パターン
- `caching.md` - APIレスポンスキャッシュ（TanStack Query, SWR, RTK Query等）
- `type-safety.md` - スキーマ駆動開発、型生成、ランタイム検証

## 関連カテゴリ

- `observability/error-tracking.md` - APIエラーの監視
- `observability/tracing.md` - 分散トレーシング
- `architecture/error-handling.md` - アプリ全体のエラー処理戦略
