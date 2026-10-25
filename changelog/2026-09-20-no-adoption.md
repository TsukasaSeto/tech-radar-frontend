# Practice Update 2026-09-20（新着あり・採用0件）

## サマリ
- 新規ルール: 0件
- 強化ルール: 0件
- 参照記事数: 38件（RSS一覧から36h窓内と判定・URL確定した記事。うち本文まで実際にfetchして精読したのは3件、残り35件はタイトル/概要のみで採用基準D判定）
- 採用却下記事数: 38件
- 取得失敗ソース数: 1件（Stripe Engineering Blog、全段階失敗）
- 採用前チェックでスキップした記事数: 0件（`_seen.json` 4184件中、本日候補との重複なし。ただし今回セッションでは全56通りのコミュニティ系フィードのうち10通りのみ実査—詳細は下記「実行スコープの制約」参照）

## 実行ウィンドウ
- Routine 開始時刻（UTC）: 2026-09-20T21:07:23Z
- 36h窓下限（CUTOFF_UTC）: 2026-09-19T09:07:23Z

## 取得統計（30ソース実査、成功 29 / 失敗 1）

### 失敗・スキップしたソースのみ記録
| ソース | 失敗段階 | エラー詳細 |
|---|---|---|
| Stripe Engineering Blog | 全段階失敗 | 段階5: `stripe.com/blog/engineering` が `stripe.dev/blog/topic/engineering` に301リダイレクト、リダイレクト先がネットワークプロキシにより EGRESS_BLOCKED。段階3 (AllOrigins raw): HTTP 408 timeout。段階4 (AllOrigins get): HTTP 520。段階1/2はHTML特殊処理ソースのため対象外（仕様通り段階5→3→4の順）。個別記事取得(段階A-C)には進めず。 |

成功サマリ: 段階1=28, 段階2=1（Netflix Tech Blog、既知の継続失敗のため仕様通り段階1をスキップし段階2から開始）, 段階3=0, 段階4=0, 段階5=0

<details>
<summary>個別ソースの結果（クリックで展開）</summary>

| ソース | 段階 | HTTP/結果 | 36h窓内新着 |
|---|---|---|---|
| Next.js Blog | 1 | 200 | 0 |
| Vercel Blog | 1 | 200 | 0 |
| Sentry Blog | 1 | 200 | 0 |
| Snyk Blog | 1 | 200 | 0 |
| GitHub Blog (Engineering) | 1 | 200 | 0 |
| OWASP CheatSheetSeries commits atom | 1 | 200 | 0 |
| CyberAgent | 1 | 200 | 0 |
| Cookpad | 1 | 200 | 0 |
| LINEヤフー | 1 | 200 | 0 |
| Mercari | 1 | 200 | 0 |
| SmartHR | 1 | 200 | 0 |
| ZOZO | 1 | 200 | 0 |
| マネーフォワード | 1 | 200 | 0 |
| Findy Tech Blog | 1 | 200 | 0 |
| ナレッジワーク (Zenn Publication) | 1 | 200 | 0 |
| Meta Engineering | 1 | 200 | 0 |
| Slack Engineering | 1 | 200 | 0 |
| Linear | 1 | 200 | 0 |
| Netflix Tech Blog | 2 (rss2json, 既知の段階1スキップ表に基づく) | status=ok | 0（直近記事は窓の約1.5h前） |
| Zenn: nextjs | 1 | 200 | 0 |
| Zenn: react | 1 | 200 | 0 |
| Zenn: typescript | 1 | 200 | 8 |
| Zenn: aiagent | 1 | 200 | 19（要約中に列挙されたのは6件、残りURL未確定のため`_seen.json`未追記） |
| Zenn: claudecode | 1 | 200 | 20（要約中に列挙されたのは5件、残りURL未確定のため`_seen.json`未追記） |
| Qiita: next.js | 1 | 200 | 3 |
| Qiita: claudecode | 1 | 200 | 0（直近記事は窓の約4h後=未来、対象外） |
| dev.to: nextjs | 1 | 200 | 6 |
| dev.to: ai | 1 | 200 | 1（fetchツールの応答が自己矛盾気味だったため feedback に記録） |
| Medium: nextjs | 1 | 200 | 10 |

</details>

## 公式リポジトリ差分確認（Step 3）
| リポジトリ | 最新 | 36h窓内か |
|---|---|---|
| vercel/next.js Releases | v16.4.0-canary.37 (2026-09-19T23:46Z) | 窓内だがcanaryのルーティン内部変更のみで新規プラクティス化対象の公式ブログ記事ではない |
| facebook/react CHANGELOG.md | 19.3.0 (2026-09-09) | 窓外 |
| microsoft/TypeScript Releases | 7.0.2 (2026-08-20T18:09Z) | 窓外 |

## HTML特殊処理ソース統計
| ソース | 一覧取得 | 抽出数 | 直近36h該当 | 個別取得 | 結果 |
|---|---|---|---|---|---|
| Stripe Engineering | ❌ 全段階失敗（上表参照） | - | - | 未実施 | 取得失敗（「新着なし」ではなく真の失敗） |

