# GitHub Actions のベストプラクティス

フロントエンドの CI で「速く・確実に・安全に」回すための GitHub Actions 設計。

## ルール

### 1. 標準パイプラインを `ci.yml` 1 本に集約する

`typecheck` / `lint` / `test` / `build` の 4 ステップを `.github/workflows/ci.yml` に集約し、PR で必ず実行する。
ワークフローを細分化しすぎず、最小限のジョブで保守性を保つ。短いチェックは 1 ジョブの中で順に実行し、checkout と依存インストールを 1 回で済ませる（Rule #14）。

**根拠**:
- 細かい workflow file が増えると保守不能になる（同じセットアップを毎回書く）
- 1 ファイルで全ジョブを管理すると node セットアップ・cache キーなどの共通化が容易
- ジョブを分けるとジョブ数だけ checkout と依存インストールが重複し、課金も 1 分未満切り上げでジョブごとに積み上がる（Rule #12）
- ジョブ間の依存（`needs:`）で並列化と直列化を制御できる
- failed run の調査も 1 ファイル内なら追いやすい

**標準テンプレート**（pnpm + Node セットアップは Composite Action に切り出して共通化する）:

```yaml
# .github/actions/setup/action.yml — Install pnpm, Node, deps
name: Setup
description: Install pnpm, Node.js and dependencies
runs:
  using: composite
  steps:
    - uses: actions/setup-node@v7
      with: { node-version: 24 }
    # 実運用ではサードパーティ action を SHA でピン留めする（Rule #9）
    # pnpm/setup は pnpm v11+ 専用。pnpm のバージョンは package.json の packageManager から読む
    # setup-node の cache: pnpm は pnpm が先に PATH にないと失敗するため、store のキャッシュは pnpm/setup 側で行う
    - uses: pnpm/setup@v3
      with: { node-version-file: false, cache: true, install: false }   # Node は setup-node に任せ、install は次で明示する
    - shell: bash
      run: pnpm install --frozen-lockfile
```

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  # Good: 4 ステップを 1 ジョブで順に実行し、セットアップを 1 回にまとめる
  # Bad: typecheck / lint / test / build を別ジョブにして、セットアップを 4 回繰り返す
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with: { persist-credentials: false }
      - uses: ./.github/actions/setup
      - run: pnpm typecheck
      - run: pnpm lint
      - run: pnpm test --reporter=verbose
      - run: pnpm build
        env:
          NEXT_PUBLIC_APP_URL: https://example.com
```

テストやビルドが長くなり、並列化で縮む時間が重複するセットアップ（checkout + install）の時間を上回ったときだけ、そのジョブを分ける（シャーディングは Rule #10）。分けた場合も setup は Composite Action を使い、`actions/setup-node` / `pnpm/setup` / `pnpm install` の 3 ステップを各ジョブに直書きしない。`actions/cache` を使う場合も Composite Action 内に取り込む。

**判断軸**:
- ワークフローは「トリガー × デプロイ先」で分ける
  - `ci.yml`: PR・push の検証
  - `deploy-staging.yml`: staging デプロイ（main ブランチ）
  - `release.yml`: タグ作成・本番リリース
  - `nightly.yml`: スケジュールジョブ（依存監査・E2E 等）
- 同じトリガーで複数 workflow を分けない（並列実行の依存管理が困難）。ただし次の場合は別 workflow にしてよい
  - 必要な `permissions` が違う（例: PR にコメントする `bundle-size.yml`（Rule #5）は `pull-requests: write` が要るが、`ci.yml` は `contents: read` だけで済む）
  - トリガーの `types` や `paths` が違う（例: PR タイトルの lint は `edited` でも動かす `lint-pr-title.yml`（`practices/dx/code-review.md` #1））
  - 秘密情報を扱う処理を信頼できないコードの実行から分離する（Rule #6 の二分割パターン）

**出典**:
- [GitHub Actions: Workflow syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions) (GitHub Docs)
- [GitHub: Composite actions](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action) (GitHub Docs)
- [pnpm/setup](https://github.com/pnpm/setup) (pnpm 公式、v3 は pnpm v11+ 専用。`cache` / `install` / `node-version-file` 入力。pnpm 10 以前は `pnpm/action-setup` を使う) ※2026-10-25 確認

**バージョン**: GitHub Actions（例は Node 24、pnpm 11、actions/checkout v7 / actions/setup-node v7 / pnpm/setup v3）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 2. `concurrency` で同じブランチの古い実行を自動キャンセル

PR で連続 push した時に古い CI run をキャンセルする `concurrency:` 設定を入れる。
無駄な CI 実行時間を削減し、フィードバックを速くする。

**根拠**:
- PR で 5 回連続 push すると最初の 4 回の CI run は無意味（最終結果しか見ない）
- `concurrency.cancel-in-progress: true` で自動的に古い run をキャンセル
- main ブランチや release タグでは実行中の run をキャンセルしない
- `cancel-in-progress: false` でも、同じグループで待機（pending）できるのは既定で 1 件だけ。新しい run が来ると既存の pending は置き換えられる（キャンセルされる）ため、「全 run が順に実行される」わけではない
- 全 run を順に実行したい場合は `queue: max` を指定する（pending を最大 100 件まで保持し、超えた分はキャンセル。`cancel-in-progress: true` との併用は workflow の検証エラーになる）
- GitHub Actions の billing は run-minute 課金なので、cancel は直接コスト削減

**設定例**:
```yaml
# Workflow レベル
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

# main / release では cancel しない
# pull_request では cancel する
```

**`group:` のキー設計**:
```yaml
# パターン 1: workflow + ref（最も一般的）
group: ${{ github.workflow }}-${{ github.ref }}

# パターン 2: workflow + PR 番号（PR 専用ワークフロー）
group: ${{ github.workflow }}-pr-${{ github.event.pull_request.number }}

# パターン 3: deployment 排他制御（同じ環境への並行デプロイを防止）
group: deploy-${{ inputs.environment }}
cancel-in-progress: false  # 実行中のデプロイは止めない。待機できるのは最新の 1 件だけ
```

**ジョブレベルの concurrency**:
```yaml
jobs:
  deploy:
    concurrency:
      group: deploy-production
      cancel-in-progress: false  # 実行中の production デプロイは中断しない
      # 既定（queue: single）では、待機中の古いデプロイは新しい run に置き換わり、最新だけが続けて実行される
      # 全コミットを順にデプロイしたい場合だけ queue: max にする（最大 100 件。cancel-in-progress: true とは併用不可）
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

**よくある失敗**:
- main への push にも `cancel-in-progress: true` を適用 → 連続 push で実行中の検証がキャンセルされ、どのコミットで壊れたか追えなくなる
- `cancel-in-progress: false` なら全 run が順に実行されると思い込む → 待機できるのは 1 件だけで、間の run はキャンセルされる
- group キーが PR 単位でない → 別 PR が同じグループに入ってお互いキャンセル
- production デプロイで cancel → 中途半端なデプロイ状態が残る

**判定の原則**:
| シチュエーション | cancel-in-progress |
|---|---|
| PR の検証（typecheck / test 等） | `true` |
| main ブランチへの merge 後の検証 | `false`（実行中は止めない。全 run を残すなら `queue: max`） |
| production デプロイ | `false`（中断しない） |
| preview deploy | `true`（最新のみ） |
| release ワークフロー（タグ作成） | `false` |

**出典**:
- [GitHub: Workflow concurrency](https://docs.github.com/en/actions/using-jobs/using-concurrency) (GitHub Docs)
- [Control the concurrency of workflows and jobs](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency) (GitHub Docs、pending は既定で 1 件・`queue: max` は最大 100 件・`cancel-in-progress: true` と併用不可) ※2026-10-25 確認

**バージョン**: GitHub Actions
**確信度**: 高
**最終更新**: 2026-10-25

---

### 3. `actions/cache` で `node_modules` と Next.js キャッシュを永続化する

`pnpm` の store・Next.js の `.next/cache` を CI run 間で再利用し、ビルド時間を 30-70% 短縮する。
`actions/setup-node` の `cache:` オプションだけでは Next.js キャッシュが取れない。

**根拠**:
- `pnpm install --frozen-lockfile` も 30-60 秒かかる。キャッシュで 5 秒に短縮
- Next.js は `.next/cache` に webpack / babel / SWC のキャッシュを保存。これがあると build 時間が半分以下
- `actions/setup-node` の `cache:`（pnpm 11 で `pnpm/setup` を使う場合はその `cache: true`）は依存パッケージのみ。Next.js キャッシュは別途設定
- Turborepo / Nx を使う場合は Turbo / Nx のリモートキャッシュも併用（[build-cache.md](./build-cache.md) 参照）

**設定例**（依存は setup 系 action のキャッシュ、それ以外は `actions/cache` で保存する）:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with: { persist-credentials: false }

      - uses: actions/setup-node@v7
        with: { node-version: 24 }
      - uses: pnpm/setup@v3                # 実運用では SHA でピン留めする（Rule #9）
        with:
          node-version-file: false
          cache: true                      # ← pnpm store のキャッシュ
          install: false

      # Next.js の .next/cache は build-cache.md #2 の actions/cache ステップをここに置く
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
```

Next.js キャッシュの `path` / `key` / `restore-keys` は `practices/ci-cd/build-cache.md` #2、キャッシュキーの設計（lockfile ハッシュ + ソースハッシュ、`hashFiles` のパターンの書き方）は同 #3、TypeScript / ESLint のインクリメンタルキャッシュは同 #4 を参照。

**Playwright ブラウザのキャッシュ**:
```yaml
- uses: actions/cache@v6
  id: playwright-cache
  with:
    path: ~/.cache/ms-playwright
    key: ${{ runner.os }}-playwright-${{ hashFiles('**/pnpm-lock.yaml') }}

- if: steps.playwright-cache.outputs.cache-hit != 'true'
  run: pnpm exec playwright install --with-deps
```

**キャッシュサイズの上限**:
- GitHub Actions: リポジトリあたり 10GB（古いキャッシュは LRU で削除）
- 大きすぎるキャッシュ（Playwright 等）は run 別に切り分ける

**出典**:
- [GitHub: Caching dependencies](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows) (GitHub Docs)
- [Next.js Docs: CI build caching](https://nextjs.org/docs/app/building-your-application/deploying/ci-build-caching) (Next.js 公式)

**バージョン**: actions/cache v6 / actions/setup-node v7
**確信度**: 高
**最終更新**: 2026-10-25

---

### 4. Matrix で Node.js バージョン・OS を限定的に実行する

`matrix:` でテスト対象 Node.js バージョン・OS を並列実行する。
ただし「すべての OS × すべての Node バージョン」は CI 時間を浪費するため、PR では最小構成・main / nightly で全構成にする。

**根拠**:
- ライブラリはサポート期間内の複数 Node.js バージョン（例: 22 / 24）でテストすべき
- アプリは LTS 1 バージョンに固定するのが現実的
- Windows / Linux でテストすると path 区切り・改行コード差で発見できるバグがある
- 並列ジョブ数は GitHub Actions の billing に影響（private リポジトリの標準 runner は 1 分あたり Linux 2-core $0.006 / Windows $0.010 / macOS $0.062。macOS は Linux の約 10 倍）
- `matrix.include` に `condition:` のようなキーを書いても、matrix の変数が 1 つ増えるだけで条件にはならない。どこからも参照されないため、追加した OS・バージョンのジョブが全 PR で静かに実行される。イベントで構成を変えるときは `fromJSON` で matrix 自体を切り替える

**Matrix 設定**:
```yaml
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        # Bad: include の要素に condition: ${{ github.ref == 'refs/heads/main' }} を書く（条件として評価されない）
        # Good: イベントに応じて matrix の値そのものを切り替える
        # PR では Linux のみ、main への push と nightly（schedule）では macOS / Windows を追加する
        os: ${{ fromJSON(github.event_name == 'pull_request' && '["ubuntu-latest"]' || '["ubuntu-latest","macos-latest","windows-latest"]') }}
        # nightly だけ Node 22 も検証する
        node: ${{ fromJSON(github.event_name == 'schedule' && '[22, 24]' || '[24]') }}

    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v7
        with: { persist-credentials: false }
      - uses: actions/setup-node@v7
        with: { node-version: '${{ matrix.node }}' }
      - uses: pnpm/setup@v3                # 実運用では SHA でピン留めする（Rule #9）
        with: { node-version-file: false, cache: true, install: false }
      - run: pnpm install --frozen-lockfile
      - run: pnpm test
```

テストスイートの並列分割（シャーディング）による高速化は Rule #10 を参照。

**コスト計算（GitHub 公式の runner 単価、private リポジトリ）**:
- Linux 2-core: $0.006 / 分
- Windows 2-core: $0.010 / 分（Linux の約 1.7 倍）
- macOS 3/4-core: $0.062 / 分（Linux の約 10 倍）
- 公開リポジトリの標準 runner は無料。課金はジョブごとに 1 分未満を切り上げる（Rule #12）
- `pnpm/setup@v3` は pnpm v11 の Intel macOS（`darwin-x64`）向けバイナリがないため、Intel の macOS runner では pnpm v12 以降が必要

「全 OS でテスト」は明確な理由がある時だけ。フロントエンドアプリは通常 Linux のみで十分。

**Self-hosted runner**:
- 大規模リポジトリ・特殊環境では self-hosted runner を使う
- セキュリティ上の理由から、public リポジトリでは推奨しない（OSS 攻撃の経路になる）
- 設定: `runs-on: self-hosted` または `runs-on: [self-hosted, linux, x64]`

**出典**:
- [GitHub: Using a matrix](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs) (GitHub Docs)
- [GitHub Actions: Billing](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions) (GitHub Docs)
- [Actions runner pricing](https://docs.github.com/en/billing/reference/actions-runner-pricing) (GitHub Docs、runner 種別ごとの 1 分あたり単価とジョブ単位の切り上げ) ※2026-10-25 確認

**バージョン**: GitHub Actions
**確信度**: 高
**最終更新**: 2026-10-25

---

### 5. PR コメントで成果物（カバレッジ・bundle size）を可視化する

CI 結果（テストカバレッジ・bundle size 差分・Lighthouse スコア）を PR コメントに自動投稿する。
ダッシュボードを見に行く必要がない UX で運用する。

**根拠**:
- PR レビュー時に「数値が改善 / 悪化したか」を即座に把握できる
- 別ツールに飛ばずコンテキストスイッチを最小化
- 数値の経緯がコメント履歴として残り、後から「なぜここで悪化したか」を追える
- bundle size の悪化検出は performance regression を未然に防ぐ最も効果的な防衛策

PR にコメントするジョブには `permissions: pull-requests: write`（checkout するなら `contents: read` も）を明記する。Rule #6 のとおりトップレベルを `permissions: {}` や read-only にしていると、書かない限り 403 になる。以下はステップの抜粋で、サードパーティ action は実運用では SHA でピン留めする（Rule #9）。

**Bundle Size Analyzer**:
- `vio/bundle-stats-action` は最終リリースが 2022 年の v1.3.4 で、`runs.using: node16` のまま（Rule #15）。入力も `webpack-stats-path`（webpack の stats ファイル）を前提とするため、採用する場合は保守状況と入力を action.yml で確認する。ここでは次の自前投稿を基本にする

**自前で bundle size を投稿**:
```yaml
# .github/workflows/bundle-size.yml（ジョブに permissions: { contents: read, pull-requests: write }）
- name: Build
  run: pnpm build

- name: Calculate bundle size
  id: size
  run: |
    SIZE=$(du -sk .next/static/chunks/ | cut -f1)
    echo "size_kb=$SIZE" >> $GITHUB_OUTPUT

- name: Comment PR
  uses: actions/github-script@v9
  with:
    script: |
      const size = ${{ steps.size.outputs.size_kb }};
      const body = `📦 Bundle size: **${size} KB** (chunks)`;
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body
      });
