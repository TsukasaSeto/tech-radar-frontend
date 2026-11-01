# Data Access Layer (DAL)

データアクセスを単一の入出口に集約し、セキュリティ・認可・DTO 変換・キャッシュ管理を統一する設計パターン。Next.js App Router の Server Components / Server Actions 環境で特に重要。

## トピック

- セキュリティと認可の単一出入り口
- DTO 変換と境界の明示
- Request Memoization との連携
- 多層認可（Proxy / page / Server Action / DAL）

## ルール

### 1. データアクセスは DAL 経由に統一し、コンポーネント直叩きの fetch / DB アクセスを禁止する

データの読み書きはすべて `app/_lib/dal/` 配下の関数を経由させ、Server Component / Server Action / Route Handler から直接 `fetch(API)` や ORM の呼び出しを散在させない。
読み取り用 (`post.ts`) と書き込み用 (`post.mutations.ts`) でファイルを分け、ファイル先頭に必ず `import "server-only"` を書く。
ユーザー固有のキャッシュ不可な読み取りは `cache(fn)` で React Request Memoization にラップし、ユーザー横断で再利用できるものは `"use cache"` + `cacheTag` を組み合わせる。

**根拠**:
- 認可漏れの防止: コンポーネント内に fetch を散在させると認可チェックを忘れる箇所が出る。DAL に集約すれば「DAL 関数の冒頭で必ず認可」というルールが機械的に守れる
- キャッシュキーの一貫性: 同じデータを別の場所で別の引数・オプションで取ると、Request Memoization も `"use cache"` もキーが分かれて効かない。取得を DAL の関数にまとめておくと、呼び出し側が意識しなくてもキーが揃う（同じ URL・オプションの `fetch` は DAL がなくてもメモ化されるが、揃え続けるのが難しい）
- リファクタ耐性: API のレスポンス形が変わっても DAL の中だけで吸収できる
- `import "server-only"` により Client Bundle への混入をビルド時に検出できる

**コード例**:
```ts
// Good: app/_lib/dal/post.ts — 読み取り DAL に集約
import "server-only";
import { cache } from "react";
import { cacheLife, cacheTag } from "next/cache";
// unauthorized / forbidden は experimental.authInterrupts の有効化が前提（#2 参照）
import { notFound, unauthorized, forbidden } from "next/navigation";
import { verifySession } from "./auth";

// 公開記事: 未認証でも読める公開データ。認可を省く意図を関数名とコメントで明示する（#2 参照）
// ユーザー横断でキャッシュ可
export async function getPublicPost(slug: string) {
  "use cache";
  cacheLife("hours");
  cacheTag("posts", `post:${slug}`);

  const res = await fetch(`${API}/posts/${slug}`);
  if (res.status === 404) notFound();
  if (!res.ok) throw new Error("failed");
  return toPostDTO(await res.json());
}

// ユーザー固有・キャッシュ不可: cache() でリクエスト単位にメモ化
export const getMyDraft = cache(async (id: string) => {
  const session = await verifySession();
  if (!session) unauthorized();

  const post = await db.post.findUnique({ where: { id } });
  if (!post) notFound();
  if (post.authorId !== session.userId) forbidden();

  return toPostDTO(post);
});
```

```tsx
// Good: page.tsx は DAL 関数を呼ぶだけ
import { getPublicPost } from "@/app/_lib/dal/post";

export default async function Page({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await getPublicPost(slug);
  return <PostView post={post} />;
}

// Bad: コンポーネント内で fetch を直書き
export default async function Page({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const res = await fetch(`${API}/posts/${slug}`); // 認可・DTO 変換・キャッシュキー統制なし
  const post = await res.json();
  return <PostView post={post} />;
}
```

**アンチパターン**:
- Server Component や Server Action から直接 `fetch(API)` / DB アクセスを書く
- DAL ファイルに `import "server-only"` を付け忘れ、Client Bundle に紛れ込ませる
- 同一データの fetch でオプションを微妙に変えて Memoization を無効化する

