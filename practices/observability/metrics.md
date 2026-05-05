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

### 4. メトリクスに高カーディナリティ属性を付与し、スパイクからトレースへの調査チェーンを構築する

メトリクスを記録する際に `userId`・`projectId`・`region` などの属性を必ず付与する。
スパイク検知後に属性でフィルタリングし、影響を受けたリクエストのトレースに直接ジャンプできる体制を作る。

**根拠**:
- 従来のインフラ監視ツールはメトリクス集約時に高カーディナリティフィールドを削除するため、「東海岸ユーザーのみ遅い」などの細粒度の質問に答えられない
- メトリクスにコンテキスト属性（ユーザーID・リージョン等）を含めることで、スパイクの影響範囲を即座に特定できる
- Sentry Application Metrics は属性ベースのフィルタリングからトレース・エラーへの直接ジャンプをサポートする

**コード例**:
```ts
// OpenTelemetry: カスタムメトリクスに高カーディナリティ属性を付与
import { metrics } from '@opentelemetry/api';

const meter = metrics.getMeter('frontend');
const checkoutFailed = meter.createCounter('checkout.failed');

async function handleCheckout(cart: Cart, user: User) {
  try {
    await submitOrder(cart);
  } catch (error) {
    // 高カーディナリティ属性を付与 → スパイク時に特定ユーザー・リージョンを絞り込み可能
    checkoutFailed.add(1, {
      'user.id': user.id,
      'user.region': user.region,
      'payment.method': cart.paymentMethod,
      'error.type': (error as Error).name,
    });
    throw error;
  }
}

// Sentry SDK: より簡易な代替手段（Sentry を既に導入済みのプロジェクト向け）
import * as Sentry from '@sentry/nextjs';

async function handleCheckout(cart: Cart, user: User) {
  try {
    await submitOrder(cart);
  } catch (error) {
    // Counter: 発生率の追跡
    Sentry.metrics.increment('checkout.failed', 1, {
      tags: { region: user.region, paymentMethod: cart.paymentMethod },
    });

    // Distribution: 値の分布（例: カートアイテム数）
    Sentry.metrics.distribution('cart.item_count', cart.items.length, {
      tags: { userId: user.id },
    });
    throw error;
  }
}
```

**調査フロー**:
```
1. メトリクスダッシュボードでスパイクを検知
2. 属性フィルター（例: region=us-east）でスパイクの範囲を絞り込み
3. 影響ユーザーのトレース / エラーレポートに直接ジャンプ
4. 根本原因（特定の決済手段・リージョン・プロジェクト）を特定
```

**出典**:
- [Introducing Application Metrics: Track the signal, see the spike, jump to the trace](https://blog.sentry.io/introducing-application-metrics/) (Sentry Blog / 2026-05-05)

**バージョン**: @sentry/nextjs 8+ または @opentelemetry/api 1.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

## 関連プラクティス

- [`observability/tracing.md`](./tracing.md) - OpenTelemetry のトレース設定
- [`observability/rum.md`](./rum.md) - RUM とメトリクスの統合
- [`observability/error-tracking.md`](./error-tracking.md) - エラーメトリクスと追跡の連携
