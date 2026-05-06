# Bootstrap v3.2: architecture (2026-05-06)

## 結果サマリ
- 既存ルール強化: 2件
- 新規ルール追加: 0件（既存ファイルが非常に充実しているため全知見は既存ルールの強化に充当）
- 全段階失敗でスキップしたソース: 0件

## 公式ドキュメント取得統計

| リポジトリ | 候補1(raw) | 候補2(raw) | 候補3(raw) | Search API | 結果 |
|---|---|---|---|---|---|
| alan2207/bulletproof-react (README.md) | ✅ 200 (README.md) | - | - | - | 高レベル原則のみ、具体的なルール抽出困難 |
| alan2207/bulletproof-react (docs/project-structure.md) | - | ✅ 200 (docs/project-structure.md) | - | - | 候補2で成功（Unidirectional Architecture・直接インポート推奨・eslint/no-restricted-paths確認） |

## コミュニティ取得統計

| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 結果 |
|---|---|---|---|---|---|---|
| Zenn frontend | ✅ 200 | - | - | - | - | 段階1で成功 |

## 個別記事取得統計

| URL | 取得経路 | 結果 |
|---|---|---|
| https://zenn.dev/fujishu/articles/c7c7799140b488 | 段階A 直接WebFetch ✅ 200 | バレルexportのオーバーヘッド数学的証明 + コンパイル時間実測（7.44s→5.98s、1.55s改善）抽出 |

## 抽出されたルール

### 既存ルール強化

1. `practices/architecture/directory-structure.md` - ルール5「バレルエクスポート（index.ts）の過剰使用を避ける」
   - 追加根拠: Zenn記事がオーバーヘッド率の数学的証明（(N-k)/N → 1 as N→∞）とコンパイル時間実測（7.44s→5.98s、21%改善）を提示。Bulletproof Reactも「直接インポート推奨」を明示。
   - 出典: Zenn fujishu記事 + Bulletproof React docs/project-structure.md ※2026-05-06に実際にfetch成功

2. `practices/architecture/module-boundary.md` - ルール1「依存の方向を一方向に保つ（Dependency Rule）」
   - 追加根拠: Bulletproof React公式が "Unidirectional Architecture" として shared→features→app の一方向を明示推奨。"compose different features at the application level" 指針も確認。
   - 出典: Bulletproof React docs/project-structure.md ※2026-05-06に実際にfetch成功

### 新規ルール
- 既存ファイルが非常に充実しているため、新規追加なし（全知見は既存ルールの強化に充当）

## 参照した記事一覧

| URL | 取得経路 | 抽出形式 |
|---|---|---|
| https://raw.githubusercontent.com/alan2207/bulletproof-react/master/README.md | raw.githubusercontent.com 直接 ✅ 200 | README（高レベル原則のみ） |
| https://raw.githubusercontent.com/alan2207/bulletproof-react/master/docs/project-structure.md | raw.githubusercontent.com 直接 ✅ 200 | プロジェクト構造ガイド / 既存ルール追加根拠 |
| https://zenn.dev/topics/frontend/feed | Zenn RSS 直接 ✅ 200 | フィード一覧取得 |
| https://zenn.dev/fujishu/articles/c7c7799140b488 | 直接WebFetch ✅ 200 | 個別記事 / バレルexportのオーバーヘッド数学的証明 + コンパイル時間実測 |

## アクセス経路統計サマリ

| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 最終結果 |
|---|---|---|---|---|---|---|
| Zenn frontend | ✅ 200 | - | - | - | - | 段階1で成功 |

## 公式ドキュメント経路統計

| リポジトリ | 候補1(raw) | 候補2(raw) | 候補3(raw) | Search API | 結果 |
|---|---|---|---|---|---|
| alan2207/bulletproof-react (README.md) | ✅ 200 | - | - | - | 取得成功・高レベル原則のみのためルール抽出に候補2も使用 |
| alan2207/bulletproof-react (project-structure.md) | - | ✅ 200 | - | - | 候補2で成功 |

合計成功: 4/4 (100%)
