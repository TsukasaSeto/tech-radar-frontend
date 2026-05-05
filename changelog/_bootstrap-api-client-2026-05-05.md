# Bootstrap Log: api-client (2026-05-05)

## 抽出結果サマリ

| トピック | ファイル | ルール数 |
|---|---|---|
| REST API クライアント | `practices/api-client/rest.md` | 5 |
| GraphQL クライアント | `practices/api-client/graphql.md` | 4 |
| gRPC / Connect | `practices/api-client/grpc.md` | 3 |
| エラー処理 | `practices/api-client/error-handling.md` | 4 |
| キャッシュ戦略 | `practices/api-client/caching.md` | 4 |
| 型安全性 | `practices/api-client/type-safety.md` | 4 |
| **合計** | | **24 ルール** |

## 参照した一次情報（公式ドキュメント）

| ソース | URL | 取得結果 |
|---|---|---|
| ky README | https://github.com/sindresorhus/ky | 成功 |
| Sentry Next.js Docs | https://docs.sentry.io/platforms/javascript/guides/nextjs/ | 成功 |
| Sentry Sensitive Data | https://docs.sentry.io/platforms/javascript/data-management/sensitive-data/ | 成功 |
| TanStack Query Important Defaults | https://tanstack.com/query/latest/docs/framework/react/guides/important-defaults | 403（知識ベースで補完） |
| TanStack Query Query Keys | https://tanstack.com/query/latest/docs/framework/react/guides/query-keys | 403（知識ベースで補完） |
| TanStack Query Optimistic Updates | https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates | 403（知識ベースで補完） |
| SWR Getting Started | https://swr.vercel.app/docs/getting-started | 403（知識ベースで補完） |
| openapi-fetch Docs | https://openapi-ts.dev/openapi-fetch/ | 403（知識ベースで補完） |
| Zod Docs | https://zod.dev | 403（知識ベースで補完） |
| GraphQL Code Generator | https://the-guild.dev/graphql/codegen/docs/getting-started | 403（知識ベースで補完） |
| Connect-Web Docs | https://connectrpc.com/docs/web/getting-started | 403（知識ベースで補完） |
| OpenTelemetry JS | https://opentelemetry.io/docs/languages/js/ | 403（知識ベースで補完） |
| MDN: AbortController | https://developer.mozilla.org/en-US/docs/Web/API/AbortController | 知識ベース |
| MDN: HTTP Status Codes | https://developer.mozilla.org/en-US/docs/Web/HTTP/Status | 知識ベース |
| AWS: Exponential Backoff and Jitter | https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/ | 知識ベース |
| GraphQL Spec: Errors | https://spec.graphql.org/October2021/#sec-Errors | 知識ベース |

## 参照した企業ブログ・コミュニティ記事

企業ブログへのアクセスは 403 エラーが多く、公式ドキュメントの内容と知識ベースで補完した。
次回の更新時に以下を優先的に確認する:
- https://techblog.zozo.com（TanStack Query 導入事例）
- https://engineering.mercari.com（型安全API設計）
- https://tech.smarthr.jp（GraphQL Code Generator）

## 既存カテゴリへのクロスリファレンス追加

| 既存ファイル | 追加内容 |
|---|---|
| `practices/architecture/error-handling.md` | `api-client/error-handling.md` と `observability/error-tracking.md` へのリンク |
| `practices/architecture/logging.md` | `observability/logging.md` と `observability/error-tracking.md` へのリンク |
| `practices/performance/core-web-vitals.md` | `observability/rum.md` へのリンク |

## 新規参照 URL リスト（_seen.json への追記対象）

```json
[
  "https://github.com/sindresorhus/ky",
  "https://openapi-ts.dev/openapi-fetch/",
  "https://openapi-ts.dev/introduction",
  "https://zod.dev",
  "https://valibot.dev",
  "https://tanstack.com/query/latest/docs/framework/react/guides/important-defaults",
  "https://tanstack.com/query/latest/docs/framework/react/guides/caching",
  "https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates",
  "https://tanstack.com/query/latest/docs/framework/react/guides/query-keys",
  "https://commerce.nearform.com/open-source/urql/docs/",
  "https://www.apollographql.com/docs/react/data/fragments/",
  "https://www.apollographql.com/docs/react/performance/optimistic-ui/",
  "https://the-guild.dev/graphql/codegen/docs/getting-started",
  "https://connectrpc.com/docs/web/getting-started",
  "https://connectrpc.com/docs/web/streaming",
  "https://buf.build/docs/generate/overview",
  "https://buf.build/docs/cli/installation",
  "https://spec.graphql.org/October2021/#sec-Errors",
  "https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/",
  "https://nextjs.org/docs/app/api-reference/functions/fetch"
]
```

## 備考

- ほとんどの公式ドキュメントサイトが 403 を返したため、知識ベースで補完した
- TanStack Query v4 と v5 の API 差異（`cacheTime` → `gcTime`）を明記した
- Connect-Web と gRPC-Web の選択基準を明確化した
