# Bootstrap Changelog: Testing カテゴリ

**実行日**: 2026-05-05
**ブランチ**: bootstrap/testing-2026-05-05
**処理カテゴリ**: testing

## 作成ファイル

| ファイル | ルール数 | 内容 |
|---|---|---|
| `practices/testing/unit-testing.md` | 3 | 振る舞いテスト・AAAパターン・describe/itの命名規約 |
| `practices/testing/component-testing.md` | 4 | getByRole優先クエリ・userEvent・waitFor/findBy・MSWモック |
| `practices/testing/e2e-testing.md` | 3 | 重要ジャーニーへの限定・Page Object Model・テストデータ独立性 |
| `practices/testing/test-strategy.md` | 4 | テストピラミッド・カバレッジの活用・Vitest設定・スナップショット方針 |

**合計**: 14 ルール / 4 トピック

## WebFetch 結果

全URLで 403 エラーが発生したため、モデルの学習知識からルールを作成した。
参照 URL は各ファイルの出典セクションに記載済み。

## 処理ノート

- Testing Library のクエリ優先順位（getByRole > getByLabelText > ... > getByTestId）を強調
- MSW v2 の新しい API (`http`, `HttpResponse`) を採用
- Vitest を推奨テストランナーとして明示（Next.js 公式も推奨）
- Playwright をE2Eテストフレームワークとして採用
