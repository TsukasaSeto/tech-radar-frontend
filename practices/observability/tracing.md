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
- `instrumentation.ts` は Next.js 15.0.0 で安定版になり、v15 以降は設定不要で読み込まれる（v14 以前は `next.config` の `experimental.instrumentationHook: true` が必要）
- フロントエンドの Sentry SDK とバックエンドの OTel を組み合わせるハイブリッド構成は #5 を参照

**コード例**:
```bash
npm install @vercel/otel @opentelemetry/sdk-logs @opentelemetry/api-logs @opentelemetry/instrumentation @opentelemetry/api
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
- [Next.js Docs: OpenTelemetry](https://nextjs.org/docs/app/guides/open-telemetry) (Next.js公式) ※2026-10-25 確認
- [Next.js Docs: instrumentation.js（Version History）](https://nextjs.org/docs/app/api-reference/file-conventions/instrumentation) (Next.js公式) ※2026-10-25 確認
- [OpenTelemetry JS](https://opentelemetry.io/docs/languages/js/) (OpenTelemetry公式)

**バージョン**: Next.js 15+（v14 以前は `experimental.instrumentationHook` が必要）, @opentelemetry/api 1.x
**確信度**: 高
**最終更新**: 2026-10-25

---

### 2. W3C Trace Context ヘッダーを使ってフロント→BFF→バックエンドでトレースを連結する

フロントエンドのトレース ID をバックエンドへの HTTP リクエストヘッダー
（`traceparent`）に付与し、分散トレースを一本の線として可視化する。

**根拠**:
- `traceparent` は W3C の標準ヘッダーで、OpenTelemetry が自動的に注入・抽出する
- トレース ID が連結することで、UI の操作から DB クエリまでを1つのトレースで表示できる
- 本番障害の根本原因分析（RCA）に不可欠
- OpenTelemetry の `fetch` 計装を使えば、手動での `traceparent` 付与は不要
- 手動で伝播するときは、`propagation.inject()` / `extract()` の第 2 引数（carrier）に普通のオブジェクトを渡す。既定の setter / getter は carrier のプロパティを読み書きするだけなので、`Headers` を直接扱うなら setter / getter を第 3 引数で渡す
- OTel JS の `NodeSDK` の既定 propagator はすでに W3C Trace Context + Baggage（`textMapPropagator` の既定値。仕様上の `OTEL_PROPAGATORS` の既定も `tracecontext,baggage`）なので、`CompositePropagator` で同じ組み合わせを明示しても挙動は変わらない
- v16 の `proxy.ts` は Node.js ランタイム固定で、ヘッダーを書き換えなければ受信した `traceparent` / `tracestate` はそのまま後段に届く。同じ値を詰め直すだけの転送処理は不要
- `NodeSDK` は Edge ランタイムで動かないため、`instrumentation.ts` で `NEXT_RUNTIME === 'nodejs'` の分岐の中から動的 import する。Edge でも計装が必要なら `@vercel/otel` を使う
- **Server Actions はトレースコンテキストを自動伝播しない**: OTel の観点では Server Action は新規 HTTP リクエストとして扱われるため、スパンを手動で作成してコンテキストを継続させる必要がある

**コード例**:
```ts
// lib/instrumented-fetch.ts
// @vercel/otel または @opentelemetry/instrumentation-fetch が自動計装する
// 手動計装が必要な場合（NodeSDK / registerOTel が登録した propagator が使われる）:
import { context, propagation } from '@opentelemetry/api';

export async function fetchWithTracing(url: string, options: RequestInit = {}) {
  // 普通のオブジェクトを carrier にして traceparent / tracestate を書き出し、Headers に詰め替える
  const carrier: Record<string, string> = {};
  propagation.inject(context.active(), carrier);

  const headers = new Headers(options.headers);
  for (const [key, value] of Object.entries(carrier)) {
    headers.set(key, value);
  }

  return fetch(url, { ...options, headers });
}
```

```ts
// app/api/checkout/route.ts - サーバー間でトレースを伝播
import { context, propagation } from '@opentelemetry/api';

