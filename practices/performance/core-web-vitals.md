# Core Web Vitals のベストプラクティス

## ルール

### 1. LCP（最大コンテンツ描画）を最適化する

LCP は 2.5秒以内を目標とする。ヒーロー画像・大きなテキストブロックが主な対象。
LCP 画像を lazy にしないこと（`next/image` の既定は `loading="lazy"` なので `loading="eager"` を付ける）、CDN の活用が主な改善手段。preload や `fetchPriority="high"` は JS の重いページでは逆効果になりうるため、画像の取得がボトルネックだと計測で確認できた場合に限って使う（下の追加根拠を参照）。画像の `loading` の使い分けは `practices/web-standards/html.md` #12 を参照。

**根拠**:
- LCP はページの主要コンテンツが表示されるタイミングを測定する
- Google の検索ランキング（Core Web Vitals）に影響する
- 遅い LCP はユーザーが離脱する主要因の1つ
- Next.js 16 で `next/image` の `priority` は非推奨になり、`preload`・`loading="eager"`・`fetchPriority` に分かれた。公式は多くの場合 `preload` ではなく `loading="eager"` か `fetchPriority="high"` を使うよう案内している

**コード例**:
```tsx
// Next.js の Image コンポーネントでヒーロー画像を最適化
import Image from 'next/image';

// Good: LCP 対象のヒーロー画像は lazy にしない
export default function HeroSection() {
  return (
    <Image
      src="/hero.jpg"
      alt="ヒーロー画像"
      width={1200}
      height={600}
      loading="eager"
      // 画像の取得が LCP のボトルネックだと計測で確認できた場合だけ付ける
      // fetchPriority="high"
      sizes="100vw"
    />
  );
}
// v15 以前も loading="eager" は使える。priority は preload と fetchpriority="high" まで付くので、
// 画像の取得がボトルネックだと確認できた場合に使う

// サーバーコンポーネントで fonts を preload
// app/layout.tsx
import { Inter } from 'next/font/google';

// Next.js Font Optimization: フォントファイルを自動的に最適化・セルフホスト
const inter = Inter({
  subsets: ['latin'],
  display: 'swap',  // 本文用は swap。font-display の選び方は practices/performance/font-loading.md #1
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja" className={inter.className}>
      <body>{children}</body>
    </html>
  );
}
```

