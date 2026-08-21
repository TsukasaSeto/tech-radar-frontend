# GitHub Actions のベストプラクティス

フロントエンドの CI で「速く・確実に・安全に」回すための GitHub Actions 設計。

## ルール

### 1. 標準パイプラインを `ci.yml` 1 本に集約する

`typecheck` / `lint` / `test` / `build` の 4 ステップを `.github/workflows/ci.yml` に集約し、PR で必ず実行する。
ワークフローを細分化しすぎず、最小限のジョブで保守性を保つ。

**根拠**:
- 細かい workflow file が増えると保守不能になる（同じセットアップを毎回書く）
- 1 ファイルで全ジョブを管理すると node セットアップ・cache キーなどの共通化が容易
- ジョブ間の依存（`needs:`）で並列化と直列化を制御できる
- failed run の調査も 1 ファイル内なら追いやすい

**標準テンプレート**（pnpm + Node セットアップは Composite Action に切り出して共通化する）:

```yaml
# .github/actions/setup/action.yml — Install pnpm, Node, deps
name: Setup
runs:
  using: composite
  steps:
    - uses: pnpm/action-setup@v3
    - uses: actions/setup-node@v4
      with: { node-version: '20', cache: pnpm }
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

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  typecheck: { runs-on: ubuntu-latest, steps: [ {uses: actions/checkout@v4}, {uses: ./.github/actions/setup}, {run: pnpm typecheck} ] }
  lint:      { runs-on: ubuntu-latest, steps: [ {uses: actions/checkout@v4}, {uses: ./.github/actions/setup}, {run: pnpm lint} ] }
  test:      { runs-on: ubuntu-latest, steps: [ {uses: actions/checkout@v4}, {uses: ./.github/actions/setup}, {run: pnpm test --reporter=verbose} ] }
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup
      - run: pnpm build
        env:
          NEXT_PUBLIC_APP_URL: https://example.com
```

Composite Action を使わずに setup ステップを各ジョブで直書きすると、`pnpm/action-setup` / `actions/setup-node` / `pnpm install` の 3 ステップが 4 ジョブで重複し、保守性が落ちる。`actions/cache` を使う場合も Composite Action 内に取り込む。

**判断軸**:
- ワークフローは「トリガー × デプロイ先」で分ける
  - `ci.yml`: PR・push の検証
  - `deploy-staging.yml`: staging デプロイ（main ブランチ）
  - `release.yml`: タグ作成・本番リリース
  - `nightly.yml`: スケジュールジョブ（依存監査・E2E 等）
- 同じトリガーで複数 workflow を分けない（並列実行の依存管理が困難）

**出典**:
- [GitHub Actions: Workflow syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions) (GitHub Docs)
- [GitHub: Composite actions](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action) (GitHub Docs)

**バージョン**: GitHub Actions
**確信度**: 高
**最終更新**: 2026-05-16

---

### 2. `concurrency` で同じブランチの古い実行を自動キャンセル

PR で連続 push した時に古い CI run をキャンセルする `concurrency:` 設定を入れる。
無駄な CI 実行時間を削減し、フィードバックを速くする。

**根拠**:
- PR で 5 回連続 push すると最初の 4 回の CI run は無意味（最終結果しか見ない）
- `concurrency.cancel-in-progress: true` で自動的に古い run をキャンセル
- main ブランチや release タグでは cancel しない（履歴として全 run が必要）
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
cancel-in-progress: false  # キャンセルせずに wait
```

**ジョブレベルの concurrency**:
```yaml
jobs:
  deploy:
    concurrency:
      group: deploy-production
      cancel-in-progress: false  # production デプロイは順次実行
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

**よくある失敗**:
- 全 workflow で `cancel-in-progress: true` → main ブランチでの run が他 PR にキャンセルされる
- group キーが PR 単位でない → 別 PR が同じグループに入ってお互いキャンセル
- production デプロイで cancel → 中途半端なデプロイ状態が残る

