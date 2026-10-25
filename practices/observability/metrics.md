# カスタムメトリクスのベストプラクティス

## ルール

### 1. ビジネスメトリクスとシステムメトリクスを分けて計測する

技術的なシステムメトリクス（CPU・メモリ・レイテンシ）だけでなく、
ビジネス価値に直結するメトリクス（コンバージョン率・カート放棄率等）を計測する。

**根拠**:
- システムメトリクスは「サービスが動いているか」を示すが、「ユーザー価値を提供できているか」を示さない
- ビジネスメトリクスは SLO（サービスレベル目標）の設計に必要
- 技術改善とビジネス成果のつながりを示す根拠になる

**コード例**:
```ts
// lib/metrics.ts
// OpenTelemetry でカスタムメトリクスを計測

import { metrics, ValueType } from '@opentelemetry/api';

const meter = metrics.getMeter('frontend', '1.0.0');

// ビジネスメトリクス
export const businessMetrics = {
  // カウンター（増加のみ）
  orderCreated: meter.createCounter('business.order.created', {
    description: '注文作成数',
    unit: 'count',
    valueType: ValueType.INT,
  }),

  checkoutStarted: meter.createCounter('business.checkout.started', {
    description: 'チェックアウト開始数',
  }),

  checkoutAbandoned: meter.createCounter('business.checkout.abandoned', {
    description: 'チェックアウト中断数',
  }),

  // ヒストグラム（分布を計測）
  checkoutDuration: meter.createHistogram('business.checkout.duration', {
    description: 'チェックアウト完了までの時間（ms）',
    unit: 'ms',
  }),

  cartItemCount: meter.createHistogram('business.cart.item_count', {
    description: 'カートのアイテム数',
    unit: 'items',
  }),
};

// システムメトリクス
export const systemMetrics = {
  apiLatency: meter.createHistogram('system.api.latency', {
    description: 'APIレスポンスタイム（ms）',
    unit: 'ms',
  }),

  apiErrorCount: meter.createCounter('system.api.error', {
    description: 'APIエラー数',
  }),

  cacheHitRate: meter.createObservableGauge('system.cache.hit_rate', {
    description: 'キャッシュヒット率（%）',
    unit: '%',
  }),
};

// 使用例
async function handleCheckout(cart: Cart) {
  businessMetrics.checkoutStarted.add(1, {
    'cart.item_count': cart.items.length,
    'user.plan': user.plan,
  });

  const startTime = Date.now();
  try {
    const order = await submitOrder(cart);
    const duration = Date.now() - startTime;

    businessMetrics.orderCreated.add(1, {
      'order.payment_method': order.paymentMethod,
      'order.currency': 'JPY',
    });
    businessMetrics.checkoutDuration.record(duration);
  } catch (error) {
    businessMetrics.checkoutAbandoned.add(1, { reason: 'payment_error' });
    throw error;
  }
}
```

