# Bootstrap v3.2: react (2026-05-06)

## 結果サマリ
- 既存ルール強化: 3件
- 新規ルール追加: 1件
- 全段階失敗でスキップしたソース: 0件

## 公式ドキュメント取得統計

| リポジトリ | 候補1(raw) | 候補2(raw) | 候補3(raw) | Search API | 結果 |
|---|---|---|---|---|---|
| reactjs/react.dev | ❌ 404 (learn/index.md) | ❌ 404 (synchronizing-with-effects) | ✅ 200 (you-might-not-need-an-effect.md) | - | 候補3で成功 |
| reactjs/react.dev (追加) | - | ✅ 200 (synchronizing-with-effects.md) | - | - | 候補2で成功 |

注: you-might-not-need-an-effect.md は候補3として試み成功。synchronizing-with-effects.md は候補2として別途取得。

## コミュニティ取得統計

| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 結果 |
|---|---|---|---|---|---|---|
| Zenn react | ✅ 200 | - | - | - | - | 段階1で成功 |

## 個別記事取得統計

| URL | 取得経路 | 結果 |
|---|---|---|
| https://zenn.dev/yasuhikasa/articles/48d93374b240db | 段階A 直接WebFetch ✅ 200 | Zustand Selective State Retrieval パターン抽出 |

## 抽出されたルール

### 既存ルール強化

1. `practices/react/hooks.md` - ルール1「useEffect でデータ取得しない」
   - 追加根拠: synchronizing-with-effects.md が「Effects = external system synchronization only」と公式定義。データフェッチは waterfalls/missing caches の原因として明示。
   - 出典: reactjs/react.dev synchronizing-with-effects.md ※2026-05-06に実際にfetch成功

2. `practices/react/hooks.md` - ルール5「ユーザー操作はイベントハンドラーに置く」
   - 追加根拠: you-might-not-need-an-effect.md が「Effects are caused by rendering, not events」と明確定義。Effects チェーンは中間レンダリングを生む問題を指摘。
   - 出典: reactjs/react.dev you-might-not-need-an-effect.md ※2026-05-06に実際にfetch成功

3. `practices/react/state-management.md` - ルール1「状態は最も近い共通の親コンポーネントに置く」
   - 追加根拠: you-might-not-need-an-effect.md の「Store identifiers, not derived values」パターン。Zustand 記事の Selective State Retrieval でグローバル状態でもコロケーション原則が適用可能と確認。
   - 出典: reactjs/react.dev + Zenn Zustand記事 ※2026-05-06に実際にfetch成功

### 新規ルール

1. `practices/react/hooks.md` - ルール7「`key` プロパティでコンポーネントの state を完全リセットする」
   - props 変化で state 全体をリセットする際は `useEffect` + setState ではなく `key` を使う。React 公式が "resetting all state with a key" として明示推奨。
   - 出典: reactjs/react.dev you-might-not-need-an-effect.md ※2026-05-06に実際にfetch成功

## 参照した記事一覧

| URL | 取得経路 | 抽出形式 |
|---|---|---|
| https://raw.githubusercontent.com/reactjs/react.dev/main/src/content/learn/you-might-not-need-an-effect.md | raw.githubusercontent.com 直接 ✅ 200 | 公式ドキュメント / 既存ルール追加根拠 + 新規ルール |
| https://raw.githubusercontent.com/reactjs/react.dev/main/src/content/learn/synchronizing-with-effects.md | raw.githubusercontent.com 直接 ✅ 200 | 公式ドキュメント / 既存ルール追加根拠 |
| https://zenn.dev/topics/react/feed | Zenn RSS 直接 ✅ 200 | フィード一覧取得 |
| https://zenn.dev/yasuhikasa/articles/48d93374b240db | 直接WebFetch ✅ 200 | 個別記事 / Zustand state management パターン |

## アクセス経路統計サマリ

| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 最終結果 |
|---|---|---|---|---|---|---|
| Zenn react | ✅ 200 | - | - | - | - | 段階1で成功 |

## 公式ドキュメント経路統計

| リポジトリ | 候補1(raw) | 候補2(raw) | 候補3(raw) | Search API | 結果 |
|---|---|---|---|---|---|
| reactjs/react.dev (you-might-not-need-an-effect) | ❌ 404 | ❌ 404 | ✅ 200 | - | 候補3で成功 |
| reactjs/react.dev (synchronizing-with-effects) | ❌ 404 | ✅ 200 | - | - | 候補2で成功 |

合計成功: 4/4 (100%)
