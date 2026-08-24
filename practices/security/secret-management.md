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

**公開値は「隠さず縛る」**:
`NEXT_PUBLIC_GOOGLE_API_KEY` のようにブラウザが必要とするキーは `NEXT_PUBLIC_` でバンドルに焼き込まれることが前提。これを「サーバー側で隠す」設計にしても、SPA のビルド成果物に必ず入るため無意味。代わりに Google Console / Stripe Dashboard 等で「縛る（制限する）」:

| 公開値の種類 | 保護方法 |
|---|---|
| Google Maps API Key | HTTP referrer 制限（自ドメインのみ許可） |
| Stripe Publishable Key `pk_` | 用途スコープ制限（決済のみ） |
| Analytics ID | クォータ・予算アラートで不正消費を検知 |

```bash
# 公開値の .env.local
NEXT_PUBLIC_GOOGLE_MAPS_KEY=AIzaSy...   # Google Console で referrer 制限必須
NEXT_PUBLIC_STRIPE_PK=pk_live_...       # スコープ制限済み

# シークレット（絶対に NEXT_PUBLIC_ にしない）
STRIPE_SK=sk_live_...
DATABASE_URL=postgres://...
```

**LLM API キーには「三層ロック」を適用する（AI 統合時の強化パターン）**:

Gemini・OpenAI 等の LLM API キーは漏洩したときのコスト爆発リスクが大きい。命名規約（`NEXT_PUBLIC_` なし）＋ビルド時強制（`import 'server-only'`）に加え、**Server Action を唯一のゲートウェイ**にすることで三層防御になる:

```typescript
// Layer 1: lib/ai/gemini.ts — server-only で build 時に import を封鎖
import 'server-only';
import { GoogleGenAI } from '@google/genai';

export async function generateContent(prompt: string) {
  const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY! });
  // ...
}

// Layer 2: app/actions/ai.ts — Server Action が唯一のゲートウェイ
'use server';
import { generateContent } from '@/lib/ai/gemini';

export async function generateAction(input: string) {
  if (!input.trim()) return { ok: false, error: 'Input required.' };
  const result = await generateContent(input);
  return { ok: true, result };
}

// Layer 3: クライアントは Server Action 経由のみ（直接 API キーにアクセス不可）
```

