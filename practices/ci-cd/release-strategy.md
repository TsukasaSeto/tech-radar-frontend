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
- npm 公開は Trusted Publishing（OIDC）にすると `NPM_TOKEN` を保存せずに済む。publish ジョブに `permissions.id-token: write` を付けるだけで OIDC トークンを取得でき、`contents: write`（バージョン更新コミット・タグ・Release）と `pull-requests: write`（Version Packages PR）はジョブ単位で付与し、ワークフロー既定は `contents: read` にする。パッケージが未公開だと Trusted Publisher を設定できないため、初回バージョンだけは手動公開が必要

**選択肢**:

| ツール | 適用 | 強み |
|---|---|---|
| **Changesets** | モノレポ・複数パッケージ | パッケージ単位のバージョニング、UI わかりやすい |
| **semantic-release** | 単一パッケージ | Conventional Commits から完全自動 |
| **release-please** | 単一 / モノレポ | Google 製、GitHub 統合強力 |
| **changelogen** | シンプルな単一パッケージ | UnJS 製、軽量 |

**Changesets 設定**:
```bash
# @changesets/cli 3.x は Node ^22.11 / ^24 / >=26、pnpm 10 以上が前提
# config.json で指定する changelog 生成器（@changesets/changelog-github）も一緒に入れる。入れないと changeset version が module not found で落ちる
pnpm add -D @changesets/cli @changesets/changelog-github
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

# 実行中の release は止めない。待機できる run は最新の 1 件だけで、間の run は置き換えられる
# （Changesets は最新の main の状態を処理するので問題ない。`practices/ci-cd/github-actions.md` #2）
concurrency: ${{ github.workflow }}-${{ github.ref }}

permissions:
  contents: read

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write        # バージョン更新コミット・タグ・GitHub Release
      pull-requests: write   # "Version Packages" PR
      id-token: write        # npm Trusted Publishing（OIDC）。NPM_TOKEN は不要
    steps:
      - uses: actions/checkout@v7
        with: { persist-credentials: false }
      - uses: actions/setup-node@v7
        with:
          node-version: 24     # Trusted Publishing は npm CLI 11.5.1+ / Node 22.14.0+ が前提
          registry-url: 'https://registry.npmjs.org'
      # 実運用ではサードパーティ action を SHA でピン留めする（`practices/ci-cd/github-actions.md` #9）
      # リリースのビルドでは依存キャッシュを使わない（npm 公式の Trusted Publishing の例と同じ方針）
      - uses: pnpm/setup@v3
        with: { node-version-file: false, install: false }
      - run: pnpm install --frozen-lockfile

      - name: Create Release PR or Publish
        id: changesets
        uses: changesets/action@v2
        with:
          # 未消化の changeset があれば "Version Packages" PR を作る
          # 全 changeset 消化済み + version bump がある場合は publish + tag 作成
          # v2 で入力名が変わった（v1 の version / publish / commit / title は使えない）
          version-script: pnpm changeset version
          publish-script: pnpm release  # = changeset publish + 必要なら npm publish
          commit-message: 'chore: release packages'
          pr-title: 'chore: release packages'
          # GitHub トークンは既定で github.token。別のトークンは env ではなく github-token 入力で渡す
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
- [Changesets と npm Trusted Publishing を利用して npm パッケージ公開フローを作った](https://zenn.dev/su3_hokkaido/articles/2026-10-24_publish_npm_package) (Zenn、Changesets + OIDC 公開の `permissions` 構成と初回手動公開の制約。Qiita に同一著者のクロスポストあり) ※2026-10-24 fetch
- [npm Docs: Trusted publishing](https://docs.npmjs.com/trusted-publishers) (npm 公式、npm CLI 11.5.1+ / Node 22.14.0+、`id-token: write`、リリースビルドではキャッシュを使わない例) ※2026-10-25 確認
- [changesets/action README（v2）](https://github.com/changesets/action/blob/v2/README.md) (Changesets 公式、v2 は Changesets v3 対応。入力名 `version-script` / `publish-script` / `commit-message` / `pr-title`、必要な `permissions`) ※2026-10-25 確認

> "これがあるとジョブが OIDC トークンを取得でき、npm へのログインがトークンなしで済みます。"
> ([Changesets と npm Trusted Publishing を利用して npm パッケージ公開フローを作った](https://zenn.dev/su3_hokkaido/articles/2026-10-24_publish_npm_package), セクション "4. リリース用のワークフローを書く") ※2026-10-24に実際にfetch成功

**バージョン**: Changesets v3（@changesets/cli 3.x）/ changesets/action v2（v1 は Changesets v2 向け）/ npm Trusted Publishing は npm CLI 11.5.1+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 2. カナリアリリース戦略（Branch Deploy / Feature Flag）

100% のユーザーに一気にリリースせず、最初は 1-10% に絞って観測する。
Vercel の rolling release、Cloudflare の gradual rollout、または Feature Flag（LaunchDarkly / Statsig / GrowthBook）を活用する。

**根拠**:
- 大規模変更を全ユーザーに即時展開するとバグ波及が大きい
- 1-10% → 50% → 100% と段階展開で、エラー率・パフォーマンスを観測してから広げる
- Feature Flag なら code レベルで切り替えられ、re-deploy なしで rollback できる
- 「カナリア = 炭鉱のカナリア（先に異常を検知する）」が語源

**Vercel での gradual rollout（Rolling Releases）**:

Vercel の段階配信は Rolling Releases で行う（Pro / Enterprise。Pro は 1 プロジェクトまで）。`vercel alias set` に `--percent` オプションはなく、未知のオプションは使用法エラー（終了コード 2）になる。Rolling Releases を有効にすると、本番への promote（Git push での自動 promote、`vercel promote`、ダッシュボードの Promote）が最初のステージから始まる。クライアントとサーバーの版がずれないよう、Skew Protection の併用が推奨されている。

```bash
# Bad: vercel alias set <new-deployment-url> example.com --percent 10（存在しないオプション）

