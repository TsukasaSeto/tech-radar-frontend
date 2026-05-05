# バンドルサイズ最適化のベストプラクティス

## ルール

### 1. `next/dynamic` で非クリティカルなコンポーネントを遅延読み込みする

ファーストビューに不要なコンポーネント（モーダル・リッチテキストエディタ・チャートなど）は
`dynamic()` で遅延読み込みする。

**根拠**:
- 初期バンドルサイズが減少し、TTI（Time to Interactive）が改善する
- SSRが不要なコンポーネントは `{ ssr: false }` でサーバーサイド実行を省略できる
- モーダルなど条件付き表示要素は表示時に読み込めば十分

**コード例**:
```tsx
import dynamic from 'next/dynamic';

// ヘビーなコンポーネントを遅延読み込み
const RichTextEditor = dynamic(
  () => import('@/components/RichTextEditor'),
  {
    loading: () => <EditorSkeleton />,  // 読み込み中の表示
    ssr: false,  // サーバーサイドレンダリングをスキップ（ブラウザAPI依存のため）
  }
);

// モーダルは開いた時に読み込む
const ConfirmDialog = dynamic(() => import('@/components/ConfirmDialog'));

export default function BlogEditor() {
  const [isDialogOpen, setIsDialogOpen] = useState(false);

  return (
    <div>
      <RichTextEditor />  {/* 表示時に遅延読み込み */}

      {isDialogOpen && <ConfirmDialog onClose={() => setIsDialogOpen(false)} />}
    </div>
  );
}

// チャートライブラリなど大きな依存を持つコンポーネント
const DataChart = dynamic(
  () => import('@/components/DataChart'),  // chart.js などを含む
  { ssr: false }
);
```

**出典**:
- [Next.js Docs: Lazy Loading](https://nextjs.org/docs/app/building-your-application/optimizing/lazy-loading) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. バンドルアナライザーで依存関係のサイズを可視化する

`@next/bundle-analyzer` でバンドルの中身を可視化し、
肖大化した依存ライブラリを特定・置き換えする。

**根拠**:
- バンドルサイズの問題は可視化しなければ発見が困難
- 特定のライブラリ（moment.js、lodash など）が不必要に大きいことがある
- tree-shaking が効かない形でインポートしているケースを発見できる

**コード例**:
```ts
// next.config.ts
import type { NextConfig } from 'next';
import withBundleAnalyzer from '@next/bundle-analyzer';

const analyzerConfig = withBundleAnalyzer({
  enabled: process.env.ANALYZE === 'true',
});

const nextConfig: NextConfig = {
  // 設定...
};

export default analyzerConfig(nextConfig);

// package.json
// "analyze": "ANALYZE=true next build"
```

```tsx
// Bad: lodash 全体をインポート（tree-shaking が効きにくい）
import _ from 'lodash';
const sorted = _.sortBy(items, 'name');

// Good: 必要な関数のみインポート
import sortBy from 'lodash/sortBy';
const sorted = sortBy(items, 'name');

// Better: ネイティブ実装で置き換え（依存なし）
const sorted = [...items].sort((a, b) => a.name.localeCompare(b.name));

// moment.js → day.js または date-fns に移行
// moment.js: ~300KB → day.js: ~2KB
import dayjs from 'dayjs';
const formatted = dayjs(date).format('YYYY-MM-DD');
```

**出典**:
- [Next.js Docs: Bundle Analyzer](https://nextjs.org/docs/app/building-your-application/optimizing/package-bundling) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. Server Components でクライアントバンドルを削減する

重い処理やデータ変換ロジックは Server Components に移動し、
クライアントに送るJavaScriptを最小化する。

**根拠**:
- Server Components はクライアントバンドルに含まれない（JS ゼロ）
- マークダウンパーサー・日付ライブラリなどの重いライブラリをサーバーのみで使える
- `react-server-components` の思想に沿った設計でパフォーマンスが向上する

**コード例**:
```tsx
// Bad: クライアントコンポーネントで重いライブラリを使う
// → marked (~500KB) がクライアントバンドルに含まれる
'use client';
import { marked } from 'marked';

export function MarkdownRenderer({ content }: { content: string }) {
  const html = marked(content);
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}

// Good: Server Component に移動
// → marked がクライアントバンドルに含まれない
import { marked } from 'marked';

export async function MarkdownRenderer({ content }: { content: string }) {
  const html = await marked(content);
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}

// 日付フォーマット、通貨変換などの変換処理も同様
// Server Component で処理してからクライアントに文字列として渡す
async function PriceDisplay({ productId }: { productId: string }) {
  const product = await db.product.findUnique({ where: { id: productId } });
  const formattedPrice = new Intl.NumberFormat('ja-JP', {
    style: 'currency',
    currency: 'JPY',
  }).format(product!.price);

  // クライアントには処理済みの文字列だけ渡す
  return <span>{formattedPrice}</span>;
}
```

**出典**:
- [Next.js Docs: Server and Client Composition Patterns](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05
