# Bootstrap v3.2: api-client (2026-05-06)

## 結果サマリ
- 既存ルール強化: 3件
- 新規ルール追加: 1件
- 全段階失敗でスキップしたソース: 0件

## 公式ドキュメント取得統計

| リポジトリ | 候補1(raw) | 候補2(raw) | 候補3(raw) | Search API | 結果 |
|---|---|---|---|---|---|
| TanStack/query (README.md) | ✅ 200 (README.md) | - | - | - | 高レベル概要のみ、詳細ルール抽出に候補2も使用 |
| TanStack/query (query-keys.md) | - | ✅ 200 (docs/framework/react/guides/query-keys.md) | - | - | 候補2で成功（オブジェクト順序決定論的ハッシュ・配列順序は重要確認） |

## コミュニティ取得統計

| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 結果 |
|---|---|---|---|---|---|---|
| Zenn reactquery | ✅ 200 | - | - | - | - | 段階1で成功 |

## 個別記事取得統計

| URL | 取得経路 | 結果 |
|---|---|---|
| https://zenn.dev/correlate_dev/articles/react-query-patterns | 段階A 直接WebFetch ✅ 200 | useSuspenseQuery・HydrationBoundary・initialPageParam必須・gcTime 2x設計抽出 |
| https://zenn.dev/lv/articles/214f22bbc6df17 | 段階A 直接WebFetch ✅ 200 | Zod v4 破壊的変更（z.email・z.flattenError・z.uuid RFC厳密化）・safeParse直接使用推奨 抽出 |

## 抽出されたルール

### 既存ルール強化

1. `practices/api-client/caching.md` - ルール1「TanStack Query と SWR の使い分けを理解する」
   - 追加根拠: v5で `useQuery` の `suspense: true` 廃止→`useSuspenseQuery` 正式化。`useInfiniteQuery` の `initialPageParam` 必須化（v4→v5破壊的変更）。gcTime はstaleTimeの約2倍を目安にする指針確認。
   - 出典: Zenn correlate_dev記事 ※2026-05-06に実際にfetch成功

2. `practices/api-client/caching.md` - ルール3「クエリキーはファクトリ関数で構造化する」
   - 追加根拠: 公式ドキュメントが「オブジェクト内プロパティ順序は無関係（決定論的ハッシュ）」「配列要素順序は区別する」という重要な細則を明示。
   - 出典: TanStack/query docs/query-keys.md ※2026-05-06に実際にfetch成功

3. `practices/api-client/type-safety.md` - ルール2「Zod でAPIレスポンスをランタイム検証する」
   - 追加根拠: Zod v4で `z.email()` / `z.flattenError()` / `z.uuid()` RFC厳密化等の破壊的変更。safeParse結果を型ガードにラップせず直接使う推奨パターン。アプリ境界で1回だけ検証するアーキテクチャ原則確認。
   - 出典: Zenn lv記事（TypeScript 5.5 + Zod v4） ※2026-05-06に実際にfetch成功

### 新規ルール

1. `practices/api-client/caching.md` - ルール5「Next.js App Router では HydrationBoundary でサーバーサイドプリフェッチを実装する」
   - 内容: Server ComponentでprefetchQuery → dehydrate() でシリアライズ → HydrationBoundary でクライアントに注入。React.cache()でQueryClientの重複生成防止。useSuspenseQueryと組み合わせて初期ローディング排除。
   - 出典: Zenn correlate_dev記事 ※2026-05-06に実際にfetch成功

## 参照した記事一覧

| URL | 取得経路 | 抽出形式 |
|---|---|---|
| https://raw.githubusercontent.com/TanStack/query/main/README.md | raw.githubusercontent.com 直接 ✅ 200 | 概要README（詳細ルール抽出に不十分） |
| https://raw.githubusercontent.com/TanStack/query/main/docs/framework/react/guides/query-keys.md | raw.githubusercontent.com 直接 ✅ 200 | 公式ドキュメント / 既存ルール追加根拠 |
| https://zenn.dev/topics/reactquery/feed | Zenn RSS 直接 ✅ 200 | フィード一覧取得 |
| https://zenn.dev/correlate_dev/articles/react-query-patterns | 直接WebFetch ✅ 200 | 個別記事 / TanStack Query v5実践パターン |
| https://zenn.dev/lv/articles/214f22bbc6df17 | 直接WebFetch ✅ 200 | 個別記事 / Zod v4破壊的変更 + TypeScript 5.5型ガード |

## アクセス経路統計サマリ

| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 最終結果 |
|---|---|---|---|---|---|---|
| Zenn reactquery | ✅ 200 | - | - | - | - | 段階1で成功 |

## 公式ドキュメント経路統計

| リポジトリ | 候補1(raw) | 候補2(raw) | 候補3(raw) | Search API | 結果 |
|---|---|---|---|---|---|
| TanStack/query (README.md) | ✅ 200 | - | - | - | 取得成功・概要のみのため候補2も使用 |
| TanStack/query (query-keys.md) | - | ✅ 200 | - | - | 候補2で成功・クエリキー細則確認 |

合計成功: 5/5 (100%)
