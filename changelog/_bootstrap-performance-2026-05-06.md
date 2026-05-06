# Bootstrap v2: performance (2026-05-06)

## 概要

performanceカテゴリへの補強コンテンツ追加（v2ブートストラップ）。
既存3ルール×4ファイル計12ルールに対し、重複しない新規ルールを8件追加。

## 追加ルール一覧

### core-web-vitals.md (+1件)

| # | ルール | 確信度 |
|---|--------|--------|
| 4 | `web-vitals` ライブラリで Core Web Vitals を本番計測してアナリティクスに送信する | 高 |

### rendering.md (+2件)

| # | ルール | 確信度 |
|---|--------|--------|
| 4 | `memo`・`useMemo`・`useCallback` の過剰適用を避ける | 高 |
| 5 | `startTransition` でレンダリング優先度を制御し UI の応答性を保つ | 高 |

### bundle-size.md (+2件)

| # | ルール | 確信度 |
|---|--------|--------|
| 4 | `import()` による動的インポートとRoute-based Code Splitting を実装する | 高 |
| 5 | Bundle Analyzer によるサイズ検証を CI に組み込む | 中 |

### image-optimization.md (+3件)

| # | ルール | 確信度 |
|---|--------|--------|
| 4 | `<picture>` 要素と `srcset` でレスポンシブ画像を最適化する（Next.js 外の場合） | 高 |
| 5 | AVIF フォーマットを優先して使用する | 高 |

## アクセス経路統計

| ソース | ファイル数 | 成功/失敗 |
|--------|-----------|----------|
| GitHub MCP (tsukasaseto/tech-radar-frontend) | 4 | 4/0 |
| GitHub MCP (GoogleChrome/web-vitals) | 1 | 0/1（アクセス制限） |
| GitHub MCP (mdn/content) | 1 | 0/1（アクセス制限） |
| WebFetch (web.dev/articles/inp) | 1 | 0/1（403） |
| WebFetch (web.dev/articles/lcp) | 1 | 0/1（403） |
| WebFetch (github.com README経由) | 1 | 1/0 |
| WebFetch (developer.mozilla.org) | 複数 | 0/複数（403） |

主要ドキュメントの取得制限により、web-vitalsライブラリのAPIはGitHub README経由で取得。
その他のルール（picture要素、AVIF、startTransition等）は仕様の知識と既存ドキュメントURLを参照して作成。

## 変更ファイル

- `practices/performance/core-web-vitals.md` （ルール4追加）
- `practices/performance/rendering.md` （ルール4・5追加）
- `practices/performance/bundle-size.md` （ルール4・5追加）
- `practices/performance/image-optimization.md` （ルール4・5追加）
- `changelog/_bootstrap-performance-2026-05-06.md` （本ファイル、新規作成）

## 作成日

2026-05-06
