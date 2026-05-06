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
  } catch {
    businessMetrics.checkoutAbandoned.add(1, { reason: 'payment_error' });
    throw;
  }
}
```

**出典**:
- [OpenTelemetry: Metrics API](https://opentelemetry.io/docs/languages/js/instrumentation/#creating-metrics) (OpenTelemetry公式)

**バージョン**: @opentelemetry/api 1.0+
**確信度**: 高
**最終更新**: 2026-05-05

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
    const { Resource } = await import('@opentelemetry/resources');
    const { ATTR_SERVICE_NAME } = await import('@opentelemetry/semantic-conventions');

    const sdk = new NodeSDK({
      resource: new Resource({
        [ATTR_SERVICE_NAME]: 'tech-radar-frontend',
        'service.version': process.env.APP_VERSION ?? 'unknown',
        'deployment.environment': process.env.NODE_ENV,
      }),

      // メトリクスエクスポーター（OTLP形式）
      metricReader: new PeriodicExportingMetricReader({
        exporter: new OTLPMetricExporter({
          url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT,
          headers: {
            'DD-API-KEY': process.env.DATADOG_API_KEY ?? '',  // Datadog の場合
          },
        }),
        exportIntervalMillis: 60_000,  // 60秒ごとにエクスポート
      }),
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
- [Datadog: OpenTelemetry](https://docs.datadoghq.com/opentelemetry/) (Datadog公式)

**バージョン**: @opentelemetry/sdk-node 0.50+
**確信度**: 中
**最終更新**: 2026-05-05

---

### 3. アラートを設定し、メトリクスを計測するだけで終わらせない

収集したメトリクスに対してアラートしきい値を設定し、
問題を自動検知できる体制を作る。

**根拠**:
- ダッシュボードを眺めるだけでは問題の検知が遅れる
- SLO（サービスレベル目標）に基づくアラートで誤アラートを減らせる
- エラー率・レイテンシ・ビジネス指標それぞれにアラートを設定する

**コード例**:
```yaml
# Datadog モニター定義例（IaC で管理）
# datadog-monitors.yaml
monitors:
  - name: "API Error Rate > 1%"
    type: metric alert
    query: "sum(last_5m):sum:system.api.error{service:frontend}.as_count() / sum:system.api.requests{service:frontend}.as_count() * 100 > 1"
    message: |
      API エラー率が 1% を超えています。
      @slack-alerts-channel
    thresholds:
      critical: 1
      warning: 0.5

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
**最終更新**: 2026-05-05

---

### 4. ダッシュボードは P50/P75/P90/P99 パーセンタイルで設計し、平均値トラップを避ける

レイテンシ・処理時間などの分布メトリクスはパーセンタイルで可視化し、
「遅いユーザー」を平均値で見えなくしないダッシュボード設計を行う。

**根拠**:
- 平均値（mean）は外れ値の影響を受け、ユーザー体験の実態を過小評価する（例：99% のユーザーが 100ms でも 1% が 30s ならば平均は 400ms 程度で「問題なし」に見える）
- P99 は「最も遅い 1% のユーザー」を示し、SLO 違反検知や高優先度ユーザーの保護に不可欠
- Google の Core Web Vitals も p75 を基準としており、分布ベース評価が業界標準になっている
- OpenTelemetry の `Histogram` 計装と Datadog の `distribution` メトリクスタイプを組み合わせることでパーセンタイルを自動計算できる

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

export async function fetchWithMetrics(url: string, options?: RequestInit) {
  const start = performance.now();
  const endpoint = new URL(url).pathname;

  try {
    const res = await fetch(url, options);
    const duration = performance.now() - start;

    apiLatencyHistogram.record(duration, {
      'http.route': endpoint,
      'http.status_code': res.status,
      'http.method': options?.method ?? 'GET',
    });

    return res;
  } catch (error) {
    const duration = performance.now() - start;
    apiLatencyHistogram.record(duration, {
      'http.route': endpoint,
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
- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/) (Google SRE / 2016)

**バージョン**: @opentelemetry/api 1.0+
**確信度**: 高
**最終更新**: 2026-05-06

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
# 例: 月次 SLO 99.9% → Error Budget = 0.1% = 月43.8分
monitors:
  # Fast Burn: 1時間で Error Budget の 5% を消費しているか
  - name: "SLO Burn Rate: Fast (1h window)"
    type: slo alert
    slo_id: "<your-slo-id>"
    time_window: 1h
    burn_rate_threshold: 14.4   # 1/0.001 * 5% / (1h/720h) = 14.4x
    message: |
      SLO の高速消費を検知。今すぐ対応が必要です。
      @pagerduty-critical

  # Slow Burn: 6時間で Error Budget の 10% を消費しているか
  - name: "SLO Burn Rate: Slow (6h window)"
    type: slo alert
    slo_id: "<your-slo-id>"
    time_window: 6h
    burn_rate_threshold: 6.0
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
- [Google SRE Workbook: Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) (Google SRE / 公開)
- [Datadog Docs: SLO-based Alerts](https://docs.datadoghq.com/service_management/service_level_objectives/burn_rate/) (Datadog公式)

**バージョン**: Datadog 最新版
**確信度**: 高
**最終更新**: 2026-05-06

---

## 関連プラクティス

- [`observability/tracing.md`](./tracing.md) - OpenTelemetry のトレース設定
- [`observability/rum.md`](./rum.md) - RUM とメトリクスの統合
- [`observability/error-tracking.md`](./error-tracking.md) - エラーメトリクスと追跡の連携
