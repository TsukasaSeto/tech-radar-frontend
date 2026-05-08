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
**確信度**: 高
**最終更新**: 2026-05-08

---

#### 追加根拠 (2026-05-08) — ルール1「Claude Code をチームで活用するために、ルール管理と失敗からの学習を体系化する」

新たに以下の記事でエージェント向けルールを実装レイヤーで機能させる3パターンが示された:
- [AIエージェント自律完遂率を上げるハーネス設計パターン3選](https://zenn.dev/aerign/articles/e5b561d7f1650b) (Zenn aerign / 2026-05-08) ※2026-05-08に実際にfetch成功

**出典引用**:
> "Every time I have to type continue to the agent is like a failure of the harness to provide enough context"
> (セクション "Background（希少資源の転換）")

**Lint as Prompt パターン**: CLAUDE.md・AGENTS.md に書くだけでなく、ESLint カスタムルールとして実装することで、違反時にエージェントが自己修正できる具体的な手順メッセージをリアルタイムで受け取れる。ルール本文でなく lint エラーメッセージが「今すぐ直す方法」を伝える。

**Structure as Prompt パターン**: ファイル行数制限（350行）を pytest で CI 強制することで、エージェントが自然とヘルパー関数に分割するよう誘導できる。過剰な局所最適化を防ぎ、コードベース全体の一貫性を維持する。

**実装優先順位**:
1. AGENTS.md（または CLAUDE.md）にルールを記述（既存アプローチ1・2）
2. ルールを ESLint カスタムルールに変換し lint エラーとして伝達（Lint as Prompt）
3. ファイル構造制約を CI で強制（Structure as Prompt）
4. Reviewer Agent を CI に組み込み（GitHub Actions で PR を自動レビュー）

```js
// ESLint カスタムルール例: fetch() のタイムアウト必須化（Lint as Prompt）
// エラーメッセージ文言がエージェントへの修正指示になる
module.exports = {
  create(context) {
    return {
      CallExpression(node) {
        if (node.callee.name === 'fetch') {
          const options = node.arguments[1];
          const hasSignal = options?.properties?.some(
            (p) => p.key?.name === 'signal'
          );
          if (!hasSignal) {
            context.report({
              node,
              message:
                'fetch() must include a signal for timeout control. ' +
                'Add: const controller = new AbortController(); ' +
                'fetch(url, { signal: controller.signal })',
            });
          }
        }
      },
    };
  },
};
```

**確信度**: 既存（中）→ 高（コミュニティ3記事＋コード例で確認）

---

### 2. マルチエージェント設計にはAnthropicの5パターンを活用し、Orchestrator-Subagentを基本とする

複数の AI エージェントを組み合わせる場合、Anthropic が定義する5つの調整パターンを参照して設計を分類し、コスト・品質・実装複雑度を意識して選択する。

**根拠**:
- マルチエージェントは単一エージェントの3〜10倍のトークンを消費するため、パターン選択の根拠が必要
- Orchestrator-Subagent は情報ボトルネックによる整合性維持と実装のシンプルさから、多くのユースケースで最も適切な出発点となる
- Generator-Verifier は明示的な評価基準（ルーブリック）を設ければコードレビューや品質チェックの自動化に有効
- Agent Teams は役割別コンテキストを分離して並列作業を実現するが、管理コストが高く3〜5体から開始すること

**5パターンの選択ガイド**:

| パターン | 適用場面 | 注意点 |
|---|---|---|
| **Orchestrator-Subagent**（推奨デフォルト） | 順次タスク分解、多くのユースケース | 逐次実行が律速。並列化は明示的に設計 |
| **Generator-Verifier** | 品質基準が明確な自動レビュー・テスト生成 | 評価基準が曖昧だとラバースタンプになる |
| **Agent Teams** | 3〜5エージェントによる役割分担・高速並列 | 共有タスクリスト実装が必須 |
| **Shared State** | 設計システム進化・並行更新が必要な場合 | リアクティブループの終了条件を明示 |
| **Message Bus** | 本当に非同期イベント分解が必要な場合のみ | DLQ + 分散トレーシングが必須 |

**コード例**:
```typescript
// Generator-Verifier パターン: 評価基準と反復上限を必ず設ける
interface VerificationResult {
  passed: boolean;
  feedback: string;
}

async function generatorVerifier(
  task: string,
  rubric: string[],
  maxIterations = 3,
): Promise<{ result: string; converged: boolean }> {
  let result = '';
  let feedback = '';

  for (let i = 0; i < maxIterations; i++) {
    result = await generator.generate(task, feedback);
    const verification: VerificationResult = await verifier.evaluate(result, rubric);

    if (verification.passed) {
      return { result, converged: true };
    }
    feedback = verification.feedback;
  }

  // 収束しない場合: サイレント失敗を防ぎ caveats 付きで返す
  return { result: `[CAVEAT: 未収束] ${result}`, converged: false };
}
```

**出典引用**:
> "The verifier is only as good as its criteria" / "Acts as information bottleneck; sequential execution limits throughput unless explicitly parallelized"
> ([Anthropic の 5 パターンで Claude Code エージェント設計を分類する](https://zenn.dev/motowo/articles/anthropic-multi-agent-coordination-patterns-guide), セクション "Generator-Verifier Pattern" / "Orchestrator-Subagent Pattern") ※2026-05-08に実際にfetch成功

**バージョン**: Claude Code（全バージョン共通）
**確信度**: 中
**最終更新**: 2026-05-08

---
