# AI エージェント活用のベストプラクティス

AI エージェント（Claude Code・Cursor・Devin 等）をフロントエンド開発に活用する際の
ベストプラクティスを管理する。

## ファイル構成

- `claude-code.md` — Claude Code をチームで活用するためのプラクティス（Codex CLI など他のコーディングエージェントとの併用、Next.js のエージェント向け機能との連携を含む）
- `ai-sdk.md` — アプリに AI エージェント機能を組み込むためのプラクティス。Vercel AI SDK・Claude Managed Agents・WebMCP（`mcp-handler`）の使い方に加え、プロンプトインジェクション対策・UI 状態の渡し方・権限委譲など、フレームワークに依存しない設計原則を含む
