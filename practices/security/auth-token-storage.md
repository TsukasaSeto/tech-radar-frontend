# 認証トークン管理のベストプラクティス

セッショントークン・アクセストークン・リフレッシュトークンの保存方式と CSRF・XSS リスクのトレードオフ。
フロントエンドが直接トークンを扱う設計は脆弱になりやすい。

## ルール

### 1. アクセストークンは HttpOnly Cookie で保存し、localStorage を避ける

セッショントークン・アクセストークンは `HttpOnly` + `Secure` + `SameSite=Lax` 以上の Cookie で保存する。
`localStorage` / `sessionStorage` への保存は XSS で盗まれるため避ける。

**根拠**:
- `HttpOnly` Cookie は JavaScript から読み取れない。XSS で `document.cookie` を実行されても盗まれない
- `localStorage` は同一オリジンの任意のスクリプトから読める。XSS が発生した瞬間に全トークンが流出
- 第三者スクリプト（広告・分析・チャットウィジェット）が侵害された場合の被害範囲も Cookie のほうが小さい
- OWASP・Auth0・Okta 等の公式ガイドはすべて HttpOnly Cookie 推奨

**コード例（サーバー側でセット）**:
```ts
// app/api/auth/login/route.ts
import { cookies } from 'next/headers';

export async function POST(req: Request) {
  const { email, password } = await req.json();
  const session = await authenticate(email, password);

  if (!session) {
    return Response.json({ error: 'invalid_credentials' }, { status: 401 });
  }

  const cookieStore = await cookies();
  cookieStore.set('session', session.token, {
    httpOnly: true,                              // JS から読めない
    secure: process.env.NODE_ENV === 'production', // HTTPS のみ
    sameSite: 'lax',                              // CSRF 対策
    path: '/',
    maxAge: 60 * 60 * 24 * 7,                     // 7 日
  });

  return Response.json({ ok: true });
}
```

**クライアント側の扱い**:
```tsx
// Bad: トークンを JS で扱う
const token = localStorage.getItem('access_token');
fetch('/api/data', { headers: { Authorization: `Bearer ${token}` } });

// Good: Cookie は fetch が自動送信する。JS でトークンを触らない
fetch('/api/data', { credentials: 'include' });
// または same-origin なら credentials 指定不要
```

**NextAuth.js での典型的なアンチパターン**:
```ts
// Bad: session() コールバックでアクセストークンをクライアントに露出
// auth.ts
callbacks: {
  async session({ session, token }) {
    session.accessToken = token.accessToken; // useSession()・/api/auth/session・DevTools で可視
    return session;
  }
}
// → XSS で session.accessToken が盗まれる。/api/auth/session のレスポンスにも含まれ誰でも確認可能

// Good: jwt() のみにアクセストークンを保持し、server-only 経由で BFF からのみ利用
// auth.ts
callbacks: {
  async jwt({ token, account }) {
    if (account) token.accessToken = account.access_token; // サーバー側のみ
    return token;
  },
  async session({ session }) {
    return session; // accessToken を含めない — クライアントに露出しない
  }
}
```

```ts
// app/api/data/route.ts — BFF からのみトークンを使用（サーバー側限定）
import 'server-only';
import { getToken } from 'next-auth/jwt';

export async function GET(req: Request) {
  const token = await getToken({ req }); // サーバー側のみアクセス可能
  const accessToken = typeof token?.accessToken === 'string' ? token.accessToken : null;
  if (!accessToken) {
    // 未認証は上流に送らずに 401 を返す（`Bearer undefined` を送らない）
    return Response.json({ error: 'unauthorized' }, { status: 401 });
  }

  const res = await fetch('https://api.example.com/data', {
    headers: { Authorization: `Bearer ${accessToken}` },
  });
  if (res.status === 401 || res.status === 403) {
    // クライアントが再認証・リフレッシュ（Rule #7）を判断できるようにステータスを伝える
    return Response.json({ error: 'unauthorized' }, { status: res.status });
  }
  if (!res.ok) {
    // 上流のエラー本文はクライアントに転送しない
    return Response.json({ error: 'upstream_error' }, { status: 502 });
  }
  return Response.json(await res.json());
}
```

ブラウザ↔BFF 間は Cookie、BFF↔外部 API 間は Bearer Token の2層設計にする。
XSS をゼロにできない以上、「漏れたら終わり」のトークンを可読領域に置くこと自体が設計上の妥協である。

**例外（HttpOnly Cookie が使えないケース）**:
- WebSocket 接続: 同一サイトならハンドシェイクで Cookie が送られるので Cookie 認証を使う。別ドメインで Cookie が使えない場合は下の判断軸の「別ドメイン API」と同じ扱いにし、localStorage には置かない
- マイクロフロントエンドでサブドメインまたぐ SSO: Cookie の Domain 設定で対応するのが第一選択
- どうしても JS でトークンが必要 → アクセストークンを **5-15 分の短命** にしてメモリにだけ保持し、リフレッシュは Cookie で（Rule 3）

**判断軸**（DPoP を使う構成もこの表にまとめる。Rule #6 はこの表を参照する）:
| シチュエーション | トークン保存 | 理由 |
|---|---|---|
| Web アプリ（一般的） | HttpOnly Cookie | XSS 耐性最優先 |
| BFF パターン | HttpOnly Cookie | BFF が token を持ち、フロントは Cookie のみ |
| ネイティブアプリ | Secure Storage (Keychain / Keystore) | OS のセキュアストレージ |
| 完全 SPA + 別ドメイン API（BFF を置けない） | アクセストークンは短命でメモリのみ + DPoP（鍵は `extractable: false` で IndexedDB、Rule #6） | トークン単体の持ち出し・再利用を防ぐ。リフレッシュトークンは rotation（Rule #3）か sender-constraint が必須（RFC 9700） |
| WebSocket / long-polling（別ドメイン） | 上の行と同じ（メモリ + DPoP） | localStorage には置かない |
| Native アプリ相当の PWA | メモリ + DPoP（Rule #6） | 同上 |