# Good: Rolling Releases を設定する（10% → 50% → 100% を手動承認で進める。最後の 100% は自動で追加される）
vercel rolling-release configure --enable --advancement-type=manual-approval --stage=10 --stage=50

# 本番向けにビルドだけして（ドメインは割り当てない）、CI から明示的に開始する
DEPLOYMENT_URL=$(vercel deploy --prod --skip-domain)
vercel rolling-release start --dpl="$DEPLOYMENT_URL" --yes
# → 10% のリクエストが新バージョンに

# 観測後、次のステージへ進める（currentStageIndex は今のステージの番号）
vercel rolling-release approve --dpl="$DEPLOYMENT_URL" --currentStageIndex=0
# 問題があれば中止する（プロジェクトは Instant Rollback と同じロールバック状態になる）
vercel rolling-release abort --dpl="$DEPLOYMENT_URL"
```

CI から開始・完了するときは `vercel rolling-release start` / `complete` を使い、`vercel promote` は使わない（進行中の rolling release に promote すると 100% まで一気に完了する）。

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

**GrowthBook での運用例**（サーバー側）:

サーバーでは 1 つのクライアントを全リクエストで共有するため、ユーザーの属性をクライアントに持たせてはいけない。`GrowthBook` の単一インスタンスに `setAttributes` で属性を差し替える書き方は、並行するリクエストが同じ状態を共有し、非同期処理を挟むと他のユーザーの属性で評価されうる。マルチユーザー向けの `GrowthBookClient` を使い、評価のたびにユーザーのコンテキストを渡す（または `createScopedInstance` でリクエストごとのインスタンスを作る）。

```ts
// lib/feature-flags.ts（サーバー側で使う）
import { GrowthBookClient } from '@growthbook/growthbook';

const gbClient = new GrowthBookClient({
  apiHost: 'https://cdn.growthbook.io',
  clientKey: process.env.GROWTHBOOK_CLIENT_KEY,
});

// 起動時に 1 回だけ初期化し、各リクエストは完了を待つだけにする
const ready = gbClient.init({ timeout: 3000 });

// Bad: 共有インスタンスに gb.setAttributes(attributes) してから gb.isOn(name) する
// Good: 共有クライアントには状態を持たせず、評価ごとにユーザーのコンテキストを渡す
export async function isFeatureEnabled(name: string, attributes: Record<string, unknown> = {}): Promise<boolean> {
  await ready;
  return gbClient.isOn(name, { attributes });
}

