# Claude Code 活用のベストプラクティス

## ルール

### 1. Claude Code をチームで活用するために、ルール管理と失敗からの学習を体系化する

個人の Claude Code セッション知識をチーム全体で共有・改善する仕組みを構築する。
失敗は「起きた瞬間に直す」だけでは不十分であり、次のセッションのエージェントが参照できる形に残すことが再発防止の鍵となる。

**根拠**:
- 個々のセッションの学びは、組織共有ルールに昇格しなければ再現できない
- 同じ失敗を別のセッション・チームメンバーが繰り返さないには、パターン検出と文書化が必要
- ルールは「なければ Claude が間違える」ものだけを追加し、ノイズを排除する

**アプローチ1: ルール配布システム（SonicGarden方式）**

組織ルールを3段階で管理する:
1. **extract**: 各プロジェクトのルールを抽出
2. **merge**: 複数プロジェクトに共通するパターン（出現率 50% 以上）を組織ルールに昇格
3. **apply**: 組織ルールを新規・既存プロジェクトに配布（スタック検出・競合解決付き）

ファイルを3種に分類:
- `.md` — 組織全体のポータブル原則（更新で上書き）
- `.local.md` — プロジェクト固有カスタマイズ（更新時も保持）
- `.examples.md` — 参照用コード例

**アプローチ2: 失敗→ルール変換パイプライン（dely方式）**

失敗を段階的にチームルールに昇格させる:
1. **蓄積**: エージェントの失敗（NEEDS_REVISION / GENERATOR_FAILED）を `FAILURES.md` に記録（git-ignore）
2. **昇格**: 同一パターンが3回以上発生したら `/failure-promote` スキルで `.claude/rules/{project}/` へのPRを自動作成
3. **振り返り**: `/retro` スキルがコミット履歴・PRコメント・手動編集を5カテゴリ・3優先度に分類して学習を整理

**コード例**:
```sh
# FAILURES.md (gitignore 対象 — 個人蓄積フェーズ)
## 2026-05-07 Session
- NEEDS_REVISION: import 順序が自動整列で崩れた（3回目）
  → /failure-promote で昇格候補
```

```markdown
<!-- .claude/rules/project/imports.md (チームルールに昇格後) -->
### インポート順序
- React → サードパーティ → 社内共通 → ページ別の順で記述
- auto-sort-imports は無効化する（ESLint 設定を優先）
```

