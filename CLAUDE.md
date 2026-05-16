# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## このリポジトリの性質

フロントエンド技術のベストプラクティスを継続的に蓄積する**ドキュメント中心リポジトリ**。コードを持たないので build / lint / test サイクルは無い。「読み物」ではなく「実プロジェクトに適用するルールブック」として、`AGENTS.md` / `CLAUDE.md` 経由で他プロジェクトに取り込まれる前提（詳細は @README.md）。

主な変更主体は人間ではなく cloud（claude.ai）上で毎朝 6:00 に走る `Frontend Practice Curator` Routine。**この repo での Claude Code セッションの大半は「Routine が生成した PR の取り込み」と「プロンプト改善」**であり、実装作業ではない。

## アーキテクチャ（ワークフロー全体像）

```
[claude.ai cloud Routine]                         [このリポジトリ]
   Frontend Practice Curator                          ↓
   ├── 直近 48h の RSS を収集                  practices/{cat}/{topic}.md  (主成果物)
   ├── _seen.json で既読除外               +  changelog/YYYY-MM-DD.md
   ├── 候補記事を採用基準で絞り込み        +  feedback/YYYY-MM-DD.md      (Routine 自己評価)
   ├── practices/ に新規 / 強化 ルール追加         ↑
   └── PR を 1 日に最大 3 種類起票:               ローカル Claude Code セッション
       ├── practice-update/YYYY-MM-DD              ├── PR を pair 単位で取り込み
       ├── seen-json-update/YYYY-MM-DD             ├── feedback を読んで改善案を抽出
       └── changelog-only/...-fetch-failed (稀)    └── .memo/prompt-v11.5.md を編集
```

- **プロンプト本体**は cloud env 側が正本。ローカルの `.memo/prompt-v11.5.md` は編集用コピーで、変更しても cloud には自動同期されない（claude.ai の Routine UI から手動反映が必要）
- **環境定義**（許可ドメイン・setup script）も同様に `.memo/env-tech-radar-curator.md` が編集用コピー

## メインの作業フロー: routine-followup スキル

PR 取り込みの全手順は `routine-followup` スキルが持つ。トリガ:

- `Routine の PR を見て` / `今日の Curator マージして` / `feedback 反映して`

詳細手順は `.claude/skills/routine-followup/SKILL.md` を参照。Claude Code セッションでこの種の依頼を受けたら、まず routine-followup スキルを invoke する。

## このリポジトリ固有の Don't

- **`_seen.json` の URL 削除禁止**。過去に PR #40 で `utility-types.html` を誤削除した事故あり。`_seen.json` を含む PR は merge 前に**必ず削除検知 diff を取る**（merge base からの sort 比較。手順は SKILL.md Step 2-3）
- **`git push --force` ではなく `--force-with-lease`** を使う（rebase 後の再 push が頻発するため）
- **changelog の `Rule #N` 表記とプラクティス本体の番号を同期させる**。rebase で番号衝突したら両方振り直す（`practices/ai-agent/claude-code.md` が常連）
- **`.memo/prompt-v11.5.md` の編集は cloud env プロンプトに自動反映されない** — 反映時は「claude.ai の Routine UI から手動編集が必要」と明示する
- **重複ルールに気づいたら新規ではなく既存ルールへの追加根拠化** or discard

## プラクティスファイルの書式（追加・編集時）

各プラクティスは固定構造。`### N.` 連番、ルール間 `---` 区切り。

```
### N. {主張を 1 文で動詞止めで}

{2-4 行の本文}

**根拠**: 箇条書き
**コード例**: ```lang フェンス（// Good: / // Bad:）
**アンチパターン**: 箇条書き（任意）
**出典**: リンク
**取り込み元**: ...（別プロジェクトからの手動取り込みのみ）
**バージョン**: 例: Next.js 16+
**確信度**: 高 / 中 / 低
**最終更新**: YYYY-MM-DD
```

- 確信度: 公式 or 複数独立ソース = 高 / 1 つの権威ある記事 = 中 / 1 記事の主張 = 低
- 完成形の典型例: `practices/architecture/data-access-layer.md`
- 新規トピック追加時は対応カテゴリの `README.md` のトピック一覧も更新

### Next.js 系の設計判断順位

`practices/nextjs/README.md` で **A: セキュリティ → B: アーキテクチャ → C: パフォーマンス** の優先順位が明示されている。Next.js プラクティスを追加・改訂する際は、ルール同士の衝突をこの優先順位で解決する。

## 詳細メモ

`.serena/memories/` に詳細を分割格納済み。必要に応じて Serena の `read_memory` で参照:

- `routine_workflow` — Curator Routine と PR 取り込みの全体像
- `practice_file_format` — プラクティス書式の詳細
- `key_files_and_locations` — 重要ファイルパス一覧
- `suggested_commands` — gh / git / jq の頻出コマンド（`_seen.json` の必須 4 検証込み）
- `task_completion_checklist` — プラクティス・`_seen.json`・PR マージ後の確認項目
- `code_style_conventions` — 文体・命名規約
- `directory_structure` / `project_overview` — リポジトリ概観
- `_meta/last_sync` — `/my:sync-serena-memory` の同期履歴
