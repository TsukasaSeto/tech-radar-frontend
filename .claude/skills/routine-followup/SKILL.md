---
name: routine-followup
description: Use when the user wants to review and merge open PRs produced by the daily Frontend Practice Curator Routine (branches matching `practice-update/YYYY-MM-DD` and `seen-json-update/YYYY-MM-DD`). Also handles reading merged `feedback/YYYY-MM-DD.md` files and proposing prompt improvements to `.memo/prompt-v11.5.md`. Specific to the tech-radar-frontend repository. Trigger phrases include "Routine の PR を見て", "今日の Curator マージして", "feedback 反映して".
---

# Routine PR Follow-up Workflow

このスキルは tech-radar-frontend の毎朝 6:00 実行される Curator Routine が生成する PR を取り込み、feedback を読んでプロンプト改善案をユーザーに提示する一連の作業を担う。

## 前提知識

- Routine は 1 日に最大 3 種の PR を生成する:
  - `practice-update/YYYY-MM-DD` : `practices/` と `changelog/` の追加（PR #55 など）
  - `seen-json-update/YYYY-MM-DD` : `_seen.json` への URL 追加（PR #56 など）
  - `changelog-only/YYYY-MM-DD-fetch-failed` : 全取得失敗時のみ（稀）
- いずれのケースでも `feedback/YYYY-MM-DD.md` がいずれかの PR に必ず含まれる（v11.5 以降）
- マージ順序: **古い日付ペアから順に**、ペア内では **Practice Update → _seen.json** の順
- 主プロンプト: `.memo/prompt-v11.5.md`（実行時の正本は cloud env 側、こちらは編集用の手元コピー）
- 環境定義: `.memo/env-tech-radar-curator.md`

## 全体フロー

```
1. オープン PR を一覧化（gh pr list）
2. ペアを古い順にループ:
   a. ペア内容を要約してユーザー確認（GO/NoGo）
   b. Practice Update PR を rebase → ルール採番修正 → push → merge
   c. _seen.json PR を rebase → URL dedup + 削除検知 → push → merge
3. マージ済 feedback ファイルを読んで改善提案を抽出
4. 改善提案をユーザーに提示、承認分を .memo/prompt-v11.5.md に反映
```

## Step 1: PR 一覧化と分類

```bash
gh pr list
```

タイトルとブランチ名から日付別ペアに振り分ける。例:

```
55 Practice Update 2026-05-16 ... practice-update/2026-05-16
56 chore: update _seen.json ...   seen-json-update/2026-05-16
```

未処理ペアがなければスキルを終了する（feedback 読み取りステップへも進まない）。

## Step 2: ペアごとのレビュー＆マージ

### 2-1. ユーザーへの中間確認

各ペアに着手する前に、必ず以下を要約してユーザーに GO/NoGo を取る:

- 日付・PR 番号・タイトル
- Practice Update の追加内容（新規 N 件 / 強化 M 件）
- `_seen.json` PR の `additions` / `deletions` 数値（**deletions > 0 は URL 削除疑惑なので強調**）
- mergeable 状態

例:

> 2026-05-16 ペア（PR #55 / #56）に進みます:
> - #55: 3 rules enhanced（ai-agent #5/#10, architecture #4）
> - #56: +13 / -1 ← 削除が 1 行ありますが JSON 末尾 `]` の移動なので URL 削除ではないことを確認済み
> - 両方 MERGEABLE CLEAN
>
> 進めてよいですか？

### 2-2. Practice Update PR の処理

#### A. コンフリクト確認

```bash
gh pr view <PR#> --json mergeable,mergeStateStatus
```

- `CLEAN` なら次の C へ
- `CONFLICTING` なら B へ

#### B. rebase とコンフリクト解消

```bash
git fetch origin practice-update/YYYY-MM-DD --quiet
git checkout -B practice-update/YYYY-MM-DD origin/practice-update/YYYY-MM-DD
git rebase main
```

コンフリクトが起きる主要ファイル: `practices/ai-agent/claude-code.md`（特に複数日触れる）

**ルール採番の振り直し手順**:

1. main 上の対象ファイルで最大ルール番号を取得:
   ```bash
   grep -E "^### [0-9]+\." practices/ai-agent/claude-code.md | tail -1
   ```
2. PR が追加しようとしているルール（`<<<<<<< HEAD` と `=======` の間に main の既存ルール、`=======` と `>>>>>>>` の間に PR の新ルールが入る）の番号を次番から振り直す
3. コンフリクトマーカー (`<<<<<<<`, `=======`, `>>>>>>>`) を全て削除
4. ルール間に `---` 区切り行が抜けていないか確認
5. changelog ファイル（例: `changelog/YYYY-MM-DD.md`）の `Rule #N` 表記も同じ番号に揃える

**重複ルール検知（v11.5 で軽減されるが旧 PR では発生）**:

同じソース URL から複数日にまたがって同じ趣旨のルールが追加されている場合（例: 2026-05-13 と 2026-05-14 で `zenn.dev/ui_memo/.../efec949e5cd9d1` から `<details name>` ルール）、後発を discard する:

- HEAD 側を採用、PR 側のルール本体を削除
- changelog の該当ルールエントリを「マージ時に重複として除外」注記に書き換え
- サマリの「新規 N 件」も対応して減算

#### C. push と merge

```bash
git add <resolved files>
git rebase --continue
git push --force-with-lease origin practice-update/YYYY-MM-DD
sleep 6  # mergeable 状態の更新待ち
gh pr merge <PR#> --squash --body "<concise title summarizing the merge>"
git checkout main
git pull --ff-only origin main
```

