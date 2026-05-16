# リリース戦略のベストプラクティス

タグ作成・バージョニング・カナリアリリース・即時ロールバックを自動化する。
手動デプロイは深夜の事故と運用負荷の温床。

## ルール

### 1. Changesets（モノレポ）または semantic-release（単一パッケージ）で自動バージョニング

バージョン番号は手で書かない。Conventional Commits からの自動生成、または明示的な changeset ファイルで管理する。
リリースノートも同じツールで自動生成する。

**根拠**:
- 「次のバージョンは何にすべきか」を人が判断するとブレる（major / minor / patch の境界）
- Conventional Commits → 自動的に semver の判定が機械化できる（`fix:` → patch, `feat:` → minor, `BREAKING CHANGE:` → major）
- Changesets は「PR ごとに changeset ファイルを置く」ワークフローでモノレポに最適
- リリースノートが自動で生成され、開発者が手動編集する必要がない

**選択肢**:

| ツール | 適用 | 強み |
|---|---|---|
| **Changesets** | モノレポ・複数パッケージ | パッケージ単位のバージョニング、UI わかりやすい |
| **semantic-release** | 単一パッケージ | Conventional Commits から完全自動 |
| **release-please** | 単一 / モノレポ | Google 製、GitHub 統合強力 |
| **changelogen** | シンプルな単一パッケージ | UnJS 製、軽量 |

**Changesets 設定**:
```bash
pnpm add -D @changesets/cli
pnpm changeset init
```

```json
// .changeset/config.json
{
  "$schema": "https://unpkg.com/@changesets/config@2.3.0/schema.json",
  "changelog": ["@changesets/changelog-github", { "repo": "myorg/myrepo" }],
  "commit": false,
  "fixed": [],
  "linked": [["@example/web", "@example/mobile"]],  // 同期するパッケージ群
  "access": "restricted",
  "baseBranch": "main",
  "updateInternalDependencies": "patch",
  "ignore": []
}
```

**PR 作成フロー**:
```bash
# PR で changeset を追加（インタラクティブ）
pnpm changeset
# → 影響パッケージ・bump 種別（patch/minor/major）・サマリーを入力
# → .changeset/<random-name>.md ファイルが生成される
```

```markdown
<!-- .changeset/poetic-pandas-dance.md -->
---
"@example/web": minor
"@example/ui": patch
---

Add dark mode support to the dashboard
```

**main マージ時の自動リリース**:
```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    branches: [main]

concurrency: ${{ github.workflow }}-${{ github.ref }}

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: pnpm }
      - run: pnpm install --frozen-lockfile

      - name: Create Release PR or Publish
        uses: changesets/action@v1
        with:
          # 未消化の changeset があれば "Version Packages" PR を作る
          # 全 changeset 消化済み + version bump がある場合は publish + tag 作成
          version: pnpm changeset version
          publish: pnpm release  # = changeset publish + 必要なら npm publish
          commit: 'chore: release packages'
          title: 'chore: release packages'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

**semantic-release（単一パッケージ）**:
```bash
pnpm add -D semantic-release @semantic-release/git @semantic-release/changelog
```

```json
// .releaserc.json
{
  "branches": ["main", { "name": "beta", "prerelease": true }],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    "@semantic-release/npm",
    "@semantic-release/github",
    ["@semantic-release/git", {
      "assets": ["package.json", "CHANGELOG.md"],
      "message": "chore(release): ${nextRelease.version} [skip ci]"
    }]
  ]
}
```

**判定**:
- パッケージが 1 つ → semantic-release（コミット message 完全準拠）
- パッケージが複数（モノレポ）→ Changesets（PR ごとに明示）
- Web アプリ単独で npm publish しない → release-please or 自前タグ付け

**出典**:
- [Changesets](https://github.com/changesets/changesets) (Atlassian)
- [semantic-release](https://semantic-release.gitbook.io/) (semantic-release)
- [release-please](https://github.com/googleapis/release-please) (Google)
- [Conventional Commits](https://www.conventionalcommits.org/) (Conventional Commits)

**バージョン**: パターン
**確信度**: 高
**最終更新**: 2026-05-16

---

### 2. カナリアリリース戦略（Branch Deploy / Feature Flag）

100% のユーザーに一気にリリースせず、最初は 1-10% に絞って観測する。
Vercel の rolling release、Cloudflare の gradual rollout、または Feature Flag（LaunchDarkly / Statsig / GrowthBook）を活用する。

**根拠**:
- 大規模変更を全ユーザーに即時展開するとバグ波及が大きい
- 1-10% → 50% → 100% と段階展開で、エラー率・パフォーマンスを観測してから広げる
- Feature Flag なら code レベルで切り替えられ、re-deploy なしで rollback できる
- 「カナリア = 炭鉱のカナリア（先に異常を検知する）」が語源

**Vercel での gradual rollout**:
```bash
# 新バージョンをデプロイ（まだトラフィック 0%）
vercel deploy

