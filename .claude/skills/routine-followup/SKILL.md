---
name: routine-followup
description: Use when the user wants to review and merge open PRs produced by the daily Frontend Practice Curator Routine (branches matching `practice-update/YYYY-MM-DD` and `seen-json-update/YYYY-MM-DD`). Also handles reading merged `feedback/YYYY-MM-DD.md` files and proposing prompt improvements to `.memo/prompt-v12.md`. Specific to the tech-radar-frontend repository. Trigger phrases include "Routine の PR を見て", "今日の Curator マージして", "feedback 反映して".
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
- 主プロンプト: `.memo/prompt-v12.md`（実行時の正本は cloud env 側、こちらは編集用の手元コピー）
- 環境定義: `.memo/env-tech-radar-curator.md`

## 全体フロー

```
1. オープン PR を一覧化（gh pr list）
2. ペアを古い順にループ:
   a. ペア内容を要約してユーザー確認（GO/NoGo）
   b. Practice Update PR を rebase → ルール採番修正 → push → merge
   c. _seen.json PR を rebase → URL dedup + 削除検知 → push → merge
3. マージ済 feedback ファイルを読んで改善提案を抽出
4. 改善提案をユーザーに提示、承認分を .memo/prompt-v12.md に反映

※ 未処理ペアが多いとき（目安 5 組以上）は、2 の代わりに「滞留時の手順」で進める
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

`gh pr list` は既定で 30 件までしか返さないため、件数がちょうど 30 なら `--limit 300` で取り直す。未処理ペアが目安 5 組以上ある、または全ブランチの merge base が同じ（main が止まっていた）場合は、Step 2 の代わりに「滞留時の手順」に進む。

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

## 滞留時の手順（未処理ペアが多いとき）

main が長く更新されないまま Routine が毎日走ると、未処理ペアが溜まる。各回の Routine は古い main を基準に動くため、次の問題が重なる。

- `_seen.json` が未マージなので、36h 窓が重なる隣接日の PR が同じ記事を再処理し、同じルールの二重強化や同趣旨の新規ルールが生じる
- 新規ルールがすべて同じ番号（例: claude-code.md の #50）で採番される
- Step 2 の流れで 1 本ずつ rebase すると、ほぼ毎回 `_seen.json` の末尾と採番が衝突する

そのため「全件を一時 worktree で先に統合する → まとめて確認を取る → PR ごとに機械的に push・merge する」の順で進め、判断が要る作業（重複の除外・採番・内容の修正）を push 前に済ませる。

### 滞留-1. 事前調査

```bash
# 分岐元の確認: 全件が同じ SHA なら滞留
for b in $(git branch -r --format='%(refname:short)' | grep -E 'origin/(practice-update|seen-json-update)/'); do
  git merge-base main "$b"
done | sort | uniq -c

# 隣接日重複の候補: practices に追加された URL のうち、複数の PR に出るもの
for b in $(git branch -r --format='%(refname:short)' | grep 'origin/practice-update/' | sort); do
  git diff main "$b" -- practices/ | grep '^+' | grep -oE 'https?://[^ )>"`]+' | sort -u | sed "s#^#${b##*/}\t#"
done > /tmp/practice-urls.tsv
cut -f2 /tmp/practice-urls.tsv | sort | uniq -d
```

- `_seen.json` の削除検知（2-3 の A）を全ブランチで実行し、PR 間で重複する URL の件数も数えておく
- 内容レビューは、期間を分けてサブエージェントに並列で依頼する。観点は、公式ドキュメント・原文との照合による正しさ、main 既存ルールとの重複、書式、changelog と差分の整合、既存内容の欠落。重大 / 中 / 軽微で報告させる
- 報告をもとに、指摘の扱い（統合時に直す / 誤りを除外するだけ / 重複処理だけ）をユーザーに確認する

### 滞留-2. 一時 worktree で日付順に統合

```bash
git worktree add <scratchpad>/wt -b tmp/integrate-YYYY-MM-DD main
jq . _seen.json | cmp - _seen.json   # jq の整形が main と同一か先に確認する
```

worktree から `git config` を実行すると、本体の `.git/config` に書き込まれる。diff3 などはコマンド単位で指定する（`git -c merge.conflictstyle=diff3 merge ...`）。

日付の古い順に、1 PR = 1 コミットで積む。PR 番号・squash の件名・本文は、コミットとは別にファイルへ記録しておく。

- practice PR: `git -c merge.conflictstyle=diff3 merge --squash origin/practice-update/YYYY-MM-DD` の後、コンフリクトの解消・採番の振り直し・重複の除外・レビュー指摘の修正を行い、changelog に注記してコミットする
- _seen.json PR: rebase はしない。現在の `_seen.json` に「PR が自分の merge base から追加した URL のうち未登録のもの」だけを PR 内の順序で末尾に足し、`jq` で整形し直す。`_seen.json` 以外のファイル（feedback 等）は PR から取り込む。追加が 0 件になった PR は merge せずクローズ候補にする

```bash
b=origin/seen-json-update/YYYY-MM-DD
git show "$(git merge-base main "$b"):_seen.json" | jq -r '.[]' > /tmp/prbase.txt
git show "$b:_seen.json" | jq -r '.[]' > /tmp/pr.txt
jq -r '.[]' _seen.json > /tmp/cur.txt
awk 'NR==FNR{s[$0]=1;next} !s[$0]' /tmp/prbase.txt /tmp/pr.txt \
  | awk 'NR==FNR{s[$0]=1;next} !s[$0]++' /tmp/cur.txt - > /tmp/to_add.txt
