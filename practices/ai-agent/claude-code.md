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

**出典引用**:
> "The strength of Claude Code Skills lies in autonomously executing well-designed workflows. These 15 patterns ensure that initial design gets right."
> ([Claude Code Skills実装で踏んだ地雷15選 ─ 失敗パターンと対策を体系化した](https://zenn.dev/correlate_dev/articles/claude-code-skills-15), セクション "Key Insight") ※2026-05-08に実際にfetch成功

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 中
**最終更新**: 2026-05-08

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

### 5. Claude Code の Skills 機能でチーム作業ワークフローを自動化する

`.claude/skills/` ディレクトリに Markdown ファイルを置くとファイル名がスラッシュコマンドとして登録され、反復する作業手順を「実行可能なワークフロー」として切り出せる。
スキルは「ブランチ作成→実装→コミット→PR 作成」のように一連の操作をまとめる単位として設計する。
Claude がスキルを自動起動しない under-trigger 問題を防ぐため、`description` フィールドは「このスキルを使うべき条件」を明示的かつ強めの表現で記述する。

**根拠**:
- `.claude/skills/` に置いた Markdown ファイルのファイル名がそのままスラッシュコマンドになる
- under-trigger（使うべき場面で Claude がスキルを起動しない）は `description` の書き方で防げる
- `devcontainer.json` の `mounts` で Claude Config ディレクトリをボリュームに永続化すると rebuild 後も再認証が不要

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

**バージョン**: Claude Code（Skills 機能、2026年4月以降 gh CLI v2.90+）
**確信度**: 高
**最終更新**: 2026-05-09

---

### 6. `.claude/rules/` の `paths` フロントマターで条件付き読み込みを行い、コンテキストトークンを削減する

CLAUDE.md に全ルールを直接書かずに、`.claude/rules/` ディレクトリにファイルを分割し、
`paths` フロントマターで「そのルールが必要になる場面のみ」自動的に読み込まれるようにする。
`@import` はファイルを分割してもコンテキスト全体が注入されるため、トークン削減にならない点に注意。

**根拠**:
- `@import` はファイルを分割しても全体がコンテキストに注入される — それ自体ではトークン削減にならない
- `paths` フロントマターを設定すると、対象ファイル編集時のみルールが読み込まれる（条件付き読み込み）
- CLAUDE.md を目次として詳細を rules/ に委譲することで、トークン消費を最大 1/3 に削減できた事例がある
- 使用頻度の低いルールは `.claude/skills/` に `disable-model-invocation: true` で置くと呼び出すまでトークン0
- HTML コメント `<!-- -->` はコンテキスト注入前に除去されるためトークン0（内部メモに活用できる）

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

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 高
**最終更新**: 2026-05-11

---
