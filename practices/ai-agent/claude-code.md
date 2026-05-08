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

### 2. Claude Code ハーネスを設計してエージェントの自律完遂率を上げる

エージェントが途中で止まらない「ハーネス（実行環境）」を先に整備する。
人間が「continue」と入力し続けているのはハーネス設計の失敗サインであり、コーディング本体に着手する前にハーネスを設計することで自律完遂率が大きく向上する。

**根拠**:
- エージェントは巨大なファイルを生成すると「局所コンテキスト」しか持てず、既存ヘルパーを見落として再実装するため、行数制限がコンテキスト節約に有効
- Lint as Prompt はエージェントが自律的に修正できるヒントを機械的に提供できる
- Reviewer Agent CI はレビュー観点を人間に依存せず自動化する

**3つのハーネスパターン**:

**パターン1: 構造ガードレール（Structural Guardrails）**
ファイル行数・複雑度の上限を Linter/テストで自動強制する。エラーメッセージには修正手順を必ず含める。

**パターン2: Lint as Prompt**
ESLint カスタムルールのエラーメッセージに「望ましいコードの例」を直接埋め込む。
エージェントはエラーメッセージを読んで自律的に修正できる。

**パターン3: Reviewer Agent CI**
GitHub Actions で LLM ベースのレビュアーを動作させ、非機能要件（セキュリティ・パフォーマンス・アクセシビリティ等）との照合を自動化する。

**コード例**:
```js
// ESLint カスタムルール: lint as prompt の実装例
create(context) {
  return {
    CallExpression(node) {
      if (node.callee.name === 'fetch' && !hasAbortController(node)) {
        context.report({
          node,
          message:
            'fetchにはタイムアウト制御が必要です。以下のパターンを使用してください:\n\n' +
            'const controller = new AbortController();\n' +
            'const timeoutId = setTimeout(() => controller.abort(), 5000);\n' +
            'try {\n' +
            '  const res = await fetch(url, { signal: controller.signal });\n' +
            '} finally { clearTimeout(timeoutId); }'
        });
      }
    }
  };
}
```

**実装の優先順序**:
1. `AGENTS.md` 作成（非機能要件を明記）
2. Linter エラーメッセージの改善（修正手順を含める）
3. Reviewer Agent CI の導入

