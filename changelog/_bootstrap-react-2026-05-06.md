# Bootstrap v2: React カテゴリ補強ルール

**実行日**: 2026-05-06  
**ブランチ**: `bootstrap/react-2026-05-06`  
**担当**: フロントエンド技術キュレーター（Claude / Sonnet 4.6）

## 追加ルール一覧

### hooks.md (+2件)

| # | ルール | バージョン | 確信度 |
|---|--------|-----------|--------|
| 5 | ユーザー操作に起因する処理は `useEffect` ではなくイベントハンドラーに置く | React 18+ | 高 |
| 6 | `useEffectEvent` でエフェクトの「非反応的なロジック」を分離する | React 19+ | 高 |

### rendering.md (+2件)

| # | ルール | バージョン | 確信度 |
|---|--------|-----------|--------|
| 5 | `startTransition` / `useTransition` で緊急度の低い更新を後回しにする | React 18+ | 高 |
| 6 | `useDeferredValue` で重い子コンポーネントの再レンダリングを遅延させる | React 18+ | 高 |

### patterns.md (+3件)

| # | ルール | バージョン | 確信度 |
|---|--------|-----------|--------|
| 5 | Server Components と Client Components の境界を意識して設計する | React 19+, Next.js 14+ | 高 |
| 6 | `"use client"` ディレクティブは境界の葉ノードに最小化し、Serverコンポーネントを Props で貫通させる | React 19+, Next.js 14+ | 高 |

## 合計

- **追加ルール数**: 7件
- **変更ファイル数**: 3ファイル（hooks.md, rendering.md, patterns.md）
- **新規ファイル**: 1件（このchangelogファイル）

## 情報源・アクセス経路

| ソース | 取得方法 | 結果 |
|--------|---------|------|
| `reactjs/react.dev` (GitHub MCP) | `mcp__github__get_file_contents` | 失敗（セッション制限） |
| `react.dev/learn/you-might-not-need-an-effect` | WebFetch (rawgithubusercontent) | 成功 |
| `react.dev/learn/separating-events-from-effects` | WebFetch (rawgithubusercontent) | 成功 |
| `react.dev/reference/react/use` | WebFetch (rawgithubusercontent) | 成功 |
| `react.dev` 各URL | WebFetch (直接) | 失敗（403） |

## 重複チェック

- `hooks.md` 既存ルール1-4を確認し、ルール5（イベントハンドラー優先）はuseEffectでデータ取得しないというルール1と趣旨が異なる「副作用のトリガー判定」を扱うため追加
- `hooks.md` ルール6（useEffectEvent）はルール3（依存配列）と補完関係にあり、React 19+ の新機能として追加
- `rendering.md` 既存ルール1-4は memo/useMemo/key/batching を扱い、Transition/Deferredは未カバーのため追加
- `patterns.md` 既存ルール1-4は Composition/CustomHooks/SingleResponsibility/RenderProps を扱い、RSC境界は未カバーのため追加
