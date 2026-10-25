# シークレット管理のベストプラクティス

API キー・DB 接続文字列・OAuth クレデンシャル等を「サーバーだけが持つ」状態を保ち、クライアントバンドルに漏れないようにする。

## ルール

### 1. `NEXT_PUBLIC_` プレフィックスでクライアント・サーバー境界を明示する

Next.js では、`.env` や実行環境から読み込む環境変数のうち `NEXT_PUBLIC_` プレフィックスのものだけがクライアントバンドルに含まれる（`next.config` の `env` に書いた値は例外で、接頭辞に関係なく常にバンドルに入る）。
プレフィックスなしの環境変数は **サーバー専用**。混同しないようコマンド規約を統一する。
また `NEXT_PUBLIC_` はビルド時にソースコードへ直接インライン化される値であり、デプロイ環境（Vercel 等）でダッシュボードの値を更新しただけでは反映されない点に注意する。**再ビルド（再デプロイ）して初めて新しい値が焼き込まれる**。

**根拠**:
- 認証情報をうっかり `NEXT_PUBLIC_*` で公開する事故が業界全体で頻発（GitHub の secret scanning でも検出される）
- `.env`・実行環境から読み込む変数に限れば、プレフィックスがビルド時に文字列置換される仕組みのため、ビルド成果物に含まれるかどうかは `NEXT_PUBLIC_` で決まる。`next.config` の `env`（Legacy 扱いの設定）に書いた値は接頭辞に関係なくバンドルにインライン化されるので、秘密情報を書かない
- Next.js 13+ では `import 'server-only'` で「サーバー専用モジュール」を強制でき、クライアントから誤って import するとビルドエラーになる
- Vite・Remix にも同等の規約（`VITE_*` / `PUBLIC_*` ）がある
- スキーマ検証（下記の `@t3-oss/env-nextjs` や Zod）で起動時に設定漏れ・型の誤りを検出する。自前の Zod で書く場合は、サーバー用スキーマの parse をサーバー専用モジュール（`import 'server-only'`）に置き、Client Component が import するモジュールのトップレベルで `process.env` 全体を parse しない（クライアントでは非公開の変数が `undefined` になるため、import しただけで検証エラーになる）
- ビルド時インライン化の帰結として、値のローテーション（ライセンスキー更新等）は環境変数更新だけでは反映されず、**Production・Preview 双方の明示的な再デプロイ**が必要（例: Vercel の `vercel redeploy <deployment-url>`）。複数アプリで同じ `NEXT_PUBLIC_*` を共有している場合、1 つだけ再デプロイを忘れると古い値のまま本番稼働し続ける事故につながる

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

