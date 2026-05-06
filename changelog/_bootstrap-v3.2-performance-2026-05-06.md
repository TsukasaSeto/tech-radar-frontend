# Bootstrap v3.2: performance (2026-05-06)

## 結果サマリ
- 既存ルール強化: 3件
- 新規ルール追加: 1件
- 全段階失敗でスキップしたソース: 0件

## 公式ドキュメント取得統計

| リポジトリ | 候補1(raw) | 候補2(raw) | 候補3(raw) | Search API | 結果 |
|---|---|---|---|---|---|
| GoogleChrome/web-vitals | ✅ 200 (README.md) | - | - | - | 候補1で成功（LCP/CLS/INP閾値・sendBeacon・attribution・バッチ送信パターン確認） |

## コミュニティ取得統計

| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 結果 |
|---|---|---|---|---|---|---|
| Zenn performance | ✅ 200 | - | - | - | - | 段階1で成功 |

## 個別記事取得統計

| URL | 取得経路 | 結果 |
|---|---|---|
| https://zenn.dev/kimsuho/articles/b678c3b084c6d1 | 段階A 直接WebFetch ✅ 200 | lazy-loading TBT劣化（20ms→170ms）+ fetchpriority="high" Speed Index劣化（1.0s→4.6s）実測データ抽出 |
| https://zenn.dev/aldagram_tech/articles/nextjs-bundle-size-reduction | 段階A 直接WebFetch ✅ 200 | 内部バレルexport削除でバンドルサイズ半減・optimizePackageImports・no-restricted-imports抽出 |

## 抽出されたルール

### 既存ルール強化

1. `practices/performance/bundle-size.md` - ルール1「`next/dynamic` で非クリティカルなコンポーネントを遅延読み込みする」
   - 追加根拠: Zenn記事が初期表示コンポーネントへのlazy-loading適用でTBT 20ms→170ms（8倍増）を実測。コード分割は実行タイミングを変えるだけで総量を減らさないため、初期レンダリング対象コンポーネントへの適用は逆効果になると実証。
   - 出典: Zenn kimsuho Lighthouseスコア失敗談記事 ※2026-05-06に実際にfetch成功

2. `practices/performance/core-web-vitals.md` - ルール1「LCP（最大コンテンツ描画）を最適化する」
   - 追加根拠: Zenn記事がJSヘビーなSPAでのfetchpriority="high"誤用でSpeed Index 1.0s→4.6s（4.6倍増）・スコア99→83を実測。JSボトルネック環境では画像優先化がクリティカルJS遅延を招き逆効果と実証。
   - 出典: Zenn kimsuho Lighthouseスコア失敗談記事 ※2026-05-06に実際にfetch成功

3. `practices/performance/core-web-vitals.md` - ルール4「`web-vitals` ライブラリで Core Web Vitals を本番計測してアナリティクスに送信する」
   - 追加根拠: web-vitals公式READMEがLCP/CLS/INP閾値エクスポート・attributionビルドのフィールド詳細・バッチ送信パターンを明示確認。
   - 出典: GoogleChrome/web-vitals README ※2026-05-06に実際にfetch成功

### 新規ルール

1. `practices/performance/bundle-size.md` - ルール6「内部バレルexportを削除してtree-shakingを有効にする」
   - 内容: atoms/molecules/organisms等の内部UIディレクトリのindex.tsバレルを削除し直接パスインポートに統一。no-restricted-importsで回帰防止、外部ライブラリはoptimizePackageImportsで対応。
   - 出典: Zenn aldagram_tech バンドルサイズ削減記事（全ページ共有チャンク半減の実測事例） ※2026-05-06に実際にfetch成功

## 参照した記事一覧

| URL | 取得経路 | 抽出形式 |
|---|---|---|
| https://raw.githubusercontent.com/GoogleChrome/web-vitals/main/README.md | raw.githubusercontent.com 直接 ✅ 200 | 公式 README / 既存ルール追加根拠 |
| https://zenn.dev/topics/performance/feed | Zenn RSS 直接 ✅ 200 | フィード一覧取得 |
| https://zenn.dev/kimsuho/articles/b678c3b084c6d1 | 直接WebFetch ✅ 200 | 個別記事 / lazy-loading TBT劣化 + fetchpriority劣化の実測アンチパターン |
| https://zenn.dev/aldagram_tech/articles/nextjs-bundle-size-reduction | 直接WebFetch ✅ 200 | 個別記事 / 内部バレルexport削除によるバンドルサイズ半減 |

## アクセス経路統計サマリ

| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 最終結果 |
|---|---|---|---|---|---|---|
| Zenn performance | ✅ 200 | - | - | - | - | 段階1で成功 |

## 公式ドキュメント経路統計

| リポジトリ | 候補1(raw) | 候補2(raw) | 候補3(raw) | Search API | 結果 |
|---|---|---|---|---|---|
| GoogleChrome/web-vitals | ✅ 200 | - | - | - | 候補1で成功 |

合計成功: 4/4 (100%)
