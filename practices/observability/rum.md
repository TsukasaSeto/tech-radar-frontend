# Real User Monitoring（RUM）のベストプラクティス

## ルール

### 1. web-vitals ライブラリで LCP・INP・CLS を本番環境で計測する

`web-vitals` ライブラリを使い、実際のユーザー環境での Core Web Vitals を
継続的に計測してアナリティクスエンドポイントに送信する。
Next.js App Router では組み込みの `useReportWebVitals`（`next/web-vitals`）を `'use client'` の専用コンポーネントで呼び、
root layout は Server Component のまま、そのコンポーネントを置くだけにする。
RUM の計測と送信はこのルールを正本とする（閾値は `performance/core-web-vitals.md` を参照）。

**根拠**:
- Lab データ（Lighthouse 等）は原因の特定に使い、実ユーザーの体験と SEO 評価は Field データ（RUM / CrUX）で判断する
- `web-vitals` は Chrome が Core Web Vitals の計測に使うものと同じロジックを使用する
- `rating` フィールドで「good / needs-improvement / poor」の三段階で評価できる
- `navigator.sendBeacon` はページ離脱時にも送信をキューに積める。ただしキューに積めなかったときは `false` を返す（キューに積めるデータは合計 64KiB まで）ので、戻り値を見て `fetch` の `keepalive` に切り替える
- 指標ごとに送らず、Set などに溜めて `visibilitychange` で `document.visibilityState === 'hidden'` になったときにまとめて送ると、離脱時の送信漏れを防ぎつつリクエスト数を減らせる（web-vitals README のバッチ送信パターン）
- `useReportWebVitals` に渡す関数の参照が変わると、それまでの指標が再度渡されて重複送信になる。モジュールのトップレベルで定義した関数を渡す
- 送信データにデバイス種別などのセグメント情報を含めると「モバイルの特定ルートで INP が悪化している」といった絞り込みができる。ページはルートテンプレートに正規化してから集計する（#3）
- 遅い原因の診断情報（attribution ビルド）は #4、JS ブロッキングの特定は #5 を参照

**コード例**:
```ts
// lib/web-vitals.ts
import { onCLS, onFCP, onINP, onLCP, onTTFB } from 'web-vitals';

// web-vitals と next/web-vitals のどちらの Metric も渡せるよう、送信に使う項目だけを受け取る
export type ReportableMetric = {
  name: string;
  value: number;
  rating: 'good' | 'needs-improvement' | 'poor';
  delta: number;
  id: string;
  navigationType?: string;
};

export function sendToAnalytics(metric: ReportableMetric) {
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

  // sendBeacon はキューに積めなかったとき false を返すので、そのときは fetch（keepalive）で送る
  if (navigator.sendBeacon?.('/api/vitals', body)) return;

  fetch('/api/vitals', {
    method: 'POST',
    body,
    headers: { 'Content-Type': 'application/json' },
    keepalive: true,
  });
}

// Next.js 以外（Vite などの SPA）では、ページ読み込みごとに 1 回だけ呼ぶ
export function measureWebVitals() {
  onCLS(sendToAnalytics);   // Cumulative Layout Shift
  onFCP(sendToAnalytics);   // First Contentful Paint
  onINP(sendToAnalytics);   // Interaction to Next Paint
  onLCP(sendToAnalytics);   // Largest Contentful Paint
  onTTFB(sendToAnalytics);  // Time to First Byte
}
```

```tsx
// app/_components/web-vitals.tsx — Next.js App Router
'use client';

import { useReportWebVitals } from 'next/web-vitals';
import { sendToAnalytics } from '@/lib/web-vitals';

export function WebVitals() {
  // 参照が変わらないモジュールトップレベルの関数を渡す（重複送信を防ぐ）
  useReportWebVitals(sendToAnalytics);
  return null;
}
```

```tsx
// app/layout.tsx — Server Component のまま、クライアント境界は WebVitals だけにする
import type { ReactNode } from 'react';
import { WebVitals } from './_components/web-vitals';

export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html lang="ja">
      <body>
        <WebVitals />
        {children}
      </body>
    </html>
  );
}
```