# 新しい deployment を取得してエイリアス（カスタムドメイン）に部分的に割り当て
vercel alias set <new-deployment-url> example.com --percent 10
# → 10% のリクエストが新バージョンに

# 観測後、徐々に増やす
vercel alias set <new-deployment-url> example.com --percent 50
vercel alias set <new-deployment-url> example.com --percent 100
```

**Feature Flag による段階展開**:
```tsx
// app/page.tsx
import { getFlag } from '@/lib/feature-flags';

export default async function HomePage() {
  const showNewDashboard = await getFlag('new-dashboard', {
    userId: await getUserId(),
    rolloutPercent: 10,    // ユーザーの 10% に展開
  });

  return showNewDashboard ? <NewDashboard /> : <OldDashboard />;
}
```

**GrowthBook での運用例**:
```ts
// lib/feature-flags.ts
import { GrowthBook } from '@growthbook/growthbook';

const gb = new GrowthBook({
  apiHost: 'https://cdn.growthbook.io',
  clientKey: process.env.NEXT_PUBLIC_GROWTHBOOK_KEY,
  enableDevMode: process.env.NODE_ENV !== 'production',
});

await gb.init();

export function isFeatureEnabled(name: string, attributes: Record<string, unknown> = {}): boolean {
  gb.setAttributes(attributes);
  return gb.isOn(name);
}

// 使用
const newCheckoutEnabled = isFeatureEnabled('new-checkout', {
  userId: user.id,
  plan: user.plan,
  country: user.country,
});
```

**カナリア観測項目**:
| 指標 | 閾値 | 対応 |
|---|---|---|
| エラー率 | base + 0.1pp | 即時 rollback |
| Core Web Vitals (LCP/INP/CLS) | base から悪化 5% 以上 | rollback or 調査 |
| API レスポンス時間 | base + 20% 以上 | 調査 |
| Conversion rate | base から低下 5% 以上 | rollback |
| Crash rate | base + 0.05% 以上 | 即時 rollback |

**自動 rollback の閾値設定**（Vercel）:
```yaml
# vercel.json
{
  "git": {
    "deploymentEnabled": {
      "main": true
    }
  }
}
```

Vercel Analytics の閾値で自動 rollback は現時点（2026）では UI ベース。LaunchDarkly や Optimizely は API ベースで完全自動化が可能。

**カナリアと A/B テストの違い**:
- **カナリア**: 全ユーザーに展開する前の安全装置（パフォーマンス・エラー観測）
- **A/B テスト**: ビジネス指標（conversion / engagement）を観測してどちらが良いか決める実験

両者は併用可能だが目的が違うので変数も分ける。

**出典**:
- [Vercel: Atomic Deployments](https://vercel.com/docs/deployments/atomic-deployments) (Vercel)
- [GrowthBook Docs](https://docs.growthbook.io/) (GrowthBook)
- [LaunchDarkly: Progressive Delivery](https://launchdarkly.com/blog/progressive-delivery-a-history-condensed/) (LaunchDarkly)
- [Google SRE Book: Canarying Releases](https://sre.google/workbook/canarying-releases/) (Google)

**バージョン**: パターン
**確信度**: 高
**最終更新**: 2026-05-16

---

### 3. 即時ロールバック手段を確立する

5 分以内に本番を元に戻せる手段を、コマンド・GitHub Action・Slack ボタンのいずれかで用意する。
インシデント時に「どうやって戻すか」を考える時間をゼロにする。

**根拠**:
- 障害時の MTTR（Mean Time To Recovery）はビジネスインパクトに直結
- 慌てて手動デプロイすると別の障害を引き起こす
- 事前に rollback 手順を runbook 化し、定期演習で習熟しておく
- Vercel の Instant Rollback、Cloudflare の Deployment History、Git revert + redeploy 等の選択肢

**Vercel の Instant Rollback**:
```bash
# 過去デプロイの一覧
vercel ls

