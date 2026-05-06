# Bootstrap v2: api-client (2026-05-06)

## 概要

api-client カテゴリに補強コンテンツ（v2）として5件のルールを追加。

## 追加ルール一覧

| ファイル | ルール# | タイトル |
|---|---|---|
| `rest.md` | 6 | TanStack Query の Query Key をファクトリパターンで管理する |
| `rest.md` | 7 | Optimistic Update は useMutation の onMutate / onError / onSettled で実装する |
| `type-safety.md` | 5 | openapi-typescript で OpenAPI spec から型を自動生成し、手動型定義を排除する |
| `type-safety.md` | 6 | Zod によるランタイムバリデーションと型推論を統合する（z.infer） |
| `error-handling.md` | 5 | HTTPステータスコード別の回復戦略を実装する |

## 変更ファイル

- `practices/api-client/rest.md` — ルール6, 7 を追加
- `practices/api-client/type-safety.md` — ルール5, 6 を追加
- `practices/api-client/error-handling.md` — ルール5 を追加
