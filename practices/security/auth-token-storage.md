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

**例外（localStorage が必要なケース）**:
- WebSocket 接続: Cookie が送れないなら短命のトークンを localStorage に置く許容（リスク受容）
- マイクロフロントエンドでサブドメインまたぐ SSO: Cookie の Domain 設定で対応するのが第一選択
- どうしても JS でトークンが必要 → アクセストークンを **5-15 分の短命** にし、リフレッシュは Cookie で（Rule 3）

**判断軸**:
| シチュエーション | トークン保存 | 理由 |
|---|---|---|
| Web アプリ（一般的） | HttpOnly Cookie | XSS 耐性最優先 |
| BFF パターン | HttpOnly Cookie | BFF が token を持ち、フロントは Cookie のみ |
| ネイティブアプリ | Secure Storage (Keychain / Keystore) | OS のセキュアストレージ |
| 完全 SPA + 別ドメイン API | 短命メモリ + リフレッシュ Cookie | XSS リスクを最小化 |

**出典**:
- [OWASP: Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) (OWASP)
- [OWASP: HTML5 Security Cheat Sheet - Local Storage](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html#local-storage) (OWASP)
- [Auth0: Token Storage](https://auth0.com/docs/secure/security-guidance/data-security/token-storage) (Auth0)
- [Next.js Docs: cookies()](https://nextjs.org/docs/app/api-reference/functions/cookies) (Next.js 公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-16

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
import { cookies } from 'next/headers';

// セッション Cookie（短期）
cookies().set('session', sessionToken, {
  httpOnly: true,
  secure: true,
  sameSite: 'lax',
  path: '/',
  maxAge: 60 * 60 * 24,  // 1 day
});

// CSRF トークン（JS から読む必要があるので HttpOnly false）
cookies().set('csrf-token', csrfToken, {
  httpOnly: false,        // JS で読み取り header に付けるため
  secure: true,
  sameSite: 'strict',     // strict で OK（同一サイト内のみ）
  path: '/',
});

// リフレッシュトークン（長期、より厳格）
cookies().set('refresh', refreshToken, {
  httpOnly: true,
  secure: true,
  sameSite: 'strict',     // refresh は strict で良い
  path: '/api/auth',      // refresh エンドポイントのみに送る
  maxAge: 60 * 60 * 24 * 30,
});
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

**バージョン**: 全モダンブラウザ
**確信度**: 高
**最終更新**: 2026-05-16

---

### 3. リフレッシュトークン rotation を実装する

リフレッシュトークンは一度使ったら無効化し、新しいリフレッシュトークン + 新しいアクセストークンを発行する（rotation）。
盗まれたリフレッシュトークンの再利用を検知して全セッションを失効させる。

**根拠**:
- アクセストークンは短命（5-15 分）でも、リフレッシュトークンが流出すると無期限に認証突破される
- Rotation で「同じリフレッシュトークンが 2 回使われた」を検知できる（攻撃 + 正規ユーザーの両方が使った状態）
- OAuth 2.1 Best Current Practice (BCP) で必須化が議論されている
- Refresh token reuse detection は Auth0・Okta・Clerk のデフォルト機能

**コード例（基本フロー）**:
```ts
// app/api/auth/refresh/route.ts
import { cookies } from 'next/headers';

export async function POST() {
  const cookieStore = await cookies();
  const refreshToken = cookieStore.get('refresh')?.value;

  if (!refreshToken) {
    return Response.json({ error: 'no_refresh_token' }, { status: 401 });
  }

  const tokenRecord = await db.refreshToken.findUnique({
    where: { token: hashToken(refreshToken) },
  });

  if (!tokenRecord) {
    // 未知のトークン → 全セッション失効（攻撃の可能性）
    return Response.json({ error: 'invalid_token' }, { status: 401 });
  }

  // Rotation: 一度使ったら無効化
  if (tokenRecord.usedAt) {
    // すでに使用済みのトークンを再利用 → 攻撃検知
    await db.refreshToken.deleteMany({ where: { userId: tokenRecord.userId } });
    cookieStore.delete('session');
    cookieStore.delete('refresh');
    await notifySecurityTeam(tokenRecord.userId, 'refresh_token_reuse');
    return Response.json({ error: 'token_reuse_detected' }, { status: 401 });
  }

  await db.refreshToken.update({
    where: { id: tokenRecord.id },
    data: { usedAt: new Date() },
  });

  // 新しいトークンペアを発行
  const newAccess = generateAccessToken(tokenRecord.userId, 15 * 60);      // 15min
  const newRefresh = generateRefreshToken();
  await db.refreshToken.create({
    data: {
      token: hashToken(newRefresh),
      userId: tokenRecord.userId,
      family: tokenRecord.family,  // 同一系列であることを記録
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
    },
  });

  cookieStore.set('session', newAccess, { httpOnly: true, secure: true, sameSite: 'lax', maxAge: 15 * 60 });
  cookieStore.set('refresh', newRefresh, { httpOnly: true, secure: true, sameSite: 'strict', path: '/api/auth', maxAge: 30 * 24 * 60 * 60 });

  return Response.json({ ok: true });
}
```

**Family 機構**:
- 一連の rotation を「family」として記録
- 同じ family 内の古いトークンが使われたら → family 全体（= そのセッション）を失効
- ユーザーが「別端末で使い続けている」場合に巻き込まれないよう、family は端末単位で分ける運用も

**ライブラリでの実装**:
- **NextAuth.js (Auth.js)**: refresh token rotation を自前で実装する必要あり（公式アダプタ + ロジック追加）
- **Clerk**: rotation 標準対応、自動
- **Auth0**: refresh_token_rotation オプションで有効化

**出典**:
- [OAuth 2.0 for Browser-Based Apps (draft)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-browser-based-apps) (IETF)
- [Auth0: Refresh Token Rotation](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation) (Auth0)
- [OWASP: JWT Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html) (OWASP)

**バージョン**: パターン（実装依存）
**確信度**: 高
**最終更新**: 2026-05-16

---

### 4. Cookie 認証では CSRF 対策を併用する

`SameSite=Lax` でも防ぎきれない CSRF 経路がある（GET ベースの破壊的操作など）。
状態変更系のリクエストは CSRF トークン（Double-Submit Cookie / Synchronizer Token）で防ぐ。

**根拠**:
- `SameSite=Lax` は GET / HEAD / OPTIONS の top-level navigation を許可する。POST フォームの自動送信は防ぐが、`<a>` クリックは防げない
- 状態変更を GET で行う API は CSRF に脆弱（そもそも GET でやらないのが第一原則）
- OAuth callback など SameSite で守れないエンドポイントは個別対策必要
- Server Actions（Next.js）は内部で CSRF 対策済み（Origin チェック）

**コード例（Double-Submit Cookie パターン）**:
```ts
// middleware.ts — リクエスト時に CSRF トークンを生成
import { NextResponse } from 'next/server';

export function middleware(request: NextRequest) {
  const response = NextResponse.next();
  let csrf = request.cookies.get('csrf-token')?.value;

  if (!csrf) {
    csrf = crypto.randomUUID();
    response.cookies.set('csrf-token', csrf, {
      httpOnly: false,    // JS で読めるようにする（header に付けるため）
      secure: true,
      sameSite: 'strict',
      path: '/',
    });
  }
  return response;
}

// クライアント側: ヘッダーに付けて送信
const csrf = document.cookie.match(/csrf-token=([^;]+)/)?.[1];
fetch('/api/posts', {
  method: 'POST',
  headers: { 'X-CSRF-Token': csrf ?? '' },
  body: JSON.stringify({ /* ... */ }),
});

// サーバー側: Cookie のトークンと Header のトークンを照合
export async function POST(req: Request) {
  const csrfCookie = (await cookies()).get('csrf-token')?.value;
  const csrfHeader = req.headers.get('X-CSRF-Token');
  if (!csrfCookie || csrfCookie !== csrfHeader) {
    return Response.json({ error: 'csrf_mismatch' }, { status: 403 });
  }
  // ...
}
```

**Next.js Server Actions の CSRF**:
```ts
// Server Action は Origin ヘッダーを自動検証する（同一オリジンのみ受付）
'use server';

export async function deletePost(id: string) {
  // 別オリジンからの呼び出しは Next.js が自動拒否
  await db.post.delete({ where: { id } });
}
```

**SameSite=Strict で CSRF が不要なケース**:
- 全 Cookie が `Strict` で、cross-site からのリクエストが完全に Cookie を送らない
- 同時に、すべての破壊的操作が POST/PUT/DELETE で、API が Origin / Referer を検証している
- ただし `Strict` は UX 課題があるため、`Lax` + CSRF トークンが現実解

**判断軸**:
| Cookie SameSite | CSRF トークン | 状態 |
|---|---|---|
| `Strict` | 不要（推奨） | 最も安全だが UX 制約 |
| `Lax` | **必須** | 一般的な構成、推奨 |
| `None` | **必須** + Origin チェック | cross-origin API、最大限の注意 |

**出典**:
- [OWASP: CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) (OWASP)
- [Next.js Docs: Security - Server Actions](https://nextjs.org/blog/security-nextjs-server-components-actions) (Next.js 公式)
- [web.dev: Schemeful Same-Site](https://web.dev/articles/schemeful-samesite) (web.dev)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-16

#### 追加根拠 (2026-05-16) — 手動取り込み

新たに sstf-5461-admin-app チームドキュメント（原典: akfm_sato 氏 Zenn book）から以下の知見を追加:

**CVE-2025-29927 の実例**:
Next.js middleware 単体での認可チェックには重大な脆弱性事例があり、特定の HTTP ヘッダ操作で middleware をバイパスできた。
この事案から「middleware/proxy 単体の認可は十分ではない」ことが教訓化されている。

**多層認可（defense-in-depth）の必要性**:
- 第1層: proxy.ts (旧 middleware) — 早期リダイレクト・粗いガード
- 第2層: page/layout — Server Component 内での認可
- 第3層: DAL (Data Access Layer) — データアクセス直前の最終チェック

3 層で重複してチェックすることで、いずれか 1 層に脆弱性があってもデータ漏洩を防ぐ。

**コード例**:
```typescript
// Good: DAL レベルでの認可
// app/lib/dal/posts.ts
import { auth } from '@/lib/auth';
import { forbidden } from 'next/navigation';

export async function getPost(id: string) {
  const session = await auth();
  if (!session) forbidden();
  const post = await db.post.findUnique({ where: { id } });
  if (post.userId !== session.userId) forbidden();
  return toPostDto(post);
}
```

**出典**:
- [Next.jsの考え方 / 多層認可](https://zenn.dev/akfm/books/nextjs-basic-principle)

**確信度**: 既存（中） → 高（CVE 事例 + v16 ベストプラクティス）

---

### 5. クライアント JS は「ログイン状態」のみ持ち、トークンを直接持たない

JS から認証状態を判定する場合は、Cookie に含まれる **非機密なフラグ**（例: `logged-in=1`、または短期 JWT のクレーム）で判断し、
**トークン本体や個人情報は持たない**。JS が触れる情報は XSS で漏れる前提で設計する。

**根拠**:
- ログイン状態の表示（ユーザー名・アバター・ナビゲーション）は HttpOnly Cookie だけでは取れない
- 解決策: 非機密な情報を別 Cookie で渡す or Server Component でレンダリングする
- JS が持つ情報は最小化：認証判定 + 表示用の最小データのみ
- Auth0 等は「`auth0.is.authenticated` Cookie」のような非機密判定 Cookie を提供

**実装パターン**:

```tsx
// パターン 1: Server Component でユーザー情報を取得・表示
// app/layout.tsx
import { cookies } from 'next/headers';
import { decodeJwt } from 'jose';

export default async function Layout({ children }: { children: React.ReactNode }) {
  const session = (await cookies()).get('session')?.value;
  const user = session ? await verifySessionAndGetUser(session) : null;

  return (
    <html>
      <body>
        <Header user={user} />  {/* Server Component 経由で渡す */}
        {children}
      </body>
    </html>
  );
}
```

```tsx
// パターン 2: 非機密 Cookie + Client Component
// サーバー側でセット
cookies().set('user-display', JSON.stringify({
  name: user.name,
  avatarUrl: user.avatarUrl,
}), {
  httpOnly: false,    // JS で読む
  secure: true,
  sameSite: 'lax',
  // 注意: パスワード・メール・トークンは絶対に含めない
});

// Client Component
'use client';
function UserMenu() {
  const display = JSON.parse(getCookie('user-display') ?? 'null');
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

// Bad: API でユーザー情報を返して useState で持ち回る
// （セッション中の漏洩リスク。可能なら Server Component で渡す）

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
- [ ] `useState(user)` でフルプロファイルを持ち回っていないか？（必要な属性だけ）
- [ ] 表示用 Cookie に email / phone / 住所等の PII が入っていないか？

**出典**:
- [OWASP: HTML5 Security - Web Storage](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html#local-storage) (OWASP)
- [Auth0: Sessions](https://auth0.com/docs/manage-users/sessions) (Auth0)
- [Vercel: Authentication](https://vercel.com/guides/nextjs-authentication) (Vercel)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-16
