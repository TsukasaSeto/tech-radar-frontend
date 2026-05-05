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
