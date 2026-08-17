# Practice Update 2026-08-17 (no adoption)

## サマリ
- 新規ルール: 0件
- 強化ルール: 0件
- 参照記事数（全文読了）: 3件
- 採用却下記事数: 3件
- 取得失敗ソース数: 1件（Stripe Engineering Blog）
- 時間制約により未着手のソース: 25件（定義済み76ソース中51ソースに着手）
- 採用前チェックでスキップした記事数: 0件

## 取得統計（76ソース、着手51 / 未着手25 / 失敗1）

### 失敗・スキップしたソースのみ記録
| ソース | 失敗段階 | エラー詳細 |
|---|---|---|
| Stripe Engineering Blog | 全段階失敗 | 段階5: `https://stripe.com/blog/engineering` が `https://stripe.dev/blog/topic/engineering` へ301リダイレクト、リダイレクト先が許可ドメイン外で `EGRESS_BLOCKED`。フォールバック段階（AllOrigins raw/get）は本ソースに定義されていないため未実施。2026-08-13, 08-15, 08-16 に続き4日連続の再現 |
| Zenn: web, accessibility, graphql, i18n, githubactions, monorepo（6） | 時間制約・未試行 | 76ソース中、優先度の高いソース（公式・国内・海外・core community topics）を先に着手した結果、ターン予算内で未着手 |
| Qiita: accessibility, performance, graphql, i18n, githubactions, monorepo（6） | 時間制約・未試行 | 同上 |
| dev.to: webdev, accessibility, graphql, i18n, githubactions, monorepo, performance（7） | 時間制約・未試行 | 同上 |
| Medium: performance, css, accessibility, i18n, devops, monorepo（6） | 時間制約・未試行 | 同上 |

成功サマリ: 段階1=48, 段階2=2, 段階3=0, 段階4=0, 段階5=0（失敗1、時間制約未試行25）

<details>
<summary>個別ソースの結果（クリックで展開）</summary>

| ソース | 段階 | 結果 |
|---|---|---|
| Next.js Blog | 1 | 200・36h窓内0件 |
| Vercel Blog | 1 | 200・2件（いずれもchangelog性の告知） |
| Sentry Blog | 1 | 200・36h窓内0件 |
| Snyk Blog | 1 | 200・36h窓内0件 |
| GitHub Blog Engineering | 1 | 200・36h窓内0件 |
| OWASP CheatSheetSeries commits | 1 | 200・1件（Logging Cheat Sheetのvendorリンク削除、housekeeping） |
| CyberAgent | 1 | 200・1件（iOS MetricKitトピック、対象領域外） |
| Cookpad | 1 | 200・36h窓内0件 |
| LINEヤフー | 1 | 200・36h窓内0件 |
| Mercari | 2（段階1は既知失敗のため最初からスキップ） | rss2json 200・1件（TiDB、バックエンド、対象領域外） |
| SmartHR | 1 | 200・1件（フロントエンドSandboxテスト、全文読了） |
| ZOZO | 1 | 200・1件（カンファレンス参加レポート、事例紹介のため対象外） |
| マネーフォワード | 1 | 200・36h窓内0件 |
| Findy Tech Blog | 1 | 200・1件（カンファレンスUI改善事例、事例紹介のため対象外） |
| ナレッジワーク | 1 | 200・36h窓内0件 |
| Meta Engineering | 1 | 200・36h窓内0件 |
| Netflix Tech Blog | 2（段階1は既知失敗のため最初からスキップ） | rss2json 200・36h窓内0件 |
| Slack Engineering | 1 | 200・36h窓内0件 |
| Linear | 1 | 200・36h窓内0件（changelog除外0件含む） |
| Zenn: nextjs/react/typescript/testing/performance/security/aiagent/claudecode/css（9） | 1 | いずれも200、多数記事あり（個人開発記録・ポエム中心） |
| Qiita: next.js/react/typescript/jest/playwright/claudecode/security/css（8） | 1 | いずれも200 |
| dev.to: nextjs/react/typescript/testing/security/css/devops/ai（8） | 1 | いずれも200（`ai`タグはスパム多数、指示通り厳格判定） |
| Medium: nextjs/react/typescript/frontend/testing/security（6） | 1 | いずれも200（個人ブログ中心、公式性なし） |

</details>

## HTML特殊処理ソース統計
| ソース | 一覧取得 | 抽出数 | 直近36h該当 | 個別取得 | 結果 |
|---|---|---|---|---|---|
| Stripe Engineering | ❌ EGRESS_BLOCKED | - | - | - | 取得失敗（4日連続） |

## ソース別フィルタ統計
- Linear: 36h窓内の記事自体が0件のため、`/changelog/` フィルタの適用対象なし

## 採用前チェックでスキップ（0件）
（該当なし）

## 全文読了して採用却下した記事（3件）

| URL | 却下理由 |
|---|---|
| [＠ファイル指定はpath指定より速いという事実](https://zenn.dev/secula/articles/ccd31dd1035417) | Anthropic公式ブログの内容を引用・紹介するのみで著者自身の検証・ベンチマークがなく、単なるTipsレベル（採用基準D）。加えて内容は「Readコール1回節約」という細部Tipsで、確信度低（コード例は`@utils.test.ts`という記法紹介のみ、設定ファイル/APIの直接提示ではないためパターン1c不成立） |
| [Sandboxテスト 〜フロントエンドとAI時代の変化の激しさへの挑戦〜](https://tech.smarthr.jp/entry/2026/08/17/080914) | SmartHR独自の呼称による社内プラクティス（Playwright + Prism(OpenAPIモック)によるフロントエンド結合テスト）。コード例はあるが単一記事・非公式ブログのため、パターン2（複数著者の言及）の条件を満たさず却下。将来同趣旨の記事が見つかれば強化候補として再評価する |
| [Remove vendor links and keep authoritative references from Logging Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/commit/8df01a5ecf58b3e0489b75b07d40d567d477f464) | OWASP公式コミットだが、内容はvendorリンクの削除のみでロギングに関する新規ガイダンスの追加はなし（housekeeping、採用基準D） |

## 見出しのみ確認し全文取得を見送った記事について

上記3件以外は、フィード一覧のタイトル・投稿日時のみで関心領域外（事例紹介・個人開発日記・スパム・オフトピック）と判定し、全文取得を行っていない。該当URLは重複読み込み防止のため `_seen.json` に追加済み。

## 参照した記事一覧（全文読了、3件）

| URL | 取得経路 | カテゴリ | 判定 |
|---|---|---|---|
| https://zenn.dev/secula/articles/ccd31dd1035417 | 段階A | ai-agent | D: 抽出対象外 |
| https://tech.smarthr.jp/entry/2026/08/17/080914 | 段階A | testing | D: 抽出対象外（パターン2不成立） |
| https://github.com/OWASP/CheatSheetSeries/commit/8df01a5ecf58b3e0489b75b07d40d567d477f464 | 段階1 | security | D: 抽出対象外（housekeeping） |