# 過去のデプロイを production に promote
vercel promote <deployment-url>

# UI からは Deployments タブで [Promote to Production] ボタン
```

**特徴**:
- 既にビルド済みの artifact を再利用、ビルド不要で 30 秒以内に切り替え
- DNS は変更しない（atomic deployment）
- データベース schema 変更がない限り完全に安全

**GitHub Actions による rollback workflow**:
```yaml
# .github/workflows/rollback.yml
name: Rollback Production
on:
  workflow_dispatch:
    inputs:
      target_sha:
        description: 'Commit SHA to rollback to'
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    environment: production  # 承認フロー必須
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ inputs.target_sha }}

      - name: Find Vercel deployment
        id: find
        run: |
          DEPLOYMENT=$(curl -s -H "Authorization: Bearer ${{ secrets.VERCEL_TOKEN }}" \
            "https://api.vercel.com/v6/deployments?meta-githubCommitSha=${{ inputs.target_sha }}&limit=1" \
            | jq -r '.deployments[0].url')
          echo "url=$DEPLOYMENT" >> $GITHUB_OUTPUT

      - name: Promote to production
        run: |
          curl -X POST \
            -H "Authorization: Bearer ${{ secrets.VERCEL_TOKEN }}" \
            "https://api.vercel.com/v9/projects/${{ secrets.VERCEL_PROJECT_ID }}/promote/${{ steps.find.outputs.url }}"

      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "🚨 Production rolled back to ${{ inputs.target_sha }} by ${{ github.actor }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_INCIDENT_WEBHOOK }}
```

**Git revert + redeploy**:
```bash
# 直近の問題コミットを revert
git revert <bad-commit-sha>
git push origin main
# → 通常の deploy パイプラインで自動デプロイ（2-5 分かかる）
```

Git revert は「rollback の事実が履歴に残る」点で監査追跡しやすい。

**DB マイグレーションとの整合性**:
- Schema 変更（DROP COLUMN 等）は **backward incompatible** な変更を伴う場合がある
- expand-contract pattern: 「先に追加 → アプリ切替 → 後で削除」の 2 step リリースに分割
- マイグレーションのロールバックは慎重に（データロスのリスク）

**Runbook テンプレート**:
```markdown
# Production Rollback Runbook

## Trigger conditions
- Error rate > 1% for 5 minutes
- LCP P75 > 4 seconds for 10 minutes
- Critical bug reported by users

