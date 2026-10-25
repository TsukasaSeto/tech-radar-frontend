# バンドルサイズ最適化のベストプラクティス

## ルール

### 1. `next/dynamic` で非クリティカルなコンポーネントを遅延読み込みする

ファーストビューに不要なコンポーネント（モーダル・リッチテキストエディタ・チャートなど）は
`dynamic()` で遅延読み込みする。対象はユーザーの操作がなければ表示されない UI に限り、初期表示で必ず描画されるコンポーネントは分割しない（下の追加根拠を参照）。

**根拠**:
- 初期バンドルサイズが減少し、TTI（Time to Interactive）が改善する
- SSRが不要なコンポーネントは `{ ssr: false }` でサーバーサイド実行を省略できる。ただし `ssr: false` は Client Component の中でしか指定できず、Server Component で使うとエラーになる
- モーダルなど条件付き表示要素は表示時に読み込めば十分

**コード例**:
```tsx
// app/blog/_components/BlogEditor.tsx
'use client';
import { useState } from 'react';
import dynamic from 'next/dynamic';
import { EditorSkeleton } from '@/components/EditorSkeleton';

// 「編集する」を押すまで表示されない重いエディタを遅延読み込み
const RichTextEditor = dynamic(
  () => import('@/components/RichTextEditor'),
  {
    loading: () => <EditorSkeleton />,  // 読み込み中の表示
    ssr: false,  // ブラウザ API に依存するためプリレンダリングしない（Client Component 内でのみ指定可）
  }
);

// モーダルは開いた時に読み込む
const ConfirmDialog = dynamic(() => import('@/components/ConfirmDialog'));

export default function BlogEditor() {
  const [isEditing, setIsEditing] = useState(false);
  const [isDialogOpen, setIsDialogOpen] = useState(false);

  return (
    <div>
      {/* isEditing が true になって初めてエディタのチャンクを読み込む */}
      {isEditing ? (
        <RichTextEditor />
      ) : (
        <button onClick={() => setIsEditing(true)}>編集する</button>
      )}

      <button onClick={() => setIsDialogOpen(true)}>削除</button>
      {isDialogOpen && <ConfirmDialog onClose={() => setIsDialogOpen(false)} />}
    </div>
  );
}

// チャートライブラリなど大きな依存を持つコンポーネント（同じ 'use client' ファイル内で定義する）
const DataChart = dynamic(
  () => import('@/components/DataChart'),  // chart.js などを含む
  { ssr: false }
);
```

