# Next.js Server Actions のベストプラクティス

## ルール

### 1. フォームのミューテーションには Server Actions を使う

`'use server'` で定義した Server Actions でフォーム送信やデータ変更操作を実装する。
Server Action は公開エンドポイントなので、最小構成でも「認証 → 入力検証 → DAL 経由の書き込み」の順に書く（詳細は #5）。

**根拠**:
- クライアント→サーバーの通信を型安全に実装できる
- Next.js が自動的に CSRF 保護を行う
- JavaScript なしでも動作する（プログレッシブエンハンスメント）
- Server Action は UI からだけでなく直接の POST でも呼べるので、Action の中で毎回認証・認可を確認する（公式）

**コード例**:
```tsx
// app/_actions/user.ts
'use server';

import { z } from 'zod';
import { revalidatePath } from 'next/cache';
import { redirect, unauthorized } from 'next/navigation';
import { verifySession } from '@/app/_lib/dal/auth';
import { createUser } from '@/app/_lib/dal/user.mutations'; // 権限の確認と書き込みは DAL で行う

const CreateUserInput = z.object({
  name: z.string().min(1).max(50),
  email: z.email(),
});

export async function createUserAction(formData: FormData): Promise<void> {
  const session = await verifySession();
  if (!session) unauthorized(); // experimental.authInterrupts: true が前提

  // 許可したフィールドだけを取り出す（formData.get() を as string で信用しない）
  const parsed = CreateUserInput.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
  });
  if (!parsed.success) return; // 画面にエラーを返す場合は #2 の useActionState を使う

  await createUser(parsed.data);
  revalidatePath('/users');
  redirect('/users');
}

// app/users/new/page.tsx
import { createUserAction } from '@/app/_actions/user';

export default function NewUserPage() {
  return (
    <form action={createUserAction}>
      <input name="name" type="text" required />
      <input name="email" type="email" required />
      <button type="submit">作成</button>
    </form>
  );
}
```

