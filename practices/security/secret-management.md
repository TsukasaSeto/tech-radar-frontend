# シークレット管理のベストプラクティス

API キー・DB 接続文字列・OAuth クレデンシャル等を「サーバーだけが持つ」状態を保ち、クライアントバンドルに漏れないようにする。

## ルール

### 1. `NEXT_PUBLIC_` プレフィックスでクライアント・サーバー境界を明示する

Next.js では `NEXT_PUBLIC_` プレフィックスの環境変数のみクライアントバンドルに含まれる。
プレフィックスなしの環境変数は **サーバー専用**。混同しないようコマンド規約を統一する。

**根拠**:
- 認証情報をうっかり `NEXT_PUBLIC_*` で公開する事故が業界全体で頻発（GitHub の secret scanning でも検出される）
- プレフィックスがビルド時に文字列置換される仕組みのため、ビルド成果物に含まれるかどうかは `NEXT_PUBLIC_` で完全に決まる
- Next.js 13+ では `import 'server-only'` で「サーバー専用モジュール」を強制でき、クライアントから誤って import するとビルドエラーになる
- Vite・Remix にも同等の規約（`VITE_*` / `PUBLIC_*` ）がある

**コード例**:
```bash
# .env.local
# 公開 OK（クライアントで使用）
NEXT_PUBLIC_APP_URL=https://example.com
NEXT_PUBLIC_ANALYTICS_ID=G-XXXXX
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...

# サーバー専用（クライアント不可）
DATABASE_URL=postgres://...
STRIPE_SECRET_KEY=sk_test_...
JWT_SIGNING_KEY=...
OAUTH_CLIENT_SECRET=...
```

```ts
// lib/db.ts — サーバー専用
import 'server-only';  // クライアントから import するとビルドエラー
import { PrismaClient } from '@prisma/client';

export const db = new PrismaClient({
  datasources: { db: { url: process.env.DATABASE_URL! } },
});
```

```tsx
// Client Component で誤って import すると Next.js がビルドエラー
'use client';
import { db } from '@/lib/db';  // ← error: 'server-only' module imported from client

// Good: クライアントは公開キーのみ使う
'use client';
import { loadStripe } from '@stripe/stripe-js';
const stripe = loadStripe(process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY!);
```

**TypeScript で env を型付け**:
```ts
// env.d.ts
declare namespace NodeJS {
  interface ProcessEnv {
    // Public
    NEXT_PUBLIC_APP_URL: string;
    NEXT_PUBLIC_ANALYTICS_ID: string;
    // Server-only
    DATABASE_URL: string;
    STRIPE_SECRET_KEY: string;
    JWT_SIGNING_KEY: string;
  }
}
```

**`@t3-oss/env-nextjs` を使う場合**（推奨）:
```ts
// env.ts
import { createEnv } from '@t3-oss/env-nextjs';
import { z } from 'zod';

export const env = createEnv({
  server: {
    DATABASE_URL: z.string().url(),
    STRIPE_SECRET_KEY: z.string().startsWith('sk_'),
  },
  client: {
    NEXT_PUBLIC_APP_URL: z.string().url(),
    NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY: z.string().startsWith('pk_'),
  },
  runtimeEnv: {
    DATABASE_URL: process.env.DATABASE_URL,
    STRIPE_SECRET_KEY: process.env.STRIPE_SECRET_KEY,
    NEXT_PUBLIC_APP_URL: process.env.NEXT_PUBLIC_APP_URL,
    NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY: process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY,
  },
});

// 使用: env.DATABASE_URL は server-only かつ型安全
```