## Steps (5-minute target)
1. Verify the issue in [Sentry](https://sentry.io/...)
2. Open [Vercel Dashboard](https://vercel.com/...)
3. Find the last known-good deployment
4. Click [Promote to Production]
5. Notify #incidents in Slack
6. Open a post-mortem ticket within 24h

## Escalation
- On-call: PagerDuty
- Tech Lead: Slack DM
- Backend impact: notify backend team

## Verification
- [ ] Error rate dropped to baseline
- [ ] Site responds with old version
- [ ] Sentry shows new errors stopped
```

**判断軸**:
| 状況 | 手段 |
|---|---|
| Frontend のみのバグ、DB 変更なし | Vercel Promote |
| バックエンド連携の問題 | 後方互換 deploy + feature flag off |
| Schema 変更のロールバック | DB 復旧手順を別途実施 |
| 機密漏洩・セキュリティ | 即時 promote + secret rotation |

**出典**:
- [Vercel: Instant Rollback](https://vercel.com/docs/deployments/managing-deployments#instant-rollback) (Vercel)
- [Google SRE Book: Postmortem Culture](https://sre.google/sre-book/postmortem-culture/) (Google)
- [Atlassian: Incident management](https://www.atlassian.com/incident-management) (Atlassian)

**バージョン**: パターン
**確信度**: 高
**最終更新**: 2026-05-16

---

### 4. Release Notes を自動生成し、ユーザー向けに公開する

リリースノート（CHANGELOG）はコミット履歴から自動生成する。
内部向けの詳細と、ユーザー向けの「機能追加 / 改善 / 修正」を分けて発信する。

**根拠**:
- 手動 CHANGELOG は更新漏れが起きやすい（merge 時に追記を忘れる）
- Conventional Commits または Changesets でメタ情報を残しておけば自動生成可能
- 内部向け（開発者）と外部向け（ユーザー）で要求される情報粒度が違う
- ユーザー向けは「何が嬉しくなったか」、内部向けは「何のコミットが入ったか」

**Changesets による自動生成**:
```
# .changeset/cool-feature.md
---
"@example/web": minor
---

Add support for dark mode in the dashboard
```

`pnpm changeset version` 実行時:
```markdown
# CHANGELOG.md
## 1.5.0

### Minor Changes

- ab12345: Add support for dark mode in the dashboard (#1234)

### Patch Changes

- def67890: Fix dropdown alignment in Safari (#1235)
```

**ユーザー向け公開**（マーケサイトの What's new ページ）:
```tsx
// app/changelog/page.tsx
import { parseChangelog } from '@/lib/changelog';

export default async function ChangelogPage() {
  const releases = await parseChangelog('CHANGELOG.md');

  return (
    <main>
      <h1>What's new</h1>
      {releases.map((release) => (
        <section key={release.version}>
          <h2>v{release.version} <time>{release.date}</time></h2>
          {release.features.length > 0 && (
            <>
              <h3>✨ New features</h3>
              <ul>{release.features.map((f) => <li key={f}>{f}</li>)}</ul>
            </>
          )}
          {release.fixes.length > 0 && (
            <>
              <h3>🐛 Fixes</h3>
              <ul>{release.fixes.map((f) => <li key={f}>{f}</li>)}</ul>
            </>
          )}
        </section>
      ))}
    </main>
  );
}
```

**GitHub Release との連携**:
```yaml
# release が作成されたら自動で GitHub Release を作成
- uses: changesets/action@v1
  with:
    publish: pnpm release
    createGithubReleases: true
```

**Slack / Discord 通知**:
```yaml
- name: Notify Slack on release
  if: steps.changesets.outputs.published == 'true'
  uses: slackapi/slack-github-action@v1
  with:
    payload-file-path: ./release-notes.json
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_RELEASE_WEBHOOK }}
```

**ユーザー向けと内部向けの分離**:
```
# Internal CHANGELOG（自動生成、すべての commit）
## 1.5.0
- feat(dashboard): Add dark mode (#1234)
- fix(safari): Fix dropdown position (#1235)
- chore(deps): Bump react to 18.3.1 (#1236)
- refactor(api): Extract user service (#1237)

# User-facing release notes（手動・厳選）
## 1.5.0 - ダークモード対応 🌙

ダッシュボードでダークモードが使えるようになりました。
ヘッダーの 🌙 アイコンから切り替え可能です。

### バグ修正
- Safari でドロップダウンの位置がズレる問題を修正
```

**Linear / Productboard と連携**:
- 製品開発タスクが Linear で管理されている場合、リリース時に該当チケットを自動 close
- Productboard / Roadmap ツールで「リリース済み」を自動マーク

**判定軸**:
- 内部 changelog → 完全自動、全コミット入れる
- 外部向け → 自動生成をベースに、人間が選別 + 文体調整
- 大規模機能 → 別途 blog 記事

**出典**:
- [Changesets: Changelog format](https://github.com/changesets/changesets/blob/main/docs/intro-to-using-changesets.md) (Changesets)
- [Keep a Changelog](https://keepachangelog.com/) (Olivier Lacan)
- [Conventional Commits](https://www.conventionalcommits.org/) (Conventional Commits)

**バージョン**: パターン
**確信度**: 高
**最終更新**: 2026-05-16
