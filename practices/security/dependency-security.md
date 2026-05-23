# 依存セキュリティのベストプラクティス

> **役割分担**: このファイルは **脆弱性スキャン / supply chain attack 対策 / 緊急アップデート / SBOM** を扱う。
> **lockfile 戦略 / packageManager 固定 / 通常の更新ポリシー**は [`dx/dependency-management.md`](../dx/dependency-management.md) を参照。
> 「正しく固定する」は dx 側、「壊れた依存を検知・封じ込める」は security 側 という分担。

npm エコシステムは supply chain attack の最大の標的。lockfile・脆弱性監査・自動更新・SBOM の 4 点セットで運用する。

## ルール

### 1. `npm audit` / `pnpm audit` を CI に組み込み、高深刻度を fail させる

PR / main ブランチへの push 時に `audit` を実行し、`high` 以上の脆弱性があればビルドを失敗させる。
新規依存追加時に脆弱性が紛れ込むのを CI でブロックする。

**根拠**:
- 既知の脆弱性は CVE データベースに記録され、npm registry の audit API で照会できる
- `high` / `critical` レベルは即時対応が必要。`moderate` / `low` は別途トリアージ
- CI でブロックすると「気づかないまま脆弱性のあるバージョンをリリース」を防げる
- ただし audit は false positive も多い。`audit-ci` / `audit-resolver` で例外管理する

**CI 設定例**:
```yaml
# .github/workflows/security.yml
name: Security Audit
on:
  pull_request:
  schedule:
    - cron: '0 0 * * *'  # 毎日 0:00 UTC に main ブランチを再監査

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile

      # high 以上で fail
      - name: Audit production dependencies
        run: pnpm audit --prod --audit-level=high

      # dev 含めての監査も別ジョブで（warning のみ）
      - name: Audit all dependencies (warn only)
        run: pnpm audit --audit-level=moderate || true
```

**例外管理（false positive・修正不能な脆弱性）**:
```json
// audit-ci.json
{
  "high": true,
  "critical": true,
  "allowlist": [
    "GHSA-xxxx-yyyy-zzzz"
  ],
  "report-type": "summary"
}
```

```bash
# audit-ci を実行
npx audit-ci --config audit-ci.json
```

**運用ルール**:
- allowlist に追加するときは **理由・対応期限を必ずコメント**
- `pnpm audit --json` で JSON 出力し Datadog / Slack に通知してダッシュボード化
- 毎週 audit log をレビューする slot を sprint に組み込む

**`npm audit` の出力解釈**:
| 深刻度 | 対応 |
|---|---|
| **Critical** | 即時対応（24h 以内）。例: RCE、認証バイパス |
| **High** | 当該 sprint で対応 |
| **Moderate** | 次の sprint で対応 |
| **Low** | バージョン上げのタイミングで吸収 |