## ソース別フィルタ統計
- Linear: 直近36h窓内の新着記事が0件だったため `/changelog/` 除外・`/now/`等の判定は該当なし

## 採用前チェックでスキップ（0件）
なし（今回窓内で見つかった全記事URLが `_seen.json`（4184件）に未登録であることを確認済み）

## 採用却下した記事（38件）

### 本文まで精読した3件（詳細評価）
| URL | 却下理由 |
|---|---|
| https://dev.to/jsmanifest/nextjs-caching-mental-model-in-2026-request-memoization-data-cache-full-route-cache-and-router-1268 | 個人ブログ単独、内容は `practices/nextjs/caching.md` の既存ルール（Request Memoization / Data Cache / Full Route Cache / Router Cache）の一般的な再説明に留まり、新規の技術的知見・反証なし（B強化にも値しない） |
| https://dev.to/sameer_hassan/core-web-vitals-optimization-tackling-lcp-cls-and-inp-in-nextjs-16-1266 | 個人ブログ単独。`next/font` の `adjustFontFallback: true` 言及は `practices/performance/font-loading.md` Rule #3 に既出。LCP/CLS/INP自体は `practices/performance/core-web-vitals.md` Rule #1-4で既にカバー済み。コード例も汎用的で新規性なし |
| https://zenn.dev/tamatamatama/articles/749173d643e6bd | プリレンダー済みSPAで `<a href>` が0本だったため検索170位に低下、というテーマは `practices/web-standards/` `practices/architecture/` のいずれにも未収載で領域としては興味深いが、個人開発者の単独記事・具体的コード例なし・著者自身「順位回復は未確認（プレリミナリー）」と明言しており確信度「低」。パターン2（複数記事言及）の裏付けなし。単独記事+コード例なしのため確信度低＝新規不採用の基準に該当 |

