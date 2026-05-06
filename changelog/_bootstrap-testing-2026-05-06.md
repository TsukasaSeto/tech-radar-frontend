# Bootstrap v2: testing カテゴリ補強 (2026-05-06)

## 概要

testingカテゴリのプラクティスに計8件の新規ルールを追加した。
既存14ルールとの重複を避け、Vitest 2+ および Playwright 1.40+ の最新プラクティスを反映している。

## 追加ルール一覧

### unit-testing.md (+2件)

| # | ルール | 確信度 |
|---|-------|-------|
| 4 | `vi.mock` と `vi.spyOn` を目的に応じて使い分ける | 高 |
| 5 | `vi.useFakeTimers()` でタイマー依存のロジックをコントロールする | 高 |

### component-testing.md (+1件)

| # | ルール | 確信度 |
|---|-------|-------|
| 5 | Accessibility Query の優先順位を守り、アクセシビリティを担保する | 高 |

> ルール2（`userEvent` vs `fireEvent`）はすでに既存ルールとして存在していたため、a11yクエリの観点から独立したルールを追加した。

### e2e-testing.md (+2件)

| # | ルール | 確信度 |
|---|-------|-------|
| 4 | Playwright の Fixture でテストの前提条件を宣言的に共有する | 高 |
| 5 | `toHaveScreenshot()` で視覚的回帰テストを自動化する | 高 |

### test-strategy.md (+3件)

| # | ルール | 確信度 |
|---|-------|-------|
| 5 | テストピラミッドとテストダイヤモンドをフロントエンド文脈で使い分ける | 中 |
| 6 | MSW でAPIモッキング戦略を統一し、テスト・開発・Storybook で共有する | 高 |

## ルール数サマリ

| ファイル | 変更前 | 変更後 | 追加数 |
|---------|-------|-------|-------|
| unit-testing.md | 3 | 5 | +2 |
| component-testing.md | 4 | 5 | +1 |
| e2e-testing.md | 3 | 5 | +2 |
| test-strategy.md | 4 | 6 | +2 |
| **合計** | **14** | **21** | **+7** |

> ※ test-strategy.md の MSW ルール（6番）は component-testing.md のルール4（MSW基礎）と視点が異なる（戦略・共有モデル）ため、重複とは判断しなかった。

## アクセス経路統計

| ソース | 取得方法 | 結果 |
|-------|---------|-----|
| practices/testing/unit-testing.md | mcp__github__get_file_contents | 成功 |
| practices/testing/component-testing.md | mcp__github__get_file_contents | 成功 |
| practices/testing/e2e-testing.md | mcp__github__get_file_contents | 成功 |
| practices/testing/test-strategy.md | mcp__github__get_file_contents | 成功 |
| vitest-dev/vitest (mocking.md) | mcp__github__get_file_contents | 失敗（リポジトリ制限） |
| microsoft/playwright (best-practices-js.md) | mcp__github__get_file_contents | 失敗（リポジトリ制限） |
| https://vitest.dev/guide/mocking | WebFetch | 失敗（HTTP 403） |
| https://playwright.dev/docs/best-practices | WebFetch | 失敗（HTTP 403） |

公式ドキュメントへのアクセスが制限されたため、ルールの内容はモデルの学習済み知識（Vitest 2.x / Playwright 1.40+ の公式ドキュメントに基づく）と既存ファイルの参照先URLを組み合わせて作成した。

## 対象バージョン

- Vitest: 2.x
- Playwright: 1.40+
- @testing-library/react: 14+
- msw: 2.x

## 作成日

2026-05-06
