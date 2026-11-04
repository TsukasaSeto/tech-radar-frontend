# Practice Update 2026-11-04（採用0件）

## サマリ
- 新規ルール: 0件 / 強化ルール: 0件
- 窓内候補 403（重複除外後 361）/ タイトル除外 357 / 本文取得 4 / 未読 0（本文取得した 4 件は全て採用基準未達）
- 取得失敗ソース数: 0（Netflix・OWASP は既知の段階1スキップ）

## 取得統計（定義 76 / 試行 76 / 成功 75 / 失敗 1 / 未試行 0）

### 失敗・スキップのみ記録
| ソース | 失敗段階 | エラー詳細 |
|---|---|---|
| Stripe Engineering | 一覧は 200 だが記事URL抽出 0 件 | `stripe.dev/blog/<slug>` パターンに一致する href なし（HTML 構造が想定と異なる可能性。未検証事項） |

成功サマリ: 段階1=73, 段階2=2（Netflix, OWASP: 既知スキップ表どおり）, 段階3=0, 段階4=0, 段階5=1（Stripe 一覧 200）

- 36h 窓内に打ち切りの疑い（全件窓内）: dev.to 11 / Medium 9 / Qiita 5 / Zenn 3 フィード
- Netflix: 最新 2026-10-26 で窓内なし。OWASP: commits atom のみ（記事ではない）で対象なし
- Linear: 窓内 0 件

## 採用却下した記事（本文取得 4 件）
| URL | 却下理由 |
|---|---|
| https://zenn.dev/yukina200/articles/nextjs-cache-components-errors-12 | 個人単独記事（確信度 低）。`cacheComponents` 移行は nextjs/caching.md Rule #6〜#8 が既にカバー。B 強化に足る新規 WHY なし |
| https://zenn.dev/muuuuuuuuuuu/articles/sri-cdn-tamper-test | 個人単独記事。security/dependency-security.md Rule #9（SRI）と同主題 |
| https://zenn.dev/asumiminori/articles/claude-code-hooks-harness | 個人単独記事。ai-agent/claude-code.md の hooks 系既存ルールと同主題。公式照合なしでは強化せず |
| https://vercel.com/changelog/vercel-ci-no-longer-requires-a-linked-vercel-project | 機能告知で行動可能なプラクティスではない（D） |

## 参照した記事一覧
上記 4 件（全て D）。