export async function POST(request: Request) {
  // Headers を carrier にする場合は getter を第 3 引数で渡す
  const ctx = propagation.extract(context.active(), request.headers, {
    get: (carrier, key) => carrier.get(key) ?? undefined,
    keys: (carrier) => Array.from(carrier.keys()),
  });

  // コンテキストを引き継いでバックエンドを呼び出し
  return context.with(ctx, async () => {
    const outgoing: Record<string, string> = { 'Content-Type': 'application/json' };
    propagation.inject(context.active(), outgoing);

    const res = await fetch('https://payment-service/process', {
      method: 'POST',
      headers: outgoing,
      body: await request.text(),
    });

    return Response.json(await res.json());
  });
}
```

**Node.js 専用 SDK の読み込み例（Next.js 公式の手動構成）**:
```ts
// instrumentation.ts — Node.js 専用の SDK は nodejs 分岐の中で動的 import する（Edge 用バンドルに入れない）
export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    await import('./instrumentation.node');
  }
}
```

```ts
// instrumentation.node.ts — propagator は既定（W3C Trace Context + Baggage）のままでよい
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { NodeSDK } from '@opentelemetry/sdk-node';

const sdk = new NodeSDK({
  serviceName: 'next-app',
  traceExporter: new OTLPTraceExporter(),
});
sdk.start();
```

**出典**:
- [W3C: Trace Context](https://www.w3.org/TR/trace-context/) (W3C)
- [OpenTelemetry: Context Propagation](https://opentelemetry.io/docs/concepts/context-propagation/) (OpenTelemetry公式)
- [opentelemetry-js: api/src/api/propagation.ts](https://github.com/open-telemetry/opentelemetry-js/blob/main/api/src/api/propagation.ts) (OpenTelemetry公式ソース、`inject` / `extract` の第 3 引数と既定の setter / getter) ※2026-10-25 確認
- [OpenTelemetry: SDK Environment Variables](https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/) (OpenTelemetry公式、`OTEL_PROPAGATORS` の既定値) ※2026-10-25 確認
- [opentelemetry-js: sdk-node README](https://github.com/open-telemetry/opentelemetry-js/tree/main/experimental/packages/opentelemetry-sdk-node) (OpenTelemetry公式、`textMapPropagator` の既定値と `serviceName` / `traceExporter`) ※2026-10-25 確認
- [Next.js Docs: OpenTelemetry（Manual OpenTelemetry configuration）](https://nextjs.org/docs/app/guides/open-telemetry) (Next.js公式) ※2026-10-25 確認
- [Next.js Docs: proxy.js（Runtime）](https://nextjs.org/docs/app/api-reference/file-conventions/proxy) (Next.js公式) ※2026-10-25 確認
- [OpenTelemetry in Next.js: traces that survive the edge/server boundary without losing context](https://dev.to/jtorchia/opentelemetry-in-nextjs-traces-that-survive-the-edgeserver-boundary-without-losing-context-3d09) (dev.to、Server Actions の伝播) ※2026-06-17 fetch

**バージョン**: @opentelemetry/api 1.x, Next.js 16（`proxy.ts` は Node.js ランタイム）
**確信度**: 高
**最終更新**: 2026-10-25

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

### 4. `trace.getActiveSpan()` でコンテキスト伝播を確認し、孤立スパンを防ぐ

`trace.getActiveSpan()` を使って現在アクティブなスパンの有無を確認し、
トレースコンテキストが途切れていないかをランタイムで検証する。
ユーザー操作の起点など、親がないのが正常な場所ではルートスパンを始め、親があるはずの処理では子スパンを始める、と分けて扱う。

**根拠**:
- ブラウザ側でのトレーシングは、`async/await` の境界やコールバックでコンテキストが失われやすい
- アクティブなスパンがないときに開始したスパンは、新しいトレースのルートになる。クリックやページ遷移の起点で意図して始めるルートスパンは正当だが、本来は親があるはずの処理（fetch の内側など）で起きるとトレースが分断される
- `@opentelemetry/sdk-trace-web` と `ZoneContextManager` を組み合わせると、Zone.js ベースでコンテキストを維持できる。ただし `ZoneContextManager` は ES2017 以降をターゲットにしたコード（native の `async/await`）では動かず、ES2015 へのトランスパイルが必要
- 開発時に `getActiveSpan()` チェックをアサーションとして使うことで、コンテキスト喪失バグを早期発見できる

**コード例**:
```ts
// Good
// lib/tracer-browser.ts - ブラウザ向け OTel 初期化（@opentelemetry/sdk-trace-web）
import { WebTracerProvider } from '@opentelemetry/sdk-trace-web';
import { ZoneContextManager } from '@opentelemetry/context-zone';
import { W3CTraceContextPropagator } from '@opentelemetry/core';
import { trace, context } from '@opentelemetry/api';

