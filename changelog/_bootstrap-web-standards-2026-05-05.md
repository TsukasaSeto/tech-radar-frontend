# Bootstrap Changelog: Web Standards カテゴリ

**実行日**: 2026-05-05
**ブランチ**: bootstrap/web-standards-2026-05-05
**処理カテゴリ**: web-standards

## 作成ファイル

| ファイル | ルール数 | 内容 |
|---|---|---|
| `practices/web-standards/html.md` | 3 | セマンティックHTML・フォームのlabel/type・Next.js Metadata API |
| `practices/web-standards/css.md` | 3 | Tailwindユーティリティファースト・CSS変数でデザイントークン・モバイルファーストレスポンシブ |
| `practices/web-standards/browser-apis.md` | 3 | IntersectionObserver・ResizeObserver・Web Storage使い分け |
| `practices/web-standards/accessibility.md` | 3 | ARIA属性・キーボードナビゲーション・コントラスト比 |

**合計**: 12 ルール / 4 トピック

## WebFetch 結果

全URLで 403 エラーが発生したため、モデルの学習知識からルールを作成した。
参照 URL は各ファイルの出典セクションに記載済み。

## 処理ノート

- アクセシビリティは WCAG 2.1 AA 基準を採用
- Web Storage のセキュリティリスク（XSS/機密情報）について明記
- `IntersectionObserver` と `ResizeObserver` は `window.scroll` / `window.resize` の代替として紹介
- CSS カスタムプロパティによるダークモード実装パターンを含む
