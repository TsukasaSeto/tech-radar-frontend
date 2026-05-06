# Bootstrap v3.2: testing (2026-05-06)

## 結果サマリ
- 既存ルール強化: 3件
- 新規ルール追加: 0件
- 全段階失敗でスキップしたソース: 0件

## 公式ドキュメント取得統計

| リポジトリ | 候補1(raw) | 候補2(raw) | 候補3(raw) | Search API | 結果 |
|---|---|---|---|---|---|
| vitest-dev/vitest | ✅ 200 (README.md) | - | - | - | 候補1で成功（Vite統合・TypeScriptネイティブサポート確認） |
| microsoft/playwright | ✅ 200 (README.md) | - | - | - | 候補1で成功（auto-waiting・user-centric locators・trace確認） |
| testing-library/react-testing-library | ✅ 200 (README.md) | - | - | - | 候補1で成功（テスト哲学・semantic queries確認） |

## コミュニティ取得統計

| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 結果 |
|---|---|---|---|---|---|---|
| Zenn testing | ✅ 200 | - | - | - | - | 段階1で成功 |

## 個別記事取得統計

| URL | 取得経路 | 結果 |
|---|---|---|
| https://zenn.dev/miyan/articles/agile-testing-strategy-2026 | 段階A 直接WebFetch ✅ 200 | アーキテクチャ別モデル選択 + DORA メトリクス + フレーキーテスト分離 抽出 |

## 抽出されたルール

### 既存ルール強化

1. `practices/testing/test-strategy.md` - ルール5「テストピラミッドとテストダイヤモンドをフロントエンド文脈で使い分ける」
   - 追加根拠: Zenn 2026記事がアーキテクチャ別モデル（モノリス→ピラミッド/SPA→トロフィー/マイクロサービス→ダイヤモンド）を実務調査で確認。DORA メトリクス（変更失敗率・MTTR）による品質測定、フレーキーテスト別パイプライン分離も確認。
   - 出典: Zenn miyan 2026年テスト戦略記事 ※2026-05-06に実際にfetch成功

2. `practices/testing/e2e-testing.md` - ルール1「E2E テストはユーザーの重要なジャーニーに絞る」
   - 追加根拠: Playwright README が auto-waiting（人工タイムアウト不要）・user-centric locators・trace/debug機能を公式説明。`trace: 'on-first-retry'` 設定が推奨設定として確認。
   - 出典: microsoft/playwright README ※2026-05-06に実際にfetch成功

3. `practices/testing/e2e-testing.md` - ルール2「Page Object Model でテストコードを保守しやすくする」
   - 追加根拠: Playwright README が POM を明示推奨。React Testing Library README が「テスト実装ではなくユーザー操作に着目する」哲学を補強。
   - 出典: microsoft/playwright README + testing-library/react-testing-library README ※2026-05-06に実際にfetch成功

### 新規ルール
- 既存ファイルが非常に充実しているため、新規追加なし（全知見は既存ルールの強化に充当）

## 参照した記事一覧

| URL | 取得経路 | 抽出形式 |
|---|---|---|
| https://raw.githubusercontent.com/vitest-dev/vitest/main/README.md | raw.githubusercontent.com 直接 ✅ 200 | 公式 README / Vite統合・TypeScriptサポート確認 |
| https://raw.githubusercontent.com/microsoft/playwright/main/README.md | raw.githubusercontent.com 直接 ✅ 200 | 公式 README / 既存ルール追加根拠 |
| https://raw.githubusercontent.com/testing-library/react-testing-library/main/README.md | raw.githubusercontent.com 直接 ✅ 200 | 公式 README / 既存ルール追加根拠 |
| https://zenn.dev/topics/testing/feed | Zenn RSS 直接 ✅ 200 | フィード一覧取得 |
| https://zenn.dev/miyan/articles/agile-testing-strategy-2026 | 直接WebFetch ✅ 200 | 個別記事 / アーキテクチャ別テスト戦略 + DORA メトリクス |

## アクセス経路統計サマリ

| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 最終結果 |
|---|---|---|---|---|---|---|
| Zenn testing | ✅ 200 | - | - | - | - | 段階1で成功 |

## 公式ドキュメント経路統計

| リポジトリ | 候補1(raw) | 候補2(raw) | 候補3(raw) | Search API | 結果 |
|---|---|---|---|---|---|
| vitest-dev/vitest | ✅ 200 | - | - | - | 候補1で成功 |
| microsoft/playwright | ✅ 200 | - | - | - | 候補1で成功 |
| testing-library/react-testing-library | ✅ 200 | - | - | - | 候補1で成功 |

合計成功: 5/5 (100%)
