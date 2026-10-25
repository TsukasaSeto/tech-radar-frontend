# プレビューデプロイのベストプラクティス

PR ごとに固有 URL を割り当て、レビュアー・QA・PdM が実環境で動作確認できる体制を作る。
ビジュアルリグレッション・E2E テストもプレビュー URL を入口に統合する。

## ルール

### 1. PR ごとに固有のプレビュー URL を自動生成する

Vercel / Netlify / Cloudflare Pages 等の PaaS は PR 単位でプレビュー URL を自動生成する機能を持つ。
これを採用し、Vercel なら `<project-name>-git-<branch-name>-<scope-slug>.vercel.app` のようなブランチ単位の URL でレビューを行う。

**根拠**:
- code review 時に「URL を踏むだけで動作確認」できる UX は最強
- localhost で動作確認しているとブラウザ拡張・キャッシュ等のローカル状態に依存しがち
- PR 単位なら staging 環境の競合が起きない（同じ staging で複数 PR を検証できない問題を解消）
- Backend サービスが必要なら preview 用の DB・API スタブも自動構築できる

**Vercel での標準動作**:
- GitHub と連携すれば設定不要で全 PR にプレビュー URL が生成される
- URL: ブランチ単位は `<project-name>-git-<branch-name>-<scope-slug>.vercel.app`（push しても変わらない）、コミット単位は `<project-name>-<unique-hash>-<scope-slug>.vercel.app`
- PR コメントに自動投稿される

**Netlify / Cloudflare Pages も同様の機能を持つ**。

**自前で同等の挙動を作る場合（GitHub Actions + Vercel CLI）**:

`VERCEL_TOKEN` は長期トークンなので、`practices/ci-cd/github-actions.md` #7 の「OIDC 未対応のデプロイ先」の扱いにする（有効期限を付け、environment の secret に置き、定期的にローテーションする）。CLI には `--token=` の引数ではなく環境変数で渡す（引数はプロセス一覧やログに出うる）。fork からの PR には secrets が渡らずデプロイが失敗するため、このワークフローの対象から外す（fork の PR のプレビューが必要なら Vercel の Git 連携に任せる）。

```yaml
# .github/workflows/preview.yml
name: Preview Deploy
on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions: {}

jobs:
  deploy-preview:
    # fork からの PR は secrets が渡らないので対象外
    if: github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    environment: preview   # VERCEL_TOKEN はこの environment の secret に置く
    permissions:
      contents: read
      pull-requests: write   # Rule #3 の PR コメント投稿に使う
    env:
      VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}          # Bad: --token=${{ secrets.VERCEL_TOKEN }} を引数に書く
      VERCEL_ORG_ID: ${{ vars.VERCEL_ORG_ID }}           # この 2 つがあると vercel link なしでプロジェクトを特定できる
      VERCEL_PROJECT_ID: ${{ vars.VERCEL_PROJECT_ID }}
    steps:
      - uses: actions/checkout@v7
        with: { persist-credentials: false }
      - uses: actions/setup-node@v7
        with: { node-version: 24 }
      - run: npm install -g vercel@65.2.0   # Bad: vercel@latest（CLI の更新で突然挙動が変わる）。固定して Renovate 等で上げる

      - name: Pull Vercel Environment Information
        run: vercel pull --yes --environment=preview

      - name: Build
        run: vercel build

      - name: Deploy
        id: deploy
        run: |
          URL=$(vercel deploy --prebuilt)
          echo "url=$URL" >> "$GITHUB_OUTPUT"

      # PR へのプレビュー URL の投稿は Rule #3 の「Update PR with preview info」ステップをここに続ける
      # （投稿を 1 つの sticky comment にまとめ、コメントが二重にならないようにする）
```

**preview 環境の DB 構成**:

| パターン | 用途 | コスト |
|---|---|---|
| **共有 preview DB** | 軽量 SaaS、PR 間でデータ共有可 | ◎ 安い |
| **ブランチごとに新規 DB**（Neon / PlanetScale 等の DB branching） | データ整合性が重要 | ○ 中 |
| **Read-only な production snapshot** | 機微データ除く実データで検証 | △ 高（マスキング工数） |

**Neon の DB branching 例**:

PR を閉じたときにブランチを消すには、`on.pull_request.types` に `closed` を加え、作成と削除をイベントで分ける。`closed` を加えないと削除のステップは一度も動かず、DB ブランチが溜まり続ける。上の `deploy-preview` ジョブも `closed` では動かないよう `if: github.event.action != 'closed'` を足す。

