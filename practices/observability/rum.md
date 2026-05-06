# Real User Monitoring（RUM）のベストプラクティス

## ルール

### 1. web-vitals ライブラリで LCP・INP・CLS を本番環境で計測する

`web-vitals` ライブラリを使い、実際のユーザー環境での Core Web Vitals を
継続的に計測してアナリティクスエンドポイントに送信する。

**根拠**:
- Lab データ（Lighthouse 等）は実際のユーザー体験を反映しない場合がある
- `web-vitals` は Chrome が Core Web Vitals の計測に使うものと同じロジックを使用する
- `rating` フィールドで「good / needs-improvement / poor」の三段階で評価できる
- `navigator.sendBeacon` を使うことでページ離脱時にもデータが確実に送信される

**コード例**:
```ts
// lib/web-vitals.ts
import { onCLS, onFCP, onINP, onLCP, onTTFB } from 'web-vitals';
import type { Metric } from 'web-vitals';

function sendToAnalytics(metric: Metric) {
  const body = JSON.stringify({
    name: metric.name,
    value: metric.value,         // 実測値（ms または スコア）
    rating: metric.rating,       // 'good' | 'needs-improvement' | 'poor'
    delta: metric.delta,         // 前回レポートからの変化量
    id: metric.id,               // このメトリクスの一意ID（重複排除に使用）
    navigationType: metric.navigationType,
    // ページ情報
    page: window.location.pathname,
    // ビルドバージョン（リグレッション検知に使用）
    version: process.env.NEXT_PUBLIC_APP_VERSION,
  });

  // sendBeacon: ページ離脱時でも確実に送信される
  if (navigator.sendBeacon) {
    navigator.sendBeacon('/api/vitals', body);
  } else {
    // フォールバック（keep-alive で接続を維持）
    fetch('/api/vitals', {
      method: 'POST',
      body,
      headers: { 'Content-Type': 'application/json' },
      keepalive: true,
    });
  }
}

// 全ての Core Web Vitals を計測
export function measureWebVitals() {
  onCLS(sendToAnalytics);   // Cumulative Layout Shift
  onFCP(sendToAnalytics);   // First Contentful Paint
  onINP(sendToAnalytics);   // Interaction to Next Paint
  onLCP(sendToAnalytics);   // Largest Contentful Paint
  onTTFB(sendToAnalytics);  // Time to First Byte
}
```

```tsx
// app/layout.tsx での初期化
'use client';
import { useEffect } from 'react';
import { measureWebVitals } from '@/lib/web-vitals';

export function WebVitalsReporter() {
  useEffect(() => {
    measureWebVitals();
  }, []);
  return null;
}

// または Next.js の組み込み計測（シンプルな場合）
// app/layout.tsx
export { reportWebVitals } from 'next/dist/pages/_app';
```

