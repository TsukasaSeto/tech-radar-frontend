# Bootstrap v3.2: web-standards (2026-05-06)

## 結果サマリ
- 既存ルール強化: 2件
- 新規ルール追加: 1件
- 全段階失敗でスキップしたソース: 0件（mdn/content README はコントリビュータ用だったが、ARIA index.md で有効なデータ取得）

## 公式ドキュメント取得統計

| リポジトリ | 候補1(raw) | 候補2(raw) | 候補3(raw) | Search API | 結果 |
|---|---|---|---|---|---|
| mdn/content (README.md) | ✅ 200 (README.md) | - | - | - | 取得成功だがコントリビュータガイドのみ。アプリ開発ルール抽出不可 |
| mdn/content (aria/index.md) | - | ✅ 200 (files/en-us/web/accessibility/aria/index.md) | - | - | 候補2で成功（WebAIM 41%エラー増加データ確認） |

## コミュニティ取得統計

| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 結果 |
|---|---|---|---|---|---|---|
| Zenn accessibility | ✅ 200 | - | - | - | - | 段階1で成功 |
| Zenn css | ✅ 200 | - | - | - | - | 段階1で成功 |

## 個別記事取得統計

| URL | 取得経路 | 結果 |
|---|---|---|
| https://zenn.dev/seekseep/articles/css-new-features-catch-up-2026 | 段階A 直接WebFetch ✅ 200 | @layer（Cascade Layers）・Widely Available確認 + 新興CSS機能一覧抽出 |
| https://zenn.dev/gemcook/articles/css-accessibility-tips3 | 段階A 直接WebFetch ✅ 200 | prefers-contrast:more実装パターン・scroll-margin-top・reading-flow抽出 |

## 抽出されたルール

### 既存ルール強化

1. `practices/web-standards/accessibility.md` - ルール1「インタラクティブ要素には適切な ARIA 属性を付与する」
   - 追加根拠: MDN ARIA index が WebAIM 調査（ARIA使用ページは非ARIA使用ページより41%多くエラー検出）を引用。定量データで「No ARIA is better than bad ARIA」の根拠を強化。
   - 出典: mdn/content aria/index.md ※2026-05-06に実際にfetch成功

2. `practices/web-standards/accessibility.md` - ルール3「色のみで情報を伝えない。十分なコントラスト比を確保する」
   - 追加根拠: Zenn記事が `@media (prefers-contrast: more)` によるOSハイコントラスト設定への対応パターンを紹介。WCAG最低基準達成後の上乗せ対応として位置づけ。
   - 出典: Zenn gemcook CSS accessibilityアクセシビリティ記事 ※2026-05-06に実際にfetch成功

### 新規ルール

1. `practices/web-standards/css.md` - ルール7「`@layer` でCSSカスケードの優先順位を明示的に管理する」
   - 内容: @layer でスタイルをレイヤーに分割し、詳細度ハックや!important不要でCSSの優先順位を管理。サードパーティCSSのレイヤー封じ込めパターンも含む。Widely Availableで本番使用可。
   - 出典: Zenn seekseek CSS新機能記事（2026） ※2026-05-06に実際にfetch成功

## 参照した記事一覧

| URL | 取得経路 | 抽出形式 |
|---|---|---|
| https://raw.githubusercontent.com/mdn/content/main/README.md | raw.githubusercontent.com 直接 ✅ 200 | コントリビュータガイド（アプリ開発ルール抽出不可） |
| https://raw.githubusercontent.com/mdn/content/main/files/en-us/web/accessibility/aria/index.md | raw.githubusercontent.com 直接 ✅ 200 | ARIA原則・WebAIM定量データ / 既存ルール追加根拠 |
| https://zenn.dev/topics/accessibility/feed | Zenn RSS 直接 ✅ 200 | フィード一覧取得 |
| https://zenn.dev/topics/css/feed | Zenn RSS 直接 ✅ 200 | フィード一覧取得 |
| https://zenn.dev/seekseep/articles/css-new-features-catch-up-2026 | 直接WebFetch ✅ 200 | 個別記事 / @layer（Cascade Layers）新規ルール根拠 |
| https://zenn.dev/gemcook/articles/css-accessibility-tips3 | 直接WebFetch ✅ 200 | 個別記事 / prefers-contrast:more 実装パターン |

## アクセス経路統計サマリ

| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 最終結果 |
|---|---|---|---|---|---|---|
| Zenn accessibility | ✅ 200 | - | - | - | - | 段階1で成功 |
| Zenn css | ✅ 200 | - | - | - | - | 段階1で成功 |

## 公式ドキュメント経路統計

| リポジトリ | 候補1(raw) | 候補2(raw) | 候補3(raw) | Search API | 結果 |
|---|---|---|---|---|---|
| mdn/content (README.md) | ✅ 200 | - | - | - | 取得成功・内容はコントリビュータ用のためスキップ |
| mdn/content (aria/index.md) | - | ✅ 200 | - | - | 候補2で成功・ARIA原則・WebAIMデータ抽出 |

合計成功: 6/6 (100%)