**判定の原則**:
| シチュエーション | cancel-in-progress |
|---|---|
| PR の検証（typecheck / test 等） | `true` |
| main ブランチへの merge 後の検証 | `false`（履歴を残す） |
| production デプロイ | `false`（中断しない） |
| preview deploy | `true`（最新のみ） |
| release ワークフロー（タグ作成） | `false` |

**出典**:
- [GitHub: Workflow concurrency](https://docs.github.com/en/actions/using-jobs/using-concurrency) (GitHub Docs)

**バージョン**: GitHub Actions
**確信度**: 高
**最終更新**: 2026-05-16

---

### 3. `actions/cache` で `node_modules` と Next.js キャッシュを永続化する

`pnpm` の store・Next.js の `.next/cache` を CI run 間で再利用し、ビルド時間を 30-70% 短縮する。
`actions/setup-node` の `cache:` オプションだけでは Next.js キャッシュが取れない。

**根拠**:
- `pnpm install --frozen-lockfile` も 30-60 秒かかる。キャッシュで 5 秒に短縮
- Next.js は `.next/cache` に webpack / babel / SWC のキャッシュを保存。これがあると build 時間が半分以下
- `actions/setup-node` の `cache:` は依存パッケージのみ。Next.js キャッシュは別途設定
- Turborepo / Nx を使う場合は Turbo / Nx のリモートキャッシュも併用（[build-cache.md](./build-cache.md) 参照）

**設定例**:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: pnpm                      # ← pnpm store のキャッシュ

      # Next.js キャッシュ
      - uses: actions/cache@v4
        with:
          path: |
            ${{ github.workspace }}/.next/cache
          key: ${{ runner.os }}-nextjs-${{ hashFiles('**/pnpm-lock.yaml') }}-${{ hashFiles('**/*.[jt]s', '**/*.[jt]sx') }}
          restore-keys: |
            ${{ runner.os }}-nextjs-${{ hashFiles('**/pnpm-lock.yaml') }}-

      - run: pnpm install --frozen-lockfile
      - run: pnpm build
```

**キャッシュキー設計の判断軸**:

| 構成要素 | 例 | 意味 |
|---|---|---|
| OS | `${{ runner.os }}` | OS が違うとバイナリ互換性なし |
| 依存変更 | `hashFiles('pnpm-lock.yaml')` | lockfile が変わったら別キャッシュ |
| ソース変更 | `hashFiles('**/*.{ts,tsx}')` | ソース変更で Next.js キャッシュをセグメント化 |

**`restore-keys` でフォールバック**:
```yaml
key: ${{ runner.os }}-nextjs-${{ hashFiles('pnpm-lock.yaml') }}-${{ hashFiles('src/**/*') }}
restore-keys: |
  ${{ runner.os }}-nextjs-${{ hashFiles('pnpm-lock.yaml') }}-
  ${{ runner.os }}-nextjs-
```

完全一致が無くても、最も近い古いキャッシュを取得して部分再利用できる。

**Playwright ブラウザのキャッシュ**:
```yaml
- uses: actions/cache@v4
  id: playwright-cache
  with:
    path: ~/.cache/ms-playwright
    key: ${{ runner.os }}-playwright-${{ hashFiles('**/pnpm-lock.yaml') }}

- if: steps.playwright-cache.outputs.cache-hit != 'true'
  run: pnpm exec playwright install --with-deps
```

**ESLint / TypeScript のインクリメンタル**:
```yaml
- uses: actions/cache@v4
  with:
    path: |
      .eslintcache
      .tsbuildinfo
    key: ${{ runner.os }}-lint-${{ hashFiles('**/*.{ts,tsx}') }}
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "incremental": true,
    "tsBuildInfoFile": ".tsbuildinfo"
  }
}
```

**キャッシュサイズの上限**:
- GitHub Actions: リポジトリあたり 10GB（古いキャッシュは LRU で削除）
- 大きすぎるキャッシュ（Playwright 等）は run 別に切り分ける

**出典**:
- [GitHub: Caching dependencies](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows) (GitHub Docs)
- [Next.js Docs: CI build caching](https://nextjs.org/docs/app/building-your-application/deploying/ci-build-caching) (Next.js 公式)

**バージョン**: actions/cache v4
**確信度**: 高
**最終更新**: 2026-05-16

---

### 4. Matrix で Node.js バージョン・OS を限定的に実行する

`matrix:` でテスト対象 Node.js バージョン・OS を並列実行する。
ただし「すべての OS × すべての Node バージョン」は CI 時間を浪費するため、PR では最小構成・main / nightly で全構成にする。

**根拠**:
- ライブラリは複数 Node.js バージョン（18, 20, 22）でテストすべき
- アプリは LTS 1 バージョンに固定するのが現実的
- Windows / Linux でテストすると path 区切り・改行コード差で発見できるバグがある
- 並列ジョブ数は GitHub Actions の billing に影響（Linux 1/Win 2/Mac 10 倍）

**Matrix 設定**:
```yaml
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest]
        node: [20]
        include:
          # main ブランチや nightly で別 OS / バージョンを追加
          - os: macos-latest
            node: 20
            condition: ${{ github.ref == 'refs/heads/main' }}
          - os: windows-latest
            node: 20
            condition: ${{ github.ref == 'refs/heads/main' }}
          - os: ubuntu-latest
            node: 22
            condition: ${{ github.event_name == 'schedule' }}

    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '${{ matrix.node }}' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm test