**出典引用**:
> "ルールを適用しただけではClaudeにルール" — 組織全体への適用には extract/merge/apply の3段階が必要
> ([Claude Codeにオレたち流のコードを書かせる（中編）— 組織のルールを共有する](https://zenn.dev/sonicgarden/articles/claude-code-custom-rules-part2), セクション "merge-rules") ※2026-05-08に実際にfetch成功

> "失敗は『起きた瞬間に直す』だけでは不十分で、『次のセッションのエージェントが参照できる形』に残さないと"
> ([Claude Codeの失敗をチームルールに昇格させる仕組み](https://zenn.dev/dely_jp/articles/5bc3e9cf62d776), セクション "設計の考え方") ※2026-05-08に実際にfetch成功

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 中
**最終更新**: 2026-05-08

---

### 2. マルチエージェント設計は Anthropic の5パターンを起点に選択する

複数エージェントを協調させる設計では、まず Orchestrator-Subagent パターンを検討し、
特定の制約（品質ゲート・並列スループット・疎結合）が必要な場合に他のパターンへ展開する。

**根拠**:
- Anthropic が定義した5パターン（Generator-Verifier / Orchestrator-Subagent / Agent Teams / Message Bus / Shared State）は実装の多様性を網羅する
- パターンを明示することでチーム内の設計議論が抽象論から具体的選択に変わる
- マルチエージェントはシングルエージェント比 3〜10倍のトークンコストがかかるため、安易な採用は避ける
- 複数エージェント運用が不安定になる主因は「誰が最終判断者か」の曖昧さ。役割を明示しないと重複作業・矛盾回答が発生する
- 実装前の並列レビュー段階（PM視点・技術視点・リスク視点）で先に潰す方が、全体の所要時間が短くなる場面が多い

**5パターンの選択基準**:

| パターン | 適用条件 | 最初の選択肢 |
|----------|----------|-----------|
| **Orchestrator-Subagent** | 独立タスク・リアルタイム・3〜10並列 | ✅ 第一候補（最も広い適用範囲） |
| **Generator-Verifier** | 明示的な検証基準がある・品質重視 | 品質ゲートが必要な場合 |
| **Agent Teams** | バッチ処理・10〜20並列以上 | 大規模リファクタリング |
| **Message Bus** | イベント駆動・疎結合必須 | ETL系ワークフロー |
| **Shared State** | 相互依存タスク・研究型ワークフロー | 協調型調査 |

**コード例（Generator-Verifier のイテレーション上限制御）**:
```typescript
async function generatorVerifierLoop(
  task: Task,
  maxIterations = 3
): Promise<FinalResult> {
  let attempt = 0;
  let output: string | null = null;
  let currentTask = task;

  while (attempt < maxIterations) {
    output = await generator(currentTask, output);
    const result = await verifier(output);

    if (result.accepted) return { output };
    currentTask = { ...currentTask, feedback: result.feedback };
    attempt++;
  }

  return { output, caveats: 'Did not converge after max iterations' };
}
```

**アンチパターン**:
- Generator-Verifier で検証基準を曖昧にする → ゴム印承認・無限ループリスク
- Orchestrator-Subagent でスコープ未定義 → 重複作業が発生する
- Shared State でループ終了条件なし → 収束しない無限サイクル

**ハイブリッドパターン（本番実装の現実）**:
本番システムは複数パターンの組み合わせが主流。典型例: Orchestrator でワークフロー全体を制御しながら、
協調が重いサブタスクには Shared State を使う（`whiteboard.md` 経由で各エージェントが知見を共有）。

**役割三分化の実装例（Claude + 専門 CLI の分業）**:
```
Claude（司令塔）:  企画・判断・統合・最終レビュー
Codex CLI:        大規模コード実装・リファクタリング
Gemini CLI:       情報収集・リサーチ・ドキュメント調査
```
並列レビューパイプラインの例:
1. 計画分解 → 2. 「PM視点・技術視点・リスク視点」3並列レビュー → 3. 実装 → 4. 最終検証

**出典引用**:
> "Production systems often combine patterns. A common hybrid uses orchestrator-subagent for overall workflow with shared state for collaboration-heavy subtasks."
> ([Anthropicの5パターンでClaude Codeエージェント設計を分類する](https://zenn.dev/motowo/articles/anthropic-multi-agent-coordination-patterns-guide), セクション "Hybrid Patterns (Production Reality)") ※2026-05-08に実際にfetch成功

> 「複数エージェント運用が不安定になる原因は『誰が最終判断者か』が曖昧になることです」
> ([Claude Code Workflowで複数エージェントを並列実行する設計](https://zenn.dev/tadkud/articles/claude-code-workflow-pipeline-parallel-agents), セクション "設計の核心") ※2026-06-05に実際にfetch成功

**出典**:
- [Anthropicの5パターンでClaude Codeエージェント設計を分類する](https://zenn.dev/motowo/articles/anthropic-multi-agent-coordination-patterns-guide)
- [Claude Code Workflowで複数エージェントを並列実行する設計](https://zenn.dev/tadkud/articles/claude-code-workflow-pipeline-parallel-agents) (役割三分化・並列レビューパイプライン実装例) ※2026-06-05fetch

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 中
**最終更新**: 2026-06-05

---

### 3. Claude Code カスタムスキル（Skills）は設計前に15の落とし穴を確認する

スキルの実装前に、プロンプト設計・コンテキスト管理・環境設定・アーキテクチャの4カテゴリで
失敗パターンを確認する。「動かない」「意図と違う」「壊れやすい」の3分類で設計を検証する。

**根拠**:
- スキル定義が曖昧だと Claude が無視するか誤ったタイミングで実行する（#1）
- 長いセッションでスキル定義がコンテキスト外に追い出される（#6）
- 過剰な `allowedTools: ["*"]` は意図しないファイル削除や API 呼び出しを招く（#12）
- スキルが責任過多になると「〜と〜」で説明が必要 → その時点で分割すべきサイン（#13）

**重要度の高い落とし穴（8選）**:

| # | 分類 | 落とし穴 | 対策 |
|---|------|----------|------|
| 1 | Prompt | 説明が曖昧で Claude が無視 | "when to use / when NOT to use" を明記 |
| 2 | Prompt | パラメータ型が緩い（`string` のみ） | `pattern`, `enum`, `required` で厳密に定義 |
| 5 | Prompt | 中間状態が未定義で後続ステップが続行 | 各ステップに成功条件と中止条件を書く |
| 6 | Context | 長セッションでスキル定義が消える | `/clear` で定期リセット、頻用スキルを末尾配置 |
| 9 | Context | 並列スキルのレース条件でファイル破損 | 直列化要件を文書化、ロックファイル機構を実装 |
| 12 | Env | `allowedTools: ["*"]` の過剰権限 | ユースケース別に `allowedTools` / `deniedTools` を最小定義 |
| 13 | Design | 単一スキルが多責任（「〜と〜」が必要） | 単一責任原則: 1スキル1つのテスト可能な目的 |
| 15 | Design | エラーが Claude に届かずループ | 「何が・なぜ・次のステップ」を含む構造化エラーを返す |

**コード例（環境変数の明示宣言・最小権限設定）**:
```json
// .claude/settings.json
{
  "env": {
    "API_BASE_URL": "${API_BASE_URL}",
    "NODE_ENV": "development"
  },
  "skills": {
    "code-review": {
      "allowedTools": ["Read", "Bash"],
      "deniedTools": ["Write", "Edit"]
    },
    "deploy": {
      "allowedTools": ["Bash"],
      "deniedTools": ["Write", "Edit", "Read"]
    }
  }
}
```

**実装前チェックリスト（抜粋）**:
- [ ] 1文で目的が言える（「〜と〜」が必要なら分割）
- [ ] 既存スキルとの責任重複がない
- [ ] 各ステップの失敗時の動作が明記されている
- [ ] 環境変数が `settings.json` の `env` に宣言されている
- [ ] パスが絶対パスまたは `$(git rev-parse --show-toplevel)` 相対
- [ ] `allowedTools` がユースケース別に最小スコープ

**Plugin API 固有の落とし穴（storehero 観測、2026-05-26）**:

| # | カテゴリ | 落とし穴 | 対策 |
|---|---|---|---|
| P1 | 環境変数 | `CLAUDE_PLUGIN_*` はプラグインレベルフックで渡るが、Bash ツール内では未設定 | Bash ツールが必要な環境変数は `settings.json` の `env` に別途宣言する |
| P2 | 変数置換 | `${VAR}` の置換範囲はティアごとに異なる（`${CLAUDE_PLUGIN_ROOT}` だけが全ティアで共通） | ティアをまたいで使う変数は `${CLAUDE_PLUGIN_ROOT}` のみ。他は `.claude/settings.json` 経由で渡す |
| P3 | フック登録 | skill frontmatter hook はスキルを**一度 invoke した後**に登録される（初回実行前は未登録） | 初期セットアップが必要なフックはプラグインレベルフックに配置する |
| P4 | 起動差異 | `PreToolUse:Skill` はスラッシュコマンド（`/skill-name`）起動時に発火しない | 自然言語起動を前提に設計するか、両方をカバーする別フックを用意する |

**出典引用**:
> "The strength of Claude Code Skills lies in autonomously executing well-designed workflows. These 15 patterns ensure that initial design gets right."
> ([Claude Code Skills実装で踏んだ地雷15選 ─ 失敗パターンと対策を体系化した](https://zenn.dev/correlate_dev/articles/claude-code-skills-15), セクション "Key Insight") ※2026-05-08に実際にfetch成功

> "全 tier で共通して使える変数は ${CLAUDE_PLUGIN_ROOT} だけ"
> ([ハーネスエンジニアなら知っておきたい Claude Code Plugin の落とし穴](https://zenn.dev/storehero/articles/20ed4b2e8772b3), セクション "変数置換のティアごとの制限") ※2026-05-26に実際にfetch成功

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 中
**最終更新**: 2026-05-26

---

### 4. /btw コマンドでセッションを中断せずにコンテキスト内の情報を確認する

`/btw`（"By the way"）はセッション全コンテキストを参照しつつツール実行を伴わない軽量な質問に使う。
コンテキスト内の情報確認に最適で、新規情報の取得・コマンド実行が必要な場合はサブエージェントを使い分ける。

**根拠**:
- プロンプトキャッシュにより最小コストでコンテキスト参照が可能
- 応答が会話履歴に入らないため余分なコンテキスト消費を防ぐ
- 公式ドキュメント: "it sees your full conversation but has no tools, while a subagent has full tools but starts with an empty context"

**使い分け基準**:

| ケース | 推奨 |
|--------|------|
| 以前触れたファイル名・関数名の確認 | `/btw` |
| 設計判断の再確認（なぜ〜にしたか） | `/btw` |
| 新しいファイルを読む・コマンドを実行 | subagent |
| 会話を過去の状態に戻したい | `/rewind` |
| 独立した調査を並列で行う | `/fork` / subagent |

**典型的な使い方**:
```
/btw さっき読んだ config ファイルの名前は何だった？
/btw handleAuth はどういう目的の関数だっけ？
/btw なぜ fetch ではなく axios を選んだんだっけ？
```

**制約**:
- Claude Code v2.1.72+（2026年3月リリース）が必要（`claude --version` で確認）
- フォローアップ質問不可（1回限りの応答、会話履歴に残らない）
- ツール不使用のためファイル読み込みや Web 検索は不可

**出典引用**:
> "it sees your full conversation but has no tools, while a subagent has full tools but starts with an empty context."
> ([Claude Codeの /btw で開発を止めない：コンテキスト共有と /rewind・subagent との使い分け基準](https://zenn.dev/dearone/articles/672ef7b7e9f407), セクション "What is /btw?") ※2026-05-08に実際にfetch成功

**バージョン**: Claude Code v2.1.72+
**確信度**: 中
**最終更新**: 2026-05-08

---

### 5. Claude Code の Skills 機能でチーム作業ワークフローと個人の学習ループを自動化する（命名・粒度・description設計を徹底する）

`.claude/skills/` ディレクトリに Markdown ファイルを置くとファイル名がスラッシュコマンドとして登録され、反復する作業手順を「実行可能なワークフロー」として切り出せる。
スキルは「ブランチ作成→実装→コミット→PR 作成」のようなチーム作業だけでなく、「ふりかえり KPT」「日報生成」のような**個人の学習ループ**にも有効。フィードバックを Skill 自体に蓄積していくことで、繰り返すたびに精度が上がる。
Claude がスキルを自動起動しない under-trigger 問題を防ぐため、`description` フィールドは「このスキルを使うべき条件」を明示的かつ強めの表現で記述する。
Skill を設計する際は①命名は外部サービス名でなく機能名にする、②関連機能はサブコマンドで一本化する、③`description` は英日両語のキーワードと「使うべき場面」例を含める、④品質指標（テストカバレッジ等）で改善を数値確認する、の4原則を守る。

**根拠**:
- `.claude/skills/` に置いた Markdown ファイルのファイル名がそのままスラッシュコマンドになる
- under-trigger（使うべき場面で Claude がスキルを起動しない）は `description` の書き方で防げる
- `devcontainer.json` の `mounts` で Claude Config ディレクトリをボリュームに永続化すると rebuild 後も再認証が不要
- フィードバックを Skill ファイル自体に追記していくと、Claude が過去の改善履歴を参照して深掘りの精度を上げる学習ループが回る（チームの暗黙知の形式知化にも応用可能）
- 命名を外部サービス名でなく機能名にすることで、サービス仕様変更・廃止時の名前腐敗を防げる
- 関連サブコマンドを1Skillに統合し、呼び出し条件が本当に異なる場合のみ分割する（投稿 vs 予約 → 統合、投稿 vs 分析 → 分割）
- Skill 効果を数値化（ブランチカバレッジ 41.6% → 74.08% 等）することで「感覚的な改善」でなく根拠ある判断ができる
- `AskUserQuestion` ツールを Skill に組み込むと選択肢形式の入力ガイドができ、チームメンバーごとの出力ばらつきを低減できる
- Skill 設計の品質を「予測可能性（毎回同じプロセスで動くこと）」で統一する。Matt Pocock が体系化した6つのアンチパターンを事前に把握しておく: ①Premature Completion（後続ステップ意識による手抜き） ②Duplication（複数箇所での重複記述） ③Sediment（陳腐化した指示の蓄積） ④Sprawl（Skill の肥大化） ⑤No-op（動作変化のない指示） ⑥Uncontrolled Model-Invocation（全 Skill を自動起動してコンテキスト枯渇）。これらを定期的に点検し、戦略的削除を習慣にする

**コード例**:
```json
// .devcontainer/devcontainer.json — Claude Config の永続化
{
  "mounts": [
    "source=project-claude-config,target=/home/node/.claude,type=volume"
  ]
}
```

```markdown
<!-- .claude/skills/dev-flow.md — ワークフロースキル -->
---
description: "新しい変更を実装する際は必ずこのスキルを使って、ブランチ作成→実装→コミット→PR作成を一気通貫で行う"
---

# dev-flow

1. `git checkout -b feature/<task-slug>` でブランチを作成する
2. 変更を実装し、テストを通す
3. 変更内容を説明したコミットメッセージでコミットする
4. `gh pr create` で PR を作成する
```

```markdown
<!-- .claude/skills/briefing.md — 定期確認スキルの例 -->
---
description: "朝の SNS 確認や定期レポートを生成するときはこのスキルを使う"
---

# briefing

1. 昨日のパフォーマンス数値（フォロワー数、エンゲージメント率）を確認する
2. 優先タスク Top3 を抽出する
3. 1〜2 文の運用インサイトを生成する
```

**コード例（Skill 命名・粒度・description 設計のパターン）**:
```markdown
<!-- 良い例: 機能名（sns-post）で命名、サブコマンド統合、英日キーワード付き description -->
<!-- .claude/skills/sns-post.md -->
---
description: "SNS投稿・予約・同期を行う際は必ずこのスキルを使う。Use this skill for posting, scheduling, or syncing social media content. サブコマンド: post / schedule / sync"
---

# sns-post

引数: post|schedule|sync [options]

## post: 投稿実行
1. 文章・画像を確認する
2. 各プラットフォームAPIで投稿する

## schedule: 予約投稿
...

## sync: 外部サービス同期
...
```

```markdown
<!-- 悪い例: サービス名で命名 → サービス廃止・仕様変更でスキル名が腐敗する -->
<!-- .claude/skills/twitter-post.md -->
```

**出典引用**:
> "under-trigger（trigger すべき場面で使わない）の傾向があるため、明示しなくてもスキルが発動するよう description を強めに書く"
> ([Claude Code の skill 機能を本格的に試す — PR まで完結させた話](https://qiita.com/atsushi11o7/items/5cbef4b10f3ec55c75a1), セクション "重要な設計指針") ※2026-05-09に実際にfetch成功

> "`.claude/skills/` フォルダに Markdown ファイルを置くと、そのファイル名がスラッシュコマンドとして使えるようになります。"
> ([Claude Code Skills で /briefing コマンドを5分で実装する](https://qiita.com/hinaworks/items/93ae032d392bdad25655), セクション "実装ステップ") ※2026-05-09に実際にfetch成功

> 「そのフィードバックを貯めておくことで、`furikaeri-kpt` Skill がそのフィードバックを参考に深掘りを改善してくれます。」
> ([使うほどふりかえりの深掘りが上達していく Claude Code Skill を作った話](https://zenn.dev/sonicgarden/articles/03c81ca6413ad4), Zenn sonicgarden) ※2026-05-16に実際にfetch成功

> "サービス側の変化（仕様変更・廃止・上位互換）に影響を受けない"
> ([Claude Code Skills でなぜその設計にするのか — 命名・粒度・description設計の判断基準](https://qiita.com/YujiNaramoto/items/93f393418f2da91e2777), セクション "命名戦略") ※2026-05-30に実際にfetch成功

> "感覚で『なんか良くなった気がする』ではなく、数値で判断するのがポイント"
> ([AIスキルを「育てる」 ── 反復改善でコード自動生成の品質を上げた話](https://zenn.dev/kini/articles/7fae915d406851), セクション "定量的な進捗確認") ※2026-05-30に実際にfetch成功

> "unexperienced users could easily practice it as a 'pattern'"
> ([AI-DLCをClaude Skill化して「エンジニアの役割越境」を実現した話](https://tech.findy.co.jp/entry/2026/05/29/070000), セクション "スキル化の効果") ※2026-05-30に実際にfetch成功

**自己改善サイクルパターン（Mercari 実例）**: レビュースキル (`/review-pr`) と改善スキル (`/improve-review-pr`) をペアで設計し、修正 PR が生まれるたびに「なぜ元のレビューで見落としたか」を分析してスキルファイルに反映する。修正 PR を「スキルへのフィードバック」として自動取り込みすることで、同じパターンの見落としを繰り返さなくなる自己改善ループを構築できる。スキル更新は必ずユーザー承認を挟み、誤ったパターンが定着しないようにする。

> "修正PRが発生するたびに「なぜ元のレビューで見落としたか」を分析し、同じパターンの見落としを二度と起こさないようスキルを更新します。"
> ([修正PRを食べてレビュースキルが賢くなる：Claude Codeによる自己改善サイクル](https://engineering.mercari.com/blog/entry/20260616-0f89ad4c7b/), セクション "/improve-review-pr: Learning from Corrections") ※2026-06-17に実際にfetch成功

> "予測可能性とは「毎回同じ出力を出す」ことではなく、「毎回同じプロセスで動く」こと。"
> ([AIスキル設計の6つの罠 — Matt Pocock Skills v1.0.0 が体系化した「予測可能なAIマニュアル」の書き方](https://zenn.dev/417/articles/masakazu-matt-pocock-skills-design-20260618), セクション "すべては「予測可能性」に帰結する") ※2026-06-18に実際にfetch成功

**出典（追加）**:
- [修正PRを食べてレビュースキルが賢くなる：Claude Codeによる自己改善サイクル](https://engineering.mercari.com/blog/entry/20260616-0f89ad4c7b/) (Mercari Engineering、レビュースキルの自己改善ループと Human-in-the-Loop 設計) ※2026-06-17 fetch

**バージョン**: Claude Code（Skills 機能、2026年4月以降 gh CLI v2.90+）
**確信度**: 高
**最終更新**: 2026-06-18

---

### 6. `.claude/rules/` の `paths` フロントマターで条件付き読み込みを行い、コンテキストトークンを削減する

CLAUDE.md に全ルールを直接書かずに、`.claude/rules/` ディレクトリにファイルを分割し、
`paths` フロントマターで「そのルールが必要になる場面のみ」自動的に読み込まれるようにする。
`@import` はファイルを分割してもコンテキスト全体が注入されるため、トークン削減にならない点に注意。
コンテキスト管理は「何を選ぶか（Select）」だけでなく「いつ消すか（Compress）」まで含む体系として設計する。

**根拠**:
- `@import` はファイルを分割しても全体がコンテキストに注入される — それ自体ではトークン削減にならない
- `paths` フロントマターを設定すると、対象ファイル編集時のみルールが読み込まれる（条件付き読み込み / Select 戦略）
- CLAUDE.md を目次として詳細を rules/ に委譲することで、トークン消費を最大 1/3 に削減できた事例がある
- 使用頻度の低いルールは `.claude/skills/` に `disable-model-invocation: true` で置くと呼び出すまでトークン0
- HTML コメント `<!-- -->` はコンテキスト注入前に除去されるためトークン0（内部メモに活用できる）
- タスクのフェーズ境界（調査完了→実装開始など）で `/clear` を実行する Compress 戦略は、前フェーズの不要な会話コンテキストを消去し、新フェーズに必要な情報のみで再スタートできる。情報を「入れる」だけでは不十分で、「何を入れないか」「いつ入れるか」を設計する必要がある

**コード例**:
```yaml
# .claude/rules/scss.md
---
paths:
    - 'app/scss/**/*.scss'
---
# SCSS規則
- ネストは2段まで
- BEM記法を使う
```

```yaml
# .claude/rules/testing.md
---
paths:
    - '**/*.test.tsx'
    - '**/*.spec.ts'
---
# テスト規則
- describe/it ブロックで構造化する
- モックは vi.mock() でファイルトップに宣言する
```

```markdown
# CLAUDE.md（目次として機能）
# プロジェクトルール

## 常時適用
- [一般ルール](.claude/rules/general.md)

## 条件付き（paths フロントマター — 対象ファイル編集時のみ自動読み込み）
- SCSS: .claude/rules/scss.md
- テスト: .claude/rules/testing.md

<!-- このコメントはコンテキスト注入前に除去されるためトークン0 -->
```

```markdown
# .claude/skills/release-draft.md（disable-model-invocation: true で呼び出すまでトークン0）
---
disable-model-invocation: true
---
# リリースノート作成手順
1. git log から変更を抽出
2. カテゴリ別に整理
...
```

**hook による瞬間注入（paths ではカバーできないケース）**:
`paths` はファイル編集時のみ起動するが、特定コマンド実行タイミングでルールを注入したい場合は `PreToolUse` フックの `hookSpecificOutput.additionalContext` を使う。ファイル種別に依存しないトリガー（`gh pr review` 実行時のみレビュー規約を届けるなど）に有効。

```bash
#!/usr/bin/env bash
# .claude/hooks/inject-pr-review-rule.sh
cmd="$(printf '%s' "$CLAUDE_TOOL_INPUT" | jq -r '.tool_input.command // empty')"
printf '%s' "$cmd" | grep -Eq 'gh pr (review|comment)' || exit 0
rule_file=".claude/rules/pr-review-conduct.md"
jq -n --arg ctx "$(cat "$rule_file")" \
  '{ hookSpecificOutput: { hookEventName: "PreToolUse", additionalContext: $ctx } }'
```

使い分けの原則: `paths` = ファイル種別で自動注入 / `hooks` = コマンド条件で瞬間注入 / `skills` = 明示呼び出しで都度注入。3経路を組み合わせると「常駐は判断骨格のみ・必要な瞬間に必要なルールだけ届く」設計になる。

> "正しい場所に置くだけじゃ足りない。**正しい瞬間に読ませろ。**"
> ([CLAUDE.mdを太らせるな——ルールは必要な瞬間に読ませろ](https://zenn.dev/generald/articles/claude-rules-context-diet), セクション "最終形") ※2026-06-14に実際にfetch成功

**出典引用**:
> "グローバル側の `CLAUDE.md` は目次に寄せました。定型作業をskillに寄せると、レビュー観点のぶれも減ります。"
> ([Claude Code運用を数ヶ月で見直してrulesとskillsに分けた話](https://zenn.dev/harness/articles/claude-code-workflow-evolution), セクション "skill化したのは定型作業を短くするため") ※2026-05-11に実際にfetch成功

> "コンテキストは削減されません...`paths` で対象ファイル編集時のみ読み込まれるようにすると実質トークン消費を抑えられる"
> ([CLAUDE.mdを整理したら、めちゃくちゃトークン消費が増えていた話](https://zenn.dev/penguingymlinux/articles/5335e74359b3d9), セクション "@import を使ってもコンテキストは削減されない") ※2026-05-11に実際にfetch成功

> "情報を「入れる」だけでは不十分で、「何を入れないか」「いつ入れるか」を設計する必要がある"
> ([コンテキストエンジニアリング実践ガイド — Claude Codeで学ぶ4つの戦略](https://zenn.dev/miyan/articles/claude-code-context-engineering-guide-2026), セクション "4つの戦略") ※2026-05-21に実際にfetch成功

**出典**:
- [Claude Code運用を数ヶ月で見直してrulesとskillsに分けた話](https://zenn.dev/harness/articles/claude-code-workflow-evolution) (Zenn) ※2026-05-11 fetch
- [CLAUDE.mdを整理したら、めちゃくちゃトークン消費が増えていた話](https://zenn.dev/penguingymlinux/articles/5335e74359b3d9) (Zenn) ※2026-05-11 fetch
- [コンテキストエンジニアリング実践ガイド — Claude Codeで学ぶ4つの戦略](https://zenn.dev/miyan/articles/claude-code-context-engineering-guide-2026) (Zenn) ※2026-05-21 fetch
- [CLAUDE.mdを太らせるな——ルールは必要な瞬間に読ませろ](https://zenn.dev/generald/articles/claude-rules-context-diet) (Zenn、hook による瞬間注入・paths/hooks/skills 3経路の使い分け) ※2026-06-14 fetch

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 高
**最終更新**: 2026-06-14

---

### 7. Claude Code の実行環境を devcontainer + firewall で多層防御する

Claude Code（および他のAIエージェント）をDev Container内で実行し、
iptables ホワイトリストによるネットワーク分離とパッケージ脆弱性チェックを組み合わせて
サプライチェーン攻撃・外部通信の無制限アクセスを防ぐ。

**根拠**:
- Dev Containers のデフォルトは外向き通信が NAT 経由で素通しになり、AIエージェントが
  意図しない外部通信を行うリスクがある
- 多層防御（ネットワーク分離 + パッケージ検証 + ファイル権限制限）により、
  コンテナエスケープやゼロデイ攻撃の被害をホストOSへ波及させないことが目標

**コード例**:
```json
// .devcontainer/devcontainer.json
{
  "image": "mcr.microsoft.com/devcontainers/base:bookworm",
  "features": {
    "ghcr.io/devcontainers/features/node:1": { "version": "lts" }
  },
  "runArgs": ["--cap-add=NET_ADMIN", "--cap-add=NET_RAW"],
  "postCreateCommand": "bash .devcontainer/setup.sh",
  "postStartCommand": "sudo /usr/local/bin/init-firewall.sh"
}
```

**多層防御の4要素**:
| 対策 | 実装手段 |
|------|----------|
| ネットワーク分離 | iptables ホワイトリスト（npm registry・GitHub のみ許可） |
| npm リスク軽減 | osv-scanner + safe-chain による脆弱性チェック |
| ゼロデイ対策 | `min-release-age=3`（3日未満の新規パッケージを弾く） |
| ファイル権限 | マウント対象の限定・sudo 権限制限 |

**出典引用**:
> "Dev Containersはデフォルトのbridge ネットワークでは、外向き通信は NAT 経由で素通しになります"
> ([Claude Codeとサプライチェーン攻撃（npm＆バイナリ）を隔離するdevcontainer.json](https://zenn.dev/isosa/articles/971ab0b2281f53), セクション "ネットワーク分離の背景") ※2026-05-13に実際にfetch成功

**バージョン**: Claude Code（全バージョン）, Dev Containers（全バージョン）
**確信度**: 中
**最終更新**: 2026-05-13

---

### 8. 複数AIツール（Claude Code・Codex等）を AGENTS.md で一元管理し、3層の権限モデルで操作境界を明示する

同一プロジェクトで複数のAIコーディングエージェントを併用する場合、
「ツール非依存の共通ルール」を `AGENTS.md` に集約し、各ツール固有の設定ファイルから
`@AGENTS.md` でインポートする「共通正本 + 薄いアダプタ」構成を採用する。
`AGENTS.md` はAIへの「お願い」ではなく「安全帯・朝礼・作業指示書」として、3層の権限モデルで操作境界を明示する。

**根拠**:
- Claude Code（`CLAUDE.md`）と Codex（`AGENTS.md`）など各ツールの設定形式は互いに非互換
- 設定ファイルを直接共有できないため、各ツールが参照する**スクリプトや指示ファイルの実体を一元化**する
- Skill（`~/.agents/skills/`）と hook スクリプト（`shared/hooks/`）を共通正本に置き、
  各ツールの設定からシンボリックリンクで参照することで1箇所の変更が全ツールに反映される
- 人間が最低限のルールを明示的に書いた仕様の方が、AIが自動生成した指示ファイルよりも成功率が高い（ETH Zurich研究で自動生成はコスト20%増・成功率低下を確認）
- `AGENTS.md` は「恒久的なルール」のみを置き肥大化を防ぐ：手順は SOP へ、設計決定は `_docs/decisions/` へ、作業ログは `_docs/logs/` へ分離する。重要な規則が大量の作業メモに埋もれると遵守率が下がる
- MCP は **Opt-in モデル**で管理する：常時接続 MCP が増えるほど起動遅延・コンテキスト肥大・CI 不安定が生じる。個人（`~/.codex/config.toml`）→ リポジトリ（`.codex/config.toml` でツールレベル allowlist）→ CI（最小構成）の3層で制御し、タスク要件なしに MCP を自動呼び出ししない

**3層の権限モデル（AGENTS.md 推奨フォーマット）**:
```markdown
## 常に許可
- 読み取り、調査、lint、ユニットテスト

## 承認が必要
- 依存関係の変更、ファイルの削除、スキーマ変更

## 絶対に実行しない
- 認証情報の操作、本番への force push、認可なしの外部通信
```

**セッション引き継ぎテンプレート**:
```markdown
## handoff（次のセッションへ）
- 目的: （1文で）
- 今回の1タスク: （1つだけ）
- 正本: （ソースファイルパス）
- やらないこと: （スコープ外の明示）
- 完了条件: （検証可能な条件）
- 検証手順: （コマンドまたは操作）
```

**コード例**:
```markdown
<!-- CLAUDE.md（最小構成）-->
@AGENTS.md

## Claude Code固有の補足
プロジェクト基本ルールはAGENTS.md側に集約。
ここはClaude Code固有機構だけ記述。
```

```bash
# Skill の一元管理（実体を ~/.agents/skills/ に置き symlink で参照）
mv ~/.claude/skills/my-skill ~/.agents/skills/my-skill
ln -s ~/.agents/skills/my-skill ~/.claude/skills/my-skill

# hook スクリプトは shared/hooks/ に実体を置き、各ツール設定からそれを参照する
# 例: .claude/settings.json の hooks.PostToolUse から shared/hooks/ruff-format.sh を呼ぶ
```

**出典引用**:
> "共通正本 + 薄いアダプタ。両ツールに守ってほしい情報を1箇所に集約し、各ツール固有の機構をその上に薄く乗せる"
> ([Claude Code × Codex 共存セットアップ — ルール・Skills・hooks を一元管理する](https://qiita.com/kirozero/items/aec53be56a5427475969), セクション "設計原則") ※2026-05-13に実際にfetch成功

> "AGENTS.mdはAIへの『お願い』ではない。AI現場の安全帯・朝礼・作業指示書だ"
> ([AIエージェントに仕事を任せるために、AGENTS.md を「就業規則」にした](https://zenn.dev/takayoshioyama/articles/d386faa8bbb57f), セクション "AGENTS.md = AI現場の就業規則") ※2026-06-06に実際にfetch成功

> "AGENTS.md の効果は長さより役割分離で決まる。巨大な総合マニュアルより、エントリポイント＋差分ドキュメントとして機能させる方が運用しやすい"
> ([AGENTS.mdに何を書くべきか](https://zenn.dev/tadkud/articles/codex-agents-md-best-practices), セクション "構造パターン") ※2026-06-09に実際にfetch成功

> "AGENTS.md は恒久的なルールだけに絞る。ログ・メモ・タスク履歴が混入すると重要なルールが埋もれる"
> ([エージェント開発の反パターン10 — 速さの代償をrepoに払わせない](https://zenn.dev/zapabob/articles/agent-dev-antipatterns-10), セクション "反パターン#6: AGENTS.md 肥大化") ※2026-06-19に実際にfetch成功

> "勝手に叩かない。MCP は Suggest → Opt-in で管理し、タスク要件が明示されるまで MCP ツールを自動呼び出ししない"
> ([MCPを増やしすぎない — エージェントの工具箱の整理](https://zenn.dev/zapabob/articles/mcp-toolbox-organization), セクション "原則1: Suggest → Opt-in") ※2026-06-19に実際にfetch成功

**バージョン**: Claude Code（全バージョン）、複数AIエージェント共存環境
**確信度**: 中
**最終更新**: 2026-06-19

---

### 9. Claude Code の hooks で危険なコマンドをブロックし、操作規律をコードに落とす

JSON stdin → 処理 → exit コードというパイプラインを理解し、誤って破壊的操作を実行しないよう安全装置をスクリプトで実装する。

**根拠**:
- hooks はツール実行前後に介入できる仕組みで、`exit 2` で実行をブロック、`exit 0` で許可する
- `rm -rf`・本番 DB の直接書き換え・機密ファイルへのアクセスなど「やってはいけない操作」を機械的に防げる
- スクリプトは再利用可能で、チーム全体の操作規律を `.claude/settings.json` で共有できる
- **CLAUDE.md への指示は「お願い」、Hooks は「強制」** — セッションをまたいでも必ず適用される規律をコードに落とす
- `.env` ファイルへのアクセスを PreToolUse でブロックすることで、シークレットの意図しない読み取りを防げる
- **ブラックリスト戦略**（禁止リストのみを列挙し、それ以外は全部許可）はホワイトリスト方式（許可リストを書き続ける）より保守コストが低く、エージェントの自律性を保ちながら安全を担保できる
- **「Lost in the Middle」問題**: long context 環境ではコンテキスト中盤に置かれたルールが LLM に無視されやすい。禁止チェックを LLM の判断に委ねるのではなく、PreToolUse フックで決定論的にブロックすることが本質的な防衛になる
- **4層防御モデル**: CLAUDE.md（確率的・劣化あり）→ settings.json deny（決定論的・文字列変形に弱い）→ PreToolUse フック（決定論的・複雑条件に対応）→ OS サンドボックス（能力を根本的に制限）の順に信頼性が増す。フックは3番目の層として最も実装コスパが高い
- **フレームワーク固有の破壊コマンドは shell レベルのブロックをすり抜ける**: `rm -rf` や `dd` といった shell コマンドのパターンマッチだけでは `php artisan migrate:fresh`・`rails db:drop`・`prisma migrate reset` 等のフレームワーク CLI が検知されない。DB ワイプ系コマンドは別途 PreToolUse フックに追加する
- **「危険なのはコマンドの種類ではなく、対象が本番かどうか」**: PreToolUse フックでコマンドそのものをブロックするだけでなく、`AWS_PROFILE`・ホスト名・DB エンドポイントのパターンから「本番向けか」を判定し、read（allow）/ non-prod write（ask）/ prod write（ask）/ prod 不可逆（deny）の3段階で制御することで、ステージング環境の生産性を損なわずに本番安全性を高められる。`PreToolUse` フックは `--dangerously-skip-permissions` モード中でも動作するため、自動実行セッションでの安全層として有効

**コード例（実践的な PreToolUse パターン）**:
```bash
# ~/.claude/hooks/pre-bash.sh — 複数ルールをまとめたブロックフック
#!/bin/bash
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

# 1. rm -rf をブロック（破壊的操作）
if echo "$CMD" | grep -qE 'rm[[:space:]]+-rf|rm[[:space:]]+-fr'; then
  echo "rm -rf は hooks でブロックされています。削除対象を明示してください。" >&2
  exit 2
fi

# 2. コマンドチェーン（&& / ; / ||）をブロック（一括実行で意図しない副作用を防ぐ）
if echo "$CMD" | grep -qE '&&|;|\|\|'; then
  echo "コマンドチェーンは禁止されています。コマンドは 1 つずつ実行してください。" >&2
  exit 2
fi

# 3. 1 MB 超のファイルの無制限読み込みをブロック
if echo "$CMD" | grep -qE '^cat '; then
  FILE=$(echo "$CMD" | awk '{print $2}')
  if [ -f "$FILE" ] && [ "$(wc -c < "$FILE")" -gt 1048576 ]; then
    echo "1 MB 超のファイルを cat するには head/tail で範囲を指定してください。" >&2
    exit 2
  fi
fi

exit 0
```

```bash
# デバッグ: 実際に受け取る JSON 構造を確認するダンプフック
#!/bin/bash
cat > "/tmp/hook-debug-$(date +%s).json"
exit 0
```

```json
// .claude/settings.json への登録例
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "~/.claude/hooks/pre-bash.sh" }]
      }
    ]
  }
}
```

**exit コードの意味**:

| コード | 動作 |
|---|---|
| 0 | ツールを通常実行 |
| 2 | 実行をブロック（stderr のメッセージを Claude に通知） |
| その他 | 警告のみ（ツールは実行される） |

**アンチパターン**:
- `async: true` のフックで `exit 2` を期待する → バックグラウンドプロセスはブロックできない（Rule #12 参照）
- memory file の注意書きで代替しようとする → 同じ失敗を新規セッションで繰り返す

**コード例（.env ファイルブロック — PreToolUse）**:
```bash
#!/bin/bash
INPUT=$(cat)
TOOL=$(echo "$INPUT" | jq -r '.tool_name // empty')
FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // .tool_input.command // empty')

# .env ファイルへのアクセスを全ツールでブロック
if echo "$FILE" | grep -qE '(^|/)\.env($|\.)'; then
  echo ".env ファイルは hooks でブロックされています。環境変数が必要な場合は process.env を使ってください。" >&2
  exit 2
fi

exit 0
```

**コード例（多エージェント統合ゲートウェイ — --no-verify ブロック + PII 検出）**:
```bash
# git config --global core.hooksPath "$HOME/.git-hooks" で全ツール共通 pre-commit フックを設定
# ~/.git-hooks/pre-commit

#!/bin/bash

# 1. gitleaks でシークレット検出
gitleaks git --pre-commit --staged --redact --verbose . || exit 1

# 2. --no-verify / -n フラグをステージ差分経由でのみ許可（フック自体のバイパスを防ぐ）
# ※ このフックを bypass する手口（git commit -n）を Claude Code / Codex hooks でブロックする
BYPASS_REGEX='--no-verify|( -n[[:space:]])|( -[a-zA-Z]*n[a-zA-Z]*[[:space:]])|-nm'
if echo "$GIT_PARAMS" | grep -qE "$BYPASS_REGEX"; then
  echo "ERROR: --no-verify flag is blocked" >&2
  exit 1
fi

# 3. PII 検出（メールアドレス・ローカルパス）
STAGED=$(git diff --cached -U0)
PII_REGEX='[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}|/Users/[A-Za-z0-9_.-]+/|[A-Z]:\\\\Users\\\\'
if echo "$STAGED" | grep -qE "$PII_REGEX"; then
  # bot アカウントは許可
  if ! echo "$STAGED" | grep -qE 'noreply@github\.com'; then
    echo "WARNING: possible PII detected in staged diff. Review before committing." >&2
  fi
fi

exit 0
```

```bash
# Claude Code hooks からも --no-verify をブロック（.claude/settings.json の PreToolUse）
# pre-bash.sh に追記
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
if echo "$CMD" | grep -qE '(git commit|gh pr).*(--no-verify|-n[[:space:]])'; then
  echo "--no-verify は hooks でブロックされています。" >&2
  exit 2
fi
```

```bash
# フレームワーク固有の DB 破壊コマンドをブロック（pre-bash.sh に追記）
# shell レベルのブロックをすり抜けるため別途追加が必要
if echo "$CMD" | grep -qE \
  'migrate:fresh|migrate:reset|db:wipe|db:drop|db:reset|manage\.py flush|prisma migrate reset|db push --force-reset'; then
  echo "DB ワイプ系コマンドは hooks でブロックされています。意図する場合は手動で実行してください。" >&2
  exit 2
fi
```

```bash
# push 先リポジトリ所有者を allowlist で検証（誤 push・フォーク外流出を防ぐ）
# PostToolUse（Bash）または pre-push hook で実行
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

# git push コマンド以外は早期リターン（パフォーマンス）
echo "$CMD" | grep -qE '^git push' || exit 0

# push 先リモートの URL を取得
REMOTE=$(echo "$CMD" | grep -oE 'origin|upstream|[a-z0-9_-]+' | head -1)
REMOTE=${REMOTE:-origin}
REMOTE_URL=$(git remote get-url "$REMOTE" 2>/dev/null || true)

# 許可リスト（自分の GitHub ユーザー名 / 組織名）
ALLOWED_OWNERS=("myorg" "myusername")

owner_allowed=false
for owner in "${ALLOWED_OWNERS[@]}"; do
  if echo "$REMOTE_URL" | grep -qiE "github\.com[/:]${owner}/"; then
    owner_allowed=true
    break
  fi
done

if ! $owner_allowed; then
  echo "ERROR: push 先 '$REMOTE_URL' は許可されたリポジトリ所有者に含まれていません。" >&2
  echo "意図する場合は手動で git push を実行してください。" >&2
  exit 2
fi
```

**出典**:
- [Claude Codeのhookの仕組み:JSONとexitコードで作る最小の安全装置](https://zenn.dev/yurukusa/articles/be79dbe97e34bb) (Zenn) ※2026-05-14に実際にfetch成功
- [Claude Code hook で AI coding assistant の規律を補強する — 個人運用での設計パターン参考](https://zenn.dev/shogaku/articles/claude-code-hook-discipline) (Zenn、&&/;/|| チェーンブロック・1MB超ファイルブロックのパターン追加) ※2026-05-22に実際にfetch成功
- [Claude Code Hooksで作る7つの安全装置 ─ PreToolUse/PostToolUse 実装集](https://zenn.dev/kenimo49/articles/claude-code-hooks-7-safety-patterns) (Zenn、.env ブロック・Stop フック・CLAUDE.md request vs Hooks 強制の整理) ※2026-05-29に実際にfetch成功
- [Claude Code と Codex の両方に、機密情報と個人情報を漏らさせない hook を作った話](https://zenn.dev/todayama_r/articles/multi-agent-secret-pii-guard-hooks) (Zenn、machine-wide core.hooksPath・--no-verify ブロック・PII 検出パターン・escape hatch 防止) ※2026-05-30に実際にfetch成功
- [Claude Codeのハーネス設計 -- 「禁止事項だけを決め、hooksで強制する」ブラックリスト戦略](https://zenn.dev/awesome_kou/articles/claude-code-harness-deny-hooks) (Zenn、deny-list vs allow-list 設計・Lost in the Middle 問題・4層防御モデル) ※2026-06-09に実際にfetch成功
- [`migrate:fresh`で本番DBが消える——Claude Codeの自動承認とフレームワークの破壊コマンド](https://qiita.com/yurukusa/items/a8ba73afd314b7fb822d) (Qiita、フレームワーク固有 DB 破壊コマンドが shell レベルブロックをすり抜ける問題と対策パターン) ※2026-06-17に実際にfetch成功
- [Claude Codeのpush事故を機械で止める ― APIキー流出と誤push先ガード](https://qiita.com/bokuwalily/items/31b3b4f3a447b6c5d68e) (Qiita、push 先 owner allowlist・ステージ差分のみスキャン・hook bypass フラグ禁止) ※2026-06-22に実際にfetch成功
- [rm -rf をブロックしても、AIの本番事故は防げない ── hookに『環境』を判定させる](https://zenn.dev/sprix_it/articles/4c44e56ba6f28a) (Zenn sprix_it、環境検知型3段階許可制・prod/staging/local の分岐パターン) ※2026-06-23 fetch

> "Laravel: migrate:fresh / migrate:reset / db:wipe; Rails: db:drop / db:reset; Django: manage.py flush; Prisma: migrate reset / db push --force-reset — これらは shell 固有の危険コマンドのパターンマッチをすり抜けます"
> ([`migrate:fresh`で本番DBが消える——Claude Codeの自動承認とフレームワークの破壊コマンド](https://qiita.com/yurukusa/items/a8ba73afd314b7fb822d), セクション "なぜ既存の防御策が効かないのか") ※2026-06-17に実際にfetch成功

> "Memory file の「注意書き」だけだと同じハマりを繰り返すと感じることがあります。強制可能な規律は強制する、memory rule で誤魔化さない"
> ([Claude Code hook で AI coding assistant の規律を補強する](https://zenn.dev/shogaku/articles/claude-code-hook-discipline), セクション "Incremental Expansion") ※2026-05-22に実際にfetch成功

> "CLAUDE.mdへの指示は『お願い』、Hooksは『強制』です"
> ([Claude Code Hooksで作る7つの安全装置](https://zenn.dev/kenimo49/articles/claude-code-hooks-7-safety-patterns), セクション "はじめに") ※2026-05-29に実際にfetch成功

> "禁止事項のチェックを、確率的に揺らぐLLMの判断に依存させてはいけません"
> ([Claude Codeのハーネス設計 -- 「禁止事項だけを決め、hooksで強制する」ブラックリスト戦略](https://zenn.dev/awesome_kou/articles/claude-code-harness-deny-hooks), セクション "LLMベースの判断の問題点") ※2026-06-09に実際にfetch成功

> "すべての入口を一本化する関門で、コミットは必ずそこを通る"
> ([Claude Code と Codex の両方に、機密情報と個人情報を漏らさせない hook を作った話](https://zenn.dev/todayama_r/articles/multi-agent-secret-pii-guard-hooks), セクション "Layer 1: Unified Git Gateway") ※2026-05-30に実際にfetch成功

> "hook bypass フラグが含まれています。明示依頼が無い限り使用禁止"
> ([Claude Codeのpush事故を機械で止める](https://qiita.com/bokuwalily/items/31b3b4f3a447b6c5d68e), セクション "Hook Bypass Prevention") ※2026-06-22に実際にfetch成功

> "the danger isn't the command type, but whether it's directed at production"
> ([rm -rf をブロックしても、AIの本番事故は防げない ── hookに『環境』を判定させる](https://zenn.dev/sprix_it/articles/4c44e56ba6f28a), セクション "Core Principle") ※2026-06-23に実際にfetch成功

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 高
**最終更新**: 2026-06-23

---

### 10. Git Worktree で Claude Code タスクを並列実行し、編集境界と隔離境界を厳格に固定する

`git worktree add` で複数のワーキングツリーを作成し、独立した Claude Code セッションを並走させる。
ただし worktree はブランチを分けても**共通ファイルの競合を自動解決しない**ため、各エージェントの編集対象を明示的に固定し、共通スタイルファイル等はシリアル処理に倒す前提で組む。
また、`isolation: "worktree"` オプションは「デフォルトの作業ディレクトリを worktree 内に向ける補助機能」にすぎず、**完全な隔離ではない**。絶対パス指定や cwd 継承でサブエージェントが主リポジトリを操作できてしまうため、`PreToolUse` フックで物理的な境界を強制する。

**根拠**:
- 各 worktree は独立したブランチ・ファイルシステムを持つため、Claude Code セッション間で「同じワーキングディレクトリのファイル」を奪い合うことは起きない
- 1つのリポジトリを複数クローンするより効率的（`.git` を共有するためディスク使用量が少ない）
- レビュー中 PR へのコメント対応・バグ修正・機能開発を同時並行できる
- 反面、`_common.scss` / `style.scss` / `main.scss` のような共通ファイルは別ブランチでも内容衝突する。エージェントごとに「このファイルのみ編集」と明示しないとコンフリクトが集中する
- **`isolation: "worktree"` の限界**: 絶対パス経由でのアクセス・サブエージェント完了後の cwd 継承など、worktree 境界を踏み越える事象が現実に発生する（1週間13件のインシデント報告あり）。エージェントへの指示では隔離を担保できず、`PreToolUse` フックによる決定論的なブロックが必要
- **3段階の判定ロジック**: ①リードエージェントは無条件通過、②サブエージェントの worktree ルートを `git rev-parse --show-toplevel` で検証、③サブエージェントの worktree が主リポジトリと一致する場合はブロック

**コード例**:
```bash
# worktree を追加（issue#123 の作業用ブランチ）
git worktree add ../project-issue123 -b feat/issue-123

# worktree 一覧確認
git worktree list

# 各 worktree で独立した Claude Code セッションを起動
cd ../project-issue123 && claude
```

```bash
# vibe CLI を使った統合管理（OSS: https://github.com/kexi/vibe）
vibe new feat/my-feature   # worktree + Claude Code セッション作成
vibe list                   # アクティブなセッション一覧
vibe clean                  # 完了した worktree を削除
```

**実践的な知見**:
- 4〜6 並列がレビューの観点でスイートスポット（これを超えると追跡が困難になる）
- 各 worker が実装前に計画を提示し、人間がレビュー後に実行承認するフローが安全
- 依存関係のあるタスクは順次キューイング、独立タスクのみ並列化する

**コンフリクト対策の優先順位**（上から試す）:
1. **エージェントごとに編集ファイルを厳格に固定する**: 「エージェント A はこのファイルのみ編集」を明示。共通ファイルはシリアル処理に倒す
2. **コンポーネント単位の閉じた構造を採用する**: Vue SFC や CSS Modules など「1 ファイルに HTML / JS / CSS が閉じている構造」はそもそもコンフリクトが起きにくい
3. **並列作業をやめる**: 認知負荷（コンフリクト対応）を考えると「順次実行 + 人間は要件定義・交通整理に注力」の方が結果的に速いケースがある

**worktree 境界を強制する PreToolUse フック（追加パターン）**:
```json
// .claude/settings.json — サブエージェント用 worktree ガード
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash|Edit|Write|NotebookEdit",
      "hooks": [{
        "type": "command",
        "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/worktree-isolation-guard.sh"
      }]
    }]
  }
}
```
```bash
#!/bin/bash
# .claude/hooks/worktree-isolation-guard.sh
# サブエージェントが主リポジトリを操作しようとした場合にブロック
INPUT=$(cat)
AGENT_TYPE=$(echo "$INPUT" | jq -r '.agent_type // "lead"')

# リードエージェントは無条件通過
[[ "$AGENT_TYPE" == "lead" ]] && exit 0

# サブエージェント: 現在の worktree ルートを検証
CURRENT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
MAIN_ROOT=$(git -C "$CLAUDE_PROJECT_DIR" rev-parse --show-toplevel 2>/dev/null)

if [[ "$CURRENT_ROOT" == "$MAIN_ROOT" ]]; then
  echo "ERROR: サブエージェントが主リポジトリへの操作を試みました。worktree 内で実行してください。" >&2
  exit 2
fi
exit 0
```

**出典引用**:
> "エージェントの隔離は、エージェントへの指示では担保できません。"
> ([Claude Code の worktree 隔離は信用するな — 1週間13件のインシデントと PreToolUse フックによる解決](https://zenn.dev/penne_inc/articles/claude-code-worktree-isolation-bug), セクション "根本原因の特定") ※2026-06-10に実際にfetch成功

> "プロンプトを補完するもの、と捉えるとちょうどいい。エージェントに任せたい領域はプロンプトで、確実に守りたい領域はフックで"
> ([Claude Code の PreToolUse フックで AI エージェントの行動を物理的に制御する](https://zenn.dev/penne_inc/articles/claude-code-pretooluse-hook-permission-control), セクション "エージェントの善意を信じつつ、構造で守る") ※2026-06-10に実際にfetch成功

> "4-6 parallel issues are the sweet spot to maintain readability"
> ([Break It Small, Ship It Right – Skills for Coding Agents](https://developers.cyberagent.co.jp/blog/archives/63674/), セクション "Enable Parallel Development on Independent Issues") ※2026-05-14に実際にfetch成功

> "git worktreeで並走する開発フローを作ることで、PRレビュー待ち中も別タスクをClaude Codeに任せられる"
> ([Git Worktree CLI「vibe」で Claude Code と並走する開発フローを作る](https://zenn.dev/kexi/articles/git-worktree-vibe-claude-code), セクション "なぜ worktree か") ※2026-05-14に実際にfetch成功

> 「ブランチは切った時点の状態を元に作業するので、別ブランチにしてワークツリーを立てたとて同じファイルを別々に編集すれば当然衝突する。」「『他ファイル編集禁止』くらいまで指定した方が安全そうである。」
> ([【初心者】ワークツリーを切ってイキりまくり、エージェント並行開発をしていたらコンフリクトを連発した話と対策。](https://zenn.dev/wahe/articles/eda7fb1a7a2848), Zenn wahe) ※2026-05-16に実際にfetch成功

**バージョン**: Claude Code + Git（全バージョン共通）
**確信度**: 中
**最終更新**: 2026-06-10

---

### 11. Claude Code の繰り返し作業を独自 Skill に変換し、拡張機能（Skills/MCP/Hooks）を用途別に使い分ける

繰り返し発生する作業（タスク分解・検証・品質ゲート）を再利用可能な Skill に変換する。
外部マーケットプレイスの Skill を集めるのではなく、自チームのワークフローに特化した独自 Skill を育てることが持続的な生産性向上につながる。**「同じ指示を 2 回書いたら、3 回目を書く前に Skill 化」**を判断閾値にする（CLAUDE.md / README へのルール追記が 2 回繰り返された時点がシグナル）。

**根拠**:
- 繰り返す作業を Skill に変換することで、同じプロンプト指示を毎回書かずに済む
- Skills（「何を考えるか」）と MCP（「何にアクセスするか」）を組み合わせることで複雑なタスクに対応できる
- Hooks で品質ゲート（linting・テスト実行）を自動化することで、コンテキストを保ちながら品質を維持できる
- 第三者 Skill には悪意あるコードが混入するリスクがある（2025年のClawHavoc事件では341個の不正Skillが発見された）
- Skill ファイルを Single Source of Truth として扱う運用（ASDD: Agent Skill Driven Development）では、Wiki / Slack / README にルールを分散させずに Skill に集約する。ライフサイクルは Bootstrap → Develop → Capture → Review → Measure の 5 フェーズ
- Skill を修正した後は「①AI による公式ベストプラクティスレビュー」→「② SKILL.md のプログレッシブディスクロージャー確認（詳細を `references/` に委譲しているか）」→「③検証サイクル（validate → fix → rerun）の埋め込み確認」の軽量3ステップで品質劣化を早期検知できる
- **Skill が失敗するパターン**: 優先度付け・発散思考・複雑な判断など「次回も同じ手順になるか」が不確かなタスクを Skill 化すると「同じパターンに収束して発散の意味が消える」問題が起きる。Skill は「配管（定型固定手順）」に限定し、判断はエージェントに委ねる。Skill 化の前に「次回も手順が同じか？」を必ず問う
- 2,000人規模の運用が示す通り、個人の発見を組織資産（Skill/Routine）に変換し続けることが自動化の深度を決める。ツールを配布するだけでは個人知見が組織に広まらない——使い込む時間の長さが深度を決めるため、フィードバックループを持つ研修・スキル化サイクルを組む。ROI 評価は「コスト削減」ではなく「開発速度・品質改善のボトルネック解消」で測る

**拡張機能の使い分け**（起動条件とコンテキスト共有で選ぶ）:
| 拡張 | 起動条件 | コンテキスト | 用途 |
|---|---|---|---|
| **Agents** | 明示的な委任 | 独立（分離） | 並列実行・ロール分担（品質review と security review を同時） |
| **Skills** | ユーザー / モデルが呼び出す | 共有 | ドリフトしやすい繰り返しワークフローの固定 |
| **MCP** | ツール呼び出し | 共有 | 外部サービスへのアクセス（Notion, GitHub, Slack 連携） |
| **Hooks** | イベント自動トリガー | スクリプト I/O のみ | 確定的な強制（フォーマット・lint・シークレット検出） |
| **Plugins** | ─ | ─ | Skills/MCP/Hooks のパッケージ化（チーム別ロールベース配布） |

**選択の判断軸**: 「コンテキスト分離が必要か → Agents」「呼ばれた時だけ効けば良いか → Skills」「毎回強制的に実行したいか → Hooks」。
段階的実装の順: `CLAUDE.md` → Skills → Agents → Hooks（前の段階で動作を確認してから次に進む）

**MCP スコープ優先順位**（同名サーバーが複数ある場合は上位が優先）:
| スコープ | 保存先 | 用途 |
|---|---|---|
| **local**（最高優先） | `~/.claude.json` | 機密情報・実験的設定（バージョン管理外） |
| **project** | `{repo}/.mcp.json` | チーム共有の承認済みサーバー（環境変数展開可） |
| **user** | `~/.claude.json` | 個人クロスプロジェクト設定 |
| plugin-provided | - | Skills/プラグイン同梱 |
| claude.ai connectors | - | プラットフォーム標準 |

**MCP トランスポート選択**: クラウドサービス（Notion, Stripe, Sentry）→ HTTP が推奨。ローカルツール・CLI → Stdio。外部コンテンツを fetch する MCP サーバーはプロンプトインジェクションリスクがあるため、read-access と external-send を同時に有効化しない（詳細は `security/dependency-security.md` Rule #7 参照）

**コード例（独自 Skill の定義例）**:
```yaml
# .claude/skills/plan-task.yaml
name: plan-task
description: GitHub Issue を複数サブタスクに分解し、依存関係を解析して並列実行計画を立てる
steps:
  - コードベースを探索して影響範囲のファイル・サービス・パターンを特定する
  - Issue を独立した作業単位に分割する（GitHub のサブイシュー API で親子関係を作成）
  - 依存関係のないタスクを並列キューに入れ、依存するものは順次キューに入れる
  - 各ワーカーに作業前にプランを提示させ、承認後にコーディングを開始させる
```

```yaml
# .claude/skills/post-task-verification.yaml
name: post-task-verification
description: 実装完了後、各要件が実際のコード変更で満たされているかを読み取り専用で検証する
steps:
  - Issue の受け入れ条件を一覧化する
  - 各条件について、実装がどのファイル・関数で実現しているか確認する
  - 未実装・不完全な実装を列挙し、ギャップレポートを作成する
  - lint とテストを自動実行する
```

**コミット粒度の規則（AI エージェントのコミット順序）**:
```
# 変更を目的別に分割してコミットする順序:
lint/docs → dependencies → tests → schema → refactoring → fixes → features

# フォーマット: <prefix>: <description>
feat: add rate-limit feature
fix: resolve null pointer in user service
refactor: extract auth logic to separate module
```

意図的な順序でコミットを積むことで、エージェントが誤った変更をした際に `git revert <commit>` で精密な取り消しができる。

**出典引用**:
> "The most valuable approach: 'Identify tasks repeating in your own work, then Skill-ify them' rather than collecting external marketplace items. This creates sustainable, customized productivity gains."
> ([Break It Small, Ship It Right – Skills for Coding Agents](https://developers.cyberagent.co.jp/blog/archives/63674/), CyberAgent Engineering Blog) ※2026-05-15に実際にfetch成功

> "Skills teach *what to think*; MCP provides *what to access* — both are necessary for complex tasks like document analysis."
> ([Skills/MCP/Hooks/Plugins Claude Code 4つの拡張を使い分ける実践ガイド](https://zenn.dev/kenimo49/articles/claude-code-multi-tool-skills-mcp-hooks), Zenn) ※2026-05-15に実際にfetch成功

> "Skill First ── 同じ指示を 2 回書いたら、3 回目を書く前に Skill 化" / "Single Source ── ルール分散（Wiki、Slack、README）を避け、Skill に集約"
> ([エージェントスキルを中心とした開発手法を考える](https://zenn.dev/tnkn08/articles/asdd-agent-skill-driven-development), Zenn tnkn08, セクション "7 つの実践ルール") ※2026-05-16に実際にfetch成功

**出典引用**:
> "obvious quality degradation is prevented through lighter, faster validation steps: review against official best practices, then verify progressive disclosure and feedback loops remain intact"
> ([【Claude Code】Agent Skill作成時の雑なTips 3選](https://zenn.dev/dk96424/articles/claude-skill-checklist), セクション "Tip 3") ※2026-05-24に実際にfetch成功

**出典**:
- [Break It Small, Ship It Right – Skills for Coding Agents](https://developers.cyberagent.co.jp/blog/archives/63674/) (CyberAgent Engineering Blog) ※2026-05-15 fetch
- [Skills/MCP/Hooks/Plugins Claude Code 4つの拡張を使い分ける実践ガイド](https://zenn.dev/kenimo49/articles/claude-code-multi-tool-skills-mcp-hooks) (Zenn) ※2026-05-15 fetch
- [エージェントスキルを中心とした開発手法を考える](https://zenn.dev/tnkn08/articles/asdd-agent-skill-driven-development) (Zenn) ※2026-05-16 fetch
- [【Claude Code】Agent Skill作成時の雑なTips 3選](https://zenn.dev/dk96424/articles/claude-skill-checklist) (Zenn) ※2026-05-24 fetch
- [Claude Codeのagents / skills / hooksをどう使い分ける？実プロダクト開発で出した運用ルール](https://zenn.dev/dx_pm_product/articles/claude-code-agents-skills-hooks) (Zenn、Agents/Skills/Hooks の起動条件・コンテキスト共有による選択フレームワーク・段階的実装順) ※2026-06-02に実際にfetch成功
- [Claude Code スキルを作り始めて気づいた落とし穴 ── 2分で作れるが、一問だけ先に答えろ](https://zenn.dev/i_ichi/articles/claude-code-skills-pitfall) (Zenn / i_ichi、判断ベースタスクを Skill 化した際の失敗パターンと「配管 vs 判断」の区別) ※2026-06-11 fetch
- [Claude Code Skills設計パターン ： 段階的開示とコンテキスト2%ルール](https://zenn.dev/correlate_dev/articles/claude-code-skills-progressive-disclosure) (Zenn correlate_dev、コンテキスト2%ルール・Iron Law 品質パターン・description文字数計測スクリプト) ※2026-06-15 fetch
- [約2,000人が使うClaude Codeと向き合う。](https://developers.cyberagent.co.jp/blog/archives/64334/) (CyberAgent Engineering Blog、2,000人規模でのスキル・ルーティン化・ROI評価の組織的実践) ※2026-06-18 fetch
- [Claude Code の MCP 連携 — 3 つの transport、scope の優先順位、企業向けコントロールまでの全体像](https://qiita.com/akihidem/items/405d5d07837c1e202af1) (Qiita akihidem、HTTP/Stdio/SSE の選択・5段階スコープ優先順位・managed-mcp.json によるエンタープライズ制御) ※2026-06-23 fetch

**コンテキスト2%ルール（Skill description の総量管理）**:
200K トークンのコンテキストウィンドウのうち Skill description 全体の総文字数を **約 16,000 文字（2%）** 以内に収める。超過すると実作業のためのトークンが不足する。定期的に下記スクリプトで計測する:

```bash
#!/bin/bash
# Skill description の文字数を集計する
SKILLS_DIR=~/.claude/skills
TOTAL=0
for skill_file in "$SKILLS_DIR"/*.md; do
  skill_name=$(basename "$skill_file" .md)
  # description = ファイル先頭の非見出し行のみを対象
  desc=$(awk '/^## /{exit} /^[^#]/{print}' "$skill_file" | head -5 | tr -d '\n')
  char_count=${#desc}
  TOTAL=$((TOTAL + char_count))
  printf "%-30s | %d 文字\n" "$skill_name" "$char_count"
done
echo "合計: $TOTAL 文字 / 上限目安: 16000 文字"
```

**Iron Law（品質ゲート）**:「時間がない」「小さな変更だから」などの合理化を防ぐため、Skill 修正後は必ず ① 公式ベストプラクティス照合 → ② 3層構造（description/SKILL.md/references）の確認 → ③ 検証サイクル埋め込み確認 の3ステップを踏む。

**出典引用**:
> 「このSkillが何をするか」を20〜50文字で伝えるだけにします。詳細な手順は `SKILL.md` に書き、Skill実行時に初めてロードします。
> ([Claude Code Skills設計パターン ： 段階的開示とコンテキスト2%ルール](https://zenn.dev/correlate_dev/articles/claude-code-skills-progressive-disclosure), セクション "Skills設計の3層構造") ※2026-06-15に実際にfetch成功

> 「どれだけ長く使い込むかが自動化の深さを決める」
> ([約2,000人が使うClaude Codeと向き合う。](https://developers.cyberagent.co.jp/blog/archives/64334/), CyberAgent Engineering Blog、セクション "手作業はスキルに、繰り返し作業はループやルーティンにする") ※2026-06-18に実際にfetch成功

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 高
**最終更新**: 2026-06-23

---

### 12. Claude Code hooks を3層（セキュリティ / 状態管理 / 学習）で設計し、`async` と `exit 2` の非互換を把握する

hooks はイベント種別ごとに「何を目的にするか」を層で分けて設計する。Layer 0（セキュリティゲート）では `exit 2` でブロック、Layer 1（状態管理）では情報収集とコンテキスト注入、Layer 2（学習）では非同期の観測と失敗パターン分析を担う。
**`async: true` と `exit 2` は同時に使えない**——バックグラウンドプロセスは Claude の実行をブロックできないため、ブロック用フックは必ず同期実行にする。

**根拠**:
- `PreToolUse` + `exit 2` でシークレット検出・予算超過・破壊操作を機械的にブロックできる
- `UserPromptSubmit` でキーワードマッチしてドキュメントやコンテキストを自動注入し、毎回の手動指示を省ける
- `SessionStart` フックでポリシーファイルを stdout に出力すると、`/clear` 後もセッション毎にコンテキストが自動復元される（「/clearが怖くなくなる」効果）。stdout 上限は約10,000文字 — 超える場合は同スクリプトを `-Part 0`, `-Part 1` 等に分割して複数フックで並列実行する。ポリシーを英語テレグラフ形式で記述すると60〜70%トークン削減できる
- `Stop` フックで会話終了時に git の未 push 状態を表示すると、コミット漏れを防げる
- `Stop` フックはループガード（暴走時のハードストップ）にも使える—同一行が8回以上繰り返される場合に `{"continue": false}` を返して強制停止。「自己制御は自己が壊れたとき機能しない」ため外部 failsafe が必要
- `PreCompact` フックでコンテキスト圧縮前にセッションログを `_inbox/` へ退避し、情報損失を防げる
- `async: true` にするとバックグラウンドで実行されるが `exit 2` の効果がなくなる—ロギング目的のみに使う
- 導入は副作用の小さい順に段階的に行う—`Stop`（完了通知）から始め、動作確認後に `PostToolUse`（軽量な後処理）、最後に `PreToolUse`（ブロック）を足す。`PostToolUse` は毎回実行されるため設定ミスの影響が大きく、いきなり入れない。各 Hook のコマンドは3秒以内に終わる処理に限る（重い処理は別ツールへ）
- CLAUDE.md に書いた禁止操作は Claude が「この文脈では例外」と解釈し得る（口頭の禁止は破られる）—`rm -rf` や `git push --force` などの破壊的コマンドは Layer 0 の `PreToolUse` フックでコマンドパターンをマッチして物理的にブロックし、Claude の判断を迂回しない制約にする。`.claude/settings.json` に含めればチームのポリシーとして共有できる
- **冪等性チェックの必須化**: 実行後の検証フェーズでは「同じスクリプトを2回実行してもデータが重複しないか」を必ず確認する。AI が「完了した」と言うだけでなく実際のログ・diff・クエリ結果などの証拠を表示させることで、「流暢だが未検証の主張」によるインシデントを防ぐ

**フック層の分担**:
| 層 | イベント | 目的 | `exit 2` |
|---|---|---|---|
| Layer 0（セキュリティ） | `PreToolUse` | 危険操作ブロック・シークレット検出 | 使用可 |
| Layer 1（状態管理） | `UserPromptSubmit`, `PostToolUse`, `SessionStart` | コンテキスト注入・バリデーション・ポリシー永続化 | 使用可 |
| Layer 2（学習） | `Stop`, `PreCompact` | ログ退避・git 状態表示 | **使用不可**（async 推奨）|

**MCP とフックの使い分け**（コンテキスト依存か、必須の強制制約か）:
| 用途 | 適切な手段 |
|---|---|
| DB クエリ・Issue 参照などコンテキスト依存の操作 | MCP サーバー |
| 強制 lint・センシティブファイル保護など必須の制約 | `PreToolUse` / `PostToolUse` フック |
| CLAUDE.md に書いた禁止操作を実際に強制する | `PreToolUse` フック |

**コード例**:
```python
# Layer 0: シークレット検出フック（PreToolUse）
import sys, re, json
sys.stdout.reconfigure(encoding="utf-8")  # Windows 環境での文字化け防止（必須）
sys.stderr.reconfigure(encoding="utf-8")

input_data = json.load(sys.stdin)
tool_input = input_data.get("tool_input", {})
content = json.dumps(tool_input)

PATTERNS = [r'sk-[a-zA-Z0-9]{48}', r'sk-ant-[a-zA-Z0-9-]{90,}']
for pattern in PATTERNS:
    if re.search(pattern, content):
        print("シークレットが含まれています。コミット前に確認してください。", file=sys.stderr)
        sys.exit(2)  # Claude の実行をブロック
sys.exit(0)
```

```bash
# Layer 1: キーワードでドキュメントを自動注入（UserPromptSubmit）
#!/bin/bash
PROMPT=$(jq -r '.prompt' <<< "$CLAUDE_TOOL_INPUT")
if echo "$PROMPT" | grep -qE 'n8n|workflow'; then
  cat .claude/skills/n8n-guide.md  # stdout への出力がコンテキストに追加される
fi
exit 0
```

```bash
# Layer 2: Stop フックで git 状態を表示（async 不要、exit 2 使わない）
#!/bin/bash
for repo in . ../sub-module; do
  modified=$(git -C "$repo" diff --name-only | wc -l)
  unpushed=$(git -C "$repo" log @{u}.. --oneline 2>/dev/null | wc -l)
  echo "[$(basename $repo)] 変更:$modified 未push:$unpushed"
done
exit 0
```

```bash
# Loop guard: Stop フックで同一行反復暴走を検出してハードストップ
# settings.json に "Stop": [{"hooks": [{"type": "command", "command": "/path/loop_guard.sh"}]}] で登録
#!/bin/bash
THRESHOLD=8
last_text=$(jq -r '.last_text // ""' <<< "$CLAUDE_TOOL_INPUT" 2>/dev/null)
maxrep=$(printf '%s\n' "$last_text" | sed 's/^[[:space:]]*//; s/[[:space:]]*$//' | \
         grep -v '^$' | sort | uniq -c | sort -rn | head -1 | awk '{print $1}')
if [ -n "$maxrep" ] && [ "$maxrep" -ge "$THRESHOLD" ]; then
  printf '{"continue": false, "reason": "loop guard: same line repeated %s times"}\n' "$maxrep"
  exit 0  # 判定不能時は exit 0 でセッション継続（failsafe design）
fi
exit 0
```

```bash
# Layer 2: PreCompact でセッションログを退避
#!/bin/bash
DEST="_inbox/claude-sessions/$(date +%Y%m%d-%H%M%S).md"
mkdir -p _inbox/claude-sessions
cp "$CLAUDE_SESSION_FILE" "$DEST" 2>/dev/null || true
exit 0
```

**Layer 0 強化: gitleaks + PII 検出の統合**:
```bash
# core.hooksPath に設定する共有 pre-commit フック
# gitleaks でシークレットをスキャンし、PII パターンも検出する
#!/bin/bash
set -e

# 1. gitleaks でシークレット検出（API キー・トークン等）
gitleaks git --pre-commit --staged --redact --verbose . || {
  echo "ERROR: gitleaks がシークレットを検出しました。コミットをブロックします。" >&2
  exit 1
}

# 2. PII パターン検出（メールアドレス・クラウドパス・UUID）
PII_REGEX='[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}|/Users/[a-zA-Z0-9_\-]+/|[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}'
if git diff --cached | grep -qE "$PII_REGEX"; then
  echo "WARNING: PII パターンが検出されました。意図的なコミットか確認してください。" >&2
  exit 1
fi
exit 0
```
- Claude Code/Codex ともに `core.hooksPath` を共通フックディレクトリに向けることで、ツール間で一貫したセキュリティゲートを維持できる
- PR/Issue の本文へのリーク防止はツール固有のフック（`PostToolUse`）で別途カバーする必要がある
- **Stop フック: leaked tool call 検出**：Claude が XML 形式のツール呼び出しを実行せずにプレーンテキストとして漏洩させるリグレッション（few-shot 自己汚染または1ターン複数呼び出し時の解析崩壊）を `Stop`/`SubagentStop` フックで検出し、再発行を促す。**漏洩マークアップを自分でパースして実行することは厳禁**（プロンプトインジェクション脆弱性になる）

```python
# Layer 2（補強）: leaked XML tool call 検出 Stop フック
import sys, re, json

data = json.load(sys.stdin)
last_text = data.get("stop_reason", {}).get("response", {}).get("text", "")

# XML 形式のツール呼び出しが残っている場合は再発行を要求
if re.search(r'<(invoke|function_calls|tool_call)\b', last_text):
    print(json.dumps({
        "continue": True,
        "reason": "leaked tool call detected: re-issue as structured tool calls"
    }))
else:
    print(json.dumps({"continue": False}))
```

**出典引用**:
> 「このサイレント変種が複数回発生しました。発生したのはいずれも1つの応答に複数のEdit呼び出しをまとめたときで、1編集ずつに分けるとそのまま通りました。」
> ([Claude Codeでツール呼び出しが実行されず生テキストとして表示されるときの対策](https://zenn.dev/ultimatile/articles/claude-code-leaked-tool-call-stop-hook), セクション "サイレント変種") ※2026-06-10に実際にfetch成功

> "指示する」から「設計する」に変わった瞬間、世界が変わった。指示は消耗する。設計は蓄積する。"
> ([Claude Code hooksを47本実装した話：AIへの自動指示を設計するという仕事](https://zenn.dev/thinkyou0714/articles/claude-code-hooks-47), セクション "設計への転換") ※2026-05-17に実際にfetch成功

> "生ログを手元に残すほうが情報損失が少ない"
> ([Claude Code hookをauto-recapより先に試すべき3パターン](https://zenn.dev/joemike/articles/claude-code-hooks-auto-recap-alternative-2026), セクション "PreCompact フック") ※2026-05-17に実際にfetch成功

> "gitleaks git --pre-commit --staged --redact --verbose ."
> ([Claude Code と Codex の両方に、機密情報と個人情報を漏らさせない hook を作った話](https://zenn.dev/todayama_r/articles/multi-agent-secret-pii-guard-hooks), Zenn, セクション "Layer 1 共有フック") ※2026-05-31に実際にfetch成功

> "「触ったついでに全体整形」が最も事故りやすい"
> ([Claude Codeのフォーマッタフックが1行修正を316行diffに肥大化させた話と復旧手順](https://zenn.dev/cutlet_of_pork/articles/claude-code-format-hook-scope-creep), セクション "教訓：AI作業のスコープ逸脱を検知・防御する") ※2026-06-14に実際にfetch成功

**PostToolUse フォーマッタの落とし穴**:
`PostToolUse` で `ruff format` / `prettier --write` 等のファイル全体整形ツールを自動実行すると、1行修正が数百行 diff に肥大化し「1コミット = 1つの意図」の原則が崩れる。git blame の可読性とリバート容易性が失われる。対策: 変更行のみを対象とするオプション（`eslint --fix` 等）を使うか、整形はフック経由ではなく明示的な skill 呼び出しに分離する。

**出典**:
- [Claude Code hooksを47本実装した話](https://zenn.dev/thinkyou0714/articles/claude-code-hooks-47) (Zenn) ※2026-05-17 fetch
- [Claude Code hookをauto-recapより先に試すべき3パターン](https://zenn.dev/joemike/articles/claude-code-hooks-auto-recap-alternative-2026) (Zenn) ※2026-05-17 fetch
- [Claude Code と Codex の両方に、機密情報と個人情報を漏らさせない hook を作った話](https://zenn.dev/todayama_r/articles/multi-agent-secret-pii-guard-hooks) (Zenn、gitleaks + PII pattern + multi-agent 対応) ※2026-05-31 fetch
- [Hooksで自動化する——「毎回同じ指示」から卒業する【中級Ch4】](https://zenn.dev/shun_producer/articles/1c63765ff7524a) (Zenn、段階的導入順序 Stop→PostToolUse→PreToolUse) ※2026-06-01 fetch
- [Claude Codeの処理が終わったら音で通知！Stop Hookで作業効率を劇的に改善する](https://zenn.dev/lumichy/articles/claude-code-stop-hook-sound-2026) (Zenn、Stop フックでの完了通知) ※2026-06-01 fetch
- [Claudeが暴走してトークンを溶かした話 — 「同一行反復」を止める二段防御](https://zenn.dev/fixu/articles/claude-loop-guard-token-burn) (Zenn、Stop フック ループガード実装・failsafe 設計・THRESHOLD=8 スクリプト) ※2026-06-02に実際にfetch成功
- [Claude Code を「補完」ではなく「運用」する — .claude/ 設計と実プロジェクトの落とし穴](https://zenn.dev/agotoh/articles/7cf2239a76b5e7) (Zenn、`PreToolUse` フックで禁止操作を物理ブロック・「口頭の禁止は破られる」) ※2026-06-03 fetch
- [Claude Codeをチームで運用するためのCLAUDE.md設計とカスタムエージェント分担](https://zenn.dev/siromiya/articles/2026-06-03-01-claudecode-team-agents) (Zenn、MCP vs Hooks 判断フレームワーク) ※2026-06-03 fetch
- [Claude Codeでツール呼び出しが実行されず生テキストとして表示されるときの対策](https://zenn.dev/ultimatile/articles/claude-code-leaked-tool-call-stop-hook) (Zenn、Stop フックによる leaked XML tool call 検出) ※2026-06-10 fetch
- [【後編】Claude Codeの暴走を止める ── 「お願い（CLAUDE.md）」と「物理的な壁（settings.json）」の二段構え](https://zenn.dev/hi_met/articles/f13088230d9e64) (Zenn / hi_met、冪等性チェック必須化・証拠ベース検証・具体的ブロック操作リスト) ※2026-06-11 fetch
- [Claude Codeのフォーマッタフックが1行修正を316行diffに肥大化させた話と復旧手順](https://zenn.dev/cutlet_of_pork/articles/claude-code-format-hook-scope-creep) (Zenn、PostToolUse 全ファイル整形アンチパターン・スコープ逸脱防止) ※2026-06-14 fetch
- [Claude Code の /clear が怖くなくなる — Hook でポリシーを自動注入する実装](https://zenn.dev/dev_shota/articles/claude-code-session-load-hook-impl) (Zenn dev_shota、SessionStart フックによるポリシー永続化・stdout 上限チャンク分割) ※2026-06-20 fetch

> "セッション開始時にスクリプトを自動実行して、その出力を Claude のコンテキストに注入できます。つまり、ポリシーファイルを用意しておけば、/clear しても次のセッションで勝手に復元される"
> ([Claude Code の /clear が怖くなくなる — Hook でポリシーを自動注入する実装](https://zenn.dev/dev_shota/articles/claude-code-session-load-hook-impl), セクション "仕組みの概要") ※2026-06-20に実際にfetch成功

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 高
**最終更新**: 2026-06-20

---

### 13. 組織での Claude Code 導入を Managed Settings・最小権限・OpenTelemetry 監査の3層で制御する

Claude Code を組織全体で安全に展開するには、個人設定で上書きできない **Managed Settings**、
**最小権限の原則**（使うものだけを有効にする）、そして **OpenTelemetry 監査**（初日から証跡を残す）の3層を組み合わせる。
1層でも欠けると、個人が意図せず規制を回避したり、インシデント後の追跡が不可能になる。

**根拠**:
- `disableBypassPermissionsMode: 'disable'` を Managed Settings に設定しないと、個人が無制限権限モードを有効化できる
- `.gitignore` だけでは不十分：ローカルに実在する `.env`・`~/.ssh/**`・`~/.aws/**` への Read アクセスは Deny ルールで明示的に禁止する
- OS サンドボックスは `failIfUnavailable: true` にしないと、未対応環境でサイレントに無効化される
- `CLAUDE_CODE_ENABLE_TELEMETRY=1` を後付けにすると、インシデント発生前のログが存在せず追跡不可能になる
- 最小権限は「後で追加する」前提で始める：拡張機能も半年ごとに棚卸しし、未使用を無効化する
- `.claudeignore` は Deny ルールと補完する形で使う。「漏れたら困るもの」ではなく「業務上 Claude に読ませる必要がないもの」という基準で記載する（テストフィクスチャ、ビルド成果物、大容量ログ等）
- **`Tool(param:value)` 構文でサブエージェントのモデル使用を制限できる**（Claude Code v2.1.178+）: `Agent(model:opus)` を deny に設定すると、ネストしたサブエージェントが Opus モデルを起動するのをブロックできる。ワイルドカード `Agent(model:*opus*)` で複数バリアントに対応。コスト管理とモデルポリシーの強制に有効
- **`permissions.deny` の3つの落とし穴**（書いても効かない代表例）:
  1. **設置場所の誤り**: `~/.claude/settings.local.json` はユーザーレベルでは読まれない。個人専用の強制 deny は `/Library/Application Support/ClaudeCode/managed-settings.json`（macOS）に配置する
  2. **コマンドチェーンの解釈**: `rm -rf` が deny されていても `xargs rm -rf` は通過する。deny はコマンドを `|` / `&&` で分割して単独評価するため、`Bash(xargs rm:*)` のようにチェーンの先頭コマンドからパターンを書く
  3. **テスト不省略**: deny ルールを書いても実際にサンドボックス環境でテストしないと「効いていない安全策」になる。存在しないパスへの `rm` を試して本当にブロックされるか確認する
  4. **Bash 経由の git コマンドはすり抜ける**: `deny` は `Read` ツールにしか効かない。`git diff HEAD -- <file>`・`git show HEAD:<file>`・`git log -p <file>` のような Bash ツール経由のコマンドは `deny` をバイパスして保護ファイルの内容を読み取れる。保護が必要なファイルは `PreToolUse` フックで Bash コマンドのパスも検査する必要がある

**コード例（MCP ツールの最小権限化）**:
```json
// .mcp.json — MCP サーバー本体を定義（プロジェクト共有スコープ）
{
  "mcpServers": {
    "accounting-mcp": { "command": "npx", "args": ["accounting-mcp"] }
  }
}
```

```json
// .claude/settings.json — permissions.allow で使うツールだけを許可リスト化
// 接続＝全ツール全開放がデフォルト。mcp__<server>__<tool> 形式で列挙する
{
  "permissions": {
    "allow": [
      "mcp__accounting-mcp__list_account_items",
      "mcp__accounting-mcp__create_deal"
    ]
  }
}
```

**コード例（Managed Settings 最小安全設定）**:
```json
// managed-settings.json（組織統制ファイル）
{
  "disableBypassPermissionsMode": "disable",
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(~/.ssh/**)",
      "Read(~/.aws/**)",
      "Agent(model:opus)",
      "Agent(model:*opus*)"
    ]
  },
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true,
    "allowedDomains": [
      "registry.npmjs.org",
      "github.com",
      "api.github.com"
    ]
  }
}
```

```bash
# OTel 監査（初日から有効化）
export CLAUDE_CODE_ENABLE_TELEMETRY=1
```

**最小権限チェックリスト**:
- [ ] 使わない MCP Server は無効化する（`disabled: true`）
- [ ] MCP サーバーを有効にする際は `permissions.allow` に `mcp__<server>__<tool>` を列挙して必要なツールのみ許可する（接続＝全ツール全開放がデフォルト）。270個→12個に絞ると不要ツール定義の読み込みによるトークン浪費も消える
- [ ] 信頼できないソース（外部URL・ユーザー入力）をAIに渡す前に、ファイル変更・git push・API呼び出しを一時無効化する
- [ ] 拡張機能は半年ごとに棚卸しして未使用を削除する
- [ ] `curl`/`wget` を Bash deny に追加する（ワイルドカードパターン回避より確実）
- [ ] `.claudeignore` で「業務上 Claude に読ませる必要がないもの」を Deny ルールと補完する形で除外する
- [ ] deny ルールを書いたら、サンドボックス環境で実際にブロックされるかテストする（書いて安心は禁物）
- [ ] コマンドチェーン（`xargs rm`、`find ... -delete` 等）はチェーン先頭コマンドからパターンを書く
- [ ] `deny` で保護したファイルパスが `git diff`/`git show`/`git log -p` 等の Bash コマンドからもアクセスできないか確認する（deny は Read ツールにしか効かない）

**出典引用**:
> "すべての層を強制力のあるレベルで設定する" — `disableBypassPermissionsMode: 'disable'` が Managed Settings のコア設定
> ([Claude Codeを社内に安全に導入する：システムエンジニアのためのセキュリティ実践ガイド](https://zenn.dev/nocodesolutions/articles/f71479fa5711fe), セクション "4層モデル") ※2026-05-18に実際にfetch成功

> "信頼できないソースをAIに読ませるときは、その会話でファイル変更・git push・API呼び出しができない状態にしてから渡します"
> ([AIエージェントは最小権限で使う｜Claude Code・MCP・VS Code拡張の安全な設定](https://zenn.dev/takibilab/articles/ai-agent-least-privilege), セクション "信頼できないソースの扱い") ※2026-05-18に実際にfetch成功

> "『漏れたら困るもの』ではなく『業務上 Claude に読ませる必要がないもの』という基準で書く"
> ([.claudeignore の正しい書き方：Deny ルールとの使い分けと「読ませない」設計思想](https://zenn.dev/siromiya/articles/claudeignore-design-guide), セクション ".claudeignore の設計思想") ※2026-05-20に実際にfetch成功

> "効いていない安全策は、無いよりむしろ危険かもしれない"
> ([permissions.deny を書いたのに効いていなかった — Claude Code で安全床を作る3つの落とし穴](https://zenn.dev/taketakekaho/articles/2e35f53184f336), セクション "書いた deny が、実は効いていませんでした") ※2026-06-06に実際にfetch成功

> "gitコマンド経由で同じ情報にアクセスできてしまう"
> ([Claude Codeのdenyはgitコマンドをすり抜ける——hooksで塞いだ話](https://zenn.dev/makkyemmanuel/articles/zenn-article-claude-code-hooks-git-guard), セクション "問題の発見") ※2026-06-12に実際にfetch成功

**在席/無人セッションでパーミッション戦略を切り替える（個人・開発者視点）**:

組織の Managed Settings に加え、**セッションに人間がいるか否か**でパーミッション設定を切り替えると、安全網を外さずに確認疲れを減らせる:

| 状況 | モード | 考え方 |
|------|--------|--------|
| 在席（開発者が見ている） | `auto`（デフォルト） | 機械的に安全な操作は自動承認。残った重要操作だけ人間が判断 |
| 無人（Routine / CI / 夜間実行） | `dontAsk` + hooks で代替 | 確認プロンプトが出てもロボットは答えられない。Deny/Allow ルールで完全に機械化する |

- 在席時: 「確認疲れ研究で93%のプロンプトは精査なく承認される」→ 問いかけ回数を減らし、残った問いを真剣に判断する
- 無人時: エージェントが拒否されると自己修正ループに入れないため、`dontAsk` + PreToolUse hooks で安全境界を構造的に定義する
- **両モード共通の基盤**: フック → Deny → Ask → Allow の評価順序、シークレット2層保護（`server-only` + 命名規約）、sandbox 封じ込め

> "安全で許可済み → 走る / それ以外 → プロンプトでなく即 deny"
> ([Claude Code の許可プロンプトを安全網を外さず減らす — 在席と無人で最適解は違う](https://zenn.dev/kojisumiyoshi/articles/claude-code-permission-prompts-guide), セクション "在席/無人の分岐") ※2026-06-16に実際にfetch成功

**出典**:
- [Claude Codeを社内に安全に導入する：システムエンジニアのためのセキュリティ実践ガイド](https://zenn.dev/nocodesolutions/articles/f71479fa5711fe) (Zenn、元出典) ※2026-05-18に実際にfetch成功
- [AIエージェントは最小権限で使う｜Claude Code・MCP・VS Code拡張の安全な設定](https://zenn.dev/takibilab/articles/ai-agent-least-privilege) (Zenn、信頼できないソースの扱い) ※2026-05-18に実際にfetch成功
- [.claudeignore の正しい書き方：Deny ルールとの使い分けと「読ませない」設計思想](https://zenn.dev/siromiya/articles/claudeignore-design-guide) (Zenn、.claudeignore 設計思想) ※2026-05-20に実際にfetch成功
- [Claude Codeのdenyはgitコマンドをすり抜ける——hooksで塞いだ話](https://zenn.dev/makkyemmanuel/articles/zenn-article-claude-code-hooks-git-guard) (Zenn、deny の Read ツール限定問題と PreToolUse git-guard フックによる解決) ※2026-06-12 fetch
- [Claude Code の許可プロンプトを安全網を外さず減らす — 在席と無人で最適解は違う](https://zenn.dev/kojisumiyoshi/articles/claude-code-permission-prompts-guide) (Zenn kojisumiyoshi、在席/無人モード切替) ※2026-06-16 fetch
- [Claude Code 2.1.178の権限ルールはツール引数までマッチできる](https://zenn.dev/okssusucha/articles/20260617-claude-code-tool-param-permission-syntax) (Zenn、Tool(param:value) 構文でサブエージェントのモデル制限) ※2026-06-17 fetch
- [MCPサーバーの権限を絞ったら、AIが触れる範囲が初めて見えた](https://zenn.dev/kenimo49/articles/mcp-permission-scope-minimization-design) (Zenn kenimo49、permissions.allow による MCP 内ツール単位の最小権限・Read/Write 分離・マトリクス設計) ※2026-06-20 fetch

> "つなぐ＝全許可。デフォルトが全開放だったのです"
> ([MCPサーバーの権限を絞ったら、AIが触れる範囲が初めて見えた](https://zenn.dev/kenimo49/articles/mcp-permission-scope-minimization-design), セクション "全ツール許可が起こす、静かな越境") ※2026-06-20に実際にfetch成功

**バージョン**: Claude Code v2.1.172+（ネスト型サブエージェント）、v2.1.178+（Tool(param:value) 構文）、Enterprise Managed Settings 対応環境
**確信度**: 中
**最終更新**: 2026-06-20

---

### 14. 大規模タスクのコンテキスト消費は70%/85%を閾値として管理し、設計決定・進捗・文脈を`.claude/`ファイルで永続化する

Claude Code のコンテキストウィンドウを「記憶」として使うと、大規模リファクタリングや長期タスクでは
70% 超え以降に「コンテキストロット」（設計決定の忘却・矛盾の増加）が起き、85% 超えでコンパクションループや損失が発生する。
コンテキストウィンドウは「作業スペース」として扱い、永続化が必要な情報は必ず `.claude/` 配下のファイルに外部化する。

**根拠**:
- ファイル1件の読み込み ≈ 3,000 トークン。100ファイルのモノリスは200Kウィンドウを超過する
- 70%: 応答速度の低下・確認頻度の増加など最初の症状が出る → この段階で外部化を実施する
- 85%: 自動コンパクションループ・データ損失が起き始める → この段階では手遅れ
- セッション間の引き継ぎを外部ファイルに書くと、複数セッション・並列ワークツリーが同じ決定を参照できる
- Plan Mode（設計フェーズのみで使用）を先行させるとトークン消費を40〜60%削減できる
- CLAUDE.md でステートファイル保存先パスを **明示的に指定する**。「シリアライズしてください」という抽象的な指示だけだと、ネストしたプロジェクト構造でサブディレクトリに保存され、ルートの状態ファイルが古いままになる事故が起きる

**コード例**:
```markdown
# .claude/architecture-map.md — パッケージ依存関係の記録
## Package Dependencies
- domain.user → domain.payment（暗黙依存、分離必要）
- domain.payment → infrastructure.stripe（直接参照、抽象化必要）

## Module Split Priority
1. Payment（外部依存が多い・変更頻度高）
2. User（クロスモジュール参照が最多）
3. Notification（独立・リスク最低）
```

```markdown
# .claude/handoff.md — ワークツリー内・セッション間引き継ぎ
## Current State
- Payment モジュールリファクタリング: 40% 完了
- Next: domain.user → domain.payment の依存を切り離す

## Files Modified
- packages/payment/src/index.ts（リファクタリング済み）
- packages/user/src/service.ts（作業中）

## Next Actions
- [ ] IPaymentRepository インターフェースを定義
- [ ] UserPaymentService を packages/user から分離
```

**セッション設計のベストプラクティス**:
- タスクをフェーズ分割: 分析 → インターフェース定義 → 実装（各フェーズで1セッション）
- 1セッション当たりのファイル変更は5〜20件に抑える
- `HANDOFF.md` にフェーズ完了時の状態・次アクションを記録してからセッションを閉じる
- **セッションは消えうる前提で設計する**: 二層永続化（コード成果 → GitHub、思考記録 → 作業ファイル）を実践し、セッション喪失時は親セッションが `git log` / `git status` / PR 一覧 / 作業ファイルからトレースを再構成する
- 復旧コスト公式 `= 復旧時間 × 障害頻度` を意識し、閾値超えたら復旧設計の改善より実行環境変更（CLI → デスクトップ等）を優先する

**アンチパターン**:
- コンテキスト使用率が 85% を超えてから新規セッションを開始する（設計記憶が消える）
- 設計判断を「Claude が覚えている」として外部化しない
- 新規セッションに「前回の続き」とだけ伝える（「現状・次のステップ・参照ファイル・確定済み設計決定」の4要素を含む再起動プロンプトが必要）

**CLAUDE.md での状態ファイルパス明示例**:
```markdown
## 状態ファイル管理
- 作業状態は必ず **プロジェクトルートの `.claude_state.md`** を更新する
  - パス: /path/to/project-root/.claude_state.md  ← 絶対パスまたはルートからの相対パスを明示
  - 「シリアライズしてください」だけでは不十分。保存先ファイルパスを明記すること
- サブプロジェクトの詳細は projects/{name}/.claude_state.md に書き、ルートには要約を記録する
```

**出典引用**:
> "コンテキスト使用率 **70% が「手を打つべき」ライン。85% を超えたら手遅れ**"
> ([Claude Codeのコンテキスト枯渇に立ち向かう──大規模リファクタリングのセッション設計](https://zenn.dev/miyan/articles/claude-code-context-exhaustion-strategy), セクション "コンテキストロットの3段階") ※2026-05-23に実際にfetch成功

> "バトンタッチとは、エージェントを並べることではなく **作業の文脈に持ち運び可能な実体を与えること** だった"
> ([worktree の『バトンタッチ』問題を、自前 skill・Codex・Augment で比べて反省した](https://zenn.dev/yasunami_daichi/articles/worktree-handoff-codex-augment), セクション "結論") ※2026-05-23に実際にfetch成功

> "設計は実運用の中で完成に近づいていく。「シリアライズしてください」という指示だけでは保存先が曖昧で、サブディレクトリに書かれたステートがルートで参照されない事故が起きる。"
> ([Claude Codeのセーブ設計、実運用で踏んだ穴](https://zenn.dev/tsukasa_art/articles/claude-workflow-2), セクション "根本原因") ※2026-05-25に実際にfetch成功

> "個々の AI セッションは消えうる (ephemeral) と前提し"
> ([AIセッションは消えうる前提で設計する](https://zenn.dev/fixu/articles/ai-session-resilience-recovery), セクション "コアプリンシプル") ※2026-05-30に実際にfetch成功

**バージョン**: Claude Code（全バージョン）
**確信度**: 中
**最終更新**: 2026-05-30

---

### 15. 並列サブエージェント実行前にgrill-with-docsで暗黙前提を外部化し、CONTEXT.mdとADRで受け渡す

Claude Code のサブエージェントはメイン会話のコンテキストを一切継承しない空の状態で起動する。
「会話で決めたこと」は書かれていなければサブエージェントにとって存在しないのと同じため、
並列実行の前に必ず「grill-with-docs」フェーズでプランを問い詰め、暗黙の前提を `CONTEXT.md` や ADR として外部化する。

**根拠**:
- "Subagents inherit zero context from the main conversation. What wasn't written in the instructions simply doesn't exist to them."
- grill-with-docs はプランを「問い詰め（grill）」て、用語定義・設計決定・制約を洗い出し文書化する
- 外部化した `CONTEXT.md` + ADR があれば、セッション跨ぎ・ツール跨ぎ（Codex / Augment）での引き継ぎも可能
- grill フェーズを省いて委任すると「薄い指示が薄い結果を生む」—再作業コストは節約したトークンを上回る

**コード例**:
```
# 並列サブエージェント起動の手順
1. /grill → プランを問い詰め、前提を CONTEXT.md と ADR に書き出す
2. git worktree add -b feature/X ../worktree-X  （タスクごとにブランチを分離）
3. 各サブエージェントに委任: "Read CONTEXT.md and ADR-001.md, then implement Y"
```

```markdown
# CONTEXT.md — サブエージェント向け共有コンテキスト
## 用語定義
- PaymentService: ユーザーから直接呼ばれる公開サービス（domain.payment に存在）
- IPaymentRepository: domain.payment が依存するインターフェース（まだ未定義）

## 設計決定（ADRs）
- ADR-001: UserPaymentService を packages/user から packages/payment に移動する
- ADR-002: domain.user は domain.payment に直接依存しない

## 実装の境界
- subagent A: packages/payment のインターフェース定義
- subagent B: packages/user の依存クリーン
```

**アンチパターン**:
- grill フェーズを省いてサブエージェントに委任する（プランの細部が「暗黙前提」のまま渡る）
- `CONTEXT.md` を会話の要約として書く（問い詰めずに書いた要約は暗黙前提を含んだまま）
- 全タスクを1つのサブエージェントに渡す（コンテキスト分離の恩恵がなくなる）

**出典引用**:
> "Subagent を並列で回す前に、やることがある。**頭の中の暗黙の前提を、subagent が読める形に外部化すること**"
> ([独立コンテキストの subagent には grill-with-docs を渡せ](https://zenn.dev/yasunami_daichi/articles/parallel-agent-context-grill-with-docs), セクション "並列委任の落とし穴") ※2026-05-23に実際にfetch成功

> "Skipping the 'grill' phase before parallel delegation amplifies thin instructions into thin results—tripling rework instead of output."
> ([独立コンテキストの subagent には grill-with-docs を渡せ](https://zenn.dev/yasunami_daichi/articles/parallel-agent-context-grill-with-docs), セクション "grill-with-docs とは") ※2026-05-23に実際にfetch成功

**バージョン**: Claude Code（全バージョン）
**確信度**: 中
**最終更新**: 2026-05-23

---

---

### 16. CLAUDE.md の構成と自動メモリの運用で、セッション越えの知識基盤を構築する

セッション開始時に自動読み込みされる CLAUDE.md は「チーム共通の恒久ルール」を置く場所。
自動メモリ（`~/.claude/projects/<project>/memory/`）は「個人作業で積み上がる文脈」を蓄積する。
長期タスクではさらに `backlog.md` + `MEMORY.md` を手動で保守し、次セッションへのバトンにする。
この3層を意識するだけで「セッションを閉じたら全部消えた」問題の大部分を解消できる。
チームで再現性を優先する場合は **`autoMemoryEnabled` を無効化する**：自動メモリはプロジェクトパスごとに個人ローカルに蓄積されるため、開発者Aと開発者BのClaudeが異なるメモリを持ち挙動が再現できなくなる。チームPoCや再現性を重視する場面では `.claude/settings.json`（Gitトラッキング対象）に `"autoMemoryEnabled": false` を設定して全員の挙動を揃える。個人設定は `.claude/settings.local.json`（`.gitignore` 対象）に書く。

```json
// .claude/settings.json（チーム共有 — Gitコミット対象）
{
  "autoMemoryEnabled": false
}
```

CLAUDE.md は**スコープ別に複数配置できる**: `~/.claude/CLAUDE.md`（個人グローバル：ロール・ツール選択）→ `{repo}/CLAUDE.md`（プロジェクト共通：アーキテクチャ方針・禁止操作）→ `{repo}/{subdir}/CLAUDE.md`（サブディレクトリ局所制約）。チーム共有すべきルールはリポジトリルートに、個人スタイルはグローバルに分離することで、ファイルの肥大化と個人情報の混入を防ぐ。サブディレクトリ CLAUDE.md は「そのディレクトリ以下でのみ有効」な制約を書く場所として、モジュール別の独立したルール管理を実現する。

**CLAUDE.md に含めるべきもの**:
- ディレクトリ構成・エントリポイント
- ビルド・テスト・lint コマンド
- 命名規約・設計根拠（「Redux ではなく Zustand を選んだ理由」等）
- デプロイ手順・API 契約上の制約
- 禁止操作（「`.env` を git にコミットしない」「`git push` 禁止」等）— 含める判断軸: **「この行を削除したら Claude は間違えるか？」Yes なら含める**

**CLAUDE.md に含めないもの**:
- `git blame` / `grep` で取り出せる情報（コードコメントに属する）
- スプリント内容・担当者等の頻繁に変わる情報（トークンを消費するだけでノイズになる）
- 長大な説明（`.claude/rules/` に委譲して目次化する — Rule #6 参照）
- **ミスログや自己改善記録など頻繁に追記されるファイル**: プロンプトキャッシュは先頭からの完全一致 prefix ベースで動作するため、常駐ファイルへの1行追記がそれ以降全体のキャッシュを無効化する。常駐は「静的・安定・全セッション関連」の3条件を満たすもの（アーキテクチャ方針・禁止操作・コマンド一覧など）に絞り、頻繁に変わる観察記録は `.claude/knowledge/` 等の非常駐ファイルへ退避する

**量の目安**: 50〜150 行が適切。200 行を超えると Claude の注意力が分散しルール遵守率が低下する。2 週間に一度見直して陳腐化したエントリを削除する。

**自動メモリの4種（`~/.claude/projects/<project>/memory/`）**:
| 種別 | 内容 |
|---|---|
| **user** | 自分のロール・専門性・作業スタイルの好み |
| **feedback** | Claude への訂正・確認済みパターン |
| **project** | 現在のゴール・タイムライン |
| **reference** | 外部システムの場所・アクセス方法 |

自動メモリはチーム共有の CLAUDE.md と異なり個人が蓄積する文脈。コード場所・関数名はリファクタリングで腐りやすいので定期的に棚卸しする。

**1ファイル1事実＋インデックス方式（スケールする記憶設計）**:
MEMORY.md をすべての決定を詰め込む1ファイルとして使うと肥大化してコンテキスト汚染になる。代わりに「1事実1ファイル＋MEMORY.md をインデックス専用」に分割する。各セッションは MEMORY.md（インデックス）のみを読み込み、必要なファイルを個別にロードする:

```markdown
# MEMORY.md（インデックス専用 — 1行1ファイル）
- [数式記法ルール](feedback_math_notation.md) — LaTeX不可、Unicodeで代替
- [認証方式の決定](project_auth_decision.md) — JWT → Cookie-based へ変更（2026-06-10）
- [API仕様の場所](reference_api_spec.md) — /docs/api.yaml を参照
```

```markdown
---
name: feedback-math-notation
description: LaTeX ($$...$$) はレンダリング不可。Unicode で代替する
metadata:
  type: feedback
---
# 修正内容
数式は $$ ではなく Unicode 記号（×・÷・√等）または plain text で記述する
```

ポイント: description が「索引キー」なので、具体的に書く（「フォーマットルール」より「LaTeX不可・Unicode代替」）。重複防止のため新規ファイル作成前に既存インデックスを確認する。

**クロスセッション引き継ぎファイル（長期タスク用）**:
```markdown
# backlog.md — 次セッションへのタスク引き継ぎ（タスクリスト）
## 完了
- ✅ UserService の抽象化

## 進行中
- 🔄 PaymentRepository インターフェース定義（50%）

## 次にやること
- [ ] domain.user → domain.payment の依存を切り離す
```

```markdown
# MEMORY.md — 決定・理由の記録（タスクリストではなく「なぜか」の記録）
- Zustand ではなく Redux Toolkit を採用した理由: ミドルウェアが必要 (2026-05-24)
- API の pagination は cursor-based のみ（limit-offset は廃止済み, 2026-05-20）
```

**PR チェックリストへの組み込み**:
```markdown
## PR チェックリスト
- [ ] ルール変更・命名規約追加・新コマンドがあれば CLAUDE.md を更新した
- [ ] CLAUDE.md を更新した場合、対応する `.claude/rules/` ファイルも更新した
```

**CLAUDE.md のセキュリティ**:
CLAUDE.md の内容はシステムプロンプトと等価なので、API キー・内部 URL・認証情報を絶対に書かない。
公開リポジトリでは `.gitignore` に `CLAUDE.md`（と `.cursorrules`・`AGENTS.md`・`MEMORY.md`）を追加し、
チーム共有可能な部分だけを `CLAUDE.shared.md` に分離してコミットする運用が安全。

```gitignore
# ローカル専用 AI コンテキスト（機密情報を含む可能性）
CLAUDE.md
.cursorrules
AGENTS.md
MEMORY.md
```

**アンチパターン**:
- 「Claude が覚えているはず」でセッションを閉じる → 次セッションでゼロから説明が必要
- 自動メモリにコード場所・関数名を記録する → リファクタリングで即腐る
- CLAUDE.md に全ルールを直接書く → トークン過消費（Rule #6 の `paths` フロントマターで目次化）
- **「口頭で伝えた＝更新した」と思い込む（Coherence Bias 問題）**: Claude に「このツールはもう使わない」と会話で伝えても、memory ファイル・global-rules・過去の記録ファイルは自動更新されない。複数のファイルが同じ旧情報で整合している場合、Claude はその前提を疑いにくくなる（整合している文書が多いほど "証拠" として扱われる）。変更を口頭で伝えるだけで終わらせず、「どのファイルが影響を受けるか」を確認して同時更新するワンステップを必ず挟む
- CLAUDE.md に API キー・内部 URL を書いてコミット → 公開リポジトリで漏洩（.gitignore 必須）
- `AGENTS.md` と `CLAUDE.md` を symlink で共有する → Windows ユーザーは `core.symlinks=false`（デフォルト）のため symlink が9バイトのダミーファイルになり、指示が無警告で消える。クロスプラットフォームチームでは symlink の代わりに `CLAUDE.md` 冒頭に `@AGENTS.md` 1行を書くインクルード形式（Claude Code 公式機能）を使う
- チームリポジトリで `autoMemoryEnabled`（デフォルト有効）のまま運用する → 開発者ごとに異なる暗黙の文脈が蓄積し、同じコードで異なる挙動になり再現調査が困難になる

**出典引用**:
> "CLAUDE.md はリポジトリにコミットして新規メンバーの立ち上がりコスト削減にも使える。PR レビューの観点に『CLAUDE.md の更新が必要か』を入れるとルールの陳腐化を防げる"
> ([Claude Code を使い倒すための設定術：CLAUDE.md・自動メモリ・コンテキスト管理の3本柱](https://zenn.dev/tamai_hideyuki/articles/claude-code-config-best-practices), セクション "CLAUDE.md: プロジェクト文脈の確立") ※2026-05-24に実際にfetch成功

> "CLAUDE.md だけでは長期開発には足りない。backlog.md でタスクを引き継ぎ、MEMORY.md で『なぜそう決めたか』を残してからセッションを閉じないと、次セッションのエージェントにとって存在しないのと同じだった"
> ([#1 セッションを閉じたら全部消えた——Claude Codeと長期開発するための設計論](https://zenn.dev/yuichi1996/articles/2006df87f3209c), セクション "三ファイルシステム") ※2026-05-24に実際にfetch成功

> "「指示を改善する」より「環境を設計する」。Claude が間違えそうな箇所を CLAUDE.md の禁止ルールで先手を打つほうが、プロンプト調整を繰り返すより確実に機能する"
> ([Claude Codeを1ヶ月使って気づいた「指示の改善では解決しない問題」](https://zenn.dev/yamada_ai_dev/articles/claude-code-why-harness), セクション "環境を設計する") ※2026-05-26に実際にfetch成功

> "Claude Codeを使い始めたけど『なんか思ったより賢くない』と感じたことはありませんか？その原因はほぼ確実に CLAUDE.md の設計にあります。効果的な設計の目安は 50〜150 行、2 週間ごとに見直しを行う。"
> ([Claude CodeのCLAUDE.md設計完全ガイド — 上級者が実践する7つの原則](https://zenn.dev/streamsolty/articles/90ff7a49d1bc53), セクション "7原則") ※2026-05-26に実際にfetch成功

> "CLAUDE.mdに書いた内容は、毎回のClaudeへのシステムプロンプトと等価です。気密性の高い情報が書かれている場合はgitignoreに追加することを忘れずに"
> ([CLAUDE.mdをgitignoreに追加してセキュリティリスクを回避する](https://qiita.com/kenimo49/items/0d5fd778d0dad4e14f68)) ※2026-06-02に実際にfetch成功

**出典**:
- [Claude Code を使い倒すための設定術：CLAUDE.md・自動メモリ・コンテキスト管理の3本柱](https://zenn.dev/tamai_hideyuki/articles/claude-code-config-best-practices) (Zenn) ※2026-05-24 fetch
- [#1 セッションを閉じたら全部消えた——Claude Codeと長期開発するための設計論](https://zenn.dev/yuichi1996/articles/2006df87f3209c) (Zenn) ※2026-05-24 fetch
- [Claude Codeを1ヶ月使って気づいた「指示の改善では解決しない問題」](https://zenn.dev/yamada_ai_dev/articles/claude-code-why-harness) (Zenn) ※2026-05-26 fetch
- [Claude CodeのCLAUDE.md設計完全ガイド — 上級者が実践する7つの原則](https://zenn.dev/streamsolty/articles/90ff7a49d1bc53) (Zenn) ※2026-05-26 fetch
- [Claude Codeをチームで運用するためのCLAUDE.md設計とカスタムエージェント分担](https://zenn.dev/siromiya/articles/2026-06-03-01-claudecode-team-agents) (Zenn、3スコープ階層・MCP vs Hooks 判断) ※2026-06-03 fetch
- [Claude Code を「補完」ではなく「運用」する — .claude/ 設計と実プロジェクトの落とし穴](https://zenn.dev/agotoh/articles/7cf2239a76b5e7) (Zenn、モジュール分割・コンテキスト温存) ※2026-06-03 fetch
- [AGENTS.mdとCLAUDE.mdをsymlinkで統一したら、Windowsのメンバーだけ指示が消えていた——Git for Windowsの静かな罠](https://qiita.com/yurukusa/items/3fdb95d4eae990615961) (Qiita、クロスプラット symlink 罠・@include 代替) ※2026-06-13 fetch
- [Claude Code の常駐コンテキストを 62% 削減した話 — prompt caching 時代のハーネス設計](https://zenn.dev/tarowhitey/articles/claude-code-resident-context-diet) (Zenn、プロンプトキャッシュ prefix 無効化・3条件による常駐判定) ※2026-06-14 fetch
- [Claude Codeに「ファイルベースの永続メモリ」を持たせる：1ファイル1事実＋index方式](https://zenn.dev/ai_daigakusei/articles/claude-code-file-based-memory) (Zenn ai_daigakusei、1ファイル1事実パターン・インデックス分離設計・description設計の重要性) ※2026-06-15 fetch
- [チームでClaude Codeを使うなら、自動メモリを切る選択肢もありだと思った](https://zenn.dev/hknote/articles/133b00bee2cf23) (Zenn hknote、チーム再現性のための `autoMemoryEnabled: false`・`.claude/settings.json` vs `.claude/settings.local.json` の使い分け) ※2026-06-19 fetch
- [Claude Codeのmemoryファイルに更新漏れが3か所——「伝えた ≠ 更新した」の構造](https://zenn.dev/tottoko_hamu/articles/2026-06-15-100000) (Zenn tottoko_hamu、Coherence Bias 問題・複数ファイルの同期漏れパターン) ※2026-06-23 fetch

**出典引用**:
> "索引と本体を分けて、やっと回り始めました"
> ([Claude Codeに「ファイルベースの永続メモリ」を持たせる](https://zenn.dev/ai_daigakusei/articles/claude-code-file-based-memory), セクション "Critical Insight") ※2026-06-15に実際にfetch成功

> "開発者AのClaudeと開発者BのClaudeで、裏側に持っているメモリが違う状態になりえます"
> ([チームでClaude Codeを使うなら、自動メモリを切る選択肢もありだと思った](https://zenn.dev/hknote/articles/133b00bee2cf23), セクション "自動メモリの問題点") ※2026-06-19に実際にfetch成功

> "変更を『口頭で伝えた』だけで終わらせず、『どのファイルが影響を受けるか』を確認するワンステップを挟む"
> ([Claude Codeのmemoryファイルに更新漏れが3か所——「伝えた ≠ 更新した」の構造](https://zenn.dev/tottoko_hamu/articles/2026-06-15-100000), セクション "Fundamental Principle") ※2026-06-23に実際にfetch成功

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 中
**最終更新**: 2026-06-23

---

### 17. AI エージェントに本番エラーのコンテキストを渡し、自己修正サイクルを構成する

エージェントが「何が壊れているか」を知らない状態では自律的な修正は不可能。
Sentry MCP または Sentry CLI でエラーの詳細（スタックトレース・パンくず・再現手順）をエージェントコンテキストに取り込み、
修正案を **人間レビュー用の draft PR** として出力する設計にする（自動マージは行わない）。

**根拠**:
- エージェントの自律修正のボトルネックは知能ではなく「何が壊れたかを知らないこと」
- Sentry MCP はエラーデータをエージェントが直接読める形式で提供し、IDE 内で修正まで完結する
- draft PR + 人間レビューにより、エージェントの過誤が本番コードに直接入ることを防ぐ
- Sentry MCP は Claude Code・Cursor・Windsurf・VS Code の各エージェントで動作する

**コード例**:
```bash
# Sentry MCP を Claude Code に追加（ワンコマンド）
npx add-mcp https://mcp.sentry.dev/mcp

# または Sentry CLI を直接インストール
curl https://cli.sentry.dev/install -fsS | bash
```

**典型的なワークフロー**:
1. Sentry MCP 経由でエラー詳細（スタックトレース・ユーザー操作パンくず）を取得
2. エージェントがエラーの根本原因を解析
3. 修正案を **draft PR** として出力（自動マージは行わない）
4. 人間がレビュー・承認後にマージ

**バリエーション: バグ再現スキル（`repro`）**:
SDK やライブラリのバグ修正フローでは、修正前に「最小再現環境の構築」をエージェントに任せることで診断効率が向上する。
Sentry が社内実装している `repro` スキルの設計原則: issue URL からSDK言語・バージョンを解析し、言語ネイティブツール（uv / npm / bundle）で隔離ディレクトリ・ブランチを生成して再現手順を文書化する。複雑な再現に詰まった場合はエラーを説明して中断する（無理に進ませない）。

**アンチパターン**:
- エラーログへのアクセスなしにエージェントにバグ修正を依頼する（「何が壊れているか」を伝えずに依頼）
- エージェントの修正 PR を自動マージする（人間レビューなしの本番デプロイ）

**出典**:
- [Sentry Blog: Agents Need Production Context](https://blog.sentry.io/agents-need-production-context/) (Sentry公式ブログ) ※2026-05-27に実際にfetch成功
- [Works on my machine: how we use AI to reproduce reported bugs](https://blog.sentry.io/ai-bug-reproduction/) (Sentry公式ブログ、バグ再現スキルの設計) ※2026-06-08に実際にfetch成功

**確信度**: 高
**最終更新**: 2026-06-09

---

### 18. 大規模・複雑なタスクは Dynamic Workflows（/effort ultracode）で Claude に自律的にワークフローを設計させる

Claude Code の Dynamic Workflows は、タスク目標・リポジトリ構造・依存関係・CI 設定を分析してワークフローを実行時に動的に構築する機能。
人間がスキルを手動で組み合わせるのではなく、高レベルの目標を記述して `/effort ultracode` で委任するとClaude が最適な工程を自律設計する。
フレームワーク大規模移行・アーキテクチャレビュー・大規模リファクタリングのような「作業工程自体が不確定なタスク」に特に有効。
エージェント設計の透明性も高まるため、Dynamic Workflows が生成したプロンプトを読むことで「観点別分解・厳格な報告基準・空の結果許容」という構造化エージェント設計のベストプラクティスを学べる。

**根拠**:
- 従来は人間が「移行スキル → 依存監査スキル → レビューエージェント」を手動組み合わせ; Dynamic Workflows ではリポジトリ条件に合わせて実行時に自律構成される
- 小規模の修正では従来手法との差が薄い; 大規模・複雑タスクで最も効果が出る
- Dynamic Workflows が生成するエージェントプロンプトは「問題の内容・対象ファイル・行番号・影響・重要度・修正方針」を必須フィールドとして構造化しており、空の結果を返すことも許可する設計になっている
- 観点ごとに担当を分けることで「何を見るべきか」が明確になり、エージェント間の作業重複が減る

**コード例**:
```
# Dynamic Workflows の起動
/effort ultracode

# 高レベルな目標を一度に記述する（工程の詳細は Claude が設計する）
Spring Boot 2.x → 3.x migration needed.
Review full repository, identify breaking changes,
map javax → jakarta impacts, propose PR strategy,
then execute backend changes with test fixes.
```

```
# エージェント出力の構造化フォーマット（Dynamic Workflows が自動設計するパターン）
# 実際のファイルと行番号に基づいた、説明できる欠陥だけを返す。
# 何も見つからなければ空の結果を返すこと。
{
  "findings": [
    {
      "file": "src/auth/TokenService.java",
      "line": 42,
      "issue": "javax.servlet → jakarta.servlet への移行が必要",
      "impact": "ビルドエラー",
      "severity": "high",
      "fix": "import を jakarta.servlet.* に変更"
    }
  ]
}
```

**使い分け**:
- 複雑な大規模タスク（移行・リファクタリング・バグスイープ） → `/effort ultracode` + Dynamic Workflows
- 明確な単純タスク（特定ファイルの修正など） → 従来の手動スキル組み合わせ

**出典引用**:
> "workflows now 'change while executing' rather than following pre-designed paths"
> ([Claude Code Dynamic Workflows とは一体何なのか](https://qiita.com/alpha123/items/bbe84d53a4d95e1cb044), セクション "Core Innovation") ※2026-05-30に実際にfetch成功

> "実際のファイルと行番号に基づいた、説明できる欠陥だけを返す"
> ([Claude CodeのDynamic Workflowsからエージェントの作業設計を見てみた](https://zenn.dev/yamk/articles/claude-code-dynamic-workflows-prompt), セクション "厳格な報告基準") ※2026-05-30に実際にfetch成功

**バージョン**: Claude Code（Dynamic Workflows 機能、2026年以降）
**確信度**: 中
**最終更新**: 2026-05-30

---

### 19. `dontAsk` モードとリスク分類で無人 Claude Code セッションを多層防御する

無人自律実行（ヒューマン監視なし）では `ask`（パーミッション確認）が機能しないため、
`--dangerously-skip-permissions` ではなく `dontAsk` モードに切り替えて「確認を自動拒否」に変換し、
Hook / Sandbox / Backup の3層 + 6カテゴリのリスク分類で安全境界を静的に設計する。

**根拠**:
- `dontAsk` は確認プロンプトを「拒否」に変換し、許可済みコマンドはそのまま通過させる。`skip-permissions` はすべての安全装置をバイパスするため運用NG
- 脅威を6カテゴリに分けることで「どの層で止めるか」が明確になり、単一機構への依存を排除できる
- エージェント設計の核心は「信頼」ではなく「どこまで自律して何を人間に戻すか」の境界設計
- MCP は接続を広げるだけでなく「エージェントが見える世界を意図的に狭める」ツールとして活用できる
- Hook (PreToolUse) だけに頼ると `env VAR=val cmd` 形式でラップされた際に文字列検査をすり抜けるため、サンドボックスと組み合わせる

**6カテゴリのリスク分類と対策**:

| カテゴリ | リスク | 対策層 |
|---|---|---|
| ① 不可逆な外部操作（外部API・DBへの書き込み） | 自動拒否 + サンドボックス分離 | Hook (deny) + Sandbox |
| ② 機密情報漏洩（.env・認証トークン参照） | 読み取りを PreToolUse で拒否 | Hook (deny) |
| ③ git 管理外データ破壊（DB・ログ・ファイルシステム） | バックアップ＋リカバリ機構 | Backup |
| ④ git 履歴破壊（force push 等） | deny ルールで静的ブロック | Hook (deny) |
| ⑤ ローカルの可逆変更 | 自動許可（git で復元可） | Auto-allow |
| ⑥ 任意コード実行 | サンドボックス封じ込め | Sandbox |

**コード例**:
```json
{
  "permissions": {
    "defaultMode": "dontAsk",
    "deny": [
      "Bash(curl *)",
      "Bash(git push --force *)",
      "Read(.env)",
      "Read(.env.local)"
    ]
  },
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "allowUnsandboxedCommands": false
  }
}
```

**アンチパターン**:
- `--dangerously-skip-permissions` の使用（すべての安全装置を無効化するため運用NG。代わりに `dontAsk` モードを使う）
- 単一の Hook (PreToolUse) のみに依存する（ラッパーコマンドによる迂回を防げない）

**出典引用**:
> "単一の機構で守れる前提を捨てるのがこの記事の背骨です"
> ([Claude Code 無人自律編 — ask が無力な世界で機構で守る](https://zenn.dev/kojisumiyoshi/articles/ai-agent-unattended-autonomy), セクション "設計の前提") ※2026-06-09に実際にfetch成功

> "AIエージェントに仕事を任せるとは、AIを信頼することではない。AIが間違えてもチームが回収できる形に仕事を切ることだ"
> ([AIエージェント時代、開発者の仕事は「許可する環境」を設計することになる](https://zenn.dev/heftykoo/articles/1c647688784214)) ※2026-06-08に実際にfetch成功

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 中
**最終更新**: 2026-06-09

---

### 20. `/goal`・agents ビュー・auto mode を段階的に組み合わせて並列作業の完了条件と承認を自動化する

Claude Code の自律度を安全に高めるには「agents → /goal → hard deny → auto mode → Routines」の順で機能を積み上げる。
`claude agents` で並列セッションの状態を一画面で把握し、`/goal` で**機械的に判定できる完了条件**を宣言し、auto mode で日常承認をクラシファイアに委ねる。
auto mode は「全部手動承認」と `--dangerously-skip-permissions` の中間を埋める位置づけで、危険操作はブロックしつつルーティン承認を自動化できる。

**根拠**:
- `claude agents` コマンドはすべてのアクティブセッションを1ダッシュボードに表示し、「人間の承認待ち」になっているタスクを明示する。並列数を増やすほど「どのタスクがブロックされているか」の把握コストが上がるため、統合ビューが必須
- `/goal` は完了条件を機械的に判定できる形で記述する（「`npm test` が exit 0」「指定ファイルに関数が存在する」など）。曖昧な完了条件（「きれいにリファクタして」）は auto mode 下で無限ループの原因になる
- auto mode（2026年3月 research preview）はクラシファイアがルーティン承認を処理し、危険操作のみブロックする。有効化前に hard deny ルールを確定させる順序が重要
- **導入推奨順序**: agents ビューで状況把握 → /goal で完了条件を定義 → hard deny で安全境界を確定 → auto mode を有効化 → Routines で完全自動化

**コード例**:
```bash
# agents ビューで並列セッションを確認
claude agents
# → 「承認待ち: 3件」のようにブロック中タスクが表示される

# /goal で機械的に判定できる完了条件を宣言
/goal
> npm test が exit 0 になること
> src/auth/token.ts に refreshToken 関数が存在すること
> git diff --staged でコード削除が 5 行以下であること
```

```json
// プロジェクト設定（.claude/settings.json）: 決定論的な deny ルールを先に確定する
{
  "permissions": {
    "deny": [
      "Bash(git push --force *)",
      "Bash(rm -rf *)",
      "Write(.env*)"
    ]
  }
}
```
```json
// ユーザー設定（~/.claude/settings.json）: auto mode はプロジェクト設定からは無視されるためここに置く
// クラシファイアの無条件ブロックは hard_deny に自然文で記述する（enabled フラグは存在しない）
{
  "autoMode": {
    "environment": ["信頼するインフラ・デプロイ先の説明"],
    "hard_deny": [
      "force-push to any branch",
      "delete files outside the working tree",
      "write to .env or other secret files"
    ]
  }
}
```

**アンチパターン**:
- hard deny を設定せずに auto mode を有効化する（危険な操作も自動承認されるリスク）
- `/goal` を「きれいにして」「最適化して」などの定性的な条件で書く（auto mode 下で無限ループの原因）
- `--dangerously-skip-permissions` を auto mode の代替として使う（すべての安全装置をバイパスする）

**出典引用**:
> 「完了条件を**機械的に判定できる形**で書くのがコツです」
> ([Claude Codeを「貼り付いて見守る」運用から卒業する：/goal・agentsビュー・auto mode実務導入ガイド](https://zenn.dev/yushiyamamoto/articles/claude-code-goal-agents-autonomy), セクション "2. /goal") ※2026-06-10に実際にfetch成功

> 「全部手動承認」と `--dangerously-skip-permissions` の中間を埋める位置づけです
> ([Claude Codeを「貼り付いて見守る」運用から卒業する：/goal・agentsビュー・auto mode実務導入ガイド](https://zenn.dev/yushiyamamoto/articles/claude-code-goal-agents-autonomy), セクション "3. auto mode") ※2026-06-10に実際にfetch成功

**バージョン**: Claude Code（2026年3月以降）
**確信度**: 中
**最終更新**: 2026-06-10

---

### 21. AI エージェントへのタスク依頼は仕様書として構造化し、完了条件を事前定義する

大きなタスクを曖昧なまま依頼すると、AI エージェントは「見た目で正しそう」と判断した時点で作業を終了し、残タスクを記録するだけで完了とみなす。
**依頼前に8セクションを含む仕様書を作成**し、スコープ・完了条件・停止条件を明示することで、放置・スコープ逸脱・暗黙前提による失敗を防ぐ。

**根拠**:
- AI は確認できるテストや完了条件がなければ「ドキュメントに書いた」が完了のシグナルになる
- タスクを巨大なまま渡すと、AI は見えない前提（アーキテクチャ・API 変更・状態管理方針）を独断で決める
- 仕様書に「停止条件（Stop conditions）」を書くことで、不明点の前に AI が自己判断で進むのを防げる
- CLAUDE.md に明示的に「残タスクを通知なしに記録するだけで終了することは禁止」と書かなければ、記録が完了シグナルになる

**コード例**:
```markdown
## タスク仕様書テンプレート（8セクション）

### Purpose（目的）
外部 API ポーリング実装が完了した状態 = DB の result カラムが自動更新されること

### Non-purpose（対象外）
UI の再設計・認証フローの変更は本タスクに含まない

### Background（背景）
現在ポーリングは手動トリガー。自動化することで運用コストを削減する

### Change scope（変更スコープ）
変更可能: src/polling/**, src/db/results.ts
変更禁止: src/auth/**, src/ui/**

### Input context（参照情報）
- src/polling/README.md を最初に読むこと
- API 仕様書: docs/external-api.md

### Acceptance criteria（完了条件）
- `npm test src/polling` が exit 0 になること
- DB の result カラムが受信後 5 秒以内に更新されること

### Test conditions（テスト条件）
- `pnpm run test:e2e polling` で疎通を確認すること

### Stop conditions（停止条件）
- API 仕様書に記載のない動作を発見した場合は実装前に報告して確認を取る
- テストが 2 回連続で失敗したら解決策を報告して停止する
```

```markdown
## CLAUDE.md への必須追記（放置防止）

## 未完了フロー処理（必須）
スコープ外の未完了フローが発生した場合は、ユーザーに報告し継続の確認を取ること。
通知なしに残タスクを記録するだけで作業を終了することは禁止。
```

**アンチパターン**:
- 「検索・フィルター・エクスポート・CSV出力」を一度に依頼する（順番に分割して依頼する）
- 完了条件を「きれいにして」「最適化して」などの定性表現にする
- CLAUDE.md に「残タスクを記録すること」だけ書いて完了とみなす設定にしている

**出典引用**:
> "The key issue isn't the prompt itself—it's how you hand over the specification."
> ([AIエージェントに渡す仕様書の書き方──巨大な依頼を壊さず小さく渡す](https://zenn.dev/aiwatch_jp/articles/agent-flow-spec-writing), セクション "仕様書の渡し方") ※2026-06-12に実際にfetch成功

> "Claude stops when the work **looks** done. Without a check it can run, 'looks done' is the only signal available."
> ([Claude Codeが残課題を放置する理由と対策](https://zenn.dev/acntechjp/articles/ce83f62acf41c0), セクション "Layer 1") ※2026-06-12に実際にfetch成功

**出典**:
- [AIエージェントに渡す仕様書の書き方──巨大な依頼を壊さず小さく渡す](https://zenn.dev/aiwatch_jp/articles/agent-flow-spec-writing) (Zenn、8セクション仕様書フレームワーク) ※2026-06-12に実際にfetch成功
- [Claude Codeが残課題を放置する理由と対策](https://zenn.dev/acntechjp/articles/ce83f62acf41c0) (Zenn、3層構造の診断と CLAUDE.md 強制ルールによる解決) ※2026-06-12に実際にfetch成功

**バージョン**: Claude Code（全バージョン）
**確信度**: 高
**最終更新**: 2026-06-12

---

### 22. ネスト型サブエージェントは `.claude/agents/` フロントマターで定義し、階層ごとに Haiku / Sonnet / Opus を使い分ける

Claude Code v2.1.172 以降、サブエージェントは最大5階層まで入れ子にできる。各階層は独立したコンテキストを持つため、深い階層ほどトークン消費が増大する。「葉は Haiku（探索・読み取り）、中間推論は Sonnet、最終合成のみ Opus」という深さ別モデル設計で、品質を保ちながらコストを抑える。

**根拠**:
- `.claude/agents/` 配下の YAML フロントマターで `model:` フィールドを指定すると、サブエージェントごとにモデルを切り替えられる（Claude Code 公式機能）
- 深い階層のエージェントは上位からの指示が既に絞り込まれた状態で動くため Opus の推論力は不要
- `CLAUDE_CODE_SUBAGENT_MODEL` 環境変数でデフォルトモデルを一括設定できる

**コード例**:
```yaml
---
name: log-summariser
description: コンテナのログファイルを構造化要約する。ログ調査が必要なときに呼ぶ
tools: Read, Grep
model: haiku
---
あなたはログ要約の専門家です。
与えられたログファイルを読み、エラー・警告・タイムスタンプを構造化して要約してください。
ファイルの書き込みは行いません。
```

```bash
# デフォルトモデルの一括設定（深い階層の探索エージェントを Haiku に固定）
export CLAUDE_CODE_SUBAGENT_MODEL=haiku
```

**3層ネスト構成例**:
| 深さ | 役割 | モデル |
|---|---|---|
| 0（メイン） | 統括・最終合成 | Opus |
| 1 | 中間推論・再現実行 | Sonnet |
| 2 | ログ読み取り・ファイル探索 | Haiku |

**アンチパターン**:
- 全階層を Opus で統一する（コストが深さに比例して急増）
- `description` を曖昧に書く（ルーティング精度が下がり意図しないタイミングで呼ばれる）
- 逐次処理タスクをネストする（単一エージェントの方が速い）
- 横方向の並走タスクをネストする（Agent Teams の方が適切）

**出典引用**:
> 「葉はHaiku、推論が必要な中間はSonnet、最終合成だけOpus——というように下に行くほど軽くするのがセオリー」
> ([Claude Codeのネスト型サブエージェント入門 — 最大5階層の設計とトークン設計の勘所](https://qiita.com/kai_kou/items/618da2497af1c1bf0f91), セクション "トークン設計パターン") ※2026-06-13に実際にfetch成功

**バージョン**: Claude Code v2.1.172+
**確信度**: 中
**最終更新**: 2026-06-13

---

### 23. マルチエージェント組織は Playbook 孤立と CLAUDE.md 増殖を定期 grep 検査で能動的に検出する

マルチエージェント組織は「大きな事件」なしに静かに腐敗する。代表的な2パターン:
1. **Playbook 孤立**: 追加した Playbook ファイルが CLAUDE.md やルールファイルから参照されなくなる
2. **CLAUDE.md 増殖**: 想定外のパスに重複 CLAUDE.md が生まれ、どのルールが有効か不明になる

構造変更（Playbook 追加・ルール改訂）のタイミングで、参照関係と配置の健全性チェックを並行して実施する。

**根拠**:
- 腐敗の発見が「偶然」に依存するうちは、組織が大きくなるほど問題が潜伏し続ける
- grep による参照スキャンは低コストかつ確実な検出手段であり、CI に組み込みやすい
- **サブエージェントの主な価値は「速度向上」ではなく「コンテキスト分離」**: 分析専用エージェントに `Read/Grep/Glob` のみを与えてメインコンテキストを汚染しない設計が長続きの鍵。マルチエージェントパターンはシングルエージェントの約 15 倍のトークンを消費する（Anthropic データ）ため、並列化は「独立した同時分析が必要なとき」に限定する
- **map-implementation 整合はフックで機械強制する**: PostToolUse フックで「ロールマップが参照するエージェントファイルが存在するか（dead link）」「エージェントファイルがマップに登録されているか（orphan）」を自動検出することで、組織が大きくなっても整合を保てる

**コード例**:
```bash
# Playbook の参照状況を確認（参照ゼロ = 孤立ファイル）
for f in .claude/rules/*.md; do
  count=$(grep -r "$(basename "$f" .md)" .claude/ --include="*.md" | grep -v "^$f:" | wc -l)
  echo "$count refs: $f"
done

# リポジトリ内の全 CLAUDE.md の配置を確認（想定外パスの増殖を検知）
find . -name "CLAUDE.md" -not -path "*/node_modules/*"
```

**構造変更時のチェックリスト**:
- [ ] 新規 Playbook 追加: どのルールファイルから参照するかを同時に決めたか
- [ ] Playbook 削除・改名: 参照元 CLAUDE.md・ルールファイルの言及を除いたか
- [ ] 2 週間ごと: 参照ゼロ Playbook と想定外パスの CLAUDE.md を grep で確認したか

**アンチパターン**:
- Playbook を「後でリンクを貼ればいい」と仮置きしてそのままにする
- CLAUDE.md の配置場所を grep せず記憶で管理する
- 構造変更後に健全性チェックをまとめて後回しにする

**出典引用**:
> 「腐敗に事件は要らない。組織が存在するだけで、日常的に腐敗は始まる。」
> ([Claude Codeのマルチエージェント組織は、何もしなくても腐っていく——CLAUDE.md増殖とPlaybook孤立、2つの腐敗パターン](https://qiita.com/saitoko/items/23f6709fc0de71332f91), セクション "腐敗のメカニズム") ※2026-06-13に実際にfetch成功

> "The first value of subagents is not 'becoming faster in parallel' but context isolation."
> ([Claude Code でAIチームを設計する ─ 増やしても破綻させない5層モデルと3原則](https://zenn.dev/takna/articles/claude-code-ai-team-design), セクション "Subagents: Context Isolation as Primary Value") ※2026-06-23に実際にfetch成功

**出典**:
- [Claude Codeのマルチエージェント組織は、何もしなくても腐っていく——CLAUDE.md増殖とPlaybook孤立、2つの腐敗パターン](https://qiita.com/saitoko/items/23f6709fc0de71332f91) (Qiita) ※2026-06-13 fetch
- [Claude Code でAIチームを設計する ─ 増やしても破綻させない5層モデルと3原則](https://zenn.dev/takna/articles/claude-code-ai-team-design) (Zenn takna、5層モデル・コンテキスト分離の価値・15倍トークン消費・map-implementation 整合フック) ※2026-06-23 fetch

**バージョン**: Claude Code（マルチエージェント対応以降）
**確信度**: 中
**最終更新**: 2026-06-23

---

### 24. AI エージェントの PR を「意思決定ログ」として Issue 仕様・検証ログ・不採用案の 3 点セットでレビューする

AI エージェントの PR は「何が変わったか」は見えるが「なぜそう判断したか」が不透明になりやすい。Issue を「実行仕様」として精密に書き、PR 説明に検証結果と不採用案を含めることで、人間レビュアーが判断できる形に整える。

**根拠**:
- AI の出力を「信じるか信じないか」ではなく「断れる形にしておく」ことが安全な運用の核心
- Issue に禁止事項・検証コマンドを明記することで、エージェントの暴走範囲を事前に絞れる
- 不採用案の記録により、同じ実装を再度試みることをレビュアーが止めやすくなる

**Issue 仕様テンプレート**:
```markdown
## 目的
signup form の email フィールドのバリデーションを強化する

## 対象範囲
- 変更OK: `components/signup/EmailField.tsx`、既存の `validation/email.ts` ヘルパーを使う
- 変更禁止: UI 文言の変更、他フォームへの波及

## 検証コマンド（PR に実行結果を貼ること）
pnpm test signup
pnpm lint
```

**PR 説明に含めるべき項目**:
| 項目 | 内容 |
|---|---|
| 作業目的 | Issue の目的と合致しているか |
| 変更範囲 | 指定外ファイルへの波及がないか |
| 検証結果 | 指定コマンドの出力ログ |
| 不採用案 | 検討して却下した代替アプローチ |
| 確認点 | レビュアーに意見を求めたい箇所 |

**差し戻し基準**:
- 検証ログが PR に貼られていない
- Issue で禁止した変更（対象外ファイルの変更・UI 文言変更）が含まれる
- 変更理由をエージェントが Explain できない箇所がある

**アンチパターン**:
- Issue を「きれいにして」「最適化して」などの定性的な記述にする（エージェントが際限なく変更する）
- 検証コマンドを指定しない（PR に実行証拠が残らない）
- 差し戻し基準を事前に決めずにレビューする（後出しジャンケンになる）

**出典引用**:
> 「AIを信じるのではなく、AIの出力を断れる形にしておく」
> ([AIエージェントのPRは「差分」ではなく「意思決定ログ」としてレビューする](https://zenn.dev/heftykoo/articles/2ef4dac14f86bc), セクション "セキュリティ対応としてのAIレビュー") ※2026-06-13に実際にfetch成功

**バージョン**: Claude Code（エージェント機能対応以降）
**確信度**: 中
**最終更新**: 2026-06-13

---

### 25. AI エージェント実行は「外部送信・公開クラス」を承認ゲートで構造的に分離する

エージェントの実行結果を `autoExecuted` / `requiresApproval` / `skipped` の3分類に構造化し、
「外部への送信・公開・デプロイ」をコードレベルで承認なしに実行できない設計にする。
プロンプト指示や設定フラグではなく、**権限そのものを持たない設計**が承認ゲートの核心。

**根拠**:
- 「プロンプトが変わると承認ロジックも変わってしまう。承認はコントローラ層（外部）に書く」— プロンプトの禁止は破られる
- 「コードレベルで『この操作は承認なしに実行できない』という制約を持たせるのが、承認ゲートの役割」
- GitHub Actions × Claude の半自動 PR レビューも同パターン: 下書き生成（書き込みなし）→ ラベル承認 → 公開。「対象リポジトリに書き込まない」段階を設けることで誤指摘の流出を防ぐ
- ソフトウェア設計原則: エージェントに権限を与えないことが最も確実な制御手段（最小権限原則の適用）

**コード例（承認ゲートの型設計と実行フロー）**:
```typescript
type AgentOutput = {
  autoExecuted: ExecutedAction[];
  requiresApproval: ApprovalItem[];
  skipped: SkippedAction[];
};

type ApprovalItem = {
  id: string;
  action: "send_email" | "post_sns" | "deploy" | "publish";
  summary: string;
  draftPath?: string;
  approved?: boolean;
};

// 事前許可リスト（空 = 全件要承認）で承認不要な操作のみ自動実行
async function runWithApprovalGate(
  agent: Agent,
  task: TaskSpec,
  autoApprove: string[] = [],
): Promise<AgentOutput> {
  const output = await agent.run(task);
  const approved: ApprovalItem[] = [];
  const blocked: ApprovalItem[] = [];

  for (const item of output.requiresApproval) {
    if (autoApprove.includes(item.id) || autoApprove.includes(item.action)) {
      approved.push({ ...item, approved: true });
    } else {
      blocked.push({ ...item, approved: false }); // 人間に返す
    }
  }
  for (const item of approved) {
    await executeApprovedAction(item);
  }
  return { ...output, requiresApproval: blocked };
}
```

**CI/CD 実装パターン（GitHub Actions × Claude 半自動 PR レビュー）**:
```yaml
# Step 1: 下書き生成（対象リポジトリへの書き込みなし）
# auto-review.yml — 30分ごとに未レビュー PR を取得し private Issue に下書きを保存
name: 自動レビュー(下書き生成)
on:
  schedule:
    - cron: '*/30 * * * *'
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - run: bash scripts/auto-review.sh
```

```yaml
# Step 2: レビュアーがラベルを付けると自動公開（approve / request-changes / comment）
# publish-review.yml — ラベルトリガーで PR にインラインコメントとして反映
on:
  issues:
    types: [labeled]
jobs:
  publish:
    if: |
      github.event.label.name == 'approve' ||
      github.event.label.name == 'request-changes' ||
      github.event.label.name == 'comment'
    steps:
      - run: bash scripts/publish-review.sh
```

**操作分類の基準**:
| 分類 | 具体例 | 承認要否 |
|---|---|---|
| 外部送信・公開 | メール送信・SNS 投稿・PR コメント・デプロイ | **要承認** |
| 不可逆削除 | ファイル削除・DB レコード削除 | **要承認** |
| 外部参照（読み取り） | API 取得・コード読み込み | 自動実行 OK |
| 内部生成 | 下書き作成・ローカルファイル書き込み | 自動実行 OK |

**アンチパターン**:
- プロンプトに「外部送信の前に必ず確認する」と書く → プロンプト変更で迂回される
- フラグ変数（`let approved = false`）で制御 → エージェントが同一コンテキスト内で上書きできる
- 「エージェントに権限を与えて、使わないよう指示する」設計

**出典引用**:
> 「コードレベルで「この操作は承認なしに実行できない」という制約を持たせるのが、承認ゲートの役割」
> ([AIエージェントの承認ゲート設計——外部送信・公開クラスを権限設計で分離する](https://zenn.dev/yushiyamamoto/articles/claude-agent-approval-gate-design-2026-06), セクション "承認ゲートが必要な理由") ※2026-06-21に実際にfetch成功

> 「レビュアーは **ラベルを1つ付けるだけ** で、該当行にインラインコメントとして公開される」
> ([GitHub Actions × Claude で下書きは AI・承認は人間な半自動 PR レビューを作った](https://zenn.dev/prof_nyanko1124/articles/917c89c91a1248), セクション "完成イメージ") ※2026-06-21に実際にfetch成功

**出典**:
- [AIエージェントの承認ゲート設計——外部送信・公開クラスを権限設計で分離する](https://zenn.dev/yushiyamamoto/articles/claude-agent-approval-gate-design-2026-06) (Zenn) ※2026-06-21 fetch
- [GitHub Actions × Claude で下書きは AI・承認は人間な半自動 PR レビューを作った](https://zenn.dev/prof_nyanko1124/articles/917c89c91a1248) (Zenn) ※2026-06-21 fetch

**バージョン**: Claude Code・GitHub Actions（全バージョン共通）
**確信度**: 中
**最終更新**: 2026-06-21

---

### 26. Claude Code hookのレイテンシを `$EPOCHREALTIME` + p95 で計測し、同期フックの遅延を特定する

hookのラッパースクリプトで実行時間を JSONL ログに記録し、p95（100回中95回がこの時間以内）で日常的な遅延を把握する。
同期フックの p95 が高い場合のみボトルネックとして対処し、`async: true` フックの所要時間は問わない。

**根拠**:
- 同期フックは完了するまで Claude Code をブロックする。`async: true` フックはバックグラウンド実行のためユーザー体験に影響しない
- 平均値（mean）はゾンビプロセスや sleep 状態の外れ値で歪む。p95 が「日常的に体験する遅延」を正直に示す
- どのフックが遅いか特定できなければ最適化対象を絞れない。ラッパーで全フックを一律計測するのが最も低コスト

**コード例（ラッパースクリプト）**:
```bash
#!/bin/bash
# ~/.claude/hooks/hook-latency-wrap.sh
# 使い方: settings.json で "command": "~/.claude/hooks/hook-latency-wrap.sh my-hook.sh"

HOOK_NAME="${1:-unknown}"
HOOK_SCRIPT="$HOME/.claude/hooks/$HOOK_NAME"
LOG_FILE="$HOME/.claude/hook-latency.jsonl"

# $EPOCHREALTIME はマイクロ秒精度の POSIX epoch（bash 5+）
START=$EPOCHREALTIME

"$HOOK_SCRIPT" "${@:2}"
exit_code=$?

END=$EPOCHREALTIME
elapsed_ms=$(awk "BEGIN { printf \"%.0f\", ($END - $START) * 1000 }")

printf '{"hook":"%s","elapsed_ms":%s,"exit_code":%s,"ts":"%s"}\n' \
  "$HOOK_NAME" "$elapsed_ms" "$exit_code" "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  >> "$LOG_FILE"

# exit code を保持することが必須（省略するとエラー検出が壊れる）
exit "$exit_code"
```

```bash
# p95 集計（フック名ごと）
jq -rs '
  group_by(.hook)[] |
  sort_by(.elapsed_ms) |
  {
    hook: .[0].hook,
    count: length,
    p50_ms: .[floor(length*0.50)].elapsed_ms,
    p95_ms: .[floor(length*0.95)].elapsed_ms
  }
' ~/.claude/hook-latency.jsonl
```

**判断軸**:
| フック種別 | 計測対象か | 改善優先度 |
|---|---|---|
| 同期（async: true なし） | ✅ p95 > 500ms を閾値 | 高 — ユーザー操作をブロック |
| 非同期（async: true） | 不要 | なし — バックグラウンド実行 |
| Stop フック | ✅ 参考値 | セッション終了時のみ影響 |

**アンチパターン**:
- ラッパーで `exit "$exit_code"` を省略する（フックのエラー戻り値を Claude Code が受け取れなくなる）
- 平均値だけを見て「速い」と判断する（外れ値が平均を引き下げ、日常的な遅延を隠す）

**出典引用**:
> "hookが同期実行の場合、完了するまでClaude Codeはブロックします"
> ([Claude Codeのhookが遅い原因を特定する ― wrap 1行でp95を計測](https://zenn.dev/bokuwalily/articles/hook-latency-profiling), セクション "問題の本質") ※2026-06-22に実際にfetch成功

> "p95は『100回に95回がこの時間以内』なので、日常の遅さを正直に示します"
> ([Claude Codeのhookが遅い原因を特定する ― wrap 1行でp95を計測](https://zenn.dev/bokuwalily/articles/hook-latency-profiling), セクション "なぜ p95 か") ※2026-06-22に実際にfetch成功

**バージョン**: Claude Code（全バージョン）、bash 5+
**確信度**: 中
**最終更新**: 2026-06-22

---