**出典**:
- [OWASP: Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) (OWASP)
- [OWASP: HTML5 Security Cheat Sheet - Local Storage](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html#local-storage) (OWASP)
- [Auth0: Token Storage](https://auth0.com/docs/secure/security-guidance/data-security/token-storage) (Auth0)
- [Next.js Docs: cookies()](https://nextjs.org/docs/app/api-reference/functions/cookies) (Next.js 公式)
- [RFC 9700 — Best Current Practice for OAuth 2.0 Security](https://datatracker.ietf.org/doc/html/rfc9700) (IETF、§2.2.2 public client のリフレッシュトークンは sender-constraint か rotation が必須) ※2026-10-25 確認
- [セッションとJWTとCookieの歴史と変遷から紐解く、認証トークン管理設計](https://zenn.dev/khale/articles/web-session-jwt-cookie-history) (Zenn) ※2026-05-27に実際にfetch成功

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 2. Cookie には `Secure` / `HttpOnly` / `SameSite` を必須設定する

すべての認証 Cookie に `Secure`（HTTPS のみ）・`HttpOnly`（JS 不可）・`SameSite=Lax` 以上を設定する。
Path / Domain も最小権限で設定する。

**根拠**:
- `Secure` がないと HTTP リクエスト（HTTPS 強制が破れた瞬間）で Cookie が漏れる
- `HttpOnly` がないと XSS で取られる（Rule 1）
- `SameSite=Lax` は 2020 年から Chrome のデフォルト。CSRF 攻撃の大部分を防ぐ
- `SameSite=Strict` は最強だが、外部サイトからの遷移でセッションが失われる体験課題がある
- `Domain` を広く指定すると subdomain takeover で Cookie 流出のリスク

**Cookie 属性の判断軸**:

| 属性 | デフォルト | 推奨 | 解説 |
|---|---|---|---|
| `Secure` | false | **必須 true** | HTTPS のみで送信。本番では必ず |
| `HttpOnly` | false | **必須 true** | 認証 Cookie には必須 |
| `SameSite` | `Lax` (Chrome) | **`Lax` または `Strict`** | `None` は cross-site 必要時のみ + `Secure` 必須 |
| `Path` | リクエストの path | `/`（広いほど自然） | 細分化するメリットは少ない |
| `Domain` | リクエストの origin | **デフォルト推奨** | サブドメイン共有が必要な場合のみ明示 |
| `Max-Age` / `Expires` | session | 認証用は 1〜30 日程度 | リフレッシュトークンと組み合わせ |

**コード例（Next.js での詳細設定）**:
```ts
// app/_lib/auth-cookies.ts — Route Handler / Server Function から呼ぶ（Server Component の描画中は Cookie を設定できない）
import { cookies } from 'next/headers';

export async function setAuthCookies(tokens: {
  sessionToken: string;
  csrfToken: string;
  refreshToken: string;
}) {
  const cookieStore = await cookies(); // Next.js 15 以降は非同期。v16 では await が必須

  // セッション Cookie（短期）
  cookieStore.set('session', tokens.sessionToken, {
    httpOnly: true,
    secure: true,
    sameSite: 'lax',
    path: '/',
    maxAge: 60 * 60 * 24,  // 1 day
  });

  // CSRF トークン（JS から読む必要があるので HttpOnly false。値は Rule #4 の署名付きトークン）
  cookieStore.set('__Host-csrf', tokens.csrfToken, {
    httpOnly: false,        // JS で読み取り header に付けるため
    secure: true,           // __Host- 接頭辞の必須条件
    sameSite: 'strict',     // strict で OK（同一サイト内のみ）
    path: '/',              // __Host- 接頭辞の必須条件（Domain は指定しない）
  });

  // リフレッシュトークン（長期、より厳格）
  cookieStore.set('refresh', tokens.refreshToken, {
    httpOnly: true,
    secure: true,
    sameSite: 'strict',     // refresh は strict で良い
    path: '/api/auth',      // refresh エンドポイントのみに送る
    maxAge: 60 * 60 * 24 * 30,
  });
}
```

**`SameSite=None` を使う場合**:
- iframe での埋め込みや別ドメインからの API 呼び出しで必要
- 必ず `Secure` と組み合わせ（None + non-Secure は Chrome が拒否）
- CSRF 対策として CSRF token を併用（Rule 4）

**Cookie 確認方法**:
```bash
# 本番サイトでヘッダーを確認
curl -I https://example.com/
# Set-Cookie: session=...; Path=/; HttpOnly; Secure; SameSite=Lax
```

**出典**:
- [MDN: Set-Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie) (MDN Web Docs)
- [MDN: SameSite cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite) (MDN Web Docs)
- [RFC 6265bis: Cookies HTTP State Management Mechanism](https://datatracker.ietf.org/doc/draft-ietf-httpbis-rfc6265bis/) (IETF)
- [web.dev: SameSite cookies explained](https://web.dev/articles/samesite-cookies-explained) (web.dev)
- [Next.js Docs: cookies()](https://nextjs.org/docs/app/api-reference/functions/cookies) (Next.js 公式、非同期 API であることと、`.set` / `.delete` は Server Function・Route Handler でのみ呼べること) ※2026-10-25 確認

**バージョン**: 全モダンブラウザ
**確信度**: 高
**最終更新**: 2026-10-25

---

### 3. リフレッシュトークン rotation を実装する

リフレッシュトークンは一度使ったら無効化し、新しいリフレッシュトークン + 新しいアクセストークンを発行する（rotation）。
盗まれたリフレッシュトークンの再利用を検知したら、その系列（family）のリフレッシュトークンをすべて失効させる。

**根拠**:
- アクセストークンは短命（5-15 分）でも、リフレッシュトークンが流出すると無期限に認証突破される
- Rotation で「同じリフレッシュトークンが 2 回使われた」を検知できる（攻撃 + 正規ユーザーの両方が使った状態）
- RFC 9700（OAuth 2.0 Security BCP、2025-01）は、public client に発行するリフレッシュトークンに sender-constraint か rotation を必須（MUST）としている。再利用を検知したら、どちらが攻撃者か区別できないので有効なリフレッシュトークンを失効させる（§4.14.2）
- Auth0 は rotation を有効にすると自動の reuse detection が働き、再利用を検知した family 全体を失効させる
- 失効させられるのはリフレッシュトークンだけで、発行済みのアクセストークンは期限まで使える。アクセストークンを短命にしておくのはこのため

**コード例（基本フロー）**:
```ts
// app/api/auth/refresh/route.ts
import { randomBytes } from 'node:crypto';
import { cookies } from 'next/headers';
import { db } from '@/lib/db';
import { hashToken } from '@/lib/auth/hash-token';           // 下の「追加根拠」の実装
import { generateAccessToken } from '@/lib/auth/access-token';
import { notifySecurityTeam } from '@/lib/security/notify';

const ACCESS_TTL_SEC = 15 * 60;
const REFRESH_TTL_SEC = 30 * 24 * 60 * 60;

export async function POST() {
  const cookieStore = await cookies();
  const refreshToken = cookieStore.get('refresh')?.value;

  if (!refreshToken) {
    return Response.json({ error: 'no_refresh_token' }, { status: 401 });
  }

  const now = new Date();
  const newRefresh = randomBytes(32).toString('base64url'); // 256 bit の CSPRNG

  const result = await db.$transaction(async (tx) => {
    const record = await tx.refreshToken.findUnique({
      where: { token: hashToken(refreshToken) },
    });

    // 未知（失効で削除済みを含む）・期限切れは拒否する。
    // 未知のトークンからはユーザーも family も特定できないので、ここでは失効処理をしない
    if (!record || record.expiresAt <= now) {
      return { kind: 'invalid' } as const;
    }

    // usedAt が null のときだけ更新する。並行リクエストのうち count === 1 になるのは 1 つだけ
    const { count } = await tx.refreshToken.updateMany({
      where: { id: record.id, usedAt: null },
      data: { usedAt: now },
    });

    if (count === 0) {
      // 使用済みトークンの再利用 → 攻撃検知。同じ family のリフレッシュトークンをすべて失効させる
      await tx.refreshToken.deleteMany({ where: { family: record.family } });
      return { kind: 'reuse', userId: record.userId } as const;
    }

    // 後継の発行も同じトランザクションで行う
    await tx.refreshToken.create({
      data: {
        token: hashToken(newRefresh),
        userId: record.userId,
        family: record.family,  // 同一系列であることを記録
        expiresAt: new Date(now.getTime() + REFRESH_TTL_SEC * 1000),
      },
    });
    return { kind: 'rotated', userId: record.userId } as const;
  });

  if (result.kind !== 'rotated') {
    if (result.kind === 'reuse') {
      await notifySecurityTeam(result.userId, 'refresh_token_reuse');
    }
    cookieStore.delete('session');
    // 発行時と同じ Path を指定しないと削除されない
    cookieStore.set('refresh', '', { httpOnly: true, secure: true, sameSite: 'strict', path: '/api/auth', maxAge: 0 });
    const error = result.kind === 'reuse' ? 'token_reuse_detected' : 'invalid_token';
    return Response.json({ error }, { status: 401 });
  }

  // 新しいトークンペアを発行
  const newAccess = await generateAccessToken(result.userId, ACCESS_TTL_SEC);
  cookieStore.set('session', newAccess, { httpOnly: true, secure: true, sameSite: 'lax', path: '/', maxAge: ACCESS_TTL_SEC });
  cookieStore.set('refresh', newRefresh, { httpOnly: true, secure: true, sameSite: 'strict', path: '/api/auth', maxAge: REFRESH_TTL_SEC });

  return Response.json({ ok: true });
}
```

**Family 機構**:
- 一連の rotation を「family」として記録
- 同じ family 内の古いトークンが使われたら → family 全体（= そのセッション）を失効
- ユーザーが「別端末で使い続けている」場合に巻き込まれないよう、family は端末単位で分ける運用も
- 正規クライアントの並行リフレッシュも「再利用」として検知されるので、クライアント側で Rule #7 の同時実行制御を入れる

**ライブラリでの実装**:
- **NextAuth.js (Auth.js)**: refresh token rotation を自前で実装する必要あり（公式アダプタ + ロジック追加）
- **Clerk**: rotation 標準対応、自動
- **Auth0**: Dashboard でアプリごとに rotation を有効化する（有効にすると reuse detection も働く）

**出典**:
- [OAuth 2.0 for Browser-Based Apps (draft)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-browser-based-apps) (IETF)
- [RFC 9700 — Best Current Practice for OAuth 2.0 Security](https://datatracker.ietf.org/doc/html/rfc9700) (IETF、§2.2.2・§4.14.2) ※2026-10-25 確認
- [Auth0: Refresh Token Rotation](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation) (Auth0、Dashboard での有効化と automatic reuse detection) ※2026-10-25 確認
- [OWASP: JWT Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html) (OWASP)

**バージョン**: パターン（実装依存）
**確信度**: 高
**最終更新**: 2026-10-25

#### 追加根拠 (2026-06-21) — hashToken() の実装: HMAC-SHA256 推奨

Rule #3 のコード例が参照している `hashToken()` の実装を明示する。plain SHA-256 より HMAC-SHA256 を推奨する理由:

- SHA-256 単体でも 256 bit CSPRNG 生成のリフレッシュトークンには十分な強度があるが、HMAC を使う場合は「ハッシュ値 + secret」の両方がなければトークンを逆算できない
- 最大の利点: **secret を変更するだけで、それまで発行したリフレッシュトークンを即時全失効できる**（侵害検知後の全セッション無効化に有効）
- DB に `HMAC-SHA256(refreshToken, secret)` を格納し、検索時も同じ計算で照合する

```typescript
// lib/auth/hash-token.ts
import { createHmac } from 'node:crypto';

export function hashToken(token: string): string {
  const secret = process.env.REFRESH_TOKEN_SECRET;
  if (!secret) throw new Error('REFRESH_TOKEN_SECRET is not set');
  return createHmac('sha256', secret).update(token).digest('hex');
}
```

環境変数 `REFRESH_TOKEN_SECRET` は 32 バイト以上のランダム値にする。JWT 署名鍵など他の鍵と共用しない。

**出典引用**:
> 「HMAC を使う場合は、さらに `secret` を見つけなければいけない」「secret を変更するだけで、それまで発行した token を即時全失効できる」
> ([リフレッシュトークンをDBに保存するとき何をやれば安全なんだろうか](https://zenn.dev/natsubate/articles/20db3ce3f7ae88), セクション "HMAC-SHA256") ※2026-06-21に実際にfetch成功

**確信度**: 既存（高）→ 高（実装詳細補完）

---

### 4. Cookie 認証では CSRF 対策を併用する

`SameSite=Lax` でも防ぎきれない CSRF 経路がある（GET ベースの破壊的操作、same-site の別オリジンからのリクエストなど）。
状態変更系のリクエストは、セッションに結合した署名付きトークン（Signed Double-Submit Cookie）か Synchronizer Token で防ぐ。
CSRF 対策が拒否できるのは「ブラウザが本人の Cookie 付きで送らされたリクエスト」だけで、認可の代わりにはならない。認証と所有者確認は別に必ず行う。

**根拠**:
- `SameSite=Lax` は GET / HEAD / OPTIONS の top-level navigation を許可する。POST フォームの自動送信は防ぐが、`<a>` クリックは防げない
- 状態変更を GET で行う API は CSRF に脆弱（そもそも GET でやらないのが第一原則）
- `SameSite` の境界はオリジンではなく登録可能ドメイン。兄弟サブドメインの脆弱性や乗っ取りがあると、そこからのリクエストは same-site 扱いになり、`Strict` でも Cookie が送られる
- OAuth callback など SameSite で守れないエンドポイントは個別対策必要
- Server Actions（Next.js）は POST だけを受け付け、`Origin` ヘッダーを `Host`（または `X-Forwarded-Host`）と比較して一致しなければ中断する。ただし Server Action は直接 POST で誰でも呼べるので、各 Action の中で認証・認可を行う必要がある（Next.js 公式 Data Security ガイド）
- OWASP Cheat Sheet は新規実装に Signed Double-Submit Cookie を推奨している。サーバー側の秘密鍵で「セッションに依存する値 + ランダム値」の HMAC-SHA-256 を計算してトークンにするため、サーバー側にトークンを保存する必要がなく、ログアウト等でセッションが切れれば同時にトークンも無効化できる。署名なし・セッション非結合の naive Double-Submit Cookie は、対象ドメインに Cookie を書ける攻撃者にバイパスされる
- トークンの Cookie には `__Host-` 接頭辞を付ける。`Secure`・`Path=/`・`Domain` なしが強制され、他のサブドメインから上書きできなくなる
- OWASP Cheat Sheet Series 本体も 2026-08-13 の改訂で「`SameSite` は CSRF トークンの代替ではなく defense in depth の一層として扱う」と明記を強化した。`SameSite=Strict` は top-level cross-site navigation を除外し、`SameSite=Lax` は safe HTTP メソッドの top-level cross-site navigation のみ許可する、という属性ごとの挙動差を公式ドキュメントが明示的に区別している

**コード例（Signed Double-Submit Cookie: セッション結合 + `__Host-` 接頭辞）**:
```ts
// app/_lib/csrf.ts
import 'server-only';
import { createHmac, randomBytes, timingSafeEqual } from 'node:crypto';
import { cookies } from 'next/headers';

const CSRF_COOKIE = '__Host-csrf';

function sign(sessionId: string, random: string): string {
  const secret = process.env.CSRF_SECRET;
  if (!secret) throw new Error('CSRF_SECRET is not set');
  // 長さ接頭辞付きで連結し、区切り位置の曖昧さをなくす
  const message = `${sessionId.length}:${sessionId}${random.length}:${random}`;
  return createHmac('sha256', secret).update(message).digest('base64url');
}

// ログイン成功時（Rule #1 で session Cookie を設定した直後）に呼ぶ。
// sessionId はトークンに含めない。値がローテーションする構成では、ローテーションのたびに呼び直す
export async function setCsrfCookie(sessionId: string) {
  const random = randomBytes(32).toString('base64url');
  const cookieStore = await cookies();
  cookieStore.set(CSRF_COOKIE, `${sign(sessionId, random)}.${random}`, {
    httpOnly: false,   // クライアント JS が読んでヘッダーに付ける
    secure: true,      // __Host- 接頭辞の必須条件
    sameSite: 'strict',
    path: '/',         // __Host- 接頭辞の必須条件（Domain は指定しない）
  });
}

// 状態変更系の Route Handler の冒頭で呼ぶ
export function verifyCsrf(req: Request, sessionId: string): boolean {
  // 明らかな cross-site リクエストは先に拒否する（ヘッダーがないことは same-origin の証拠にしない）
  if (req.headers.get('sec-fetch-site') === 'cross-site') return false;

  // Cookie ではなく、送信されたヘッダーの値を検証する
  const [mac, random] = req.headers.get('x-csrf-token')?.split('.') ?? [];
  if (!mac || !random) return false;

  const expected = Buffer.from(sign(sessionId, random));
  const actual = Buffer.from(mac);
  return expected.length === actual.length && timingSafeEqual(expected, actual);
}
```

```ts
// app/api/posts/route.ts — サーバー側で検証する
import { cookies } from 'next/headers';
import { verifyCsrf } from '@/app/_lib/csrf';
import { verifySession } from '@/app/_lib/dal/auth';

export async function POST(req: Request) {
  const sessionId = (await cookies()).get('session')?.value;
  if (!sessionId || !verifyCsrf(req, sessionId)) {
    return Response.json({ error: 'csrf_mismatch' }, { status: 403 });
  }

  // CSRF 検証を通っても認証・認可は省略しない（最終判定は DAL）
  const session = await verifySession();
  if (!session) {
    return Response.json({ error: 'unauthorized' }, { status: 401 });
  }
  // ... 入力検証 → Mutation DAL の呼び出し
  return Response.json({ ok: true });
}
```

```ts
// クライアント側: Cookie から読んでヘッダーに付けて送信
const csrf = document.cookie.match(/(?:^|;\s*)__Host-csrf=([^;]+)/)?.[1];
await fetch('/api/posts', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'X-CSRF-Token': csrf ?? '' },
  body: JSON.stringify({ title: 'hello' }),
});
```

```ts
// Bad: 署名なし・セッション非結合の naive Double-Submit Cookie
// 対象ドメインに Cookie を書ける攻撃者（兄弟サブドメインの侵害など）は、Cookie とヘッダーに同じ値を入れて通せる
const csrfCookie = (await cookies()).get('csrf-token')?.value;
if (!csrfCookie || csrfCookie !== req.headers.get('X-CSRF-Token')) {
  return Response.json({ error: 'csrf_mismatch' }, { status: 403 });
}
```

**Next.js Server Actions の CSRF と認可**:
```ts
// app/_actions/post.ts
'use server';

import { revalidatePath } from 'next/cache';
import { unauthorized } from 'next/navigation';
import { verifySession } from '@/app/_lib/dal/auth';
import { deletePost } from '@/app/_lib/dal/post.mutations';

export async function deletePostAction(id: string) {
  // Origin 検証は Next.js が自動で行うが、防げるのはブラウザ経由の CSRF だけ。
  // Server Action は curl などから直接 POST できるので、認証・認可は省略しない
  const session = await verifySession();
  if (!session) unauthorized();

  await deletePost(id); // 所有者確認は Mutation DAL が最終判定する
  revalidatePath('/posts');
}
```

```ts
// app/_lib/dal/post.mutations.ts
import 'server-only';
import { forbidden, unauthorized } from 'next/navigation';
import { db } from '@/lib/db';
import { verifySession } from './auth';

export async function deletePost(id: string) {
  const session = await verifySession();
  if (!session) unauthorized();

  // WHERE 句に所有者条件を含める。存在しない投稿と他人の投稿を区別せずに拒否する
  const { count } = await db.post.deleteMany({ where: { id, authorId: session.userId } });
  if (count === 0) forbidden();
}
```

```ts
// Bad: Origin 検証だけを頼りに、認証も所有者確認もせずに削除する
// → curl などから任意の id を指定して、誰でも他人の投稿を削除できる（IDOR）
'use server';

export async function deletePost(id: string) {
  await db.post.delete({ where: { id } });
}
```

`unauthorized()` / `forbidden()` は Experimental で、`next.config` の `experimental.authInterrupts: true` が前提。使わない場合は Error を throw するか、`{ ok: false }` のような結果を返す。
多層認可（Proxy / page / DAL）と CVE-2025-29927 の教訓は `practices/architecture/data-access-layer.md` #4 を参照。

**SameSite だけで CSRF トークンを省略してよい条件**（OWASP。すべて満たす場合に限る）:
- 自分が完全に管理していないホスト・サブドメイン・サービスと登録可能ドメインを共有していない
- GET など安全なメソッドで状態を変更するエンドポイントがない
- セッション Cookie が `SameSite=Strict`、または `SameSite=Lax` + `__Host-` 接頭辞
- 状態変更のエンドポイントで Origin / Referer を検証している（defense in depth）
- `SameSite` を実装しないブラウザの利用者を除外してよい

満たせない場合や、`Strict` の UX 課題（外部リンクからの遷移でセッションが失われる）を避けたい場合は、`Lax` + CSRF トークンが現実解。

**判断軸**:
| Cookie SameSite | CSRF トークン | 状態 |
|---|---|---|
| `Strict` | 上の条件をすべて満たす場合のみ省略可 | UX 制約あり。兄弟サブドメインなど same-site の別オリジンからの攻撃は SameSite では防げない |
| `Lax` | **必須** | 一般的な構成、推奨 |
| `None` | **必須** + Origin チェック | cross-origin API、最大限の注意 |

**出典**:
- [OWASP: CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) (OWASP、Signed Double-Submit Cookie・`__Host-` 接頭辞・SameSite だけで足りる条件・Fetch Metadata) ※2026-10-25 確認
- [Next.js Docs: Data Security](https://nextjs.org/docs/app/guides/data-security) (Next.js 公式、Server Actions の Origin / Host 比較と、Action 内での認証・認可・IDOR 対策) ※2026-10-25 確認
- [Next.js Docs: forbidden](https://nextjs.org/docs/app/api-reference/functions/forbidden) (Next.js 公式、Experimental で `experimental.authInterrupts` が必要) ※2026-10-25 確認
- [Next.js Docs: Security - Server Actions](https://nextjs.org/blog/security-nextjs-server-components-actions) (Next.js 公式)
- [web.dev: Schemeful Same-Site](https://web.dev/articles/schemeful-samesite) (web.dev)
- [OWASP Cheat Sheet に見る CSRF 対策トークンの作り方](https://zenn.dev/natsubate/articles/b2ec11005901a2) (Zenn、Signed Double-Submit Cookie の HMAC 署名によるステートレス実装の解説) ※2026-08-05に実際にfetch成功
- [OWASP CheatSheetSeries commit: docs: qualify SameSite cookie guidance (#2362)](https://github.com/OWASP/CheatSheetSeries/commit/7deb20b3217026015921ee88293bb7384b30247d) (OWASP 公式リポジトリ、SameSite の位置づけを defense in depth として明文化した改訂) ※2026-08-13に実際にfetch成功

> "この Signed Double-Submit Cookie ではサーバ側で保存する必要がない。"
> ([OWASP Cheat Sheet に見る CSRF 対策トークンの作り方](https://zenn.dev/natsubate/articles/b2ec11005901a2), セクション "メリット") ※2026-08-05に実際にfetch成功

> "Treat SameSite as defense in depth against CSRF, not as a replacement for a CSRF token."
> ([OWASP CheatSheetSeries commit #2362](https://github.com/OWASP/CheatSheetSeries/commit/7deb20b3217026015921ee88293bb7384b30247d), セクション "Cross-Site Request Forgery Prevention Cheat Sheet — SameSite 記述の改訂差分") ※2026-08-13に実際にfetch成功

**バージョン**: Next.js 16+（`unauthorized()` / `forbidden()` は `experimental.authInterrupts` が前提）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 5. クライアント JS は「ログイン状態」のみ持ち、トークンを直接持たない

JS から認証状態を判定する場合は、Cookie に含まれる **非機密なフラグ**（例: `logged-in=1`、または短期 JWT のクレーム）で判断し、
**トークン本体や個人情報は持たない**。JS が触れる情報は XSS で漏れる前提で設計する。

**根拠**:
- ログイン状態の表示（ユーザー名・アバター・ナビゲーション）は HttpOnly Cookie だけでは取れない
- 解決策: 表示に必要な非機密フィールドだけを、Server Component の props か JS から読める別 Cookie で渡す
- 取得経路で安全性は変わらない。Server Component から Client Component に渡した props も RSC Payload としてブラウザに届き（`practices/architecture/data-access-layer.md` #3）、表示用 Cookie も JS から読める。漏洩範囲を決めるのは、ブラウザに送るフィールドの範囲
- JS が持つ情報は最小化：認証判定 + 表示用の最小データのみ
- Auth0 等は「`auth0.is.authenticated` Cookie」のような非機密判定 Cookie を提供

**実装パターン**:

```ts
// app/_lib/dal/user.ts — 表示に必要なフィールドだけを返す
import 'server-only';
import { db } from '@/lib/db';
import { verifySession } from './auth';

export async function getCurrentUserDisplay() {
  const session = await verifySession();
  if (!session) return null;
  return db.user.findUnique({
    where: { id: session.userId },
    select: { name: true, avatarUrl: true }, // email・role などは送らない
  });
}
```

```tsx
// パターン 1: Server Component で取得し、最小限のフィールドだけを渡す
// app/layout.tsx
import { getCurrentUserDisplay } from '@/app/_lib/dal/user';
import { Header } from './_components/Header';

export default async function Layout({ children }: { children: React.ReactNode }) {
  // 表示のための取得だけ。認可の判定は DAL・page・Server Action で行い、layout では行わない
  const user = await getCurrentUserDisplay();

  return (
    <html>
      <body>
        <Header user={user} />  {/* Client Component に渡した値はブラウザに届く */}
        {children}
      </body>
    </html>
  );
}
```

```ts
// パターン 2: 非機密 Cookie + Client Component
// app/_lib/user-display-cookie.ts — ログイン処理（Route Handler / Server Function）から呼ぶ
import { cookies } from 'next/headers';

export async function setUserDisplayCookie(user: { name: string; avatarUrl: string }) {
  const cookieStore = await cookies();
  cookieStore.set('user-display', JSON.stringify({ name: user.name, avatarUrl: user.avatarUrl }), {
    httpOnly: false,    // JS で読む = XSS でも読まれる
    secure: true,
    sameSite: 'lax',
    path: '/',
    // 注意: パスワード・メール・トークンは絶対に含めない
  });
}
```

```tsx
// app/_components/UserMenu.tsx
'use client';

import { useEffect, useState } from 'react';
import { Avatar, SignInButton } from './ui';

type UserDisplay = { name: string; avatarUrl: string };

function readUserDisplay(): UserDisplay | null {
  const raw = document.cookie.match(/(?:^|;\s*)user-display=([^;]*)/)?.[1];
  if (!raw) return null;
  try {
    return JSON.parse(decodeURIComponent(raw)) as UserDisplay;
  } catch {
    return null;
  }
}

export function UserMenu() {
  const [display, setDisplay] = useState<UserDisplay | null>(null);
  useEffect(() => {
    setDisplay(readUserDisplay()); // document はブラウザでしか読めないので effect で読む
  }, []);

  if (!display) return <SignInButton />;
  return <Avatar src={display.avatarUrl} name={display.name} />;
}
```

**やってはいけないこと**:
```tsx
// Bad: JWT を localStorage に保存して claims を読み取る
const token = localStorage.getItem('access_token');
const { email, role, permissions } = decodeJwt(token);
// → XSS で全てが盗まれる

// Bad: API や Server Component からユーザーのフルプロファイルを渡して持ち回る
// （どちらの経路でもブラウザに届いた値は XSS で読まれる。表示に必要なフィールドだけ返す）

// Bad: ログイン状態を localStorage で判定
if (localStorage.getItem('isLoggedIn')) { ... }
// → 攻撃者がブラウザコンソールから書き換え可能
```

**XSS 発生時の被害最小化**:
- JS が知っている全てが流出する前提で、トークン・PII を渡さない
- HttpOnly Cookie で守られたトークン → 盗まれない（XSS でも）
- 短命なアクセストークン + リフレッシュ rotation → 流出してもすぐ無効化

**判断チェックリスト**:
- [ ] JS で `localStorage` / `sessionStorage` にトークンを保存していないか？
- [ ] JWT を JS で decode し、内部のクレームをアプリ全体で参照していないか？
- [ ] Client Component への props や `useState(user)` でフルプロファイルを持ち回っていないか？（必要な属性だけ）
- [ ] 表示用 Cookie に email / phone / 住所等の PII が入っていないか？

**出典**:
- [OWASP: HTML5 Security - Web Storage](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html#local-storage) (OWASP)
- [Auth0: Sessions](https://auth0.com/docs/manage-users/sessions) (Auth0)
- [Vercel: Authentication](https://vercel.com/guides/nextjs-authentication) (Vercel)
- [Next.js Docs: Data Security](https://nextjs.org/docs/app/guides/data-security) (Next.js 公式、Client Component に渡す props を必要なフィールドに絞る) ※2026-10-25 確認

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 6. DPoP + Web Crypto API `extractable: false` で API リクエストに署名し、トークン単体の持ち出し・別端末での再利用を防ぐ

HttpOnly Cookie が使えない構成（別ドメイン API への直接呼び出し・WebSocket・Native-like PWA）では、
DPoP（RFC 9449）と Web Crypto API の `extractable: false` を組み合わせ、ブラウザ外に秘密鍵が出られない設計にする。
IndexedDB は Structured Clone Algorithm により `CryptoKey` オブジェクトをそのまま永続化できる唯一のWeb Storageであり、
`extractable: false` の鍵は `.exportKey()` で文字列・バイト列へのエクスポートが不可能なため、
XSS でコードを実行されても攻撃者は鍵を外部に持ち出せない。
ただし XSS が起きているあいだは、攻撃者のコードがページ内でその鍵を使って新しい DPoP Proof を作れる（RFC 9449 §11.4）。DPoP は XSS 対策の代わりにならず、XSS 対策（`practices/security/xss.md`・`practices/security/csp.md`）は別に必須。

**根拠**:
- DPoP は各リクエストに ECDSA 署名付き JWT を添付し、トークン単体を盗んでも別デバイスから再利用できなくする（RFC 9449 §1）
- `window.crypto.subtle.generateKey()` で `extractable: false` を指定した `CryptoKey` は署名には使えるが `.exportKey()` で外に出せない
- `localStorage` / `sessionStorage` は文字列のみ保存でき `CryptoKey` を格納できない。IndexedDB だけが Structured Clone Algorithm で `CryptoKey` を格納できる
- 「ブラウザ外に秘密鍵が出られない」+「盗まれたトークンを別デバイスから再送できない」の2条件が揃い、トークンや鍵を盗んで別の端末から使う攻撃を防げる。一方、RFC 9449 §11.4 のとおり、クライアントがオンラインのあいだは悪意あるコードが鍵を使って Proof を作り続けられる
- OWASP JWT Cheat Sheet も「DPoP Proof JWT」を鍵所有証明の正式なユースケースとして明記しており、署名方式は公開鍵暗号（EdDSA / ECDSA 等）を優先しHMACのような対称鍵方式は避けるべきとしている

**コード例（鍵生成・永続化・DPoP Proof 作成）**:
```javascript
import { openDB } from 'idb';

// 1. 鍵ペアを生成（extractable: false で外部エクスポート不可）
const keyPair = await window.crypto.subtle.generateKey(
  { name: "ECDSA", namedCurve: "P-256" },
  false,           // extractable: false — 鍵はブラウザ外に出られない
  ["sign", "verify"]
);

// 2. IndexedDB に CryptoKey ごと保存（Structured Clone で保持）
const db = await openDB('auth-keys', 1, {
  upgrade(db) { db.createObjectStore('keys'); }
});
await db.put('keys', keyPair, 'dpop-keypair');
```

```javascript
import { openDB } from 'idb';
import { SignJWT, exportJWK, base64url } from 'jose';

// 3. DPoP Proof の生成（API 呼び出しごとに署名）
async function createDPoPProof(method, url, accessToken) {
  const db = await openDB('auth-keys', 1);
  const { privateKey, publicKey } = await db.get('keys', 'dpop-keypair');

  const target = new URL(url);
  const htu = `${target.origin}${target.pathname}`; // query と fragment は含めない（RFC 9449 §4.2）

  // アクセストークンの SHA-256 を base64url にしたもの。リソースサーバーへの送信では必須
  const digest = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(accessToken));
  const ath = base64url.encode(new Uint8Array(digest));

  const jwk = await exportJWK(publicKey); // 公開鍵は常に export できる。秘密鍵は含めない
  return new SignJWT({ htm: method, htu, ath })
    .setProtectedHeader({ typ: 'dpop+jwt', alg: 'ES256', jwk })
    .setJti(crypto.randomUUID())  // 再送攻撃防止
    .setIssuedAt()
    .sign(privateKey);
}

// 4. 送信: アクセストークンは DPoP スキーム、Proof は DPoP ヘッダーで送る（RFC 9449 §7.1）
const url = 'https://api.example.com/data?page=2';
const proof = await createDPoPProof('GET', url, accessToken); // accessToken はメモリにだけ保持（Rule #1）
await fetch(url, {
  headers: { Authorization: `DPoP ${accessToken}`, DPoP: proof },
});
```

DPoP を使う構成（別ドメイン API・WebSocket・Native アプリ相当の PWA）の判断は Rule #1 の判断軸にまとめている。

**アンチパターン**:
- `extractable: true` で鍵を生成 → localStorage にシリアライズして保存でき XSS で盗まれる
- sessionStorage + CryptoKey — sessionStorage は文字列のみ。CryptoKey を直接保存できない
- HttpOnly Cookie で対応可能なケースに DPoP を導入 → 複雑さのコストが上回る

**出典引用**:
> "IndexedDB is the sole location capable of storing `CryptoKey` objects directly." — Structured Clone Algorithm が CryptoKey をシリアライズせずそのまま保持できる唯一の Web Storage
> ([DPoP + IndexedDB でフロントエンドセッションをがっちり守る](https://zenn.dev/kuboon/articles/e5d955f2cba108), セクション "Why IndexedDB?") ※2026-05-24に実際にfetch成功

**出典**:
- [RFC 9449 — OAuth 2.0 Demonstrating Proof of Possession (DPoP)](https://datatracker.ietf.org/doc/html/rfc9449) (IETF、§4.2 `typ`・`htu`・`ath`、§7.1 DPoP スキーム、§11.4 Untrusted Code in the Client Context) ※2026-10-25 確認
- [DPoP + IndexedDB でフロントエンドセッションをがっちり守る](https://zenn.dev/kuboon/articles/e5d955f2cba108) (Zenn、ブラウザ実装ガイド) ※2026-05-24にfetch成功
- [MDN: SubtleCrypto.generateKey()](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/generateKey) (MDN Web Docs)
- [MDN: IndexedDB API](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API) (MDN Web Docs)
- [OWASP JWT Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/commit/f54d8ded2764010ecacd0deb87e1586da32d41dd) (OWASP CheatSheetSeries、DPoP Proof JWT の公式ユースケース追加・署名方式推奨表を追記した commit) ※2026-07-04に実際にfetch成功

**出典引用**:
> "A DPoP Proof JWT can be used to prove possession of a private key."
> ([OWASP JWT Cheat Sheet — Introduction section 更新](https://github.com/OWASP/CheatSheetSeries/commit/f54d8ded2764010ecacd0deb87e1586da32d41dd), セクション "Introduction") ※2026-07-04に実際にfetch成功

**バージョン**: モダンブラウザ（Web Crypto API + IndexedDB 対応、Firefox/Chrome/Safari）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 7. トークンリフレッシュの競合はタブ内・タブ間の2階層で同時実行制御する

素朴な「401 を受けたらリフレッシュする」実装は、同一タブ内の複数リクエストが同時に 401 を受けた場合と、
複数ブラウザタブが同時にリフレッシュを試みる場合の両方で競合し、
片方が無効化された旧トークンで上書きしてセッションを壊す。
タブ内は single-flight Promise + 世代番号、タブ間は `Web Locks API` で、それぞれ別の同時実行制御が必要。

**根拠**:
- 同一タブ内: 複数リクエストが同時に 401 を受けると、リフレッシュ処理を複数回発火してしまう。進行中のリフレッシュ Promise を共有（single-flight）し、リフレッシュ完了後に届いた「リフレッシュ前に発行された古いレスポンス」による再リフレッシュを、リクエスト送信時の世代番号との比較で防ぐ
- 複数タブ間: 各ブラウザタブは独立した JS ランタイムを持つため、single-flight Promise はタブをまたいで共有できない。`navigator.locks`（Web Locks API）でタブ横断の排他ロックを取得し、ロック取得中に他タブが既にリフレッシュ済みでないか `localStorage` の更新世代（非機密の数値）で二重チェックしてから実際のリフレッシュ処理を実行する
- タブ間で共有するのは更新世代やタイムスタンプのような非機密情報だけにする。トークン本体は HttpOnly Cookie（Rule #1）か各タブのメモリに置き、localStorage・BroadcastChannel で共有しない（Rule #1・#5、`practices/security/micro-frontend.md` #3）。HttpOnly Cookie なら、他タブが更新した Cookie はブラウザが全タブで共有するので、再送するだけで新しいトークンが使われる
- 「直近 5 秒以内なら更新済み」のような時間窓で判定すると、401 を受けたリクエストより前に終わった更新まで「更新済み」とみなし、失効済みのトークンで再送してしまう。リクエスト送信時の世代と比較する
- リフレッシュトークンが rotation（Rule #3）で使い捨てになっている場合、この種の競合は「同じリフレッシュトークンを2回使おうとした」と誤検知され、正規ユーザーのセッションが強制ログアウトされる実害につながる

**コード例（タブ内: single-flight + 世代番号 / タブ間: Web Locks API + 世代の二重チェック）**:
```ts
// トークン本体は HttpOnly Cookie（Rule #1）。JS が持つのは更新世代（非機密）だけ
const GENERATION_KEY = 'auth_refresh_generation';
const readGeneration = (): number => Number(localStorage.getItem(GENERATION_KEY) ?? 0);

let refreshPromise: Promise<void> | null = null;

// タブ間: Web Locks で排他し、ロック取得後に世代を再確認する
function refreshAcrossTabs(seenGeneration: number): Promise<void> {
  return navigator.locks.request('token-refresh', async () => {
    if (readGeneration() > seenGeneration) return; // 他タブが更新済み。新しい Cookie は共有されている

    const res = await fetch('/api/auth/refresh', { method: 'POST' }); // Rule #3 のエンドポイント
    if (!res.ok) throw new Error('refresh_failed'); // 呼び出し側でログイン画面へ
    localStorage.setItem(GENERATION_KEY, String(readGeneration() + 1));
  });
}

// タブ内: 進行中のリフレッシュに相乗りする（single-flight）
function refreshOnce(seenGeneration: number): Promise<void> {
  if (!refreshPromise) {
    refreshPromise = refreshAcrossTabs(seenGeneration).finally(() => {
      refreshPromise = null;
    });
  }
  return refreshPromise;
}

export async function fetchWithRefresh(input: RequestInfo | URL, init?: RequestInit): Promise<Response> {
  const seenGeneration = readGeneration(); // 送信時点の世代
  const res = await fetch(input, init);
  if (res.status !== 401) return res;

  // 送信後に（このタブか他タブで）更新が終わっていれば、リフレッシュせずに再送する
  if (readGeneration() === seenGeneration) {
    await refreshOnce(seenGeneration);
  }
  return fetch(input, init);
}
```

**出典**:
- [トークン更新の競合を防ぐ：single-flightと世代番号による遅延401対策](https://zenn.dev/itakuradev/articles/token-refresh-single-flight-generation) (Zenn、同一タブ内の競合対策) ※2026-07-25に実際にfetch成功
- [The Multiple Browser Tab Token Trap: Synchronizing JWT Refresh Across Browser Tabs](https://dev.to/nileshcodehub/the-multiple-browser-tab-token-trap-synchronizing-jwt-refresh-across-browser-tabs-45f6) (dev.to、複数タブ間の競合対策) ※2026-07-25に実際にfetch成功

**出典引用**:
> "Promiseは、更新中の重複を防ぎます。世代番号は、更新終了後に届いた古いレスポンスから、不要な更新が再開されることを防ぎます。"
> ([トークン更新の競合を防ぐ](https://zenn.dev/itakuradev/articles/token-refresh-single-flight-generation), セクション "single-flightと世代番号の役割は異なる") ※2026-07-25に実際にfetch成功

> "By combining Axios Interceptors with the Web Locks API, we eliminated cross-tab token refresh race conditions completely."
> ([The Multiple Browser Tab Token Trap](https://dev.to/nileshcodehub/the-multiple-browser-tab-token-trap-synchronizing-jwt-refresh-across-browser-tabs-45f6), セクション "Section 8: Conclusion") ※2026-07-25に実際にfetch成功

**バージョン**: `navigator.locks`（Web Locks API: Chrome 69+, Firefox 96+, Safari 15.4+）
**確信度**: 中（それぞれ単著者・非公式の1記事だが、「タブ内」「タブ間」という異なるスコープで同じ知見（素朴なトークンリフレッシュは競合する）を補完的に裏付けているため、単独記事より一段高い確信度として扱う）
**最終更新**: 2026-10-25

---

### 8. JWT の denylist キーには生の JWT やそのハッシュ値ではなく `jti` + `iss` クレームを使う

Rotation（Rule #3）が使えないステートレスな JWT（サードパーティ発行・rotation非対応の既存トークン等）を失効させる場合、denylist（失効リスト）のキー設計を誤ると JWT malleability によってバイパスされうる。生の JWT やその単純ハッシュをキーにせず、`jti`（JWT ID）と `iss`（発行者）クレームの組み合わせをキーにする。

**根拠**:
- ヘッダーとクレームは署名対象なので、順序や空白を変えれば署名検証は失敗する。表現差が生じるのは、OWASP が挙げる 2 つのケース: 実装が JWT を厳密にパースしない場合と、ECDSA 署名の malleability（ES256 では署名値 (r, s) を (r, n−s) に変えても検証が通る）。どちらも、同じ内容のトークンが別の文字列のまま検証を通るので、生の JWT やその単純ハッシュをキーにすると denylist をすり抜ける
- `jti` + `iss` はトークンの表現差異に影響されない安定した識別子であり、denylist のキーとして安全
- OWASP は Token Status List（TSL）のような集約的失効の仕組みも新たに正式ガイダンスとして追記している

**コード例**:
```ts
import { createHash } from 'node:crypto';
import { createRemoteJWKSet, jwtVerify } from 'jose';
import { denylist } from '@/lib/auth/denylist';

const ISSUER = 'https://issuer.example.com';
const JWKS = createRemoteJWKSet(new URL(`${ISSUER}/.well-known/jwks.json`));

// Bad: 生のJWTやその単純ハッシュをdenylistキーにする（malleabilityでバイパスされうる）
await denylist.add(createHash('sha256').update(rawJwt).digest('hex'));

// Good: 署名を検証してから jti + iss を取り出してキーにする（トークン表現の違いに影響されない）
const { payload } = await jwtVerify(rawJwt, JWKS, { issuer: ISSUER, audience: 'https://api.example.com' });
if (!payload.jti || !payload.iss) throw new Error('jti and iss are required for revocation');
// `${iss}:${jti}` の連結は iss 自体に ':' を含むため一意にならない。区切りが衝突しない形にする
await denylist.add(JSON.stringify([payload.iss, payload.jti]));
```

**出典引用**:
> "Using the raw JWT or a secure hash of the JWT as the denylist key is not safe and might expose the application to denylist bypass through JWT malleability"
> ([OWASP JWT Cheat Sheet — Cleanup/streamline JWT cheat sheet (#2283)](https://github.com/OWASP/CheatSheetSeries/commit/8d23424dfe51b7639d9547a642a7fd4b7a76e135), セクション "JWT Denylist") ※2026-08-01に実際にfetch成功

**出典**:
- [OWASP: JSON Web Token Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_Cheat_Sheet.html) (OWASP、JWT denylist の節。malleability の原因として非厳密なパースと ECDSA 署名の malleability を挙げる) ※2026-10-25 確認

**バージョン**: JWT（RFC 7519）全般
**確信度**: 高
**最終更新**: 2026-10-25

---

### 9. 署名済み JWT（JWS）はペイロードを暗号化しない — 機密情報を含めるなら JWE でネスト暗号化する

署名（JWS）は完全性・発行者認証を提供するが機密性は提供しない。JWT のペイロードは base64url エンコードされているだけで暗号化されていないため、トークンを入手した者は全クレームを読める。個人情報や機密データをクレームに含めたい場合は、署名済み JWT をさらに JWE で暗号化する「ネスト JWT」構成にする。それ以外は機密データをトークンに含めずサーバー側に保持する設計を優先する。

**根拠**:
- JWS はメッセージの改ざん検知・発行者認証のための署名であり、ペイロードは平文相当（base64url）でクライアント側から読み取れる
- JWE は機密性と暗号文の完全性を提供するが、発行者認証は提供しない——JWE 単体では「誰が発行したか」を保証できないため、機密性と認証の両方が必要なら「まず JWS で署名し、その結果を JWE で暗号化する」ネスト構成にする
- JWE を使う場合も `alg`（鍵管理）・`enc`（コンテンツ暗号化）はアプリケーションが許可するアルゴリズムのみを allowlist 化し、圧縮を避けるなどの追加要件がある

**コード例**:
```ts
// Bad: 機密情報をJWSのみのJWTクレームに含める(誰でもデコードして読める)
const token = await new SignJWT({ ssn: user.ssn, salary: user.salary })
  .setProtectedHeader({ alg: 'RS256' })
  .sign(privateKey);

// Good: 機密クレームを含めない、または署名済み JWT（JWS）を JWE でネストする
// （RFC 7519 の Nested JWT。署名済みトークン文字列そのものを暗号化し、cty: 'JWT' を付ける）
const signedJwt = await new SignJWT({ sub: user.id, role: user.role })
  .setProtectedHeader({ alg: 'RS256' })
  .sign(privateKey);
const nestedJwt = await new CompactEncrypt(new TextEncoder().encode(signedJwt))
  .setProtectedHeader({ alg: 'RSA-OAEP-256', enc: 'A256GCM', cty: 'JWT' })
  .encrypt(publicKey);
```

**出典引用**:
> "The payload is only base64url encoded, not encrypted, so anyone who obtains the token can read every claim."
> ([OWASP JSON Web Token Cheat Sheet — Add Token Confidentiality and JWE section (#2321)](https://github.com/OWASP/CheatSheetSeries/commit/327812ee76aac6a87e32fbdce5b5d1cce39b5741), セクション "Signed JWTs are not confidential") ※2026-09-27に実際にfetch成功

> "JWE provides confidentiality and ciphertext integrity, not issuer authentication"
> ([OWASP JSON Web Token Cheat Sheet — Add Token Confidentiality and JWE section (#2321)](https://github.com/OWASP/CheatSheetSeries/commit/327812ee76aac6a87e32fbdce5b5d1cce39b5741), セクション "Using JWE") ※2026-09-27に実際にfetch成功

**出典**:
- [OWASP JSON Web Token Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/JSON_Web_Token_Cheat_Sheet.md) (OWASP公式、Token Confidentiality and JWE セクション追加) ※2026-09-27に実際にfetch成功

**バージョン**: JWT（RFC 7519）/ JWE（RFC 7516）全般
**確信度**: 高
**最終更新**: 2026-09-27

---