**出典引用**:
> "『continue』と入力するたび、ハーネス設計の失敗を示すシグナル"
> ([AIエージェント自律完遂率を上げるハーネス設計パターン3選](https://zenn.dev/aerign/articles/e5b561d7f1650b), セクション "核心の原理") ※2026-05-08に実際にfetch成功

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 中（単一記事 + コード例あり）
**最終更新**: 2026-05-08

---

### 3. Anthropic の 5 マルチエージェント協調パターンを用途で選択する

Claude Code のマルチエージェント設計には Anthropic が定義する 5 つの協調パターンがある。
タスク構造・レイテンシ・スケール・観測性の 4 軸で適切なパターンを選択する。

**根拠**:
- Anthropic 公式のマルチエージェント設計ガイドに基づく分類

**5パターン選定表**:

| パターン | 向くケース | 主な落とし穴 |
|---|---|---|
| Generator-Verifier | 品質検証が必要なコード生成・ファクトチェック | 基準が曖昧だと「ハンコレビュー」化する |
| Orchestrator-Subagent | 境界が明確なタスク委譲・デバッグ容易性重視 | 親が情報ボトルネックになる |
| Agent Teams | 複数視点の並列作業・チームメイト間直接通信 | トークンコストが人数分線形にスケール |
| Message Bus | 疎結合な非同期イベント処理 | サイレント失敗の検出が困難 |
| Shared State | 中央調整なしの協調・創発的情報統合 | 終了条件なしで無限ループ化するリスク |

**パターン選定の 4 軸**:
- **タスク構造**: 独立したサブタスクか、相互依存するタスクか
- **レイテンシ**: リアルタイム応答が必要か、バッチ処理でよいか
- **スケール**: 少数（2〜5）エージェントか、多数エージェントか
- **観測性**: デバッグ・監視要件がどの程度厳しいか

**実装ガード（パターン別）**:
- Generator-Verifier: 反復上限 + フォールバック戦略を必ず設定
- Orchestrator-Subagent: 結果フォーマットを事前に定義し、親の情報ボトルネック化を防ぐ
- Agent Teams: 楽観ロック・ファイル所有権を明示してファイル競合を防ぐ
- Shared State: 終了条件・収束しきい値を必ず設定し、無限ループを防ぐ

**コード例**:
```json
// Agent Teams の有効化（Claude Code v2.1.32+）
// ~/.claude/settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

**出典引用**:
> "Generator が出力を生成し、Verifier が明示的な基準に照らして評価する"
> ([Anthropic の 5 パターンで Claude Code エージェント設計を分類する](https://zenn.dev/motowo/articles/anthropic-multi-agent-coordination-patterns-guide), セクション "Generator-Verifier") ※2026-05-08に実際にfetch成功

> "チームメイト同士が直接メッセージ交換可能" で「共有タスクリストで自律調整」される
> ([Claude Code Agent Teams完全ガイド — 並列エージェントでレビュー・デバッグを高速化](https://qiita.com/kai_kou/items/e47e94b1b05cc67f219b), セクション "主な特徴") ※2026-05-08に実際にfetch成功

**バージョン**: Claude Code v2.1.32+（Agent Teams は実験的機能）
**確信度**: 高（Anthropic 公式パターンを解説した記事 + コード例）
**最終更新**: 2026-05-08

---

### 4. CLAUDE.md / Skills / Subagents / Hooks / Auto Mode を目的別に使い分ける

各機能は「advisory（参照的）vs deterministic（確実）」「コンテキスト節約 vs 永続ルール」の軸で役割が異なる。

**根拠**:
- Anthropic 公式ドキュメントに基づく各機能の設計意図の違い
- CLAUDE.md と Hooks の違いを正しく理解せずに使うと意図した動作にならない

**使い分け表**:

| やりたいこと | 使う機能 | 理由 |
|---|---|---|
| チームへの永続ルール共有 | CLAUDE.md | advisory（参照するが無視も可） |
| **必ず**実行させる処理 | Hooks | deterministic（必ず実行される） |
| コンテキスト節約・専門タスク委譲 | Subagents | 別コンテキストで独立実行 |
| ドメイン知識・繰り返し作業の再利用 | Skills | `.claude/skills/` に定義 |
| 承認プロンプトを削減 | Auto Mode | 安全性分類器が自動判定 |

**核心の区別**:
- **CLAUDE.md はアドバイス**: Claude が参照するが状況によって無視することがある
- **Hooks は強制**: イベントトリガー時に必ず実行される。lint・テスト実行など「必ず走らせたい処理」に使う
- **Subagents はコンテキスト管理ツール**: コンテキストウィンドウが根本的な制約であるため、Subagents は最も強力なリソース管理手段

**コード例**:
```json
// settings.json: Auto Mode + 環境変数の明示設定
{
  "env": {
    "NODE_ENV": "development",
    "API_BASE_URL": "http://localhost:3000"
  },
  "permissionMode": "auto"
}
```

```jsonc
// Hooks 設定例: ツール使用前に必ずリントを実行
// .claude/settings.json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo '[hook] pre-bash check OK'"
          }
        ]
      }
    ]
  }
}
```

**出典引用**:
> "CLAUDE.md is advisory, hooks are deterministic"
> ([Claude Code 固有機能の使い分け — Skills / Subagents / Hooks / Auto Mode を公式準拠で解説](https://qiita.com/teppei19980914/items/95b9bf08087bef026855), セクション "Hooks") ※2026-05-08に実際にfetch成功

> "context is your fundamental constraint, subagents are one of the most powerful"
> ([Claude Code 固有機能の使い分け — Skills / Subagents / Hooks / Auto Mode を公式準拠で解説](https://qiita.com/teppei19980914/items/95b9bf08087bef026855), セクション "Subagents") ※2026-05-08に実際にfetch成功

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 高（Anthropic 公式ドキュメント準拠）
**最終更新**: 2026-05-08

---

### 5. Claude Code Skills 実装の主要地雷パターンを回避する

Skills の実装では陥りやすいパターンが多く、特に「エラーメッセージ」「コンテキスト管理」「環境変数」の3領域で失敗が集中する。

**根拠**:
- 実際の Claude Code Skills 実装経験から抽出した15パターンのうち、影響が大きい5項目

**最優先対策5項目**:

**① エラーメッセージに「何が・なぜ・次に何を」を明記する**
- `exit 1` のみでは Claude が失敗理由を理解できずループする
- 「何が失敗した」「なぜ」「次に何をすべきか」の3点をエラーメッセージに含める

**② パラメータ型定義を厳格にする**
- `type: string` のみでは相対パス・絶対パス・URL が混在して誤動作
- `pattern` と `enum` で制約を追加し、バリデーション責務を曖昧にしない

**③ Context Window の truncation を考慮する**
- 長いセッションで Skill 定義が切り捨てられる
- 定期的に `/clear` でリセットし、長い作業は複数セッションに分割する

**④ 並列 Skill 実行の競合状態を防ぐ**
- 複数 Skill が同一ファイルに並行書き込みすると内容が破損
- ロック機構または逐次実行を明示的に指示する

**⑤ 環境変数は settings.json で宣言する**
- Bash Tool はサブシェル実行でターミナルの設定値が引き継がれない
- `.claude/settings.json` の `env` セクションで明示的に宣言する

**コード例**:
```bash
# Bad: Claude がループする原因
exit 1

# Good: 3点明記
echo "ERROR: テスト失敗（ファイル: src/user.test.ts, テスト: UserService.getById）"
echo "原因: モックの戻り値が null のため、実装側の null チェックが未実装"
echo "対処: src/user.service.ts の getById に null ガードを追加するか、モック設定を修正してください"
exit 1
```

```json
// settings.json: 環境変数の明示宣言
{
  "env": {
    "NODE_ENV": "test",
    "DATABASE_URL": "postgresql://localhost:5432/testdb"
  }
}
```

**出典引用**:
> "エラーが`exit 1`のみではClaudeが失敗理由を理解できずループする"
> ([Claude Code Skills 実装で踏んだ地雷15選 ─ 失敗パターンと対策を体系化した](https://zenn.dev/correlate_dev/articles/claude-code-skills-15), セクション "エラーメッセージがClaudeに伝わらない") ※2026-05-08に実際にfetch成功

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 中（単一記事 + 詳細なコード例）
**最終更新**: 2026-05-08

---