**出典**:
- [Next.jsの考え方 / Data Access Layer（DAL）— データの単一の出入り口](https://zenn.dev/akfm/books/nextjs-basic-principle)
- [Next.js Docs: How to implement authentication — Creating a Data Access Layer](https://nextjs.org/docs/app/guides/authentication#creating-a-data-access-layer-dal) (Next.js公式、データ取得と認可を DAL に集約する推奨) ※2026-10-25 確認

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**バージョン**: Next.js 16+
**確信度**: 高（Next.js 公式ガイド + 書籍）
**最終更新**: 2026-10-25

---

### 2. DAL を最後の砦として必ず認可する（上位層のガードは補助）

保護が必要なデータを扱う DAL 関数は、最初に `verifySession()` を呼び、失敗したら `unauthorized()` / `forbidden()`（または `redirect()`）で止める。
個別データに対する権限チェック（「この投稿の所有者は本人か」など）も DAL 内で実施する。
page / Server Action でも認証・認可は必ず行うが、それは早期の拒否と UX のための補助で、DAL の認可を代わりに省いてよい理由にはならない（各層の役割は #4）。
未認証で読める公開データ（#1 の `getPublicPost` など）は、認可を省くのが意図であることを関数名とコメントで明示する。
`verifySession()` 自体も `cache()` でラップしておくと、同一レンダー内で何度呼ばれても 1 回に集約される。

`unauthorized()` / `forbidden()` は Next.js 16 でも Experimental で、`next.config.ts` の `experimental.authInterrupts: true` が前提になる。有効にしない場合は、page / Server Action では `redirect('/login')` や `notFound()`、Route Handler では `new Response(null, { status: 401 })` / `403` を返す安定 API で代替する。

```ts
// next.config.ts — unauthorized() / forbidden() を使う場合
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  experimental: {
    authInterrupts: true,
  },
};

export default nextConfig;
```

**根拠**:
- DAL が「最後の砦」になることで、上位層（Proxy / page / Server Action）で認可を書き忘れても、データに辿り着く手前で確実に止まる
- Server Action は `"use server"` で export した関数がすべて HTTP エンドポイントとして公開されるため、フォーム経由前提の認可では不十分。DAL 内認可なら攻撃者の直接 POST も塞げる
- `verifySession()` を `cache()` でラップすることで、page 冒頭ガード + 複数 DAL 呼び出しの全体で実 DB アクセスが 1 回になる（集約されるのはレンダー内だけ。Server Action と Route Handler はレンダーの外なので、呼ぶたびに実行される）
- 認可ロジックが DAL に集約されることで「どこを直すべきか」の認知コストが下がる
- いわゆる IDOR（Insecure Direct Object Reference）は「認証は通っているが認可（本人のリソースか）を忘れる」典型パターン。`fetch → if 文で弾く` ではなく `findOne({ id, userId })` のように**クエリの WHERE 句自体にユーザー条件を含める**と、認可チェックの書き忘れがそもそも成立しにくくなる
- AI コーディングエージェントが生成した認可ロジックは「コンパイルもレビューも通るが所有者チェックが抜けている」形になりやすく、アプリ固有の認可ルールはシグネチャ型スキャナでは検知できない。エンドポイントごとの所有ルールを明文化し、**別テナントの有効なセッションで他テナントのオブジェクトを取りにいくと 404 になる**クロステナントテストで固定する

**コード例**:
```ts
// Good: app/_lib/dal/auth.ts — verifySession を cache() でラップ
import "server-only";
import { cache } from "react";
import { cookies } from "next/headers";

type Session = { userId: string; role: "admin" | "user" };

export const verifySession = cache(async (): Promise<Session | null> => {
  const sessionId = (await cookies()).get("session")?.value;
  if (!sessionId) return null;
  return await lookupSession(sessionId);
});
```

```tsx
// Good: #1 の getMyDraft を、所有者条件をクエリ自体に含める形にしたもの
// 他人の下書きは「存在しない」と同じ扱いになり、所有者チェックの書き忘れが起きにくい
export const getMyDraft = cache(async (id: string) => {
  const session = await verifySession();
  if (!session) unauthorized();                // 認証

  const post = await db.post.findFirst({ where: { id, authorId: session.userId } }); // 個別権限
  if (!post) notFound();

  return toPostDTO(post);
});

// Bad: page.tsx 側だけで認可、DAL ノーチェック
// → Server Action から直接 getDraftWithoutCheck(id) が呼ばれた瞬間に漏洩
export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const session = await verifySession();
  if (!session) unauthorized();
  const { id } = await params;
  const post = await getDraftWithoutCheck(id); // DAL に認可がない
  return <DraftView post={post} />;
}
```

**アンチパターン**:
- 上位層（Proxy / layout / page / Server Action）のチェックがあることを理由に、DAL の認可を省く（各層の役割と layout・Proxy の限界は #4）
- DAL 関数の中で認可チェックを忘れる（一覧取得で全ユーザー分が返るなど）
- 公開データの DAL 関数と保護データの DAL 関数を区別せず、どれが認可を省いているのか読み取れない

**出典**:
- [Next.jsの考え方 / Data Access Layer（DAL）— データの単一の出入り口](https://zenn.dev/akfm/books/nextjs-basic-principle)
- [AIで作ったアプリで一番危ない穴：他人のデータが見えてしまう「認可漏れ」の話](https://zenn.dev/heygeek/articles/ai-authorization-idor-deep-dive) (Zenn、IDOR という呼称とクエリ時点でのユーザー条件フィルタという具体的実装の追加) ※2026-07-27 fetch
- [Why AI Coding Agents Keep Writing Broken Access Control](https://snyk.io/blog/ai-coding-agents-broken-access-control/) (Snyk Blog、エージェント生成コードの BOLA/IDOR とクロステナントテスト) ※2026-10-01 fetch
- [Next.js Docs: How to implement authentication — Authorization](https://nextjs.org/docs/app/guides/authentication#authorization) (Next.js公式、`verifySession()` を DAL に置き `cache()` でラップする形と、`cache()` がレンダー外では重複排除しない注意) ※2026-10-25 確認
- [Next.js Docs: unauthorized](https://nextjs.org/docs/app/api-reference/functions/unauthorized) (Next.js公式、Experimental で `experimental.authInterrupts` が必要) ※2026-10-25 確認

> "if (!order || order.userId !== req.user.id) { return res.status(404).json({ error: "見つかりません" }) }"（"どう直すか" セクション。さらに安全な形として `findOne({ id: req.params.id, userId: req.user.id })` というクエリ時点フィルタを推奨）
> ([AIで作ったアプリで一番危ない穴：他人のデータが見えてしまう「認可漏れ」の話](https://zenn.dev/heygeek/articles/ai-authorization-idor-deep-dive), Zenn, セクション "どう直すか") ※2026-07-27に実際にfetch成功

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

> "When an agent omits an ownership check, the rule it broke belongs to the application rather than to a signature database."
> ([Why AI Coding Agents Keep Writing Broken Access Control](https://snyk.io/blog/ai-coding-agents-broken-access-control/), Snyk Blog, セクション "Why scanners miss these flaws") ※2026-10-01に実際にfetch成功

**バージョン**: Next.js 16+（`unauthorized()` / `forbidden()` は 15.1 で追加、現在も Experimental）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 3. DAL の戻り値は DTO に変換し、ORM / API レスポンスをそのまま返さない

DB のエンティティや外部 API のレスポンスには `password_hash` / `internal_note` / 他ユーザーの情報など、Client に渡してはいけないフィールドが混じる。
DAL の戻り値は必ず DTO（Data Transfer Object）に整形し、公開してよいフィールドだけを残す。
変換関数は `toUserDTO(user)` / `toPostDTO(post)` のように DAL ファイル内に同居させる。

**根拠**:
- 内部表現を Client に漏らさないことが Next.js App Router 環境での基本的なセキュリティ防衛線。Server Component の戻り値は RSC Payload としてクライアントに転送される
- DTO 境界を明示することで、DB スキーマ変更時に Client への影響を DAL 内で吸収できる
- 「DAL は DTO を返す」というルールがあると、Server Component → Client Component に渡す props の型設計が自然に整う
- RSC Payload の肥大化抑制にもなる（不要フィールドを落とす）
- Client Component に渡した props は HTML 内の RSC ペイロード（`self.__next_f.push(...)`）にそのまま載る。静的書き出し環境で、props に含めたスラッシュ無しの `path` 文字列を Google が拾い続けた事例が報告されている（描画側のリンク・canonical を整えても、渡したデータの中身は消えない）。渡すデータの形は読み込み口で一箇所にそろえる
- OWASP の Next.js Security Cheat Sheet も同じ境界を明示的な原則として挙げている（後述の出典引用）

**コード例**:
```ts
// Good: 内部表現 → 公開 DTO に変換
import "server-only";
import { cache } from "react";
// unauthorized は experimental.authInterrupts の有効化が前提（#2 参照）
import { notFound, unauthorized } from "next/navigation";
import { verifySession } from "./auth";

type UserInternal = {
  id: string;
  name: string;
  avatarUrl: string;
  passwordHash: string;
  internalNote: string;
};

type UserDTO = {
  id: string;
  name: string;
  avatarUrl: string;
};

function toUserDTO(user: UserInternal): UserDTO {
  return {
    id: user.id,
    name: user.name,
    avatarUrl: user.avatarUrl,
    // passwordHash / internalNote は返さない
  };
}

export const getUser = cache(async (id: string): Promise<UserDTO> => {
  const session = await verifySession();
  if (!session) unauthorized();
  const user = await db.user.findUnique({ where: { id } });
  if (!user) notFound();
  return toUserDTO(user);
});

// Bad: ORM のエンティティをそのまま返す
export const getUser = cache(async (id: string) => {
  const user = await db.user.findUnique({ where: { id } });
  return user; // passwordHash / internalNote まで Client に届く可能性
});
```

**アンチパターン**:
- DAL 関数が API レスポンスを `await res.json()` のまま返し、Client Component に内部フィールドが流れる
- DTO 変換を「呼び出し側でやる」運用にして、変換漏れが発生する
- DTO 型を `Partial<Internal>` で定義し、フィールドの取捨選択を型レベルで担保しない

**出典引用**:
> "Do not pass ORM records, session objects, or configuration merely because they are available."
> ([OWASP Cheat Sheet Series: Next.js Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/commit/057627b3df718979f972bf7491a772f422229a37), セクション "Control Data Crossing into the Browser") ※2026-08-31に実際にfetch成功

**出典**:
- [Next.js の静的書き出しで、Google がスラッシュ無しの URL を拾い続けた原因は RSC ペイロードの中の path だった](https://qiita.com/hiraokashi/items/4babf257f79a2d55b76f) (Qiita個人記事、props が RSC ペイロードとして HTML に入る点) ※2026-11-01に実際にfetch成功
  > "App Router では、Server Component から Client Component に渡した props が、RSC ペイロードとしてそのまま HTML に入ります"
  > (セクション "結論")
- [Next.jsの考え方 / Data Access Layer（DAL）— データの単一の出入り口](https://zenn.dev/akfm/books/nextjs-basic-principle)
- [OWASP Cheat Sheet Series: Next.js Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/commit/057627b3df718979f972bf7491a772f422229a37) (OWASP公式、DTO による明示的なフィールド境界を「Control Data Crossing into the Browser」として原則化) ※2026-08-31に実際にfetch成功
- [Next.js Docs: How to implement authentication — Using Data Transfer Objects](https://nextjs.org/docs/app/guides/authentication#using-data-transfer-objects-dto) (Next.js公式、必要なフィールドだけを返す DTO の推奨) ※2026-10-25 確認

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**バージョン**: Next.js 16+
**確信度**: 高（Next.js 公式ガイドの DTO 推奨 + OWASP公式チートシート）
**最終更新**: 2026-11-01

---

### 4. 多層認可（Proxy / page / DAL）で defense-in-depth を担保する

認可は単一レイヤーに依存せず、強度の異なる層で守る。Proxy（旧 middleware）は楽観的チェックで「速い拒否」、`page.tsx` 冒頭と Server Action 冒頭は認証・認可のガード、**DAL は最終的な砦**として個別データに対する権限チェックを担う（DAL の認可は #2）。page と Server Action のガードも省かないが、最終判定は DAL に置く。
`layout.tsx` では認可しない。layout はクライアント遷移（Partial Rendering）で再実行されないのでルートを移るたびにセッションが確認されるわけではなく、子セグメントや並列ルートのレンダリングを止めることもできない。
読み取りと書き込みでファイルを分け（`post.ts` / `post.mutations.ts`）、Server Action は「認可 → バリデーション → Mutation DAL 呼び出し → タグ無効化」を直列で繋ぐオーケストレーション層に留める。

**根拠**:
- 2025 年 3 月公表の **CVE-2025-29927**（CVSS 9.1）で、`x-middleware-subrequest` ヘッダによる middleware 認証バイパスが現実に発生した。フレームワーク側脆弱性が出てもアプリ側で止められる構造が必要
- 公式ガイドは layout でのチェックに注意を促し、「layout はナビゲーションで再レンダリングされない」「layout は残りのルートがレンダリングされるかを制御しない（子セグメントや並列ルートのスロットはルーターがレンダリングし、RSC Payload にも載る）」として、データソースの近く（DAL）でチェックするよう勧めている。layout 単独の認可では子ルートを守れないため、このリポジトリでは layout で認可しない
- 各層が独立して効くため、1 層が抜けても他層で止まる（defense-in-depth）
- 書き込み用 Mutation DAL を分離すると、Server Action のテストは「認可・バリデーション・タグ無効化の順序」だけに集中でき、書き込みロジックは Mutation DAL の単体テストでカバーできる

**コード例**:
```
// 認可レイヤー
| 層                 | 役割                                         | 強度                   |
|--------------------|----------------------------------------------|------------------------|
| Proxy              | クッキー存在確認、未ログインなら /login へ   | 弱（バイパス可能）     |
| layout.tsx         | 認可しない（共有 UI と Provider だけ）       | -                      |
| page.tsx 冒頭      | await verifySession() でガード               | 中                     |
| Server Action 冒頭 | 認証と操作権限の確認（公開エンドポイント扱い）| 中                     |
| DAL                | 個別データの権限チェック                     | 強（最後の砦）         |
```

```ts
// Good: proxy.ts は楽観的チェックのみ
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function proxy(request: NextRequest) {
  const hasSession = request.cookies.has("session");
  if (!hasSession && request.nextUrl.pathname.startsWith("/dashboard")) {
    return NextResponse.redirect(new URL("/login", request.url));
  }
  return NextResponse.next();
}
```

```ts
// Good: Server Action は薄く、Mutation DAL を呼ぶ
// app/_actions/post.ts
"use server";

import { z } from "zod";
import { revalidateTag } from "next/cache";
// unauthorized / forbidden は experimental.authInterrupts の有効化が前提（#2 参照）
import { unauthorized } from "next/navigation";
import { verifySession } from "@/app/_lib/dal/auth";
import { createPost } from "@/app/_lib/dal/post.mutations";

const Schema = z.object({ title: z.string().min(1).max(120), body: z.string().min(1) });

export async function createPostAction(formData: FormData) {
  const session = await verifySession();             // 1. 認可
  if (!session) unauthorized();

  const parsed = Schema.safeParse({                  // 2. バリデーション
    title: formData.get("title"),
    body: formData.get("body"),
  });
  if (!parsed.success) return { ok: false as const, error: parsed.error.flatten() };

  const post = await createPost(session.userId, parsed.data); // 3. Mutation DAL（戻り値は DTO）
  revalidateTag("posts", "max");                              // 4. キャッシュ無効化
  return { ok: true as const, post };
}

// app/_lib/dal/post.mutations.ts — Mutation DAL も認可を持ち、戻り値を DTO にする（#3）
import "server-only";
import { forbidden, unauthorized } from "next/navigation";
import { verifySession } from "./auth";
import { toPostDTO } from "./post";

export async function createPost(authorId: string, input: { title: string; body: string }) {
  const session = await verifySession();
  if (!session) unauthorized();
  if (session.userId !== authorId) forbidden();
  const post = await db.post.create({ data: { ...input, authorId } });
  return toPostDTO(post); // ORM のレコードをそのまま Server Action → クライアントへ流さない
}
```

```tsx
// Bad: layout.tsx で認可を完結させる
export default async function DashboardLayout({ children }: { children: React.ReactNode }) {
  const session = await verifySession();
  if (!session) unauthorized(); // クライアント遷移では再実行されず、子セグメントのレンダリングも止められない
  return <>{children}</>;
}
```

**アンチパターン**:
- Proxy（旧 middleware）単独で「このユーザーは管理者だから全アクセス許可」のような最終判断をする（CVE-2025-29927 のようにフレームワーク側の脆弱性で全てバイパスされうる）
- `layout.tsx` の冒頭で `verifySession()` を呼んで「これで全部守れた」と思う
- UI 側の表示制御（ボタンを隠すなど）だけで守ったつもりになり、Server Action で認可しない
- Server Action 内に書き込みロジックを直書きし、Mutation DAL を作らない（テスタビリティが下がる、認可の二重化もできない）
- セッション Cookie に `httpOnly` / `secure` / `sameSite` を指定し忘れる（XSS / CSRF の温床）

**出典引用**:
> "Do not rely on Proxy or middleware alone"
> ([OWASP Cheat Sheet Series: Next.js Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/commit/057627b3df718979f972bf7491a772f422229a37), セクション "Classify and Protect Server Entry Points") ※2026-08-31に実際にfetch成功

**出典**:
- [Next.jsの考え方 / Data Access Layer（DAL）— データの単一の出入り口](https://zenn.dev/akfm/books/nextjs-basic-principle)
- [CVE-2025-29927: Authorization Bypass in Next.js Middleware](https://nvd.nist.gov/vuln/detail/CVE-2025-29927) (NVD)
- [Next.js Docs: How to implement authentication — Layouts and auth checks](https://nextjs.org/docs/app/guides/authentication#layouts-and-auth-checks) (Next.js公式、layout でのチェックへの注意・Proxy は楽観的チェックに限る・Server Action は公開エンドポイントとして扱う) ※2026-10-25 確認
- [OWASP Cheat Sheet Series: Next.js Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/commit/057627b3df718979f972bf7491a772f422229a37) (OWASP公式、「Proxy/middlewareのみへの依存を避ける」多層認可原則の公式裏付け) ※2026-08-31に実際にfetch成功

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**バージョン**: Next.js 16+
**確信度**: 高（Next.js 公式ガイド + 実在 CVE + OWASP公式チートシート）
**最終更新**: 2026-10-25

---
