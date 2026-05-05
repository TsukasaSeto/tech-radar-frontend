# Suspense / use API のベストプラクティス

## ルール

### 1. Suspense でローディング状態を宣言的に表現する

`Suspense` の `fallback` でローディング状態を宣言的に表現する。

**根拠**:
- ローディング UI の関心をコンポーネントから分離できる
- Server Components との統合がスムーズ

**コード例**:
```tsx
// Good: Suspense で宣言的に
function UserProfile({ userId }: { userId: string }) {
  const { data: user } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });
  return <Profile user={user} />;
}

<Suspense fallback={<Spinner />}>
  <UserProfile userId={userId} />
</Suspense>
```

**出典**:
- [React Docs: Suspense](https://react.dev/reference/react/Suspense) (React公式)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. ErrorBoundary と Suspense を組み合わせてエラー処理する

`Suspense` はローディング状態を処理し、`ErrorBoundary` はエラーを処理する。

**根拠**:
- 両方を組み合わせることで、ローディング・成功・エラーをカバーできる
- Next.js の `error.tsx` は Error Boundary として機能する

**コード例**:
```tsx
import { ErrorBoundary } from 'react-error-boundary';

<ErrorBoundary FallbackComponent={ErrorFallback}>
  <Suspense fallback={<Spinner />}>
    <UserProfile userId={userId} />
  </Suspense>
</ErrorBoundary>
```

**出典**:
- [React Docs: Error Boundaries](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary) (React公式)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. `use` API で Promise と Context を統一的に扱う（React 19+）

React 19 の `use` API で `Promise` と `Context` を同じインターフェースで扱える。

**根拠**:
- `use(promise)` で Server Components から渡された Promise を Client Component で解決できる
- `use(Context)` は条件分岐内にも書ける

**コード例**:
```tsx
// Server Component
async function UserPage({ userId }: { userId: string }) {
  const userPromise = fetchUser(userId);
  return (
    <Suspense fallback={<Skeleton />}>
      <UserDetails userPromise={userPromise} />
    </Suspense>
  );
}

// Client Component
'use client';
import { use } from 'react';

function UserDetails({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);
  return <div>{user.name}</div>;
}
```

**出典**:
- [React Docs: use](https://react.dev/reference/react/use) (React公式)

**バージョン**: React 19+
**確信度**: 高
**最終更新**: 2026-05-05
