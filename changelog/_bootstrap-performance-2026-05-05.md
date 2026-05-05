# Bootstrap Changelog: Performance カテゴリ

**実行日**: 2026-05-05
**ブランチ**: bootstrap/performance-2026-05-05
**処理カテゴリ**: performance

## 作成ファイル

| ファイル | ルール数 | 内容 |
|---|---|---|
| `practices/performance/core-web-vitals.md` | 3 | LCP最適化(priority/preload)・CLS防止(サイズ指定/スケルトン)・INP最適化(useTransition) |
| `practices/performance/rendering.md` | 3 | Profiler計測後の最適化・仮想スクロール(TanStack Virtual)・Context分割 |
| `practices/performance/bundle-size.md` | 3 | next/dynamic遅延読み込み・バンドルアナライザー・Server Componentsでバンドル削減 |
| `practices/performance/image-optimization.md` | 3 | next/image必須・sizes属性の正確な指定・next/ogで動的OGP生成 |

**合計**: 12 ルール / 4 トピック

## WebFetch 結果

全URLで 403 エラーが発生したため、モデルの学習知識からルールを作成した。
参照 URL は各ファイルの出典セクションに記載済み。

## 処理ノート

- INP（Interaction to Next Paint）は 2024年3月から Core Web Vitals に加わった新指標として記載
- TanStack Virtual v3 の最新 API (`useVirtualizer`) を採用
- `next/og`（ImageResponse）はエッジランタイムでの動的OGP生成として記載