```

テストスイートの並列分割（シャーディング）による高速化は Rule #10 を参照。

**コスト計算（公式 GitHub Actions）**:
- Linux runner: 1 倍
- Windows runner: 2 倍
- macOS runner: 10 倍（M1 ベース）

「全 OS でテスト」は明確な理由がある時だけ。フロントエンドアプリは通常 Linux のみで十分。

**Self-hosted runner**:
- 大規模リポジトリ・特殊環境では self-hosted runner を使う
- セキュリティ上の理由から、public リポジトリでは推奨しない（OSS 攻撃の経路になる）
- 設定: `runs-on: self-hosted` または `runs-on: [self-hosted, linux, x64]`

**出典**:
- [GitHub: Using a matrix](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs) (GitHub Docs)
- [GitHub Actions: Billing](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions) (GitHub Docs)

**バージョン**: GitHub Actions
**確信度**: 高
**最終更新**: 2026-05-16

---

### 5. PR コメントで成果物（カバレッジ・bundle size）を可視化する

CI 結果（テストカバレッジ・bundle size 差分・Lighthouse スコア）を PR コメントに自動投稿する。
ダッシュボードを見に行く必要がない UX で運用する。

**根拠**:
- PR レビュー時に「数値が改善 / 悪化したか」を即座に把握できる
- 別ツールに飛ばずコンテキストスイッチを最小化
- 数値の経緯がコメント履歴として残り、後から「なぜここで悪化したか」を追える
- bundle size の悪化検出は performance regression を未然に防ぐ最も効果的な防衛策

**Bundle Size Analyzer**:
```yaml
# .github/workflows/bundle-size.yml
- name: Compare bundle sizes
  uses: vio/bundle-stats-action@v1
  with:
    workflow-job-name: build
    package-name: '@example/web'
    annotations: true  # PR コメントに自動投稿
```

**自前で bundle size を投稿**:
```yaml
- name: Build
  run: pnpm build

- name: Calculate bundle size
  id: size
  run: |
    SIZE=$(du -sk .next/static/chunks/ | cut -f1)
    echo "size_kb=$SIZE" >> $GITHUB_OUTPUT

- name: Comment PR
  uses: actions/github-script@v7
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
  uses: treosh/lighthouse-ci-action@v11
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

**Test Coverage**:
```yaml
- name: Test with coverage
  run: pnpm test --coverage --reporter=json-summary

- name: Coverage report
  uses: ArtiomTr/jest-coverage-report-action@v2
  with:
    coverage-file: ./coverage/coverage-summary.json
    base-coverage-file: ./coverage-base.json  # main ブランチの基準
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
  uses: marocchino/sticky-pull-request-comment@v2
  with:
    header: bundle-size                # 同じ header で更新（コメント増殖を防止）
    message: |
      📦 **Bundle size**: 1.2 MB (+15 KB)
      ✅ **Tests**: 1,234 passed
      📊 **Coverage**: 85.3%
```

