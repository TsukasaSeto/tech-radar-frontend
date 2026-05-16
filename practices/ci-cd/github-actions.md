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

**標準テンプレート**:
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

env:
  NODE_VERSION: '20'

jobs:
  install:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: pnpm

      - run: pnpm install --frozen-lockfile

  typecheck:
    needs: install
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck

  lint:
    needs: install
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint

  test:
    needs: install
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm test --reporter=verbose

  build:
    needs: install
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
        env:
          NEXT_PUBLIC_APP_URL: https://example.com
```

**Composite Action でセットアップを共通化**:
```yaml
# .github/actions/setup/action.yml
name: Setup
description: Install pnpm, Node, and dependencies
runs:
  using: composite
  steps:
    - uses: pnpm/action-setup@v3
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: pnpm
    - shell: bash
      run: pnpm install --frozen-lockfile
```

```yaml
# ci.yml が短くなる
jobs:
  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup
      - run: pnpm typecheck
```

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

**シャーディング（並列実行）**:
```yaml
# Playwright shard で並列化
strategy:
  matrix:
    shard: [1, 2, 3, 4]
steps:
  - run: pnpm exec playwright test --shard=${{ matrix.shard }}/4
```

これで E2E テストを 4 並列実行し、実行時間を 1/4 にできる。

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
- [Playwright: Sharding](https://playwright.dev/docs/test-sharding) (Playwright 公式)

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
