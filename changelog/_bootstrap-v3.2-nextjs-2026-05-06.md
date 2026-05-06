# Bootstrap v3.2: nextjs (2026-05-06)

## 結果サマリ
- 既存ルール強化: 1件
- 新規ルール追加: 1件
- 全段階失敗でスキップしたソース: 2件（公式リポジトリの候補2件はアプリ開発向けでなかった）

## 公式ドキュメント取得統計

| リポジトリ | 候補1(raw) | 候補2(raw) | 候補3(raw) | Search API | 結果 |
|---|---|---|---|---|---|
| vercel/next.js | ✅ 200 (AGENTS.md) | ✅ 200 (packages/next/README.md) | - | - | 取得成功だがコントリビュータ/マーケティング用途のみ。アプリ開発ルール抽出不可でスキップ |

## コミュニティ取得統計

| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 結果 |
|---|---|---|---|---|---|---|
| Zenn nextjs | ✅ 200 | - | - | - | - | 段階1で成功 |
| Qiita next.js | ✅ 200 | - | - | - | - | 段階1で成功（4件、新規知見なし） |

## 個別記事取得統計

| URL | 取得経路 | 結果 |
|---|---|---|
| https://zenn.dev/keisuke58/articles/nextjs-ssg-seo-100pages | 段階A 直接WebFetch ✅ 200 | generateStaticParams + SEO三点セット パターン抽出 |

## 抽出されたルール

### 既存ルール強化

1. `practices/nextjs/routing.md` - ルール2「動的ルートと generateStaticParams でSSGを活用する」
   - 追加根拠: Zenn実事例で300ページ以上のSSGを generateStaticParams で実装。dynamicParams = false パターン確認。TypeScript定数によるデータ管理の有効性実証。
   - 出典: Zenn keisuke58 SSG記事 ※2026-05-06に実際にfetch成功

### 新規ルール

1. `practices/nextjs/routing.md` - ルール7「`generateMetadata` + `sitemap.ts` + `robots.ts` の SEO 三点セット」
   - App Router の SEO は generateMetadata（動的メタデータ）+ sitemap.ts + robots.ts の三点セットで設定。'use client' ページは layout.tsx に metadata を委譲する。
   - 出典: Zenn keisuke58 SSG記事 ※2026-05-06に実際にfetch成功

## 参照した記事一覧

| URL | 取得経路 | 抽出形式 |
|---|---|---|
| https://raw.githubusercontent.com/vercel/next.js/canary/AGENTS.md | raw.githubusercontent.com 直接 ✅ 200 | コントリビュータガイド（アプリ開発ルール抽出不可） |
| https://raw.githubusercontent.com/vercel/next.js/canary/packages/next/README.md | raw.githubusercontent.com 直接 ✅ 200 | マーケティングREADME（アプリ開発ルール抽出不可） |
| https://zenn.dev/topics/nextjs/feed | Zenn RSS 直接 ✅ 200 | フィード一覧取得 |
| https://qiita.com/tags/next.js/feed | Qiita RSS 直接 ✅ 200 | フィード一覧取得（4件、新規知見なし） |
| https://zenn.dev/keisuke58/articles/nextjs-ssg-seo-100pages | 直接WebFetch ✅ 200 | 個別記事 / SSG + SEO パターン抽出 |

## アクセス経路統計サマリ

| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 最終結果 |
|---|---|---|---|---|---|---|
| Zenn nextjs | ✅ 200 | - | - | - | - | 段階1で成功 |
| Qiita next.js | ✅ 200 | - | - | - | - | 段階1で成功 |

## 公式ドキュメント経路統計

| リポジトリ | 候補1(raw) | 候補2(raw) | 候補3(raw) | Search API | 結果 |
|---|---|---|---|---|---|
| vercel/next.js (AGENTS.md) | ✅ 200 | - | - | - | 取得成功・内容はコントリビュータ用のためスキップ |
| vercel/next.js (README.md) | - | ✅ 200 | - | - | 取得成功・内容はマーケティング用のためスキップ |

合計成功: 5/5 (100%) ※公式リポジトリは取得成功も内容がアプリ開発向けでなかった