**出典**:
- [web-vitals GitHub](https://github.com/GoogleChrome/web-vitals) (GoogleChrome / GitHub)
- [web.dev: User-centric performance metrics](https://web.dev/articles/user-centric-performance-metrics) (web.dev)

**バージョン**: web-vitals 4+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. サンプリング戦略でデータ量とコストを管理する

全ユーザーのデータを送信するのではなく、サンプリングレートを設定して
統計的に有効なデータを収集しながらコストを抑える。

**根拠**:
- 高トラフィックサービスで全セッションを計測するとコストが急増する
- Web Vitals のサンプリングはランダムサンプリングでも統計的に有効な結果が得られる
- Sentry や Datadog RUM のサンプリング設定と組み合わせることで全体のコストを管理できる
- 特定のページや操作は 100% 計測し、その他を低サンプリングにすることで重要データを逃さない

**コード例**:
```ts
// lib/web-vitals.ts
import { onCLS, onINP, onLCP } from 'web-vitals';

// サンプリングレート: 本番 10%、ステージング 100%
const SAMPLE_RATE = process.env.NODE_ENV === 'production' ? 0.1 : 1.0;

function shouldSample(): boolean {
  return Math.random() < SAMPLE_RATE;
}

// ページ単位でサンプリング判定（セッション開始時に1度だけ決定）
const isSampled = shouldSample();

function sendToAnalyticsIfSampled(metric: Metric) {
  if (!isSampled) return;

  // critical ページは常に計測
  const criticalPaths = ['/checkout', '/payment', '/login'];
  const isCriticalPage = criticalPaths.some(path =>
    window.location.pathname.startsWith(path),
  );

  if (!isSampled && !isCriticalPage) return;

  sendToAnalytics(metric);
}

onLCP(sendToAnalyticsIfSampled);
onINP(sendToAnalyticsIfSampled);
onCLS(sendToAnalyticsIfSampled);

// Datadog RUM のサンプリング設定
import { datadogRum } from '@datadog/browser-rum';

datadogRum.init({
  applicationId: process.env.NEXT_PUBLIC_DD_APPLICATION_ID!,
  clientToken: process.env.NEXT_PUBLIC_DD_CLIENT_TOKEN!,
  site: 'datadoghq.com',
  service: 'tech-radar-frontend',
  sessionSampleRate: 10,         // 10% のセッションを計測
  sessionReplaySampleRate: 5,    // 5% のセッションをリプレイ記録
  trackUserInteractions: true,
  trackResources: true,
  trackLongTasks: true,
  defaultPrivacyLevel: 'mask-user-input',  // フォーム入力をマスク
});
```

**出典**:
- [web.dev: RUM and Synthetic Monitoring](https://web.dev/articles/rum-and-synthetic-monitoring) (web.dev)
- [Datadog Browser RUM](https://docs.datadoghq.com/real_user_monitoring/browser/) (Datadog公式)

**バージョン**: web-vitals 4+, @datadog/browser-rum 5+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. Web Vitals のデータをパーセンタイル（p75）で評価する

Web Vitals の評価は平均値ではなく 75 パーセンタイル（p75）で行う。
Google が Core Web Vitals を判定する基準もp75。

**根拠**:
- 平均値は外れ値（極端に遅いケース）に引っ張られ、実態を反映しない場合がある
- Google の CrUX（Chrome UX Report）と PageSpeed Insights は p75 を基準に評価する
- p75 の改善が検索ランキングへの影響に直結する
- 「good」の割合をKPIとするのも有効（全セッションの何%が good 以上か）

**コード例**:
```ts
// app/api/vitals/route.ts - Web Vitals 収集 API
import { NextRequest } from 'next/server';

interface VitalReport {
  name: string;
  value: number;
  rating: 'good' | 'needs-improvement' | 'poor';
  id: string;
  page: string;
  version?: string;
}

export async function POST(request: NextRequest) {
  const vital = await request.json() as VitalReport;

  // Datadog カスタムメトリクスとして送信（p75 計算はDatadog側で実施）
  await sendMetricToDatadog({
    metric: `web_vitals.${vital.name.toLowerCase()}`,
    value: vital.vital.value,
    tags: [
      `rating:${vital.rating}`,
      `page:${vital.page}`,
      `version:${vital.version ?? 'unknown'}`,
    ],
    type: 'distribution',  // distribution 型でパーセンタイルを計算可能
  });

  return new Response(null, { status: 204 });
}

// Grafana クエリ例（参考）
// histogram_quantile(0.75, rate(web_vitals_lcp_bucket[5m]))
```

**出典**:
- [web.dev: Defining the Core Web Vitals metrics thresholds](https://web.dev/articles/defining-core-web-vitals-thresholds) (web.dev)
- [Google CrUX: Field Data Methodology](https://developer.chrome.com/docs/crux/methodology/) (Chrome Developers)

**バージョン**: web-vitals 4+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. INP（Interaction to Next Paint）を重点的に計測し改善する

2024年3月に FID を置き換えた INP を優先的に計測・改善する。
200ms 超のインタラクションを特定して最適化する。

**根拠**:
- INP は Core Web Vitals の最新指標であり、ユーザーインタラクションへの応答性を測定する
- 200ms 以下が「good」、500ms 超が「poor」と評価される
- 長いタスク・大きな DOM・不必要なレンダリングが主な原因
- `web-vitals` の `onINP` は`attribution`ビルドで詳細な原因分析ができる

**コード例**:
```ts
// lib/web-vitals.ts - attribution ビルドで INP の詳細を取得
import { onINP } from 'web-vitals/attribution';

onINP(({ name, value, rating, attribution }) => {
  // INP のインタラクション詳細を収集
  const { interactionTarget, interactionType, loadState } = attribution;

  sendToAnalytics({
    name,
    value,
    rating,
    // 遅い操作の原因を特定するための情報
    target: interactionTarget,           // どの要素が遅いか
    interactionType,                     // 'pointer' | 'keyboard'
    loadState,                           // ページロード状態
    processedEventEntries: attribution.processedEventEntries,
  });

  // 200ms 超のインタラクションを警告
  if (value > 200) {
    console.warn(`Slow INP detected: ${value}ms on ${interactionTarget}`);
  }
});
```

**出典**:
- [web.dev: INP](https://web.dev/articles/inp) (web.dev)
- [web-vitals: Attribution](https://github.com/GoogleChrome/web-vitals#attribution-build) (GoogleChrome / GitHub)

**バージョン**: web-vitals 4+（attribution ビルド）
**確信度**: 高
**最終更新**: 2026-05-05

---

### 5. Long Animation Frames API でJSブロッキングの根本原因を特定する

`PerformanceObserver` で `long-animation-frame` エントリを監視し、
50ms 超の長いアニメーションフレームを引き起こすスクリプトを特定して報告する。

**根拠**:
- Long Animation Frames（LoAF）API は Long Tasks API の後継で、レンダリングまで含めたブロッキングを計測できる
- `web-vitals` attribution ビルドの `longAnimationFrameEntries` で INP との相関を確認できる
- LoAF エントリには `scripts` 配列が含まれ、どのスクリプトがブロッキングしたかのソースURLと実行時間を提供する
- Chrome 123+ でサポートされ、`durationThreshold` を `50` に設定するとフレームドロップ相当のブロッキングを捕捉できる

**コード例**:
```ts
// Good
// lib/loaf-observer.ts - Long Animation Frames の監視
export function observeLongAnimationFrames() {
  if (!('PerformanceLongAnimationFrameTiming' in window)) {
    return; // 未サポートブラウザはスキップ
  }

  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      const loaf = entry as PerformanceLongAnimationFrameTiming;

      // 100ms 超のフレームのみ報告（ノイズ削減）
      if (loaf.duration < 100) continue;

      const scripts = loaf.scripts.map((s) => ({
        sourceURL: s.sourceURL,
        invokerType: s.invokerType,   // 'event-listener' | 'user-callback' | 'resolve-promise' etc.
        duration: s.duration,
        executionStart: s.executionStart,
      }));

      sendToAnalytics({
        type: 'long-animation-frame',
        duration: loaf.duration,
        blockingDuration: loaf.blockingDuration,
        scripts,
        renderStart: loaf.renderStart,
        styleAndLayoutStart: loaf.styleAndLayoutStart,
      });
    }
  });

  observer.observe({ type: 'long-animation-frame', buffered: true });
  return () => observer.disconnect();
}

// Bad
// Long Tasks API のみ使う（レンダリング時間を含まないため原因特定が困難）
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    // Long Tasks はスクリプトのソース情報を持たない
    console.warn('Long task detected', entry.duration);
  }
});
observer.observe({ type: 'longtask' }); // LoAF に移行することを推奨
```

**出典**:  
- [Long Animation Frames API](https://developer.chrome.com/docs/web-platform/long-animation-frames) (Chrome Developers / 2024)
- [web-vitals: longAnimationFrameEntries](https://github.com/GoogleChrome/web-vitals/blob/main/README.md#attribution) (GoogleChrome / GitHub)

**バージョン**: Chrome 123+, web-vitals 4+（attribution ビルド）
**確信度**: 中
**最終更新**: 2026-05-06

---

## 関連プラクティス

- [`performance/core-web-vitals.md`](../performance/core-web-vitals.md) - LCP・INP・CLS の最適化手法
- [`observability/metrics.md`](./metrics.md) - カスタムメトリクスとのデータ統合
