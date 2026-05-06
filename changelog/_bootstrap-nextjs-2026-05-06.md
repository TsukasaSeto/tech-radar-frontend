# Bootstrap v2: Next.js カテゴリ補強 (2026-05-06)

## 概要

Next.js カテゴリに v2 として補強ルールを追加した。既存ルールとの重複を避け、実務で頻出するパターンや Next.js 15 以降の新機能を中心に選定した。

## 変更ファイル

### `practices/nextjs/caching.md` (+2 ルール)

| # | ルール | バージョン | 確信度 |
|---|--------|-----------|-------|
| 5 | `unstable_cache` と React `cache()` を目的に応じて使い分ける | Next.js 15+ | 高 |
| 6 | `use cache` ディレクティブ（Next.js 15 実験的機能）を将来の標準として把握する | Next.js 15+ | 中 |

### `practices/nextjs/routing.md` (+2 ルール)

| # | ルール | バージョン | 確信度 |
|---|--------|-----------|-------|
| 5 | Route Groups でURLに影響しないディレクトリ整理を行う | Next.js 15+ | 高 |
| 6 | Intercepting Routes のマッチング規則（`(.)` / `(..)` / `(...)`）を正しく使う | Next.js 15+ | 高 |

### `practices/nextjs/server-components.md` (+2 ルール)

| # | ルール | バージョン | 確信度 |
|---|--------|-----------|-------|
| 4 | `"use server"` と `"use client"` の境界を意識したコンポーネント設計をする | Next.js 15+ | 高 |
| 5 | Streaming with Suspense でページの初期表示を高速化する | Next.js 15+ | 高 |

## 合計

- 追加ルール数: **6件**
- 対象ファイル数: **3ファイル**

## アクセス経路

| ソース | 内容 |
|--------|------|
| GitHub MCP: `tsukasaseto/tech-radar-frontend` | 既存6ファイルの読み込みに成功 |
| GitHub MCP: `vercel/next.js` | アクセス権限なし（フォールバック） |
| WebFetch: `nextjs.org` | 403エラー（フォールバック） |
| WebSearch: `nextjs.org`, `react.dev`, `vercel.com` | キャッシュ・ルーティング情報取得に成功 |

## 参照出典

- [Next.js Docs: Caching](https://nextjs.org/docs/app/building-your-application/caching)
- [Next.js Docs: unstable_cache](https://nextjs.org/docs/app/api-reference/functions/unstable_cache)
- [Next.js Docs: use cache directive](https://nextjs.org/docs/app/api-reference/directives/use-cache)
- [Next.js Docs: Route Groups](https://nextjs.org/docs/app/building-your-application/routing/route-groups)
- [Next.js Docs: Intercepting Routes](https://nextjs.org/docs/app/api-reference/file-conventions/intercepting-routes)
- [Next.js Docs: Parallel Routes](https://nextjs.org/docs/app/api-reference/file-conventions/parallel-routes)
- [Next.js Docs: Composition Patterns](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns)
- [Next.js Docs: Loading UI and Streaming](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming)
- [React Docs: cache](https://react.dev/reference/react/cache)
