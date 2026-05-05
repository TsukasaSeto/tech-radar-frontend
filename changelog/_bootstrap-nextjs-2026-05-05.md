# Bootstrap Changelog: Next.js カテゴリ

**実行日**: 2026-05-05
**ブランチ**: bootstrap/nextjs-2026-05-05
**処理カテゴリ**: nextjs

## 作成ファイル

| ファイル | ルール数 | 内容 |
|---|---|---|
| `practices/nextjs/server-components.md` | 3 | Server Component デフォルト、use client境界のリーフ配置、シリアライズ可能props |
| `practices/nextjs/data-fetching.md` | 3 | Promise.all並列フェッチ、cacheオプション明示、Suspenseストリーミング |
| `practices/nextjs/routing.md` | 4 | App Routerファイル構造、generateStaticParams、Parallel/Intercepting Routes、LinkコンポーネントとuseRouter |
| `practices/nextjs/caching.md` | 4 | 4層キャッシュの理解、revalidateTag/revalidatePath、unstable_cache、dynamicエクスポート |
| `practices/nextjs/server-actions.md` | 4 | Server Actionsでのフォームミューテーション、useActionState、actionsファイルへの集約、useOptimistic楽観的更新 |
| `practices/nextjs/middleware.md` | 3 | Middlewareの用途限定、エッジ互掰JWT検証（jose）、セキュリティヘッダー付与 |

**合計**: 21 ルール / 6 トピック

## WebFetch 結果

全URLで 403 エラーが発生したため、モデルの学習知識からルールを作成した。

## 処理ノート

- Next.js 15 の `use cache` directive にも言及（`unstable_cache` との対比）
- Middleware はエッジランタイム制約（Node.js API 制限）に注意が必要なため明記
- 楽観的 UI 更新（`useOptimistic`）は React 19 / Next.js 14+ の機能として記載
