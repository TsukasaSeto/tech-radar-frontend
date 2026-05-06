# Bootstrap v2: observability (2026-05-06)

## 概要

observability カテゴリに補強コンテンツを追加した。
既存 19 ルール（rum: 4, tracing: 3, error-tracking: 5, metrics: 3, logging: 4）に
新規 7 ルールを追加し、計 26 ルールとなった。

## 追加ルール一覧

### rum.md (+1 rule)

| # | ルール | 確信度 |
|---|--------|--------|
| 5 | Long Animation Frames API でJSブロッキングの根本原因を特定する | 中 |

### tracing.md (+1 rule)

| # | ルール | 確信度 |
|---|--------|--------|
| 4 | `trace.getActiveSpan()` でコンテキスト伝播を確認し、孤立スパンを防ぐ | 中 |

### error-tracking.md (+1 rule)

| # | ルール | 確信度 |
|---|--------|--------|
| 6 | `fingerprint` でエラーをグルーピングし、アラートノイズを削減する | 高 |

### metrics.md (+2 rules)

| # | ルール | 確信度 |
|---|--------|--------|
| 4 | ダッシュボードは P50/P75/P90/P99 パーセンタイルで設計し、平均値トラップを避ける | 高 |
| 5 | Alerting Fatigue を避けるしきい値設計をSLOベースで行う | 高 |

## 参照した公式ドキュメント

| ソース | アクセス方法 | 結果 |
|--------|------------|------|
| open-telemetry/opentelemetry-js README.md | GitHub MCP | アクセス拒否（リポジトリ制限） |
| GoogleChrome/web-vitals README.md | GitHub MCP | アクセス拒否（リポジトリ制限） |
| opentelemetry.io/docs/languages/js/getting-started/browser/ | WebFetch | 403 Forbidden |
| opentelemetry.io/docs/languages/js/instrumentation/ | WebFetch | 403 Forbidden |
| raw.githubusercontent.com/open-telemetry/opentelemetry-js/main/README.md | WebFetch | 成功 |
| github.com/GoogleChrome/web-vitals/blob/main/README.md | WebFetch | 成功 |

## 重複チェック

- rum.md のルール5（LoAF API）: 既存ルール4（INP attribution）は `longAnimationFrameEntries` への言及のみ。LoAF の独立した監視実装は新規。
- tracing.md のルール4（getActiveSpan）: 既存ルールはサーバーサイドの OTel のみ。ブラウザ向け ZoneContextManager と孤立スパン検出は新規。
- error-tracking.md のルール6（fingerprint）: 既存ルール3は beforeSend での PII マスキング。fingerprint によるグルーピング制御は新規。
- metrics.md のルール4（P50/P90/P99）: 既存ルール3はアラートしきい値の YAML 定義のみ。パーセンタイルを前提とした Histogram 計装は新規。
- metrics.md のルール5（Burn Rate）: 既存ルール3は固定しきい値アラート。SLO/Burn Rate ベース設計は新規。