```bash
# NEXT_PUBLIC_* の値をローテーションした後、Vercel で明示的に再デプロイしないと
# 古い値がバンドルに焼き込まれたまま配信され続ける
vercel redeploy <最新のデプロイURL>
# Production / Preview は別ビルドなので、両方を再デプロイする必要がある
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

Gemini・OpenAI 等の LLM API キーは漏洩したときのコスト爆発リスクが大きい。命名規約（`NEXT_PUBLIC_` なし）＋ビルド時強制（`import 'server-only'`）に加え、**Server Action を唯一のゲートウェイ**にすることで三層防御になる。
ただし Server Action は公開 POST エンドポイントで、第三者が curl などから直接呼べる。キーを隠しても、ゲートウェイが誰の呼び出しでも通すなら有料 API を無制限に使われる（denial-of-wallet）。Layer 2 には次の 3 つを必須とする:

- 認証（`verifySession()`）と、その機能を使う権限の確認
- ユーザー単位のレート上限と、入力長の上限
- 予算上限: ユーザー・テナント単位の月次トークン使用量を記録して上限で止める。LLM 提供元のダッシュボードでも支出上限と予算アラートを設定する（ゲートウェイが破られたときの最後の歯止め）

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
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';
import { verifySession } from '@/app/_lib/dal/auth';
import { canUseAiFeature } from '@/app/_lib/dal/entitlements'; // 契約プラン等による利用権限
import { generateContent } from '@/lib/ai/gemini';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '1 m'), // ユーザーごとに 1 分 10 回まで
});

const MAX_INPUT_LENGTH = 4_000;

export async function generateAction(input: string) {
  const session = await verifySession();
  if (!session) return { ok: false, error: 'Unauthorized.' };
  if (!(await canUseAiFeature(session.userId))) return { ok: false, error: 'Forbidden.' };

  if (!input.trim() || input.length > MAX_INPUT_LENGTH) {
    return { ok: false, error: 'Invalid input.' };
  }

  const { success } = await ratelimit.limit(`ai-generate:${session.userId}`);
  if (!success) return { ok: false, error: 'Too many requests.' };

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
- [Next.js Docs: env（next.config.js）](https://nextjs.org/docs/app/api-reference/config/next-config-js/env) (Next.js 公式、"environment variables specified in this way will always be included in the JavaScript bundle") ※2026-10-25 確認
- [upstash/ratelimit-js](https://github.com/upstash/ratelimit-js) (Upstash 公式、`Ratelimit.slidingWindow` と `limit()` の使い方) ※2026-10-25 確認
- [NEXT_PUBLIC_* を更新したのに古いキーが動き続ける（ビルド時に焼き込まれる）](https://zenn.dev/y3chi5z/articles/next-public-env-baked-at-build) (Zenn y3chi5z、Vercel でのローテーション事故と `vercel redeploy` による対処) ※2026-09-15に実際にfetch成功

**出典引用**:
> "「隠す」のではなく「縛る」ことで守るからだ"
> ([「API キーは全部隠せ」は勘違いだった](https://qiita.com/Kazy_engineer/items/c66f3fe21f25f096f571), セクション "公開値は縛る") ※2026-06-15に実際にfetch成功

> "NEXT_PUBLIC_ で始まる環境変数は、ビルドのときにソースコードへ直接インライン化されます"
> ([NEXT_PUBLIC_* を更新したのに古いキーが動き続ける（ビルド時に焼き込まれる）](https://zenn.dev/y3chi5z/articles/next-public-env-baked-at-build), セクション "原因") ※2026-09-15に実際にfetch成功

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-10-25

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
- ファイル本文への grep はコミットの author/committer メタデータ（`git config user.email` 未設定時にホスト名から合成される識別子等）を検査対象に含まない。公開リポジトリでは `git log --all` で全参照を対象に含め、noreply 形式のメールアドレスを標準にする

**CI 設定例**:
```yaml
# .github/workflows/secret-leak-check.yml
name: Secret Leak Check
on: [pull_request]