**出典**:
- [Lighthouse CI Action](https://github.com/treosh/lighthouse-ci-action) (treosh)
- [Bundle Stats Action](https://github.com/vio/bundle-stats-action) (vio)
- [sticky-pull-request-comment](https://github.com/marocchino/sticky-pull-request-comment) (marocchino)

**バージョン**: GitHub Actions
**確信度**: 高
**最終更新**: 2026-05-16

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
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}  # ← フォーク側コードを取得
      - run: pnpm build  # ← 信頼できないコードを secrets のある環境で実行 ← 危険
      - uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

**安全な二分割パターン（推奨）**:
```yaml
# ワークフロー 1: 信頼できないコードを secrets なし環境でビルド
name: Build (Untrusted)
on:
  pull_request:   # ← pull_request_target ではなく pull_request を使う

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pnpm install --frozen-lockfile && pnpm build
      - uses: actions/upload-artifact@v4
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

jobs:
  comment:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-result
          run-id: ${{ github.event.workflow_run.id }}
      - uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}  # ← secrets はここだけで使う
          script: |
            // PR にコメントを投稿する処理
```

**判断軸（いつ二分割が必要か）**:
| ケース | 対処 |
|---|---|
| 外部フォークの PR で secrets を使いたい | 二分割パターン必須 |
| 自分のブランチからの PR のみ | `pull_request` トリガーで OK |
| Bot PR (Renovate/Dependabot) | secrets 露出しないよう `pull_request` のまま |
| ラベル起動（trusted PR のみ） | `pull_request_target` + `if: github.actor == 'dependabot[bot]'` で制限可 |

**Dependabot PR に Claude Code Action で AI レビューを行う場合の安全パターン**:

`pull_request_target` + `github.actor == 'dependabot[bot]'` 条件で、外部コードの任意実行なしに Dependabot PR へのコメント権限を取得できる。AI エージェントには `--allowedTools` で権限を最小化し、任意シェルコマンド実行を防ぐ。

```yaml
# .github/workflows/dependabot-review.yml
on:
  pull_request_target:
    types: [opened]

permissions: {}   # デフォルトで全権限を拒否

jobs:
  review:
    # Dependabot PR のみに限定
    if: github.actor == 'dependabot[bot]'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          # ツールを最小限に制限 — 任意シェル実行を禁止
          allowed_tools: "Bash(gh pr comment:*),Bash(gh pr diff:*),Bash(gh pr view:*),Read,Grep,Glob,WebFetch"
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

キーポイント:
- `permissions: {}` をワークフロー冒頭に置き、すべての権限をデフォルト拒否にする
- `github.actor == 'dependabot[bot]'` で Dependabot 以外のトリガーを排除
- `--allowedTools` の `:` 区切りサブコマンド制限で、任意 `bash` 実行を防ぐ（`Bash` だけ許可すると全シェルコマンドが実行可能になる）
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
- [ ] サードパーティ Action は `uses: actions/checkout@v4` のようなタグではなくコミット SHA にピン留めしているか（タグは改ざん可能）
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

**バージョン**: GitHub Actions
**確信度**: 高
**最終更新**: 2026-07-08

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
- OIDC プロバイダー自体の作成も Terraform でコード化しておくと、複数リポジトリ・複数ロールへの展開時に手作業でのポリシー設定ミスを防げる
- **最も多い失敗は信頼ポリシーの `sub` 条件を絞り込まないこと**: 組織内の任意のワークフロー（フォークされた marketplace action を含む）が role を assume できてしまう。`sub` は `repo:<ORG>/<REPO>:ref:refs/heads/main` のように **リポジトリ＋ブランチ／environment 単位で厳密に**指定し、別ワークフローに権限を広げたい場合は既存ロールを緩めず**別の狭いロールを追加**する
- 旧クレデンシャルの削除は「新しい経路の疎通確認 → 読み取り専用コマンドで検証 → 本番影響のない操作でテスト → 監査ログ確認」の順で確実に検証してから行う。短命トークンであっても、実行中に奪われた場合の影響範囲は role の権限そのものに比例するため、IAM ロールの権限最小化（`sts:GetCallerIdentity` 相当から段階的に拡張）は OIDC 移行後も引き続き必要
- **OIDC 導入後も `sub` クレーム自体の形式変更を追う必要がある**: GitHub は 2026-07-15 以降に作成・rename されたリポジトリで `sub` のデフォルト形式を「オーナー名/リポジトリ名（可変）」から「オーナーID/リポジトリID を付加した immutable subject claim」に変更した（例: `repo:octocat/my-repo:ref:refs/heads/main` → `repo:octocat@123456/my-repo@456789:ref:refs/heads/main`）。目的はリポジトリ名の再利用（削除→別オーナーが同名で再作成）によるなりすましを防ぐこと。信頼ポリシーの `sub` 条件を名前ベースで書いている既存環境は、移行時に**新形式を先に追加し旧形式を残したまま切り替え、最後に旧形式を削除する**順序を守らないと、切り替え中に正当なワークフローの認証が失敗する

**コード例**:
```yaml
# Bad: 長期間有効なクレデンシャルを Secrets に保存
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
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
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.OIDC_ROLE_ARN }}
          aws-region: ap-northeast-1