```yaml
# on.pull_request.types: [opened, synchronize, reopened, closed]
- name: Create DB branch for preview
  if: github.event.action != 'closed'
  run: |
    BRANCH_ID=$(neonctl branches create \
      --project-id $NEON_PROJECT_ID \
      --name "preview/pr-${{ github.event.pull_request.number }}" \
      --output json | jq -r '.id')
    echo "DATABASE_URL=postgresql://..." >> $GITHUB_OUTPUT

- name: Cleanup DB branch on PR close
  if: github.event.action == 'closed'
  run: |
    neonctl branches delete --project-id $NEON_PROJECT_ID "preview/pr-${{ github.event.pull_request.number }}"
```

**シークレット・環境変数の管理**:
- preview / production で同じ環境変数を使えるよう Vercel / GitHub で個別管理
- preview 専用の API キー・トークンを発行（production を漏らさない）
- 第三者サービス（Stripe / Sentry 等）も preview 用キー

**判定の指針**:
- 全 PR で preview 自動生成が基本
- リソース節約のため、`[skip preview]` PR labels で除外可能に
- backend API がない / モックで動く PR は preview 不要

**出典**:
- [Vercel: Preview Deployments](https://vercel.com/docs/deployments/preview-deployments) (Vercel)
- [Netlify: Deploy Previews](https://docs.netlify.com/site-deploys/deploy-previews/) (Netlify)
- [Cloudflare Pages: Preview deployments](https://developers.cloudflare.com/pages/configuration/preview-deployments/) (Cloudflare)
- [Vercel Docs: Generated URLs](https://vercel.com/docs/deployments/generated-urls) (Vercel 公式、ブランチ単位・コミット単位の URL の形式) ※2026-10-25 確認
- [Vercel CLI: Global Options](https://vercel.com/docs/cli/global-options) (Vercel 公式、CI では `VERCEL_TOKEN` 環境変数を推奨・`VERCEL_ORG_ID` / `VERCEL_PROJECT_ID` でリンクを省略) ※2026-10-25 確認

**バージョン**: パターン（Vercel CLI の例は 65.x）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 2. プレビュー URL に対して Visual Regression Test を実行する

Chromatic / Percy / Argos / Playwright `toHaveScreenshot()` のいずれかでビジュアル差分を検出し、PR にレポートを投稿する。
意図しない UI 変更を CI で機械的にブロックする。

**根拠**:
- CSS 変更・コンポーネント変更が予期せぬページに影響することがある（global CSS / theme 変更）
- 単体テストや E2E では「色がちょっと変わった」「padding がズレた」を検出できない
- ピクセル単位の比較で人間レビューだけでは見逃す変化を捉える
- 多数のコンポーネント・ページに対して低コストで運用できる

**主要選択肢**:

| ツール | 強み | 弱み |
|---|---|---|
| **Chromatic** | Storybook と完全統合、Playwright / Cypress のテストにも対応、無料枠あり、UI 良好 | SaaS（スクリーンショットを外部サービスに送る） |
| **Percy** | Browserstack 統合、複数解像度 | やや高価 |
| **Argos** | Playwright スクリーンショットを直接比較、OSS | UI 専用、複雑なケース弱め |
| **Playwright `toHaveScreenshot`** | ライブラリ不要、E2E と統合 | 比較レポート UI が貧弱 |
| **Lost Pixel** | OSS、Storybook / Playwright 両対応 | エコシステム小 |

**Chromatic + Storybook の標準フロー**:
```yaml
# .github/workflows/chromatic.yml
name: Chromatic
on:
  pull_request:
  push:
    branches: [main]

jobs:
  visual-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with: { fetch-depth: 0, persist-credentials: false }  # main からの差分検出に必要
      - uses: actions/setup-node@v7
        with: { node-version: 24 }
      # 実運用ではサードパーティ action を SHA でピン留めする（`practices/ci-cd/github-actions.md` #9）
      - uses: pnpm/setup@v3
        with: { node-version-file: false, cache: true, install: false }
      - run: pnpm install --frozen-lockfile

      - name: Publish to Chromatic
        uses: chromaui/action@v18
        with:
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
          onlyChanged: true  # 変更されたストーリーのみテスト
          exitZeroOnChanges: false  # 視覚差分があれば fail
```

**Playwright `toHaveScreenshot()` をプレビュー URL に向ける**:

撮影・比較の書き方（撮影前の待ち方・`fullPage`・動的要素の `mask`・許容差）は `practices/testing/e2e-testing.md` #5 を正本とし、ここではプレビュー URL への接続と CI への組み込みだけを扱う。

```ts
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  use: {
    // CI ではデプロイ済みのプレビュー URL（Rule #1 の steps.deploy.outputs.url）を PREVIEW_URL で渡す
    baseURL: process.env.PREVIEW_URL ?? 'http://localhost:3000',
  },
  projects: [
    { name: 'chromium', use: { browserName: 'chromium' } },
    { name: 'webkit', use: { browserName: 'webkit' } },
  ],
});
```

**運用ルール**:
- PR で差分が出たら **レビュー必須**（自動マージ不可）
- 意図的な変更なら snapshot を更新（`npx playwright test --update-snapshots`）
- スナップショットを `git lfs` で管理する場合（リポジトリサイズ膨張を防ぐ）は、CI の `actions/checkout` に `lfs: true` を付ける。既定は `lfs: false` で、画像の代わりに LFS のポインタファイルと比較して失敗する
- 月次で snapshot を一括再生成（OS / フォント差等の細かなノイズを吸収）

**判断軸（どのツールを選ぶか）**:
- Storybook を使っている → **Chromatic**
- Playwright + E2E のみ → **Argos** または `toHaveScreenshot`
- 大規模 + 予算あり → **Percy**（Browserstack 統合）
- OSS + self-host 重視 → **Lost Pixel**

**出典**:
- [Chromatic Docs](https://www.chromatic.com/docs/) (Chromatic)
- [Playwright: Visual comparisons](https://playwright.dev/docs/test-snapshots) (Playwright 公式)
- [Argos](https://argos-ci.com/) (Argos)
- [chromaui/action action.yml（v18）](https://github.com/chromaui/action/blob/v18/action.yml) (Chromatic 公式、`playwright` / `cypress` 入力) ※2026-10-25 確認

**バージョン**: パターン
**確信度**: 高
**最終更新**: 2026-10-25

---

### 3. プレビュー URL を PR コメントに自動投稿する

deploy 完了後、プレビュー URL を PR コメントに投稿する。
レビュアーがコメント欄からワンクリックで動作確認できるようにする。

**根拠**:
- PR タブを「URL に行く」「コードを読む」「コメントする」の 3 タスクで完結させる
- Vercel / Netlify の標準コメント機能を使う、または GitHub Actions でカスタム
- URL に加えて、build 時刻・コミット SHA・テスト結果を一覧表示するとレビュー体験が向上
- コメントは sticky（更新時に上書き）にして PR をスクロールせずに済むように

**Vercel / Netlify は自動で投稿する**。

**自前実装（GitHub Actions）**:

Rule #1 のワークフローの Deploy ステップ（`id: deploy`）の後に続ける。PR へのプレビュー URL の投稿はこのステップに一本化する（header の違う sticky comment を複数置くとコメントが二重になる）。ジョブには `permissions: pull-requests: write` が要る。

```yaml
- name: Record build time
  id: timestamp
  run: echo "time=$(date -u +%Y-%m-%dT%H:%M:%SZ)" >> "$GITHUB_OUTPUT"

- name: Update PR with preview info
  uses: marocchino/sticky-pull-request-comment@v3   # 実運用では SHA でピン留めする（`practices/ci-cd/github-actions.md` #9）
  with:
    header: preview-deployment
    message: |
      ## 🚀 Preview Deployment

      | Field | Value |
      |---|---|
      | **URL** | [${{ steps.deploy.outputs.url }}](${{ steps.deploy.outputs.url }}) |
      | **Commit** | `${{ github.sha }}` |
      | **Built at** | ${{ steps.timestamp.outputs.time }} |
      | **Status** | ✅ Ready |

      ### Test results
      - 📦 Bundle: 1.2 MB
      - ✅ Tests: 1,234 passed
      - 📊 Coverage: 85.3%

      ### Quick links
      - [Storybook preview](${{ steps.deploy.outputs.url }}/storybook)
      - [Bundle analyzer](${{ steps.deploy.outputs.url }}/__analyze)
      - [Sentry events for this preview](https://sentry.io/...?env=preview-${{ github.event.pull_request.number }})
```

**Diff 表示の例**:
```yaml
- name: Compare with base
  id: diff
  env:
    # main の本番 URL（リポジトリ変数）。/api/build-info はアプリ側で用意するエンドポイントの例
    BASE_URL: ${{ vars.PRODUCTION_URL }}
  run: |
    BASE_SIZE=$(curl -fsS "${BASE_URL}/api/build-info" | jq -e .bundleSize)
    PR_SIZE=$(stat -c%s .next/static/chunks/*.js | awk '{sum+=$1} END {print sum}')
    DIFF=$((PR_SIZE - BASE_SIZE))
    echo "diff_bytes=$DIFF" >> "$GITHUB_OUTPUT"
    # Bad: [ $DIFF -gt 50000 ] && echo ...（閾値以下のとき最後のコマンドの終了コードが 1 になり、ステップが失敗する）
    # Good: if で書く
    if [ "$DIFF" -gt 50000 ]; then
      echo "warning=true" >> "$GITHUB_OUTPUT"
    fi

- name: Comment
  if: steps.diff.outputs.warning == 'true'
  uses: marocchino/sticky-pull-request-comment@v3
  with:
    header: bundle-warning
    message: |
      ⚠️ **Bundle size increased by ${{ steps.diff.outputs.diff_bytes }} bytes**

      Please verify this is intentional. Check the [bundle analyzer](${{ steps.deploy.outputs.url }}/__analyze).
```

**Slack 通知**（チーム全体に通知）:
```yaml
- name: Notify Slack on preview ready
  if: github.event.pull_request.draft == false
  uses: slackapi/slack-github-action@v4   # v4 は webhook / webhook-type を入力で渡す（v1 の env: SLACK_WEBHOOK_URL は使わない）
  with:
    webhook: ${{ secrets.SLACK_PREVIEW_WEBHOOK }}
    webhook-type: incoming-webhook
    payload: |
      blocks:
        - type: section
          text:
            type: mrkdwn
            text: "*Preview ready*: <${{ steps.deploy.outputs.url }}|PR #${{ github.event.pull_request.number }}>"
```

**コメント増殖を避ける**:
- `marocchino/sticky-pull-request-comment` の `header:` で同じ識別子のコメントを更新
- 古いコメントを削除する Action もあるが、履歴が消えるのでデフォルトは update が良い

**判断ポイント**:
- preview URL は当然投稿
- bundle size / coverage は変化があった時のみ投稿（spam を防ぐ）
- E2E / Visual Regression の差分は別 header で投稿
- 全部 1 つのコメントにまとめると update のタイミングが難しい

**出典**:
- [sticky-pull-request-comment](https://github.com/marocchino/sticky-pull-request-comment) (marocchino)
- [Vercel: PR Comments](https://vercel.com/docs/deployments/git/vercel-for-github#deployment-comments) (Vercel)
- [Slack: Sending data with an incoming webhook](https://docs.slack.dev/tools/slack-github-action/sending-data-slack-incoming-webhook/) (Slack 公式、slack-github-action v4 の `webhook` / `webhook-type` / `payload`) ※2026-10-25 確認

**バージョン**: GitHub Actions（marocchino/sticky-pull-request-comment v3 / slackapi/slack-github-action v4）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 4. プレビュー環境を本番に近づける（同じ設定・スタブされた依存）

プレビュー環境は production と同じビルド設定・同じセキュリティヘッダーで動作させる。
DB やサードパーティ API はスタブまたは preview 専用インスタンスに繋ぐ。

**根拠**:
- 「localhost では動いたが、production で壊れた」を防ぐには preview を production と同じビルドで作る
- 環境変数だけ preview/prod で切り替えれば、ビルド・コードは同一
- 「production だけで再現するバグ」は最も発見が遅く影響が大きい
- ただし production の実 DB を preview に繋ぐと事故リスクが高い → preview 用に隔離

**Next.js での同等環境**:
```ts
// next.config.ts
export default {
  // production と preview で同じビルド最適化
  // 環境変数だけ差し替える
};

// Vercel Environment Variables の使い分け
// - production: 本番 API キー、本番 DB URL
// - preview: preview API キー、preview DB URL（ブランチ別 or 共有）
// - development: localhost
```

**プレビュー専用 DB（推奨）**:
```bash
# Vercel Postgres / Neon / PlanetScale の branch 機能
DATABASE_URL_PREVIEW="postgresql://preview-branch..."

# preview ビルド時に切り替え
NEXT_PUBLIC_API_URL=https://preview-api.example.com
DATABASE_URL=$DATABASE_URL_PREVIEW
```

**サードパーティ API のスタブ**:
- **Stripe**: test mode のキーを preview で使用（本番の決済を発生させない）
- **SendGrid / Resend**: sandbox mode または preview 専用 sender
- **Sentry**: preview 専用 project または environment tag で分離
- **Analytics**: preview では disable

**セキュリティヘッダーも同一に**:

ヘッダーは環境で分岐させず、preview と production で同じ `next.config.ts` の `headers()` を使う。これにより preview / production の両方で同じヘッダー検証ができる。ヘッダーの値と設定方法は `practices/nextjs/security-headers.md` #1（`X-Frame-Options: DENY`、HSTS は `max-age=63072000; includeSubDomains` で `preload` は preload list に登録する場合だけ、など）、CSP は `practices/security/csp.md` #6 を正本とする。

**Robots / Search engine indexing**:
```ts
// preview では noindex を付与（検索結果に出るのを防ぐ）
export default function RootLayout() {
  const isPreview = process.env.VERCEL_ENV === 'preview';
  return (
    <html>
      <head>
        {isPreview && <meta name="robots" content="noindex, nofollow" />}
      </head>
      ...
    </html>
  );
}
```

**アクセス制限（外部から見られたくない場合）**:

Vercel では Deployment Protection を第一候補にする。Standard Protection（本番ドメイン以外のすべてのデプロイを保護）と Vercel Authentication（チームのメンバーなど権限のある Vercel ユーザーだけが見られる）は全プランで使える。パスワードで共有したい場合は Password Protection（Pro は保護するプロジェクトごとの有料オプション、Enterprise は含まれる。Hobby は不可）を使う。アプリのコードに認証を書かずに済み、Routing Middleware へのリクエストも保護される。

自前の Basic 認証は、Deployment Protection を使えないホスティングや plan の場合の代替にする。書くときはフェイルクローズにする: パスワードが未設定なら通さず、`Authorization` ヘッダーのスキームと形式を検証し、比較は定数時間で行う。

```ts
// proxy.ts（Next.js 16+。v15 以前は middleware.ts に export function middleware で書く）
import { createHash, timingSafeEqual } from 'node:crypto';
import { NextResponse, type NextRequest } from 'next/server';

const sha256 = (value: string) => createHash('sha256').update(value).digest();

function unauthorized() {
  return new NextResponse('Unauthorized', {
    status: 401,
    headers: { 'WWW-Authenticate': 'Basic realm="Preview", charset="UTF-8"' },
  });
}

export function proxy(request: NextRequest) {
  // Vercel 以外のホスティングでは、preview 環境を示す自前の環境変数で判定する
  if (process.env.VERCEL_ENV !== 'preview') return NextResponse.next();

  const password = process.env.PREVIEW_PASSWORD;
  // Bad: 未設定のまま比較する（"preview" だけを base64 にして送ると split(':') の pass が undefined になり、未設定の PREVIEW_PASSWORD と一致して通る）
  // Good: 未設定ならフェイルクローズで 503 を返す
  if (!password) {
    return new NextResponse('Preview protection is not configured', { status: 503 });
  }

  const [scheme, encoded] = request.headers.get('authorization')?.split(' ') ?? [];
  if (scheme?.toLowerCase() !== 'basic' || !encoded) return unauthorized();

  // Buffer.from は不正な base64 でも例外を投げない（atob は例外を投げて 500 になる）
  const credentials = Buffer.from(encoded, 'base64').toString('utf8');
  // "user:password" 全体を比べるので、パスワードに ":" が含まれても壊れない。ハッシュで長さを揃えて定数時間で比較する
  if (!timingSafeEqual(sha256(credentials), sha256(`preview:${password}`))) return unauthorized();

  return NextResponse.next();
}
```

**判定（preview と production の差分許容）**:
| 項目 | preview | production | 理由 |
|---|---|---|---|
| ビルド設定 | 同じ | 同じ | バグの再現性 |
| 環境変数 | preview 専用 | 本番 | 隔離 |
| DB | preview 専用 / branch | 本番 | データ汚染防止 |
| サードパーティ API | test mode / sandbox | 本番 | 課金・スパム防止 |
| Analytics / RUM | 無効 or preview env | 本番 | データ汚染防止 |
| robots.txt | noindex | 通常 | SEO 影響回避 |
| アクセス制限 | Deployment Protection（または Basic 認証） | なし | 機密保護 |

**出典**:
- [Vercel: Environment Variables](https://vercel.com/docs/environment-variables) (Vercel)
- [Neon: Database Branching](https://neon.tech/docs/introduction/branching) (Neon)
- [Vercel Docs: Deployment Protection](https://vercel.com/docs/deployment-protection) (Vercel 公式、保護方式と対象プラン・Standard Protection の範囲・Password Protection の料金) ※2026-10-25 確認

**バージョン**: パターン（`proxy.ts` は Next.js 16+）
**確信度**: 高
**最終更新**: 2026-10-25
