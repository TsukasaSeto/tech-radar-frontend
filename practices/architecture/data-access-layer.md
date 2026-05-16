# Data Access Layer (DAL)

データアクセスを単一の入出口に集約し、セキュリティ・認可・DTO 変換・キャッシュ管理を統一する設計パターン。Next.js App Router の Server Components / Server Actions 環境で特に重要。

## トピック

- セキュリティと認可の単一出入り口
- DTO 変換と境界の明示
- Request Memoization との連携
- 多層認可（Proxy / page / DAL）

## ルール

### 1. データアクセスは DAL 経由に統一し、コンポーネント直叩きの fetch / DB アクセスを禁止する

データの読み書きはすべて `app/_lib/dal/` 配下の関数を経由させ、Server Component / Server Action / Route Handler から直接 `fetch(API)` や ORM の呼び出しを散在させない。
読み取り用 (`post.ts`) と書き込み用 (`post.mutations.ts`) でファイルを分け、ファイル先頭に必ず `import "server-only"` を書く。
ユーザー固有のキャッシュ不可な読み取りは `cache(fn)` で React Request Memoization にラップし、ユーザー横断で再利用できるものは `"use cache"` + `cacheTag` を組み合わせる。

**根拠**:
- 認可漏れの防止: コンポーネント内に fetch を散在させると認可チェックを忘れる箇所が出る。DAL に集約すれば「DAL 関数の冒頭で必ず認可」というルールが機械的に守れる
- キャッシュキーの一貫性: 同じデータを別の場所で別の引数で取ると Request Memoization も `"use cache"` も効かない。DAL の関数として共通化して初めてメモ化が機能する
- リファクタ耐性: API のレスポンス形が変わっても DAL の中だけで吸収できる
- `import "server-only"` により Client Bundle への混入をビルド時に検出できる