**注意（permission）**: 既存ブランチへの初回 push は Auto Mode classifier に止められる可能性がある。止められたらユーザーに承認を求める。

### 2-3. _seen.json PR の処理

#### A. URL 削除事故の検知（必須）

main と PR ブランチの merge base を取り、削除された URL が無いか確認:

```bash
base=$(git merge-base main origin/seen-json-update/YYYY-MM-DD)
diff <(git show $base:_seen.json | jq -r '.[]' | sort) \
     <(git show origin/seen-json-update/YYYY-MM-DD:_seen.json | jq -r '.[]' | sort) \
     | grep '^<' | head -5
```

何か出力されたら **削除疑惑**。即座にユーザーに通知し、後続の merge を保留する（PR #40 で `utility-types.html` が誤削除された事故あり）。

#### B. rebase + コンフリクト解消

```bash
git fetch origin seen-json-update/YYYY-MM-DD --quiet
git checkout -B seen-json-update/YYYY-MM-DD origin/seen-json-update/YYYY-MM-DD
git rebase main
```

`_seen.json` のコンフリクトは末尾の URL 追加領域で発生する。解消方針:

1. `<<<<<<< HEAD` 〜 `=======` の HEAD 側（main で追加された URL 群）を残す
2. `=======` 〜 `>>>>>>>` の PR 側（このブランチが追加したい URL 群）を続けて append
3. HEAD 末尾の URL 行に `,` を必ず追加（JSON 配列継続のため）
4. PR 側の URL に main 側と重複するものがあれば PR 側から削除（dedup）

#### C. 検証（push 前必須）

```bash
grep -n "<<<<<\|=====\|>>>>>" _seen.json    # マーカー残存チェック → 何も出ないこと
jq 'length' _seen.json                      # 件数チェック
jq -r '.[]' _seen.json | sort | uniq -d     # 重複チェック → 何も出ないこと
grep -c utility-types _seen.json            # 既存重要URLの残存確認 → 1 が出ること
```

4 条件すべてパスしてから push。

#### D. push と merge

```bash
git add _seen.json
git rebase --continue
git push --force-with-lease origin seen-json-update/YYYY-MM-DD
sleep 6
gh pr merge <PR#> --squash --body "chore: update _seen.json with N new URLs from YYYY-MM-DD (M dups removed)"
git checkout main
git pull --ff-only origin main
```

### 2-4. ペア完了後

次のペアへ進む前に、現在の todo を完了マークし、次ペアの中間確認に進む。

## Step 3: feedback ファイルのレビュー

全ペアのマージが完了したら、今回マージした feedback ファイルを読む:

```bash
# 今回マージで main に入った feedback ファイル一覧
git log --name-only --pretty=format: -<マージ件数> main | grep '^feedback/' | sort -u
```

各ファイルを Read し、以下を抽出:

- **「観察した問題」セクション** から再現性「継続的」のものを優先
- **「プロンプト改善提案」セクション** から優先度「高」「中」のものを優先
- **「特筆事項なし」のみ** のファイルはスキップ

過去日との重複も判定（直近 7 日の feedback/*.md を grep）。

## Step 4: プロンプト改善提案の提示と適用

ユーザーに提案を提示する形式:

```
## feedback から抽出した改善提案（N 件）

### [高] {提案サマリ}
- 出典: feedback/YYYY-MM-DD.md
- 対象セクション: {セクション}
- 提案: {差分の概要}
- 既出: あり / なし（直近 7 日）

### [中] ...

どれを .memo/prompt-v11.5.md に反映しますか？（番号で指定、複数可）
```

ユーザーが選んだものを `Edit` ツールで `.memo/prompt-v11.5.md` に反映する。1 件適用ごとに以下を実行:

- 元のテキストブロックと変更後テキストブロックを示してから Edit
- 過去版マーカー（`(v11.5 追加)` 等）は付けない（実行時のノイズになるため）
- `.memo/prompt-v11.5.md` の編集は手元コピーへの編集であり、cloud env への反映はユーザー手動（その旨を最後に明示）

## 共通の Don't / Caveat

- **`_seen.json` の URL 削除は禁止**（誤削除事故あり）。必ず B-A の検知を通す
- **`git push --force` は使わず `--force-with-lease`** を使う（ブランチが想定外に進んでいた場合の上書き防止）
- **ルール採番の振り直しを忘れない**: PR の主張する番号（多くの場合 #2）をそのまま merge すると採番衝突
- **changelog 側の Rule #N 表記も同期する**: ルール本体だけ変えると changelog と乖離する
- **重複ルールに気づいたら新規ではなく追加根拠化**: 同じ URL から既存ルール趣旨と同じ内容が来たら discard or enhancement
- **Auto Mode classifier に push を止められたら無理に通さずユーザーに承認を求める**
- **feedback の改善提案を採用しても、cloud env のプロンプトには自動で反映されない**: 必ずユーザー手動の反映手順を最後にリマインドする

## 参考: ファイル位置

- 主プロンプト編集用コピー: `.memo/prompt-v11.5.md`
- env 編集用コピー: `.memo/env-tech-radar-curator.md`
- 過去 changelog: `changelog/YYYY-MM-DD.md`
- feedback（v11.5 デプロイ後）: `feedback/YYYY-MM-DD.md`
- 既存ルール格納: `practices/{category}/{topic}.md`
- URL 既読リスト: `_seen.json`（重要 URL: `https://www.typescriptlang.org/docs/handbook/utility-types.html`）
