# Practice Update 2026-11-07 (no adoption)

## サマリ
- 新規ルール: 0件
- 強化ルール: 0件
- 参照記事数: 6件
- 採用却下記事数: 6件
- 取得失敗ソース数: 0件
- 採用前チェックでスキップした記事数: 0件
- 窓内候補 377（ユニーク 351）/ タイトル除外 345 / 本文取得 6 / 未読 345（_seen.json には追記しない）

## 取得統計（定義 76 / 試行 76 / 成功 76 / 失敗 0 / 未試行 0）

### 失敗・スキップのみ記録
| ソース | 失敗段階 | エラー詳細 |
|---|---|---|
| (なし) | - | - |

成功サマリ: 段階1=73, 段階2=2, 段階3=0, 段階4=0, 段階5=1
（Netflix / OWASP は既知スキップ表に従い段階2 (rss2json) から開始し 200/ok。Stripe は段階5 で 200）

窓: 2026-11-06T09:06:49Z 〜 2026-11-07T21:06:49Z（36h）

## HTML特殊処理ソース統計
| ソース | 一覧取得 | 抽出数 | 直近36h該当 | 個別取得 | 結果 |
|---|---|---|---|---|---|
| Stripe Engineering | ✅ 200 | 29件（`/blog/<slug>` 形式の相対 href） | 判定不可（一覧 HTML から公開日を抽出できず） | - | 新着確認不可 |

## ソース別フィルタ統計
- Linear: 窓内 0 件

## 追加・強化されたルール
なし。

## 採用却下した記事（6件）
| URL | 却下理由 |
|---|---|
| https://vercel.com/changelog/vercel-tracing-is-now-generally-available | 製品の GA 告知（機能紹介）。観測側の判断軸を示すベストプラクティスではない（D） |
| https://qiita.com/akihidem/items/80341b05abb47ea094e9 | 公式 docs 更新の二次要約。sandbox / permissions の該当内容は claude-code.md の既存ルール（deny は Read ツール等、`denyRead`・`allowedDomains` の範囲）が既にカバー（B 相当だが新規根拠なし） |
| https://zenn.dev/masaru21/articles/9ef0ec6558fe67 | CSP Report-Only 段階導入は security/csp.md #4 に既存。単一個人記事で新規根拠なし |
| https://zenn.dev/maura/articles/0a3f541be9f42d | CLAUDE.md の字数上限運用。個人の運用記録で単一記事（確信度 低） |
| https://zenn.dev/tomsolog/articles/20261104-claude-code-permissions-quickstart | `permissions.deny` で `.env` 保護は claude-code.md に既存。初心者向けハウツー |
| https://qiita.com/harupython/items/6dfcd3758b98576b29b0 | 個別言語プロジェクトの CI 分割事例。事例紹介（D） |

## 参照した記事一覧
| URL | 取得経路 | カテゴリ | 判定 |
|---|---|---|---|
| https://vercel.com/changelog/vercel-tracing-is-now-generally-available | 段階A | observability | D |
| https://qiita.com/akihidem/items/80341b05abb47ea094e9 | 段階A | ai-agent | D（既存重複） |
| https://zenn.dev/masaru21/articles/9ef0ec6558fe67 | 段階A | security | D（既存重複） |
| https://zenn.dev/maura/articles/0a3f541be9f42d | 段階A | ai-agent | D |
| https://zenn.dev/tomsolog/articles/20261104-claude-code-permissions-quickstart | 段階A | ai-agent | D（既存重複） |
| https://qiita.com/harupython/items/6dfcd3758b98576b29b0 | 段階A | ci-cd | D |