const provider = new WebTracerProvider();
provider.register({
  // ZoneContextManager で async コンテキストを正確に伝播
  contextManager: new ZoneContextManager(),
  propagator: new W3CTraceContextPropagator(),
});

const tracer = trace.getTracer('browser-app', '1.0.0');

// アクティブスパンの確認ユーティリティ
export function assertActiveSpan(operationName: string): void {
  const span = trace.getActiveSpan();
  if (!span && process.env.NODE_ENV !== 'production') {
    console.warn(
      `[OTel] No active span in "${operationName}". ` +
      'This operation will not be traced. Check context propagation.',
    );
  }
}

// ユーザー操作の起点ではルートスパンを始める（親がないのは正常）
export async function withInteractionSpan<T>(
  name: string,
  fn: () => Promise<T>,
): Promise<T> {
  return tracer.startActiveSpan(name, { root: true }, async (span) => {
    try {
      return await fn();
    } finally {
      span.end();
    }
  });
}

// 親があるはずの処理では子スパンを始め、親がなければ開発時に警告する
export async function withChildSpan<T>(
  name: string,
  fn: () => Promise<T>,
): Promise<T> {
  assertActiveSpan(name);
  return tracer.startActiveSpan(name, async (childSpan) => {
    try {
      return await fn();
    } finally {
      childSpan.end();
    }
  });
}

// Bad
// startSpan はスパンをアクティブにしないため、内側の fetch のスパンが親と紐付かず別のトレースになる
const span = tracer.startSpan('checkout.submit');
await fetch('/api/checkout', { method: 'POST' });
span.end();
```

**出典**:
- [OpenTelemetry JS: Context API](https://opentelemetry.io/docs/languages/js/context/) (OpenTelemetry公式 / 2024)
- [OpenTelemetry JS README](https://github.com/open-telemetry/opentelemetry-js/blob/main/README.md) (open-telemetry/opentelemetry-js)
- [opentelemetry-js: sdk-trace-web README（v2.12.0）](https://github.com/open-telemetry/opentelemetry-js/blob/v2.12.0/packages/opentelemetry-sdk-trace-web/README.md) (OpenTelemetry公式、`ZoneContextManager` は ES2017+ ターゲットで動かない) ※2026-10-25 確認
- [opentelemetry-js: sdk-trace-web README（main）](https://github.com/open-telemetry/opentelemetry-js/blob/main/packages/opentelemetry-sdk-trace-web/README.md) (OpenTelemetry公式、`@opentelemetry/sdk-trace` への統合) ※2026-10-25 確認

**バージョン**: @opentelemetry/api 1.x, @opentelemetry/sdk-trace-web 2.x, @opentelemetry/context-zone 2.x（OTel JS SDK 3.0 では sdk-trace-web は `@opentelemetry/sdk-trace` に統合され、2.12.0 が最終版）
**確信度**: 中
**最終更新**: 2026-10-25

---

### 5. Sentry と OpenTelemetry を排他的でなく補完的に統合する

「Sentry か OpenTelemetry か」という二択ではなく、**OpenTelemetry が既に機能している箇所はそのまま維持し、
Sentry SDK をアプリケーションコンテキスト（エラー詳細・ユーザー情報・リプレイ）が必要な箇所に追加する**設計にする。
OTLP エクスポートは現時点でログ・トレースのみサポートし、メトリクスは未対応（制限事項として把握しておく）。

**根拠**:
- OpenTelemetry はサービス間トレース伝播（W3C `traceparent`）に強く、ベンダー中立性がある
- Sentry SDK はスタックトレース・セッションリプレイ・パフォーマンス詳細など**アプリケーションコンテキスト**の収集に強い
- 2者は OTLP 経由で共存でき、`traceparent` / `baggage` ヘッダーを通じてトレースを連結できる
- どちらか一方に統一しようとすると、相手の強みを失う
- Sentry の `OtlpIntegration` はアクティブな OTel トレースコンテキストを読み取り、発生した Sentry イベントへ自動でスタンプする。これにより「エラーから原因スパンへ」「スパンから発生したエラーへ」の双方向ナビゲーションが可能になる
- 典型的なハイブリッド構成は、フロントエンドで Session Replay・source maps・ブラウザエラー追跡に強い Sentry SDK を使い、バックエンドは既存の OTel 計装を維持して OTLP スパンを Sentry に送る形。SDK を置き換えずに単一の分散トレースにまとまる
- `tracePropagationTargets` の既定は、ブラウザでは同一オリジンへのリクエストだけ、サーバーではすべての送信リクエストにトレースヘッダーを付ける。別オリジンの API へ伝播したいときは、そのオリジンを明示的に加える

**コード例**:
```ts
// instrumentation-client.ts — Sentry 初期化（OTel との共存設定）
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  integrations: [Sentry.browserTracingIntegration()],
  // sentry-trace / baggage に加えて W3C の traceparent も付与し、OTel 計装のバックエンドへつなぐ（既定は false）
  propagateTraceparent: true,
  // 既定（ブラウザ）は同一オリジンだけ。指定すると一致した URL にだけ付くので、同一オリジンの /api と本番 API のオリジンを並べる
  tracePropagationTargets: [/^\/api\//, /^https:\/\/api\.example\.com\//],
});
```

```python
# バックエンド CORS 設定 — Sentry/OTel ヘッダーを許可（FastAPI / Django 等）
allow_headers = ["sentry-trace", "baggage", "traceparent"]
```

**ハイブリッド構成の留意点**:
- フロントエンド: `@sentry/nextjs` で ブラウザエラー・Session Replay・source maps を取得
- バックエンド: 既存 OTel の OTLP スパンを `https://o<id>.ingest.sentry.io/api/<proj>/otlp/` へ Export（SDK 置き換え不要）
- CORS 許可: `sentry-trace`, `baggage`, `traceparent` ヘッダーを許可し、cross-origin トレース断絶を防ぐ

