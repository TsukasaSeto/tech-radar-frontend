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
肥大化した依存ライブラリを特定・置き換えする。

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

---

### 4. `import()` による動的インポートとRoute-based Code Splitting を実装する

ページ単位（Route-based）でコードを分割し、現在のルートに必要なJavaScriptのみを
読み込む。Vite・Webpack どちらでも `import()` で実現できる。

**根拠**:
- ユーザーが訪問しないページのJSは初期ロード時に不要
- Route-based splitting は最もコスト対効果が高いコード分割戦略
- Next.js App Router はページコンポーネントを自動的にルートごとに分割するが、
  ページ内の重い機能モジュールは追加で動的インポートが必要

**コード例**:
```tsx
// Vite + React Router でのRoute-based Code Splitting
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

// lazy() で各ページコンポーネントを動的インポート
const HomePage = lazy(() => import('@/pages/HomePage'));
const DashboardPage = lazy(() => import('@/pages/DashboardPage'));
const SettingsPage = lazy(() => import('@/pages/SettingsPage'));
// → それぞれ別のチャンクに分割される

export function AppRouter() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/" element={<HomePage />} />
        <Route path="/dashboard" element={<DashboardPage />} />
        <Route path="/settings" element={<SettingsPage />} />
      </Routes>
    </Suspense>
  );
}

// ページ内の機能モジュールも動的インポートで遅延読み込み
// （PDFエクスポートなど、特定アクション時のみ必要な機能）
async function handleExportPDF() {
  // クリック時に初めて読み込む（初期バンドルに含めない）
  const { generatePDF } = await import('@/lib/pdf-generator');
  await generatePDF(currentData);
}

// Next.js App Router: ページは自動分割されるが、ページ内コンポーネントは手動
import dynamic from 'next/dynamic';

// 重い機能は明示的に動的インポート
const MapComponent = dynamic(() => import('@/components/MapComponent'), {
  loading: () => <MapSkeleton />,
  ssr: false,
});
```

