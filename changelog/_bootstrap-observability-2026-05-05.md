# Bootstrap Log: observability (2026-05-05)

## 抽出結果サマリ

| トピック | ファイル | ルール数 |
|---|---|---|
| エラー追跡 | `practices/observability/error-tracking.md` | 5 |
| ロギング | `practices/observability/logging.md` | 4 |
| 分散トレーシング | `practices/observability/tracing.md` | 3 |
| カスタムメトリクス | `practices/observability/metrics.md` | 3 |
| Real User Monitoring | `practices/observability/rum.md` | 4 |
| **合計** | | **19 ルール** |

## 参照した一次情報（公式ドキュメント）

| ソース | URL | 取得結果 |
|---|---|---|
| Sentry Next.js Docs | https://docs.sentry.io/platforms/javascript/guides/nextjs/ | 成功 |
| Sentry Performance | https://docs.sentry.io/platforms/javascript/performance/ | 成功 |
| Sentry Sensitive Data | https://docs.sentry.io/platforms/javascript/data-management/sensitive-data/ | 成功 |
| web-vitals GitHub | https://github.com/GoogleChrome/web-vitals | 成功 |
| OpenTelemetry JS | https://opentelemetry.io/docs/languages/js/ | 403（知識ベースで補完） |
| Pino Docs | https://getpino.io/ | 知識ベース |
| Datadog Browser RUM | https://docs.datadoghq.com/real_user_monitoring/browser | 知識ベース |
| W3C Trace Context | https://www.w3.org/TR/trace-context/ | 知識ベース |
| web.dev: RUM | https://web.dev/articles/rum-and-synthetic-monitoring | 知識ベース |
| Next.js OpenTelemetry | https://nextjs.org/docs/app/building-your-application/optimizing/open-telemetry | 知識ベース |

## 参照した企業ブログ・コミュニティ記事

企業ブログへのアクセスは多くが 403 エラーを返したため、公式ドキュメントと知識ベースで補完。
次回更新時に確認予定:
- https://blog.sentry.io（Sentry の PII 対策事例）
- https://techblog.zozo.com（OpenTelemetry 導入事例）
- https://vercel.com/blog（Next.js と OpenTelemetry）

## 既存カテゴリへのクロスリファレンス確認

以下のリンクは api-client カテゴリの処理時に追加済み:

| 既存ファイル | 追加内容 |
|---|---|
| `practices/architecture/error-handling.md` | `observability/error-tracking.md` へのリンク（api-client PR 時追加） |
| `practices/architecture/logging.md` | `observability/logging.md` と `observability/error-tracking.md` へのリンク（api-client PR 時追加） |
| `practices/performance/core-web-vitals.md` | `observability/rum.md` へのリンク（api-client PR 時追加） |

## 特記事項

- `observability/logging.md` は `architecture/logging.md` の詳細ガイドとして位置づけ、冒頭に参照ノートを記載
- Sentry の設定は v8 以降の `instrumentation-client.ts` 方式（旧 `sentry.client.config.ts` から変更）
- OpenTelemetry は Next.js 13.4+ の組み込みサポート（`instrumentationHook`）を活用
- INP は 2024年3月に FID を置き換えた新指標として明記

## 新規参照 URL リスト（_seen.json への追記対象）

```json
[
  "https://docs.sentry.io/platforms/javascript/guides/nextjs/",
  "https://docs.sentry.io/platforms/javascript/performance/",
  "https://docs.sentry.io/platforms/javascript/data-management/sensitive-data/",
  "https://docs.sentry.io/platforms/javascript/guides/react/features/error-boundary/",
  "https://docs.sentry.io/platforms/javascript/guides/nextjs/manual-setup/",
  "https://github.com/GoogleChrome/web-vitals",
  "https://web.dev/articles/rum-and-synthetic-monitoring",
  "https://web.dev/articles/user-centric-performance-metrics",
  "https://web.dev/articles/inp",
  "https://web.dev/articles/defining-core-web-vitals-thresholds",
  "https://opentelemetry.io/docs/languages/js/",
  "https://opentelemetry.io/docs/concepts/context-propagation/",
  "https://opentelemetry.io/docs/concepts/semantic-conventions/",
  "https://opentelemetry.io/docs/languages/js/exporters/",
  "https://opentelemetry.io/docs/languages/js/instrumentation/",
  "https://nextjs.org/docs/app/building-your-application/optimizing/open-telemetry",
  "https://www.w3.org/TR/trace-context/",
  "https://getpino.io/#/docs/redaction",
  "https://getpino.io/#/docs/child-loggers",
  "https://docs.datadoghq.com/real_user_monitoring/browser/",
  "https://docs.datadoghq.com/opentelemetry/",
  "https://developer.chrome.com/docs/crux/methodology/"
]
```