cat /tmp/cur.txt /tmp/to_add.txt | jq -R . | jq -s . > _seen.json
```

changelog への注記は、`## 追加・強化されたルール` 節の末尾に `### マージ時の修正（routine-followup）` 小節を作り、除外・統合・振り直し・修正の内容を箇条書きにする。サマリ行には調整後の件数を併記する。

コミットごとに次を確認する: コンフリクトマーカーが残っていない、採番が 1..N の連番、各ルールの前に `---` がある、ルールごとに最終更新が 1 行、`_seen.json` は 2-3 の C の 4 条件。

注意点:

- 新規ルールを除外・統合すると、それ以降に同じファイルへ追加される新規ルールの番号がすべて変わる。除外・統合の判断はレビュー結果が揃ってから始め、途中で判断を変えたらその PR から統合をやり直す
- macOS の BSD sed は `\|` による選択が効かない。置換は `-e` を分けて書く
- ファイル末尾の `---` の有無はファイルごとに異なる。main のそのファイルに合わせる

### 滞留-3. 最終検証と確認

- `_seen.json` が「main の base ∪ 全 PR の追加分」と一致する（sort して比較）。削除 0、重複 0
- 全 practices の採番とコンフリクトマーカー
- 各 changelog の `Rule #N` が最終状態の見出しを指している
- 実行計画（PR 番号・種別・統合コミット・件名。クローズする PR も含む）と修正内容の一覧をユーザーに示し、GO を取る

### 滞留-4. PR ごとに push → squash merge

古い順に、各 PR のブランチを「現在の main + その PR の統合結果」の 1 コミットに置き換えてから merge する。各 merge の後に main のツリーが統合結果と一致しなければ、その時点で止める。

```bash
tree=$(git rev-parse "<統合コミット>^{tree}")
git fetch origin main "<branch>" --quiet
old=$(git rev-parse "origin/<branch>")
new=$(git commit-tree "$tree" -p origin/main -m "<件名>")
git push --force-with-lease="refs/heads/<branch>:$old" origin "$new:refs/heads/<branch>"
# gh pr view <PR#> --json headRefOid,mergeStateStatus が "$new CLEAN" になるまで待つ
gh pr merge <PR#> --squash --subject "<件名>" --body-file <本文ファイル> --match-head-commit "$new"
git fetch origin main --quiet
[ "$(git rev-parse 'origin/main^{tree}')" = "$tree" ] || echo "STOP: main tree differs"
```

- 件数が多いときはスクリプトにしてバックグラウンドで流し、ログで進捗を確認する
- 追加 0 件の `_seen.json` PR は `gh pr close <PR#> --comment "<理由>"` で閉じる
- 完了後、ローカル main を `git pull --ff-only` で追従させ、worktree と一時ブランチを削除する

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

どれを .memo/prompt-v12.md に反映しますか？（番号で指定、複数可）
```

ユーザーが選んだものを `Edit` ツールで `.memo/prompt-v12.md` に反映する。1 件適用ごとに以下を実行:

- 元のテキストブロックと変更後テキストブロックを示してから Edit
- 過去版マーカー（`(v11.5 追加)` 等）は付けない（実行時のノイズになるため）
- `.memo/prompt-v12.md` の編集は手元コピーへの編集であり、cloud env への反映はユーザー手動（その旨を最後に明示）

## 共通の Don't / Caveat

- **`_seen.json` の URL 削除は禁止**（誤削除事故あり）。必ず B-A の検知を通す
- **`git push --force` は使わず `--force-with-lease`** を使う（ブランチが想定外に進んでいた場合の上書き防止）
- **ルール採番の振り直しを忘れない**: PR の主張する番号（多くの場合 #2）をそのまま merge すると採番衝突
- **changelog 側の Rule #N 表記も同期する**: ルール本体だけ変えると changelog と乖離する
- **重複ルールに気づいたら新規ではなく追加根拠化**: 同じ URL から既存ルール趣旨と同じ内容が来たら discard or enhancement
- **Auto Mode classifier に push を止められたら無理に通さずユーザーに承認を求める**
- **feedback の改善提案を採用しても、cloud env のプロンプトには自動で反映されない**: 必ずユーザー手動の反映手順を最後にリマインドする

## 参考: ファイル位置

- 主プロンプト編集用コピー: `.memo/prompt-v12.md`
- env 編集用コピー: `.memo/env-tech-radar-curator.md`
- 過去 changelog: `changelog/YYYY-MM-DD.md`
- feedback（v11.5 デプロイ後）: `feedback/YYYY-MM-DD.md`
- 既存ルール格納: `practices/{category}/{topic}.md`
- URL 既読リスト: `_seen.json`（重要 URL: `https://www.typescriptlang.org/docs/handbook/utility-types.html`）