permissions:
  contents: read
  pull-requests: write # gitleaks-action が PR にコメントするため

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0 # gitleaks がコミット履歴を走査するため
      - uses: actions/setup-node@v7
        with: { node-version: 24 }
      - run: npm ci && npm run build

      # 1. ビルド済み JS にシークレット候補がないか grep
      #    -l でファイル名だけを出す（-n だと一致した行＝シークレットそのものが CI ログに残る）
      - name: Grep build for known secret prefixes
        run: |
          if grep -rlE "sk_(live|test)_|rk_(live|test)_|password=|AKIA[0-9A-Z]{16}" .next/static/; then
            echo "ERROR: secret pattern found in client bundle"
            exit 1
          fi

      # 2. TruffleHog で広範な検査。TruffleHog の GitHub Action は git 履歴を走査するので、
      #    git 管理外の .next/static は CLI の filesystem モードで走査する
      - name: TruffleHog (build output)
        run: |
          docker run --rm -v "$PWD/.next/static:/scan:ro" \
            trufflesecurity/trufflehog:3.99.7 \
            filesystem /scan --results=verified,unknown --fail

      # 3. リポジトリ全体にも secret scanner（コミット時の漏洩）
      # betterleaks（gitleaks 作者による後継ツール）を使うと BPE トークナイザーで
      # 検出率 98.6% vs gitleaks 70.4%（CredData 評価）を達成できる
      - uses: gitleaks/gitleaks-action@v3
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }} # Organization のリポジトリでは必須
```

サードパーティの action（`gitleaks/gitleaks-action`）と Docker イメージ（`trufflesecurity/trufflehog`）は、実運用では SHA / digest でピン留めする（`practices/ci-cd/github-actions.md` #9）。`gitleaks-action@v2` は Node 20 ランタイムのため、GitHub ホストランナーから Node 20 が削除された 2026-09-16 以降は動かない。

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
- 環境変数だけでなく、Source Map にコメントで PII / コメントが残るケースもある → Source Map をクライアント配信しない（Sentry へアップロードする場合は SDK の既定でアップロード後にクライアントの `.map` が削除される。`practices/observability/error-tracking.md` #4・#9）
- `.env.local` を accidentally commit する事故は `.gitignore` + pre-commit hook で防ぐ
- Vercel・Cloudflare のビルドログにもシークレットを `***` で伏字化する機能あり

**出典**:
- [TruffleHog](https://github.com/trufflesecurity/trufflehog) (Truffle Security、`filesystem` サブコマンドと `--results` / `--fail`。GitHub Action の action.yml は `git file://` で履歴を走査する) ※2026-10-25 確認
- [gitleaks/gitleaks-action](https://github.com/gitleaks/gitleaks-action) (Gitleaks 公式、v3 の使用例と v2 から v3 への移行、Node 20 の削除日) ※2026-10-25 確認
- [Gitleaks](https://github.com/gitleaks/gitleaks)
- [GitHub: secret scanning](https://docs.github.com/en/code-security/secret-scanning) (GitHub Docs)
- [gitleaksの作者が一から作り直したbetterleaks](https://zenn.dev/shuymn/articles/600779b488d6d3) (Zenn shuymn、BPE トークナイザーによる検出率改善) ※2026-06-08に実際にfetch成功
- [GitLeaksからBetterLeaksに乗り換えた話](https://zenn.dev/mohhh_ok/articles/gitleaks-to-betterleaks) (Zenn mohhh_ok、実際の移行事例) ※2026-06-09に実際にfetch成功
- [公開されるのは本文だけではない。grep検査を全部通したのにコミット履歴から個人情報が漏れていた話](https://qiita.com/fujiken818/items/e96cf84bc2bee61512b2) (Qiita、ファイル本文grepでは検知できないコミット著者欄メタデータ漏洩の実例と全数検査スクリプト) ※2026-09-05に実際にfetch成功

> "本文のgrepは0件だったが、著者欄に情報が焼き込まれている"
> ([公開されるのは本文だけではない。grep検査を全部通したのにコミット履歴から個人情報が漏れていた話](https://qiita.com/fujiken818/items/e96cf84bc2bee61512b2), Qiita, セクション "コミットの著者欄は本文ではない") ※2026-09-05に実際にfetch成功

**バージョン**: Next.js 13+, GitHub Actions（Node 24、gitleaks-action v3、TruffleHog 3.99）
**確信度**: 高
**最終更新**: 2026-10-25

#### 追加根拠 (2026-05-16)

`.map` をデプロイに残す構成（ブラウザの DevTools でも本番のソースを読みたい場合）でのアクセス制限は、Vercel なら Protected Source Maps（`practices/observability/error-tracking.md` #9）を使う。Sentry SDK の既定のようにアップロード後に `.map` を削除していれば、配信される `.map` は残らない。

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
新規メンバーは `.env.example` をコピーして `.env.local` を作る運用に統一する。`.env.local` に入れるのは、漏れても被害が限られる開発用の値（ローカル DB・テスト用キー等）だけにし、本番・共有環境の実値はリポジトリツリーの外（Secret Manager・OS の Keychain。#4・#6）に置く。

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
```sh
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
      run: gitleaks git --pre-commit --redact --staged --verbose
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

リポジトリツリー内（`.env.local` を含む）に置いてよいのは、`.env.example` のようなキー名のみのテンプレートと、漏れても被害が限られる開発用の値までとする。本番・共有環境の実値は repo tree の外側 or Secret Manager に置く。

**出典引用**:
> "AIコーディングのテンポは速く、一つ前のターンで何を保留にしたか、平気で忘れます。"
> ([AI コーディングで secret を漏らさないための４層防御](https://zenn.dev/takna/articles/secret-leak-prevention-4-layer), セクション "問題の本質") ※2026-05-17に実際にfetch成功

> "gitignore は、`git add` `git commit` `git push` が対象にするファイルの範囲を決めるものであり、それ以外の経路（AIツールのファイルシステム走査・tar/rsync・Dockerビルドコンテキスト等）は一切尊重してくれません。"
> ([gitignore は防御じゃない。Grok Build の repo アップロード事案から、秘密の置き場所を見直す](https://zenn.dev/ishizakahiroshi/articles/20260714-grok-build-repo-upload-timeline), Zenn, セクション "本題: gitignore は防御じゃない") ※2026-07-14に実際にfetch成功

**事故った時の対応**:
1. シークレットを **即座に rotate**（git history から消しても、すでに漏れた値は永久に流出している前提）
2. git history からの削除は `git filter-repo` または BFG Repo-Cleaner で
3. force push 後、リポジトリの他クローンから再取り込みされないよう全員に伝達
4. Vercel / Cloudflare 等の deployment 履歴のログも確認（過去のビルドログに残っていないか）

**ドキュメント化（README）**:
```markdown
## 環境変数のセットアップ

1. `.env.example` を `.env.local` にコピー
2. 開発用の値を以下のドキュメントから取得して埋める（本番・共有環境の値は入れない）：
   - DATABASE_URL: [Confluence: DB セットアップ](https://...)（ローカル DB）
   - STRIPE_*: [Stripe Dashboard](https://dashboard.stripe.com/test/apikeys)（テストモードのキー）
   - JWT_SIGNING_KEY: `openssl rand -base64 32` で生成

3. `.env.local` は **絶対に commit しない**
```

**出典**:
- [dotenv-safe](https://github.com/rolodato/dotenv-safe) — 必須環境変数のチェック
- [Next.js Docs: Environment Variables](https://nextjs.org/docs/app/building-your-application/configuring/environment-variables#test-environment-variables) (Next.js 公式)
- [gitleaks: .pre-commit-hooks.yaml](https://github.com/gitleaks/gitleaks/blob/master/.pre-commit-hooks.yaml) (Gitleaks 公式、pre-commit フックの `gitleaks git --pre-commit --redact --staged --verbose`) ※2026-10-25 確認
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
**最終更新**: 2026-10-25

---

### 4. Secret Manager を採用し、本番で平文 `.env` を使わない

production 環境では Vercel Environment Variables / AWS Secrets Manager / Doppler / Infisical 等のシークレット管理サービスを使う。
平文ファイル（`.env.production`）でシークレットを管理しない。

**根拠**:
- 平文 `.env.production` は EC2 / VM に置くと OS レベルでアクセスできる人が広がる
- Secret Manager は IAM で「誰が」「いつ」「何の値を」読んだかを監査ログ化できる
- ローテーション（key rotation）が手動でなく自動化できる
- Vercel / Netlify / Cloudflare Pages はビルド時に環境変数を注入する仕組みを提供（OS の環境変数として渡される）
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
| **AWS Secrets Manager** | IAM 完全統合、ローテーション自動化、KMS 暗号化 | コスト高め（$0.40/secret/month） |
| **Doppler** | マルチクラウド、UI 良好、無料枠あり | サードパーティ依存 |
| **Infisical** | OSS（self-host 可）、E2E 暗号化 | 運用負荷 |
| **HashiCorp Vault** | 大企業向け、機能豊富 | 学習コスト高、オーバースペック |

**Vercel での例**:
```bash
# CLI で環境変数を設定。値はプロンプトで入力する（CLI 履歴に残らない）
# Secret 種別にすると保存後は誰も値を閲覧・取得できない（Production / Preview の既定は Secret。#7）
vercel env add DATABASE_URL production --type secret

# プレビュー・開発用にも個別に設定可能（Development の既定は Config なので Secret を明示する）
vercel env add DATABASE_URL preview --type secret
vercel env add DATABASE_URL development --type secret
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
- 即時: 漏洩疑いがあれば即時ローテーション + 全環境再デプロイ

**緊急時のチェックリスト**:
- [ ] 漏洩した key を **即座に invalidate**（API 提供元のダッシュボードで revoke）
- [ ] 新しい key を Secret Manager に登録
- [ ] 全環境を再デプロイ（ビルド時に環境変数が確定するため）
- [ ] アクセスログを確認（漏洩した key で異常なアクセスがあったか）
- [ ] post-mortem を書き、再発防止策を導入

**出典**:
- [Vercel: Environment Variables](https://vercel.com/docs/environment-variables) (Vercel)
- [Vercel CLI: vercel env](https://vercel.com/docs/cli/env) (Vercel 公式、`--type config|secret` と環境ごとの既定の種別、標準入力での値の受け渡し) ※2026-10-25 確認
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
**最終更新**: 2026-10-25

---

### 5. サーバーレス関数の JWT 署名は、秘密鍵を配布せず KMS の署名 API 経由で行う

サーバーレス/エッジ関数が JWT に署名する場合、秘密鍵そのものを環境変数やコードに持たせるのではなく、マネージド鍵管理サービス（KMS）の署名 API を呼び出す設計にする。秘密鍵は KMS 側から一度も出ないため、関数インスタンス間でのローテーション・漏洩リスクを構造的に排除できる。

**根拠**:
- 秘密鍵を各関数インスタンスの環境変数に配布する方式は、インスタンスごとのローテーション漏れや環境変数経由の漏洩リスクを抱える
- KMS 側で署名すれば、秘密鍵を一切配布せずに外部サービスとの相互検証が成立する。Vercel KMS では、検証側は発行者（issuer）ごとに公開される JWKS（OpenID Connect Discovery）から公開鍵を取得する
- Vercel KMS では、関数は Vercel の OIDC トークンで KMS に認証するため、アプリケーションコードが鍵のライフサイクル管理を持つ必要がない
- 検証側は署名だけでなく `issuer`（必要なら `audience`）と scope も検証する。署名の検証だけでは、同じ鍵・発行者が別の用途に発行したトークンも受け入れてしまう。Vercel KMS の issuer はプロジェクト・環境ごとに分かれるが、1 つの issuer で複数の受け手向けに発行するなら `aud` を付けて `audience` も検証する

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
// 検証側（外部サービス） — JWKS は issuer ごとに公開される
import { createRemoteJWKSet, jwtVerify } from 'jose';

const issuer = 'https://kms.vercel.com/123e4567-e89b-42d3-a456-426614174000';
const JWKS = createRemoteJWKSet(new URL(`${issuer}/jwks.json`));

export async function verifyServiceToken(token: string) {
  const { payload } = await jwtVerify(token, JWKS, { issuer });
  if (payload.scope !== 'read:data') throw new Error('insufficient scope');
  return payload;
}
```

**出典引用**:
> "private keys never live in your code or environment variables"
> ([Sign JWTs from your Functions without managing private keys](https://vercel.com/changelog/sign-jwts-from-your-functions-without-managing-private-keys), セクション "Overview") ※2026-08-18に実際にfetch成功

> "Verify signed tokens anywhere. Each issuer publishes a public OpenID Connect Discovery document"
> ([Sign JWTs from your Functions without managing private keys](https://vercel.com/changelog/sign-jwts-from-your-functions-without-managing-private-keys), セクション "Capabilities") ※2026-08-18に実際にfetch成功

**出典**:
- [Sign JWTs from your Functions without managing private keys](https://vercel.com/changelog/sign-jwts-from-your-functions-without-managing-private-keys) (Vercel 公式changelog、Vercel KMS による署名API・JWKS配布の仕組み。`signToken` の引数、issuer の URL 形式と `jwtVerify(token, jwks, { issuer })` の検証例は ※2026-10-25 確認) ※2026-08-18 fetch

**バージョン**: Vercel KMS（`@vercel/kms`、2026年8月時点）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 6. ローカル開発のシークレットは種類別に保管場所を分離する（アプリ用 / CLI ログイン用 / CI 用）

`.env.local` に何でも詰め込むのではなく、①アプリケーションシークレット（Vercel 等のランタイムシークレットストア）、②ローカル CLI の認証情報（各 CLI がネイティブに使う Keychain 等の暗号化ストレージ）、③CI トークン（GitHub Environment Secrets 等、権限を絞った専用トークン）の3種類に分けて保管場所を選ぶ。

**根拠**:
- `.env.local` を Git 管理から除外していても、エディタ・バックアップツール・検索インデックス・AI コーディングエージェントなど「読める範囲」は Git 管理の有無とは無関係に広がる。Rule #3 の gitignore 徹底だけでは、ファイルシステム上に平文で存在するという根本的なリスクは残る
- CLI のログイン情報（`vercel login` 等でローカルに保存される認証情報）とアプリへ渡す API キーは性質が異なる。前者は開発者個人の権限、後者はアプリケーションの実行時権限であり、同じ `.env.local` にまとめると権限の境界が曖昧になる
- macOS では `security` コマンドで Keychain にサービス単位のエントリを作成でき、Preview/Production 用に別エントリを分ければ環境ごとの取り違えも防げる
- CI トークンは個人の認証情報を使い回さず、権限を絞った専用トークンを発行する
- git の remote URL や `.git/config` に PAT を直接埋め込む（`https://<token>@github.com/...`）と平文で残り続ける。`gh auth login` 等 CLI ネイティブの credential helper 方式に切り替えれば、OS のクレデンシャルストア経由でトークンを扱え、`.git/config` には平文トークンが残らない
- AI エージェントと共有するリポジトリでは、未コミットの機密ファイルを「誰が置いたか（人間かAIか）」を特定できないまま、未コミット変更を自動回収する仕組みが善意でコミット履歴に焼き込むリスクがある。書く場所を「どこにも書かない/書くが公開しない/公開前に必ずチェックする」の3層に分け、機密ファイルは決まった名前・場所に限定して `.gitignore` で封じ、AI にチェックさせる際は中身をチャットに書かせず場所と種類だけ報告させる

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
- [見覚えのないファイルが落ちていた——AIと共有するリポジトリの秘密境界](https://zenn.dev/hilopon/articles/ai-shared-repository-secret-boundaries) (Zenn、AIエージェントとの共有リポジトリにおける3層の秘密情報境界設計) ※2026-09-05に実際にfetch成功

> "次に起動したAIが、完全に善意で、トークンをgitの履歴に焼き込んでいた"
> ([見覚えのないファイルが落ちていた——AIと共有するリポジトリの秘密境界](https://zenn.dev/hilopon/articles/ai-shared-repository-secret-boundaries), Zenn, セクション "怖いのは「誰が置いたか」ではなかった") ※2026-09-05に実際にfetch成功
- [【GitHub】セキュリティ強化のためPAT を卒業する](https://qiita.com/kura13/items/78073e51eac9a4e72383) (Qiita、`.git/config` への PAT 平文埋め込みから `gh auth login` の credential helper への切り替え手順) ※2026-08-23 fetch

**バージョン**: 一般原則（例は macOS `security` コマンド + Vercel CLI + GitHub CLI）
**確信度**: 中（単一記事だが各CLIの実際のコマンド・フラグを直接示す実機検証のためパターン1c扱い）
**最終更新**: 2026-09-05

---

### 7. Vercel の環境変数は Secret 種別 + 「本番シークレット値分離」ポリシーで管理する

Vercel は環境変数の「Sensitive」トグルを廃止し、Config（保存後も値を閲覧可能）と Secret（保存後は誰も値を閲覧・取得できず、デプロイからのみ参照可能）の2種別に分離した。あわせて、Secret 型の本番環境の値が Preview / Development / カスタム環境と同一であることを禁止する新ポリシー「Separate Production Secret Values」が提供されている。

**根拠**:
- Config は公開してよい非機密設定（例: 公開フレームワーク変数）向けで、保存後もメンバーが値を閲覧できる
- Secret はパスワード・APIキー・トークン向けで、保存後はメンバーが値を閲覧・取得できない（デプロイからのみ参照可能）
- 既存の Sensitive フラグ付き変数は自動的に Secret 扱いとなり、移行作業は不要（後方互換）
- 旧ポリシー「Enforce Sensitive Environment Variables」は廃止され、新ポリシー「Separate Production Secret Values」に置き換えられた。このポリシーは Secret 型の本番値が Preview/Development/カスタム環境の値と異なることを強制する
- CLI の `--sensitive` / `--no-sensitive` フラグは引き続き動作し、それぞれ Secret / Config にマッピングされる。種別の明示は `--type config|secret`（`--visibility` は非推奨のエイリアス）
- Vercel のダッシュボードは値を生の文字列のまま保存するため、末尾改行つきで登録すると本番だけ `Invalid character in header content ["host"]`（`ERR_INVALID_CHAR`）が出る（ローカルの `.env.local` は読み込み時に改行が除かれるため再現しない）。登録時は `echo` ではなく `printf` を使い、`vercel env pull` した結果を `cat -A` で確認し、URL 等の値は読み込み側でも `trim()` する
- Rule #4（Secret Manager 採用）とは補完関係にある: #4 は「本番で平文 `.env` を使わない」というプラットフォーム非依存の原則、本ルールは Vercel 自身の環境変数システムが提供する write-only な値保護と環境間の値分離強制という、プラットフォーム固有の具体的な実装手段
- v0 のような AI コード生成ツールも同じ Secret/Config 分離の恩恵を受ける: `NPM_TOKEN` / `NPM_RC` を Secret 種別の shared environment variable として設定すると、プライベート npm パッケージ導入時もクレデンシャルが LLM に渡らず、サンドボックスのファイルシステムにも書き込まれない

**コード例**:
```bash
# Config: 公開してよい設定値（保存後も値を閲覧可能）
vercel env add API_URL production --value "https://api.example.com" --type config --yes

# Secret: 機密値（保存後は誰も閲覧・取得できない）
# 実値は --value でコマンドライン引数に書かない（シェル履歴・プロセス一覧に残る）
# 方法 1: プロンプトで入力する
vercel env add API_KEY production --type secret
# 方法 2: #6 のように Keychain 等から標準入力で渡す（tr で末尾改行を除く）
security find-generic-password -a "$(id -un)" -s "myapp-api-key-production" -w \
  | tr -d '\n' \
  | vercel env add API_KEY production --type secret
```

`--visibility` は `--type` の非推奨エイリアス。`--sensitive` / `--no-sensitive` はそれぞれ Secret / Config に対応する。

**出典引用**:
> "The value remains available to your deployments and can be replaced, but members cannot view or retrieve it after saving."
> ([Environment variables now use Config and Secret types](https://vercel.com/changelog/environment-variables-now-use-config-and-secret-types), セクション本文) ※2026-08-24に実際にfetch成功

> "the Production value for a Secret must differ from the values used for the same key in Preview, Development, and custom environments."
> ([Environment variables now use Config and Secret types](https://vercel.com/changelog/environment-variables-now-use-config-and-secret-types), セクション "Separate Production Secret Values") ※2026-08-24に実際にfetch成功

**出典**:
- [Environment variables now use Config and Secret types](https://vercel.com/changelog/environment-variables-now-use-config-and-secret-types) (Vercel 公式 changelog) ※2026-08-24 fetch
- [Vercel CLI: vercel env](https://vercel.com/docs/cli/env) (Vercel 公式、"`--visibility` is a deprecated alias for `--type`"、`echo [value] | vercel env add` は "this will save the value in bash history") ※2026-10-25 確認
- [v0 now reads npm credentials from shared environment variables](https://vercel.com/changelog/v0-now-reads-npm-credentials-from-shared-environment-variables) (Vercel 公式 changelog、AIツールがSecret値をモデルに露出させない設計の実例) ※2026-09-19 fetch
  > "Credentials can be marked sensitive, and v0 never exposes them to the model or writes them to the sandbox filesystem."
- [Next.js 本番だけ ERR_INVALID_CHAR が出る原因と直し方 — 環境変数末尾改行の罠](https://zenn.dev/joemike/articles/nextjs-err-invalid-char-env-newline-sentry-2026) (Zenn joemike、Vercel 環境変数の末尾改行混入と `trim()` 対策) ※2026-10-13 fetch
  > "ローカルは自動で安全、Vercel は生の文字列をそのまま保存する。" (セクション "なぜローカルで再現しないか")

**バージョン**: Vercel（環境変数 Config/Secret 種別、2026-08時点）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 8. 長期保存データの鍵確立には post-quantum 移行を見据えたハイブリッド方式を検討する

RSA・ECC（Curve25519 を含む）ベースの公開鍵暗号は post-quantum 安全ではない。将来 crypto-relevant な量子コンピュータが実用化されれば破られうるため、長期間の機密性を要するデータ（保存期間が数年〜数十年に及ぶバックアップ・アーカイブ等）の鍵確立方式には、移行を見据えた設計が必要になる。

**根拠**:
- OWASP Cryptographic Storage Cheat Sheet が「Post-Quantum Consideration」節を新設し、RSA/ECC が post-quantum 安全でないことを明記した
- post-quantum な鍵確立が必要な場合は、標準化された **ML-KEM（NIST FIPS 203）** を、移行期間中は古典アルゴリズムと組み合わせた **ハイブリッド構成** で使うことが推奨されている
- 既存の RSA/ECC 実装を今すぐ全廃する話ではなく、「保存期間が長いデータの鍵確立から優先的に post-quantum 移行を計画する」というリスクベースの判断が前提

**コード例**:
```text
// Bad: 長期保存データの鍵確立を ECC 単独に依存し続ける
KeyAgreement: X25519 のみ

// Good: 移行期間中はハイブリッド構成で post-quantum 耐性を確保する
KeyAgreement: X25519 + ML-KEM (NIST FIPS 203) のハイブリッド
```

**出典引用**:
> "RSA and ECC-based public-key cryptography (including Curve25519) are not post-quantum secure."
> ([Cryptographic Storage Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/commit/394fe51fe08c5c53e99eee5781f28b886056fa25), OWASP CheatSheetSeries, セクション "Post-Quantum Consideration") ※2026-09-23に実際にfetch成功

**出典**:
- [Cryptographic Storage Cheat Sheet — Post-Quantum Consideration 追加コミット](https://github.com/OWASP/CheatSheetSeries/commit/394fe51fe08c5c53e99eee5781f28b886056fa25) (OWASP CheatSheetSeries 公式) ※2026-09-23 fetch

**バージョン**: 該当なし（暗号方式の選定指針）
**確信度**: 高
**最終更新**: 2026-09-23

---
