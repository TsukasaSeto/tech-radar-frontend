# 依存セキュリティのベストプラクティス

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
4. **postinstall malware**: install 時に環境変数や `~/.ssh` を流出

**defense in depth**:
- `npm audit` + `pnpm audit`（Rule 1）
- `--frozen-lockfile`（Rule 3）
- `--ignore-scripts`（このルール）
- `socket-security` / `snyk` / `Aikido` 等の OSS supply chain スキャナーを CI に追加
- Renovate の `:disablePeerDependencies` などで不要な依存追加を制限

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

**バージョン**: pnpm 9+
**確信度**: 高
**最終更新**: 2026-05-16

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