**出典**:
- [web.dev: LCP](https://web.dev/lcp/) (web.dev)
- [Next.js Docs: Image Optimization](https://nextjs.org/docs/app/building-your-application/optimizing/images) (Next.js公式)
- [Next.js Docs: Image Component](https://nextjs.org/docs/app/api-reference/components/image) (Next.js公式、"Starting with Next.js 16, the `priority` property has been deprecated in favor of the `preload` property"、"In most cases, you should use `loading="eager"` or `fetchPriority="high"` instead of `preload`.") ※2026-10-25 確認

**バージョン**: Next.js 13+（`priority` は 16 で非推奨）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 2. CLS（累積レイアウトシフト）を防ぐ

CLS は 0.1 以下を目標とする。画像・動的コンテンツ・フォントによる
レイアウトのずれを防ぐ。

**根拠**:
- CLS はコンテンツの視覚的な安定性を測定する
- レイアウトシフトはユーザーが誤タップ・誤クリックする原因になる
- 広告・埋め込みコンテンツ・動的に挿入される要素が主な原因

**コード例**:
```tsx
// Bad: 画像のサイズ指定なし → レイアウトシフト発生
<img src="/photo.jpg" alt="写真" />

// Good: 明示的なサイズ指定でスペース確保
import Image from 'next/image';

// 固定サイズの場合
<Image src="/photo.jpg" alt="写真" width={800} height={600} />

// レスポンシブ画像（fill + aspect-ratio で CLS を防ぐ）
<div style={{ position: 'relative', aspectRatio: '16/9' }}>
  <Image
    src="/photo.jpg"
    alt="写真"
    fill
    style={{ objectFit: 'cover' }}
  />
</div>

// スケルトンUIで動的コンテンツの領域を確保
function UserCard() {
  return (
    <Suspense fallback={
      // スケルトンが実際のコンテンツと同じ高さを持つことで CLS を防ぐ
      <div className="h-24 w-full animate-pulse rounded bg-gray-200" />
    }>
      <UserCardContent />
    </Suspense>
  );
}
```

**出典**:
- [web.dev: CLS](https://web.dev/cls/) (web.dev)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. INP（インタラクションから次の描画まで）を最適化する

INP は 200ms 以内を目標とする。長いタスクを分割し、メインスレッドのブロックを防ぐ。

**根拠**:
- INP は FID の後継として 2024年3月から Core Web Vitals に組み込まれた。Vercel Speed Insights も 2026-11-01 に FID 対応を終了し、応答性指標は INP に一本化される（FID 前提の閾値・ダッシュボードは INP に移行する）
- 200ms を超えるインタラクション遅延はユーザーが「遅い」と感じる閾値
- 長時間のJS実行、不必要なレンダリング、大きなDOMが主な原因
- `startTransition` のコールバックはその場で同期的に実行されるため、`startTransition(() => setResults(items.filter(...)))` の `filter` はイベントハンドラの中で走り、入力をブロックする。後回しになるのは再レンダリングだけなので、重い計算は `useDeferredValue` で遅延させた値を使ってレンダー中に行う。1 回の計算自体が長いなら Web Worker やタスク分割に移す

**コード例**:
```tsx
// Bad: 重い処理をクリックハンドラで直接実行
function handleClick() {
  const result = processLargeDataset(hugeArray);  // メインスレッドをブロック
  setState(result);
}

// Good: useDeferredValue で、入力の反映を先に行い、重いフィルタリングは遅れて追従させる
'use client';
import { useDeferredValue, useMemo, useState } from 'react';

function SearchResults({ items }: { items: Item[] }) {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  // 入力直後の緊急レンダーでは deferredQuery が変わらないのでキャッシュが返り、入力をブロックしない
  const results = useMemo(
    () => items.filter(item =>
      item.name.toLowerCase().includes(deferredQuery.toLowerCase())
    ),
    [items, deferredQuery]
  );

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      {isStale && <span>検索中...</span>}
      <ResultList results={results} />
    </>
  );
}
```

**出典**:
- [web.dev: INP](https://web.dev/inp/) (web.dev)
- [React Docs: useTransition](https://react.dev/reference/react/useTransition) (React公式)
- [React Docs: useDeferredValue](https://react.dev/reference/react/useDeferredValue) (React公式)
- [Speed Insights deprecates First Input Delay on November 1st](https://vercel.com/changelog/speed-insights-deprecates-first-input-delay-on-november-first) (Vercel Changelog、FID 計測の終了と INP への一本化) ※2026-10-02に実際にfetch成功
  > "The deprecation won't require changes on your side, affect your score, or prevent you from viewing existing FID data."

**バージョン**: React 18+, Next.js 13+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 4. `web-vitals` ライブラリで Core Web Vitals を本番計測してアナリティクスに送信する

RUM（Real User Monitoring）として `web-vitals` ライブラリを使い、実ユーザー環境での
LCP・INP・CLS を継続的に収集する。ラボデータ（Lighthouse）だけでは現実の問題を捉えられない。
計測と送信の実装（`onCLS` / `onINP` / `onLCP` の登録、送信、App Router への組み込み）は `practices/observability/rum.md` #1、遅い原因の診断（attribution ビルド）は同 #4 を正本とする。このルールでは良否を判定する閾値だけを扱う。

**根拠**:
- Lighthouse はシミュレーション環境での計測であり、実ユーザーの環境差異を反映しない
- 閾値は `web-vitals` から `LCPThresholds` / `INPThresholds` / `CLSThresholds` としてエクスポートされており、自前で数値を持たずに済む。各配列は `[good の上限, needs-improvement の上限]` で、これを超えると poor になる
- 評価はページロード単位の平均ではなく p75 で行う（`practices/observability/rum.md` #3）

**コード例**:
```ts
import { CLSThresholds, INPThresholds, LCPThresholds } from 'web-vitals';

console.log(LCPThresholds); // [2500, 4000]（ms）
console.log(INPThresholds); // [200, 500]（ms）
console.log(CLSThresholds); // [0.1, 0.25]

// 各 Metric の rating（'good' | 'needs-improvement' | 'poor'）はこの閾値で判定されている
```

**出典**:
- [GoogleChrome/web-vitals README](https://github.com/GoogleChrome/web-vitals) (Google Chrome、"Rating Thresholds" の節) ※2026-10-25 確認
- [web.dev: Measuring performance in the field](https://web.dev/articles/vitals-measurement-getting-started) (web.dev)

**バージョン**: web-vitals 4+
**確信度**: 高
**最終更新**: 2026-10-25

---

## 関連プラクティス

- [`observability/rum.md`](../observability/rum.md) - web-vitals ライブラリで LCP/INP/CLS を本番計測しアナリティクスに送信する方法

---

#### 追加根拠 (2026-05-06) — ルール1「LCP（最大コンテンツ描画）を最適化する」

新たに以下の記事で `fetchpriority="high"` の誤用アンチパターンが実測データで示された:
- [Lighthouse のスコアを上げようとして、逆に下げてしまった 2 つの失敗](https://zenn.dev/kimsuho/articles/b678c3b084c6d1) (Zenn kimsuho / 2026) ※2026-05-06に実際にfetch成功

LCP対象ウォールペーパー画像に `<link rel="preload" as="image" fetchpriority="high">` を追加したところ、Speed Index が 1.0s → 4.6s（4.6倍増）、スコアが 99 → 83 に急悪化した。原因: JavaScriptヘビーなSPAではボトルネックはスクリプト解析・実行であり画像取得ではない。`fetchpriority="high"` によって帯域が画像に割り当てられると、クリティカルなJavaScriptのダウンロードが遅れ、コンポーネントツリーの構築が遅延し、LCP要素のDOM生成自体が遅れるという逆転現象が起きる（JS依存が少ないログインページは影響なし）。属性を削除してデフォルト（`auto`）に戻す1行の修正でスコアが99に回復した。安全な使用条件: (1) LCPボトルネックが画像ダウンロードと計測で確認済み、かつ (2) JSが最小限の静的ページ・コンテンツサイト等。

**確信度**: 既存（高）→ 高（実測データで適用条件を明確化）

---

#### 追加根拠 (2026-05-06) — ルール4「`web-vitals` ライブラリで Core Web Vitals を本番計測してアナリティクスに送信する」

新たに以下のドキュメントで同じプラクティスが公式確認された:
- [GoogleChrome/web-vitals README](https://raw.githubusercontent.com/GoogleChrome/web-vitals/main/README.md) (GoogleChrome / mainブランチ) ※2026-05-06に実際にfetch成功

公式READMEが3点を明確に説明: (1) **閾値の公式値** — LCP ≤2.5s / CLS ≤0.1 / INP ≤200ms が "good"、それぞれ `LCPThresholds`・`CLSThresholds`・`INPThresholds` としてライブラリからエクスポート済み。(2) **Attribution ビルド** — `web-vitals/attribution` からインポートすると約1.5KB（brotli）追加で `attribution` プロパティが付与される。CLS なら `largestShiftTarget`、INP なら `interactionTarget`・`inputDelay`・`processingDuration`・`presentationDelay`、LCP なら `target` を取得できる。(3) **バッチ送信パターン** — 指標をSetにキューイングし `document.visibilityState === 'hidden'` 時に `navigator.sendBeacon()` で一括送信することで、ページ離脱時の送信漏れを防ぎつつリクエスト数を最小化する。
(2) の実装は `practices/observability/rum.md` #4、(3) の実装は同 #1 を参照。

**確信度**: 既存（高）→ 高（公式READMEで全パターン実証済み）

---

#### 追加根拠 (2026-05-15) — ルール4「`web-vitals` ライブラリで Core Web Vitals を本番計測してアナリティクスに送信する」

新たに以下の記事で Lab データと Field データの使い分け、および RUM データ拡充の重要性が実践的に示された:
- [DX支援エンジニアのための『サイト計測 × 表示速度改善』ガイド](https://zenn.dev/bizlink/articles/52de291b484008) (Zenn / 2026-05-14) ※2026-05-15に実際にfetch成功

**出典引用**:
> "Lab data enables root cause identification; Field data drives business decisions and SEO evaluation"
> ([DX支援エンジニアのための『サイト計測 × 表示速度改善』ガイド], セクション "Lab vs. Field データ")

既存ルール4（`web-vitals` ライブラリ）を補完する2点:

1. **Lab/Field データの目的分離**: Lighthouse（ラボ）は原因特定に使い、CrUX/RUM（フィールド）は SEO 評価とビジネス意思決定に使う。「Lighthouse 100点 ≠ 実ユーザーが良好」であることを認識する。Google が検索ランキングに使うのは実ユーザー計測の Field データ（75パーセンタイル）。

2. **RUM をデバイス種別とルートでセグメント化**: 送信データにデバイス種別とルートを含めることで「モバイルの特定ページで INP が悪化している」といった問題を特定できる。送信の実装は `practices/observability/rum.md` #1 を参照。閾値の80%（LCP 2.0s、INP 160ms、CLS 0.08）でアラートを設定し、悪化を早期検知する。

**確信度**: 既存（高）→ 高（Lab/Field 分離の実践的観点を追加）

---

#### 追加根拠 (2026-05-06) — ルール1「LCP（最大コンテンツ描画）を最適化する」（2回目）

新たに以下の記事でサードパーティスクリプトをユーザーインタラクションまで遅延させる手法が実測データで示された:
- [How I Engineered a 0.6s LCP and a Perfect 100/100 GTmetrix Score: Next.js Optimization Guide](https://dev.to/dattasble/how-i-engineered-a-06s-lcp-and-a-perfect-100100-gtmetrix-score-nextjs-optimization-guide-1j24) (dev.to dattasble / 2026-05) ※2026-05-06に実際にfetch成功

Google Analytics・チャットウィジェット・A/Bテストスクリプトなどサードパーティスクリプトの一括遅延により、LCP 0.6s・GTmetrix 100/100 を達成した事例。`mousedown`, `scroll`, `touchstart`, `keydown` のいずれかが発火するまでスクリプトを読み込まないことで TBT（Total Blocking Time）が 0ms になる。Next.js での推奨実装:

```tsx
// app/_components/DeferredScripts.tsx
'use client';
import { useEffect } from 'react';

declare global {
  interface Window {
    dataLayer: unknown[];
    gtag: (...args: unknown[]) => void;
  }
}

export function DeferredScripts() {
  useEffect(() => {
    let loaded = false;

    function loadScripts() {
      if (loaded) return;
      loaded = true;

      // Google Analytics: gtag/js を読み込む前に dataLayer と gtag を初期化する（これがないと計測されない）
      window.dataLayer = window.dataLayer || [];
      // gtag.js は arguments オブジェクトが積まれる前提なので、配列ではなく arguments を push する
      window.gtag = function gtag() {
        window.dataLayer.push(arguments);
      };
      window.gtag('js', new Date());
      window.gtag('config', process.env.NEXT_PUBLIC_GA_ID);

      const gaScript = document.createElement('script');
      gaScript.src = `https://www.googletagmanager.com/gtag/js?id=${process.env.NEXT_PUBLIC_GA_ID}`;
      gaScript.async = true;
      document.head.appendChild(gaScript);
    }

    const events = ['mousedown', 'scroll', 'touchstart', 'keydown'] as const;
    events.forEach(e => window.addEventListener(e, loadScripts, { once: true }));

    return () => {
      events.forEach(e => window.removeEventListener(e, loadScripts));
    };
  }, []);

  return null;
}
```

注意点: チャットウィジェットなど「ページロード直後から表示が必要」なスクリプトには適用不可。また、イベント発火前にコンバージョンが発生した場合（直帰ユーザー）はアナリティクスが計測されない。適用可否は計測要件と相談して判断すること。

**確信度**: 既存（高）→ 高（実測 LCP 0.6s・GTmetrix 100点で実証済み）
