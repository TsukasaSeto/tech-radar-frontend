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

**バージョン**: GitHub Actions
**確信度**: 高
**最終更新**: 2026-05-29

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

> "本手法では秘密鍵は Cloud KMS から外に出ることなく、署名処理のみを KMS API で行います。秘密鍵そのものを手元で管理する必要がなくなるため、流出リスクを大幅に低減できます。"
> ([GitHub App の秘密鍵を Cloud KMS に閉じ込める](https://zenn.dev/acntechjp/articles/64c6deacee1c97), セクション "Flow/Process") ※2026-05-22に実際にfetch成功

**バージョン**: GitHub Actions, aws-actions/configure-aws-credentials v4+, Google Cloud KMS
**確信度**: 高
**最終更新**: 2026-05-22

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
- `ghasec`・`zizmor`・`ghalint` 等の静的解析ツールで全 workflow の設定漏れを自動検出できる

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
```

**出典引用**:
> "the auth token is persisted in the local git config. This enables your scripts to run authenticated git commands. The token is removed during post-job cleanup."
> ([actions/checkout README](https://raw.githubusercontent.com/actions/checkout/main/README.md)) ※2026-05-25に実際にfetch成功

> "後続ステップは credential ファイルから GitHub トークンを容易に奪取できます。persist-credentials: false を設定することで、このリスクを根本から排除できます。"
> ([【GitHub Actions】actions/checkout には persist-credentials: false を設定するべき](https://zenn.dev/kou_pg_0131/articles/gha-checkout-persist-credentials), セクション "persist-credentials: false を設定するべき理由") ※2026-05-25に実際にfetch成功

**バージョン**: actions/checkout v4+
**確信度**: 高
**最終更新**: 2026-05-25

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

**出典**:
- [CIを高速化するテクニック集](https://zenn.dev/mandenaren/articles/ci_speedup_techniques) (Zenn) ※2026-06-04 fetch
- [Playwright: Sharding](https://playwright.dev/docs/test-sharding) (Playwright 公式、テスト並列分割)

**バージョン**: GitHub Actions 全バージョン、dorny/paths-filter v3
**確信度**: 中（コミュニティ記事、実績多数）
**最終更新**: 2026-06-04