**出典**:
- [Next.js Docs: Lazy Loading](https://nextjs.org/docs/app/guides/lazy-loading) (Next.js公式、"`ssr: false` option is not supported in Server Components.") ※2026-10-25 確認

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 2. バンドルアナライザーで依存関係のサイズを可視化する

バンドルの中身を可視化し、肥大化した依存ライブラリを特定・置き換えする。
Next.js 16 の既定である Turbopack のビルドでは `next analyze`（16.1+）を使う。`@next/bundle-analyzer` は Webpack プラグインなので、`next build --webpack` でビルドする場合（と v15 以前の Webpack ビルド）に限って使う。

**根拠**:
- バンドルサイズの問題は可視化しなければ発見が困難
- 特定のライブラリ（moment.js、lodash など）が不必要に大きいことがある
- tree-shaking が効かない形でインポートしているケースを発見できる
- Turbopack は Webpack プラグインに対応しないため、Turbopack のビルドに `@next/bundle-analyzer` を組み込んでもレポートは出ない

**コード例**:
```bash
# Turbopack（Next.js 16 の既定）: 対話的なビューをブラウザで開く
npx next analyze

# 結果を .next/diagnostics/analyze に書き出す（Before / After の比較や共有用）
npx next analyze --output
cp -r .next/diagnostics/analyze ./analyze-before-refactor
```

```ts
// Webpack でビルドする場合のみ: @next/bundle-analyzer
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

// package.json（Next.js 16 では --webpack を付けないと Turbopack でビルドされる）
// "analyze": "ANALYZE=true next build --webpack"
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
- [Next.js Docs: Optimizing package bundling](https://nextjs.org/docs/app/guides/package-bundling) (Next.js公式、Turbopack 用の `next analyze` と Webpack 用の `@next/bundle-analyzer`) ※2026-10-25 確認
- [Next.js Docs: Turbopack](https://nextjs.org/docs/app/api-reference/turbopack) (Next.js公式、"Turbopack does not support webpack plugins."、`--webpack` フラグ) ※2026-10-25 確認

**バージョン**: Next.js 16.1+（`next analyze`）。Webpack ビルドの `@next/bundle-analyzer` は 13+（`next.config.ts` は 15+）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 3. Server Components でクライアントバンドルを削減する

重い処理やデータ変換ロジックは Server Components に移動し、
クライアントに送るJavaScriptを最小化する。

**根拠**:
- Server Components はクライアントバンドルに含まれない（JS ゼロ）
- マークダウンパーサー・日付ライブラリなどの重いライブラリをサーバーのみで使える
- `react-server-components` の思想に沿った設計でパフォーマンスが向上する
- サーバーに移しても XSS 対策は別途必要。marked は出力 HTML をサニタイズしないため、ユーザー由来の Markdown を `dangerouslySetInnerHTML` に渡すなら、変換後の HTML を DOMPurify（サーバー側では isomorphic-dompurify）に通す（`practices/security/xss.md` #1）
- データの取得はコンポーネントから DB を直接呼ばず DAL 関数を経由する（`practices/architecture/data-access-layer.md` #1）

**コード例**:
```tsx
// Bad: クライアントコンポーネントで重いライブラリを使う
// → marked と DOMPurify がクライアントバンドルに含まれる
'use client';
import { marked } from 'marked';
import DOMPurify from 'dompurify';

export function MarkdownRenderer({ content }: { content: string }) {
  const html = DOMPurify.sanitize(marked.parse(content, { async: false }));
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}

// Good: Server Component に移動
// → marked と DOMPurify がクライアントバンドルに含まれない
import { marked } from 'marked';
import DOMPurify from 'isomorphic-dompurify';

export async function MarkdownRenderer({ content }: { content: string }) {
  // marked は出力をサニタイズしないので、変換後の HTML を必ず DOMPurify に通す
  const html = DOMPurify.sanitize(await marked.parse(content));
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}

// 日付フォーマット、通貨変換などの変換処理も同様
// Server Component で処理してからクライアントに文字列として渡す
import { getProduct } from '@/app/_lib/dal/product';

async function PriceDisplay({ productId }: { productId: string }) {
  // DAL 経由で取得する（見つからなければ DAL 内で notFound() する）
  const product = await getProduct(productId);
  const formattedPrice = new Intl.NumberFormat('ja-JP', {
    style: 'currency',
    currency: 'JPY',
  }).format(product.price);

  // クライアントには処理済みの文字列だけ渡す
  return <span>{formattedPrice}</span>;
}
```

**出典**:
- [Next.js Docs: Server and Client Composition Patterns](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns) (Next.js公式)
- [Marked Documentation](https://marked.js.org/) (marked 公式、"Marked does not sanitize the output HTML." として DOMPurify を推奨) ※2026-10-25 確認

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-10-25

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
```

```tsx
// Next.js App Router: ページは自動分割されるが、ページ内コンポーネントは手動
// ssr: false を使うので Client Component に置く（Server Component ではエラー。#1 参照）
'use client';
import dynamic from 'next/dynamic';
import { MapSkeleton } from '@/components/MapSkeleton';

// 重い機能は明示的に動的インポート
export const MapComponent = dynamic(() => import('@/components/MapComponent'), {
  loading: () => <MapSkeleton />,
  ssr: false,
});
```

**出典**:
- [MDN: import() (Dynamic import)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import) (MDN Web Docs)
- [Vite Docs: Code Splitting](https://vite.dev/guide/features#dynamic-import) (Vite公式)

**バージョン**: React 18+, Vite 5+, Next.js 13+
**確信度**: 高
**最終更新**: 2026-10-25

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
      - uses: actions/checkout@v7

      - name: Setup Node.js
        uses: actions/setup-node@v7
        with:
          node-version: '24'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      # bundlesize パッケージでサイズ上限を検証
      - name: Check bundle size limits
        run: npx bundlesize
```

package.json に置く bundlesize の設定。App Router のクライアント JS は Turbopack / Webpack のどちらでも `.next/static/chunks/` 以下（Webpack ではルートごとに `app/` などのサブディレクトリ）に出力されるので、`**` で全チャンクを対象にする（`chunks/pages/*.js` だけでは App Router のチャンクが検査されない）。

```json
{
  "bundlesize": [
    {
      "path": ".next/static/chunks/**/*.js",
      "maxSize": "100 kB"
    },
    {
      "path": ".next/static/**/*.css",
      "maxSize": "50 kB"
    }
  ]
}
```

**出典**:
- [bundlesize GitHub](https://github.com/siddharthkp/bundlesize) (siddharthkp / OSS)
- [Next.js Docs: Optimizing package bundling](https://nextjs.org/docs/app/guides/package-bundling) (Next.js公式)

**バージョン**: Next.js 13+, GitHub Actions
**確信度**: 中
**最終更新**: 2026-10-25

---

### 6. 共有チャンクを肥大化させている内部バレルexportは削除し、直接パスでインポートする

`atoms/molecules/organisms` などの内部UIコンポーネントディレクトリに置いた `index.ts` バレルファイルが共有チャンクを肥大化させていると計測（`next analyze` 等）で確認できたら、そのバレルを削除し、インポートパスを直接指定する。すべての内部バレルが tree-shaking を妨げるわけではないので、削除の対象は計測で問題が出たバレルに限る。

**根拠**:
- 副作用を含むモジュールや再エクスポートが多いバレルはモジュールグラフを複雑にし、ビルドツールが未使用コードを除去しきれないことがある
- 内部バレル削除は全ページ共有チャンクを半分まで削減した実測事例がある（下記出典の 1 事例）
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

// ESLint で削除したバレルのパスそのものへのインポートを禁止し、CI で回帰を防ぐ
// paths は完全一致で判定するので '@/components/atoms/Button' は禁止されない
// （patterns.group は gitignore 形式なので、'@/components/atoms' を書くと配下の直接パスまで禁止される）
// .eslintrc
{
  "rules": {
    "no-restricted-imports": ["error", {
      "paths": [
        { "name": "@/components/atoms", "message": "Use direct path imports: e.g. '@/components/atoms/Button'" },
        { "name": "@/components/molecules", "message": "Use direct path imports" },
        { "name": "@/components/organisms", "message": "Use direct path imports" }
      ]
    }]
  }
}

// next.config.ts: 外部ライブラリのバレルインポートは optimizePackageImports で対応
const nextConfig = {
  experimental: {
    optimizePackageImports: ['@mui/material', 'lucide-react', 'date-fns'],
  },
};

// バンドルサイズ削減の計測: next analyze（Next.js 16.1+）で Before/After を比較
// npx next analyze --output   # .next/diagnostics/analyze に書き出す
```