**出典**:
- [OpenTelemetry: Metrics API](https://opentelemetry.io/docs/languages/js/instrumentation/#creating-metrics) (OpenTelemetry公式)

**バージョン**: @opentelemetry/api 1.0+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 2. Datadog / Prometheus 形式でメトリクスをエクスポートする

OpenTelemetry のメトリクスを OTLP エクスポーターで送信し、
Datadog・Prometheus・Grafana など任意のバックエンドで可視化する。

**根拠**:
- OpenTelemetry のエクスポーター設定を変えるだけでバックエンドを切り替えられる
- OTLP（OpenTelemetry Protocol）は CNCF 標準で多くのサービスがネイティブサポート
- Vercel Analytics・Datadog Real User Monitoring と組み合わせることでフルスタックの可観測性を実現

**コード例**:
```ts
// instrumentation.ts
export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    const { NodeSDK } = await import('@opentelemetry/sdk-node');
    const { OTLPMetricExporter } = await import('@opentelemetry/exporter-metrics-otlp-http');
    const { PeriodicExportingMetricReader } = await import('@opentelemetry/sdk-metrics');
    const { resourceFromAttributes } = await import('@opentelemetry/resources');
    const { ATTR_SERVICE_NAME } = await import('@opentelemetry/semantic-conventions');

    const sdk = new NodeSDK({
      // SDK 2.x 以降では Resource クラスが export されない。1.x 系（sdk-node 0.200 未満）は new Resource({...})
      resource: resourceFromAttributes({
        [ATTR_SERVICE_NAME]: 'tech-radar-frontend',
        'service.version': process.env.APP_VERSION ?? 'unknown',
        'deployment.environment': process.env.NODE_ENV,
      }),

      // メトリクスエクスポーター（OTLP形式）
      // 配列の metricReaders を使う（単数形の metricReader は 0.204 で非推奨になり、0.300（SDK 3.x）で削除された）
      metricReaders: [
        new PeriodicExportingMetricReader({
          exporter: new OTLPMetricExporter({
            url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT,
            headers: {
              'DD-API-KEY': process.env.DATADOG_API_KEY ?? '',  // Datadog の場合
            },
          }),
          exportIntervalMillis: 60_000,  // 60秒ごとにエクスポート
        }),
      ],
    });

    sdk.start();
  }
}
```

```ts
// クライアントサイドのメトリクスは API 経由で送信
// app/api/metrics/route.ts
import { NextRequest } from 'next/server';

export async function POST(request: NextRequest) {
  const metrics = await request.json();

  // Datadog の statsd/API 形式でフォワード
  await sendToDatadog(metrics);

  return new Response(null, { status: 204 });
}

// クライアント側から送信
function reportMetric(name: string, value: number, tags: Record<string, string>) {
  navigator.sendBeacon('/api/metrics', JSON.stringify({ name, value, tags }));
}
```

**出典**:
- [OpenTelemetry: Exporters](https://opentelemetry.io/docs/languages/js/exporters/) (OpenTelemetry公式)
- [OpenTelemetry: Resources（JS）](https://opentelemetry.io/docs/languages/js/resources/) (OpenTelemetry公式、`resourceFromAttributes()`) ※2026-10-25 確認
- [Upgrade to OpenTelemetry JS SDK 2.x](https://github.com/open-telemetry/opentelemetry-js/blob/main/doc/upgrade-to-2.x.md) (OpenTelemetry公式、`new Resource()` → `resourceFromAttributes()`、安定版 2.0.0+ / 実験版 0.200.0+) ※2026-10-25 確認
- [Upgrade to OpenTelemetry JS SDK 3.x](https://github.com/open-telemetry/opentelemetry-js/blob/main/doc/3.x/migration-guide.md) (OpenTelemetry公式、`NodeSDKConfiguration.metricReader` の削除と `metricReaders` への置き換え) ※2026-10-25 確認
- [opentelemetry-js experimental CHANGELOG 0.204.0](https://github.com/open-telemetry/opentelemetry-js/blob/main/experimental/CHANGELOG.md#02040) (OpenTelemetry公式、`metricReaders` の追加と `metricReader` の非推奨化) ※2026-10-25 確認
- [Datadog: OpenTelemetry](https://docs.datadoghq.com/opentelemetry/) (Datadog公式)

**バージョン**: @opentelemetry/sdk-node 0.204+（`metricReaders`。2026-10-25 時点の最新は 0.300.0 / SDK 3.x）/ @opentelemetry/resources 2.0+（`resourceFromAttributes()`。3.0 でも同じ）
**確信度**: 中
**最終更新**: 2026-10-25

---

### 3. アラートを設定し、メトリクスを計測するだけで終わらせない

収集したメトリクスに対してアラートしきい値を設定し、
問題を自動検知できる体制を作る。

**根拠**:
- ダッシュボードをながめるだけでは問題の検知が遅れる
- SLO（サービスレベル目標）に基づくアラートで誤アラートを減らせる
- エラー率・レイテンシ・ビジネス指標それぞれにアラートを設定する。しきい値の決め方は #5（SLO ベースの Burn Rate）に従う

**コード例**:
```yaml
# Datadog モニター定義例（IaC で管理）
# datadog-monitors.yaml
# API のエラー率・レイテンシは SLO を定義し、Burn Rate アラートにする（#5 を参照）
monitors:
  # SLO をまだ定義していないビジネス指標は、過去の分布から決めたしきい値で暫定運用し、
  # SLO を定めたら #5 の Burn Rate アラートに移す
  - name: "Checkout Abandonment Rate > 20%"
    type: metric alert
    query: "sum(last_10m):sum:business.checkout.abandoned{service:frontend}.as_count() / sum:business.checkout.started{service:frontend}.as_count() * 100 > 20"
    message: |
      チェックアウト中断率が 20% を超えています。
      @pagerduty
    thresholds:
      critical: 20
      warning: 15
```

**出典**:
- [Datadog Docs: Monitors](https://docs.datadoghq.com/monitors/create/types/metric/) (Datadog公式)

**バージョン**: Datadog 最新版
**確信度**: 中
**最終更新**: 2026-10-25

---

### 4. ダッシュボードは P50/P75/P90/P99 パーセンタイルで設計し、平均値トラップを避ける

レイテンシ・処理時間などの分布メトリクスはパーセンタイルで可視化し、
「遅いユーザー」を平均値で見えなくしないダッシュボード設計を行う。

**根拠**:
- 平均値（mean）は外れ値の影響を受け、ユーザー体験の実態を過小評価する
- P99 は「最も遅い 1% のユーザー」を示し、SLO 違反検知や高優先度ユーザーの保護に不可欠
- Google の Core Web Vitals も p75 を基準としており、分布ベース評価が業界標準になっている
- OpenTelemetry の `Histogram` 計装と Datadog の `distribution` メトリクスタイプを組み合わせることでパーセンタイルを自動計算できる
- `http.route` にはルートテンプレート（`/users/{id}`）を入れる。生のパスや ID を入れると時系列が爆発する（OpenTelemetry のセマンティック規約は低カーディナリティを要求している）

**コード例**:
```ts
// Good
// lib/metrics.ts - Histogram でレイテンシを計測（パーセンタイル計算対応）
import { metrics } from '@opentelemetry/api';

const meter = metrics.getMeter('frontend');

// Histogram: バックエンドがパーセンタイルを自動計算してくれる
const apiLatencyHistogram = meter.createHistogram('api.request.duration', {
  description: 'API リクエストのレイテンシ分布',
  unit: 'ms',
  // 境界値のヒントを与えることで精度を高める
  advice: {
    explicitBucketBoundaries: [10, 25, 50, 100, 250, 500, 1000, 2500, 5000],
  },
});

// route にはテンプレートを渡す（例: fetchWithMetrics('/users/{id}', `/api/users/${id}`)）。
// URL から生のパスを取り出さないので、相対 URL でも new URL() の例外が起きない
export async function fetchWithMetrics(route: string, url: string, options?: RequestInit) {
  const start = performance.now();

  try {
    const res = await fetch(url, options);
    const duration = performance.now() - start;

    apiLatencyHistogram.record(duration, {
      'http.route': route,
      'http.status_code': res.status,
      'http.method': options?.method ?? 'GET',
    });

    return res;
  } catch (error) {
    const duration = performance.now() - start;
    apiLatencyHistogram.record(duration, {
      'http.route': route,
      'error': 'true',
    });
    throw error;
  }
}

// Bad
// 平均値のみを計測するゲージはパーセンタイルを計算できない
const avgLatency = meter.createObservableGauge('api.avg_latency');
// 平均値だけでは「遅いユーザー」が見えない
```

```yaml
# Datadog ダッシュボード定義例（Terraform / JSON で IaC 管理）
# widgets:
#   - レイテンシ分布ウィジェット（ヒートマップ）
#     query: "p50:api.request.duration{service:frontend}"
#             "p75:api.request.duration{service:frontend}"
#             "p90:api.request.duration{service:frontend}"
#             "p99:api.request.duration{service:frontend}"
#
#   - SLO ウィジェット
#     target: p99 < 1000ms （99% のリクエストが 1 秒以内）
#     warning: p99 < 800ms
```

**出典**:
- [OpenTelemetry: Histogram](https://opentelemetry.io/docs/concepts/signals/metrics/#histogram) (OpenTelemetry公式)
- [OpenTelemetry Semantic Conventions: HTTP attributes](https://opentelemetry.io/docs/specs/semconv/registry/attributes/http/) (OpenTelemetry公式、`http.route` は "The matched route template" で低カーディナリティ必須) ※2026-10-25 確認
- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/) (Google SRE / 2016)

**バージョン**: @opentelemetry/api 1.0+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 5. Alerting Fatigue を避けるしきい値設計をSLOベースで行う

アラートのしきい値は「感覚」ではなくSLO（サービスレベル目標）から逆算して設計し、
アラート疲れによる重大インシデントの見落としを防ぐ。

**根拠**:
- しきい値が低すぎると誤アラートが多発し、開発チームがアラートを無視するようになる（Alerting Fatigue）
- SLO から Error Budget を計算し、「Error Budget の消費速度」をアラート基準にする（Burn Rate アラート）ことで誤検知を削減できる
- アラートは「人間が今すぐ行動する必要がある」場合のみ発火させる設計が重要
- Datadog・Grafana の Composite Monitor 機能で複数条件の AND/OR アラートを設定し、誤検知をさらに削減できる

**コード例**:
```yaml
# Good: SLO ベースの Burn Rate アラート（Datadog）
# 例: 30 日（720h）の SLO 99.9% → Error Budget = 0.1% = 30 日で 43.2 分
# Burn Rate = 消費した予算の割合 × (SLO 期間 / ウィンドウ)
monitors:
  # Fast Burn: 1時間で Error Budget の 2% を消費しているか
  - name: "SLO Burn Rate: Fast (1h window)"
    type: slo alert
    slo_id: "<your-slo-id>"
    time_window: 1h
    burn_rate_threshold: 14.4   # 2% × (720h / 1h) = 14.4x
    message: |
      SLO の高速消費を検知。今すぐ対応が必要です。
      @pagerduty-critical

  # Slow Burn: 6時間で Error Budget の 5% を消費しているか
  - name: "SLO Burn Rate: Slow (6h window)"
    type: slo alert
    slo_id: "<your-slo-id>"
    time_window: 6h
    burn_rate_threshold: 6.0    # 5% × (720h / 6h) = 6x
    message: |
      SLO の緩やかな消費を検知。作業時間内に確認してください。
      @slack-oncall

# Bad: 固定しきい値アラート（誤検知が多い）
  - name: "API Error Rate > 1%"  # トラフィック急増時に誤検知しやすい
    type: metric alert
    query: "sum(last_5m):sum:system.api.error.as_count() / sum:system.api.requests.as_count() * 100 > 1"
    # SLO との関連がなく、Error Budget を使い切っても気づかない場合がある
```

**出典**:
- [Google SRE Workbook: Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) (Google SRE / 公開、99.9% SLO で 1h・14.4 = 予算の 2%、6h・6 = 5%) ※2026-10-25 確認
- [Datadog Docs: SLO-based Alerts](https://docs.datadoghq.com/service_management/service_level_objectives/burn_rate/) (Datadog公式)

**バージョン**: Datadog 最新版
**確信度**: 高
**最終更新**: 2026-10-25

---

### 6. Sentry のアプリケーションメトリクスでエラー化する前のシグナルを追跡する

`Sentry.metrics` API（Counter / Distribution / Gauge）を使い、エラー発生前の
ビジネス・UXシグナルを計測する。「決済失敗数が急増しているがまだエラーは出ていない」
といった状況を事前に検知できる。ログとメトリクスの使い分けは「デバッグ用はログ、時系列で追う数値はメトリクス」とし、
同じイベント名・属性キーで両方を出して、メトリクスのスパイクから該当ログへ辿れるようにする。

**根拠**:
- ログは「その時何が起きたか」の再構築用、メトリクスは「事前に決めた問いを時系列で追う」用で、エラーが出ていない平常時でも追いたい数値はメトリクスにする。数えるためだけに全件ログ出力するとコストが無駄になる
- ログとメトリクスに同じイベント名・属性キーを使うと、メトリクスの異常から該当ログへ直接ドリルダウンできる（`Sentry.metrics.count(name, 1, { attributes })` 形式、SDK 10.25+）
- Sentry のエラートラッキングは「すでに壊れたもの」を捕えるが、問題の予兆（決済拒否・API 遅延・キュー滞積）は数値シグナルで現れる
- 同一ツール（Sentry）でエラーとメトリクスを一元管理でき、メトリクス異常とエラースパイクを同一ダッシュボードで相関分析できる
- Counter・Distribution・Gauge の3種類で主要なメトリクスパターンを網羅できる
- 属性（`attributes`）を付与することで `reason` / `queue` 別のドリルダウン分析が可能。`userId`・生のパス・エラーメッセージ全文のように値の種類が多いものや PII は属性に入れない（カーディナリティが爆発し、個人情報も送られる）
- 同じ仕組みは自作の Counter/Distribution だけでなく、SDK 組み込みの `bfcacheMetricsIntegration()` のように「ブラウザイベントを Application Metrics に変換する」用途にも使える。back/forward navigation の hit/miss を `browser.bfcache.navigation`（Counter）、ブロック理由別件数を `browser.bfcache.not_restored`（Counter）、ミス時のリロード所要時間を `browser.bfcache.reload.duration`（Distribution）として計測でき、初期化は1行で済む

**コード例**:
```ts
import * as Sentry from '@sentry/nextjs';

// Counter: イベントの発生回数を増加のみで追跡
// 用途: 決済失敗数、ログイン失敗数、API エラー数
// reason は決まった値（'insufficient_funds' など）に限る。userId は属性に入れない
function handlePaymentDeclined(reason: string) {
  Sentry.metrics.count('payment.declined', 1, {
    attributes: { reason },
  });
}

// Distribution: 値の分布を記録（パーセンタイル計算可能）
// 用途: API レスポンスタイム、ファイルサイズ、処理件数
// route にはテンプレートを渡す（例: fetchWithMetrics('/users/{id}', `/api/users/${id}`)）
async function fetchWithMetrics(route: string, url: string) {
  const start = Date.now();
  const res = await fetch(url);
  const latencyMs = Date.now() - start;

  Sentry.metrics.distribution('api.response_time', latencyMs, {
    unit: 'millisecond',
    attributes: { endpoint: route },
  });

  return res;
}

// Gauge: 現在の状態値を記録（最新値が保持される）
// 用途: キュー滞積数、アクティブセッション数、在庫数
function reportQueueDepth(queue: string, depth: number) {
  Sentry.metrics.gauge('queue.depth', depth, {
    attributes: { queue },
  });
}

// 使用例: 注文処理フロー全体を監視
async function processOrder(order: Order) {
  Sentry.metrics.count('order.processing.started', 1, {
    attributes: { paymentMethod: order.paymentMethod },
  });

  try {
    const result = await submitOrder(order);
    Sentry.metrics.count('order.processing.completed', 1);
    return result;
  } catch (error) {
    Sentry.metrics.count('order.processing.failed', 1, {
      attributes: {
        // error.message は値の種類が多く PII も含みうるので、エラーの種類（name）にとどめる
        reason: error instanceof Error ? error.name : 'unknown',
      },
    });
    throw error;
  }
}
```

**出典**:
- [Introducing Application Metrics](https://blog.sentry.io/introducing-application-metrics/) (Sentry Engineering / 2026-05) ※2026-05-06に実際にfetch成功
- [Sentry Docs: Metrics](https://docs.sentry.io/product/metrics/) (Sentry公式)
- [Sentry Docs: Metrics（Next.js）](https://docs.sentry.io/platforms/javascript/guides/nextjs/metrics/) (Sentry公式、`count` / `gauge` / `distribution` と `attributes`、SDK 10.25.0+) ※2026-10-25 確認（routine-followup マージ時の追加）
- [Sentry Docs: BFCache Metrics](https://docs.sentry.io/platforms/javascript/configuration/integrations/bfcache/) (Sentry公式、`bfcacheMetricsIntegration` は v11.0.0 から、3 つのメトリクス名と種類、`maxReasons`) ※2026-10-25 確認
- [Should you use a log or a metric?](https://blog.sentry.io/when-you-need-a-metric/) (Sentry Blog、ログとメトリクスの使い分けと同名イベントでの相関) ※2026-10-22 fetch
  > "Make it a metric when there's a number you'll track over time, even when nothing is wrong."
  > (セクション "When to use which")

**出典引用**:
> "The integration emits application metrics, so rather than hand you a pre-packaged dashboard that answers some questions and needs constant tweaking, you get to build your own."
> ([Measuring the back/forward cache with Application Metrics](https://blog.sentry.io/bfcache-metrics/), Sentry Engineering, セクション "How it works") ※2026-09-24に実際にfetch成功

**バージョン**: @sentry/nextjs 10.25+（`Sentry.metrics.count` / `gauge` / `distribution` と `attributes`）/ @sentry/browser 11+（`bfcacheMetricsIntegration`）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 7. Vercel Functions のカスタムメトリクスを `metric()` API で計測し、組み込み Observability データと並べて分析する

Vercel Functions 内で任意のアプリケーションメトリクスを `@vercel/functions` の `metric()` で送信し、Vercel Observability の組み込みデータ（デプロイ元・関数リージョン等のコンテキスト属性が自動付与される）と並べてクエリ・可視化できる。DB クエリ時間やキャッシュヒット率など、トレースだけでは追いにくい業務メトリクスを Observability 基盤に統合できる。

**根拠**:
- Vercel Observability の Query builder / Notebooks / CLI (`vc metrics`) から組み込みデータと同じ基盤でカスタムメトリクスを参照できる
- デプロイソースや関数リージョンなどのコンテキスト属性が Vercel 側で自動付与されるため、タグ設計の手間が減る
- Pro/Enterprise の Observability Plus プランが前提で、メトリクスは observability イベントとして課金される

**コード例**:
```ts
import { metric } from "@vercel/functions";

export async function GET() {
  const start = Date.now();
  const rows = await db.query('SELECT * FROM "my-table"');
  metric("database.query_ms", Date.now() - start, {
    table: "my-table",
    db: "my-database",
  });
  return Response.json(rows);
}
```

**出典引用**:
> "You can now emit your own metrics directly from your Vercel Functions and analyze them alongside Vercel's built-in observability data."
> ([Custom metrics are now supported in Vercel Observability](https://vercel.com/changelog/custom-metrics-are-now-supported-in-vercel-observability), 本文冒頭) ※2026-08-21に実際にfetch成功

**出典**:
- [Custom metrics are now supported in Vercel Observability](https://vercel.com/changelog/custom-metrics-are-now-supported-in-vercel-observability) (Vercel公式changelog、Pro / Enterprise の Observability Plus、observability イベントとして課金) ※2026-10-25 確認
- [Vercel Docs: Emit Custom Metrics from Vercel Functions](https://vercel.com/docs/observability/custom-metrics) (Vercel公式、`metric(name, value, attributes?)` の引数、名前と属性の制約、1 回の呼び出しの上限、自動で付く属性、`vercel metrics`) ※2026-10-25 確認
- [@vercel/functions CHANGELOG](https://github.com/vercel/vercel/blob/main/packages/functions/CHANGELOG.md) (vercel/vercel、3.9.0 で `metric()` を追加) ※2026-10-25 確認

**バージョン**: @vercel/functions 3.9.0+（`metric()`）、Pro / Enterprise の Observability Plus
**確信度**: 高（Vercel公式changelog・公式ドキュメント）
**最終更新**: 2026-10-25

---

## 関連プラクティス

- [`observability/tracing.md`](./tracing.md) - OpenTelemetry のトレース設定
- [`observability/rum.md`](./rum.md) - RUM とメトリクスの統合
- [`observability/error-tracking.md`](./error-tracking.md) - エラーメトリクスと追跡の連携
