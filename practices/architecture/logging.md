# ロギングのアーキテクチャ責務

ロギング自体の実装パターン（Pino 設定・redact・ログレベル設計・Correlation ID 伝播）は [`observability/logging.md`](../observability/logging.md) に集約する。
このファイルは **「どの層が何をログするか」というアーキテクチャレベルの責任分担**のみを定義する。

## ルール

### 1. 「ビジネスイベントログ」と「運用テレメトリ」をアーキテクチャ層で分離する

アプリ内部のドメインイベント（注文作成 / 認証成功 / 課金完了など）と、運用観測のためのテレメトリ（HTTP メトリクス / レイテンシ / リソース使用量）は、書き出し経路と保持期間が異なるため、コードの層も分けて扱う。

**根拠**:
- ビジネスイベントは長期保持・監査対象になりやすく、フィールドスキーマを安定させる必要がある（ドメイン層 / use case 層が責任を持つ）
- 運用テレメトリは短期保持で構わないが、サンプリング・カーディナリティ管理が要る（インフラ層 / middleware が責任を持つ）
- 両者を 1 つのロガー呼び出しで混ぜると、PII 混入とコスト過大の双方が起きやすい

**層と責任の対応**:

| 層 | ログするもの | 主なツール |
|---|---|---|
| ドメイン / use case | ビジネスイベント（誰が何をしたか、結果は何か） | `logger.info` / `logger.warn` |
| Server Action / Route Handler | 入力検証失敗・認可拒否・外部呼び出し結果 | `logger.warn` / `logger.error` |
| middleware / proxy | リクエスト境界のコンテキスト付与（trace ID / user ID） | `logger.child()` + `AsyncLocalStorage` |
| インフラ層 | HTTP メトリクス・レイテンシ・リソース使用量 | OpenTelemetry / Vercel Analytics |
| クライアント | UI からの未捕捉エラー・パフォーマンス計測 | Sentry / Web Vitals |

具体的な書き方は [`observability/logging.md`](../observability/logging.md)、エラー特化の経路は [`observability/error-tracking.md`](../observability/error-tracking.md)、分散トレーシングは [`observability/tracing.md`](../observability/tracing.md) を参照。

**バージョン**: 全フロントエンド共通
**確信度**: 高
**最終更新**: 2026-05-16

---

### 2. クライアントから送るログは PII を含めないことをアーキテクチャ境界で担保する

サーバー / クライアントの境界で「Client から外に出すログ・エラー」のサニタイズを必ず通す。
個別の `logger.error(err)` 呼び出し側で都度マスクするのではなく、**Client logger の wrapping 層と Sentry の `beforeSend` でグローバルに通す**。

**根拠**:
- 個別呼び出しでのマスクは抜けが出る。境界で 1 回掛けるほうが防御が確実
- Cookie / Authorization ヘッダー / フォームの password・token 系は network のスナップショットにも乗りやすい
- DAL のレスポンスは DTO に整形してから渡す前提（[`architecture/data-access-layer.md`](./data-access-layer.md) Rule #3 と整合）

実装パターン（`beforeSend` / `beforeBreadcrumb` での自動マスク、Pino `redact` 設定）は [`observability/logging.md`](../observability/logging.md) Rule #1 および [`observability/error-tracking.md`](../observability/error-tracking.md) Rule #3 に集約。

**バージョン**: Pino 9+, @sentry/nextjs 8+
**確信度**: 高
**最終更新**: 2026-05-16

---

## 関連プラクティス

- [`observability/logging.md`](../observability/logging.md) - ロギングの実装ガイド（Pino 設定 / ログレベル / Correlation ID）
- [`observability/error-tracking.md`](../observability/error-tracking.md) - Sentry の詳細設定とリリーストラッキング
- [`observability/tracing.md`](../observability/tracing.md) - 分散トレーシング
- [`architecture/data-access-layer.md`](./data-access-layer.md) - DTO 境界での PII 除去
