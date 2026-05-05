# 分散トレーシングのベストプラクティス

## ルール

### 1. OpenTelemetry で統一されたトレーシング基盤を構築する

ベンダー固有のSDKではなく、OpenTelemetry（OTel）の標準 API を使い、
バックエンドに依存しないトレーシング基盤を構築する。

**根拠**:
- OpenTelemetry は CNCF のオープンスタンダードであり、Datadog・Jaeger・Zipkin・Sentry など複数バックエンドに対応
- OTel に準拠すれば、バックエンドのログサービスを切り替えてもコード変更が不要
- Next.js は OpenTelemetry の組み込みサポートを持ち、`@vercel/otel` で最小設定で導入できる
- W3C Trace Context ヘッダー（`traceparent`）で異なるサービス間のトレースを連結できる

**コード例**:
```bash
npm install @vercel/otel @opentelemetry/api
```

```ts
// instrumentation.ts（Next.js 組み込みの OpenTelemetry 設定）
export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    const { registerOTel } = await import('@vercel/otel');
    registerOTel({
      serviceName: process.env.SERVICE_NAME ?? 'tech-radar-frontend',
      // エクスポーター設定（OTEL_EXPORTER_OTLP_ENDPOINT 環境変数で制御）
    });
  }
}

// next.config.ts
const nextConfig = {
  experimental: {
    instrumentationHook: true,  // Next.js 14 以降は不要（安定版）
  },
};
```

```ts
// lib/tracer.ts - カスタムスパンの作成
import { trace, SpanStatusCode, context } from '@opentelemetry/api';

const tracer = trace.getTracer('tech-radar-frontend', '1.0.0');

// カスタムスパンで重要な処理を計測
export async function traced<T>(
  spanName: string,
  fn: () => Promise<T>,
  attributes?: Record<string, string | number | boolean>,
): Promise<T> {
  return tracer.startActiveSpan(spanName, async (span) => {
    if (attributes) {
      span.setAttributes(attributes);
    }
    try {
      const result = await fn();
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: String(error) });
      span.recordException(error as Error);
      throw error;
    } finally {
      span.end();
    }
  });
}

// 使用例
const products = await traced(
  'product.list',
  () => fetchProducts(filters),
  { 'query.limit': filters.limit, 'query.category': filters.category ?? 'all' },
);
```

**出典**:
- [Next.js Docs: OpenTelemetry](https://nextjs.org/docs/app/building-your-application/optimizing/open-telemetry) (Next.js公式)
- [OpenTelemetry JS](https://opentelemetry.io/docs/languages/js/) (OpenTelemetry公式)

**バージョン**: Next.js 13.4+, @opentelemetry/api 1.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. W3C Trace Context ヘッダーを使ってフロント→BFF→バックエンドでトレースを連結する

フロントエンドのトレース ID をバックエンドへの HTTP リクエストヘッダー
（`traceparent`）に付与し、分散トレースを一本の線として可視化する。

**根拠**:
- `traceparent` は W3C の標準ヘッダーで、OpenTelemetry が自動的に注入・抽出する
- トレース ID が連結することで、UI の操作から DB クエリまでを1つのトレースで表示できる
- 本番障害の根本原因分析（RCA）に不可欠
- OpenTelemetry の `fetch` 計装を使えば、手動での `traceparent` 付与は不要

**コード例**:
```ts
// lib/instrumented-fetch.ts
// @vercel/otel または @opentelemetry/instrumentation-fetch が自動計装する
// 手動計装が必要な場合:
import { context, propagation, trace } from '@opentelemetry/api';

export async function fetchWithTracing(url: string, options: RequestInit = {}) {
  // 現在のコンテキストから traceparent ヘッダーを生成
  const headers = new Headers(options.headers);
  propagation.inject(context.active(), {
    set(carrier, key, value) {
      headers.set(key, value);
    },
  });

  return fetch(url, { ...options, headers });
}

// app/api/checkout/route.ts - サーバー間でトレースを伝播
import { headers } from 'next/headers';
import { context, propagation } from '@opentelemetry/api';

export async function POST(request: Request) {
  const incomingHeaders = await headers();

  // 受け取ったトレースコンテキストを抽出
  const ctx = propagation.extract(context.active(), {
    get(carrier, key) { return incomingHeaders.get(key) ?? undefined; },
    keys() { return Array.from(incomingHeaders.keys()); },
  });

  // コンテキストを引き継いでバックエンドを呼び出し
  return context.with(ctx, async () => {
    const outgoingHeaders = new Headers();
    propagation.inject(context.active(), {
      set(carrier, key, value) { outgoingHeaders.set(key, value); },
    });

    const res = await fetch('https://payment-service/process', {
      method: 'POST',
      headers: outgoingHeaders,
      body: await request.text(),
    });

    return Response.json(await res.json());
  });
}
```

**出典**:
- [W3C: Trace Context](https://www.w3.org/TR/trace-context/) (W3C)
- [OpenTelemetry: Context Propagation](https://opentelemetry.io/docs/concepts/context-propagation/) (OpenTelemetry公式)

**バージョン**: @opentelemetry/api 1.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. スパン属性にビジネスコンテキストを付与して検索性を高める

技術的な情報だけでなく、ユーザー ID・注文 ID・商品カテゴリなどのビジネス属性を
スパンに付与し、トレーシングツールでの検索を容易にする。

**根拠**:
- 「ユーザー X の注文 Y でエラーが起きた」という調査で Trace ID がわからない場合でも、属性で絞り込める
- スパン属性は OpenTelemetry のセマンティック規約（Semantic Conventions）に従うと他ツールとの相互運用性が高い
- PII（個人情報）を直接付与せず、ハッシュ化した ID や匿名化した値を使う

**コード例**:
```ts
// lib/tracer.ts
import { trace, SpanStatusCode } from '@opentelemetry/api';
import { ATTR_HTTP_ROUTE, ATTR_HTTP_REQUEST_METHOD } from '@opentelemetry/semantic-conventions';

const tracer = trace.getTracer('frontend');

// ビジネスコンテキストを持つスパン
export async function processOrder(orderId: string, userId: string) {
  return tracer.startActiveSpan('order.process', async (span) => {
    // セマンティック規約に従う属性
    span.setAttributes({
      'order.id': orderId,
      'user.id': hashId(userId),       // PII: ハッシュ化して付与
      'order.total': 5000,
      'order.currency': 'JPY',
      'order.item_count': 3,
    });

    try {
      const result = await submitOrder(orderId);
      span.setAttributes({ 'order.status': result.status });
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: error instanceof Error ? error.message : 'Unknown error',
      });
      span.recordException(error as Error);
      throw error;
    } finally {
      span.end();
    }
  });
}
```

**出典**:
- [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/concepts/semantic-conventions/) (OpenTelemetry公式)

**バージョン**: @opentelemetry/api 1.0+, @opentelemetry/semantic-conventions 1.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

## 関連プラクティス

- [`observability/logging.md`](./logging.md) - リクエスト ID とログの相関
- [`observability/metrics.md`](./metrics.md) - トレースとメトリクスの連携
- [`api-client/error-handling.md`](../api-client/error-handling.md) - API エラー時のスパンへのエラー記録
