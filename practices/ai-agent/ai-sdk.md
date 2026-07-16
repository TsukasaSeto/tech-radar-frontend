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

### 2. AI エージェントへの UI 状態伝達は「視覚的スナップショット」ではなく「意味的な構造化データ」で行う

チャット UI にエージェント機能を組み込む際、画面の内容をそのままエージェントに渡す実装（ASCII 化した座標データ、スクリーンショット等）は直感的だが、トークン効率と精度の両方で劣る。
代わりに、UI コンポーネント自身が「今どういう状態か」を構造化データとして自己申告する仕組みにする。React ではコンポーネントを高階コンポーネントで包むだけで、React のツリー構造に沿って自動的に文脈の階層が組み上がる。

**根拠**:
- 座標ベースの視覚的表現は同じ情報量に対してトークン消費が大きい。Sentry の実測では、ページ文脈がシステムプロンプトに占める割合が 85〜93% から 50〜80% まで低下し、Issue 詳細ページで約 2,100 → 約 300 トークン、ダッシュボードで約 5,500 → 約 1,300 トークンまで削減された
- 詳細データは最初から全部渡さず、エージェントが必要になった時点でオンデマンド取得する設計にすると、初期コンテキストをさらに小さく保てる
- トークン削減後も満足度・会話あたりのツール呼び出し数は従来方式と同水準を維持しており、精度を犠牲にしていない

**コード例**:
```tsx
// Good: コンポーネントが自身の意味的な状態を自己申告する
function DashboardWidget({ widget }: { widget: Widget }) {
  return (
    <AgentContextProvider
      description={`widget "${widget.title}" showing ${widget.metric}, filtered to ${widget.filter}`}
    >
      <WidgetView widget={widget} />
    </AgentContextProvider>
  );
}
// → エージェントが受け取るのは「本番環境に絞った8個のウィジェットを含む編集中のダッシュボード」
//   という意味であって、グリッド座標上の文字の羅列ではない

// Bad: 画面のASCII/座標スナップショットをそのままプロンプトに詰め込む
const pageSnapshot = renderAsciiGrid(domSnapshot); // 座標×文字の羅列、トークン消費大
const prompt = `Current screen:\n${pageSnapshot}\nWhat should I do?`;
```

**出典引用**:
> "Semantic, not visual. The agent receives meaning, 'dashboard in edit mode, 8 widgets filtered to production,' rather than characters at grid coordinates."
> ([Your agent should understand what you see](https://blog.sentry.io/seer-agent-page-context/), セクション "The new approach: pages that describe themselves") ※2026-07-16に実際にfetch成功

**バージョン**: フレームワーク非依存（React の高階コンポーネントパターンで実装）
**確信度**: 高（Sentry 公式ブログ）
**最終更新**: 2026-07-16

---