```

**Lighthouse CI**:
```yaml
- name: Lighthouse CI
  uses: treosh/lighthouse-ci-action@v12
  with:
    urls: |
      ${{ steps.deploy.outputs.preview-url }}
    uploadArtifacts: true
    temporaryPublicStorage: true
    runs: 3
    configPath: ./.lighthouserc.js
```

```js
// .lighthouserc.js
module.exports = {
  ci: {
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
        'categories:accessibility': ['error', { minScore: 0.95 }],
        'categories:best-practices': ['warn', { minScore: 0.9 }],
        'categories:seo': ['warn', { minScore: 0.9 }],
      },
    },
  },
};
```

**Test Coverage**（Vitest の場合）:
```yaml
- name: Test with coverage
  # Bad: --reporter=json-summary はテスト reporter の指定になり、カバレッジの出力形式は変わらない
  # Good: カバレッジの形式は --coverage.reporter で指定する（json-summary と、ファイル別表示用の json）
  run: pnpm vitest run --coverage --coverage.reporter=json-summary --coverage.reporter=json

- name: Coverage report
  if: always()
  # Jest 用の ArtiomTr/jest-coverage-report-action ではなく、Vitest の出力を読む action を使う
  uses: davelosert/vitest-coverage-report-action@v2
  with:
    json-summary-path: ./coverage/coverage-summary.json
    json-final-path: ./coverage/coverage-final.json
```

**よくある PR コメント追加項目**:
- Bundle size（chunks / pages / static）
- Test coverage（line / branch / statement）
- Lighthouse スコア（performance / a11y）
- TypeScript エラー数
- ESLint warning 数
- Visual regression（Chromatic / Percy のリンク）
- Preview URL

**コメント更新（再 push 時）**:
```yaml
- name: Comment PR
  uses: marocchino/sticky-pull-request-comment@v3
  with:
    header: bundle-size                # 同じ header で更新（コメント増殖を防止）
    message: |
      📦 **Bundle size**: 1.2 MB (+15 KB)
      ✅ **Tests**: 1,234 passed
      📊 **Coverage**: 85.3%
```

**出典**:
- [Lighthouse CI Action](https://github.com/treosh/lighthouse-ci-action) (treosh)
- [Bundle Stats Action](https://github.com/vio/bundle-stats-action) (vio、action.yml で入力と `node16` ランタイムを確認) ※2026-10-25 確認
- [sticky-pull-request-comment](https://github.com/marocchino/sticky-pull-request-comment) (marocchino)
- [vitest-coverage-report-action](https://github.com/davelosert/vitest-coverage-report-action) (davelosert、Vitest の `json-summary` / `json` を読む) ※2026-10-25 確認

**バージョン**: GitHub Actions
**確信度**: 高
**最終更新**: 2026-10-25

---

### 6. `pull_request_target` + フォーク checkout の組み合わせを避け、二分割パターンで安全に PR を処理する

外部フォークからの PR を `pull_request_target` + フォーク checkout で処理すると、信頼できないコードが secrets にアクセスできる状態で実行されてしまう。
Cache Poisoning を利用して本番環境を汚染される（「PR を出しただけで汚染」）リスクがある。
解決策は「信頼できないコードの実行」と「secrets を使う処理」をワークフローレベルで分離すること。

**根拠**:
- `pull_request_target` はデフォルトブランチのコンテキストで実行されるが、`actions/checkout` でフォーク側のコードを取得するとその制限が無効になる
- CI cache はワークフロー間で共有されるため、悪意あるコードがキャッシュを汚染→後続ビルドに混入
- npm supply chain 攻撃解説（Zenn 2026-05-15）でも「GitHub Actions の `pull_request_target` を避けるべき」と明記
- OWASP GitHub Actions Security Cheat Sheet でも同パターンを危険と指摘
- Grafana Labs の実インシデント（2026-05）では単一のアクセストークン奪取からリポジトリ全体が流出した。攻撃者が狙うのは「コード本体」ではなく「コードに触れる権限を持ったトークン」であり、`GITHUB_TOKEN` の過剰な権限付与・長期間有効な PAT・サードパーティ Action の未ピン留めが複合的な侵害経路になる

**危険なパターン（避けること）**:
```yaml
# ❌ BAD: pull_request_target + フォーク checkout の組み合わせ
on:
  pull_request_target:

jobs:
  comment:
    steps:
      - uses: actions/checkout@v7
        with:
          ref: ${{ github.event.pull_request.head.sha }}  # ← フォーク側コードを取得
          # checkout v7 以降はこの組み合わせを既定で拒否する。allow-unsafe-pr-checkout: true で解除できるが、解除しない
      - run: pnpm build  # ← 信頼できないコードを secrets のある環境で実行 ← 危険
      - uses: actions/github-script@v9
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

**安全な二分割パターン（推奨）**:
```yaml
# ワークフロー 1: 信頼できないコードを secrets なし環境でビルド
name: Build (Untrusted)
on:
  pull_request:   # ← pull_request_target ではなく pull_request を使う

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with: { persist-credentials: false }
      - uses: ./.github/actions/setup   # Rule #1 の Composite Action（pnpm + Node + install）
      - run: pnpm build
      - uses: actions/upload-artifact@v7
        with:
          name: build-result
          path: ./dist

---
# ワークフロー 2: ビルド結果を使った secrets 処理は別ワークフローで
name: Comment (Trusted)
on:
  workflow_run:
    workflows: ["Build (Untrusted)"]
    types: [completed]

permissions: {}

jobs:
  comment:
    if: github.event.workflow_run.conclusion == 'success'
    runs-on: ubuntu-latest
    permissions:
      actions: read          # 別 run の artifact を取得するのに必要
      pull-requests: write   # PR へのコメント
    steps:
      - uses: actions/download-artifact@v8
        with:
          name: build-result
          run-id: ${{ github.event.workflow_run.id }}
          github-token: ${{ secrets.GITHUB_TOKEN }}  # 別 run から取得するときは必須（省略すると現在の run を探す）
      - uses: actions/github-script@v9
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}  # ← secrets はここだけで使う
          script: |
            // PR にコメントを投稿する処理（artifact の中身は信頼できない入力として扱い、実行しない）
```

**判断軸（いつ二分割が必要か）**:
| ケース | 対処 |
|---|---|
| 外部フォークの PR で secrets を使いたい | 二分割パターン必須 |
| 自分のブランチからの PR のみ | `pull_request` トリガーで OK |
| Bot PR (Renovate/Dependabot) | secrets 露出しないよう `pull_request` のまま |
| ラベル起動（trusted PR のみ） | `pull_request_target` + `if: github.event.pull_request.user.login == 'dependabot[bot]'` で制限可 |

**Dependabot PR に Claude Code Action で AI レビューを行う場合の安全パターン**:

`pull_request_target` + PR 作成者（`github.event.pull_request.user.login`）が `dependabot[bot]` という条件で、外部コードの任意実行なしに Dependabot PR へのコメント権限を取得できる。`github.actor` は「その run を起こした人」で、PR 作成者とは限らない（人が再実行すると変わる）ため、作成者で判定する。AI エージェントには `claude_args` の `--allowedTools` で権限を最小化し、任意シェルコマンド実行を防ぐ。

```yaml
# .github/workflows/dependabot-review.yml
on:
  pull_request_target:
    types: [opened]

permissions: {}   # デフォルトで全権限を拒否

jobs:
  review:
    # Dependabot が作成した PR のみに限定
    if: github.event.pull_request.user.login == 'dependabot[bot]'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      id-token: write        # claude-code-action が GitHub App のトークンを取得するのに使う
    steps:
      # pull_request_target の既定はベースブランチの checkout（PR 側のコードは取得しない）
      - uses: actions/checkout@v7
        with: { persist-credentials: false }
      - uses: anthropics/claude-code-action@v1   # 実運用では SHA でピン留めする（Rule #9）
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          # Bot が起こしたイベントは既定で拒否されるため、Dependabot だけ明示的に許可する
          allowed_bots: 'dependabot[bot]'
          prompt: |
            REPO: ${{ github.repository }}
            PR NUMBER: ${{ github.event.pull_request.number }}
            この依存更新 PR の差分を読み、破壊的変更の有無と確認すべき点を gh pr comment で報告してください。
          # v1 に allowed_tools 入力はない。ツールの制限は claude_args の --allowedTools で渡す
          # WebFetch は許可しない（リリースノート等に仕込まれたプロンプトインジェクションの経路になる）
          claude_args: |
            --allowedTools "Bash(gh pr comment:*),Bash(gh pr diff:*),Bash(gh pr view:*),Read,Grep,Glob"
```