> "「気をつける」はスケールしませんが、「そもそもビルドが通らない」は絶対にスケールします。"
> ([AIのAPIキーがクライアントに混入したら「ビルドエラー」にする — server-onlyで作る安全なGemini統合](https://zenn.dev/shippai/articles/f1625c5587629a), セクション "設計の要点") ※2026-06-16に実際にfetch成功
**出典**:
- [Next.js Docs: Environment Variables](https://nextjs.org/docs/app/building-your-application/configuring/environment-variables) (Next.js 公式)
- [Next.js Docs: server-only](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns#keeping-server-only-code-out-of-the-client-environment) (Next.js 公式)
- [t3-oss/env-nextjs](https://env.t3.gg/) (t3-oss)
- [「API キーは全部隠せ」は勘違いだった — ブラウザに出る"公開値"とサーバだけの"秘密"の見分け方](https://qiita.com/Kazy_engineer/items/c66f3fe21f25f096f571) (Qiita Kazy_engineer、「隠す」ではなく「縛る」の原則と公開値別の保護方法) ※2026-06-15 fetch
- [AIのAPIキーがクライアントに混入したら「ビルドエラー」にする — server-onlyで作る安全なGemini統合](https://zenn.dev/shippai/articles/f1625c5587629a) (Zenn shippai、LLM API キーの三層ロックパターン) ※2026-06-16 fetch

**出典引用**:
> "「隠す」のではなく「縛る」ことで守るからだ"
> ([「API キーは全部隠せ」は勘違いだった](https://qiita.com/Kazy_engineer/items/c66f3fe21f25f096f571), セクション "公開値は縛る") ※2026-06-15に実際にfetch成功

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-06-16

---

### 2. ビルド成果物にシークレットが混入していないか CI で検査する

ビルドした `.next/static/**/*.js` を grep して秘密情報が含まれていないかを CI ジョブで検査する。
人手のレビュー漏れと運用ミスを機械的に防ぐ。

**根拠**:
- `process.env.X` をクライアントコードで参照していると、`X` が `NEXT_PUBLIC_` でなくても `process.env.X` のリテラル文字列がバンドルに残ることがある（ビルド時置換されないため undefined になるが、コード自体は残る）
- 開発者が `console.log(process.env.SECRET)` のような debug コードを残したまま push する事故
- TruffleHog / gitleaks 等のツールがクライアントバンドルから secret パターンを発見できる
- **betterleaks**（gitleaks 作者による後継ツール）は BPE トークナイザーで自然言語とシークレットを区別し、Shannon エントロピー方式（gitleaks）より高精度（CredData 評価: 98.6% vs 70.4%）
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
      # betterleaks（gitleaks 作者による後継ツール）を使うと BPE トークナイザーで
      # 検出率 98.6% vs gitleaks 70.4%（CredData 評価）を達成できる
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
- [gitleaksの作者が一から作り直したbetterleaks](https://zenn.dev/shuymn/articles/600779b488d6d3) (Zenn shuymn、BPE トークナイザーによる検出率改善) ※2026-06-08に実際にfetch成功
- [GitLeaksからBetterLeaksに乗り換えた話](https://zenn.dev/mohhh_ok/articles/gitleaks-to-betterleaks) (Zenn mohhh_ok、実際の移行事例) ※2026-06-09に実際にfetch成功

**バージョン**: Next.js 13+, GitHub Actions
**確信度**: 高
**最終更新**: 2026-06-09

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
- AI コーディング環境では作業テンポが速く注意力が有限なため、`.env` ファイルだけでなく Markdown ドキュメントや設定ファイルへの誤混入も gitleaks で機械的に検出する
- `.gitignore` の否定パターン（`!pattern`）は要注意: マッチ判定は上から順に評価され**最後にマッチした行が勝つ**ため、後続の `!pattern` が意図せず先行する `.env` 等の ignore を打ち消して include に戻すことがある。動作確認に使う `git check-ignore` は既定で index を参照するため、**追跡済みのパスに対しては `.gitignore` が壊れていても「問題なし」を返す**（`--no-index` を付けて未追跡状態から検証する必要がある）。ブラックリスト（否定パターンでの部分許可）よりも、ディレクトリ丸ごと ignore してから必要なファイルだけを明示的に許可するホワイトリスト構成の方が事故りにくい

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

**gitleaks + lefthook による広範な秘密情報検出**:

`.env` パターンだけでは検出できない Markdown ドキュメントや設定ファイルへの誤混入を、gitleaks + lefthook の組み合わせで広範に検出する。AI コーディング環境では「AIコーディングのテンポは速く、一つ前のターンで何を保留にしたか、平気で忘れます」という注意力の有限性を機械的に補う:

```yaml
# lefthook.yml
pre-commit:
  commands:
    gitleaks:
      run: gitleaks git --staged --redact --verbose
```

```toml
# .gitleaks.toml — プロジェクト固有パターン追加（デフォルトルールを継承）
[extend]
  useDefault = true

[[rules]]
  id = "japanese-secret-fields"
  description = "Japanese field names often used for secrets in any file type"
  regex = '''(パスワード|APIキー|api.?key|シークレット)\s*[:=]\s*\S+'''
  severity = "ERROR"
```

**AI コーディングエージェントによるデプロイ時の流出防止**:

AI エージェントが `netlify deploy --dir=.` のようにプロジェクトルートをそのまま指定すると、`.gitignore` 対象外のファイル（`config.py`・`credentials.json` 等）に混入した認証情報が公開される。「消したから大丈夫」は通用しない——インターネットに出た瞬間から永久に流出している前提で対応する:

```bash
# Bad: ルートをそのまま指定（.gitignore 対象外の設定ファイルも公開される）
netlify deploy --dir=.

# Good: ビルド成果物ディレクトリを明示する
netlify deploy --dir=dist
```

```
# .netlifyignore — .gitignore と独立したデプロイ除外リスト
config.py
credentials*
*.key
*.pem
.env*
```

PreToolUse フックで `deploy` コマンドの引数を実行前に検査し、`--dir=.` が指定されていたら即座にブロックする（フックの書き方は `ai-agent/claude-code.md` Rule #9 参照）。

**`.gitignore` が一切効かないケース（AI ツールが git を経由しない場合）**:

`.gitignore` は `git add` / `git commit` / `git push` が対象にするファイル範囲を決めるだけであり、AI エージェントのローカルファイルシステムスキャン・`tar`/`rsync`・Docker ビルドコンテキスト・IDE のクラウド同期機能は `.gitignore` を一切参照しない。あるビルドツールへのプロジェクトアップロード事案では、`.gitignore` されていたはずのシークレットファイルが git を経由しない全量アップロードで漏洩した（該当ログで245件のアップロードイベントを確認）。対策は「git 管理外に置く」ではなく「**リポジトリのディレクトリツリーの外に置く**」こと:

```bash
# Bad: リポジトリ直下（.gitignore 済みでも、非git経路のツールには丸見え）
project/.env.local
project/secrets/credentials.json

# Good: リポジトリツリーの外側、OS 標準の設定ディレクトリに保存
~/.config/myapp/credentials.json
# または Secret Manager 経由で環境変数として注入（Rule #4 参照）
```

リポジトリ内に残してよいのは `.env.example` のようなキー名のみのテンプレートだけとし、実値は repo tree の外側 or Secret Manager に置く。

**出典引用**:
> "AIコーディングのテンポは速く、一つ前のターンで何を保留にしたか、平気で忘れます。"
> ([AI コーディングで secret を漏らさないための４層防御](https://zenn.dev/takna/articles/secret-leak-prevention-4-layer), セクション "問題の本責") ※2026-05-17に実際にfetch成功

> "gitignore は、`git add` `git commit` `git push` が対象にするファイルの範囲を決めるものであり、それ以外の経路（AIツールのファイルシステム走査・tar/rsync・Dockerビルドコンテキスト等）は一切尊重してくれません。"
> ([gitignore は防御じゃない。Grok Build の repo アップロード事案から、秘密の置き場所を見直す](https://zenn.dev/ishizakahiroshi/articles/20260714-grok-build-repo-upload-timeline), Zenn, セクション "本題: gitignore は防御じゃない") ※2026-07-14に実際にfetch成功

**事故った時の対応**:
1. シークレットを **即座に rotate**ﾈgit history から消しても、すでに漏れた値は永久に流出している前提）
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
- [AI コーディングで secret を漏らさないための４層防御](https://zenn.dev/takna/articles/secret-leak-prevention-4-layer) (Zenn takna, gitleaks+lefthook+GitHub Push Protection+Claude Hooks による4層防御) ※2026-05-17 fetch
- [AIエージェントに deploy を任せたら設定ファイルごとデプロイされた話](https://qiita.com/yurukusa/items/c2fdcf5c0be30929b686) (Qiita yurukusa、AI deploy での --dir=. 流出事故・.netlifyignore・PreToolUse 防御) ※2026-06-28に実際にfetch成功
- [gitignore は防御じゃない。Grok Build の repo アップロード事案から、秘密の置き場所を見直す](https://zenn.dev/ishizakahiroshi/articles/20260714-grok-build-repo-upload-timeline) (Zenn ishizakahiroshi、git を経由しないアップロード経路での gitignore 無効化事案) ※2026-07-14に実際にfetch成功
- [`.gitignore` に否定行を1行足したら、`.env` と秘密鍵まで追跡対象になっていた](https://zenn.dev/rkpg/articles/gitignore-negation-unignores-secrets) (Zenn rkpg、否定パターンの評価順による事故と `git check-ignore` の index 参照による検知漏れ) ※2026-08-12に実際にfetch成功

> "「消したから大丈夫」は、ここでは通用しない"
> ([AIエージェントに deploy を任せたら設定ファイルごとデプロイされた話](https://qiita.com/yurukusa/items/c2fdcf5c0be30929b686), Qiita yurukusa, セクション "事後対応") ※2026-06-28に実際にfetch成功

> "`.gitignore` の照合は上から順で、最後にマッチした行が勝ちます。`.env` は4行目でいったん ignore になり、9行目の `!.claude/skills/logs/**` がそれを上書きして include に戻す。"
> ([`.gitignore` に否定行を1行足したら、`.env` と秘密鍵まで追跡対象になっていた](https://zenn.dev/rkpg/articles/gitignore-negation-unignores-secrets), セクション "事故") ※2026-08-12に実際にfetch成功

**バージョン**: 一般原則
**確信度**: 高
**最終更新**: 2026-08-12

---

### 4. Secret Manager を採用し、本番で平文 `.env` を使わない

production 環境では Vercel Environment Variables / AWS Secrets Manager / Doppler / Infisical 等のシークレット管理サービスを使う。
平文ファイル（`.env.production`）でシークレットを管理しない。

**根拠**:
- 平文 `.env.production` は EC2 / VM に置くと OS レベルでアクセスできる人が広がる
- Secret Manager は IAM で「誰が」「いつ」「何の値を」読んだかを監査ログ化できる
- ローテーションﾈkey rotation）が手動でなく自動化できる
- Vercel / Netlify / Cloudflare Pages はビルド時に環境変数を注入する仕組みを提供ﾈOS の環境変数として渡される）
- **サーバーレス関数の「環境変数」も要注意**: AWS Lambda 等の platform-level function configuration（Environment Variables 欄への直接設定）は永続的なデプロイメタデータとしてコンソールや IaC 定義に平文で残り、Secret Manager 経由の一時的なプロセスメモリ上の値とは性質が異なる。シークレットは実行時にランタイムで vault / caching extension から取得し、platform-level function configuration には置かない
- **Google Cloud Run + Secret Manager**: シークレット単位で `roles/secretmanager.secretAccessor` を実行時サービスアカウントに付与し、`--set-secrets` で環境変数としてマウントする。IAM 権限がなければ注入自体が発生しないため、コンテナイメージにもリポジトリにもシークレットの実体が存在しない状態を作れる（プロジェクト全体ではなくシークレット単位の最小権限）

**サーバーレス関数での取得例**:
```ts
// Bad: Lambda の環境変数欄に直接シークレットを設定（デプロイ設定に平文で残る）
const apiKey = process.env.STRIPE_SECRET_KEY;

// Good: 実行時に vault / Secrets Manager から取得（関数設定には残らない）
const apiKey = await getSecret('prod/stripe/secret-key'); // 前掲 getSecret() を利用
```

**主要な選択肢**:

| サービス | 強み | 弱み |
|---|---|---|
| **Vercel Environment Variables** | Vercel デプロイなら無料・自動連携 | Vercel ロックイン |
| **AWS Secrets Manager** | IAM 完全統合、ローテーション自動化、KMS 暗号化 | コスト高めﾈ0.40/secret/month） |
| **Doppler** | マルチクラウド、UI 良好、無料枚あり | サードパーティ依存 |
| **Infisical** | OSSﾈself-host 可）、E2E 暗号化 | 運用負荷 |
| **HashiCorp Vault** | 大企業向け、機能豊富 | 学習コスト高、オーバースペック |

**Vercel での例**:
```bash
# CLI で環境変数を設定
vercel env add DATABASE_URL production
# プロンプトで値を入力ﾈCLI 履歴に残らない）

# プロジェクト設定 → Environment Variables → Sensitive にチェック
# → ダッシュボードでも値が見えなくなるﾈ最低限の権限分離）

# プレビュー・開発用にも個別に設定可能
vercel env add DATABASE_URL preview
vercel env add DATABASE_URL development
```

**Google Cloud Run + Secret Manager**:
```bash
printf 'sk-live-ABCDEF1234567890' | gcloud secrets create api-key --data-file=-

gcloud secrets add-iam-policy-binding api-key \
  --member="serviceAccount:<PN>-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

gcloud run services update app --set-secrets="API_KEY=api-key:latest"
```
アプリ側は `os.Getenv` 等で通常の環境変数として読むだけでよく、注入の可否は IAM バインディングのみが決める。

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
- 即時: 漏洩疊いがあれば即時ローテーション + 全環境再デプロイ

**紧急時のチェックリスト**:
- [ ] 漏洩した key を **即座に invalidate**ﾈAPI 提供元のダッシュボードで revoke）
- [ ] 新しい key を Secret Manager に登録
- [ ] 全環境を再デプロイﾈビルド時に環境変数が確定するため）
- [ ] アクセスログを確認ﾈ漏洩した key で異常なアクセスがあったか）
- [ ] post-mortem を書き、再発防止策を導入

**出典**:
- [Vercel: Environment Variables](https://vercel.com/docs/environment-variables) (Vercel)
- [AWS Secrets Manager Docs](https://docs.aws.amazon.com/secretsmanager/) (AWS)
- [Doppler](https://www.doppler.com/) (Doppler)
- [Infisical](https://infisical.com/) (Infisical)
- [OWASP CheatSheetSeries: Serverless secrets - platform config, not env vars](https://github.com/OWASP/CheatSheetSeries/commit/ae0b3f22f5b24381455f742af9b5fa84063ad770) (OWASP 公式、Serverless/FaaS Security Cheat Sheet の改訂コミット。platform-level function configuration と実行時取得の違いを明確化) ※2026-07-16 fetch
- [アプリに秘密を1文字も書かない ―Secret Manager×Cloud RunをAzure Key Vault目線で](https://zenn.dev/ccie26302/articles/zenn-gcp-secret-manager-cloud-run) (Zenn、Cloud Run への `--set-secrets` 注入とシークレット単位 IAM バインディングの具体例) ※2026-08-08 fetch

**出典引用**:
> "Fetch secrets at runtime from a vault or caching extension — not from platform-level function configuration (e.g. Lambda Environment Variables)"
> ([OWASP CheatSheetSeries: Serverless/FaaS Security Cheat Sheet 改訂](https://github.com/OWASP/CheatSheetSeries/commit/ae0b3f22f5b24381455f742af9b5fa84063ad770), セクション "Secrets Management") ※2026-07-16に実際にfetch成功

> "権限（secretAccessor）が無ければ、そもそも秘密を注入できません。秘密は IAM でガードされている。"
> ([アプリに秘密を1文字も書かない ―Secret Manager×Cloud RunをAzure Key Vault目線で](https://zenn.dev/ccie26302/articles/zenn-gcp-secret-manager-cloud-run), セクション IAM境界) ※2026-08-08に実際にfetch成功

**バージョン**: パターン（実装依存）
**確信度**: 高
**最終更新**: 2026-08-08

---

### 5. サーバーレス関数の JWT 署名は、秘密鍵を配布せず KMS の署名 API 経由で行う

サーバーレス/エッジ関数が JWT に署名する場合、秘密鍵そのものを環境変数やコードに持たせるのではなく、マネージド鍵管理サービス（KMS）の署名 API を呼び出す設計にする。秘密鍵は KMS 側から一度も出ないため、関数インスタンス間でのローテーション・漏洩リスクを構造的に排除できる。

**根拠**:
- 秘密鍵を各関数インスタンスの環境変数に配布する方式は、インスタンスごとのローテーション漏れや環境変数経由の漏洩リスクを抱える
- KMS 側で署名すれば、検証側は発行者ごとに公開される JWKS（OpenID Connect Discovery）から公開鍵を取得でき、秘密鍵を一切配布せずに外部サービスとの相互検証が成立する
- 認証は基盤側の OIDC トークンで行われるため、アプリケーションコードが鍵のライフサイクル管理を持つ必要がない

**コード例**:
```typescript
// 署名側（サーバーレス関数）
import { signToken } from '@vercel/kms';

export async function GET() {
  const token = await signToken({
    issuerId: '123e4567-e89b-42d3-a456-426614174000',
    claims: { sub: 'user_123', scope: 'read:data' },
    ttl: 300,
  });
  const res = await fetch('https://api.example.com/data', {
    headers: { Authorization: `Bearer ${token}` },
  });
  return new Response(await res.text(), { status: res.status });
}
```
```typescript
// 検証側（外部サービス） — JWKS は issuerId ごとに公開される
import { createRemoteJWKSet, jwtVerify } from 'jose';

const JWKS = createRemoteJWKSet(
  new URL(`https://kms.vercel.com/${issuerId}/jwks.json`)
);
const { payload } = await jwtVerify(token, JWKS);
```

**出典引用**:
> "private keys never live in your code or environment variables"
> ([Sign JWTs from your Functions without managing private keys](https://vercel.com/changelog/sign-jwts-from-your-functions-without-managing-private-keys), セクション "Overview") ※2026-08-18に実際にfetch成功

> "Verify signed tokens anywhere. Each issuer publishes a public OpenID Connect Discovery document"
> ([Sign JWTs from your Functions without managing private keys](https://vercel.com/changelog/sign-jwts-from-your-functions-without-managing-private-keys), セクション "Capabilities") ※2026-08-18に実際にfetch成功

**出典**:
- [Sign JWTs from your Functions without managing private keys](https://vercel.com/changelog/sign-jwts-from-your-functions-without-managing-private-keys) (Vercel 公式changelog、Vercel KMS による署名API・JWKS配布の仕組み) ※2026-08-18 fetch

**バージョン**: Vercel KMS（`@vercel/kms`、2026年8月時点）
**確信度**: 高
**最終更新**: 2026-08-18

---

### 6. ローカル開発のシークレットは種類別に保管場所を分離する（アプリ用 / CLI ログイン用 / CI 用）

`.env.local` に何でも詰め込むのではなく、①アプリケーションシークレット（Vercel 等のランタイムシークレットストア）、②ローカル CLI の認証情報（各 CLI がネイティブに使う Keychain 等の暗号化ストレージ）、③CI トークン（GitHub Environment Secrets 等、権限を絞った専用トークン）の3種類に分けて保管場所を選ぶ。

**根拠**:
- `.env.local` を Git 管理から除外していても、エディタ・バックアップツール・検索インデックス・AI コーディングエージェントなど「読める範囲」は Git 管理の有無とは無関係に広がる。Rule #3 の gitignore 徹底だけでは、ファイルシステム上に平文で存在するという根本的なリスクは残る
- CLI のログイン情報（`vercel login` 等でローカルに保存される認証情報）とアプリへ渡す API キーは性質が異なる。前者は開発者個人の権限、後者はアプリケーションの実行時権限であり、同じ `.env.local` にまとめると権限の境界が曖昧になる
- macOS では `security` コマンドで Keychain にサービス単位のエントリを作成でき、Preview/Production 用に別エントリを分ければ環境ごとの取り違えも防げる
- CI トークンは個人の認証情報を使い回さず、権限を絞った専用トークンを発行する
- git の remote URL や `.git/config` に PAT を直接埋め込む（`https://<token>@github.com/...`）と平文で残り続ける。`gh auth login` 等 CLI ネイティブの credential helper 方式に切り替えれば、OS のクレデンシャルストア経由でトークンを扱え、`.git/config` には平文トークンが残らない

**命名規約**: `<app>-<purpose>-<environment>`（例: `comic-app-openai-preview`）

**コード例**:
```bash
# macOS Keychain へ API キーを保存する
security add-generic-password \
  -U \
  -a "$(id -un)" \
  -s "comic-app-openai-preview" \
  -w

# Keychain から取得して Vercel の環境変数へ渡す（ローカルにも .env にも平文で残さない）
security find-generic-password \
  -a "$(id -un)" \
  -s "comic-app-openai-preview" \
  -w \
  | vercel env add OPENAI_API_KEY preview --sensitive

# 各 CLI のネイティブなログイン方式を使う（.env に手動転記しない）
vercel login
neon auth
wrangler login --use-keyring

# git/GitHub: remote URL に PAT を平文で埋め込む代わりに credential helper を使う
gh auth login
gh auth setup-git
git config --global --get credential.https://github.com.helper
```

**出典引用**:
> "Git管理から除外していても、エディタ、バックアップ、検索、AIコーディングエージェントなどから読める範囲が広がるため"
> ([シークレットはどこに置く？ .env、Keychain、CLIログイン、Secretストアを整理した](https://zenn.dev/optimisuke/articles/d7a4c2e91f6b30), セクション本文) ※2026-08-19に実際にfetch成功

> "CLIのログイン情報は、アプリへ渡すAPIキーとは別物です"
> ([シークレットはどこに置く？ .env、Keychain、CLIログイン、Secretストアを整理した](https://zenn.dev/optimisuke/articles/d7a4c2e91f6b30), セクション本文) ※2026-08-19に実際にfetch成功

**出典**:
- [シークレットはどこに置く？ .env、Keychain、CLIログイン、Secretストアを整理した](https://zenn.dev/optimisuke/articles/d7a4c2e91f6b30) (Zenn、アプリ/CLI/CIの3分類とmacOS Keychainへの実際の保存・取得コマンド) ※2026-08-19 fetch
- [【GitHub】セキュリティ強化のためPAT を卒業する](https://qiita.com/kura13/items/78073e51eac9a4e72383) (Qiita、`.git/config` への PAT 平文埋め込みから `gh auth login` の credential helper への切り替え手順) ※2026-08-23 fetch

**バージョン**: 一般原則（例は macOS `security` コマンド + Vercel CLI + GitHub CLI）
**確信度**: 中（単一記事だが各CLIの実際のコマンド・フラグを直接示す実機検証のためパターン1c扱い）
**最終更新**: 2026-08-23

---

### 7. Vercel の環境変数は Secret 種別 + 「本番シークレット値分離」ポリシーで管理する

Vercel は環境変数の「Sensitive」トグルを廃止し、Config（保存後も値を閲覧可能）と Secret（保存後は誰も値を閲覧・取得できず、デプロイからのみ参照可能）の2種別に分離した。あわせて、Secret 型の本番環境の値が Preview / Development / カスタム環境と同一であることを禁止する新ポリシー「Separate Production Secret Values」が提供されている。

**根拠**:
- Config は公開してよい非機密設定（例: 公開フレームワーク変数）向けで、保存後もメンバーが値を閲覧できる
- Secret はパスワード・APIキー・トークン向けで、保存後はメンバーが値を閲覧・取得できない（デプロイからのみ参照可能）
- 既存の Sensitive フラグ付き変数は自動的に Secret 扱いとなり、移行作業は不要（後方互換）
- 旧ポリシー「Enforce Sensitive Environment Variables」は廃止され、新ポリシー「Separate Production Secret Values」に置き換えられた。このポリシーは Secret 型の本番値が Preview/Development/カスタム環境の値と異なることを強制する
- レガシーの `--sensitive` / `--no-sensitive` CLI フラグは引き続き動作し、それぞれ Secret / Config にマッピングされる
- Rule #4（Secret Manager 採用）とは補完関係にある: #4 は「本番で平文 `.env` を使わない」というプラットフォーム非依存の原則、本ルールは Vercel 自身の環境変数システムが提供する write-only な値保護と環境間の値分離強制という、プラットフォーム固有の具体的な実装手段

**コード例**:
```bash
# Config: 公開してよい設定値（保存後も値を閲覧可能）
vercel env add API_URL production --value "https://api.example.com" --visibility config --yes

# Secret: 機密値（保存後は誰も閲覧・取得できない）
vercel env add API_KEY production --value "sk_live_..." --visibility secret --yes
```

**出典引用**:
> "The value remains available to your deployments and can be replaced, but members cannot view or retrieve it after saving."
> ([Environment variables now use Config and Secret types](https://vercel.com/changelog/environment-variables-now-use-config-and-secret-types), セクション本文) ※2026-08-24に実際にfetch成功

> "the Production value for a Secret must differ from the values used for the same key in Preview, Development, and custom environments."
> ([Environment variables now use Config and Secret types](https://vercel.com/changelog/environment-variables-now-use-config-and-secret-types), セクション "Separate Production Secret Values") ※2026-08-24に実際にfetch成功

**出典**:
- [Environment variables now use Config and Secret types](https://vercel.com/changelog/environment-variables-now-use-config-and-secret-types) (Vercel 公式 changelog) ※2026-08-24 fetch

**バージョン**: Vercel（環境変数 Config/Secret 種別、2026-08時点）
**確信度**: 高
**最終更新**: 2026-08-24

---
