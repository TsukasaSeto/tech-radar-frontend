# Vercel AI SDK 活用のベストプラクティス

## ルール

### 1. AI SDK 7 の `WorkflowAgent` / `HarnessAgent` でエージェント実行を耐障害化し、`reasoning` パラメータでプロバイダーを横断する

Vercel AI SDK v7（2026-06-25 GA）は、フロントエンド・Next.js アプリへの AI 組み込みにおいて以下の新パターンを提供する。

- **`WorkflowAgent`**: エージェント実行を永続化し、プロセス再起動・デプロイをまたいでも実行を継続できる。承認フロー（ヒューマン・イン・ザ・ループ）や型付きランタイムコンテキストを組み合わせると、長時間タスクのバックグラウンド実行が安全になる
- **`HarnessAgent`**: Claude Code・Codex などの確立済みエージェントハーネスを統一 API でラップする。ハーネスの差異を吸収しながら同一インターフェースで複数 AI ツールを扱える
- **`reasoning` パラメータ**: `generateText` / `streamText` に渡すだけでプロバイダーネイティブな推論設定にマッピングされる。プロバイダーを切り替えても推論強度の指定方法が変わらない
- **移行**: v6 → v7 は `npx @ai-sdk/codemod v7` で自動マイグレーション可能

**根拠**:
- v7 以前の `generateText` / `streamText` はプロセス終了でエージェントの状態が失われた。`WorkflowAgent` はワークフローベースのストリーミングで状態を永続化する
- `reasoning` は各プロバイダー固有の推論設定（Anthropic の thinking、OpenAI の o1 推論量など）をプロバイダー非依存の単一オプションに統一する。マルチプロバイダー構成でも推論強度を一貫して制御できる
- テレメトリが 1 回の登録でグローバル適用（従来の per-call コールバック方式から刷新）。Node.js トレーシングチャネルによる構造化診断でパフォーマンス統計（レスポンス時間・トークンスループット・TTFO）を本番計測できる

**コード例**:
```ts
// WorkflowAgent: プロセス再起動をまたいで実行継続
import { WorkflowAgent } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';

const agent = new WorkflowAgent({
  model: anthropic('claude-opus-4-8'),
  tools: { /* ツール定義 */ },
});

// ストリーム中にプロセスが再起動しても、再接続後に続きから再開できる
const { stream, workflowId } = await agent.stream({
  messages: [{ role: 'user', content: 'コードレビューを実行して' }],
});
```

```ts
// reasoning パラメータ: プロバイダーを問わず推論強度を統一制御
import { generateText } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';

const { text } = await generateText({
  model: anthropic('claude-opus-4-8'),
  reasoning: { effort: 'high' }, // プロバイダー固有設定に自動マッピング
  prompt: 'このアーキテクチャの設計上の問題点を分析して',
});
```

```ts
// HarnessAgent: Claude Code / Codex などを統一 API でラップ
import { HarnessAgent } from 'ai';

const agent = new HarnessAgent({
  harness: 'claude-code',
  // または 'codex' など確立済みハーネス名を指定
});

const result = await agent.run({ task: 'テストを追加して' });
```

```bash
# v6 → v7 自動マイグレーション
npx @ai-sdk/codemod v7
```

**アンチパターン**:
- `WorkflowAgent` を使わず通常の `generateText` で長時間エージェントを動かす（プロセス再起動で状態消失）
- プロバイダーごとに `thinking: { budget: 1000 }` など固有パラメータで推論を制御する（プロバイダー切り替え時に全書き換えが必要）
- コードモッドを実行せず手動でブレーキングチェンジを移行しようとする（漏れが生じやすい）

**出典引用**:
> "maps to provider-native reasoning settings, letting you control reasoning effort in a single line."
> ([AI SDK 7](https://vercel.com/blog/ai-sdk-7), セクション "Reasoning Control") ※2026-06-25に実際にfetch成功

> "global coverage of all AI SDK functions with a single registration"
> ([AI SDK 7](https://vercel.com/blog/ai-sdk-7), セクション "Production-Ready Observability") ※2026-06-25に実際にfetch成功

**バージョン**: AI SDK 7+（`ai` npm パッケージ）
**確信度**: 高（Vercel 公式ブログ）
**最終更新**: 2026-06-25

---