**出典**:
- [Next.js Docs: Environment Variables](https://nextjs.org/docs/app/building-your-application/configuring/environment-variables) (Next.js 公式)
- [Next.js Docs: server-only](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns#keeping-server-only-code-out-of-the-client-environment) (Next.js 公式)
- [t3-oss/env-nextjs](https://env.t3.gg/) (t3-oss)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 2. ビルド成果物にシークレットが混入していないか CI で検査する

ビルドした `.next/static/**/*.js` を grep して秘密情報が含まれていないかを CI ジョブで検査する。
人手のレビュー漏れと運用ミスを機械的に防ぐ。

**根拠**:
- `process.env.X` をクライアントコードで参照していると、`X` が `NEXT_PUBLIC_` でなくても `process.env.X` のリテラル文字列がバンドルに残ることがある（ビルド時置換されないため undefined になるが、コード自体は残る）
- 開発者が `console.log(process.env.SECRET)` のような debug コードを残したまま push する事故
- TruffleHog / gitleaks 等のツールがクライアントバンドルから secret パターンを発見できる
- `secret_lint` や `actionlint` も合わせて運用

**CI 設定例**:
```yaml
# .github/workflows/secret-leak-check.yml
name: Secret Leak Check
on: [pull_request]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci && npm run build

      # 1. ビルド済み JS にシークレット候補がないか grep
      - name: Grep build for known secret prefixes
        run: |
          if grep -rEn "sk_(live|test)_|rk_(live|test)_|password=|AKIA[0-9A-Z]{16}" .next/static/; then
            echo "ERROR: secret pattern found in client bundle"
            exit 1
          fi

      # 2. TruffleHog で広範な検査
      - uses: trufflesecurity/trufflehog@main
        with:
          path: ./.next/static
          extra_args: --only-verified

      # 3. リポジトリ全体にも secret scanner（コミット時の漏洩）
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**自前で検出するパターン例**:
```ts
// scripts/check-bundle.ts
import { readdir, readFile } from 'node:fs/promises';
import { join } from 'node:path';

const FORBIDDEN_PATTERNS = [
  /sk_(live|test)_[a-zA-Z0-9]{24,}/, // Stripe secret
  /AKIA[0-9A-Z]{16}/,                  // AWS access key
  /ghp_[a-zA-Z0-9]{36}/,               // GitHub PAT
  /xox[abp]-[a-zA-Z0-9-]+/,            // Slack token
  /-----BEGIN (RSA |EC |DSA )?PRIVATE KEY-----/,
];

async function* walk(dir: string): AsyncGenerator<string> {
  for (const entry of await readdir(dir, { withFileTypes: true })) {
    const path = join(dir, entry.name);
    if (entry.isDirectory()) yield* walk(path);
    else if (entry.name.endsWith('.js')) yield path;
  }
}

let leaked = false;
for await (const file of walk('.next/static')) {
  const content = await readFile(file, 'utf-8');
  for (const pattern of FORBIDDEN_PATTERNS) {
    if (pattern.test(content)) {
      console.error(`Leaked: ${file} matched ${pattern}`);
      leaked = true;
    }
  }
}
process.exit(leaked ? 1 : 0);
```

**カバレッジ補足**:
- 環境変数だけでなく、Source Map にコメントで PII / コメントが残るケースもある → Source Map をクライアント配信しない
- `.env.local` を accidentally commit する事故は `.gitignore` + pre-commit hook で防ぐ
- Vercel・Cloudflare のビルドログにもシークレットを `***` で伏字化する機能あり

**出典**:
- [TruffleHog](https://github.com/trufflesecurity/trufflehog) (Truffle Security)
- [Gitleaks](https://github.com/gitleaks/gitleaks)
- [GitHub: secret scanning](https://docs.github.com/en/code-security/secret-scanning) (GitHub Docs)

**バージョン**: Next.js 13+, GitHub Actions
**確信度**: 高
**最終更新**: 2026-05-16

#### 追加根拠 (2026-05-16)

Vercel が「Protected Source Maps」機能をリリースし、ソースマップへのアクセス制御の具体的な実装手段が確立された:
- [Protected Source Maps: Ship browser source maps securely](https://vercel.com/changelog/protected-source-maps-ship-browser-source-maps-securely) (Vercel / 2026-05-14) ※2026-05-16に実際にfetch成功

**出典引用**:
> "Source maps are how you debug minified production code. They give you readable stack traces and your original source code, with the real filenames and line numbers intact."
> ([Protected Source Maps](https://vercel.com/changelog/protected-source-maps-ship-browser-source-maps-securely), セクション "Why source map protection matters")

**Protected Source Maps の設定（Vercel）**:
- **新規プロジェクト**: デフォルトで有効（自動保護、追加設定不要）
- **既存プロジェクト**: Settings → Deployment Protection → Protected Source Maps を有効化（再デプロイ不要）
- 有効化後: チームメンバー（Vercel 認証済み）のみ `.map` ファイルにアクセス可能
- 攻撃者は source map から元コードのファイル名・行番号・ロジックを逆解析できなくなる

**Vercel 以外の対応（参考）**:
```nginx
# Nginx: 本番で .map ファイルへのパブリックアクセスを無効化
location ~* \.js\.map$ {
  return 404;
  # または IP allowlist: allow 10.0.0.0/8; deny all;
}
```

**確信度**: 既存（高）→ 高（Vercel 公式機能として確立）

---

### 3. `.env.local` は git 管理外、`.env.example` をテンプレートとして提供する

`.gitignore` で `.env*.local` を完全に除外し、`.env.example` でキー名のみを共有する。
新規メンバーは `.env.example` をコピーして `.env.local` を作る運用に統一する。

**根拠**:
- `.env` の commit は frontend 業界で最頻発の事故。一度 git 履歴に入ったシークレットは rotate するまで永遠に有効
- `.env.example` を整備すると「どの環境変数が必要か」がドキュメント化され、オンボーディングが速くなる
- pre-commit フックで `.env*.local` の commit を機械的に防ぐ
- Vercel / Cloudflare Pages はダッシュボード経由で環境変数を設定し、リポジトリには入れない

**標準的な `.gitignore`**:
```gitignore
# 環境変数（ローカル）
.env
.env.local
.env.*.local

# ビルド成果物
.next/
out/
dist/
build/

# 依存
node_modules/

# IDE
.vscode/settings.json
.idea/
```

**`.env.example` テンプレート**:
```bash
# .env.example — このファイルは commit する。値は空 or ダミー

# ---- 公開（NEXT_PUBLIC_）----
NEXT_PUBLIC_APP_URL=http://localhost:3000
NEXT_PUBLIC_ANALYTICS_ID=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_xxxxx

# ---- サーバー専用 ----
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
STRIPE_SECRET_KEY=
JWT_SIGNING_KEY=  # openssl rand -base64 32 で生成

# ---- OAuth ----
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# 取得方法のコメント
# - DATABASE_URL: Vercel Postgres Dashboard > Settings > Storage
# - STRIPE_SECRET_KEY: https://dashboard.stripe.com/test/apikeys
```

**pre-commit hook で防御**:
```yaml
# .husky/pre-commit
#!/bin/sh
# .env*.local の commit を防ぐ
if git diff --cached --name-only | grep -E '\.env(\..+)?\.local$'; then
  echo "ERROR: .env*.local files cannot be committed"
  exit 1
fi

# .env のような汎用名も警告
if git diff --cached --name-only | grep -E '^\.env$'; then
  echo "WARNING: committing .env — is this intentional?"
  exit 1
fi
```

**事故った時の対応**:
1. シークレットを **即座に rotate**（git history から消しても、すでに漏れた値は永久に流出している前提）
2. git history からの削除は `git filter-repo` または BFG Repo-Cleaner で
3. force push 後、リポジトリの他クローンから再取り込みされないよう全員に伝達
4. Vercel / Cloudflare 等の deployment 履歴のログも確認（過去のビルドログに残っていないか）

**ドキュメント化（README）**:
```markdown
## 環境変数のセットアップ

1. `.env.example` を `.env.local` にコピー
2. 各値を以下のドキュメントから取得して埋める：
   - DATABASE_URL: [Confluence: DB セットアップ](https://...)
   - STRIPE_*: [Stripe Dashboard](https://dashboard.stripe.com/test/apikeys)
   - JWT_SIGNING_KEY: `openssl rand -base64 32` で生成

3. `.env.local` は **絶対に commit しない**
```

**出典**:
- [dotenv-safe](https://github.com/rolodato/dotenv-safe) — 必須環境変数のチェック
- [Next.js Docs: Environment Variables](https://nextjs.org/docs/app/building-your-application/configuring/environment-variables#test-environment-variables) (Next.js 公式)
- [GitHub: Removing sensitive data](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository) (GitHub Docs)

**バージョン**: 一般原則
**確信度**: 高
**最終更新**: 2026-05-16

---

### 4. Secret Manager を採用し、本番で平文 `.env` を使わない

production 環境では Vercel Environment Variables / AWS Secrets Manager / Doppler / Infisical 等のシークレット管理サービスを使う。
平文ファイル（`.env.production`）でシークレットを管理しない。

**根拠**:
- 平文 `.env.production` は EC2 / VM に置くと OS レベルでアクセスできる人が広がる
- Secret Manager は IAM で「誰が」「いつ」「何の値を」読んだかを監査ログ化できる
- ローテーション（key rotation）が手動でなく自動化できる
- Vercel / Netlify / Cloudflare Pages はビルド時に環境変数を注入する仕組みを提供（OS の環境変数として渡される）

**主要な選択肢**:

| サービス | 強み | 弱み |
|---|---|---|
| **Vercel Environment Variables** | Vercel デプロイなら無料・自動連携 | Vercel ロックイン |
| **AWS Secrets Manager** | IAM 完全統合、ローテーション自動化、KMS 暗号化 | コスト高め（$0.40/secret/month） |
| **Doppler** | マルチクラウド、UI 良好、無料枠あり | サードパーティ依存 |
| **Infisical** | OSS（self-host 可）、E2E 暗号化 | 運用負荷 |
| **HashiCorp Vault** | 大企業向け、機能豊富 | 学習コスト高、オーバースペック |

**Vercel での例**:
```bash
# CLI で環境変数を設定
vercel env add DATABASE_URL production
# プロンプトで値を入力（CLI 履歴に残らない）

# プロジェクト設定 → Environment Variables → Sensitive にチェック
# → ダッシュボードでも値が見えなくなる（最低限の権限分離）

# プレビュー・開発用にも個別に設定可能
vercel env add DATABASE_URL preview
vercel env add DATABASE_URL development
```

**AWS Secrets Manager + Next.js**:
```ts
// lib/secrets.ts
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

const client = new SecretsManagerClient({ region: 'ap-northeast-1' });
const cache = new Map<string, { value: string; expiresAt: number }>();

export async function getSecret(name: string): Promise<string> {
  const cached = cache.get(name);
  if (cached && cached.expiresAt > Date.now()) return cached.value;

  const res = await client.send(new GetSecretValueCommand({ SecretId: name }));
  const value = res.SecretString!;
  cache.set(name, { value, expiresAt: Date.now() + 5 * 60 * 1000 });  // 5min cache
  return value;
}

// 使用
const dbUrl = await getSecret('prod/database/url');
```

**ローテーション戦略**:
- 自動ローテーション: AWS Secrets Manager は Lambda でローテーション関数を実行可能
- 半自動: 月次で手動ローテーション + Slack 通知でリマインド
- 即時: 漏洩疑いがあれば即時ローテーション + 全環境再デプロイ

**緊急時のチェックリスト**:
- [ ] 漏洩した key を **即座に invalidate**（API 提供元のダッシュボードで revoke）
- [ ] 新しい key を Secret Manager に登録
- [ ] 全環境を再デプロイ（ビルド時に環境変数が確定するため）
- [ ] アクセスログを確認（漏洩した key で異常なアクセスがあったか）
- [ ] post-mortem を書き、再発防止策を導入

**出典**:
- [Vercel: Environment Variables](https://vercel.com/docs/environment-variables) (Vercel)
- [AWS Secrets Manager Docs](https://docs.aws.amazon.com/secretsmanager/) (AWS)
- [Doppler](https://www.doppler.com/) (Doppler)
- [Infisical](https://infisical.com/) (Infisical)

**バージョン**: パターン（実装依存）
**確信度**: 高
**最終更新**: 2026-05-16
