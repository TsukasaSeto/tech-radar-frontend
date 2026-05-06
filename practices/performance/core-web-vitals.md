# Core Web Vitals のベストプラクティス

## ルール

### 1. LCP（最大コンテンツ描画）を最適化する

LCP は 2.5秒以内を目標とする。ヒーロー画像・大きなテキストブロックが主な対象。
画像の `priority` 属性、preload、CDN の活用が主な改善手段。

**根拠**:
- LCP はページの主要コンテンツが表示されるタイミングを測定する
- Google の検索ランキング（Core Web Vitals）に影響する
- 遅い LCP はユーザーが離脱する主要因の1つ

**コード例**:
```tsx
// Next.js の Image コンポーネントでヒーロー画像を最適化
import Image from 'next/image';

// Good: ファーストビューの画像に priority を設定（preload される）
export default function HeroSection() {
  return (
    <Image
      src="/hero.jpg"
      alt="ヒーロー画像"
      width={1200}
      height={600}
      priority  // LCP の対象画像には必須
      sizes="100vw"
    />
  );
}

// サーバーコンポーネントで fonts を preload
// app/layout.tsx
import { Inter } from 'next/font/google';

// Next.js Font Optimization: フォントファイルを自動的に最適化・セルフホスト
const inter = Inter({
  subsets: ['latin'],
  display: 'swap',  // フォント読み込み中はフォールバックフォントを表示
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

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

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
- INP は FID の後継として 2024年3月から Core Web Vitals に組み込まれた
- 200ms を超えるインタラクション遅延はユーザーが「遅い」と感じる閘値
- 長時間のJS実行、不必要なレンダリング、大きなDOMが主な原因

**コード例**:
```tsx
// Bad: 重い処理をクリックハンドラで直接実行
function handleClick() {
  const result = processLargeDataset(hugeArray);  // メインスレッドをブロック
  setState(result);
}

// Good: useTransition で重い更新を優先度を下げて処理
'use client';
import { useTransition, useState } from 'react';

function SearchResults({ items }: { items: Item[] }) {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState(items);
  const [isPending, startTransition] = useTransition();

  function handleSearch(value: string) {
    setQuery(value);  // 即座に更新（入力の表示）

    // 重いフィルタリングは優先度を下げる（ユーザー入力をブロックしない）
    startTransition(() => {
      setResults(items.filter(item =>
        item.name.toLowerCase().includes(value.toLowerCase())
      ));
    });
  }

  return (
    <>
      <input value={query} onChange={e => handleSearch(e.target.value)} />
      {isPending && <span>検索中...</span>}
      <ResultList results={results} />
    </>
  );
}
```

**出典**:
- [web.dev: INP](https://web.dev/inp/) (web.dev)
- [React Docs: useTransition](https://react.dev/reference/react/useTransition) (React公式)

**バージョン**: React 18+, Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. `web-vitals` ライブラリで Core Web Vitals を本番計測してアナリティクスに送信する

RUM（Real User Monitoring）として `web-vitals` ライブラリを使い、実ユーザー環境での
LCP・INP・CLS を継続的に収集する。ラボデータ（Lighthouse）だけでは現実の問題を捉えられない。

**根拠**:
- Lighthouse はシミュレーション環境での計測であり、実ユーザーの環境差異を反映しない
- `web-vitals` ライブラリは PerformanceObserver API を抽象化し、主要3指標を正確に計測する
- `navigator.sendBeacon` でページ離脱時も確実にデータ送信できる
- attribution ビルドを使うと「どの要素が問題か」の診断情報も取得できる

**コード例**:
```tsx
// Good: web-vitals ライブラリで全指標を計測してアナリティクスに送信
import { onCLS, onINP, onLCP } from 'web-vitals';

type Metric = { name: string; value: number; rating: string; id: string; delta: number };

function sendToAnalytics(metric: Metric) {
  // navigator.sendBeacon でページ離脱時も送信が保証される
  const body = JSON.stringify({
    metricName: metric.name,
    value: metric.value,
    rating: metric.rating,  // 'good' | 'needs-improvement' | 'poor'
    id: metric.id,
    delta: metric.delta,    // 前回計測からの差分
  });
  navigator.sendBeacon('/api/vitals', body);
}

// 各指標を登録
onCLS(sendToAnalytics);
onINP(sendToAnalytics);
onLCP(sendToAnalytics);

// attribution ビルドで「なぜ遅いか」の診断情報も取得
import { onINP } from 'web-vitals/attribution';

onINP((metric) => {
  const { interactionTarget, inputDelay, processingDuration, presentationDelay } =
    metric.attribution;
  console.log('遅いインタラクション要素:', interactionTarget);
  console.log('入力遅延:', inputDelay, 'ms');
  console.log('処理時間:', processingDuration, 'ms');
});

// Next.js App Router での組み込み（app/layout.tsx または専用コンポーネント）
// 'use client' コンポーネント内で useEffect で呼び出す
'use client';
import { useEffect } from 'react';

export function WebVitalsReporter() {
  useEffect(() => {
    import('web-vitals').then(({ onCLS, onINP, onLCP }) => {
      onCLS(sendToAnalytics);
      onINP(sendToAnalytics);
      onLCP(sendToAnalytics);
    });
  }, []);
  return null;
}
```

**出典**:
- [GoogleChrome/web-vitals README](https://github.com/GoogleChrome/web-vitals) (Google Chrome / 2024)
- [web.dev: Measuring performance in the field](https://web.dev/articles/vitals-measurement-getting-started) (web.dev)

**バージョン**: web-vitals 4+, React 18+
**確信度**: 高
**最終更新**: 2026-05-06

---

## 関連プラクティス

- [`observability/rum.md`](../observability/rum.md) - web-vitals ライブラリで LCP/INP/CLS を本番計測しアナリティクスに送信する方法