**出典**:
- [web-vitals GitHub](https://github.com/GoogleChrome/web-vitals) (GoogleChrome / GitHub) ※2026-10-25 確認（バッチ送信パターン）
- [web.dev: User-centric performance metrics](https://web.dev/articles/user-centric-performance-metrics) (web.dev)
- [Next.js Docs: useReportWebVitals](https://nextjs.org/docs/app/api-reference/functions/use-report-web-vitals) (Next.js公式) ※2026-10-25 確認
- [MDN: Navigator.sendBeacon()](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/sendBeacon) (MDN) ※2026-10-25 確認

**バージョン**: web-vitals 4+, Next.js 16（App Router の `useReportWebVitals`）
**確信度**: 高
**最終更新**: 2026-10-25

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
// lib/web-vitals-sampling.ts
import { onCLS, onINP, onLCP } from 'web-vitals';
import type { Metric } from 'web-vitals';
import { datadogRum } from '@datadog/browser-rum';
import { sendToAnalytics } from '@/lib/web-vitals'; // #1 の送信関数

// サンプリングレート: 本番 10%、ステージング 100%
const SAMPLE_RATE = process.env.NODE_ENV === 'production' ? 0.1 : 1.0;

function shouldSample(): boolean {
  return Math.random() < SAMPLE_RATE;
}

// ページ単位でサンプリング判定（セッション開始時に1度だけ決定）
const isSampled = shouldSample();

function sendToAnalyticsIfSampled(metric: Metric) {
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
**最終更新**: 2026-10-25

---

### 3. Web Vitals のデータをパーセンタイル（p75）で評価する

Web Vitals の評価は平均値ではなく 75 パーセンタイル（p75）で行う。
Google が Core Web Vitals を判定する基準もp75。

**根拠**:
- 平均値は外れ値（極端に遅いケース）に引っ張られ、実態を反映しない場合がある
- Google の CrUX（Chrome UX Report）と PageSpeed Insights は p75 を基準に評価する
- p75 の改善が検索ランキングへの影響に直結する
- 「good」の割合をKPIとするのも有効（全セッションの何%が good 以上か）
- ページのタグは `/products/123` のような生のパスではなく `/products/[id]` のようなルートテンプレートに正規化する。生のパスはタグの種類が際限なく増え、集計コストと p75 の母数の両方が崩れる

**コード例**:
```ts
// app/api/vitals/route.ts - Web Vitals 収集 API
import { NextRequest } from 'next/server';
import { sendMetricToDatadog } from '@/lib/datadog'; // Datadog への送信ラッパー（各自の実装）

interface VitalReport {
  name: string;
  value: number;
  rating: 'good' | 'needs-improvement' | 'poor';
  id: string;
  page: string;
  version?: string;
}

// 数値や UUID のセグメントを [id] に置き換え、ルート単位に集約する（生のパスはタグのカーディナリティが爆発する）
function toRouteTemplate(pathname: string): string {
  return pathname
    .split('/')
    .map((segment) =>
      /^\d+$/.test(segment) || /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i.test(segment)
        ? '[id]'
        : segment,
    )
    .join('/');
}

export async function POST(request: NextRequest) {
  const vital = await request.json() as VitalReport;

  // Datadog カスタムメトリクスとして送信（p75 計算はDatadog側で実施）
  await sendMetricToDatadog({
    metric: `web_vitals.${vital.name.toLowerCase()}`,
    value: vital.value,
    tags: [
      `rating:${vital.rating}`,
      `page:${toRouteTemplate(vital.page)}`,
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
**最終更新**: 2026-10-25

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
- Chrome 123+ でサポートされる。`PerformanceObserver` の `durationThreshold` は Event Timing（`PerformanceEventTiming`）用のオプションで LoAF には効かないため、報告対象はコード例のように `duration` で間引く

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
- [MDN: PerformanceObserver.observe()](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceObserver/observe) (MDN、`durationThreshold` の適用対象) ※2026-10-25 確認

**バージョン**: Chrome 123+, web-vitals 4+（attribution ビルド）
**確信度**: 中
**最終更新**: 2026-10-25

---

### 6. SPA のルート遷移（soft navigation）の Web Vitals は、ブラウザが報告する遷移単位の LCP・CLS・INP で計測し、ページロードと分けて評価する

ページロード単位の Web Vitals は「ユーザーが着地したページ」しか測れない。SPA ではその後の遷移（soft navigation）が閲覧の大半を占めるため、遷移ごとの LCP・CLS・INP を計測し、Sentry v11 などの計測ツールでは遷移種別（ページロード / soft navigation / bfcache 復元）で分けて評価する。

**根拠**:
- Chrome は soft navigation を「ユーザー操作で開始され、URL が目に見えて変わり、画面が描画される遷移」と定義している。この判定ヒューリスティクスは確定版ではなく、ブラウザごとに異なりうる（Sentry）
- 遷移スパンの終了は「アプリがアイドルになった時点」で、ユーザーが新ルートを見られた時点とは一致しない。Sentry の例では、LCP は 0.65s、遷移スパンは 2.85s で終わっており、ルート単位の体感はスパン長では測れない
- ページロードの CLS / INP は「ページを開いている間ずっと」を対象にするため、SPA では後から遷移した先のレイアウトシフトが着地ページのスコアに混ざる。soft navigation の Web Vitals を有効にすると、ページロードの Vitals は最初の soft navigation で確定し、以降は遷移先ルートに帰属する（Sentry）
- 設定ページや決済の 2 ステップ目のように、クリックでしか到達しないページは、ページロード単位の Vitals にはほぼ載らない。Sentry の自社アプリでは、ページビューの約 62% が soft navigation（Sentry）
- soft navigation は文書リクエストがなく JS が既に実行中のため、ページロードより速く出やすい。混在させると全体スコアが見かけ上改善する（Sentry の例: 全遷移の p75 LCP 2.48s に対し soft navigation のみで 1.17s）。両者は比較せず、ページロード用の good / poor 閾値を soft navigation に当てはめない
- 既存のダッシュボード・アラートは遷移種別（Sentry では `browser.navigation.type` 属性。soft navigation は `soft-navigation`、bfcache 復元は `back-forward-cache`）で絞り込む。ページロードのアラートを維持するには、この 2 値を除外する
- 制約: Chromium 151 以降のみが soft navigation を報告する（記事執筆時点）。結果はその割合のユーザーを表し、ブラウザの判定はルーターの遷移と一致しない（Sentry の自社アプリでは Chrome のルート変更の約 4 分の 3 が該当）

**コード例**:
```ts
// Good: Sentry JavaScript SDK v11。browserTracingIntegration で soft navigation の LCP / CLS / INP が既定で有効
import * as Sentry from '@sentry/browser';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  tracesSampleRate: 1.0, // 本番では Rule #2 のサンプリング戦略に従う
  integrations: [Sentry.browserTracingIntegration()],
});

// ページ全体の寿命で 1 組の Vitals に戻したい場合のみ、soft navigation を無効化する
// integrations: [Sentry.browserTracingIntegration(), Sentry.webVitalsIntegration({ softNavigations: false })]

// Bad: soft navigation を含む全遷移を 1 つの p75 にまとめて、ページロード用の閾値で評価する
// （遷移種別で絞り込まないダッシュボード / アラート）
```

**アンチパターン**:
- SPA のルート遷移の速さを「遷移スパンの長さ」だけで判断する（アプリがアイドルになった時点で終わるため、ユーザーが見えた時点とずれる）
- v11 へ上げた後に全体スコアが改善した理由を調べず、成果として報告する（遷移種別の構成が変わっただけの可能性がある）

**出典引用**:
> "Chrome now reports web vitals for soft navigations, and JavaScript SDKs v11 capture Largest Contentful Paint (LCP), Cumulative Layout Shift (CLS), and Interaction to Next Paint (INP) for each one."
> ([Measuring soft navigation web vitals](https://blog.sentry.io/measuring-soft-navigation-web-vitals/), Sentry Blog、セクション "TL;DR") ※2026-10-29に実際にfetch成功

> "Soft navigations are Chromium-only for now, and the heuristics for what counts as one may still change."
> ([Measuring soft navigation web vitals](https://blog.sentry.io/measuring-soft-navigation-web-vitals/), Sentry Blog、セクション "Measuring the whole journey") ※2026-10-29に実際にfetch成功

**出典**:
- [Measuring soft navigation web vitals](https://blog.sentry.io/measuring-soft-navigation-web-vitals/) (Sentry Blog、Sentry SDK v11 の soft navigation Web Vitals) ※2026-10-29に実際にfetch成功

**バージョン**: Sentry JavaScript SDK v11+（`browserTracingIntegration`）、soft navigation の報告は Chromium 151+
**確信度**: 高（Sentry SDK の挙動は提供元である Sentry の公式ブログ。soft navigation のブラウザ仕様は発展途上のため変更の可能性あり）
**最終更新**: 2026-10-29

---

## 関連プラクティス

- [`performance/core-web-vitals.md`](../performance/core-web-vitals.md) - LCP・INP・CLS の最適化手法
- [`observability/metrics.md`](./metrics.md) - カスタムメトリクスとのデータ統合
