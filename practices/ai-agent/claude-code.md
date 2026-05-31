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

**出典引用**:
> "Production systems often combine patterns. A common hybrid uses orchestrator-subagent for overall workflow with shared state for collaboration-heavy subtasks."
> ([Anthropicの5パターンでClaude Codeエージェント設計を分類する](https://zenn.dev/motowo/articles/anthropic-multi-agent-coordination-patterns-guide), セクション "Hybrid Patterns (Production Reality)") ※2026-05-08に実際にfetch成功

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 中
**最終更新**: 2026-05-08

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

### 5. Claude Code の Skills 機能でチーム作業ワークフローと個人の学習ループを自動化する

`.claude/skills/` ディレクトリに Markdown ファイルを置くとファイル名がスラッシュコマンドとして登録され、反復する作業手順を「実行可能なワークフロー」として切り出せる。
スキルは「ブランチ作成→実装→コミット→PR 作成」のようなチーム作業だけでなく、「ふりかえり KPT」「日報生成」のような**個人の学習ループ**にも有効。フィードバックを Skill 自体に蓄積していくことで、繰り返すたびに精度が上がる。
Claude がスキルを自動起動しない under-trigger 問題を防ぐため、`description` フィールドは「このスキルを使うべき条件」を明示的かつ強めの表現で記述する。

**根拠**:
- `.claude/skills/` に置いた Markdown ファイルのファイル名がそのままスラッシュコマンドになる
- under-trigger（使うべき場面で Claude がスキルを起動しない）は `description` の書き方で防げる
- `devcontainer.json` の `mounts` で Claude Config ディレクトリをボリュームに永続化すると rebuild 後も再認証が不要
- フィードバックを Skill ファイル自体に追記していくと、Claude が過去の改善履歴を参照して深掘りの精度を上げる学習ループが回る（チームの暗黙知の形式知化にも応用可能）

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

**出典引用**:
> "under-trigger（trigger すべき場面で使わない）の傾向があるため、明示しなくてもスキルが発動するよう description を強めに書く"
> ([Claude Code の skill 機能を本格的に試す — PR まで完結させた話](https://qiita.com/atsushi11o7/items/5cbef4b10f3ec55c75a1), セクション "重要な設計指針") ※2026-05-09に実際にfetch成功

> "`.claude/skills/` フォルダに Markdown ファイルを置くと、そのファイル名がスラッシュコマンドとして使えるようになります。"
> ([Claude Code Skills で /briefing コマンドを5分で実装する](https://qiita.com/hinaworks/items/93ae032d392bdad25655), セクション "実装ステップ") ※2026-05-09に実際にfetch成功

> 「そのフィードバックを貯めておくことで、`furikaeri-kpt` Skill がそのフィードバックを参考に深掘りを改善してくれます。」
> ([使うほどふりかえりの深掘りが上達していく Claude Code Skill を作った話](https://zenn.dev/sonicgarden/articles/03c81ca6413ad4), Zenn sonicgarden) ※2026-05-16に実際にfetch成功

**バージョン**: Claude Code（Skills 機能、2026年4月以降 gh CLI v2.90+）
**確信度**: 高
**最終更新**: 2026-05-16

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

**出典引用**:
> "グローバル側の `CLAUDE.md` は目次に寄せました。定型作業をskillに寄せると、レビュー観点のぶれも減ります。"
> ([Claude Code運用を数ヶ月で見直してrulesとskillsに分けた話](https://zenn.dev/harness/articles/claude-code-workflow-evolution), セクション "skill化したのは定型作業を短くするため") ※2026-05-11に実際にfetch成功

> "コンテキストは削減されません...`paths` で対象ファイル編集時のみ読み込まれるようにすると実質トークン消費を抑えられる"
> ([CLAUDE.mdを整理したら、めちゃくちゃトークン消費が増えていた話](https://zenn.dev/penguingymlinux/articles/5335e74359b3d9), セクション "@import を使ってもコンテキストは削減されない") ※2026-05-11に実際にfetch成功

> "情報を「入れる」だけでは不十分で、「何を入れないか」「いつ入れるか」を設計する必要がある"
> ([コンテキストエンジニアリング実践ガイド — Claude Codeで学ぶ4つの戦略](https://zenn.dev/miyan/articles/claude-code-context-engineering-guide-2026), セクション "4つの戦略") ※2026-05-21に実際にfetch成功

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 高
**最終更新**: 2026-05-21

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

### 8. 複数AIツール（Claude Code・Codex等）を AGENTS.md で一元管理し、ツール固有機構を薄くアダプトする

同一プロジェクトで複数のAIコーディングエージェントを併用する場合、
「ツール非依存の共通ルール」を `AGENTS.md` に集約し、各ツール固有の設定ファイルから
`@AGENTS.md` でインポートする「共通正本 + 薄いアダプタ」構成を採用する。

**根拠**:
- Claude Code（`CLAUDE.md`）と Codex（`AGENTS.md`）など各ツールの設定形式は互いに非互換
- 設定ファイルを直接共有できないため、各ツールが参照する**スクリプトや指示ファイルの実体を一元化**する
- Skill（`~/.agents/skills/`）と hook スクリプト（`shared/hooks/`）を共通正本に置き、
  各ツールの設定からシンボリックリンクで参照することで1箇所の変更が全ツールに反映される

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

**バージョン**: Claude Code（全バージョン）、複数AIエージェント共存環境
**確信度**: 中
**最終更新**: 2026-05-13

---

### 9. Claude Code の hooks で危険なコマンドをブロックし、操作規律をコードに落とす

JSON stdin → 処理 → exit コードというパイプラインを理解し、誤って破壊的操作を実行しないよう安全装置をスクリプトで実装する。

**根拠**:
- hooks はツール実行前後に介入できる仕組みで、`exit 2` で実行をブロック、`exit 0` で許可する
- `rm -rf`・本番 DB の直接書き換え・機密ファイルへのアクセスなど「やってはいけない操作」を機械的に防げる
- スクリプトは再利用可能で、チーム全体の操作規律を `.claude/settings.json` で共有できる
- **CLAUDE.md への指示は「お願い」、Hooks は「強制」** — セッションをまたいでも必ず適用される規律をコードに落とす
- `.env` ファイルへのアクセスを PreToolUse でブロックすることで、シークレットの意図しない読み取りを防げる

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

**出典**:
- [Claude Codeのhookの仕組み:JSONとexitコードで作る最小の安全装置](https://zenn.dev/yurukusa/articles/be79dbe97e34bb) (Zenn) ※2026-05-14に実際にfetch成功
- [Claude Code hook で AI coding assistant の規律を補強する — 個人運用での設計パターン参考](https://zenn.dev/shogaku/articles/claude-code-hook-discipline) (Zenn、&&/;/|| チェーンブロック・1MB超ファイルブロックのパターン追加) ※2026-05-22に実際にfetch成功
- [Claude Code Hooksで作る7つの安全装置 ─ PreToolUse/PostToolUse 実装集](https://zenn.dev/kenimo49/articles/claude-code-hooks-7-safety-patterns) (Zenn、.env ブロック・Stop フック・CLAUDE.md request vs Hooks 強制の整理) ※2026-05-29に実際にfetch成功

> "Memory file の「注意書き」だけだと同じハマりを繰り返すと感じることがあります。強制可能な規律は強制する、memory rule で誤魔化さない"
> ([Claude Code hook で AI coding assistant の規律を補強する](https://zenn.dev/shogaku/articles/claude-code-hook-discipline), セクション "Incremental Expansion") ※2026-05-22に実際にfetch成功

> "CLAUDE.mdへの指示は『お願い』、Hooksは『強制』です"
> ([Claude Code Hooksで作る7つの安全装置](https://zenn.dev/kenimo49/articles/claude-code-hooks-7-safety-patterns), セクション "はじめに") ※2026-05-29に実際にfetch成功

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 中
**最終更新**: 2026-05-29

---

### 10. Git Worktree で Claude Code タスクを並列実行し、編集境界を厳格に固定する

`git worktree add` で複数のワーキングツリーを作成し、独立した Claude Code セッションを並走させる。
ただし worktree はブランチを分けても**共通ファイルの競合を自動解決しない**ため、各エージェントの編集対象を明示的に固定し、共通スタイルファイル等はシリアル処理に倒す前提で組む。

**根拠**:
- 各 worktree は独立したブランチ・ファイルシステムを持つため、Claude Code セッション間で「同じワーキングディレクトリのファイル」を奪い合うことは起きない
- 1つのリポジトリを複数クローンするより効率的（`.git` を共有するためディスク使用量が少ない）
- レビュー中 PR へのコメント対応・バグ修正・機能開発を同時並行できる
- 反面、`_common.scss` / `style.scss` / `main.scss` のような共通ファイルは別ブランチでも内容衝突する。エージェントごとに「このファイルのみ編集」と明示しないとコンフリクトが集中する

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

**出典引用**:
> "4-6 parallel issues are the sweet spot to maintain readability"
> ([Break It Small, Ship It Right – Skills for Coding Agents](https://developers.cyberagent.co.jp/blog/archives/63674/), セクション "Enable Parallel Development on Independent Issues") ※2026-05-14に実際にfetch成功

> "git worktreeで並走する開発フローを作ることで、PRレビュー待ち中も別タスクをClaude Codeに任せられる"
> ([Git Worktree CLI「vibe」で Claude Code と並走する開発フローを作る](https://zenn.dev/kexi/articles/git-worktree-vibe-claude-code), セクション "なぜ worktree か") ※2026-05-14に実際にfetch成功

> 「ブランチは切った時点の状態を元に作業するので、別ブランチにしてワークツリーを立てたとて同じファイルを別々に編集すれば当然衝突する。」「『他ファイル編集禁止』くらいまで指定した方が安全そうである。」
> ([【初心者】ワークツリーを切ってイキりまくり、エージェント並行開発をしていたらコンフリクトを連発した話と対策。](https://zenn.dev/wahe/articles/eda7fb1a7a2848), Zenn wahe) ※2026-05-16に実際にfetch成功

**バージョン**: Claude Code + Git（全バージョン共通）
**確信度**: 中
**最終更新**: 2026-05-16

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

**拡張機能の使い分け**:
| 拡張 | 用途 | 例 |
|---|---|---|
| **Skills** | 繰り返し指示・思考テンプレート | `plan-task`, `post-task-verification`, `/daily-schedule` |
| **MCP** | 外部サービスへのアクセス | Notion, GitHub, Slack 連携 |
| **Hooks** | イベント駆動の自動化 | コミット前 lint・テスト自動実行 |
| **Plugins** | Skills/MCP/Hooks のパッケージ化 | チーム別ロールベース配布 |

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

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 高
**最終更新**: 2026-05-24

---

### 12. Claude Code hooks を3層（セキュリティ / 状態管理 / 学習）で設計し、`async` と `exit 2` の非互換を把握する

hooks はイベント種別ごとに「何を目的にするか」を層で分けて設計する。Layer 0（セキュリティゲート）では `exit 2` でブロック、Layer 1（状態管理）では情報収集とコンテキスト注入、Layer 2（学習）では非同期の観測と失敗パターン分析を担う。
**`async: true` と `exit 2` は同時に使えない**——バックグラウンドプロセスは Claude の実行をブロックできないため、ブロック用フックは必ず同期実行にする。

**根拠**:
- `PreToolUse` + `exit 2` でシークレット検出・予算超過・破壊操作を機械的にブロックできる
- `UserPromptSubmit` でキーワードマッチしてドキュメントやコンテキストを自動注入し、毎回の手動指示を省ける
- `Stop` フックで会話終了時に git の未 push 状態を表示すると、コミット漏れを防げる
- `PreCompact` フックでコンテキスト圧縮前にセッションログを `_inbox/` へ退避し、情報損失を防げる
- `async: true` にするとバックグラウンドで実行されるが `exit 2` の効果がなくなる—ロギング目的のみに使う

**フック層の分担**:
| 層 | イベント | 目的 | `exit 2` |
|---|---|---|---|
| Layer 0（セキュリティ） | `PreToolUse` | 危険操作ブロック・シークレット検出 | 使用可 |
| Layer 1（状態管理） | `UserPromptSubmit`, `PostToolUse` | コンテキスト注入・バリデーション | 使用可 |
| Layer 2（学習） | `Stop`, `PreCompact` | ログ退避・git 状態表示 | **使用不可**（async 推奨）|

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

**出典引用**:
> "指示する」から「設計する」に変わった瞬間、世界が変わった。指示は消耗する。設計は蓄積する。"
> ([Claude Code hooksを47本実装した話：AIへの自動指示を設計するという仕事](https://zenn.dev/thinkyou0714/articles/claude-code-hooks-47), セクション "設計への転換") ※2026-05-17に実際にfetch成功

> "生ログを手元に残すほうが情報損失が少ない"
> ([Claude Code hookをauto-recapより先に試すべき3パターン](https://zenn.dev/joemike/articles/claude-code-hooks-auto-recap-alternative-2026), セクション "PreCompact フック") ※2026-05-17に実際にfetch成功

> "gitleaks git --pre-commit --staged --redact --verbose ."
> ([Claude Code と Codex の両方に、機密情報と個人情報を漏らさせない hook を作った話](https://zenn.dev/todayama_r/articles/multi-agent-secret-pii-guard-hooks), Zenn, セクション "Layer 1 共有フック") ※2026-05-31に実際にfetch成功

**出典**:
- [Claude Code hooksを47本実装した話](https://zenn.dev/thinkyou0714/articles/claude-code-hooks-47) (Zenn) ※2026-05-17 fetch
- [Claude Code hookをauto-recapより先に試すべき3パターン](https://zenn.dev/joemike/articles/claude-code-hooks-auto-recap-alternative-2026) (Zenn) ※2026-05-17 fetch
- [Claude Code と Codex の両方に、機密情報と個人情報を漏らさせない hook を作った話](https://zenn.dev/todayama_r/articles/multi-agent-secret-pii-guard-hooks) (Zenn、gitleaks + PII pattern + multi-agent 対応) ※2026-05-31 fetch

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 高
**最終更新**: 2026-05-31

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

**コード例（Managed Settings 最小安全設定）**:
```json
// managed-settings.json（組織統制ファイル）
{
  "disableBypassPermissionsMode": "disable",
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(~/.ssh/**)",
      "Read(~/.aws/**)"
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
- [ ] 信頼できないソース（外部URL・ユーザー入力）をAIに渡す前に、ファイル変更・git push・API呼び出しを一時無効化する
- [ ] 拡張機能は半年ごとに棚卸しして未使用を削除する
- [ ] `curl`/`wget` を Bash deny に追加する（ワイルドカードパターン回避より確実）
- [ ] `.claudeignore` で「業務上 Claude に読ませる必要がないもの」を Deny ルールと補完する形で除外する

**出典引用**:
> "すべての層を強制力のあるレベルで設定する" — `disableBypassPermissionsMode: 'disable'` が Managed Settings のコア設定
> ([Claude Codeを社内に安全に導入する：システムエンジニアのためのセキュリティ実践ガイド](https://zenn.dev/nocodesolutions/articles/f71479fa5711fe), セクション "4層モデル") ※2026-05-18に実際にfetch成功

> "信頼できないソースをAIに読ませるときは、その会話でファイル変更・git push・API呼び出しができない状態にしてから渡します"
> ([AIエージェントは最小権限で使う｜Claude Code・MCP・VS Code拡張の安全な設定](https://zenn.dev/takibilab/articles/ai-agent-least-privilege), セクション "信頼できないソースの扱い") ※2026-05-18に実際にfetch成功

> "『漏れたら困るもの』ではなく『業務上 Claude に読ませる必要がないもの』という基準で書く"
> ([.claudeignore の正しい書き方：Deny ルールとの使い分けと「読ませない」設計思想](https://zenn.dev/siromiya/articles/claudeignore-design-guide), セクション ".claudeignore の設計思想") ※2026-05-20に実際にfetch成功

**バージョン**: Claude Code（全バージョン）、Enterprise Managed Settings 対応環境
**確信度**: 中
**最終更新**: 2026-05-20

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

**アンチパターン**:
- コンテキスト使用率が 85% を超えてから新規セッションを開始する（設計記憶が消える）
- 設計判断を「Claude が覚えている」として外部化しない

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

**バージョン**: Claude Code（全バージョン）
**確信度**: 中
**最終更新**: 2026-05-25

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

**量の目安**: 50〜150 行が適切。200 行を超えると Claude の注意力が分散しルール遵守率が低下する。2 週間に一度見直して陳腐化したエントリを削除する。

**自動メモリの4種（`~/.claude/projects/<project>/memory/`）**:
| 種別 | 内容 |
|---|---|
| **user** | 自分のロール・専門性・作業スタイルの好み |
| **feedback** | Claude への訂正・確認済みパターン |
| **project** | 現在のゴール・タイムライン |
| **reference** | 外部システムの場所・アクセス方法 |

自動メモリはチーム共有の CLAUDE.md と異なり個人が蓄積する文脈。コード場所・関数名はリファクタリングで腐りやすいので定期的に棚卸しする。

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

**アンチパターン**:
- 「Claude が覚えているはず」でセッションを閉じる → 次セッションでゼロから説明が必要
- 自動メモリにコード場所・関数名を記録する → リファクタリングで即腐る
- CLAUDE.md に全ルールを直接書く → トークン過消費（Rule #6 の `paths` フロントマターで目次化）

**出典引用**:
> "CLAUDE.md はリポジトリにコミットして新規メンバーの立ち上がりコスト削減にも使える。PR レビューの観点に『CLAUDE.md の更新が必要か』を入れるとルールの陳腐化を防げる"
> ([Claude Code を使い倒すための設定術：CLAUDE.md・自動メモリ・コンテキスト管理の3本柱](https://zenn.dev/tamai_hideyuki/articles/claude-code-config-best-practices), セクション "CLAUDE.md: プロジェクト文脈の確立") ※2026-05-24に実際にfetch成功

> "CLAUDE.md だけでは長期開発には足りない。backlog.md でタスクを引き継ぎ、MEMORY.md で『なぜそう決めたか』を残してからセッションを閉じないと、次セッションのエージェントにとって存在しないのと同じだった"
> ([#1 セッションを閉じたら全部消えた——Claude Codeと長期開発するための設計論](https://zenn.dev/yuichi1996/articles/2006df87f3209c), セクション "三ファイルシステム") ※2026-05-24に実際にfetch成功

> "「指示を改善する」より「環境を設計する」。Claude が間違えそうな箇所を CLAUDE.md の禁止ルールで先手を打つほうが、プロンプト調整を繰り返すより確実に機能する"
> ([Claude Codeを1ヶ月使って気づいた「指示の改善では解決しない問題」](https://zenn.dev/yamada_ai_dev/articles/claude-code-why-harness), セクション "環境を設計する") ※2026-05-26に実際にfetch成功

> "Claude Codeを使い始めたけど『なんか思ったより賢くない』と感じたことはありませんか？その原因はほぼ確実に CLAUDE.md の設計にあります。効果的な設計の目安は 50〜150 行、2 週間ごとに見直しを行う。"
> ([Claude CodeのCLAUDE.md設計完全ガイド — 上級者が実践する7つの原則](https://zenn.dev/streamsolty/articles/90ff7a49d1bc53), セクション "7原則") ※2026-05-26に実際にfetch成功

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 中
**最終更新**: 2026-05-26

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

**アンチパターン**:
- エラーログへのアクセスなしにエージェントにバグ修正を依頼する（「何が壊れているか」を伝えずに依頼）
- エージェントの修正 PR を自動マージする（人間レビューなしの本番デプロイ）

**出典**:
- [Sentry Blog: Agents Need Production Context](https://blog.sentry.io/agents-need-production-context/) (Sentry公式ブログ) ※2026-05-27に実際にfetch成功

**確信度**: 高
**最終更新**: 2026-05-27

---
