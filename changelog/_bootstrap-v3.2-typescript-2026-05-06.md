# Bootstrap v3.2: typescript (2026-05-06)

## 結果サマリ
- 既存ルール強化: 3件
- 新規ルール追加: 1件
- 全段階失敗でスキップしたソース: 0件

## 公式ドキュメント取得統計

| リポジトリ | 候補1(raw) | 候補2(raw) | 候補3(raw) | Search API | 結果 |
|---|---|---|---|---|---|
| microsoft/TypeScript-Website | ✅ 200 (Everyday Types.md) | ✅ 200 (Narrowing.md) | - | - | 候補1+2で成功 |
| microsoft/TypeScript | - | - | - | - | スキップ（候補1,2で十分） |

## コミュニティ取得統計

| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 結果 |
|---|---|---|---|---|---|---|
| Zenn typescript | ✅ 200 | - | - | - | - | 段階1で成功 |
| Qiita typescript | ✅ 200 | - | - | - | - | 段階1で成功（記事4件、TypeScript固有の新規知見なし） |
| dev.to typescript | ✅ 200 | - | - | - | - | 段階1で成功（記事7件、TypeScript固有の新規知見なし） |

## 個別記事取得統計

| URL | 取得経路 | 結果 |
|---|---|---|
| https://zenn.dev/shusaku009/articles/83a16123e8805d | 段階A 直接WebFetch ✅ 200 | 既存rules確認（generics知見）、新規なし |

## 抽出されたルール

### 既存ルール強化

1. `practices/typescript/inference.md` - ルール1「推論できる型は明示的に書かない」
   - 追加根拠: Everyday Types.md でコンテキスト型付けが明示的に解説。forEach/map/filter のコールバック引数は原則注釈不要。
   - 出典: microsoft/TypeScript-Website Everyday Types.md ※2026-05-06に実際にfetch成功

2. `practices/typescript/inference.md` - ルール4「型ガードで unknown / Union 型を安全に絞り込む」
   - 追加根拠: Narrowing.md が typeof/in/instanceof/型述語を網羅。shared utility (`typeGuards.ts`) 推奨パターン確認。
   - 出典: microsoft/TypeScript-Website Narrowing.md ※2026-05-06に実際にfetch成功

3. `practices/typescript/type-design.md` - ルール2「Union 型で網羅的な分岐を強制する（Discriminated Union）」
   - 追加根拠: Narrowing.md がDiscriminated Union + never exhaustiveness check を公式推奨パターンとして文書化。
   - 出典: microsoft/TypeScript-Website Narrowing.md ※2026-05-06に実際にfetch成功

### 新規ルール

1. `practices/typescript/inference.md` - ルール5「Array.filter に型述語を渡して絞り込み結果の型を正確にする」
   - TypeScript 5.4 以前では filter コールバックの型絞り込みは配列戻り値型に反映されない。型述語 `(x): x is T` を使うことで正確な型を得る。
   - 出典: microsoft/TypeScript-Website Narrowing.md ※2026-05-06に実際にfetch成功

## 参照した記事一覧

| URL | 取得経路 | 抽出形式 |
|---|---|---|
| https://raw.githubusercontent.com/microsoft/TypeScript-Website/v2/packages/documentation/copy/en/handbook-v2/Everyday%20Types.md | raw.githubusercontent.com 直接 ✅ 200 | 公式ドキュメント / 既存ルール追加根拠 + 新規ルール |
| https://raw.githubusercontent.com/microsoft/TypeScript-Website/v2/packages/documentation/copy/en/handbook-v2/Narrowing.md | raw.githubusercontent.com 直接 ✅ 200 | 公式ドキュメント / 既存ルール追加根拠 + 新規ルール |
| https://zenn.dev/topics/typescript/feed | Zenn RSS 直接 ✅ 200 | フィード一覧取得 |
| https://qiita.com/tags/typescript/feed | Qiita RSS 直接 ✅ 200 | フィード一覧取得（4件、新規知見なし） |
| https://dev.to/feed/tag/typescript | dev.to RSS 直接 ✅ 200 | フィード一覧取得（7件、新規知見なし） |
| https://zenn.dev/shusaku009/articles/83a16123e8805d | 直接WebFetch ✅ 200 | 個別記事 / ジェネリクス知見確認（既存ルール補強のみ） |

## アクセス経路統計サマリ

| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 最終結果 |
|---|---|---|---|---|---|---|
| Zenn typescript | ✅ 200 | - | - | - | - | 段階1で成功 |
| Qiita typescript | ✅ 200 | - | - | - | - | 段階1で成功 |
| dev.to typescript | ✅ 200 | - | - | - | - | 段階1で成功 |

## 公式ドキュメント経路統計

| リポジトリ | 候補1(raw) | 候補2(raw) | 候補3(raw) | Search API | 結果 |
|---|---|---|---|---|---|
| microsoft/TypeScript-Website (Everyday Types) | ✅ 200 | - | - | - | 候補1で成功 |
| microsoft/TypeScript-Website (Narrowing) | - | ✅ 200 | - | - | 候補2で成功 |

合計成功: 5/5 (100%)