**出典**:
- [MDN: import() (Dynamic import)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import) (MDN Web Docs)
- [Vite Docs: Code Splitting](https://vite.dev/guide/features#dynamic-import) (Vite公式)

**バージョン**: React 18+, Vite 5+, Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-06

---

#### 追加根拠 (2026-05-16) — ルール4「`import()` による動的インポートとRoute-based Code Splitting を実装する」

新たに以下の記事でコード分割の**過剰適用アンチパターン**が実例付きで示された:
- [[Frontend Performance - Part 16] 配信最適化の総仕上げ：Code Splitting・Cache・CDN戦略まとめ](https://qiita.com/tuanphan/items/e332db235525cfb9d10b) (Qiita tuanphan / 2026-05-15) ※2026-05-16に実際にfetch成功

**出典引用**:
> 「チャンクを細かくしすぎるとリクエスト数が増え、モバイルネットワークではオーバーヘッドが顕著になります」
> (セクション "Code Splitting の警告")

**追加の知見（反例・例外条件）**:

コード分割の粒度が細かすぎる場合（例: 5KB チャンクが 100 個）、並列リクエスト数の増加がモバイルネットワークではむしろ遅延を招く。分割の粒度は「ルート単位 > 機能モジュール単位 > コンポーネント単位」の優先順位で粗くすることが望ましい。

また同記事では HTTP キャッシュ戦略についても言及しており、リソース種別ごとの推奨設定は以下の通り:
```text
静的アセット（JS/CSS + ハッシュ付き）: Cache-Control: max-age=31536000, immutable
HTML:                                  Cache-Control: no-cache（+ ETag / Last-Modified）
古いデータを許容できる API:             Cache-Control: stale-while-revalidate=60
```

**確信度**: 既存（高）→ 高（過剰分割のアンチパターンと粒度設計の原則を追加）

---

### 5. Bundle Analyzer によるサイズ検証を CI に組み込む

バンドルサイズの増大をプルリクエスト段階で検知するため、CI に自動チェックを組み込む。
人手によるローカル確認では見落としが発生しやすい。

**根拠**:
- 依存ライブラリの追加・アップデートでバンドルサイズが無自覚に増大するリスクがある
- CI で size budget を設定することで、レグレッションを自動検出できる
- PRごとにサイズ差分をコメントで可視化することでレビュー品質が向上する

**コード例**:
```yaml
# .github/workflows/bundle-size.yml
name: Bundle Size Check

on:
  pull_request:
    branches: [main]

jobs:
  bundle-size:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build with bundle analysis
        run: ANALYZE=true npm run build
        env:
          ANALYZE: true

      # bundlesize パッケージでサイズ上限を検証
      - name: Check bundle size limits
        run: npx bundlesize
```

```json
// package.json: bundlesize の設定
{
  "bundlesize": [
    {
      "path": ".next/static/chunks/pages/*.js",
      "maxSize": "100 kB"  // 各ページチャンクの上限
    },
    {
      "path": ".next/static/chunks/framework-*.js",
      "maxSize": "250 kB"  // React フレームワーク
    },
    {
      "path": ".next/static/css/*.css",
      "maxSize": "50 kB"   // CSS の上限
    }
  ]
}
```

```ts
// next.config.ts: ビルド時にサイズレポートを出力
const nextConfig = {
  experimental: {
    // ビルドアウトプットにバンドルサイズの詳細を含める
    bundlePagesRouterDependencies: true,
  },
};
```

**出典**:
- [bundlesize GitHub](https://github.com/siddharthkp/bundlesize) (siddharthkp / OSS)
- [Next.js Docs: Building Your Application - Optimizing](https://nextjs.org/docs/app/building-your-application/optimizing/package-bundling) (Next.js公式)

**バージョン**: Next.js 13+, GitHub Actions
**確信度**: 中
**最終更新**: 2026-05-06

---

### 6. 内部バレルexportを削除してtree-shakingを有効にする

`atoms/molecules/organisms` などの内部UIコンポーネントディレクトリに置いた `index.ts` バレルファイルを削除し、インポートパスを直接指定する。内部バレルexportはtree-shakingを妨げてバンドルサイズを肥大化させる技術的負債になる。

**根拠**:
- バレルexportはモジュールグラフを複雑にし、ビルドツールが未使用コードを除去できなくなる
- 内部バレル削除は全ページ共有チャンクを半分まで削減した実測事例がある
- バレルexportはHMR（Hot Module Replacement）速度も低下させ、開発体験を悪化させる
- 外部ライブラリには `optimizePackageImports` で対応し、内部コードは直接パスインポートに統一する

**コード例**:
```ts
// Bad: 内部バレルexport（src/components/atoms/index.ts）
export { Button } from './Button';
export { Input } from './Input';
export { Avatar } from './Avatar';
// → バンドラーが使われていないコンポーネントを除去できないリスクがある

// Good: 直接パスインポート
import { Button } from '@/components/atoms/Button';
import { Input } from '@/components/atoms/Input';

// ESLint で内部バレルインポートを禁止し、CI で回帰を防ぐ
// .eslintrc
{
  "rules": {
    "no-restricted-imports": ["error", {
      "patterns": [{
        "group": ["@/components/atoms", "@/components/molecules", "@/components/organisms"],
        "message": "Use direct path imports: e.g. '@/components/atoms/Button'"
      }]
    }]
  }
}

// next.config.ts: 外部ライブラリのバレルインポートは optimizePackageImports で対応
const nextConfig = {
  experimental: {
    optimizePackageImports: ['@mui/material', 'lucide-react', 'date-fns'],
  },
};

// バンドルサイズ削減の計測: next experimental-analyze で Before/After を比較
// npx next experimental-analyze --output analyze.json
```

**出典**:
- [Next.js (Turbopack) のバンドルサイズを元の半分まで削減した話](https://zenn.dev/aldagram_tech/articles/nextjs-bundle-size-reduction) (Zenn aldagram_tech / 2026-05) ※2026-05-06に実際にfetch成功

**バージョン**: Next.js 13+, ESLint 8+
**確信度**: 高
**最終更新**: 2026-05-06

---

#### 追加根拠 (2026-05-06) — ルール1「`next/dynamic` で非クリティカルなコンポーネントを遅延読み込みする」

新たに以下の記事で逆用例（アンチパターン）が実測データで示された:
- [Lighthouse のスコアを上げようとして、逆に下げてしまった 2 つの失敗](https://zenn.dev/kimsuho/articles/b678c3b084c6d1) (Zenn kimsuho / 2026) ※2026-05-06に実際にfetch成功

初期表示される Widgets コンポーネント（62KB gzip）を `React.lazy()` で分割したところ、TBT が 20ms → 170ms（8倍増）、Lighthouse スコアが 95 → 92 に悪化。FCP は 1.1s → 0.9s に改善したが、TBT はスコアへの加重が高い（30%）ため逆効果になった。原因: コード分割はJavaScript総量を削減せず「実行タイミング」を変えるだけ。初期レンダリングで必要なコンポーネントを分割すると、FCP以降のメインスレッドブロック（= TBT）が増大する。修正: バレルのモーダル（WidgetPicker）のみに lazy-loading を限定した。適用範囲の原則: 「ユーザーのクリック・ナビゲーションがなければ表示されないUI（モーダル・アコーディオン・遷移先ページ）」にのみ使用する。

**確信度**: 既存（高）→ 高（実測データで適用条件を明確化）