// 使用
const newCheckoutEnabled = await isFeatureEnabled('new-checkout', {
  id: user.id,
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

**指標に基づく中止・ロールバック**（Vercel）:
- Rolling Releases の進行中は、Observability タブでカナリアと現行版の指標（Speed Insights 等）を比べられる。中止は Rolling Releases ページの Abort、`vercel rolling-release abort`、または Instant Rollback で行う
- 外部の監視基盤の指標で判断する場合は、カナリアと現行版を区別できるよう Vercel のデプロイ ID を監視側に伝播させる
- LaunchDarkly や Optimizely は API ベースで完全自動化が可能

**カナリアと A/B テストの違い**:
- **カナリア**: 全ユーザーに展開する前の安全装置（パフォーマンス・エラー観測）
- **A/B テスト**: ビジネス指標（conversion / engagement）を観測してどちらが良いか決める実験

両者は併用可能だが目的が違うので変数も分ける。

**CloudFront Functions + KeyValueStore でのカナリア（Vercel を使わない静的配信構成）**:
S3 + CloudFront で静的アセットを配信する構成では、Vercel の Rolling Releases は使えない。同等のことを CloudFront Function（`viewer-request`）+ KeyValueStore（KVS）で実現できる。S3 側を slot1/slot2 の2オリジンに分け、KVS の `active_origin`（本番スロット）・`canary_weight`（カナリア比率 0-100）をエッジ側から低レイテンシで参照してルーティングする。AWS 標準の「CloudFront Continuous Deployment」機能を使わない理由は、それが Distribution 本体の状態を直接書き換えるため、CloudFormation 管理下では IaC ドリフトを起こすから。クライアントの IP ハッシュで振り分けを決定的にし、同一クライアントがセッション中に新旧バージョンを行き来しないようにする。キャッシュ汚染を避けるため、Function が付与した `x-canary-slot` リクエストヘッダーをキャッシュポリシーのキャッシュキーに含め、スロットごとにキャッシュを分離する。

障害時のフォールバック先を固定のスロット（例: 常に `slot1`）にすると、昇格後（`active_origin` が `slot2`）は KVS 障害のたびに全ユーザーが旧版やカナリア版に流れる。フォールバックは「検証済みの本番スロット」に向ける。KVS 自体を読めないときは Function でオリジンを選ばず、キャッシュビヘイビアの既定オリジンに任せ、既定オリジンは昇格のたびに IaC で本番スロットに揃える（IaC に反映するまでの間は旧スロットに戻りうるので、旧スロットの成果物は次の昇格まで消さない）。`canary_weight` だけ読めないときはカナリアを止める。ハッシュ関数は Function 内に実装する（CloudFront Functions の JavaScript runtime 2.0 は `Math.imul` 等の ES6 の Math メソッドに対応している）:
```javascript
// CloudFront Function（viewer-request、JavaScript runtime 2.0）
import cf from 'cloudfront';

// FNV-1a（32bit）でクライアント IP を 0-99 のバケットに割り当てる（同じ IP は常に同じバケット）
function bucketOf(value) {
  let hash = 0x811c9dc5;
  for (let i = 0; i < value.length; i++) {
    hash ^= value.charCodeAt(i);
    hash = Math.imul(hash, 0x01000193) >>> 0;
  }
  return hash % 100;
}

async function handler(event) {
  const request = event.request;
  const kvsHandle = cf.kvs();

  let activeOrigin;
  try {
    activeOrigin = await kvsHandle.get('active_origin');
  } catch (e) {
    // Bad: 固定の 'slot1' を選ぶ（昇格後は本番スロットではない）
    // Good: オリジンを選ばず、IaC で本番スロットに揃えたキャッシュビヘイビアの既定オリジンに任せる
    return request;
  }

  let canaryWeight = 0;
  try {
    // KVS の値は文字列なので数値に変換する
    canaryWeight = Number(await kvsHandle.get('canary_weight')) || 0;
  } catch (e) {
    canaryWeight = 0; // 比率を読めなければカナリアを止め、本番スロットだけに流す
  }

  let slot = activeOrigin;
  if (canaryWeight > 0 && bucketOf(event.viewer.ip) < canaryWeight) {
    slot = activeOrigin === 'slot1' ? 'slot2' : 'slot1';
  }

  request.headers['x-canary-slot'] = { value: slot };
  cf.selectRequestOriginById(slot);
  return request;
}
```
KVS の更新は `aws cloudfront-keyvaluestore put-key --if-match "${etag}"` で ETag による楽観ロックを効かせ、GitHub Actions を「カナリアへデプロイ（`canary_weight` を10に設定）」「本番昇格（承認ゲート付き、`active_origin` を切替＆`canary_weight` を0に戻し、キャッシュビヘイビアの既定オリジンも IaC で新しい本番スロットに揃える）」「ロールバック（承認却下時に `if: failure()` で `canary_weight` のみ0に戻し、`active_origin` は変更しない）」の3ジョブに分ける。

**出典**:
- [Vercel: Atomic Deployments](https://vercel.com/docs/deployments/atomic-deployments) (Vercel)
- [GrowthBook Docs](https://docs.growthbook.io/) (GrowthBook)
- [LaunchDarkly: Progressive Delivery](https://launchdarkly.com/blog/progressive-delivery-a-history-condensed/) (LaunchDarkly)
- [Google SRE Book: Canarying Releases](https://sre.google/workbook/canarying-releases/) (Google)
- [CloudFront Functions × KeyValueStoreで実現するCloudFront + S3のカナリアリリース](https://techblog.zozo.com/entry/cloudfront-functions-kvs-canary-release) (ZOZO TECH BLOG、S3 + CloudFront 静的配信構成でのIPハッシュ決定的ルーティングとキャッシュキー分離の実装) ※2026-08-18に実際にfetch成功
- [Vercel Docs: Rolling Releases](https://vercel.com/docs/rolling-releases) (Vercel 公式、対象プラン・promote で開始する挙動・CI では start / complete を使う・中止方法・Skew Protection の推奨) ※2026-10-25 確認
- [Vercel CLI: vercel rolling-release](https://vercel.com/docs/cli/rolling-release) (Vercel 公式、`configure` / `start` / `approve` / `abort` / `complete`) ※2026-10-25 確認
- [Vercel CLI: Global Options](https://vercel.com/docs/cli/global-options) (Vercel 公式、未知のオプションは終了コード 2 の使用法エラー) ※2026-10-25 確認
- [GrowthBook Docs: Node.js SDK](https://docs.growthbook.io/lib/node) (GrowthBook 公式、マルチユーザー向けの `GrowthBookClient` と評価ごとのユーザーコンテキスト・`createScopedInstance`) ※2026-10-25 確認
- [CloudFront Functions: JavaScript runtime 2.0 features](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/functions-javascript-runtime-20.html) (AWS 公式、対応する言語機能と `Math.imul` 等) ※2026-10-25 確認

**出典引用**:
> "CloudFront KeyValueStore（以下、KVS）は、CloudFront Functionsから低レイテンシで参照できるkey-value形式のデータストアです。値の更新はDistributionのデプロイなしに、通常数秒でエッジに反映されます。"
> ([CloudFront Functions × KeyValueStoreで実現するCloudFront + S3のカナリアリリース](https://techblog.zozo.com/entry/cloudfront-functions-kvs-canary-release), セクション "CloudFront Functions + KeyValueStoreの採用理由") ※2026-08-18に実際にfetch成功

**バージョン**: パターン（Vercel Rolling Releases は Pro / Enterprise、CloudFront Functions は JavaScript runtime 2.0）
**確信度**: 高
**最終更新**: 2026-10-25

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

# 過去の本番デプロイに戻す（Instant Rollback。Hobby は直前の本番デプロイにだけ戻せる）
vercel rollback <deployment-id-or-url>

# 過去の本番デプロイを promote しても戻せる
# ただし Preview 向けのデプロイを promote すると本番向けの新しいデプロイが作られ、ビルド不要の即時切り替えにならない
vercel promote <deployment-url>

# ロールバック後は本番ドメインの自動割り当てが止まり、main への push は本番に出なくなる
# 修正版ができたら promote してロールバック状態を解除する
vercel promote <fixed-deployment-url>

# UI からは Production Deployment タイルの [Instant Rollback]、解除は [Undo Rollback]
```

**特徴**:
- 対象は過去に本番ドメインへ割り当てられたデプロイ。ビルド済みの成果物を再利用するため、ビルド不要で 30 秒以内に切り替わる
- DNS は変更しない（atomic deployment）
- 「完全に安全」ではない。戻した版が今の環境と両立するかを先に確認する
  - 環境変数: ロールバックは当時のビルドに戻るだけで、プロジェクト設定で変えた環境変数は反映されない（ローテーション後の秘密情報も旧版には入らない）
  - DB スキーマ・データ形式: 旧版のコードが新しいスキーマやデータを読めるか（後述の expand-contract）
  - API 契約・外部サービス: バックエンド API・CMS・外部 API が旧版のリクエスト形式を今も受け付けるか
  - その他の設定: cron ジョブの定義は戻した版のものになる。カスタムエイリアスは、戻す前の本番デプロイに付いていたものしか含まれない
- 実運用規模での検証例: 6,000万ユーザー・50万件の動的ページを配信する Speechify は Vercel の Instant Rollback を採用し、2〜3日おき（時に日2回）のデプロイ頻度でも「悪いリリースをユーザーに気づかれる前に戻せる」前提でリリースできている

**GitHub Actions による rollback workflow**:
```yaml
# .github/workflows/rollback.yml
name: Rollback Production
on:
  workflow_dispatch:
    inputs:
      target_sha:
        description: 'Commit SHA of the production deployment to roll back to'
        required: true

permissions: {}

jobs:
  rollback:
    runs-on: ubuntu-latest
    environment: production  # 承認フロー必須。VERCEL_TOKEN はこの environment の secret に置く
    env:
      VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}         # CLI は環境変数から読む（--token で引数に出さない）
      VERCEL_ORG_ID: ${{ vars.VERCEL_ORG_ID }}          # CI ではこの 2 つを設定すると vercel link が不要になる
      VERCEL_PROJECT_ID: ${{ vars.VERCEL_PROJECT_ID }}
    steps:
      - uses: actions/setup-node@v7
        with: { node-version: 24 }
      - run: npm install -g vercel@65.2.0   # CLI のバージョンは固定し、Renovate 等で更新する

      - name: Find production deployment
        id: find
        env:
          TARGET_SHA: ${{ inputs.target_sha }}
        run: |
          # Bad: curl -s で検索条件に target を付けない（HTTP エラーでも成功扱い、preview を拾うと promote が再ビルドになる）
          # Good: -fsS で HTTP エラーを失敗にし、本番・READY・ロールバック可能なデプロイに絞る。見つからなければ jq -e で失敗させる
          # 個人アカウントのプロジェクトなら teamId は付けない
          DEPLOYMENT_ID=$(curl -fsS -H "Authorization: Bearer ${VERCEL_TOKEN}" \
            "https://api.vercel.com/v7/deployments?projectId=${VERCEL_PROJECT_ID}&teamId=${VERCEL_ORG_ID}&target=production&state=READY&rollbackCandidate=true&sha=${TARGET_SHA}&limit=1" \
            | jq -er '.deployments[0].uid')
          echo "id=${DEPLOYMENT_ID}" >> "$GITHUB_OUTPUT"

      - name: Roll back
        env:
          DEPLOYMENT_ID: ${{ steps.find.outputs.id }}
        run: |
          # 切り替えの完了を待ち、失敗すれば非ゼロ終了でジョブを止める（後続の通知に進まない）
          vercel rollback "$DEPLOYMENT_ID" --timeout=5m
          vercel rollback status

      - name: Notify Slack
        uses: slackapi/slack-github-action@v4   # 実運用では SHA でピン留めする（`practices/ci-cd/github-actions.md` #9）
        with:
          webhook: ${{ secrets.SLACK_INCIDENT_WEBHOOK }}
          webhook-type: incoming-webhook
          payload: |
            text: "🚨 Production rolled back to ${{ inputs.target_sha }} by ${{ github.actor }} (auto-assignment of production domains is now off until the next promote)"
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
3. Find the last known-good production deployment
4. Click [Instant Rollback] (or run the rollback workflow)
5. Notify #incidents in Slack
6. After the fix is deployed, promote it to undo the rollback (re-enables auto-assignment of production domains)
7. Open a post-mortem ticket within 24h

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
| Frontend のみのバグ、DB・API・環境変数の変更なし | Vercel Instant Rollback（`vercel rollback`） |
| バックエンド連携の問題 | 後方互換 deploy + feature flag off |
| Schema 変更のロールバック | DB 復旧手順を別途実施 |
| 機密漏洩・セキュリティ | 即時 rollback + secret rotation（ローテーション後の値は新しいデプロイで反映する） |

**出典**:
- [Vercel: Instant Rollback](https://vercel.com/docs/deployments/managing-deployments#instant-rollback) (Vercel)
- [Vercel Docs: Performing an Instant Rollback on a Deployment](https://vercel.com/docs/instant-rollback) (Vercel 公式、対象デプロイ・環境変数が更新されない・cron とカスタムエイリアス・自動割り当ての停止と Undo) ※2026-10-25 確認
- [Vercel CLI: vercel rollback](https://vercel.com/docs/cli/rollback) / [vercel promote](https://vercel.com/docs/cli/promote) (Vercel 公式、`--timeout`・`status`・Preview 向けデプロイを promote すると新しい本番デプロイになる) ※2026-10-25 確認
- [Vercel REST API: List deployments](https://vercel.com/docs/rest-api/reference/endpoints/deployments/list-deployments) (Vercel 公式、`GET /v7/deployments` の `projectId` / `target` / `state` / `rollbackCandidate` / `sha`) ※2026-10-25 確認
- [Google SRE Book: Postmortem Culture](https://sre.google/sre-book/postmortem-culture/) (Google)
- [Atlassian: Incident management](https://www.atlassian.com/incident-management) (Atlassian)
- [How Speechify serves 500,000 dynamic pages to 60 million users on Vercel](https://vercel.com/blog/how-speechify-serves-50000-dynamic-pages-to-60-million-users-on-vercel) (Vercel 公式ブログ、Instant Rollback の大規模実運用における効果を示す事例) ※2026-07-16 fetch

**出典引用**:
> "Vercel's instant rollbacks mean you fix it before anyone notices."
> ([How Speechify serves 500,000 dynamic pages to 60 million users on Vercel](https://vercel.com/blog/how-speechify-serves-50000-dynamic-pages-to-60-million-users-on-vercel), セクション "Instant Rollbacks") ※2026-07-16に実際にfetch成功

**バージョン**: パターン（Vercel CLI の例は 65.x）
**確信度**: 高
**最終更新**: 2026-10-25

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
# release が作成されたら自動で GitHub Release を作成（v2 の入力名。既定で true）
- name: Create Release PR or Publish
  id: changesets          # 後続の steps.changesets.outputs.published で参照するため必須
  uses: changesets/action@v2
  with:
    publish-script: pnpm release
    create-github-releases: true
```

**Slack / Discord 通知**:
```yaml
- name: Notify Slack on release
  # Bad: changesets/action のステップに id がないと steps.changesets は空で、通知は一度も送られない
  if: steps.changesets.outputs.published == 'true'
  uses: slackapi/slack-github-action@v4   # 実運用では SHA でピン留めする（`practices/ci-cd/github-actions.md` #9）
  with:
    webhook: ${{ secrets.SLACK_RELEASE_WEBHOOK }}
    webhook-type: incoming-webhook
    payload-file-path: ./release-notes.json
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
- [Slack: Sending data with an incoming webhook](https://docs.slack.dev/tools/slack-github-action/sending-data-slack-incoming-webhook/) (Slack 公式、v4 の `webhook` / `webhook-type` / `payload` / `payload-file-path`) ※2026-10-25 確認

**バージョン**: パターン（changesets/action v2 / slackapi/slack-github-action v4）
**確信度**: 高
**最終更新**: 2026-10-25