キーポイント:
- `permissions: {}` をワークフロー冒頭に置き、すべての権限をデフォルト拒否にする
- `github.event.pull_request.user.login == 'dependabot[bot]'` で Dependabot 以外が作成した PR を排除し、`allowed_bots` で Dependabot だけを許可する
- `--allowedTools` の `:` 区切りサブコマンド制限で、任意 `bash` 実行を防ぐ（`Bash` だけ許可すると全シェルコマンドが実行可能になる）
- `pull_request_target` のジョブは secrets を持つ。PR 本文や依存の changelog は外部から書き込めるため、外部へ通信するツール（`WebFetch` 等）を渡さず、出力先も PR コメントに限る
- PR 側のコードを checkout して Claude を動かす構成にはしない。非対話の Claude Code（`claude -p`。claude-code-action も内部で非対話実行する）は、リポジトリの `.claude/settings.json` の hooks や `.mcp.json` のサーバーを信頼確認なしで実行するため、PR に仕込まれたそれらが secrets を持つジョブでの任意コード実行の経路になる（`practices/ai-agent/claude-code.md` #11・#13）
- **`schedule` トリガーで毎朝 Claude Code を自律実行させる場合も同じ最小権限原則が要る**: `prompt` に「PRにコメントして」と書くだけでは投稿権限は付与されない。`--allowedTools` で `Bash(gh pr comment:*)` 等を明示しないと、実行はされてもコメント投稿だけ失敗する。また `ANTHROPIC_API_KEY` を secrets に残したまま OAuth（サブスクリプション）トークンに切り替えると、意図せず両方の課金経路が併存するため、切り替え時は旧 secrets を削除する