**出典**:
- [Next.js Docs: Server Actions and Mutations](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations) (Next.js公式)
- [Next.js Docs: Mutating Data](https://nextjs.org/docs/app/getting-started/mutating-data) (Next.js 公式、Server Function は直接の POST で呼べるので毎回認証・認可を確認する) ※2026-10-25 確認
  > "Server Functions are reachable via direct POST requests, not just through your application's UI. Always verify authentication and authorization inside every Server Function."

**バージョン**: Next.js 14+（コード例の `unauthorized()` は 15.1+ で `experimental.authInterrupts: true` が前提）
**確信度**: 高
**最終更新**: 2026-10-25

---

#### 追加根拠 (2026-05-15) — ルール1「フォームのミューテーションには Server Actions を使う」

新たに以下の記事で Server Actions が API Route に対して削減するボイラープレートの具体像が示された:
- [I stopped creating API routes for simple forms in Next.js](https://medium.com/@waqar105lgu.edu/i-stopped-creating-api-routes-for-simple-forms-in-next-js-5dacf5d31ff1) (Medium / 2026-05-15) ※2026-05-15に実際にfetch成功

**出典引用**:
> "Server Actions can remove a surprising amount of complexity from your Next.js apps"
> ([I stopped creating API routes for simple forms in Next.js], セクション "Benefits")

Server Actions が適切なケースと API Route が引き続き必要なケースを明確化する:

| パターン | 推奨アプローチ | 理由 |
|---|---|---|
| フォーム送信・内部データ変更 | **Server Actions** | API Route・`fetch()`・ローディング状態管理が不要になる |
| 公開 API（外部サービス向け） | **API Route** | Server Actions はアプリ内部専用 |
| モバイルアプリのバックエンド | **API Route** | エンドポイントとして外部から呼び出す必要あり |
| サードパーティ統合（webhook 等） | **API Route** | 外部サービスから叩かれる |

**確信度**: 既存（高）→ 高（適用ケースと非適用ケースを明確化）

---

### 2. `useActionState` でフォームの状態とエラーを管理する

Server Actions の結果は `useActionState`（React 19）で管理する。フォームの送信結果が「サーバーからの応答内容だけ」で決まる場合は `useActionState` を使い、複数の分岐や条件付き遷移・サーバー呼び出しを伴わないアクションが混在する複雑な状態機械には `useReducer` を使う。
`useActionState` に渡す Action は、第 1 引数に前回の状態、第 2 引数に `FormData` を受け取り、次の状態を返す形にする（#1 の `(formData) => void` の Action はそのまま渡せない）。

**根拠**:
- `useActionState` は pending・エラー・成功の各状態遷移を自動管理し、サーバーアクションの実行結果だけで次状態が決まるフォームでは `useReducer` の dispatch 処理を手書きする必要がなくなる
- 状態遷移がサーバー応答以外の条件分岐を多く含む場合（複数アクション種別・ローカルのみの状態変更など）は `useReducer` の明示的な reducer 関数の方が見通しが良い
- `useActionState` に渡した関数は、前回の状態（初回は `initialState`）を第 1 引数に、フォームデータを第 2 引数に受け取る（React 公式）

**コード例**:
```tsx
// app/_actions/user.ts — useActionState 用の Action（#1 と同じファイルに置く場合は import と CreateUserInput を共有する）
'use server';

import { z } from 'zod';
import { revalidatePath } from 'next/cache';
import { redirect, unauthorized } from 'next/navigation';
import { verifySession } from '@/app/_lib/dal/auth';
import { createUser } from '@/app/_lib/dal/user.mutations';

const CreateUserInput = z.object({
  name: z.string().min(1).max(50),
  email: z.email(),
});

export type CreateUserState = {
  errors?: { name?: string[]; email?: string[] };
  message?: string;
};

export async function createUserWithState(
  _prevState: CreateUserState,
  formData: FormData,
): Promise<CreateUserState> {
  const session = await verifySession();
  if (!session) unauthorized(); // experimental.authInterrupts: true が前提

  const parsed = CreateUserInput.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
  });
  if (!parsed.success) {
    return { errors: z.flattenError(parsed.error).fieldErrors, message: '入力内容を確認してください' };
  }

  await createUser(parsed.data);
  revalidatePath('/users');
  redirect('/users');
}
```

```tsx
// app/users/new/new-user-form.tsx
'use client';
import { useActionState } from 'react';
import { useFormStatus } from 'react-dom';
import { createUserWithState } from '@/app/_actions/user';
import type { CreateUserState } from '@/app/_actions/user';

const initialState: CreateUserState = {};

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button type="submit" disabled={pending}>{pending ? '送信中...' : '作成'}</button>;
}

export default function NewUserForm() {
  const [state, action] = useActionState(createUserWithState, initialState);
  return (
    <form action={action}>
      <input name="name" />
      {state.errors?.name && <p>{state.errors.name[0]}</p>}
      <input name="email" type="email" />
      {state.errors?.email && <p>{state.errors.email[0]}</p>}
      {state.message && <p>{state.message}</p>}
      <SubmitButton />
    </form>
  );
}
```

**出典引用**:
> "useActionState wins when the form submits to a server action and the next state depends entirely on what the server returns. useReducer wins when the state machine has multiple branches, conditional transitions, or actions that do not involve server calls."
> ([React `useActionState` in 2026: Replacing `useReducer` for Server-Driven Form Logic](https://dev.to/jsmanifest/react-useactionstate-in-2026-replacing-usereducer-for-server-driven-form-logic-4co6), セクション "useActionState vs useReducer") ※2026-09-27に実際にfetch成功

> "When you use `useActionState`, the `reducerAction` receives an extra argument as its first argument: the previous or initial state. The submitted form data is therefore its second argument instead of its first."
> ([React Docs: useActionState](https://react.dev/reference/react/useActionState), セクション "Troubleshooting") ※2026-10-25 確認

**出典**:
- [Next.js Docs: useActionState](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations#server-side-form-validation) (Next.js公式)
- [React Docs: useActionState](https://react.dev/reference/react/useActionState) (React 公式、Action の引数は前回の状態とフォームデータ) ※2026-10-25 確認
- [React `useActionState` in 2026: Replacing `useReducer` for Server-Driven Form Logic](https://dev.to/jsmanifest/react-useactionstate-in-2026-replacing-usereducer-for-server-driven-form-logic-4co6) (dev.to、useActionState と useReducer の使い分け基準) ※2026-09-27に実際にfetch成功

**バージョン**: Next.js 15+, React 19+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 3. Server Actions は `'use server'` ファイルに集約する

Client Component から呼ぶ Action と、複数の画面で使う Action は、専用の `app/_actions/` ディレクトリの `'use server'` ファイルに集約する。
Server Component の中でのインライン定義（関数本体の先頭に `'use server'`）も公式にサポートされており、そのコンポーネントだけで使う小さな Action には使ってよい。どちらの書き方でも公開エンドポイントになるので、認証・認可・入力検証は省かない（#5）。

**根拠**:
- Client Component の中では Server Function を定義できない。Client Component から呼ぶ Action は `'use server'` を先頭に書いたファイルから import する必要がある（公式）
- `'use server'` をファイル先頭に書くと、そのファイルの export がすべて Server Function になる。公開する関数の一覧がファイル単位で見えるので、レビューで認可漏れを探しやすい（export を絞る方法は #6）
- 複数画面で使う Action をコンポーネントに散らすと、同じ処理の認可・検証が画面ごとに食い違いやすい

**コード例**:
```
app/
├── _actions/
│   ├── user.ts     # 'use server' - ユーザー操作
│   ├── post.ts     # 'use server' - 投稿操作
│   └── auth.ts     # 'use server' - 認証アクション
```

```tsx
// インライン定義を使ってよい例: その Server Component だけで使う小さな Action
// app/notes/page.tsx（Server Component）
import { redirect } from 'next/navigation';
import { verifySession } from '@/app/_lib/dal/auth';
import { saveNote } from '@/app/_lib/dal/note.mutations';

export default function NotesPage() {
  async function addNote(formData: FormData) {
    'use server';
    const session = await verifySession(); // インラインでも公開エンドポイントなので認証する
    if (!session) redirect('/login');
    const note = String(formData.get('note') ?? '').trim();
    if (note) await saveNote(session.userId, note);
  }

  return (
    <form action={addNote}>
      <input name="note" />
      <button type="submit">追加</button>
    </form>
  );
}
```

**出典**:
- [Next.js Docs: Server Actions Organization](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations#behavior) (Next.js公式)
- [Next.js Docs: Mutating Data — Creating Server Functions](https://nextjs.org/docs/app/getting-started/mutating-data#creating-server-functions) (Next.js 公式、Server Component 内のインライン定義と、Client Component では定義できない点) ※2026-10-25 確認
  > "It's not possible to define Server Functions in Client Components. However, you can invoke them in Client Components by importing them from a file that has the `"use server"` directive at the top of it."

**バージョン**: Next.js 14+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 4. Server Actions で楽観的 UI 更新を実装する

`useOptimistic` でサーバーの応答を待たずにUIを先行更新して体感速度を向上させる。
`useOptimistic` に渡す確定状態（props、または `useState` の値）と同じデータを、別の state で二重に管理しない。確定状態は、Server Action 内の `revalidatePath` で更新された props で受け取るか、Action の結果を受けて同じ Transition の中で `useState` を更新する。

**根拠**:
- `useOptimistic` の楽観的な値は Action の実行中だけ描画され、Transition が完了すると確定状態と同じレンダーで収束する（React 公式）。確定状態の更新を Transition の外で別途行うと、楽観的な分と確定した分が同時に表示され、二重カウントになる
- 公式の例は、Client の `useState` を確定状態にして、`await` の後に `startTransition` の中で更新する形。Server Components + `revalidatePath` で props を更新する形もとれる。どちらでも、確定状態は 1 つにする
- offline-first（IndexedDB 等）の設計を採用する場合は `useOptimistic` 自体が不要になるケースがある

**コード例**:
```tsx
'use client';
import { useOptimistic, useTransition } from 'react';
import { toggleLike } from '@/app/_actions/post'; // Action の中で revalidatePath し、新しい post が props で届く

type Post = { id: string; liked: boolean; likeCount: number };

export function LikeButton({ post }: { post: Post }) {
  const [optimisticPost, updateOptimistic] = useOptimistic(
    post,
    (state, liked: boolean) => ({
      ...state,
      liked,
      likeCount: liked ? state.likeCount + 1 : state.likeCount - 1,
    })
  );
  const [, startTransition] = useTransition();

  function handleClick() {
    startTransition(async () => {
      updateOptimistic(!optimisticPost.liked);
      await toggleLike(post.id);
    });
  }

  return (
    <button onClick={handleClick}>
      {optimisticPost.liked ? '♥' : '♡'} {optimisticPost.likeCount}
    </button>
  );
}
```

**アンチパターン**:
- `useOptimistic` と `useState` で同一リストを二重管理する（pending と確定の二重カウントが起きる）
- Client Component 内で Server Action の応答後に、Transition の外で `setState` を使って再 append する

**出典引用**:
> "The optimistic and real state converge in the same render when the Transition completes."
> ([React Docs: useOptimistic](https://react.dev/reference/react/useOptimistic)) ※2026-10-25 確認

**出典**:
- [React Docs: useOptimistic](https://react.dev/reference/react/useOptimistic) (React公式、Client の useState を確定状態にする例を含む) ※2026-10-25 確認
- [useOptimistic を導入して、結局外した話](https://zenn.dev/ysaya_dev/articles/useoptimistic2) (Zenn、手動 state 管理との二重カウントの事例) ※2026-05-31 fetch

**バージョン**: Next.js 15+, React 19+
**確信度**: 中（二重カウントの事例は単一記事。確定状態の扱いは React 公式で確認）
**最終更新**: 2026-10-25

---

### 5. Server Actions を公開エンドポイントとして扱い、認証・認可・入力検証を Action 内で必ず行う（RLS は構成に応じた追加の防御）

`'use server'` はコードをサーバーで実行するディレクティブであり、エンドポイントを非公開にしない。保護が必要な Server Action では **① 認証（セッションの確認）・② 入力検証（Zod）・③ 認可（操作対象の所有者・権限の確認。最終判定は DAL）** を、ビジネスロジックより先に行う。ログインしているだけで他人のデータを操作できる状態（IDOR）にしないため、認証だけで済ませず必ず認可まで行う。
ログイン・新規登録のように未認証で呼ぶ Action では認証・認可の段はないが、入力検証は省かない。DB の Row Level Security（RLS）は、使える DB（Supabase / PostgreSQL 等）で、クエリと同じトランザクションにユーザー文脈を正しく設定できる場合に追加する防御として使う。

**根拠**:
- Server Action はビルド時にハッシュ化された POST エンドポイントとしてコンパイルされ、クライアントバンドルに含まれる
- Next.js は CSRF 保護を提供するが、認証（セッション確認）・認可（権限チェック）・入力バリデーションは開発者の責任
- UI で管理者ボタンを非表示にしても、エンドポイント自体はインターネット上の誰もが直接 POST 呼び出しできる
- RLS をデータベース層（Supabase / PostgreSQL）で有効化すると、コードの認可チェックが漏れても DB が行単位でアクセスを制御できる。ただしポリシーが参照するユーザー（テナント）文脈を、クエリと同じトランザクションで設定しないとガードとして成り立たない（後述の Prisma の落とし穴を参照）。RLS に対応していない DB では、DAL の認可と WHERE 句へのユーザー条件の追加で守る
- 入力検証は Action の冒頭、ビジネスロジックより前に行う。大きなペイロードを受ける Action は、型と値域に加えてサイズの上限も検証する。デシリアライズ自体の脆弱性はアプリ側の検証では防げないので、フレームワークの更新で塞ぐ（`middleware.md` #5）
- ロールベースの権限管理（RBAC）では、認証チェックを Server Action の中だけに置くのでは不十分。少なくとも①ルーティング層（未許可ロールをルート単位でブロック）②UI層（許可されない操作ボタン自体を出さない）③Server層（すべての Server Action 内部で `requirePermission()` 等により権限を再検証）の3層で防御する。UI層での非表示はあくまで見た目の制御であり、Server Action は「実体を伴うネットワークエンドポイント」であるためサーバー側の再検証を省略できない
- OWASP の Next.js Security Cheat Sheet も Server Actions を独立して呼び出し可能な POST エンドポイントとして扱い、渡された引数を信頼しないことを公式原則として挙げている
- Server Action はビルド時に ID が振られる公開エンドポイントであり、ID はビルドごとに導出されるため、デプロイをまたいだ古いページからの呼び出しは `Failed to find Server Action` になる。セルフホストでの対策は Rule #8（`NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` をビルド時に固定する）を参照
- **Prisma + PostgreSQL の RLS では、対話型 `$transaction` の中でテナント文脈が外側のトランザクションと別になる落とし穴がある**: 公式の Client extensions レシピは全クエリを `[SET, query]` のバッチトランザクションで包むため、`$transaction` 内ではクエリが外側とは別のトランザクションで走り、原子性が壊れる。トランザクションの入口で `set_config('app.tenant_id', $1, TRUE)`（`SET LOCAL` 相当）を1回実行し、`AsyncLocalStorage` で「適用済み」を印付けして拡張側をパススルーにする。`current_setting('app.tenant_id', true)` は、その接続で一度も設定されていなければ NULL を返す（PostgreSQL 公式）。ただし同じ接続で `set_config(..., TRUE)` を使ったトランザクションが終わった後は、NULL ではなく空文字を返す（PostgreSQL 16.15 / 17.11 で確認。Prisma はプールの接続を使い回すので、この状態は普通に起きる）。`::uuid` にキャストするポリシーは空文字でキャストエラーになり、クエリが失敗する。`NULLIF(current_setting('app.tenant_id', true), '')::uuid` のように空文字を NULL に寄せ、文脈がないときはどちらの場合も 0 行になる（fail closed）ようにする。テーブル所有者にもポリシーを効かせるには `FORCE ROW LEVEL SECURITY` が必要（バッチトランザクションの問題と `FORCE` は Prisma 公式の RLS サンプルでも確認）
- フォームの hidden 項目（`userId` / `role` 等）は改ざん可能。更新対象のユーザーは `getCurrentUser()` のセッションから決め、受け付けるフィールドは Zod で許可したものだけに絞る（AI 生成コードは hidden 項目の ID でそのまま UPDATE し `role` まで受け取る実装になりがち）

**コード例**:
```tsx
// Bad: 認証・バリデーションなし（誰でも呼び出せる）
'use server';
export async function deleteUser(userId: string) {
  await db.users.delete(userId);
}

// Bad: 認証だけで認可がない（ログインしていれば、任意の userId を渡して他人を削除できる＝IDOR）
'use server';
import { unauthorized } from 'next/navigation';
import { verifySession } from '@/app/_lib/dal/auth';

export async function deleteUserAction(userId: string) {
  const session = await verifySession();
  if (!session) unauthorized();
  await db.user.delete({ where: { id: userId } });
}
```

```tsx
// Good: app/_actions/user.ts — ① 認証 → ② 入力検証 → ③ 認可と削除は DAL
'use server';
import { z } from 'zod';
import { revalidatePath } from 'next/cache';
import { unauthorized } from 'next/navigation';
import { verifySession } from '@/app/_lib/dal/auth';
import { deleteUser } from '@/app/_lib/dal/user.mutations';

const DeleteUserInput = z.object({ userId: z.uuid() });

export async function deleteUserAction(raw: unknown): Promise<void> {
  const session = await verifySession();          // ① 認証
  if (!session) unauthorized();                   //   unauthorized() は experimental.authInterrupts: true が前提
  const { userId } = DeleteUserInput.parse(raw);  // ② 入力検証
  await deleteUser(userId);                       // ③ 認可は DAL で行う（下記）
  revalidatePath('/users');
}

// app/_lib/dal/user.mutations.ts — 最終的な認可は DAL で行う
import 'server-only';
import { forbidden, unauthorized } from 'next/navigation';
import { verifySession } from './auth';

export async function deleteUser(userId: string) {
  const session = await verifySession();
  if (!session) unauthorized();
  // 管理者か本人だけが削除できる。ログインしているだけでは削除させない
  if (session.role !== 'admin' && session.userId !== userId) forbidden();
  await db.user.delete({ where: { id: userId } });
}
```

```ts
// app/_lib/dal/document.mutations.ts — RLS を併用する場合（Prisma + PostgreSQL の例）
// ポリシー例: CREATE POLICY owner_only ON documents USING (owner_id = NULLIF(current_setting('app.user_id', true), '')::uuid);
// （使い回した接続では未設定時に空文字が返るので、NULLIF で NULL に寄せて 0 行にする）
import 'server-only';
import { unauthorized } from 'next/navigation';
import { prisma } from '@/lib/prisma';
import { verifySession } from './auth';

export async function deleteDocument(documentId: string) {
  const session = await verifySession();
  if (!session) unauthorized();
  await prisma.$transaction(async (tx) => {
    // クエリと同じトランザクションの入口でユーザー文脈を設定する（SET LOCAL 相当）。設定しないとポリシーが効かない
    await tx.$executeRaw`SELECT set_config('app.user_id', ${session.userId}, TRUE)`;
    await tx.document.delete({ where: { id: documentId } });
  });
}
```

**出典引用**:
> "A Server Action is a public API route wearing a function signature. Anyone on the internet can invoke it."
> (['use server' Doesn't Mean Private](https://medium.com/@raselhasan11/use-server-doesn-t-mean-private-fbffbca20ea3), セクション "The Hidden Security Risk") ※2026-05-14に実際にfetch成功

> "Server Action は公開エンドポイント。requireAuth + Zod + RLS の3段防御を全 Action で"
> ([「内部API Routeを作らない」Next.js 16 — DALとServer Actionsで読み書きを分離する](https://zenn.dev/shippai/articles/d3b146e960e3b2), セクション "Server Actions（src/app/actions/）") ※2026-06-14に実際にfetch成功

**出典**:
- ['use server' Doesn't Mean Private](https://medium.com/@raselhasan11/use-server-doesn-t-mean-private-fbffbca20ea3) (Medium) ※2026-05-14 fetch
- [「内部API Routeを作らない」Next.js 16 — DALとServer Actionsで読み書きを分離する](https://zenn.dev/shippai/articles/d3b146e960e3b2) (Zenn、requireAuth + Zod + RLS 3段防御パターン・DAL との責務分離) ※2026-06-14 fetch
- [Role-Based Access Control (RBAC) in Next.js 16](https://dev.to/muhammadalisma/role-based-access-control-rbac-in-nextjs-16-2026-beginners-guide-3ihh) (dev.to muhammadalisma、ルート/UI/サーバーの3層RBAC・`requirePermission()`パターン) ※2026-07-29に実際にfetch成功

> "Real role-based auth needs checks in at least three layers: Route level, UI level, and Server level"
> ([Role-Based Access Control (RBAC) in Next.js 16](https://dev.to/muhammadalisma/role-based-access-control-rbac-in-nextjs-16-2026-beginners-guide-3ihh), セクション "Why This Is Tricky") ※2026-07-29に実際にfetch成功

> "Validate all arguments as untrusted input"
> ([OWASP Cheat Sheet Series: Next.js Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/commit/057627b3df718979f972bf7491a772f422229a37), セクション "Secure Server Actions as Endpoints") ※2026-08-31に実際にfetch成功

> "'use server' を書いたファイルで export した関数は、Next.js がビルド時に ID を振り、POSTで叩ける状態になります"
> ([Next.jsのServer Actionは関数に見えるが、実体はIDつきの公開エンドポイントだった](https://zenn.dev/matsutake_prgrm/articles/a-server-action-is-a-public-endpoint), セクション "1つの事実") ※2026-09-27に実際にfetch成功

**出典（追加）**:
- [OWASP Cheat Sheet Series: Next.js Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/commit/057627b3df718979f972bf7491a772f422229a37) (OWASP公式、Server Actions を独立エンドポイントとして扱う原則の公式裏付け) ※2026-08-31に実際にfetch成功
- [Next.jsのServer Actionは関数に見えるが、実体はIDつきの公開エンドポイントだった](https://zenn.dev/matsutake_prgrm/articles/a-server-action-is-a-public-endpoint) (Zenn、ビルドごとのID再生成によるstaleクライアント問題) ※2026-09-27に実際にfetch成功
- [Prisma で Postgres の RLS を入れたら、公式レシピ通りでもトランザクションの中で効かなかった話](https://zenn.dev/glaciermelt/articles/prisma-postgres-rls-interactive-transaction) (Zenn glaciermelt、対話型トランザクション内のテナント文脈と `current_setting` の NULL 挙動という RLS 実装上の反例) ※2026-10-03 fetch

> 「つまり **外側の `$transaction` とは別のトランザクションで走る**。テナントの文脈は付きますが、`order.create` と `stock.update` が同じトランザクションにいる保証が無くなり、原子性が壊れます。」
> ([Prisma で Postgres の RLS を入れたら、公式レシピ通りでもトランザクションの中で効かなかった話](https://zenn.dev/glaciermelt/articles/prisma-postgres-rls-interactive-transaction), セクション "罠 1: 対話型トランザクションの中で効かない") ※2026-10-03に実際にfetch成功
- [prisma/prisma-client-extensions: Row Level Security](https://github.com/prisma/prisma-client-extensions/tree/main/row-level-security) (Prisma 公式のサンプル、全クエリを `$transaction([set_config, query])` のバッチで包むので明示的な `$transaction()` は意図どおりに動かないことがある、`set_config(..., TRUE)` で LOCAL に設定する、`FORCE ROW LEVEL SECURITY` で所有者にも効かせる) ※2026-10-25 確認
  > "Because this example extension wraps every query in a new batch transaction, explicitly running transactions with `companyPrisma.$transaction()` may not work as intended."
- [PostgreSQL Docs: System Administration Functions — current_setting / set_config](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADMIN-SET) (PostgreSQL 公式、`missing_ok` が `true` で設定が存在しなければ NULL、`is_local` が `true` なら現在のトランザクションの間だけ有効。設定後の接続で空文字に戻る挙動は記載がなく、16.15 / 17.11 で実測) ※2026-10-25 確認
  > "If there is no such setting, `current_setting` throws an error unless `missing_ok` is supplied and is `true` (in which case NULL is returned)."
- [AIに書かせたNext.jsのコードで起きやすいセキュリティの穴と、直し方](https://zenn.dev/yakankun/articles/nextjs-ai-code-security-pitfalls) (Zenn yakankun、hidden 項目の userId/role を信用せずセッションから対象を決める・許可フィールドのみ更新) ※2026-10-10に実際にfetch成功

> "更新する相手はセッションから取り、受け付ける項目は名前だけにします。"
> ([AIに書かせたNext.jsのコードで起きやすいセキュリティの穴と、直し方](https://zenn.dev/yakankun/articles/nextjs-ai-code-security-pitfalls), セクション "1. Server Action は誰でも直接呼べる。更新する相手はセッションで決める（A02）") ※2026-10-10に実際にfetch成功（引用箇所は 2026-10-25 のマージ時に差し替え）
- [RSC Is Not the Input Boundary](https://dev.to/lazarv/rsc-is-not-the-input-boundary-2aao) (dev.to / lazarv, 2026-05-09、Server Function の入力検証をビジネスロジックより先に行う・サイズ上限を設ける。`middleware.md` #5 から移した観点) ※2026-05-09に実際にfetch成功
- [Next.js Docs: Authentication — Server Actions](https://nextjs.org/docs/app/guides/authentication#server-actions) (Next.js 公式、Server Action を公開 API と同じセキュリティ上の扱いにし、操作の可否を確認する) ※2026-10-25 確認
  > "Treat Server Actions with the same security considerations as public-facing API endpoints, and verify if the user is allowed to perform a mutation."
- [Next.js Docs: unauthorized](https://nextjs.org/docs/app/api-reference/functions/unauthorized) (Next.js 公式、`experimental.authInterrupts` が前提の experimental API) ※2026-10-25 確認

**バージョン**: Next.js 14+（コード例の `unauthorized()` / `forbidden()` は 15.1+ で `experimental.authInterrupts: true` が前提）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 6. `"use server"` ファイルから export する関数を絞り、内部ヘルパーは別モジュールに置く

`"use server"` ディレクティブは「クライアント → サーバ」のバンドル境界の宣言であり、**そのファイルから export した関数はすべて HTTP エンドポイントとして公開**される。export を「公開してよい関数だけ」に絞り、内部ヘルパーは別ファイル（DAL や純粋関数モジュール）に置いて Server Action から呼び出す形にする。v15+ ではアプリ内で参照されていない Server Action は build 時に自動削除されエンドポイントが作られない最適化が入ったが、「使われている Server Action は直接 POST で呼ばれる前提」は変わらない。

**根拠**:
- `"use server"` ファイルから export した関数は、関数名がハッシュ化された POST エンドポイントとして公開される。攻撃者は任意のタイミングで任意の引数で呼び出せる
- 「フォームから呼ばれる」「内部ヘルパーだから安全」という前提は成り立たない。内部用の計算関数を同じファイルから export すると、それも公開エンドポイント化される
- v15+ で「未使用 Server Action（Server Component / Client Component / 別の Server Action のいずれからも import されていないもの）」は build 時に削除されるが、これは未使用関数の救済であって、フォームの `action` 属性に渡されているなど何らかの使用があれば当然エンドポイントが作られる
- 公開を絞ったうえで、関数内で必ず `verifySession()` による認可・`zod` 等によるランタイムバリデーション・DAL 経由のデータアクセスを行うことで多層防御になる

**コード例**:
```typescript
// Good: app/_actions/post.ts
// 公開してよい Server Action だけを export
"use server";

import { z } from "zod";
import { revalidateTag } from "next/cache";
import { unauthorized } from "next/navigation";
import { verifySession } from "@/app/_lib/dal/auth";
import { createPost } from "@/app/_lib/dal/post.mutations"; // 内部処理は DAL に置く

const Schema = z.object({
  title: z.string().min(1).max(120),
  body: z.string().min(1),
});

export async function createPostAction(formData: FormData) {
  // 1. 認可
  const session = await verifySession();
  if (!session) unauthorized();

  // 2. バリデーション
  const parsed = Schema.safeParse({
    title: formData.get("title"),
    body: formData.get("body"),
  });
  if (!parsed.success) {
    return { ok: false as const, error: z.flattenError(parsed.error) };
  }

  // 3. DAL 経由で書き込み
  const post = await createPost(session.userId, parsed.data);

  // 4. キャッシュ無効化（v16: 第 2 引数必須）
  revalidateTag("posts", "max");
  return { ok: true as const, post };
}

// Bad: 内部ヘルパーまで同じ "use server" ファイルから export
"use server";

// この関数も公開エンドポイント化される（攻撃者が直接叩ける）
// （'use server' ファイルの export は async 関数でなければならないので、同期関数の export はビルドエラーになる）
export async function calculateInternalScore(input: unknown) {
  // ...
}

export async function createPostAction(formData: FormData) {
  // ...
}
```

**アンチパターン**:
- `"use server"` ファイルから `getInternalCalculator` のような内部用ヘルパーを export し、攻撃者が直接叩けてしまう
- 「フォームから呼ばれる前提だから認可は不要」と判断する
- 「未使用 Server Action は build 時に削除される」を根拠に、使用中の Server Action の認可を省く
- 認可チェックを `<form>` 側（Client）でだけ行い、Server Action 内では行わない
- 引数の型を TypeScript の型だけで信用し、`zod` 等のランタイムバリデーションをしない

**出典**:
- [Next.jsの考え方 / 7.1 セキュリティの大前提・7.2 公開を絞るためのファイル分割](https://zenn.dev/akfm/books/nextjs-basic-principle)
- [Next.js Docs: Mutating Data — Creating Server Functions](https://nextjs.org/docs/app/getting-started/mutating-data#creating-server-functions) (Next.js 公式、ファイル先頭の `'use server'` はそのファイルのすべての export を Server Function にする。Server Function は直接の POST で呼べる) ※2026-10-25 確認

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**バージョン**: Next.js 16+（未使用 Server Action の自動削除は v15+）
**確信度**: 高（export がすべて公開エンドポイントになる点は公式ドキュメントで確認）
**最終更新**: 2026-10-25

---

### 7. `redirect()` を try/catch 内で呼ばない — NEXT_REDIRECT は再 throw が必要

`redirect()` は即座にリダイレクトを実行する関数ではなく、特殊なエラーをスローする関数。Next.js フレームワーク層がこのエラーをキャッチしてリダイレクトを実行するため、try/catch ブロックで握り潰すとリダイレクトが無効になる。`notFound()` / `unauthorized()` / `forbidden()` も同じ仕組みで動く。
try の外で呼ぶのが基本で、try の中で呼ぶ必要がある場合は、catch の先頭で `unstable_rethrow(error)` を呼んでフレームワークのエラーを投げ直す。

> "redirect() は『その場でリダイレクトを実行する関数』ではなく、『特殊なエラーをスローする関数』"
> ([Next.jsのredirect()をtry/catchの中で呼ぶと動かない理由と対処法3つ](https://zenn.dev/sato_frontend/articles/d9e458e947503d), セクション "redirect()が動かない理由") ※2026-06-16に実際にfetch成功

**根拠**:
- try/catch がこのエラーを捕捉して値を return すると、フレームワーク層にエラーが届かずリダイレクトが発火しない
- Server Actions 内の try/catch でよく発生する落とし穴で、認証チェックの `redirect('/login')` が静かに無視される
- `error.digest` や `error.message` を手で比較して判定すると、`notFound()` / `unauthorized()` など他のフレームワーク制御のエラーを取りこぼす。内部パス（`next/dist/...`）のユーティリティにも依存しない。公式の `unstable_rethrow` は catch の先頭で呼ぶと、フレームワークが処理すべきエラーだけを投げ直す

**コード例**:
```typescript
// Bad: try 内で呼ぶとリダイレクトが無効になる
export async function updateAction(id: string, data: Input) {
  try {
    await requireAuth(); // ← redirect() がここでスローされても catch が握り潰す
    const result = await updateService(id, data);
    return { success: true, data: result };
  } catch (error) {
    return { success: false, error: 'INTERNAL_ERROR' }; // NEXT_REDIRECT も catch される
  }
}

// Good: redirect() は try の外で呼ぶ
export async function updateAction(id: string, data: Input) {
  await requireAuth(); // ← try の外で呼ぶ。NEXT_REDIRECT が Next.js に届く

  try {
    const result = await updateService(id, data);
    return { success: true, data: result };
  } catch (error) {
    return { success: false, error: 'INTERNAL_ERROR' };
  }
}
```

```typescript
// Alternative: try の中で呼ぶ必要がある場合は、catch の先頭で unstable_rethrow を呼ぶ
import { unstable_rethrow } from 'next/navigation';

export async function updateAction(id: string, data: Input) {
  try {
    await requireAuth();
    const result = await updateService(id, data);
    return { success: true, data: result };
  } catch (error) {
    unstable_rethrow(error); // redirect() / notFound() などのフレームワークのエラーだけを投げ直す
    return { success: false, error: 'INTERNAL_ERROR' };
  }
}
```

**アンチパターン**:
- 認証チェック (`requireAuth()`, `notFound()`) を try 内に置き、エラーハンドリングと混在させる
- `error.digest === 'NEXT_REDIRECT'` や `error.message` の比較、内部パスの `isRedirectError()` で手動判定する（`unstable_rethrow` を使う）

**出典**:
- [Next.jsのredirect()をtry/catchの中で呼ぶと動かない理由と対処法3つ](https://zenn.dev/sato_frontend/articles/d9e458e947503d) (Zenn sato_frontend) ※2026-06-16に実際にfetch成功
- [Next.js Docs: unstable_rethrow](https://nextjs.org/docs/app/api-reference/functions/unstable_rethrow) (Next.js 公式、catch の先頭で呼んでフレームワークの内部エラーを投げ直す) ※2026-10-25 確認
  > "This method should be called at the top of the catch block, passing the error object as its only argument."
- [Next.js Docs: unauthorized](https://nextjs.org/docs/app/api-reference/functions/unauthorized) (Next.js 公式、`try/catch` で囲むと割り込みが抑止されるので `unstable_rethrow` を使う) ※2026-10-25 確認

**バージョン**: Next.js 13+ (App Router)（`unstable_rethrow` は名前のとおり unstable API）
**確信度**: 高（公式ドキュメントで確認）
**最終更新**: 2026-10-25

---

### 8. セルフホストで Server Actions を使うなら `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` を「ビルド時に」固定する

Server Action ID はビルドごとに生成される暗号鍵で導出されるため、鍵が固定されていないと「デプロイをまたいだ古いページから送信されたフォーム」や「複数レプリカが別ビルドを配信している状態」で `Failed to find Server Action` が発生する。
`openssl rand -base64 32` で生成した値を `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` としてシークレット管理に置き、**ビルドステージに** 注入する。
ランタイム（runner）側にだけ環境変数を渡しても効果がない。鍵はビルド時に成果物へ焼き込まれるためである。

**根拠**:
- 鍵未固定だとビルドのたびに Server Action ID の導出結果が変わり、クライアントが保持している古い ID がサーバー側で解決できなくなる
- 公式ドキュメントの Version Skew の節は「ナビゲーション時のフルリロード」を扱っており、フォーム送信がデプロイをまたいで失敗するケースの解決策としては読み取りにくい。実際に `Failed to find Server Action` のエラーページ側が `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` を解決策として挙げている
- マルチコンテナ（AWS ECS 等）ではローリング更新中に新旧ビルドが同時に稼働するため、鍵固定が無いと更新中だけ確率的にエラーが出る
- 検証環境は Next.js 16.3 / App Router / AWS ECS + CodeBuild のセルフホスト構成

**コード例**:
```bash
# 鍵の生成（一度だけ。生成後はシークレットマネージャ等で管理する）
openssl rand -base64 32
```
```dockerfile
# Good: builder ステージに ARG/ENV で注入する（ビルド時に焼き込まれる）
FROM node:24-alpine AS builder

ARG NEXT_SERVER_ACTIONS_ENCRYPTION_KEY
ENV NEXT_SERVER_ACTIONS_ENCRYPTION_KEY=${NEXT_SERVER_ACTIONS_ENCRYPTION_KEY}

COPY . .
RUN npm run build

# Bad: runner ステージにだけ環境変数を渡す（ビルド済み成果物には反映されず、エラーは解消しない）
# FROM node:24-alpine AS runner
# ENV NEXT_SERVER_ACTIONS_ENCRYPTION_KEY=${NEXT_SERVER_ACTIONS_ENCRYPTION_KEY}
```

**アンチパターン**:
- 鍵をリポジトリ内の `.env` にコミットする（Server Action ID の偽造につながるため、シークレット管理必須。`security/secret-management.md` 参照）
- ランタイム環境変数として設定して「効かない」と判断し、`deploymentId` などの別機能に流れる
- デプロイのたびに鍵を再生成する CI を組む（固定されないので未設定と同じ）

**出典引用**:
> 「当該エラーの公式ページに deploymentId が載っていない」— 公式のエラーページは `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` を解決策として挙げている
> ([【Next.js】デプロイをまたぐと Server Actions が失敗する問題は `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` で直る](https://qiita.com/bsj-k-hioki/items/03ba0d5f1801065522ad), セクション "当該エラーの公式ページに deploymentId が載っていない") ※2026-08-28に実際にfetch成功

> 「Version Skew の節は「ナビゲーション」の話」
> ([【Next.js】デプロイをまたぐと Server Actions が失敗する問題は `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` で直る](https://qiita.com/bsj-k-hioki/items/03ba0d5f1801065522ad), セクション "Version Skew の節は「ナビゲーション」の話") ※2026-08-28に実際にfetch成功

**出典**:
- [Next.js Docs: Self-Hosting — Server Functions encryption key](https://nextjs.org/docs/app/guides/self-hosting#server-functions-encryption-key) (Next.js 公式、既定ではビルドごとに鍵が生成され、全インスタンスで同じ鍵が必要。鍵はビルド出力に埋め込まれる) ※2026-10-25 確認
  > "The key is embedded in the build output and used automatically at runtime."
- [Next.js Docs: Failed to find Server Action](https://nextjs.org/docs/messages/failed-to-find-server-action) (Next.js 公式のエラーページ、解決策として鍵をビルド時に設定する) ※2026-10-25 確認
  > "Set this variable at build time—the key is embedded in the build output and used automatically at runtime."
- [Next.js Docs: Data Security — Overwriting encryption keys (advanced)](https://nextjs.org/docs/app/guides/data-security#overwriting-encryption-keys-advanced) (Next.js 公式、`openssl rand -base64 32` で作る AES 鍵。Action ID はコンパイル時に作られ、最大 14 日キャッシュされる) ※2026-10-25 確認
- [vercel/next.js `server-action-period-hash-custom-key.test.ts`（v16.4.3）](https://github.com/vercel/next.js/blob/v16.4.3/test/production/app-dir/server-action-period-hash/server-action-period-hash-custom-key.test.ts) (Next.js 公式リポジトリのテスト、`NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` を変えてビルドすると Action ID が変わり、同じ鍵ならビルドキャッシュを消しても同じ ID になる。ビルド設定では鍵が Action ID のハッシュの salt に使われる) ※2026-10-25 確認

**バージョン**: Next.js 14+（16.3 のセルフホスト構成で確認）
**確信度**: 高（鍵がビルド出力に埋め込まれる点は公式ドキュメント、Action ID が鍵から導出される点は公式リポジトリのテストとソースで確認）
**最終更新**: 2026-10-25
