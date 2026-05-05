# Bootstrap Log: react (2026-05-05)

## 処理サマリ

| トピック | ファイル | ルール数 |
|---|---|---|
| Hooks | `hooks.md` | 4（既存2+追記2） |
| 状態管理 | `state-management.md` | 4 |
| レンダリング | `rendering.md` | 4 |
| パターン | `patterns.md` | 4 |
| Suspense | `suspense.md` | 3 |

**合計**: 19 ルール / 5 トピック

## 参照した一次情報

| URL | ステータス | 備考 |
|---|---|---|
| https://react.dev/learn | 403 FAILED | 環境制限によりアクセス不可 |
| https://react.dev/reference/react | 403 FAILED | 環境制限によりアクセス不可 |

**注記**: 全 WebFetch が 403 で失敗したため、モデルのトレーニングデータ（React 公式ドキュメントの知識）をもとにルールを作成。

## 意図的にスキップしたトピック

なし。全5トピックを処理した（hooks.md は既存に追記）。

## 抽出方針の備考

- hooks.md の既存2ルール（useEffect でのデータ取得禁止、派生状態の計算）は保持し追記のみ
- state-management: 状態コロケーション・useReducer・URL State・サーバー/クライアント分離
- rendering: memo/useMemo/useCallback の過剰使用を避ける方針を重視
- patterns: Hooks 時代のコンポジション重視のパターン
- suspense: React 19 の `use` API を含む最新情報