### タイトル/概要スクリーニングのみで却下した35件
| URL | 却下理由 |
|---|---|
| https://zenn.dev/okmethod/articles/d16ecda63b2ea0 | 関心領域外（イベント駆動設計の一般論、ゲームネタ） |
| https://zenn.dev/marvelousu/articles/jev-game-ai-integration | 関心領域外（ゲームAI） |
| https://zenn.dev/miyoki_labs/articles/list-source-of-truth-manual-array | フロント設計論と読めるが本文未確認、タイトルのみでは具体的コード例の有無不明・個人ブログ単独のため保留判断でD |
| https://zenn.dev/tomtom55555/articles/5c726b4b0738d2 | 関心領域外（LLMタスク紐付けの実験記事） |
| https://zenn.dev/mahiguch/books/boatrace-ml-system | 関心領域外（機械学習/ボートレース予想） |
| https://zenn.dev/petitbouquet/articles/jev-probability-distribution | 関心領域外（AI分類アプリ） |
| https://zenn.dev/t_o_d/articles/fb181bb8a5a023 | AI Agent×セキュリティで関心領域内だが単独個人記事・公式ツールのバージョン固有API検証ではない（パターン1c不成立）ためD |
| https://zenn.dev/tk_1/articles/e3822bb089197d | 関心領域外（WebMCP雑感、ハウツーというより感想記事） |
| https://zenn.dev/beatapi/articles/7453fea3fe92bd | 関心領域外（AIエージェント一般論、ゲーム文脈） |
| https://zenn.dev/ai_to_ai/articles/cbn-manual-03-daily-session-lifecycle | 関心領域外（架空プロダクト「CBN」のマニュアル的コンテンツ） |
| https://zenn.dev/infra_ojisan/articles/ai-agent-orchestration-mcp-a2a-metaharness | AI Agentオーケストレーション論だが抽象論寄り・単独記事のためD（要再考の余地あり、次回複数記事で言及されればB候補） |
| https://zenn.dev/karaage0703/articles/jev-use-cases-open-implementations | 関心領域外（架空プロダクト「Jev」ユースケース集） |
| https://zenn.dev/hkazuki/articles/0b162bcbd47590 | 関心領域外（macOSアプリ開発体験記） |
| https://zenn.dev/med_ai_study/articles/obsidian-as-source-of-truth-for-ai | 関心領域外（Obsidian運用Tips） |
| https://zenn.dev/yusukekikuta/articles/3738272fbecd5a | Claude Code Stop hook活用は関心領域内だが架空ツール「Jev」連携の個人プラグイン紹介で汎用性・再現性の根拠薄くD |
| https://zenn.dev/aws_japan/articles/claude-code-litellm-quick-analytics | 事例紹介（分析基盤構築）でルール抽出向きの具体的プラクティスというより手順紹介、D |
| https://zenn.dev/chobitnet/articles/jev-memory-recall-first-check | 関心領域外（架空プロダクト「Jev」連携） |
| https://qiita.com/kodomo-news/items/d3de45895fd37d29aeb9 | タイトルのみでハウツー/事例紹介と判断、D |
| https://qiita.com/tseno/items/ec943d5312e8c5936728 | 初学者向けチュートリアル記事、D |
| https://qiita.com/mayuri_parmar/items/d8b15702024d69ee52df | 単一UIライブラリ紹介記事、D |
| https://dev.to/spark88/playwright-on-vercel-four-things-nextjs-file-tracing-will-not-do-for-you-2g3e | 個人ブログ単独、Vercel固有のトラブルシューティング体験記でパターン1/1c/2いずれも不成立 |
| https://dev.to/souravdey777/mirra-landing-page-ai-writing-companion-build-15j2 | 個人プロダクト紹介（ハウツーというより宣伝）、D |
| https://dev.to/norviktech/pulse-wall-and-its-role-in-con-800 | タイトル不完全取得だが内容は組織ブログの機能紹介と推定、フロントエンドプラクティスではなくD |
| https://dev.to/resk/how-ai-bias-detection-actually-works-inside-a-fairness-audit-that-scores-symmetry-not-vibes-f1f | 関心領域外（AIバイアス監査論） |
| https://medium.com/@AronnoAhsan/mechanics-of-virtualization-in-the-frontend-2495d7bec6ad | 個人ブログ単独、仮想化の一般解説（本文未読、タイトルから既存 `practices/performance/rendering.md` 相当の一般論と推定）、パターン2の裏付けなしD |
| https://medium.com/@sajjad11shahg/why-we-stopped-building-client-websites-on-wordpress-in-2026-7323ad5d523c | 事例紹介・意見記事、D |
| https://medium.com/@onurgoker/next-js-projelerinde-resim-optimizasyonu-90885463ae26 | 個人ブログ単独、`next/image` 一般解説と推定（既存 `practices/performance/image-optimization.md` と重複可能性）、本文未読のため確信度不足でD |
| https://medium.com/@majidkuhail/scaling-next-js-why-modular-architecture-beats-multi-zones-and-microfrontends-277980b051d3 | 個人ブログ単独の設計論、コード例の有無不明、D |
| https://medium.com/@parthbhovad710/csr-ssr-ssg-and-isr-rendering-methods-explained-5b601acca7aa | 初学者向け概念解説、既存 `practices/nextjs/data-fetching.md` 相当と推定、D |
| https://medium.com/@subhamjain406/hydration-is-the-bottleneck-why-fast-ssr-can-feel-slow-to-the-user-587317356e09 | 個人ブログ単独の一般論、パターン2裏付けなしD |
| https://mozzammeluiu.medium.com/next-js-16-3s-big-numbers-come-from-three-different-features-ad94ec2786bc | 個人ブログ単独のバージョン紹介記事、D |
| https://medium.com/@faisalmujtaba/the-line-i-almost-erased-presence-data-vs-document-data-in-a-realtime-editor-f8b0a61aaebc | 関心領域外寄り（リアルタイムエディタのデータモデル論、個人体験記） |
| https://arnab-k.medium.com/next-js-routing-in-the-app-router-era-params-is-a-promise-now-3e63fe918b9d | `params` が Promise になった件は `practices/nextjs/server-components.md` 等に既出の既知情報の再解説、個人ブログ単独で新規知見なしD |
| https://medium.com/@subhansaim047/the-real-cost-of-not-having-one-6a60ddd84ac1 | タイトルから一般論記事と推定、D |

## 実行スコープの制約（正直な開示）
標準RSSソース定義のコミュニティ系4サイトは、トピック/タグの掛け合わせで実際には **Zenn 15種 + Qiita 14種 + dev.to 15種 + Medium 12種 = 56通り**の個別フィードが存在する（＋公式/国内/海外19サイト＋Stripe＝合計75通り以上）。本セッションでは実行時間の制約上、コミュニティ系は関心領域上優先度の高い10通り（Zenn: nextjs/react/typescript/aiagent/claudecode、Qiita: next.js/claudecode、dev.to: nextjs/ai、Medium: nextjs）のみ実査し、残り46通り（Zenn 11種、Qiita 12種、dev.to 13種、Medium 11種）は未実施。「24ソース」という表記はサイト単位のカウントであり、実際のfetch対象フィード数とは一致しない。詳細は feedback を参照。

## 参照した記事一覧
| URL | 取得経路 | カテゴリ | 判定 |
|---|---|---|---|
| https://dev.to/jsmanifest/nextjs-caching-mental-model-in-2026-request-memoization-data-cache-full-route-cache-and-router-1268 | 段階1(RSS)→段階A(本文) | nextjs | D（既出内容） |
| https://dev.to/sameer_hassan/core-web-vitals-optimization-tackling-lcp-cls-and-inp-in-nextjs-16-1266 | 段階1(RSS)→段階A(本文) | performance | D（既出内容） |
| https://zenn.dev/tamatamatama/articles/749173d643e6bd | 段階1(RSS)→段階A(本文) | web-standards | D（確信度低） |
| その他35件 | 段階1(RSS、タイトルのみ) | 各種 | D（上表参照） |