**出典**:
- [npm audit](https://docs.npmjs.com/cli/v10/commands/npm-audit) (npm Docs)
- [pnpm audit](https://pnpm.io/cli/audit) (pnpm Docs)
- [audit-ci](https://github.com/IBM/audit-ci) (IBM)
- [OWASP: Vulnerable and Outdated Components](https://owasp.org/Top10/A06_2021-Vulnerable_and_Outdated_Components/) (OWASP Top 10 A06)

**バージョン**: npm 8+, pnpm 8+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 2. Renovate / Dependabot で自動 PR、定期マージサイクルを確立する

依存更新は人手で追わず Renovate または Dependabot の自動 PR で行う。
patch / minor は自動マージ、major は手動レビュー、というルールで運用する。

**根拠**:
- 依存更新を後回しにすると複数バージョンの差分が積み重なり、メジャーアップグレードが地獄になる
- 自動 PR があれば「次やればいい」を防ぎ、コミット履歴で更新可否を判断できる
- Renovate は npm 以外（GitHub Actions・Docker・Terraform）も統合管理できる
- Dependabot は GitHub 公式、設定がシンプル。Renovate は柔軟だが設定が複雑

**Renovate 設定例**（推奨）:
```json
// renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended",
    "group:allNonMajor",          // patch/minor は 1 PR にまとめる
    ":semanticCommits"
  ],
  "schedule": ["before 9am on monday"],
  "timezone": "Asia/Tokyo",
  "labels": ["dependencies"],
  "rangeStrategy": "bump",

  "packageRules": [
    {
      "matchUpdateTypes": ["patch", "minor"],
      "matchDepTypes": ["devDependencies"],
      "automerge": true,
      "automergeType": "branch"
    },
    {
      "matchPackageNames": ["next", "react", "react-dom"],
      "matchUpdateTypes": ["major"],
      "addLabels": ["framework-update"],
      "reviewers": ["team:frontend-leads"]
    },
    {
      "matchPackageNames": ["typescript"],
      "rangeStrategy": "pin"        // TS は version pinning
    }
  ],

  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security"],
    "schedule": ["at any time"]    // 脆弱性は即時 PR
  }
}
```

**Dependabot 設定例**:
```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Asia/Tokyo"
    open-pull-requests-limit: 10
    groups:
      dev-dependencies:
        dependency-type: "development"
        update-types: ["patch", "minor"]
    labels: ["dependencies"]

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule: { interval: "monthly" }
```

**自動マージの安全策**:
- CI（typecheck・test・build）がすべて通った場合のみ自動マージ
- `@types/*` パッケージは自動マージ OK
- `lockfile maintenance` も自動マージ OK
- 本体依存（next / react / typescript）の major は手動

**メジャー更新のリスクと対応**:
1. CHANGELOG / Migration Guide を読む
2. preview deploy で動作確認
3. Visual Regression Test の差分を確認
4. 段階導入（feature flag・1 ルートだけ先行）

**判断軸（自動マージしてよいパッケージ）**:
- TypeScript 型ファイル（`@types/*`）
- Lint / Format ツール（patch / minor）
- 開発専用ツール（vitest / playwright の patch / minor）
- 確実な breaking change なしのライブラリ（patch のみ）

**出典**:
- [Renovate Docs](https://docs.renovatebot.com/) (Mend)
- [Dependabot Docs](https://docs.github.com/en/code-security/dependabot) (GitHub)
- [Snyk: Automated dependency updates](https://snyk.io/blog/automate-fixes-with-snyks-prs/) (Snyk)

**バージョン**: Renovate v37+, Dependabot v2
**確信度**: 高
**最終更新**: 2026-05-16

---

### 3. lockfile を必ずコミットし、`--frozen-lockfile` で再現性を担保する

`package-lock.json` / `pnpm-lock.yaml` / `yarn.lock` をリポジトリに含め、CI では `--frozen-lockfile` で完全一致を強制する。
ローカルと本番で異なるバージョンが入る事故を防ぐ。

**根拠**:
- `package.json` の `^1.2.3` は「1.2.3 ≤ x < 2.0.0」を許容する。lockfile がないと CI ごとに異なるバージョンが入る
- supply chain attack（典型例: dependency confusion）への耐性も lockfile で部分的に得られる
- 攻撃者が同名のパッケージを高バージョンで publish しても、lockfile が古いバージョンの hash を固定していれば拾わない
- `packageManager` フィールドで CLI バージョンも固定すると、Corepack 経由でツールも統一できる

**コマンド対応表**:
| ツール | ローカル install | CI install |
|---|---|---|
| npm | `npm install` | `npm ci` |
| pnpm | `pnpm install` | `pnpm install --frozen-lockfile` |
| yarn | `yarn install` | `yarn install --immutable`（v3+） |
| bun | `bun install` | `bun install --frozen-lockfile` |

**`package.json` での固定**:
```json
{
  "name": "myapp",
  "packageManager": "pnpm@9.12.0",
  "engines": {
    "node": ">=20.0.0",
    "pnpm": ">=9.0.0"
  },
  "scripts": {
    "preinstall": "npx only-allow pnpm"
  }
}
```

**Corepack で CLI バージョン固定**:
```bash
# Node.js 16.9+ に同梱、experimental だが普及済み
corepack enable
corepack use pnpm@9.12.0
# → packageManager フィールドに従って自動的に正しいバージョンを使う
```

**CI で `--frozen-lockfile` を強制**:
```yaml
# .github/workflows/ci.yml
- uses: pnpm/action-setup@v3
  # version は package.json の packageManager から自動取得

- uses: actions/setup-node@v4
  with:
    node-version: 20
    cache: pnpm

- run: pnpm install --frozen-lockfile  # lockfile が package.json と乖離していれば fail
- run: pnpm typecheck
- run: pnpm test
- run: pnpm build
```

**lockfile merge conflict の対処**:
- 手動で編集しない（lockfile は人手で書くファイルではない）
- 競合したらどちらか一方を採用してから `pnpm install` で再生成
- `pnpm install --fix-lockfile` でツールに任せる

**判断（どのパッケージマネージャーを選ぶか）**:
- **pnpm**（推奨デフォルト）: disk 使用量最小、リンク経由のインストールで速い、モノレポ対応良好
- **npm**: Node.js 同梱、追加インストール不要、CI で安定
- **yarn (Berry)**: Plug'n'Play で `node_modules` を使わない設計、上級者向け
- **bun**: 速度最強だが、まだ実戦投入のリスク残る

**出典**:
- [npm: package-lock.json](https://docs.npmjs.com/cli/v10/configuring-npm/package-lock-json) (npm Docs)
- [pnpm: pnpm-lock.yaml](https://pnpm.io/git#lockfiles) (pnpm Docs)
- [Node.js: Corepack](https://nodejs.org/api/corepack.html) (Node.js Docs)
- [OWASP: Dependency Confusion](https://owasp.org/www-pdf-archive/OWASP-Dependency-Confusion.pdf) (OWASP)

**バージョン**: pnpm 8+, Node.js 20+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 4. Supply chain attack 対策（install scripts と lockfile lint）

npm パッケージの `postinstall` 等の install script は任意のコードが実行される。
信頼できないパッケージや CI 環境では `--ignore-scripts` で無効化し、必要なものだけ明示的に許可する。

**根拠**:
- npm の install script は過去に supply chain attack の経路として何度も悪用されている（`ua-parser-js` 2021, `colors`/`faker` 2022, `event-stream` 2018 等）
- CI 環境では install script を実行しないだけで攻撃の大部分を防げる
- pnpm の `onlyBuiltDependencies` で「必要な script のみ allowlist」運用が現実的
- lockfile-lint で lockfile の整合性（resolved URL・integrity hash）も検証する

**`--ignore-scripts` の運用**:
```bash
# 全 install script を無効化
pnpm install --ignore-scripts

# pnpm の場合は package.json で永続設定
# .npmrc
ignore-scripts=true
```

**pnpm v9+ の onlyBuiltDependencies**:
```json
// package.json
{
  "pnpm": {
    "onlyBuiltDependencies": [
      "esbuild",
      "sharp",
      "node-sass"
    ]
  }
}
// → このリスト以外の postinstall は実行されない
```

**lockfile-lint で URL を制限**:
```yaml
# .github/workflows/lockfile-lint.yml
- run: npx lockfile-lint \
    --path pnpm-lock.yaml \
    --type pnpm \
    --validate-https \
    --allowed-hosts registry.npmjs.org \
    --validate-integrity
```

**判断軸（install script を許可するか）**:
| パッケージ種別 | install script | 理由 |
|---|---|---|
| ネイティブ拡張（sharp / node-sass / esbuild） | 許可 | プラットフォーム別バイナリのダウンロード |
| 開発ツール（husky / lint-staged） | 慎重に | git hooks 自動セットアップは便利だが影響範囲大 |
| 通常のライブラリ | 拒否 | install 時にコード実行する正当な理由は稀 |
| Polyfill / shim 系 | 拒否 | これらが script を持つのは怪しい |

**Supply chain attack の検知パターン**:
1. **typosquatting**: `react-router-dom-v6` のような似た名前のパッケージ
2. **dependency confusion**: 社内パッケージと同名を npm public に publish
3. **maintainer takeover**: 既存パッケージのメンテナアカウント乗っ取り
4. **preinstall / postinstall malware**: install 時に環境変数や `~/.ssh` を流出
5. **AI ツール設定への永続化**: `.claude/settings.json`（hooks経由）・`.vscode/tasks.json` を汚染し、credential 窃取スクリプトを常駐させる
6. **tag/version redirection**: フォーク先の悪意あるコミットにタグを付け直し、npm / Composer 等のレジストリ経由で配布する（2026-05 Laravel Lang 攻撃で確認）——パッケージの信頼境界はブラウザで見るソースリポジトリではなく、コミットを成果物にする一連の仕組み（タグ・CI・レジストリへの push）にある

**defense in depth**:
- `npm audit` + `pnpm audit`（Rule 1）
- `--frozen-lockfile`（Rule 3）
- `--ignore-scripts`（このルール）
- `socket-security` / `snyk` / `Aikido` 等の OSS supply chain スキャナーを CI に追加
- Renovate の `:disablePeerDependencies` などで不要な依存追加を制限
- **release-age gate**: パッケージマネージャーに最小公開経過時間を設定し、公開直後のゼロデイ汚染を回避する（pnpm 11 はデフォルト 24h 待機）

```bash
# npm 11.10+: プロジェクト単位で最低公開 24h 経過を要求
npm config set min-release-age=1 --location=project

# yarn 4.10+: 1440分（24h）
yarn config set npmMinimalAgeGate 1d

# pnpm 10.16+: 1440分（24h）
pnpm config set --location=project minimumReleaseAge 1440
```

```yaml
# Dependabot (.github/dependabot.yml) でも冷却期間を設定
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "daily"
    cooldown:
      default-days: 1
```

**社内パッケージの dependency confusion 対策**:
```
# .npmrc
@mycompany:registry=https://npm.pkg.github.com
# 社内 scope は GitHub Packages から、それ以外は public npm から取得
```

**出典**:
- [npm: Avoiding npm substitution attacks](https://github.blog/security/supply-chain-security/avoiding-npm-substitution-attacks/) (GitHub Blog)
- [Socket: npm supply chain security](https://socket.dev/) (Socket)
- [Snyk: npm security best practices](https://snyk.io/blog/ten-npm-security-best-practices/) (Snyk Blog)
- [pnpm: Settings - onlyBuiltDependencies](https://pnpm.io/package_json#pnpmonlybuiltdependencies) (pnpm Docs)
- [Protecting your Node.js project against supply-chain attacks](https://dev.to/douglasdemoura/protecting-your-nodejs-project-against-supply-chain-attacks-5984) (dev.to、release-age gate の npm/yarn/pnpm 設定例) ※2026-05-17に実際にfetch成功
- [Mini Shai-Hulud Hits AntV: 300+ Malicious npm Packages via Compromised Maintainer Account](https://snyk.io/blog/mini-shai-hulud-antv-npm-supply-chain-attack/) (Snyk Blog、preinstall hook攻撃 + Claude Code session hooks永続化の新手口) ※2026-05-20に実際にfetch成功
- [Laravel Lang Supply Chain Advisory](https://snyk.io/blog/laravel-lang-supply-chain-advisory/) (Snyk Blog、Composerタグリダイレクション攻撃・信頼境界の原則) ※2026-05-23に実際にfetch成功

> "A package's trust boundary is not the source repository in your browser tab. It is the chain of systems that decides which commit becomes a published artifact."
> ([Laravel Lang Supply Chain Advisory](https://snyk.io/blog/laravel-lang-supply-chain-advisory/), Snyk Blog, セクション "Incident Analysis") ※2026-05-23に実際にfetch成功

> "Delaying dependency resolution gives the ecosystem time to catch bad versions before your project installs them."
> ([Protecting your Node.js project against supply-chain attacks](https://dev.to/douglasdemoura/protecting-your-nodejs-project-against-supply-chain-attacks-5984), dev.to) ※2026-05-17に実際にfetch成功

> "If you are uncertain whether you were affected, treat it as a confirmed compromise. The RSA-encrypted exfiltration means you cannot recover what was taken."
> ([Mini Shai-Hulud Hits AntV: 300+ Malicious npm Packages via Compromised Maintainer Account](https://snyk.io/blog/mini-shai-hulud-antv-npm-supply-chain-attack/), Snyk Blog) ※2026-05-20に実際にfetch成功

**バージョン**: npm 11.10+ / yarn 4.10+ / pnpm 10.16+
**確信度**: 高
**最終更新**: 2026-05-23

#### 追加根拠 (2026-05-16)

新たに以下の記事でサプライチェーン攻撃発生後の即時対応手順が明確化された:
- [Malicious node-ipc versions published to npm in suspected maintainer account compromise](https://snyk.io/blog/malicious-node-ipc-versions-published-npm/) (Snyk Blog / 2026-05-15) ※2026-05-16に実際にfetch成功
- [Shai Hulud攻撃から身を守る：npm脆弱性と対策ガイド](https://zenn.dev/7788/articles/93ff5ceaba0576) (Zenn / 2026-05-15) ※2026-05-16に実際にfetch成功

**出典引用**:
> "By preserving expected functionality while quietly harvesting credentials, the attacker increased the chance that compromised builds would continue operating normally."
> ([Malicious node-ipc versions published to npm](https://snyk.io/blog/malicious-node-ipc-versions-published-npm/), セクション "Attack analysis")

> 「単なるマルウェア配布ではなく開発インフラへの深刻な侵害」「パッケージ削除後もIDEに残存」
> ([Shai Hulud攻撃から身を守る：npm脆弱性と対策ガイド](https://zenn.dev/7788/articles/93ff5ceaba0576), セクション "永続的な感染")

**侵害検知時の即時対応チェックリスト**（node-ipc / TanStack compromise 事例より）:

```bash
# 1. 影響確認
npm ls <対象パッケージ>
snyk test
```

- [ ] **npm トークン**（`~/.npmrc` の authToken）を即座に失効・再生成
- [ ] **GitHub トークン**（PAT, Deploy Keys, GitHub Actions secrets）を全て rotate
- [ ] **クラウドクレデンシャル**（AWS IAM, GCP SA key, Azure SP）を rotate
- [ ] **SSH 秘密鍵**を rotation（特に deploy keys）
- [ ] **Vault / Kubernetes トークン**を rotation
- [ ] **IDE 設定フォルダの不審ファイルを確認**（`~/.vscode/`, `~/.config/claude/` 等）
  - パッケージ削除後もマルウェアが IDE に残存するケースが確認されている
- [ ] CI の npm/pnpm キャッシュをクリア（汚染タールボールを除去）
- [ ] C2 ドメインを DNS/proxy レベルでブロック（判明次第）
- [ ] Cloud / GitHub のアクセスログを遡及確認

**追加の知見（メンテナアカウント乗っ取りの経路）**:
- MFA 強制化だけでなく、メンテナの **期限切れメールドメイン**の再取得でも乗っ取り可能
- 定期的に npm パッケージのメンテナメールアドレスが有効かを確認する運用が重要

**確信度**: 既存（高）→ 高（実証付き・即時対応手順追加）

---

### 5. SBOM（Software Bill of Materials）を生成して依存関係を可視化する

リリース時に SBOM を生成し、利用しているすべての依存パッケージ・ライセンス・バージョンを記録する。
脆弱性が新規発見されたとき、影響範囲を即座に特定できる。

**根拠**:
- 大規模な脆弱性（log4shell / xz-utils）発生時に「自分の組織のどのアプリが影響を受けるか」を 1 時間で答えられるかが BCP の差
- SBOM は CycloneDX / SPDX 形式が標準。CISA や OpenSSF が推奨
- `@cyclonedx/cyclonedx-npm` 等でビルドパイプラインに組み込める
- 米国大統領令 14028 で連邦政府ベンダーに SBOM 提出義務化、業界全体で普及中

**SBOM 生成**:
```yaml
# .github/workflows/sbom.yml
name: Generate SBOM
on:
  release:
    types: [created]

jobs:
  sbom:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - run: pnpm install --frozen-lockfile

      # CycloneDX 形式
      - name: Generate CycloneDX SBOM
        run: npx @cyclonedx/cyclonedx-npm --output-file sbom.cyclonedx.json

      # SPDX 形式（必要に応じて）
      - uses: anchore/sbom-action@v0
        with:
          path: ./
          format: spdx-json
          output-file: sbom.spdx.json

      # アーティファクトとして保存
      - uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.*.json
```

**SBOM の中身（CycloneDX 抜粋）**:
```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.5",
  "components": [
    {
      "type": "library",
      "name": "react",
      "version": "18.3.1",
      "purl": "pkg:npm/react@18.3.1",
      "licenses": [{ "license": { "id": "MIT" } }],
      "hashes": [{ "alg": "SHA-512", "content": "..." }]
    }
  ]
}
```

**脆弱性発見時の運用**:
```bash
# 例: CVE-2024-XXXXX が発覚
# 全リリースの SBOM を grep して影響するアプリを特定
for sbom in releases/*/sbom.cyclonedx.json; do
  if jq -e '.components[] | select(.name == "vulnerable-pkg" and .version == "1.2.3")' "$sbom"; then
    echo "AFFECTED: $sbom"
  fi
done
```

**Grype / Trivy で SBOM をスキャン**:
```bash
# CycloneDX 形式の SBOM を直接スキャン
grype sbom:./sbom.cyclonedx.json --fail-on high
trivy sbom ./sbom.cyclonedx.json
```

**ライセンスコンプライアンス**:
```bash
# license-checker でライセンス監査
npx license-checker --production --excludePrivatePackages --json > licenses.json

# 禁止ライセンスを検出
npx license-checker --failOn 'GPL;AGPL;LGPL' --production
```

**運用の段階**:
1. **Stage 1**: SBOM をリリースアーティファクトとして保存（履歴ができる）
2. **Stage 2**: 自動スキャン（Grype / Trivy）を CI に組み込み
3. **Stage 3**: 脆弱性 DB 連携で常時監視（Snyk / Dependabot Alerts）
4. **Stage 4**: 顧客への SBOM 提供（B2B SaaS で要求される）

**出典**:
- [CycloneDX Specification](https://cyclonedx.org/specification/overview/) (OWASP)
- [SPDX Specification](https://spdx.dev/) (Linux Foundation)
- [CISA: Software Bill of Materials](https://www.cisa.gov/sbom) (CISA)
- [Anchore: SBOM Action](https://github.com/anchore/sbom-action) (Anchore)
- [Grype](https://github.com/anchore/grype) (Anchore)

**バージョン**: CycloneDX 1.5+, SPDX 2.3+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 6. 開発ツール拡張機能（VS Code等）のサプライチェーンリスクを管理する

VS Code 拡張機能はインストール時から Node.js プロセスとして開発者権限で実行され、
`process.env`（`TOKEN`/`SECRET`/`KEY`等）・ファイルシステム・CLI ツールへのアクセスが可能。
悪意ある拡張機能や汚染されたアップデートが配布されると、開発環境の認証情報が流出する。
拡張機能は npm パッケージとは別のサプライチェーンリスクとして、導入・更新を審査する。

**根拠**:
- 2026-05-22 に確認された事例: Nx Console v18.95.0（VS Code 拡張機能）に悪意ある
  コードが混入し、GitHub の内部リポジトリ約 3,800 件が 11〜18 分で窃取された（TeamPCP 攻撃）
- 拡張機能の `activate()` は VS Code 起動のたびに実行され、継続的なクレデンシャル収集が可能
- インストールのみで即座に env 変数・ワークスペースのファイルへアクセスできる
- 「有名ツールだから安全」という前提が攻撃の起点になる

**コード例（悪意ある拡張機能の仕組み）**:
```typescript
// 悪意ある拡張機能の activate() のパターン（参考）
export function activate(context: vscode.ExtensionContext): void {
  const folders = vscode.workspace.workspaceFolders ?? [];
  // TOKEN / SECRET / KEY / PASSWORD にマッチする環境変数を収集
  const keyNames = Object.keys(process.env)
    .filter((key) => /(TOKEN|SECRET|KEY|PASSWORD)/i.test(key));
  // 外部サーバーへ送信...
}
```

**審査・防御チェックリスト**:
- [ ] インストール前に **Publisher の信頼性を確認**（公式 Publisher か・評価数・更新頻度）
- [ ] **自動更新を無効化**または更新前にリリースノートを確認（Marketplace での手動レビュー）
- [ ] **VS Code Profiles / Workspace Trust** で作業コンテキスト別に拡張機能を分離
- [ ] **Restricted Mode** を使用して外部ソースのコードに対してはデフォルトで制限
- [ ] `.vsix` を Marketplace 経由でなく手動インストールする場合は **事前にパッケージ内容を解凍・確認**
- [ ] 未使用の拡張機能は定期的に**アンインストール**（攻撃対象領域を最小化）

**侵害検知 IOC（Nx Console / TeamPCP 攻撃の例）**:
```bash
# macOS: 感染インジケーターを確認
ls ~/.local/share/kitty/cat.py 2>/dev/null && echo "INFECTED: cat.py found"
ls ~/Library/LaunchAgents/com.user.kitty-monitor.plist 2>/dev/null && echo "INFECTED: plist found"

# 侵害が確認された場合の対応
# → GitHub トークン・AWS キーを即時失効し Rule #4 の侵害チェックリストを実行
```

**出典引用**:
> "VS Code extensions run as Node.js processes with developer permissions from the moment of installation, allowing access to sensitive information."
> ([VS Code 拡張機能のインストールだけで開発環境が侵害されるリスク — デモと対策](https://zenn.dev/hisa_tech_2973/articles/51b62bd8c3bd11), セクション "VS Code 拡張機能の仕組みとリスクの構造") ※2026-05-22に実際にfetch成功

> 「攻撃者はこのリリースに悪意のあるコードを仕込み、Marketplace 上に公開しました」「『有名なツールだから大丈夫』という常識が、一瞬で覆された瞬間です」
> ([たった11分で3,800リポジトリが流出——GitHubを陥落させたVS Code拡張機能サプライチェーン攻撃](https://zenn.dev/esta_dev/articles/fa0269c791ced3), セクション "侵入経路：「毒入りVS Code拡張機能」が仕掛けた罠") ※2026-05-22に実際にfetch成功

**アンチパターン**:
- 「有名ツールの拡張機能だから自動更新で問題ない」→ maintainer アカウント乗っ取りでいつでも汚染可能
- npm `--ignore-scripts` を設定しながら VS Code 拡張機能は無審査 → 攻撃経路が残る

**バージョン**: VS Code 全バージョン
**確信度**: 高
**最終更新**: 2026-05-22

---

### 7. MCPサーバーとAIコーディングエージェントの設定ファイルを独立したサプライチェーンリスクとして管理する

MCP（Model Context Protocol）サーバーは npm パッケージや VS Code 拡張機能とは異なる攻撃経路を持つ。
悪意ある MCP サーバーはツール定義のメタデータにプロンプトインジェクションを埋め込み、
AI が承認なしに意図しない操作を実行させる（ツールポイズニング）。
また `.mcp.json`・`.claude/settings.json` などの設定ファイル自体が攻撃ベクターとして悪用される
（TrustFall 攻撃で RCE が確認: CVE-2025-59536, CVE-2026-21852）。

**根拠**:
- MCP では「データの中に命令が混ざり得る」ため、外部ソース経由のコンテンツはすべて潜在的な攻撃経路
- read-access（ファイル読み取り）と external-send（外部送信）MCP を同時に有効化すると間接プロンプトインジェクションで情報流出が起きる
- 設定ファイルはパーミッションが評価される「前」に処理されるケースがある（TrustFall の仕組み）
- 承認ダイアログは「approval fatigue」を意図した UI デザインが存在し、ユーザーが内容を確認せずに承認してしまう
- postmark-mcp 経由の感染事例で約300組織が影響を受けた（2026年確認）

**MCP リスク管理チェックリスト**:
- [ ] MCP サーバーは公式・実績あるものだけ使用し、未知ソースからはインストールしない
- [ ] read-access MCP（ファイル読み取り等）と external-send MCP（外部送信等）を同時に有効化しない
- [ ] OAuth スコープは最小権限にする（`drive` ではなく `drive.readonly`、`calendar` ではなく `calendar.readonly`）
- [ ] `.mcp.json` や `.claude/settings.json` をプロジェクトに含める場合はコードレビューと同じ基準で内容を審査する
- [ ] 外部リポジトリの clone 時は `.mcp.json` の内容を確認してから Claude Code / Cursor を起動する
- [ ] `allowedTools` / `permissions` は Managed Settings で組織統制する（Rule #13 参照）

**3つの主要攻撃パターン**:

| パターン | 仕組み | 対策 |
|---|---|---|
| 間接プロンプトインジェクション | 文書・Web コンテンツに埋め込まれた命令を AI が実行 | 外部コンテンツを読む MCP と送信する MCP を分離 |
| ツールポイズニング | ツール定義メタデータに悪意あるプロンプトを仕込む | インストール前にツール定義の内容を確認 |
| Confused Deputy | AI の委任権限を悪用して未認可操作を実行 | 承認ダイアログの内容を毎回確認（「常に許可」を使わない） |

**出典引用**:
> "承認制はあっても、UIの設計次第でその意味が失われる"
> ([MCPを使う前に知っておくべきこと――便利さの裏にある攻撃の仕組み](https://zenn.dev/masuda_masuo/articles/2026-05-23-mcp-security), セクション "承認の限界") ※2026-05-23に実際にfetch成功

> "The attack surface is not the model, but configuration files that no one reviews"
> ([AIコーディングエージェントの本当の攻撃面は設定ファイルだった](https://zenn.dev/ju571n/articles/ai-agent-config-attack-surface), セクション "TrustFall Attack") ※2026-05-23に実際にfetch成功

**アンチパターン**:
- 知人作の MCP サーバーだからと内容を確認せずにインストールする（rug-pull: 承認後に悪意あるコードへ差し替えられる可能性）
- 承認ダイアログを習慣的に「常に許可」でクリアする
- `.mcp.json` が含まれるリポジトリをそのまま Claude Code で開く

**バージョン**: Claude Code / Cursor / MCP Protocol 全バージョン
**確信度**: 高
**最終更新**: 2026-05-23
