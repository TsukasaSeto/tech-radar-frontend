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