**コード例**:
```ts
// Good: app/_lib/dal/post.ts — 読み取り DAL に集約
import "server-only";
import { cache } from "react";
import { cacheLife, cacheTag } from "next/cache";
import { notFound, unauthorized, forbidden } from "next/navigation";
import { verifySession } from "./auth";

// 公開記事: ユーザー横断でキャッシュ可
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

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**バージョン**: Next.js 16+
**確信度**: 高（v16 公式相当の知見）
**最終更新**: 2026-05-16

---

### 2. DAL 関数の冒頭で必ず認可チェックを行い、page / component / Server Action 層では認可ロジックを書かない

`verifySession()` を DAL 関数の最初に呼び、失敗したら `unauthorized()` / `forbidden()` を throw する。
個別データに対する権限チェック（「この投稿の所有者は本人か」など）も DAL 内で実施する。
`verifySession()` 自体も `cache()` でラップしておくと、同一リクエスト内で何度呼ばれても 1 回に集約される。

**根拠**:
- DAL が「最後の砦」になることで、上位層（Proxy / page / Server Action）で認可を書き忘れても、データに辿り着く手前で確実に止まる
- Server Action は `"use server"` で export した関数がすべて HTTP エンドポイントとして公開されるため、フォーム経由前提の認可では不十分。DAL 内認可なら攻撃者の直接 POST も塞げる
- `verifySession()` を `cache()` でラップすることで、page 冒頭ガード + 複数 DAL 呼び出しの全体で実 DB アクセスが 1 回になる
- 認可ロジックが DAL に集約されることで「どこを直すべきか」の認知コストが下がる

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

```ts
// Good: DAL 内で認可 → 個別権限チェック
export const getMyDraft = cache(async (id: string) => {
  const session = await verifySession();
  if (!session) unauthorized();                // 認証

  const post = await db.post.findUnique({ where: { id } });
  if (!post) notFound();
  if (post.authorId !== session.userId) forbidden(); // 個別権限

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
- `layout.tsx` で `verifySession()` を呼んで「これで全部守れた」と思う（並行レンダリングで子ページのデータフェッチが先行する可能性があり、漏洩リスク）
- DAL 関数の中で認可チェックを忘れる（一覧取得で全ユーザー分が返るなど）
- Proxy（旧 middleware）単独で認可する（CVE-2025-29927 のようにフレームワーク側脆弱性で全てバイパスされうる）

**出典**:
- [Next.jsの考え方 / Data Access Layer（DAL）— データの単一の出入り口](https://zenn.dev/akfm/books/nextjs-basic-principle)

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**バージョン**: Next.js 16+
**確信度**: 高（v16 公式相当の知見）
**最終更新**: 2026-05-16

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

**コード例**:
```ts
// Good: 内部表現 → 公開 DTO に変換
import "server-only";

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

**出典**:
- [Next.jsの考え方 / Data Access Layer（DAL）— データの単一の出入り口](https://zenn.dev/akfm/books/nextjs-basic-principle)

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**バージョン**: Next.js 16+
**確信度**: 高（v16 公式相当の知見）
**最終更新**: 2026-05-16

---

### 4. 多層認可（Proxy / page / DAL）で defense-in-depth を担保する

認可は単一レイヤーに依存せず、強度の異なる 3 層で守る。Proxy（旧 middleware）は楽観的チェックで「速い拒否」、`page.tsx` 冒頭は粗いガード、**DAL は最終的な砦**として個別データに対する権限チェックを担う。
`layout.tsx` での認可チェックは並行レンダリングのため漏洩リスクがあり禁止する。
読み取りと書き込みでファイルを分け（`post.ts` / `post.mutations.ts`）、Server Action は「認可 → バリデーション → Mutation DAL 呼び出し → タグ無効化」を直列で繋ぐオーケストレーション層に留める。

**根拠**:
- 2025 年 3 月公表の **CVE-2025-29927**（CVSS 9.1）で、`x-middleware-subrequest` ヘッダによる middleware 認証バイパスが現実に発生した。フレームワーク側脆弱性が出てもアプリ側で止められる構造が必要
- レイアウトでの認可は、並行レンダリングのため認可が走る前に子ページのデータフェッチが始まる可能性があり、漏洩リスクがある（公式ガイダンスでも禁止）
- 各層が独立して効くため、1 層が抜けても他層で止まる（defense-in-depth）
- 書き込み用 Mutation DAL を分離すると、Server Action のテストは「認可・バリデーション・タグ無効化の順序」だけに集中でき、書き込みロジックは Mutation DAL の単体テストでカバーできる

**コード例**:
```
// 認可レイヤー
| 層             | 役割                                       | 強度                   |
|----------------|--------------------------------------------|------------------------|
| Proxy          | クッキー存在確認、未ログインなら /login へ | 弱（バイパス可能）     |
| page.tsx 冒頭  | await verifySession() で粗くガード         | 中                     |
| DAL            | 個別データの権限チェック                   | 強（最後の砦）         |
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

  const post = await createPost(session.userId, parsed.data); // 3. Mutation DAL
  revalidateTag("posts", "max");                              // 4. キャッシュ無効化
  return { ok: true as const, post };
}

// app/_lib/dal/post.mutations.ts — Mutation DAL も認可を持つ
import "server-only";
import { forbidden } from "next/navigation";
import { verifySession } from "./auth";

export async function createPost(authorId: string, input: { title: string; body: string }) {
  const session = await verifySession();
  if (!session || session.userId !== authorId) forbidden();
  return await db.post.create({ data: { ...input, authorId } });
}
```

```ts
// Bad: layout.tsx で認可を完結させる
export default async function DashboardLayout({ children }: { children: React.ReactNode }) {
  const session = await verifySession();
  if (!session) unauthorized(); // 並行レンダリングで子ページのフェッチが先行しうる
  return <>{children}</>;
}
```

**アンチパターン**:
- Proxy 単独で「このユーザーは管理者だから全アクセス許可」のような最終判断をする
- `layout.tsx` の冒頭で `verifySession()` を呼んで「これで全部守れた」と思う
- Server Action 内に書き込みロジックを直書きし、Mutation DAL を作らない（テスタビリティが下がる、認可の二重化もできない）
- セッション Cookie に `httpOnly` / `secure` / `sameSite` を指定し忘れる（XSS / CSRF の温床）

**出典**:
- [Next.jsの考え方 / Data Access Layer（DAL）— データの単一の出入り口](https://zenn.dev/akfm/books/nextjs-basic-principle)
- [CVE-2025-29927: Authorization Bypass in Next.js Middleware](https://nvd.nist.gov/vuln/detail/CVE-2025-29927) (NVD)

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**バージョン**: Next.js 16+
**確信度**: 高（v16 公式相当の知見 + 実在 CVE）
**最終更新**: 2026-05-16

---