**出典（cron/自律実行での追加知見）**:
- [GitHub ActionsにClaude Codeを組み込んで、PRに自動コードレビューを付ける](https://zenn.dev/virtualcraft/articles/claude-code-github-actions-review) (Zenn jmurayama、`prompt` だけでは投稿権限が付かず `--allowedTools` の明示が必須) ※2026-07-08に実際にfetch成功
- [APIキーなしでClaude CodeをGitHub Actionsで動かして、毎朝勝手に働かせてみた](https://qiita.com/ktdatascience/items/40d86c446779975615c1) (Qiita ktdatascience、OAuthトークンでの cron 自動化と `ANTHROPIC_API_KEY` 残置による二重課金の罠) ※2026-07-08に実際にfetch成功

**チェックリスト**:
- [ ] `pull_request_target` を使っている場合、フォーク checkout をしていないか確認
- [ ] secrets を使う処理は `workflow_run` トリガーで artifact 経由にしているか
- [ ] Renovate/Dependabot の PR ワークフローは `pull_request` トリガーか
- [ ] `GITHUB_TOKEN` のデフォルト権限をリポジトリ設定で `read-only` に制限しているか
- [ ] ワークフローのトップレベルに `permissions: {}` を置き、ジョブ単位で必要な権限のみ付与しているか
- [ ] AI エージェント（Claude Code Action 等）に `--allowedTools` でツールを制限しているか
- [ ] サードパーティ Action は `uses: some-org/some-action@v1` のようなタグではなくコミット SHA にピン留めしているか（タグは改ざん可能）
- [ ] 長期間有効な PAT を secrets に保存していない場合、GitHub Apps または OIDC に置き換えているか

**出典引用**:
> "「PRを出しただけ」で本番環境が汚染される——GitHub Actions Cache Poisoning攻撃を理解する"
> ([GitHub Actions Cache Poisoning攻撃を理解する](https://zenn.dev/singularity/articles/2026-05-13-github-actions-cache-poisoning), セクション "Cache Poisoning の仕組み") ※2026-05-16に実際にfetch成功

> "攻撃者が狙うのが『コード本体』ではなく『コードに触れる権限を持ったトークン』"
> ([たった1つのトークンだけでコードベースが丸ごと盗まれる - Grafana流出に学ぶGitHub Actionsのサプライチェーン防御](https://zenn.dev/okamyuji/articles/grafana-github-actions-token-supply-chain), セクション "Grafana Labs インシデントの教訓") ※2026-05-20に実際にfetch成功

> "シークレットへのアクセスと read/write トークンが使えます" / "複数のセーフガードでマルウェア注入を防ぐ"
> ([DependabotのPRをClaude Codeに自動レビューさせるGitHub Actions](https://zenn.dev/mandenaren/articles/dependabot_auto_review), セクション "セキュリティ上の考慮") ※2026-05-29に実際にfetch成功

**出典**:
- [GitHub Actions Cache Poisoning攻撃を理解する](https://zenn.dev/singularity/articles/2026-05-13-github-actions-cache-poisoning) (Zenn) ※2026-05-16 fetch
- [たった1つのトークンだけでコードベースが丸ごと盗まれる](https://zenn.dev/okamyuji/articles/grafana-github-actions-token-supply-chain) (Zenn) ※2026-05-20 fetch
- [DependabotのPRをClaude Codeに自動レビューさせるGitHub Actions](https://zenn.dev/mandenaren/articles/dependabot_auto_review) (Zenn、`permissions: {}` デフォルト拒否と `--allowedTools` 制限の実装例) ※2026-05-29に実際にfetch成功
- [GitHub ActionsにClaude Codeを組み込んで、PRに自動コードレビューを付ける](https://zenn.dev/virtualcraft/articles/claude-code-github-actions-review) (Zenn jmurayama) ※2026-07-08に実際にfetch成功
- [APIキーなしでClaude CodeをGitHub Actionsで動かして、毎朝勝手に働かせてみた](https://qiita.com/ktdatascience/items/40d86c446779975615c1) (Qiita ktdatascience) ※2026-07-08に実際にfetch成功
- [anthropics/claude-code-action action.yml（v1）](https://github.com/anthropics/claude-code-action/blob/v1/action.yml) (Anthropic 公式、`prompt` / `claude_args` / `allowed_bots` の入力。`allowed_tools` は存在しない) ※2026-10-25 確認
- [actions/download-artifact action.yml（v8）](https://github.com/actions/download-artifact/blob/v8/action.yml) (GitHub 公式、別 run・別リポジトリから取得するときは `github-token` が必須) ※2026-10-25 確認
- [actions/checkout README（v7）](https://github.com/actions/checkout/blob/v7/README.md) (GitHub 公式、`pull_request_target` / `workflow_run` でのフォーク PR checkout を既定で拒否) ※2026-10-25 確認

**バージョン**: GitHub Actions / actions/checkout v7 / anthropics/claude-code-action v1
**確信度**: 高
**最終更新**: 2026-10-25

---

### 7. GitHub Actions から AWS への認証は OIDC（短命トークン）に移行する

`secrets` に長期間有効な AWS アクセスキーを保存する方式を廃止し、
GitHub Actions の OIDC（OpenID Connect）で AWS に直接フェデレーションして一時認証トークンを発行する。
「保存している」という事実がリスクになる長期クレデンシャルを根本から排除する設計。

**根拠**:
- AWS アクセスキーを `secrets` に保存すると、漏洩時に長期間にわたって不正利用されるリスクがある。有効期限がないため「保存されている」だけでリスクが生じる
- OIDC では CI 実行のたびに短命トークン（15分〜1時間）を発行。漏洩しても即失効するため構造的リスクが排除できる
- `permissions.id-token: write` を workflow に追加し、`aws-actions/configure-aws-credentials` の `role-to-assume` を設定するだけで移行できる
- AWS IAM ロールの信頼ポリシーで GitHub リポジトリ・ブランチを絞れるため、最小権限原則と相性がよい
- 同じ「鍵素材を外に出さない」原則は **GitHub App 秘密鍵**にも適用できる: 秘密鍵を Cloud KMS に保存し、署名処理のみ KMS API 経由で行う（鍵素材は KMS から外に出ない）
- 同じ OIDC 短命トークンの原則は **GCP でも同様**: Workload Identity Federation（WIF）で GCP サービスアカウント JSON キーを廃止し、GitHub OIDC トークンと WIF プールを紐付けて一時トークンを発行する。属性条件（`attribute_condition`）でリポジトリ・ブランチ・タグを絞り込むことで AWS の `sub` 条件絞り込みと同等の最小権限を実現できる
- 同じ原則は **Docker Hub でも同様**: OIDC federation を使うと長期有効な Docker アクセストークンをリポジトリ・環境シークレット・Actions キャッシュのどこにも保持せず、短命トークンで `docker/login-action` を認証できる
- 同じ原則は **Vercel Container Registry（VCR）でも同様**: `vercel/vcr-action/login@v1` が workflow の OIDC トークンを短命の Vercel アクセストークンに交換して `vcr.vercel.com` へログインし、ジョブ終了時にトークンを自動失効させる。`permissions.id-token: write` とチーム ID の変数化のみで導入でき、Vercel 側に長期クレデンシャルを保持しない
- OIDC プロバイダー自体の作成も Terraform でコード化しておくと、複数リポジトリ・複数ロールへの展開時に手作業でのポリシー設定ミスを防げる
- **最も多い失敗は信頼ポリシーの `sub` 条件を絞り込まないこと**: 組織内の任意のワークフロー（フォークされた marketplace action を含む）が role を assume できてしまう。`sub` は `repo:<ORG>/<REPO>:ref:refs/heads/main` のように **リポジトリ＋ブランチ／environment 単位で厳密に**指定し、別ワークフローに権限を広げたい場合は既存ロールを緩めず**別の狭いロールを追加**する
- 旧クレデンシャルの削除は「新しい経路の疎通確認 → 読み取り専用コマンドで検証 → 本番影響のない操作でテスト → 監査ログ確認」の順で確実に検証してから行う。短命トークンであっても、実行中に奪われた場合の影響範囲は role の権限そのものに比例するため、IAM ロールの権限最小化（`sts:GetCallerIdentity` 相当から段階的に拡張）は OIDC 移行後も引き続き必要
- 同じ OIDC 短命トークンの原則は **Azure（Microsoft Entra ID）でも同様**: フェデレーション資格情報（Federated Credentials）を使うと、GitHub Actions が発行する OIDC トークンを Entra ID が検証し、Azure 向けの短命 OAuth 2.0 アクセストークンに交換する。クライアントシークレットを保存する方式と異なり、長期秘密の保管自体が不要になる。ただし OIDC トークンと Azure のアクセストークンは別物であり、「誰であるか」を確認する認証（フェデレーション資格情報の責務）と「何ができるか」を決める認可（Azure RBAC の責務）は別レイヤーである点に注意する
- **OIDC 導入後も `sub` クレーム自体の形式変更を追う必要がある**: GitHub は 2026-07-15 以降に作成・rename されたリポジトリで `sub` のデフォルト形式を「オーナー名/リポジトリ名（可変）」から「オーナーID/リポジトリID を付加した immutable subject claim」に変更した（例: `repo:octocat/my-repo:ref:refs/heads/main` → `repo:octocat@123456/my-repo@456789:ref:refs/heads/main`）。目的はリポジトリ名の再利用（削除→別オーナーが同名で再作成）によるなりすましを防ぐこと。信頼ポリシーの `sub` 条件を名前ベースで書いている既存環境は、移行時に**新形式を先に追加し旧形式を残したまま切り替え、最後に旧形式を削除する**順序を守らないと、切り替え中に正当なワークフローの認証が失敗する
- **EC2 への運用アクセスにも同じ「長期クレデンシャルを持たない」原則を延長できる**: OIDC で発行した短命トークンで IAM ロールを assume し、そのロールの権限で `aws ssm send-command` を実行すれば、SSH ポート開放・EC2 固定 IP・GitHub Secrets への長期 SSH 秘密鍵保存のいずれも不要になる。SSM Agent 経由の内部通信のみで完結するため、インバウンドのネットワーク経路自体が不要になる点が OIDC 単体（AssumeRole のみ）との違い
- **フェデレーションは「侵害されたジョブ」を信頼できるものにしない**: OWASP は、信頼ポリシー（どの外部ワークロードが資格情報を取得できるか）と権限ポリシー（その資格情報で何ができるか）を別々に設計するよう求めている。`iss` / `aud` / `sub` を厳密に絞っても、デプロイ用ロールが管理者権限なら補えない。`id-token: write` は**トークン要求を許可するだけ**で、クラウド権限そのものは付与しないため、クラウドアクセスが必要なジョブ単位で付ける。また発行されたクラウド資格情報はジョブや OIDC トークンの終了とは別の有効期限を持つ。信頼できない Pull Request のコードは、本番アクセスを取得できるジョブから切り離す
- **OIDC 未対応のデプロイ先では「長期トークン + 保護 environment」で露出面を狭める**: Vercel は Marketplace ベンダーに長期 API キーから OIDC・短命トークンへの移行を求めているが、公式の GitHub Actions デプロイガイドは依然 `VERCEL_TOKEN` シークレットを要求する。OIDC が使えるまでは、トークンに有効期限を設定し、required reviewers 付きの `environment` に置き、fork 起点のジョブから隔離し、定期的にローテーションする。OIDC は「誰がシークレットを読めるか」ではなく「どの workflow がポリシーに一致して動けるか」に信頼判断を移すため、信頼ポリシーが緩いと静的シークレットと同じ露出になる

**コード例**:
```yaml
# Bad: 長期間有効なクレデンシャルを Secrets に保存
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v6   # 実運用では SHA でピン留めする（Rule #9）
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-1

# Good: OIDC で短命トークンを都度発行（アクセスキーは不要）
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write    # OIDC トークン発行に必要
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v6   # 実運用では SHA でピン留めする（Rule #9）
        with:
          role-to-assume: ${{ vars.OIDC_ROLE_ARN }}
          aws-region: ap-northeast-1
```

**AWS IAM ロール信頼ポリシー（最小権限設定。`Statement` 配列の要素 1 つ分）**:

`aud` を `sts.amazonaws.com` に固定し、`sub` はリポジトリの形式に合わせる。2026-07-15 より後に作成された、または immutable subject claim に opt-in したリポジトリでは `sub` にオーナー ID / リポジトリ ID が入る（GitHub Enterprise Server は対象外）。移行中は新旧の形式を両方列挙し（値の配列は OR で評価される）、切り替えを確認したら旧形式を消す。

```json
{
  "Effect": "Allow",
  "Principal": { "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com" },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
      "token.actions.githubusercontent.com:sub": [
        "repo:my-org/my-repo:ref:refs/heads/main",
        "repo:my-org@123456/my-repo@456789:ref:refs/heads/main"
      ]
    }
  }
}
```

**GCP Workload Identity Federation プールの作成（`gcloud` CLI）**:
```bash
gcloud iam workload-identity-pools create github-pool \
  --location=global \
  --display-name="GitHub Actions Pool" \
  --project="<プロジェクトID>"

gcloud iam workload-identity-pools providers create-oidc github-provider \
  --location=global \
  --workload-identity-pool=github-pool \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository=='<ORG>/<REPO>'"
```

**Terraform で OIDC プロバイダー + IAM ロールをコード化する**:
```hcl
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

resource "aws_iam_role" "github_actions_deploy" {
  name = "github-actions-deploy"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Federated = aws_iam_openid_connect_provider.github.arn }
      Action    = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
          # 旧形式（名前ベース）と immutable subject（ID 付き）の両方。移行が済んだら旧形式を消す
          "token.actions.githubusercontent.com:sub" = [
            "repo:my-org/my-repo:ref:refs/heads/main",
            "repo:my-org@123456/my-repo@456789:ref:refs/heads/main",
          ]
        }
      }
    }]
  })
}
```

**移行チェックリスト**:
- [ ] workflow に `permissions.id-token: write` を追加
- [ ] `role-to-assume` に IAM ロール ARN（`vars.OIDC_ROLE_ARN` 等）を設定
- [ ] `aws-access-key-id` / `aws-secret-access-key` の指定を削除
- [ ] AWS 側で GitHub OIDC プロバイダーを作成（アカウントに1回）
- [ ] IAM ロールの信頼ポリシーで `aud` を `sts.amazonaws.com` に固定し、`sub` 条件をリポジトリ・ブランチに絞る（`*` は禁止。immutable subject の形式も確認する）
- [ ] 古い AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY を secrets から削除

**アンチパターン**:
- 信頼ポリシーの `sub` 条件を `*` にする → 全ブランチ・全フォークからの AssumeRole を許可してしまう
- 旧クレデンシャルを secrets に残したまま OIDC を追加するだけ → 移行メリットが半減する
- 複数ワークフローに対応するために最初の role の `sub` 条件をワイルドカードで緩める → 狭いロールを追加で作る方が安全

**GitHub App 秘密鍵の KMS 管理（応用パターン）**:
```javascript
// GitHub App JWT を秘密鍵なしで署名する（KMS 経由）
const digest = createHash('sha256').update(message).digest('base64');
const kmsRes = await fetch(`https://cloudkms.googleapis.com/v1/${KMS_KEY_NAME}:asymmetricSign`, {
  method: 'POST',
  headers: { Authorization: `Bearer ${gcpAccessToken}` },
  body: JSON.stringify({ digest: { sha256: digest } })
});
const { signature } = await kmsRes.json();
const jwt = `${message}.${Buffer.from(signature, 'base64').toString('base64url')}`;
// → 秘密鍵ファイルは一切手元に存在しない
```

**出典**:
- [GitHub ActionsからAWSへの認証をOIDCで行う](https://zenn.dev/hisa_tech_2973/articles/9f41f231827ec4) (Zenn) ※2026-05-21に実際にfetch成功
- [GitHub App の秘密鍵を Cloud KMS に閉じ込める](https://zenn.dev/acntechjp/articles/64c6deacee1c97) (Zenn、鍵素材を外に出さず KMS で署名する応用パターン) ※2026-05-22に実際にfetch成功
- [GitHub ActionsとWorkload Identity Federationによるサービスアカウント キーレス化の実践](https://zenn.dev/tk_nomura/articles/2026-07-wif-keyless-github-actions) (Zenn TK、GCP版OIDCキーレス化と`attribute_condition`によるリポジトリ/ブランチ/タグ絞り込み) ※2026-07-08に実際にfetch成功
- [GitHub Actions × OIDC で実現するセキュアなCI/CDパイプラインの作成](https://zenn.dev/kingdom0927/articles/fce8b036fead5f) (Zenn、OIDC プロバイダー作成込みの Terraform コード例) ※2026-07-05に実際にfetch成功
- [Auditors, OIDC and the trust policy most teams get wrong](https://dev.to/leobaniak/auditors-oidc-and-the-trust-policy-most-teams-get-wrong-4jcb) (dev.to、`sub` 条件を絞らない失敗パターン) ※2026-07-24に実際にfetch成功
- [GitHub Actionsで長期AWSキーをなくす5ステップ — OIDC移行の安全チェック付き実務レシピ](https://qiita.com/akira_papa_AI/items/9947bb45cf798b4e4d6e) (Qiita、旧クレデンシャル削除前の検証手順) ※2026-07-24に実際にfetch成功
- [GitHub Actions OIDC連携でGCPデプロイをセキュアに自動化する仕組み](https://zenn.dev/fd_ai_teacher/articles/tech-20260724151323-1) (Zenn、Workload Identity Federationプール作成の`gcloud` CLIコマンド) ※2026-07-25に実際にfetch成功
- [Immutable subject claims for GitHub Actions OIDC tokens](https://github.blog/changelog/2026-04-23-immutable-subject-claims-for-github-actions-oidc-tokens/) (GitHub Blog Changelog公式、`sub`のimmutable ID化発表) ※2026-08-09に実際にfetch成功
- [GitHub Actions OIDCのsubが変わった — 2026年7月15日以降を止めずに移行する](https://zenn.dev/kmn/articles/8e62a62ba08bde) (Zenn KMN、`gh api`での`use_immutable_subject`切替と新旧`sub`併記による無停止移行手順) ※2026-08-09に実際にfetch成功
- [GitHub ActionsからAzureへ安全に接続する 〜Entra ID・OIDC・フェデレーション資格情報を理解する〜](https://qiita.com/dev_mtech/items/aebc5be97d0151121a7d) (Qiita、Azure向けOIDCフェデレーション、認証と認可の分離) ※2026-08-22に実際にfetch成功
- [【ポート開放・固定IP不要】GitHub Actions × AWS (OIDC, SSM) × Dockerで作るセキュアなEC2自動デプロイ環境](https://qiita.com/eguchiri/items/3ec5be26ec61234e4418) (Qiita、OIDC発行の短命トークン + SSM Session ManagerでEC2への運用アクセスもポート開放なしに実現) ※2026-09-23に実際にfetch成功
- [OWASP Workload Identity Federation Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Workload_Identity_Federation_Cheat_Sheet.md) (OWASP CheatSheetSeries 公式、PR #2509 で追加。信頼ポリシーと権限ポリシーの分離、`id-token: write` の意味、侵害ジョブは信頼できない) ※2026-10-03 fetch

**出典引用**:
> "Validating a token against the configured trusted issuer does not by itself authorize production access."
> ([Workload Identity Federation Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Workload_Identity_Federation_Cheat_Sheet.md), セクション "Understand the Trust Boundary") ※2026-10-03に実際にfetch成功

> "In GitHub Actions, set `id-token: write` at the job level; it permits token requests and does not itself grant cloud permissions."
> ([Workload Identity Federation Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Workload_Identity_Federation_Cheat_Sheet.md), セクション "Limit Credential Exposure and Permissions") ※2026-10-03に実際にfetch成功

> "OIDC（OpenID Connect）を使うと、GitHub Actions が実行される際にAWSへの短期トークン（一時的なクレデンシャル）を動的に発行できます。"
> ([GitHub Actions × OIDC で実現するセキュアなCI/CDパイプラインの作成](https://zenn.dev/kingdom0927/articles/fce8b036fead5f), セクション "なぜ OIDC か？") ※2026-07-05に実際にfetch成功

> "The OIDC subject claim is the perimeter. Scope it to the exact repository, scope it to the exact ref, and when you extend to a second workflow, add a second narrower role rather than a wildcard on the first."
> ([Auditors, OIDC and the trust policy most teams get wrong](https://dev.to/leobaniak/auditors-oidc-and-the-trust-policy-most-teams-get-wrong-4jcb), セクション "The one line that most teams get wrong") ※2026-07-24に実際にfetch成功

> "短期資格情報でも、実行中に奪われた時の影響はrole権限ぶん残ります。"
> ([GitHub Actionsで長期AWSキーをなくす5ステップ — OIDC移行の安全チェック付き実務レシピ](https://qiita.com/akira_papa_AI/items/9947bb45cf798b4e4d6e), セクション "よくある失敗5つ") ※2026-07-24に実際にfetch成功

> "サービスアカウントキーが不要になることで、キーの漏洩リスクやローテーションの運用負荷がゼロになり"
> ([GitHub Actions OIDC連携でGCPデプロイをセキュアに自動化する仕組み](https://zenn.dev/fd_ai_teacher/articles/tech-20260724151323-1), セクション "OIDCとWorkload Identity Federationの概要") ※2026-07-25に実際にfetch成功

> "本手法では秘密鍵は Cloud KMS から外に出ることなく、署名処理のみを KMS API で行います。秘密鍵そのものを手元で管理する必要がなくなるため、流出リスクを大幅に低減できます。"
> ([GitHub App の秘密鍵を Cloud KMS に閉じ込める](https://zenn.dev/acntechjp/articles/64c6deacee1c97), セクション "Flow/Process") ※2026-05-22に実際にfetch成功

> "There is no long-lived Docker token in the repo, in an environment secret, or in an Actions cache."
> ([Docker Hub gets OIDC federation for GitHub Actions, retiring the PAT-in-a-secret pattern](https://dev.to/leobaniak/docker-hub-gets-oidc-federation-for-github-actions-retiring-the-pat-in-a-secret-pattern-1392), セクション "How OIDC federation eliminates stored PATs") ※2026-08-01に実際にfetch成功

> "If a repository or organization name was recycled, a new owner could mint tokens with the same subject claim, potentially gaining unauthorized access to cloud resources."
> ([Immutable subject claims for GitHub Actions OIDC tokens](https://github.blog/changelog/2026-04-23-immutable-subject-claims-for-github-actions-oidc-tokens/), セクション "What's changing") ※2026-08-09に実際にfetch成功

> "2026年7月15日より後に作られたrepositoryは新形式を自動的に使う"
> ([GitHub Actions OIDCのsubが変わった — 2026年7月15日以降を止めずに移行する](https://zenn.dev/kmn/articles/8e62a62ba08bde), セクション "何が変わったのか") ※2026-08-09に実際にfetch成功

> "OIDC トークンと Azure の Access Token は別物である"
> ([GitHub ActionsからAzureへ安全に接続する 〜Entra ID・OIDC・フェデレーション資格情報を理解する〜](https://qiita.com/dev_mtech/items/aebc5be97d0151121a7d), セクション "OIDC と OAuth 2.0") ※2026-08-22に実際にfetch成功

> "IAMロールは「一時的な権限を発行するための踏み台」であり、それ自体が能動的に何かを実行するわけではありません"
> ([【ポート開放・固定IP不要】GitHub Actions × AWS (OIDC, SSM) × Dockerで作るセキュアなEC2自動デプロイ環境](https://qiita.com/eguchiri/items/3ec5be26ec61234e4418), セクション "全体構成") ※2026-09-23に実際にfetch成功

**出典（追加）**:
- [Docker Hub gets OIDC federation for GitHub Actions, retiring the PAT-in-a-secret pattern](https://dev.to/leobaniak/docker-hub-gets-oidc-federation-for-github-actions-retiring-the-pat-in-a-secret-pattern-1392) (dev.to leobaniak、Docker Hub版OIDC federationの`permissions.id-token:write`+`docker/login-action`構成) ※2026-08-01に実際にfetch成功
- [Push images to Vercel Container Registry from GitHub Actions](https://vercel.com/changelog/vcr-login-github-action) (Vercel Changelog公式、`vercel/vcr-action/login@v1`によるVCR版OIDCログイン) ※2026-09-26に実際にfetch成功
- [Vercel wants static credentials gone. Its own GitHub Actions deploy still needs one.](https://dev.to/leobaniak/vercel-wants-static-credentials-gone-its-own-github-actions-deploy-still-needs-one-1gj6) (dev.to leobaniak、OIDC 未対応デプロイ先での暫定緩和策: 保護 environment・期限設定・fork 隔離) ※2026-10-24 fetch
- [Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws) (GitHub Docs、信頼ポリシーの `aud` / `sub` 条件と immutable subject の形式・適用条件) ※2026-10-25 確認

> "It exchanges the workflow's OIDC token for a short-lived Vercel access token, then uses that token to log in to vcr.vercel.com."
> ([Push images to Vercel Container Registry from GitHub Actions](https://vercel.com/changelog/vcr-login-github-action), セクション本文) ※2026-09-26に実際にfetch成功

**バージョン**: GitHub Actions, aws-actions/configure-aws-credentials v4+（例は v6）, Google Cloud KMS, GCP Workload Identity Federation, Docker Hub OIDC federation, immutable subject claims（2026-07-15以降の新規リポジトリでデフォルト化）, Azure Microsoft Entra ID Federated Credentials, AWS Systems Manager (SSM) Session Manager, vercel/vcr-action/login@v1
**確信度**: 高
**最終更新**: 2026-10-25

---

### 8. `actions/checkout` に `persist-credentials: false` を設定し、トークンの自動永続化を無効にする

デフォルトでは `actions/checkout` は GitHub トークンを `$RUNNER_TEMP/git-credentials-<UUID>.config` ファイルに書き込んで永続化する（v6 以降。v5 以前は `.git/config` に直接書き込む）。
後続ステップでサードパーティアクションや侵害されたスクリプトが実行された場合、このファイルからトークンを奪取できてしまう。
`persist-credentials: false` でトークン永続化を無効にし、認証が必要な場合のみ `gh auth setup-git` で都度設定する。

**根拠**:
- デフォルト (`persist-credentials: true`) では Base64 エンコードされたトークンが git 認証設定ファイルに書き込まれ、後続の全ステップからアクセス可能
- Rule #6 の pull_request_target サプライチェーンリスクと組み合わせると、外部 PR のスクリプトがトークンを奪取する攻撃経路になる
- `persist-credentials: false` を設定することでトークンが git 設定ファイルに保存されなくなる
- `gh auth setup-git` は git の credential helper として `gh` を登録するだけで、トークン自体はファイルに書かない。git が認証を求めた時点で、そのステップの環境変数 `GH_TOKEN` からトークンを読む。そのため `GH_TOKEN` は setup のステップだけでなく、`git push` するステップにも渡す
- `ghasec`・`zizmor`・`ghalint` 等の静的解析ツールで全 workflow の設定漏れを自動検出できる。`zizmor` は `persist-credentials` 漏れだけでなく、`${{ github.event.pull_request.title }}` のような信頼できない式を `run:` に直接展開してしまうテンプレートインジェクション、過剰な `permissions`、SHA 未固定のアクション参照もあわせて検出する
- `claude-code-action` はジョブ内で実行中、自分専用の一時トークンに git 認証設定を書き換える。そのため `gh auth setup-git` を先に実行していても、`claude-code-action` の後段に `git push` ステップを置くと認証設定が上書きされ、push がサイレントに失敗しうる。push 直前に remote URL をトークンを含まない形に戻し、push のステップに限って credential helper（`gh auth setup-git` + `GH_TOKEN`）で認証する。`https://x-access-token:<token>@...` を `git remote set-url` で URL に埋め込むと、トークンが `.git/config` に残り、このルールの趣旨（永続化しない）に反する
- `claude-code-action` の認証は `anthropic_api_key`（従量課金）以外に、`claude setup-token` で発行した `CLAUDE_CODE_OAUTH_TOKEN` を使う方法もある。Claude の Pro/Max サブスクリプション認証で動作し、実行ログの `total_cost_usd` が `0` になることで API キー課金が発生していないことを確認できる

**コード例**:
```yaml
# Bad: デフォルト（トークンが $RUNNER_TEMP に永続化される）
- uses: actions/checkout@v7

# Good: persist-credentials を無効化 + SHA ピニング（Rule #9: バージョンコメントを付ける）
- uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1  # v7.0.1
  with:
    persist-credentials: false

# 認証が必要な git 操作がある場合は gh auth setup-git で設定（push するジョブには permissions: contents: write）
- name: Setup git credentials
  run: gh auth setup-git  # credential helper を登録するだけで、トークンはファイルに書かない
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

- name: Push changes
  run: git push origin main
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # credential helper は push 時にこのステップの GH_TOKEN を読む

# claude-code-action の後段で push する場合は、認証設定の上書きに注意
- uses: anthropics/claude-code-action@v1   # 実運用では SHA でピン留めする（Rule #9）
  with:
    anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}

- name: Push changes after claude-code-action
  run: |
    # Bad: git remote set-url origin "https://x-access-token:${TOKEN}@github.com/..."（トークンが .git/config に残る）
    # Good: トークンを含まない URL に戻し、このステップの GH_TOKEN を credential helper 経由で使う
    git remote set-url origin "https://github.com/${GITHUB_REPOSITORY}.git"
    gh auth setup-git
    git push origin main
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**`zizmor` での自動検知例**:
```yaml
# .github/workflows/zizmor.yml
- name: Run zizmor
  uses: zizmorcore/zizmor-action@v0.8.0   # 実運用では SHA でピン留めする（Rule #9）
```
```yaml
# Bad: PR タイトルを run: に直接展開（テンプレートインジェクション、zizmor が検出）
- run: |
    echo "PR Title is: ${{ github.event.pull_request.title }}"

# Good: 環境変数経由でシェルに渡す（式展開ではなく変数参照にする）
# run: の値に ": " を含むときは、プレーンスカラーだと YAML の構文エラーになるので run: | で書く
- env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: |
    echo "PR Title is: $PR_TITLE"
```

**出典引用**:
> "the auth token is persisted in the local git config. This enables your scripts to run authenticated git commands. The token is removed during post-job cleanup."
> ([actions/checkout README](https://raw.githubusercontent.com/actions/checkout/main/README.md)) ※2026-05-25に実際にfetch成功

> "後続ステップは credential ファイルから GitHub トークンを容易に奪取できます。persist-credentials: false を設定することで、このリスクを根本から排除できます。"
> ([【GitHub Actions】actions/checkout には persist-credentials: false を設定するべき](https://zenn.dev/kou_pg_0131/articles/gha-checkout-persist-credentials), セクション "persist-credentials: false を設定するべき理由") ※2026-05-25に実際にfetch成功

> "claude-code-action は実行中、自分専用の一時トークンで git の認証設定を書き換えます"
> ([claude-code-actionで自動pushが止まる。git認証上書きの罠と回避策](https://zenn.dev/kaion/articles/claude-code-action-push-auth-pitfall), セクション "Pitfall #1") ※2026-07-11に実際にfetch成功

> "実行ログの `total_cost_usd` は `0` でした"
> ([GitHub ActionsのPR自動レビューを公式claude-code-actionで組む(APIキー課金なし)](https://qiita.com/itaraiguma/items/3a723688a2fe571c33ec), セクション サブスク課金の確認) ※2026-08-01に実際にfetch成功

> "GitHub Actionsのワークフロー定義ファイルには、テンプレートインジェクション、過剰な権限設定、未固定のアクションなど、気付かないうちに致命的なセキュリティリスクが潜んでいるケースが少なくありません。"
> ([GitHub Actions専用セキュリティスキャナ「zizmor」でワークフローの脆弱性を自動検知する](https://qiita.com/divertissement215/items/46dc02f4ae8d65291dc6), セクション "検出される代表的なリスクと修正例") ※2026-07-25に実際にfetch成功

**出典（追加）**:
- [GitHub Actions専用セキュリティスキャナ「zizmor」でワークフローの脆弱性を自動検知する](https://qiita.com/divertissement215/items/46dc02f4ae8d65291dc6) (Qiita、`zizmor` の具体的な導入手順とテンプレートインジェクション修正例) ※2026-07-25に実際にfetch成功
- [GitHub ActionsのPR自動レビューを公式claude-code-actionで組む(APIキー課金なし)](https://qiita.com/itaraiguma/items/3a723688a2fe571c33ec) (Qiita itaraiguma、`claude setup-token` による `CLAUDE_CODE_OAUTH_TOKEN` 発行とサブスク課金の実行ログ検証) ※2026-08-01に実際にfetch成功

**バージョン**: actions/checkout v4+（`$RUNNER_TEMP` への保存は v6+。例は v7）/ anthropics/claude-code-action v1 / zizmor
**確信度**: 高
**最終更新**: 2026-10-25

---

### 9. GitHub Actions のアクション参照はコミット SHA にピン留めし、バージョンコメントを必ず追記する

フローティングタグ（`@v4` 等）はリポジトリオーナーによって書き換え可能なため、サプライチェーン攻撃のベクターになる。
アクションを 40 文字の完全コミット SHA で参照し、可読性のためバージョンコメントを必ず隣接追記する。
`pinact` などのツールで既存ワークフローを一括ピン留めし、CI で SHA とコメントの整合性を自動検証する。

**根拠**:
- フローティングタグは書き換え可能。コミット SHA は不変であり、サードパーティアクションの改ざんを検知できる
- バージョンコメントがないと fork リポジトリの不審なコミットと区別できない
- SmartRound の供給チェーン監査基盤でも「アクションのコミット SHA 参照」を必須チェック項目として実装している
- `pinact v4` 以降の `--verify-comment` オプションで CI 上の SHA とバージョンコメントの整合性を自動検証できる
- SHA ピン留めで守れるのは「参照先コミットの差し替え」だけ。Action が実行時にダウンロードするバイナリ、可変タグで参照するコンテナイメージ、composite action 内部のタグ参照は対象外のため、バイナリはチェックサム検証・イメージは `@sha256:` ダイジェスト指定とし、組織設定の「Enforce SHA Pinning」ポリシーで composite 内の `uses:` までタグ参照を拒否する
- 本リポジトリの practices にあるコード例は、可読性のため `@v7` のようなタグ表記にしている。実運用では本ルールに従い SHA + バージョンコメントに置き換える（Rule #16 の lockfile に onboard 済みの workflow は、タグ表記のままでも実行時はロック済みのコミットが使われる）

**コード例**:
```yaml
# Good: 40文字 SHA + バージョンコメント（書き換え不可・可読性維持）
steps:
  - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1  # v7.0.1
    with:
      persist-credentials: false
  - uses: actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1  # v7.1.0
    with:
      node-version: 24

# Bad: フローティングタグ（タグ書き換えリスク）
  - uses: actions/checkout@v7
```

**pinact 設定例（`.pinact.yaml`）**:

pinact が自動で読む設定ファイルは `.pinact.yaml` / `.github/pinact.yaml` / `.pinact.yml` / `.github/pinact.yml` で、ルートの `pinact.yaml` は読まれない（読まれないと `min_age` が静かに効かない）。`version`（スキーマバージョン）も必須。

```yaml
# .pinact.yaml
version: 3
# 最低 7 日以上経過したコミットのみ許可（直後のマルウェア混入を防ぐ。pinact v4.0.0+）
min_age:
  value: 7
# 特定の条件下でスキップするルール（組織内アクション等）
rules:
  - ignore: true
    conditions:
      - expr: ActionRepoOwner == "your-org"
```

**出典引用**:
> "require version comments alongside commit SHAs to prevent ambiguity about fork commits"
> ([pinact v4 — GitHub Actions のバージョンピン留めツール](https://zenn.dev/shunsuke_suzuki/articles/pinact-v4-guide)) ※2026-05-26に実際にfetch成功

> "Actions pinned to commit SHAs (40-character hex) rather than version tags like @v1"
> ([サプライチェーン攻撃対策の「実効」を継続検証するGitHub監査基盤を内製した話](https://zenn.dev/smartround_dev/articles/478c195bf914b6), セクション "GitHub Actions Security") ※2026-05-26に実際にfetch成功

**出典**:
- [GitHub ActionsのSHA Pinningだけで本当に大丈夫ですか？](https://zenn.dev/nishino_hiroki/articles/d7570d3bb3408f) (Zenn nishino_hiroki、SHA Pinning の守備範囲外: 実行時取得バイナリ・可変タグのコンテナ・composite 内参照) ※2026-10-10に実際にfetch成功
- [pinact: Configuration File](https://github.com/suzuki-shunsuke/pinact/blob/main/docs/config.md) (pinact 公式、自動検出される設定ファイル名・`version: 3`・`min_age` / `rules`) ※2026-10-25 確認

> "問題は、Action が動き始めてから外から何か実行ファイルを取得するような場合です。"
> ([GitHub ActionsのSHA Pinningだけで本当に大丈夫ですか？](https://zenn.dev/nishino_hiroki/articles/d7570d3bb3408f), セクション "SHA Pinning で守れないもの") ※2026-10-10に実際にfetch成功

**バージョン**: GitHub Actions 全バージョン / pinact v4+（設定スキーマ v3）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 10. テストシャーディングと paths-filter で CI を高速化する

`strategy.matrix.shard` でテストスイートを並列分割し、`dorny/paths-filter` で変更ファイルに関連するジョブのみを条件実行することで、CI の総実行時間を大幅に短縮する。

**根拠**:
- シャーディングにより Playwright / Vitest のテスト実行時間を最大 1/N に短縮できる
- `dorny/paths-filter` により無関係のジョブをスキップし、不要なコンピュートコストを削減できる
- 両者の組み合わせで「速さ × 無駄のなさ」を両立し、開発者体験を維持しながらコストを抑えられる
- 1台の runner で `workers` を増やしても CPU / メモリの上限は変わらないため、`matrix.shard` で runner ごと分散させたほうが実効速度が上がる（総 runner-minutes は増えるが wall-clock は縮む、というトレードオフを意図的に取る）
- CI では Vite を `dev` サーバーではなく `preview` サーバーで起動すると、ランタイムのモジュール変換オーバーヘッドを避けられる
- 固定秒数の `sleep` ではなく実際のヘルスチェックで Docker / アプリの起動完了を待つと、無駄な待ち時間と不安定さの両方を減らせる
- ARM64 向け Docker イメージを x86_64 runner + QEMU エミュレーションでビルドすると、QEMU が ARM 命令を x86 命令へ逐次変換するオーバーヘッドが CPU バウンドな処理（`next build` 等）に直撃する。GitHub Actions のネイティブ ARM64 runner（`ubuntu-24.04-arm` 等）に切り替えて QEMU を撤去すると、エミュレーションのオーバーヘッドそのものがなくなる

**テストシャーディング**:
```yaml
jobs:
  e2e:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        # Bad: shard: [1/4, 2/4, ...] を artifact 名に使うと "/" が禁止文字で upload が失敗する
        shardIndex: [1, 2, 3, 4]  # 4並列で実行時間を 1/4 に
        shardTotal: [4]
    steps:
      - uses: actions/checkout@v7
        with: { persist-credentials: false }
      - uses: ./.github/actions/setup   # Rule #1 の Composite Action
      - run: pnpm exec playwright install --with-deps
      # blob reporter で出力し、後段のジョブで merge-reports する
      - run: pnpm exec playwright test --shard=${{ matrix.shardIndex }}/${{ matrix.shardTotal }} --reporter=blob
      - uses: actions/upload-artifact@v7
        if: ${{ !cancelled() }}
        with:
          name: blob-report-${{ matrix.shardIndex }}
          path: blob-report/
          retention-days: 1
```

シャードごとの blob レポートは、`needs: [e2e]` のジョブで `actions/download-artifact`（`pattern: blob-report-*`、`merge-multiple: true`）でまとめて取得し、`npx playwright merge-reports --reporter html ./all-blob-reports` で 1 つの HTML レポートにする（Playwright 公式の Sharding ページの手順）。

**paths-filter で条件実行**:
```yaml
jobs:
  changes:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: read   # pull_request では REST API で変更ファイルを取得するため必要
    outputs:
      frontend: ${{ steps.filter.outputs.frontend }}
      backend: ${{ steps.filter.outputs.backend }}
    steps:
      - uses: dorny/paths-filter@v4   # 実運用では SHA でピン留めする（Rule #9）
        id: filter
        with:
          filters: |
            frontend:
              - 'src/**'
              - 'package.json'
              - 'pnpm-lock.yaml'
            backend:
              - 'api/**'

  test-frontend:
    needs: changes
    if: ${{ needs.changes.outputs.frontend == 'true' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with: { persist-credentials: false }
      - uses: ./.github/actions/setup
      - run: pnpm test
```

**アンチパターン**:
- すべての PR で全テストを直列実行する → モノレポやテストが増えるにつれ CI が 10 分超になる
- シャード数を一度設定したまま放置する → テスト増加に伴い 1 シャードが肥大化するため、定期的に実行時間を計測して調整する

**出典引用**:
> "shardingで並列数を増やすことでPlaywrightのテスト実行時間を効果的に短縮できます"
> ([CIを高速化するテクニック集](https://zenn.dev/mandenaren/articles/ci_speedup_techniques), Zenn) ※2026-06-04に実際にfetch成功

> "一台の GitHub Actions runner で workers を増やしても、CPU とメモリの上限は変わりません。"
> ([Playwright の E2E CI を4分割し、3時間13分から25分に短縮した](https://zenn.dev/thaddeusjiang/articles/ci-e2e-runtime-optimization), セクション "workers ではなく sharding を使った理由") ※2026-08-21に実際にfetch成功

> "`next build` のステップだけで1106秒(≒ 約19分)かかっている" / "QEMUはARM命令をx86命令に逐次変換して実行するので、CPU実行そのものにオーバーヘッドが乗る"
> ([Next.jsのDockerビルド時間を約19分短縮した話](https://zenn.dev/rb_engineering/articles/b642bdb5206cb9), セクション "紐解いていく" / "なぜQEMUを使っていたのか") ※2026-09-11に実際にfetch成功

**出典**:
- [CIを高速化するテクニック集](https://zenn.dev/mandenaren/articles/ci_speedup_techniques) (Zenn) ※2026-06-04 fetch
- [Playwright: Sharding](https://playwright.dev/docs/test-sharding) (Playwright 公式、テスト並列分割。`shardIndex` / `shardTotal` の matrix・blob レポートの upload と merge-reports) ※2026-10-25 確認
- [Playwright の E2E CI を4分割し、3時間13分から25分に短縮した](https://zenn.dev/thaddeusjiang/articles/ci-e2e-runtime-optimization) (Zenn、3時間13分→25分の実測改善、Vite preview サーバー化とヘルスチェック導入を含む) ※2026-08-21 fetch
- [Next.jsのDockerビルド時間を約19分短縮した話](https://zenn.dev/rb_engineering/articles/b642bdb5206cb9) (Zenn、QEMU エミュレーションからネイティブ ARM64 runner への切り替えで `next build` を含むビルド&プッシュが24m43s→5m35sに短縮) ※2026-09-11 fetch

**バージョン**: GitHub Actions 全バージョン、dorny/paths-filter v4
**確信度**: 中(コミュニティ記事、実績多数)
**最終更新**: 2026-10-25

---

### 11. 同一ジョブ内の独立したステップは `parallel:` キーワードで並列実行し、ジョブ分割せずに待ち時間を削る

Rule #10 の test sharding / paths-filter は「ジョブを分割」して並列化するが、ジョブ分割にはランナー起動のオーバーヘッドが伴う。デプロイの複数ステップのように**同一ランナー・同一ジョブ内で完結する独立した処理**は、GitHub Actions の `background` / `wait` / `wait-all` / `cancel` / `parallel` キーワードでステップレベルの並列実行に切り替えることで、ジョブを増やさずに待ち時間を削れる。`parallel:` は複数ステップをまとめて `background` 化し、直後に `wait` を挟む糖衣構文。

**根拠**:
- GitHub 公式が 2026-06-25 にステップレベル並列実行機能（`background`/`wait`/`wait-all`/`cancel`/`parallel`）を GA した
- ジョブ分割による並列化（Rule #10）は新規ランナーの起動コストがかかるが、ステップ並列はジョブコンテキストを共有したまま同一ランナー内で完結する
- 独立した複数サービスへのデプロイのように「ジョブに分けるほどではないが直列で待つ理由もない」処理に向く
- 同一ランナーに処理を詰め込みすぎると CPU・メモリの奪い合いで逆に遅くなることが実測で確認されている。ステップ並列は「ジョブ分割 vs ステップ並列」の判断軸を増やすものであり、無条件にジョブ分割の代替にはならない

**コード例**:
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with: { persist-credentials: false }
      # AWS の認証（Rule #7 の OIDC）は省略。サードパーティ action は実運用では SHA でピン留めする（Rule #9）

      # 独立した2つのECSデプロイステップを同一ジョブ内で並列実行
      - parallel:
          - name: Deploy web service
            uses: aws-actions/amazon-ecs-deploy-task-definition@v2
            with:
              task-definition: web-task-def.json
              service: web-service
              cluster: production

          - name: Deploy worker service
            uses: aws-actions/amazon-ecs-deploy-task-definition@v2
            with:
              task-definition: worker-task-def.json
              service: worker-service
              cluster: production
```

**判断軸（ジョブ分割 vs ステップ並列）**:
- 別ランナーで独立に失敗させたい／ログを完全分離したい → ジョブ分割（Rule #10 の matrix）
- 同一ジョブのコンテキスト（checkout 済みファイル・環境変数）を共有したまま待ち時間だけ削りたい → ステップ並列（`parallel:`）
- 1 ランナーに詰め込みすぎて CPU/メモリを奪い合っていないか、実行時間を計測して確認する

**出典引用**:
> "`parallel` takes a group of steps and converts them to `background` steps with a `wait` after, enabling you to easily run multiple steps in parallel."
> ([Actions steps can now be run in parallel](https://github.blog/changelog/2026-06-25-actions-steps-can-now-be-run-in-parallel/), GitHub Changelog) ※2026-07-11に実際にfetch成功

> "直列だった頃は概ね8分前後だったものが、並列化後は3分前後になりました。"
> ([GitHub Actions の並列実行機能でデプロイを8分→3分に短縮した](https://zenn.dev/hatsu/articles/github-actions-steps-parallel), セクション "実戦①：本番ECSデプロイの並列化") ※2026-07-11に実際にfetch成功

> "同一ランナーに処理を詰め込みすぎると、CPUやメモリの奪い合いで逆に遅くなることも実測で確認できました。"
> ([GitHub Actions の並列実行機能でデプロイを8分→3分に短縮した](https://zenn.dev/hatsu/articles/github-actions-steps-parallel), セクション "job並列（needs / matrix）との使い分け") ※2026-07-11に実際にfetch成功

**出典**:
- [Actions steps can now be run in parallel](https://github.blog/changelog/2026-06-25-actions-steps-can-now-be-run-in-parallel/) (GitHub Changelog 公式、2026-06-25) ※2026-07-11 fetch
- [GitHub Actions の並列実行機能でデプロイを8分→3分に短縮した](https://zenn.dev/hatsu/articles/github-actions-steps-parallel) (Zenn、実測ベンチマークと job 並列との使い分け) ※2026-07-11 fetch

**バージョン**: GitHub Actions（2026-06-25 以降、`background`/`wait`/`wait-all`/`cancel`/`parallel` GA）
**確信度**: 高（公式 changelog + 実測ベンチマーク）
**最終更新**: 2026-10-25

---

### 12. ジョブ単位・1分未満切り上げの課金ルールを踏まえ、細切れワークフローの分割を避ける

GitHub Actions の課金は「ジョブ単位」で発生し、実行時間は**1分未満切り上げ**で計算される。数秒で終わる処理でも、別ワークフロー・別ジョブに分割していると、その都度 1 分単位で消費される。

**根拠**:
- 実処理が数秒でも、ワークフローを分割している数だけ「1分」が積み上がる（例: 4 ワークフローがそれぞれ 3 秒の curl を実行 → 実処理合計 12 秒でも課金は 1分 × 4 = 4分）
- 課金対象は private リポジトリでの GitHub ホステッド runner の利用で、公開リポジトリの標準 runner と self-hosted runner は無料。無料枠（GitHub Free は月 2,000 分）を使い切ったときの挙動は支払方法と予算の設定で変わる。支払方法が未登録なら以降の実行はブロックされ、登録済みなら超過分が課金される（予算で上限をかけられる）。使用量が 90% / 100% に達するとメールで通知されるが、見落とすと無関係なスケジュールジョブ（バックアップ・デプロイ等）も巻き添えで止まり、気づくまで時間がかかる
- 同じ処理を行うなら、ワークフロー／ジョブを分割せず 1 ジョブ内に複数ステップとしてまとめる方が、切り上げ回数を減らせる
- 組織全体で無料枠が尽きかけている場合は「実行時間を削る」前に効く順にコストを絞る: (1) `paths` フィルタとジョブのゲーティングでトリガー条件自体を絞る（最も効果が大きい）、(2) 課金単位（ジョブ単位・1分未満切り上げ）に処理の粒度を合わせて端数の無駄を減らす、(3) それでも足りない場合にキャッシュ・並列化で実処理時間を削る、という順で着手する
- Settings 側の自動実行（dependency graph の自動ジョブ等）も無料枠を消費するため、必要な場合のみワークフローから明示的に呼び出す形に倒し、常時自動発火は無効化する

**コード例（課金分の再現計算）**:
```javascript
// jobs: GET /repos/{owner}/{repo}/actions/runs/{run_id}/jobs のレスポンスの jobs 配列
// Bad: run の updated_at から run 単位で 1 回だけ切り上げる（ジョブ数ぶんの切り上げが消えて過小に見積もる）
// Good: ジョブごとに 1 分未満を切り上げてから合算する
function billableMinutes(jobs) {
  return jobs
    .filter((job) => job.started_at && job.completed_at)
    .map((job) => (Date.parse(job.completed_at) - Date.parse(job.started_at)) / 1000)
    .filter((secs) => secs > 0)
    .reduce((sum, secs) => sum + Math.ceil(secs / 60), 0);
}
```

**コード例（組織全体の消費内訳を可視化する）**:
```bash
# 直近の run 群からジョブ単位の消費分数を合算する
gh api "repos/{owner}/{repo}/actions/runs/{run_id}/jobs?per_page=100" \
  --jq '[.jobs[] | select(.started_at != null and .completed_at != null)
         | ((.completed_at|fromdateiso8601)-(.started_at|fromdateiso8601))
         | select(.>0) | (./60|ceil)] | add // 0'

# トリガーイベント別の実行回数内訳（絞り込み対象の特定に使う）
gh api "repos/{owner}/{repo}/actions/runs?created=2026-08-01..2026-08-10&per_page=100" \
  --paginate --jq '.workflow_runs[].event' | sort | uniq -c | sort -rn
```

**出典引用**:
> "4ワークフロー × 3秒 = 実処理12秒 → 課金は 1分 × 4ジョブ = 4分"
> ([GitHub Actionsの無料枠が「3秒のcurl」で溶けた話 — 課金はジョブ単位・1分未満切り上げ](https://qiita.com/my-agent-works/items/8d99600d1185938a375d), セクション "3秒のcurlが4分に化ける") ※2026-08-15に実際にfetch成功

> "課金対象はジョブごとの実行時間を分単位に切り上げた合計"
> ([GitHub Actions の無料枠が枯れた日：org 全体の分を数え直す](https://zenn.dev/propagandist/articles/0025-github-actions-org-minutes-budget), セクション "手順1") ※2026-08-18に実際にfetch成功

**出典**:
- [GitHub Actionsの無料枠が「3秒のcurl」で溶けた話 — 課金はジョブ単位・1分未満切り上げ](https://qiita.com/my-agent-works/items/8d99600d1185938a375d) (Qiita、無料枠2,000分の36%を12秒相当の処理で消費した実例と静的失敗の挙動) ※2026-08-15に実際にfetch成功
- [GitHub Actions の無料枠が枯れた日：org 全体の分を数え直す](https://zenn.dev/propagandist/articles/0025-github-actions-org-minutes-budget) (Zenn、PROPAGANDIST CORPORATION所属著者、org全体の消費内訳可視化と3段階の絞り込み戦略) ※2026-08-18 fetch
- [Actions runner pricing](https://docs.github.com/en/billing/reference/actions-runner-pricing) (GitHub Docs、「GitHub rounds the minutes and partial minutes each job uses up to the nearest whole minute.」) ※2026-10-25 確認
- [GitHub Actions billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions) (GitHub Docs、公開リポジトリは無料・支払方法の有無による超過時の挙動・90% / 100% の通知) ※2026-10-25 確認

**バージョン**: GitHub Actions（private リポジトリの GitHub ホステッド runner の課金モデル）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 13. PR ごとに axe-core で WCAG 2.2 準拠チェックを自動化し、違反を検出したらマージをブロックする

ビルド済みアプリに対して axe-core ベースの自動スキャンを PR 時に実行し、違反を検出したら CI を失敗させる。ただし自動チェックが検出できるのは a11y 課題の5〜6割程度に留まり、キーボード操作性・フォーカス順序・alt テキストの質などは引き続き手動テストが必要——「フロアであってオーディットではない」と捉える。

**根拠**:
- axe-core は Deque Systems が開発する業界標準の a11y スキャンエンジンで、WCAG 2.2 AA ルールセットに対応する
- 自動スキャンを PR 時に走らせ、CI の Job Summary / annotation に違反箇所（CSS セレクタ）を出力すると、レビュー前に機械的に検知できるものは検知できる
- 元記事で紹介されている個別の GitHub Action はメンテナ・供給元の検証ができないサードパーティ製ラッパー単独ソースであるため、本リポジトリの既存 CI/CD ルール（サードパーティ Action の SHA ピン留め等）の方針に合わせ、`@axe-core/cli` など系譜の明確な公式寄りのツールを一次候補として採用し、記事の設定パターン（重大度によるしきい値、ビルド成果物に対する実行）のみを参考にする
- 自動チェックは a11y 課題の5〜6割程度しか検出できないため、「マージをブロックする最低ライン」として位置づけ、手動テスト・スクリーンリーダー確認を代替しない
- axe のタグはバージョンごとに分かれており、`wcag22aa` が付くのは WCAG 2.2 で追加されたルール（target-size 等）だけである。WCAG 2.2 AA 全体を検査するには `wcag2a,wcag2aa,wcag21a,wcag21aa,wcag22aa` をまとめて指定する
- `@axe-core/cli` の `--exit` は違反が 1 件でもあれば非ゼロで終了する。影響度（impact）で絞ってブロックしたい場合は、結果 JSON を `--save` で保存して impact を判定するステップを別に設ける

**コード例**:
```yaml
# .github/workflows/a11y.yml
name: Accessibility Gate
on: [pull_request]

jobs:
  axe-scan:
    runs-on: ubuntu-latest
    steps:
      # 実運用では actions を SHA でピン留めする（Rule #9）
      - uses: actions/checkout@v7
        with: { persist-credentials: false }
      - uses: actions/setup-node@v7
        with: { node-version: 24 }
      - run: npm ci && npm run build
      # output: "export" の静的書き出しが前提。SSR アプリは `npm start` などで起動する
      - run: npx serve out -l 4173 &
      - run: npx wait-on http://localhost:4173 --timeout 60000
      - name: Run axe-core CLI against build output
        # Bad: --tags wcag22aa だけだと WCAG 2.2 で追加されたルールしか検査しない
        # Good: 2.0 / 2.1 / 2.2 の A・AA タグをまとめて指定する
        run: npx @axe-core/cli http://localhost:4173 --tags wcag2a,wcag2aa,wcag21a,wcag21aa,wcag22aa --exit
```

**出典引用**:
> "Automated checks catch roughly half to two-thirds of accessibility defects...Treat this action as the floor, not the audit."
> ([A zero-config accessibility gate for GitHub Actions (axe-core, WCAG 2.2, one step)](https://dev.to/syntropydigital/a-zero-config-accessibility-gate-for-github-actions-axe-core-wcag-22-one-step-29jf), セクション "What it does not do") ※2026-09-19に実際にfetch成功

**出典**:
- [A zero-config accessibility gate for GitHub Actions (axe-core, WCAG 2.2, one step)](https://dev.to/syntropydigital/a-zero-config-accessibility-gate-for-github-actions-axe-core-wcag-22-one-step-29jf) (dev.to、設定パターンのみ参考。紹介されている個別 Action は単独ソースの検証不能なサードパーティ製のため未採用) ※2026-09-19 fetch

**バージョン**: axe-core（WCAG 2.2 AA ルールセット）
**確信度**: 中（axe-core 自体は業界標準だが、記事の具体的な実装は単独ソース）
**最終更新**: 2026-10-25

---

### 14. AIエージェントによるコミット量増加でCIが律速になる場合、まずゲートジョブを軽くしてからシャーディングする

AIコーディングエージェントの普及でPR・コミットの生成速度が上がり、検証（CI）がボトルネックになるケースが増えている。対処の優先順位は「まず全体をブロックする小さなゲートジョブ（lint・typecheck・変更検知等）を軽量化し、その後にテストシャーディングへ進む」。ゲートジョブが遅いままシャード数だけ増やしても、律速要因はゲート側に残る。

**根拠**:
- 後続の並列ジョブはすべてゲートジョブの完了を待つため、ゲートの所要時間はそのままクリティカルパスに乗る。シャード数を増やしてもこの待ち時間は短くならない（例: 8 シャードが 26 秒のゲートを待つと、8 本すべての結果が 26 秒ずつ遅れる）。`needs` で待っているジョブは起動前なので runner 時間は消費しない。ゲートが増やすのは壁時計の遅延（フィードバックが届くまでの時間）であり、runner 消費が増えるのはシャードごとに重複するセットアップのほう（次項以降）
- 変更検知・チェックアウトの浅い取得（`fetch-depth: 1` やsparse/blobless checkout）でゲートジョブ自体の所要時間を削減できる
- 静的チェック（lint/typecheck/format/依存監査）は互いに依存しないが、短いチェックごとにジョブを分ける（matrix で展開するのも同じ）とチェックアウト・依存インストールがジョブ数だけ重複する。1 ジョブの中で順に実行してセットアップを 1 回にまとめる（課金面は Rule #12、同一ジョブ内の並列化は Rule #11、テストシャーディングは Rule #10 を参照）
- セットアップコスト（依存インストール・キャッシュ復元）を十分削減できてから初めてテストシャーディングの効果が最大化される。セットアップが遅いままシャード数を増やすと、シャードごとに重複するセットアップコストが積み上がる
- ネイティブ実装のコンパイラ／型チェッカーへの切り替えや、フル型グラフ構築を避けるAST解析ベースのlintルール書き換えなど、ツールチェーン自体の高速化も律速緩和に効く

**コード例（短い静的チェックを 1 ジョブにまとめる）**:
```yaml
jobs:
  # Bad: lint / typecheck / format を別ジョブや matrix に分け、セットアップをジョブ数だけ繰り返す
  # Good: 1 ジョブにまとめ、チェックアウトと依存インストールを 1 回で済ませる
  fast-checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with: { fetch-depth: 1, persist-credentials: false }
      - uses: actions/setup-node@v7
        with: { node-version: 24, cache: npm }
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm run format:check
      - run: npm run audit:deps
```

**出典引用**:
> "If eight test shards cannot start until one 26-second gate finishes, that gate is on the critical path eight times over."
> ([AI Coding Agents Made CI the Bottleneck. Here Is the Fix, Step by Step.](https://dev.to/jamilxt/ai-coding-agents-made-ci-the-bottleneck-here-is-the-fix-step-by-step-1koj), セクション "Lever 1: gate jobs, the tiny jobs that block everything") ※2026-09-22に実際にfetch成功

> "Agents have made it exponentially faster to ship code, but validating those changes hasn't quite kept up at the same rate."
> ([AI coding has made CI a bottleneck, so we reworked ours to keep up](https://linear.app/now/ci-bottleneck-reworked), セクション "Introduction") ※2026-09-22に実際にfetch成功

**出典**:
- [AI Coding Agents Made CI the Bottleneck. Here Is the Fix, Step by Step.](https://dev.to/jamilxt/ai-coding-agents-made-ci-the-bottleneck-here-is-the-fix-step-by-step-1koj) (dev.to jamilxt、計測→ゲート最適化→セットアップ削減→バッチ化→シャーディングの優先順位とYAML例) ※2026-09-22に実際にfetch成功
- [AI coding has made CI a bottleneck, so we reworked ours to keep up](https://linear.app/now/ci-bottleneck-reworked) (Linear公式ブログ、sparse/blobless checkout・ネイティブ型チェッカーへの切替・AST解析ベースのlint書き換え・7つの短いチェックを2ジョブにバッチ化した実例) ※2026-09-22に実際にfetch成功

**バージョン**: GitHub Actions（全プラン共通）
**確信度**: 高（Linear公式ブログ + dev.to、異なる著者2記事・コード例あり）
**最終更新**: 2026-10-25

---

### 15. 2026-09-23 に Node 20 が GitHub Actions から撤去済み — `node20` を宣言するアクションを棚卸しして `node24` 対応版へ更新する

GitHub は 2026-09-23 に Actions ランナーから Node 20 を完全に撤去し、JavaScript アクションは Node 24 で実行される。
`runs.using: node20` のまま更新されていないアクション（古いメジャータグ・SHA ピン留めされた旧版・サードパーティの composite アクション内部のピン留め）は動作しなくなる可能性があるため、Dependabot / Renovate の自動 PR が届かない参照を含めて棚卸しする。
GitHub 公式（first-party）アクションは Node 24 対応済みなので、最新版へ更新する。

**根拠**:
- GitHub 公式 changelog で「Node 20 は GitHub Actions で利用不可」「ランナーは JavaScript アクションに Node 24 を使う」と明言されており、一時回避策は残っていない（回避用環境変数に頼る運用は終了）
- Node 24 は macOS 13.4 以前と ARM32 を公式サポートしないため、該当するセルフホストランナーは別途対応が必要
- SHA ピン留め（本ファイル Rule #9）は不変性を保証する一方で、**旧版に固定されたまま更新が止まる**リスクを持つ。ピン留めしたら定期更新（Renovate 等）とセットで運用する
- 個人ブログの実測では、宣言が `node20` でもランナーが Node 24 を選ぶ挙動が報告されている（公式記述は「動作しなくなる」で表現が異なるため、挙動差は自環境の実行ログで確認する）

**コード例**:
```bash
# 利用中アクションの runs.using を確認する（第三者の記事で紹介された手順）
gh api "repos/actions/checkout/contents/action.yml?ref=v4" --jq .content | base64 -d | grep using:

# 組織内の node20 宣言を検索する
gh search code node20 --owner YOUR_ORG --filename action.yml
```
```yaml
# アクション自作者向け: action.yml のランタイムを更新して新リリースを出す
runs:
  using: node24   # Bad: node20
  main: dist/index.js
```

**出典引用**:
> "update its `runs.using` value to `node24` and publish a new release as soon as possible."
> ([Node 20 is no longer available in GitHub Actions](https://github.blog/changelog/2026-09-23-node-20-is-no-longer-available-in-github-actions), GitHub Changelog, セクション "Recommended migration") ※2026-10-01に実際にfetch成功

**出典**:
- [Node 20 is no longer available in GitHub Actions](https://github.blog/changelog/2026-09-23-node-20-is-no-longer-available-in-github-actions) (GitHub Changelog、公式) ※2026-10-01 fetch
- [GitHub Actions Removed Node 20. Find Every node20 Action You Still Run](https://dev.to/devopsdaily/github-actions-removed-node-20-find-every-node20-action-you-still-run-1eai) (dev.to、棚卸しコマンドと SHA ピン留め・composite 内部参照の盲点) ※2026-10-01 fetch

**バージョン**: GitHub Actions ランナー（2026-09-23 以降）
**確信度**: 高
**最終更新**: 2026-10-01

---

### 16. GitHub Actions の依存は workflow dependency lockfile（`actions.lock`）で固定し、CI で鮮度を検証する（public preview）

タグやブランチは後から動かせるため、同じ `uses:` 参照でも次回実行時に別コードを指しうる。GitHub が 2026-10-14 に public preview で公開した workflow dependency lockfile は、レビュー済みのコミットを `.github/workflows/actions.lock` に記録し、workflow ファイル側は通常の参照のまま実行時は記録済みコミットを使わせる。Rule #9 の SHA ピン留めを `uses:` ごとに手で書き換える代わりに、推移的依存（アクション内部の `uses:`）や reusable workflow まで一括で固定できる。

**根拠**:
- 公式 changelog: ロックは「依存が安全か」を判定しない。`Locking does not determine whether the dependency itself is safe, so review dependencies before locking them.` — ロック前のレビューが前提
- 第三者検証記事（dev.to）は、lockfile で固定した workflow ではインライン SHA ピン留めが不要になるとしている。ただし未 onboard の workflow は従来どおり完全長 SHA が必要で、workflow 単位で段階的に移行する（公式 changelog にこの位置づけの記載はない）
- onboard 済み workflow は全依存に lockfile エントリが必要になり、未ロックのアクションを追加すると検証が失敗する（第三者検証で `--verify` が exit 1）
- Dependabot version updates は onboard 済み workflow の lockfile エントリを同じ PR で再生成する。ロック更新が人手の作業にならない
- lockfile は「差分を読む人がいて初めてセキュリティ統制になる」。レビュー時は変更された `commit`、`owner_id` / `repo_id` の変化、新規の推移的 `uses:`、`workflows` セクションから消えた workflow を確認する
- 制約（preview 時点、第三者検証記事による）: GHES 非対応、コンテナイメージのタグは未ロック、第三者検証 v0.2.0 では job レベルの reusable workflow 呼び出しが記録されなかったため、その呼び出しは SHA ピン留めを併用する

**コード例**:
```bash
# Good: gh 拡張を導入し、workflow を 1 本ずつ onboard して lockfile を生成・コミット
gh extension install github/gh-actions-lock
# 生成物: .github/workflows/actions.lock（手編集しない）

# CI では lockfile が最新かを変更なしで検証する（未ロックの依存があると exit 1。第三者検証記事の手順で、公式 changelog には記載なし）
gh actions-lock --verify
```

```yaml
# .github/workflows/actions.lock（生成物の抜粋）
workflows:
    '.github/workflows/release.yml':
        - 'actions/checkout@v4.4.0'
dependencies:
    'actions/checkout@v4.4.0':
        ref: 'v4.4.0'
        commit: 'sha1-11d5960a326750d5838078e36cf38b85af677262'
        owner_id: ...
        repo_id: ...
```

**アンチパターン**:
- lockfile を生成しただけでレビューせずマージする（悪意ある更新を固定化しうる）
- onboard していない workflow があるのに「SHA ピン留めは不要になった」と判断して Rule #9 を外す

**出典引用**:
> "Locking does not determine whether the dependency itself is safe, so review dependencies before locking them."
> ([GitHub Actions: Workflow lockfiles in public preview](https://github.blog/changelog/2026-10-14-github-actions-workflow-lockfiles-in-public-preview/), GitHub changelog 本文) ※2026-10-24に実際にfetch成功

> "A lockfile is only a security control if someone reads the diff."
> ([What Are GitHub Actions Lockfiles, and Do They Replace SHA Pinning?](https://dev.to/alisa_hester_1/what-are-github-actions-lockfiles-and-do-they-replace-sha-pinning-3kg0), セクション "Verifying and reviewing changes") ※2026-10-24に実際にfetch成功

**出典**:
- [GitHub Actions: Workflow lockfiles in public preview](https://github.blog/changelog/2026-10-14-github-actions-workflow-lockfiles-in-public-preview/) (GitHub Changelog 公式、機能概要・有効化手順・対象プラン) ※2026-10-24に実際にfetch成功
- [What Are GitHub Actions Lockfiles, and Do They Replace SHA Pinning?](https://dev.to/alisa_hester_1/what-are-github-actions-lockfiles-and-do-they-replace-sha-pinning-3kg0) (dev.to、`--verify` 検証・レビュー観点・reusable workflow の制約を第三者検証) ※2026-10-24に実際にfetch成功

**バージョン**: GitHub Actions workflow dependency lockfiles（public preview、2026-10-14〜）/ `gh-actions-lock` 拡張（pre-1.0）
**確信度**: 中（機能の存在とロック前レビューの前提は GitHub 公式 changelog。`--verify` や制約の詳細は第三者検証記事のみ。preview 機能のため仕様変更の可能性あり）
**最終更新**: 2026-10-24