```

**AWS IAM ロール信頼ポリシー（最小権限設定）**:
```json
{
  "Effect": "Allow",
  "Principal": { "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com" },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringLike": {
      "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/heads/main"
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
        StringLike = {
          "token.actions.githubusercontent.com:sub" = "repo:my-org/my-repo:ref:refs/heads/main"
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
- [ ] IAM ロールの信頼ポリシーで `sub` 条件をリポジトリ・ブランチに絞る（`*` は禁止）
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

**出典引用**:
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

**出典（追加）**:
- [Docker Hub gets OIDC federation for GitHub Actions, retiring the PAT-in-a-secret pattern](https://dev.to/leobaniak/docker-hub-gets-oidc-federation-for-github-actions-retiring-the-pat-in-a-secret-pattern-1392) (dev.to leobaniak、Docker Hub版OIDC federationの`permissions.id-token:write`+`docker/login-action`構成) ※2026-08-01に実際にfetch成功

**バージョン**: GitHub Actions, aws-actions/configure-aws-credentials v4+, Google Cloud KMS, GCP Workload Identity Federation, Docker Hub OIDC federation, immutable subject claims（2026-07-15以降の新規リポジトリでデフォルト化）
**確信度**: 高
**最終更新**: 2026-08-09

---

### 8. `actions/checkout` に `persist-credentials: false` を設定し、トークンの自動永続化を無効にする

デフォルトでは `actions/checkout` は GitHub トークンを `$RUNNER_TEMP/git-credentials-<UUID>.config` ファイルに書き込んで永続化する。
後続ステップでサードパーティアクションや侵害されたスクリプトが実行された場合、このファイルからトークンを奪取できてしまう。
`persist-credentials: false` でトークン永続化を無効にし、認証が必要な場合のみ `gh auth setup-git` で都度設定する。

**根拠**:
- デフォルト (`persist-credentials: true`) では Base64 エンコードされたトークンが git 認証設定ファイルに書き込まれ、後続の全ステップからアクセス可能
- Rule #6 の pull_request_target サプライチェーンリスクと組み合わせると、外部 PR のスクリプトがトークンを奪取する攻撃経路になる
- `persist-credentials: false` を設定することでトークンが git 設定ファイルに保存されなくなる
- `gh auth setup-git` は git コマンド実行時にのみトークンを参照し、永続ファイルを作成しない
- `ghasec`・`zizmor`・`ghalint` 等の静的解析ツールで全 workflow の設定漏れを自動検出できる。`zizmor` は `persist-credentials` 漏れだけでなく、`${{ github.event.pull_request.title }}` のような信頼できない式を `run:` に直接展開してしまうテンプレートインジェクション、過剰な `permissions`、SHA 未固定のアクション参照もあわせて検出する
- `claude-code-action` はジョブ内で実行中、自分専用の一時トークンに git 認証設定を書き換える。そのため `gh auth setup-git` を先に実行していても、`claude-code-action` の後段に `git push` ステップを置くと認証設定が上書きされ、push がサイレントに失敗しうる。push 直前に `git remote set-url` で認証を明示的に張り直すと安全
- `claude-code-action` の認証は `anthropic_api_key`（従量課金）以外に、`claude setup-token` で発行した `CLAUDE_CODE_OAUTH_TOKEN` を使う方法もある。Claude の Pro/Max サブスクリプション認証で動作し、実行ログの `total_cost_usd` が `0` になることで API キー課金が発生していないことを確認できる

**コード例**:
```yaml
# Bad: デフォルト（トークンが $RUNNER_TEMP に永続化される）
- uses: actions/checkout@v4

# Good: persist-credentials を無効化 + SHA ピニング（Rule #6 推奨事項）
- uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd
  with:
    persist-credentials: false

# 認証が必要な git 操作がある場合は gh auth setup-git で設定
- name: Setup git credentials
  run: gh auth setup-git  # 実行時のみトークンを参照、永続ファイルは作成しない
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

- name: Push changes
  run: git push origin main  # gh auth setup-git 後は認証済みで動作する

# claude-code-action の後段で push する場合は、認証設定の上書きに注意
- uses: anthropics/claude-code-action@v1
  with:
    anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}

- name: Push changes after claude-code-action
  run: |
    # claude-code-action が git 認証設定を専用トークンで上書きしているため張り直す
    git remote set-url origin "https://x-access-token:${GITHUB_TOKEN}@github.com/${GITHUB_REPOSITORY}.git"
    git push origin main
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**`zizmor` での自動検知例**:
```yaml
# .github/workflows/zizmor.yml
- name: Run zizmor
  uses: zizmorcore/zizmor-action@v1
```
```yaml
# Bad: PR タイトルを run: に直接展開（テンプレートインジェクション、zizmor が検出）
- run: echo "PR Title is: ${{ github.event.pull_request.title }}"

# Good: 環境変数経由でシェルに渡す（式展開ではなく変数参照にする）
- env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: echo "PR Title is: $PR_TITLE"
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

**バージョン**: actions/checkout v4+ / anthropics/claude-code-action v1 / zizmor
**確信度**: 高
**最終更新**: 2026-08-01

---

### 9. GitHub Actions のアクション参照はコミット SHA にピン留めし、バージョンコメントを必ず追記する

フローティングタグ（`@v4` 等）はリポジトリオーナーによって書き換え可能なため、サプライチェーン攻撃のベクターになる。
アクションを 40 文字の完全コミット SHA で参照し、可読性のためバージョンコメントを必ず隣接追記する。
`pinact` などのツールで既存ワークフローを一括ピン留めし、CI で SHA とコメントの整合性を自動検証する。

**根拠**:
- フローティングタグは書き換え可能。コミット SHA は不変であり、サードパーティアクションの改ざんを検知できる
- バージョンコメントがないと fork リポジトリの不審なコミットと区別できない
- SmartRound の供給チェーン監査基盤でも「アクションのコミット SHA 参照」を必須チェック項目として実装している
- `pinact v4` の `--verify-comment` オプションで CI 上の SHA とバージョンコメントの整合性を自動検証できる

**コード例**:
```yaml
# Good: 40文字 SHA + バージョンコメント（書き換え不可・可読性維持）
steps:
  - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
    with:
      persist-credentials: false
  - uses: actions/setup-node@49933ea5288caeca5a7e7b5f82c67a9e37f3c69b  # v4.4.0
    with:
      node-version: '22'

# Bad: フローティングタグ（タグ書き換えリスク）
  - uses: actions/checkout@v4
```

**pinact 設定例（pinact.yaml）**:
```yaml
# 最低 7 日以上経過したコミットのみ許可（直後のマルウェア混入を防ぐ）
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

**バージョン**: GitHub Actions 全バージョン
**確信度**: 高
**最終更新**: 2026-05-26

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

**テストシャーディング**:
```yaml
jobs:
  e2e:
    strategy:
      matrix:
        shard: [1/4, 2/4, 3/4, 4/4]  # 4並列で実行時間を 1/4 に
    steps:
      - uses: actions/checkout@v4
      - run: npx playwright test --shard=${{ matrix.shard }}
      - uses: actions/upload-artifact@v4
        with:
          name: blob-report-${{ matrix.shard }}
          path: blob-report/
          retention-days: 1
```

**paths-filter で条件実行**:
```yaml
jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      frontend: ${{ steps.filter.outputs.frontend }}
      backend: ${{ steps.filter.outputs.backend }}
    steps:
      - uses: dorny/paths-filter@v3
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
      - uses: actions/checkout@v4
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

**出典**:
- [CIを高速化するテクニック集](https://zenn.dev/mandenaren/articles/ci_speedup_techniques) (Zenn) ※2026-06-04 fetch
- [Playwright: Sharding](https://playwright.dev/docs/test-sharding) (Playwright 公式、テスト並列分割)
- [Playwright の E2E CI を4分割し、3時間13分から25分に短縮した](https://zenn.dev/thaddeusjiang/articles/ci-e2e-runtime-optimization) (Zenn、3時間13分→25分の実測改善、Vite preview サーバー化とヘルスチェック導入を含む) ※2026-08-21 fetch

**バージョン**: GitHub Actions 全バージョン、dorny/paths-filter v3
**確信度**: 中（コミュニティ記事、実績多数）
**最終更新**: 2026-08-21

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
      - uses: actions/checkout@v4

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
**最終更新**: 2026-07-11

---

### 12. ジョブ単位・1分未満切り上げの課金ルールを踏まえ、細切れワークフローの分割を避ける

GitHub Actions の課金は「ジョブ単位」で発生し、実行時間は**1分未満切り上げ**で計算される。数秒で終わる処理でも、別ワークフロー・別ジョブに分割していると、その都度 1 分単位で消費される。

**根拠**:
- 実処理が数秒でも、ワークフローを分割している数だけ「1分」が積み上がる（例: 4 ワークフローがそれぞれ 3 秒の curl を実行 → 実処理合計 12 秒でも課金は 1分 × 4 = 4分）
- 無料枠（月 2,000 分）を使い切ると、GitHub は新規ジョブの起動を静かにブロックする。メール通知等の明示的な警告なしに失敗するため、無関係な他のスケジュールジョブ（バックアップ・デプロイ等）も巻き添えで停止し、気づくまで数日かかることがある
- 同じ処理を行うなら、ワークフロー／ジョブを分割せず 1 ジョブ内に複数ステップとしてまとめる方が、切り上げ回数を減らせる
- 組織全体で無料枠が尽きかけている場合は「実行時間を削る」前に効く順にコストを絞る: (1) `paths` フィルタとジョブのゲーティングでトリガー条件自体を絞る（最も効果が大きい）、(2) 課金単位（ジョブ単位・1分未満切り上げ）に処理の粒度を合わせて端数の無駄を減らす、(3) それでも足りない場合にキャッシュ・並列化で実処理時間を削る、という順で着手する
- Settings 側の自動実行（dependency graph の自動ジョブ等）も無料枠を消費するため、必要な場合のみワークフローから明示的に呼び出す形に倒し、常時自動発火は無効化する

**コード例（課金分の再現計算）**:
```javascript
const secs = Math.max(0, (Date.parse(r.updated_at) - started) / 1000);
if (secs < 8 && r.conclusion === "failure") continue;
const m = Math.max(1, Math.ceil(secs / 60)); // 1分未満切り上げを再現
minutes += m;
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

**バージョン**: GitHub Actions（全プラン共通の課金モデル）
**確信度**: 中
**最終更新**: 2026-08-18