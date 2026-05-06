# Bootstrap v3.1: nextjs (2026-05-06)

## 概要
v2 から v3.1 への補強実行。実際にfetchした記事からのみルールを抽出・補強する。

## 結果サマリ
- 既存ルール強化: 1件
- 新規ルール追加: 0件
- 既存ルール修正: 0件
- 全段階失敗でスキップしたソース: 1件（vercel/next.js GitHub MCP アクセス拒否）

## アクセス経路統計

### 公式ドキュメント
| リポジトリ | 探索 | raw取得 | 直接 | 結果 |
|---|---|---|---|---|
| vercel/next.js | ❌ 403 (アクセス拒否) | - | - | 全失敗・スキップ |

### コミュニティ・テックブログ
| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 結果 |
|---|---|---|---|---|---|---|
| Zenn nextjs | ✅ 200 | - | - | - | - | 段階1で成功 |
| dev.to nextjs | ✅ 200 | - | - | - | - | 段階1で成功 |

合計成功: 2/3 (67%)
段階別成功数: 段階1=2
HTTPコード別失敗: 403=1

### 個別記事取得
| URL | 段階A | 段階B | 結果 |
|---|---|---|---|
| https://zenn.dev/keisuke58/articles/nextjs-ssg-seo-100pages | ✅ 200 | - | 段階Aで成功 |
| https://dev.to/dattasble/how-i-engineered-a-06s-lcp-and-a-perfect-100100-gtmetrix-score-nextjs-optimization-guide-1j24 | ✅ 200 | - | 段階Aで成功 |

## 全段階失敗したソース（スキップ）
- vercel/next.js (GitHub MCP): ❌ 403 アクセス拒否 — このセッションでは tsukasaseto/tech-radar-frontend のみがアクセス許可対象のため

## 取得記事からのルール評価

**Zenn nextjs フィード** (段階1成功):
- SSG+SEO記事: generateStaticParams に TypeScript 定数配列を使えること、`"use client"` と `metadata` の共存制限、SEO3点セットが `routing.md` ルール2の追加根拠に適すると判断。

**dev.to nextjs フィード** (段階1成功):
- 最適化記事 (GTmetrix 100/100): サードパーティスクリプト遅延ロード、next/font最適化、Edge MiddlewareによるTTFB改善。既存のnextjsルールとの直接対応はないが、樋效果データを属性履歴に記録。

## 抽出されたルール

### 既存ルール強化
1. `practices/nextjs/routing.md` - ルール2「動的ルートと generateStaticParams でSSGを活用」
   - 追加根拠: [SSG+SEO記事](https://zenn.dev/keisuke58/articles/nextjs-ssg-seo-100pages) (Zenn / 2026-05-05) ※2026-05-06に実際にfetch成功
   - TypeScript定数配列をソースにするパターンの確認
   - `"use client"` と `metadata` の共存不可制約の実践確認
   - SEO3点セット (Metadata API + sitemap.ts + robots.ts) の有効性を実証

### 新規ルール
なし（fetch成功した記事から新規追加に値するプラクティスは抽出できなかった）

## 参照した記事一覧
| URL | 取得経路 | 抽出形式 |
|---|---|---|
| https://zenn.dev/topics/nextjs/feed | 段階1 (RSS直接) | フィード確認 |
| https://dev.to/feed/tag/nextjs | 段階1 (RSS直接) | フィード確認 |
| https://zenn.dev/keisuke58/articles/nextjs-ssg-seo-100pages | 段階A (直接WebFetch) | 既存ルール強化 |
| https://dev.to/dattasble/how-i-engineered-a-06s-lcp-and-a-perfect-100100-gtmetrix-score-nextjs-optimization-guide-1j24 | 段階A (直接WebFetch) | 履歴記録のみ |
