# Bootstrap v3.1: react (2026-05-06)

## 概要
v2 から v3.1 への補強実行。実際にfetchした記事からのみルールを抽出・補強する。

## 結果サマリ
- 既存ルール強化: 2件
- 新規ルール追加: 0件
- 既存ルール修正: 0件
- 全段階失敗でスキップしたソース: 0件

## アクセス経路統計

### 公式ドキュメント
| リポジトリ | 探索 | raw取得 | 直接 | 結果 |
|---|---|---|---|---|
| reactjs/react.dev | ❌ 403 (アクセス拒否) | - | - | 全失敗・スキップ |

### コミュニティ・テックブログ
| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 結果 |
|---|---|---|---|---|---|---|
| Zenn react | ✅ 200 | - | - | - | - | 段階1で成功 |
| dev.to react | ✅ 200 | - | - | - | - | 段階1で成功 |

合計成功: 2/3 (67%)
段階別成功数: 段階1=2
HTTPコード別失敗: 403=1

### 個別記事取得
| URL | 段階A | 段階B | 段階C | 結果 |
|---|---|---|---|---|
| https://zenn.dev/yasuhikasa/articles/48d93374b240db | ✅ 200 | - | - | 段階Aで成功 |
| https://zenn.dev/yuitonn/articles/5018829bfc6205 | ✅ 200 | - | - | 段階Aで成功 |

## 全段階失敗したソース（スキップ）
- reactjs/react.dev (GitHub MCP): ❌ 403 アクセス拒否 — このセッションでは tsukasaseto/tech-radar-frontend のみがアクセス許可対象のため

## 取得記事からのルール評価

**Zenn react フィード** (段階1成功):
- Zustand導入記事: コミュニティでZustandが軽量クライアント状態管理ツールとして実践されていることを確認。`state-management.md` ルール4の追加根拠として活用。
- SWR記事: `useState` + `useEffect` を使わず `useSWR` でデータ取得するパターンが実証された。`hooks.md` ルール1 および `state-management.md` ルール4の追加根拠として活用。

**dev.to react フィード** (段階1成功): ナビゲーション、フォルダ構造に関する記事。Reactの核心プラクティス強化には不適。

## 抽出されたルール

### 既存ルール強化
1. `practices/react/hooks.md` - ルール1「useEffect でデータ取得しない」
   - 追加根拠: [SWR記事](https://zenn.dev/yuitonn/articles/5018829bfc6205) (Zenn / 2026-05-04) ※2026-05-06に実際にfetch成功
   - SWRの `useSWR` 一行でデータ取得・エラー・ローディングを全て控えることを実証
   - `isMutating` による二重送信防止パターンも実践されていることを追加記載

2. `practices/react/state-management.md` - ルール4「グローバル状態はサーバー状態・クライアント状態で分離」
   - 追加根拠: [Zustand記事](https://zenn.dev/yasuhikasa/articles/48d93374b240db) (Zenn / 2026-05-06) + [SWR記事](https://zenn.dev/yuitonn/articles/5018829bfc6205) (Zenn / 2026-05-04) ※全て2026-05-06に実際にfetch成功
   - サーバー状態にSWR、クライアント状態にZustandという分離パターンがコミュニティで実践されていることを実証

### 新規ルール
なし（fetch成功した記事から新規追加に値するプラクティスは抽出できなかった）

## 参照した記事一覧
| URL | 取得経路 | 抽出形式 |
|---|---|---|
| https://zenn.dev/topics/react/feed | 段階1 (RSS直接) | フィード確認 | 
| https://dev.to/feed/tag/react | 段階1 (RSS直接) | フィード確認 |
| https://zenn.dev/yasuhikasa/articles/48d93374b240db | 段階A (直接WebFetch) | 既存ルール強化 |
| https://zenn.dev/yuitonn/articles/5018829bfc6205 | 段階A (直接WebFetch) | 既存ルール強化 |
