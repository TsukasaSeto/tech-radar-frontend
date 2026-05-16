# Next.js Server Actions のベストプラクティス

## ルール

### 1. フォームのミューテーションには Server Actions を使う

`'use server'` で定義した Server Actions でフォーム送信やデータ変更操作を実定化する。

**根拠**:
- クライアント→サーバーの通信を型安全に実装できる
- Next.js が自動的に CSRF 保護を行う
- JavaScript なしでも動作する（プログレッシブエンハンスメント）

**コード例**:
```tsx
// app/actions/user.ts
'use server';

export async function createUser(formData: FormData) {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;
  await db.user.create({ data: { name, email } });
  revalidatePath('/users');
  redirect('/users');
}

// app/users/new/page.tsx
import { createUser } from '@/app/actions/user';

export default function NewUserPage() {
  return (
    <form action={createUser}>
      <input name="name" type="text" required />
      <input name="email" type="email" required />
      <button type="submit">作成</button>
    </form>
  );
}
```

**出典**:
- [Next.js Docs: Server Actions and Mutations](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations) (Next.js公式)

**バージョン**: Next.js 14+
**確信度**: 高
**最終更新**: 2026-05-05

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

Server Actions の結果は `useActionState`（React 19）で管理する。

**コード例**:
```tsx
'use client';
import { useActionState } from 'react';
import { useFormStatus } from 'react-dom';
import { createUser } from '@/app/actions/user';

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button type="submit" disabled={pending}>{pending ? '送信中...' : '作成'}</button>;
}

export default function NewUserForm() {
  const [state, action] = useActionState(createUser, {});
  return (
    <form action={action}>
      <input name="name" />
      {state.errors?.name && <p>{state.errors.name[0]}</p>}
      <input name="email" />
      {state.message && <p>{state.message}</p>}
      <SubmitButton />
    </form>
  );
}
```

**出典**:
- [Next.js Docs: useActionState](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations#server-side-form-validation) (Next.js公式)

**バージョン**: Next.js 14+, React 18/19
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. Server Actions は `'use server'` ファイルに集約する

Server Actions を各コンポーネントファイルにインラインで定義するのではなく、専用の `actions/` ディレクトリに集約する。

**コード例**:
```
app/
├── actions/
│   ├── user.ts     # 'use server' - ユーザー操作
│   ├── post.ts     # 'use server' - 投稿操作
│   └── auth.ts     # 'use server' - 認証アクション
```

**出典**:
- [Next.js Docs: Server Actions Organization](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations#behavior) (Next.js公式)

**バージョン**: Next.js 14+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. Server Actions で楽観的 UI 更新を実装する

`useOptimistic` でサーバーの応答を待たずにUIを先行更新して体感速度を向上させる。

**コード例**:
```tsx
'use client';
import { useOptimistic, useTransition } from 'react';

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

**出典**:
- [React Docs: useOptimistic](https://react.dev/reference/react/useOptimistic) (React公式)

**バージョン**: Next.js 14+, React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 5. Server Actions を公開エンドポイントとして扱い、認証・認可・入力検証を必ず実装する

`'use server'` はコードをサーバーで実行するディレクティブであり、エンドポイントを非公開にしない。すべての Server Action でセッション確認・権限チェック・入力バリデーションを実装する。

**根拠**:
- Server Action はビルド時にハッシュ化された POST エンドポイントとしてコンパイルされ、クライアントバンドルに含まれる
- Next.js は CSRF 保護を提供するが、認証（セッション確認）・認可（権限チェック）・入力バリデーションは開発者の責任
- UI で管理者ボタンを非表示にしても、エンドポイント自体はインターネット上の誰もが直接 POST 呼び出しできる

**コード例**:
```tsx
// Bad: 認証・バリデーションなし（誰でも呼び出せる）
'use server';
export async function deleteUser(userId: string) {
  await db.users.delete(userId);
}

// Good: 認証・認可・バリデーションを必ず実装
'use server';
import { z } from 'zod';
import { getServerSession } from '@/lib/auth';

const DeleteUserInput = z.object({ userId: z.string().uuid() });

export async function deleteUser(raw: unknown) {
  // 1. 認証チェック
  const session = await getServerSession();
  if (!session?.user) throw new Error('Unauthorized');

  // 2. 認可チェック
  if (session.user.role !== 'admin') throw new Error('Forbidden');

  // 3. 入力バリデーション
  const { userId } = DeleteUserInput.parse(raw);

  await db.users.delete(userId);
  revalidatePath('/users');
}
```

**出典引用**:
> "A Server Action is a public API route wearing a function signature. Anyone on the internet can invoke it."
> (['use server' Doesn't Mean Private](https://medium.com/@raselhasan11/use-server-doesn-t-mean-private-fbffbca20ea3), セクション "The Hidden Security Risk") ※2026-05-14に実際にfetch成功

**バージョン**: Next.js 14+
**確信度**: 高
**最終更新**: 2026-05-14

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
    return { ok: false as const, error: parsed.error.flatten() };
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
export function calculateInternalScore(input: unknown) {
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

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**バージョン**: Next.js 16+（未使用 Server Action の自動削除は v15+）
**確信度**: 高（v16 公式相当の知見）
**最終更新**: 2026-05-16