**現時点の制限事項**:
- Sentry の OTLP エクスポートはログ・トレースのみサポート（メトリクスは未対応）
- メトリクスは OTel SDK 経由（Prometheus / Datadog 等）で別途エクスポートが必要

**アンチパターン**:
- OTel を全廃して Sentry SDK のみにする → ベンダーロックインとサービス間トレース連携の喪失
- 別オリジンのバックエンドを `tracePropagationTargets` に加えない → ブラウザの既定では同一オリジンにしか付与されず、フロントエンドからバックエンドへのトレースが途切れる
- 本番 API のオリジンを加えず開発用の `localhost` だけを指定する → 本番では伝播しない

**出典**:
- [Sentry Blog: How Sentry and OpenTelemetry Work Together](https://blog.sentry.io/sentry-opentelemetry-work-together/) (Sentry公式ブログ、フロントエンド Sentry SDK + バックエンド OTel のハイブリッド構成) ※2026-05-27に実際にfetch成功
- [Your OTel spans, our errors: A Sentry love story in one trace](https://blog.sentry.io/otel-spans-errors-sentry-trace/) (Sentry公式ブログ、`OtlpIntegration` によるトレースコンテキスト自動付与の実装詳細) ※2026-08-05に実際にfetch成功
- [Sentry Docs: Next.js Configuration Options](https://docs.sentry.io/platforms/javascript/guides/nextjs/configuration/options/) (Sentry公式、`tracePropagationTargets` の既定と `propagateTraceparent`) ※2026-10-25 確認
- [Sentry Docs: Next.js Distributed Tracing](https://docs.sentry.io/platforms/javascript/guides/nextjs/tracing/distributed-tracing/) (Sentry公式、`tracePropagationTargets` の設定例と CORS) ※2026-10-25 確認

> "Fewer moving parts and a short path from service to Sentry"
> ([You don't need to pick one: how Sentry and OpenTelemetry work together](https://blog.sentry.io/sentry-opentelemetry-work-together/), セクション "Direct OTLP vs Collector forwarding") ※2026-05-29に実際にfetch成功

> "It reads the active OTel trace context and stamps it onto every Sentry event"
> ([Your OTel spans, our errors: A Sentry love story in one trace](https://blog.sentry.io/otel-spans-errors-sentry-trace/), セクション "What the OtlpIntegration does under the hood") ※2026-08-05に実際にfetch成功

**バージョン**: @sentry/nextjs 10.10.0+（`propagateTraceparent`）、Sentry SDK v11 基準
**確信度**: 高
**最終更新**: 2026-10-25

---

### 6. eval に頼りきらず、AIエージェントのトレースを人間が定期的に読む

自動 eval は「既知の劣化」を検知できるが、「まだ気づいていない失敗パターン」は教えてくれない。エージェントの入出力・ツール呼び出しを安定した `conversation_id` 付きでトレースに記録し、エラー・パフォーマンス・ログと同じ場所（同一トレース基盤）に紐づけることで、モデル選定・ルーティング・プロンプト変更の判断材料を人間が直接確認できるようにする。
ただし入出力にはユーザーの個人情報や秘密値が含まれうる。全文を記録するのは社内ツールなど利用者と用途が限られる場合に絞り、顧客向けの機能では記録前にサニタイズするか入出力の記録を切る。あわせてトレース・ログの保持期間を決めておく。

**根拠**:
- eval は既知のリグレッション検知には有効だが、想定していなかった失敗モードは検出できない
- トレースがエラー・パフォーマンスデータ・ログと同じ基盤に載っていると、障害調査の導線が短くなる
- コスト/品質のトレードオフ判断は自動化しきれず、人間が実際のやり取りを読んで意思決定する必要がある
- AI SDK の telemetry は `recordInputs` / `recordOutputs` が既定で有効で、公式もプライバシーの理由で無効化できると説明している。明示的に設定し、既定のまま本番の顧客データを流さない
- Sentry v11 では genAI の入出力も `dataCollection` で収集可否を宣言するカテゴリに入る（`observability/error-tracking.md` #11）。クライアントから送るログの PII 除去は `architecture/logging.md` #2 の境界で行う

**コード例**:
```ts
// AI SDK: telemetry でエージェントの入出力とツール結果を記録
experimental_telemetry: {
  isEnabled: true,
  functionId: "conference-scheduler.schedule",
  // 入出力の記録は既定で有効。全文を残すのは社内ツールなど記録してよい範囲に限る
  recordInputs: isInternalTool,
  recordOutputs: isInternalTool,
  metadata: {
    agent: "schedule",
    model_id: config.id,
    conversation_id: context.conversationId,
  },
},
```

```ts
// Sentry: 会話開始時に conversation_id 付きでログを記録し、後から検索できるようにする
import * as Sentry from '@sentry/nextjs';
import { redactPii } from '@/lib/redact'; // メールアドレス・電話番号・トークンなどを伏せる自前のヘルパー

Sentry.logger.info("agent conversation started", {
  conversation_id: conversationId,
  // 生の発話は記録せず、サニタイズした値だけを残す
  first_message: redactPii(messages[0].content),
  user_tier: identity.tier,
  conversation_url: `https://your-app.com/admin/conversations/${conversationId}`,
});
```

**出典引用**:
> "My agent records inputs and outputs in its telemetry, so the whole exchange and every tool result land in Sentry with a stable `conversation_id` attached"
> ([Reading the agent traces is how you make the call your eval can't](https://blog.sentry.io/spot-checking-ai-agents/), セクション "Make the traces easy to find") ※2026-07-01に実際にfetch成功

> "An eval will tell you when a known thing regresses. It will not tell you what you haven't thought to check yet."
> ([Reading the agent traces is how you make the call your eval can't](https://blog.sentry.io/spot-checking-ai-agents/), セクション "Read the agent traces before you reach for the fix") ※2026-07-01に実際にfetch成功

**出典**:
- [AI SDK Docs: Telemetry](https://ai-sdk.dev/docs/ai-sdk-core/telemetry) (Vercel AI SDK公式、`recordInputs` / `recordOutputs` の既定値と無効化の理由) ※2026-10-25 確認

**バージョン**: Sentry SDK（`Sentry.logger`）, Vercel AI SDK（`experimental_telemetry`）
**確信度**: 中
**最終更新**: 2026-10-25

---

### 7. Sentry SDK v11 のトレーシングは「ライブラリ読み込み前に初期化」し、Node では `--require` ではなく `--import` を使う

Sentry JavaScript SDK v11 は、OpenTelemetry の instrumentation クラス（`require-in-the-middle` 依存）をやめ、モジュールのソースを書き換えて `tracingChannel()` で包む方式（`node:diagnostics_channel`）をデフォルトにした。
バンドル済みのサーバー（Vercel / Netlify）、Cloudflare Workers、Deno、Bun でも依存ライブラリのトレースが取れるようになる一方、**変換が効くのは hooks 登録後に読み込まれたモジュールだけ**という順序の制約がある。

**根拠**:
- v10 までの `@sentry/node/preload` / `preloadOpenTelemetry()` / `SENTRY_PRELOAD_INTEGRATIONS` は v11 で削除され（`--import` による初期化は v11 でも推奨）、「ライブラリは読み込み時に自ら instrument する」という単一の順序ルールに統一された
- `--require` で preload すると Node がローダースレッドで再実行し `Sentry.init()` が二重実行される。CommonJS アプリでも `--import` を使う
- external 指定された依存は変換されない。Sentry のビルドプラグインは既知パッケージを `external` から外し、直せないものは警告する
- 初回は `debug: true` で「実行時 hook が登録されたか・バンドラープラグインが動いたか・どのモジュールに注入されたか」を確認できる
- OTel 連携は 3 択: Sentry のみ（`tracesSampleRate`）/ `enableOpenTelemetrySetup: true`（`@opentelemetry/api` のスパンを Sentry スパン化、`@sentry/nextjs` ではデフォルト `true`）/ 自前 OTel + `Sentry.otlpIntegration()`（`tracesSampleRate` は未設定）

**コード例**:
```bash
# Good: Node は --import で hooks を先に登録（Deno は --preload）
node --import @sentry/node/import app.js
deno run --preload=@sentry/deno/import app.ts

# Bad: --require は loader thread で再実行され Sentry.init() が二重実行される
node --require ./instrument.js app.js
```

```ts
// Good: バンドラー利用時はビルド時 transform プラグインを使う
// vite.config.ts
import { sentryVitePlugin } from '@sentry/node/vite';
export default {
  plugins: [sentryVitePlugin({ org: '...', project: '...' })],
};
```

```js
// Good: 自前の OTel を使い Sentry は OTLP で連携（v11 では provider のスパンを Sentry ネイティブスパンにはできない）
const provider = new NodeTracerProvider({
  spanProcessors: [
    new BatchSpanProcessor(
      new OTLPTraceExporter(Sentry.getOtlpTracesEndpoint('__DSN__')),
    ),
  ],
});
provider.register();

Sentry.init({
  dsn: '__DSN__',
  integrations: [Sentry.otlpIntegration()],
});
```

**アンチパターン**:
- `Sentry.init()` を instrument 対象ライブラリの import より後に置く（ESM の import 巻き上げで未変換のまま読み込まれる）
- v10 の `skipOpenTelemetrySetup` を残したまま v11 に上げる（`enableOpenTelemetrySetup` に意味が反転して置き換わっている）
- OTel と Sentry の HTTP スパンが重複するからと `httpIntegration` ごと外す（リクエスト分離・セッション追跡も担うため `httpIntegration({ spans: false })` で絞る）

**出典引用**:
> "Register through --import, or keep Sentry.init() above every import of an instrumented library. --require is no longer supported."
> ([Sentry JavaScript SDK v11 adds tracing for all JS platforms](https://blog.sentry.io/tracing-for-all-js-platforms/), セクション "Pitfalls & Caveats") ※2026-10-09に実際にfetch成功（原文 HTML で逐語確認済み）

**出典**:
- [Sentry JavaScript SDK v11 adds tracing for all JS platforms](https://blog.sentry.io/tracing-for-all-js-platforms/) (Sentry公式ブログ) ※2026-10-09に実際にfetch成功

**バージョン**: Sentry JavaScript SDK 11+（Deno は 2.8.3+）
**確信度**: 高
**最終更新**: 2026-10-09

---

## 関連プラクティス

- [`observability/logging.md`](./logging.md) - リクエスト ID とログの相関
- [`observability/metrics.md`](./metrics.md) - トレースとメトリクスの連携
- [`api-client/error-handling.md`](../api-client/error-handling.md) - API エラー時のスパンへのエラー記録
