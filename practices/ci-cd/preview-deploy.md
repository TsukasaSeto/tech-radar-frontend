# プレビューデプロイのベストプラクティス

PR ごとに固有 URL を割り当て、レビュアー・QA・PdM が実環境で動作確認できる体制を作る。
ビジュアルリグレッション・E2E テストもプレビュー URL を入口に統合する。

## ルール

### 1. PR ごとに固有のプレビュー URL を自動生成する

Vercel / Netlify / Cloudflare Pages 等の PaaS は PR 単位でプレビュー URL を自動生成する機能を持つ。
これを採用し、`pr-<num>--example.vercel.app` のような URL でレビューを行う。

**根拠**:
- code review 時に「URL を踏むだけで動作確認」できる UX は最強
- localhost で動作確認しているとブラウザ拡張・キャッシュ等のローカル状態に依存しがち
- PR 単位なら staging 環境の競合が起きない（同じ staging で複数 PR を検証できない問題を解消）
- Backend サービスが必要なら preview 用の DB・API スタブも自動構築できる

**Vercel での標準動作**:
- GitHub と連携すれば設定不要で全 PR にプレビュー URL が生成される
- URL: `<project>-<branch>-<team>.vercel.app`
- PR コメントに自動投稿される

**Netlify / Cloudflare Pages も同様の機能を持つ**。

**自前で同等の挙動を作る場合（GitHub Actions + Vercel CLI）**:
```yaml
# .github/workflows/preview.yml
name: Preview Deploy
on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm install -g vercel@latest

      - name: Pull Vercel Environment Information
        run: vercel pull --yes --environment=preview --token=${{ secrets.VERCEL_TOKEN }}

      - name: Build
        run: vercel build --token=${{ secrets.VERCEL_TOKEN }}

      - name: Deploy
        id: deploy
        run: |
          URL=$(vercel deploy --prebuilt --token=${{ secrets.VERCEL_TOKEN }})
          echo "url=$URL" >> $GITHUB_OUTPUT

      - name: Comment PR
        uses: marocchino/sticky-pull-request-comment@v2
        with:
          header: preview-url
          message: |
            🚀 **Preview deployed**: ${{ steps.deploy.outputs.url }}
            ✅ Ready for review
```

**preview 環境の DB 構成**:

| パターン | 用途 | コスト |
|---|---|---|
| **共有 preview DB** | 軽量 SaaS、PR 間でデータ共有可 | ◎ 安い |
| **ブランチごとに新規 DB**（Neon / PlanetScale 等の DB branching） | データ整合性が重要 | ○ 中 |
| **Read-only な production snapshot** | 機微データ除く実データで検証 | △ 高（マスキング工数） |

**Neon の DB branching 例**:
```yaml
- name: Create DB branch for preview
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

**バージョン**: パターン
**確信度**: 高
**最終更新**: 2026-05-16

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
| **Chromatic** | Storybook と完全統合、無料枠あり、UI 良好 | Storybook 必須 |
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
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }  # main からの差分検出に必要
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: pnpm }
      - run: pnpm install --frozen-lockfile

      - name: Publish to Chromatic
        uses: chromaui/action@v11
        with:
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
          onlyChanged: true  # 変更されたストーリーのみテスト
          exitZeroOnChanges: false  # 視覚差分があれば fail
```

**Playwright `toHaveScreenshot()`**:
```ts
// e2e/visual.spec.ts
import { test, expect } from '@playwright/test';

test('home page visual regression', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveScreenshot('home.png', {
    fullPage: true,
    mask: [page.locator('[data-test="timestamp"]')],  // 動的要素をマスク
    maxDiffPixelRatio: 0.01,  // 1% 以下の差分は OK
  });
});
```

```ts
// playwright.config.ts
export default defineConfig({
  use: {
    baseURL: process.env.PREVIEW_URL ?? 'http://localhost:3000',
  },
  projects: [
    { name: 'chromium', use: { browserName: 'chromium' } },
    { name: 'webkit', use: { browserName: 'webkit' } },
  ],
  expect: {
    toHaveScreenshot: {
      threshold: 0.2,
      maxDiffPixelRatio: 0.01,
    },
  },
});
```

**動的コンテンツのマスキング**:
```ts
await expect(page).toHaveScreenshot({
  mask: [
    page.locator('[data-test="timestamp"]'),   // 時刻
    page.locator('[data-test="random-id"]'),    // ランダム ID
    page.locator('img[src*="gravatar"]'),       // ユーザーアバター
  ],
});
```

**運用ルール**:
- PR で差分が出たら **レビュー必須**（自動マージ不可）
- 意図的な変更なら snapshot を更新（`npx playwright test --update-snapshots`）
- スナップショットは `git lfs` で管理（リポジトリサイズ膨張を防ぐ）
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

**バージョン**: パターン
**確信度**: 高
**最終更新**: 2026-05-16

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
```yaml
- name: Update PR with preview info
  uses: marocchino/sticky-pull-request-comment@v2
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
  run: |
    BASE_SIZE=$(curl -s "${{ steps.base.outputs.url }}/api/build-info" | jq .bundleSize)
    PR_SIZE=$(stat -c%s .next/static/chunks/*.js | awk '{sum+=$1} END {print sum}')
    DIFF=$((PR_SIZE - BASE_SIZE))
    echo "diff_bytes=$DIFF" >> $GITHUB_OUTPUT
    [ $DIFF -gt 50000 ] && echo "warning=true" >> $GITHUB_OUTPUT

- name: Comment
  if: steps.diff.outputs.warning == 'true'
  uses: marocchino/sticky-pull-request-comment@v2
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
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "blocks": [
          {
            "type": "section",
            "text": { "type": "mrkdwn", "text": "*Preview ready*: <${{ steps.deploy.outputs.url }}|PR #${{ github.event.pull_request.number }}>" }
          }
        ]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_PREVIEW_WEBHOOK }}
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

**バージョン**: GitHub Actions
**確信度**: 高
**最終更新**: 2026-05-16

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
```yaml
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
```ts
// next.config.ts
const securityHeaders = [
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubDomains; preload' },
  { key: 'X-Frame-Options', value: 'SAMEORIGIN' },
];

export default {
  async headers() {
    return [{ source: '/(.*)', headers: securityHeaders }];
  },
};
```

これにより preview / production の両方で同じヘッダー検証ができる。

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

**Basic 認証（外部から見られたくない場合）**:
```ts
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  if (process.env.VERCEL_ENV !== 'preview') return NextResponse.next();

  const auth = request.headers.get('authorization');
  if (!auth) {
    return new NextResponse(null, {
      status: 401,
      headers: { 'WWW-Authenticate': 'Basic realm="Preview"' },
    });
  }

  const [user, pass] = atob(auth.split(' ')[1]).split(':');
  if (user !== 'preview' || pass !== process.env.PREVIEW_PASSWORD) {
    return new NextResponse('Unauthorized', { status: 401 });
  }

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
| Basic 認証 | 必要に応じて | なし | 機密保護 |

**出典**:
- [Vercel: Environment Variables](https://vercel.com/docs/environment-variables) (Vercel)
- [Neon: Database Branching](https://neon.tech/docs/introduction/branching) (Neon)

**バージョン**: パターン
**確信度**: 高
**最終更新**: 2026-05-16