**出典**:
- [Next.js (Turbopack) のバンドルサイズを元の半分まで削減した話](https://zenn.dev/aldagram_tech/articles/nextjs-bundle-size-reduction) (Zenn aldagram_tech / 2026-05) ※2026-05-06に実際にfetch成功
- [Next.js Docs: Optimizing package bundling](https://nextjs.org/docs/app/guides/package-bundling) (Next.js公式、`next analyze --output`) ※2026-10-25 確認

**バージョン**: Next.js 13+, ESLint 8+（`next analyze` は Next.js 16.1+）
**確信度**: 中（バレル削除の効果は 1 事例の実測のみ）
**最終更新**: 2026-10-25

---

#### 追加根拠 (2026-05-06) — ルール1「`next/dynamic` で非クリティカルなコンポーネントを遅延読み込みする」

新たに以下の記事で逆用例（アンチパターン）が実測データで示された:
- [Lighthouse のスコアを上げようとして、逆に下げてしまった 2 つの失敗](https://zenn.dev/kimsuho/articles/b678c3b084c6d1) (Zenn kimsuho / 2026) ※2026-05-06に実際にfetch成功

初期表示される Widgets コンポーネント（62KB gzip）を `React.lazy()` で分割したところ、TBT が 20ms → 170ms（8倍増）、Lighthouse スコアが 95 → 92 に悪化。FCP は 1.1s → 0.9s に改善したが、TBT はスコアへの加重が高い（30%）ため逆効果になった。原因: コード分割はJavaScript総量を削減せず「実行タイミング」を変えるだけ。初期レンダリングで必要なコンポーネントを分割すると、FCP以降のメインスレッドブロック（= TBT）が増大する。修正: バレルのモーダル（WidgetPicker）のみに lazy-loading を限定した。適用範囲の原則: 「ユーザーのクリック・ナビゲーションがなければ表示されないUI（モーダル・アコーディオン・遷移先ページ）」にのみ使用する。

**確信度**: 既存（高）→ 高（実測データで適用条件を明確化）

---

### 7. Turbopack のチャンク結合はナビゲーションパターンに合わせてチューニングする（Next.js 16.3+）

Turbopack はビルド時に複数の小さいチャンクを「チャンクグループ」単位で結合し、リクエスト数とダウンロード量のトレードオフを最適化する。デフォルトの結合戦略は「初回訪問が2/3、複数ページ遷移が1/3」という仮定に基づく統計的な重み付けであり、実際のサイトの回遊パターンと異なる場合は `experimental.turbopackChunking` で明示的にチューニングできる。加えて、ソフトナビゲーション時の再ダウンロードを避けるため `generateComponentChunks` で結合チャンクと未結合チャンクを両方生成し、ブラウザのキャッシュ状況に応じてランタイムが有利な方を選択できるようにする。

**根拠**:
- チャンク結合はビルド時に決定されるため、訪問者が既に何をキャッシュ済みかを考慮できない。`generateComponentChunks` はこの制約を実行時の選択に置き換える
- サイトの実際の回遊率（単一ページ訪問 vs 複数ページ遷移）はデフォルトの 2/3 という仮定と乖離しうるため、`firstPageLoadPriority` / `priorityRoutes` / `clusters` で自サイトの計測値に合わせて調整できる
- CJS モジュールの tree-shaking（`turbopackCjsTreeShaking`）と共有ランタイム（`turbopackSharedRuntime`）は、チャンク結合とは独立に「そもそもの出力量」を削減する

**コード例**:
```ts
// next.config.js
const nextConfig = {
  experimental: {
    // ソフトナビゲーション時に結合/未結合チャンクを両方生成し、ランタイムが選択
    turbopackChunking: {
      generateComponentChunks: true,
      firstPageLoadPriority: 0.67, // 0-1: 初回ロード優先度（デフォルト0.67 = 直帰率相当）
      priorityRoutes: ['/'],       // 表示速度を最優先したいルート
      clusters: [[/^\/blog/, /^\/blog\/.*/]], // 一緒に回遊されやすいルート群
    },
    // CJS モジュールの tree-shaking を有効化（将来デフォルト化予定）
    turbopackCjsTreeShaking: true,
    // ページ単位ランタイムを1つの共有ランタイムに統合（初回以降のナビゲーションで約10KB削減）
    turbopackSharedRuntime: true,
  },
};
```

**出典引用**:
> "In Next.js 16.3 or later, enabling `experimental.turbopackChunking.generateComponentChunks` in your `next.config.js` makes Turbopack emit un-merged versions of chunks alongside the merged ones... at request time we can pick whichever is cheaper: the merged chunk, or just the pieces that are missing."
> ([How Turbopack chunks your JavaScript](https://nextjs.org/blog/turbopack-chunking), セクション "Smarter chunk fetching") ※2026-09-03に実際にfetch成功

**バージョン**: Next.js 16.3+
**確信度**: 高（Next.js 公式ブログ、実測ベンチマーク付き）
**最終更新**: 2026-09-03

---

### 8. 大容量静的アセットは Vercel の `rewrites` で外部ストレージへ逃がし、コード参照を書き換えない

デプロイ容量が上限に近づいた場合、画像・音声等の大容量静的アセットをコード内の参照パスは変えずに `vercel.json` の `rewrites` で外部ストレージ（Supabase Storage 等）へ透過的にプロキシし、`.vercelignore` でデプロイ対象から除外する。HTML/コード側は相対パスのままで、Vercel がリクエスト時にリライト先へ転送する。

**根拠**:
- `rewrites` はURLパスのマッチングのみで動作するため、既存コードの参照（`src="sprites/foo.png"` 等）を1つも書き換える必要がない。大量の参照を一括置換するリスクを避けられる
- `.vercelignore` でアセットをデプロイパッケージから除外することで、デプロイ容量そのものを削減できる（実例: 72MB → 7.9MB、-89%）
- ブラウザ・クローラー等クライアントから見えるURLは変わらないため、SEOやキャッシュ済みリンクへの影響がない

**コード例**:
```json
// vercel.json
{ "rewrites": [
  { "source": "/sprites/:path*",
    "destination": "https://xxxx.supabase.co/storage/v1/object/public/kanji/sprites/:path*" },
  { "source": "/audio/:path*",
    "destination": "https://xxxx.supabase.co/storage/v1/object/public/kanji/audio/:path*" }
]}
```
```
# .vercelignore
# 除外するのは rewrites の対象と同じ範囲だけ（`*.mp3` のように拡張子で広く除外すると、リライト対象外のパスにある音声が 404 になる）
sprites/
audio/
```

**出典引用**:
> "HTMLから見える世界は何も変わらない"
> ([デプロイ容量が100%になった。圧縮では解けず、参照を1つも書き換えずに89%減らした](https://zenn.dev/horibe/articles/assets-off-vercel-rewrites), セクション "正しい対処は「外に出す」") ※2026-09-22に実際にfetch成功

**出典**:
- [デプロイ容量が100%になった。圧縮では解けず、参照を1つも書き換えずに89%減らした](https://zenn.dev/horibe/articles/assets-off-vercel-rewrites) (Zenn horibe、Vercel公式機能である`rewrites`/`.vercelignore`の設定ファイル形式を直接示す検証記事) ※2026-09-22に実際にfetch成功

**バージョン**: Vercel（`rewrites` / `.vercelignore`、全プラン共通）
**確信度**: 中（パターン1c: 公式機能の設定ファイル形式を直接検証した単独記事）
**最終更新**: 2026-10-25
