# エラーハンドリングのベストプラクティス

## ルール

### 1. Next.js の `error.tsx` と `not-found.tsx` でエラー境界を設定する

ルートセグメントに `error.tsx` と `not-found.tsx` を配置し、
エラー発生時のUIをルート単位で制御する。

**根拠**:
- `error.tsx` は ErrorBoundary を自動的にラップし、サーバーコンポーネントのエラーをキャッチする
- ルート単位でエラーを封じ込めることで、一部のエラーがページ全体を壊さない
- `notFound()` 関数と組み合わせて、存在しないリソースへの適切なレスポンスが可能

**コード例**:
```tsx
// app/error.tsx（ルートレベルのエラーキャッチ）
'use client';

import { useEffect } from 'react';

export default function GlobalError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    reportError(error);
  }, [error]);

  return (
    <div role="alert">
      <h2>問題が発生しました</h2>
      <p>しばらく経ってから再試行してください。</p>
      <button onClick={reset}>再試行</button>
    </div>
  );
}

// app/dashboard/error.tsx
'use client';

export default function DashboardError({
  error,
  reset,
}: {
  error: Error;
  reset: () => void;
}) {
  return (
    <div>
      <h2>ダッシュボードの読み込みに失敗しました</h2>
      <button onClick={reset}>再読み込み</button>
    </div>
  );
}

// app/users/[id]/page.tsx
import { notFound } from 'next/navigation';

export default async function UserPage({ params }: { params: { id: string } }) {
  const user = await db.user.findUnique({ where: { id: params.id } });
  if (!user) notFound();
  return <UserProfile user={user} />;
}

// app/users/[id]/not-found.tsx
export default function UserNotFound() {
  return <div>ユーザーが見つかりません</div>;
}
```

**出典**:
- [Next.js Docs: Error Handling](https://nextjs.org/docs/app/building-your-application/routing/error-handling) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. Result 型パターンでエラーを値として扱う

例外のスロー（throw/catch）の代わりに Result 型でエラーを戻り値として表現する。
Server Actions や API 呼び出しで特に有効。

**根拠**:
- 例外は型システムに現れないため、エラーケースの処理が見えなくなる
- Result 型にすることでエラーケースを TypeScript が強制的に処理させられる
- `try/catch` ブロックの乱用を防ぎ、エラー処理の一貫性が保てる

**コード例**:
```ts
// shared/types/result.ts
export type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

export function ok<T>(value: T): Result<T, never> {
  return { ok: true, value };
}

export function err<E>(error: E): Result<never, E> {
  return { ok: false, error };
}

// features/auth/actions/login.ts
'use server';

import { ok, err, Result } from '@/shared/types/result';

type LoginError =
  | { type: 'invalid_credentials' }
  | { type: 'account_locked'; until: Date }
  | { type: 'network_error'; message: string };

export async function login(
  email: string,
  password: string
): Promise<Result<{ userId: string }, LoginError>> {
  try {
    const user = await db.user.findUnique({ where: { email } });

    if (!user || !verifyPassword(password, user.passwordHash)) {
      return err({ type: 'invalid_credentials' });
    }

    if (user.lockedUntil && user.lockedUntil > new Date()) {
      return err({ type: 'account_locked', until: user.lockedUntil });
    }

    return ok({ userId: user.id });
  } catch (e) {
    return err({ type: 'network_error', message: String(e) });
  }
}

// 呼び出し側
const result = await login(email, password);
if (!result.ok) {
  switch (result.error.type) {
    case 'invalid_credentials':
      showError('メールアドレスまたはパスワードが間違っています');
      break;
    case 'account_locked':
      showError(`アカウントは ${result.error.until.toLocaleString()} までロックされています`);
      break;
    case 'network_error':
      showError('通信エラーが発生しました');
      break;
  }
} else {
  redirectToDashboard(result.value.userId);
}
```

**出典**:
- [TypeScript Docs: Discriminated Unions](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#discriminated-unions) (TypeScript公式)

**バージョン**: TypeScript 5+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. エラー境界とローディング状態を Suspense で統合管理する

React の ErrorBoundary と Suspense を組み合わせて、
ローディング・エラー・成功の状態を宣言的に管理する。

**根拠**:
- `loading.tsx` / `error.tsx` が ErrorBoundary と Suspense を自動的に設定する
- ローディング・エラー状態の管理ロジックをコンポーネントから分離できる
- React 19 の `use()` API と組み合わせることでより簡潔に書ける

**コード例**:
```tsx
// app/dashboard/loading.tsx
export default function DashboardLoading() {
  return (
    <div>
      <div className="h-8 w-48 animate-pulse rounded bg-gray-200" />
      <div className="mt-4 grid grid-cols-3 gap-4">
        {Array.from({ length: 3 }).map((_, i) => (
          <div key={i} className="h-32 animate-pulse rounded bg-gray-200" />
        ))}
      </div>
    </div>
  );
}

// app/dashboard/page.tsx
import { Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

export default function DashboardPage() {
  return (
    <div>
      <h1>ダッシュボード</h1>

      <ErrorBoundary fallback={<StatsError />}>
        <Suspense fallback={<StatsSkeleton />}>
          <StatsPanel />
        </Suspense>
      </ErrorBoundary>

      <ErrorBoundary fallback={<FeedError />}>
        <Suspense fallback={<FeedSkeleton />}>
          <ActivityFeed />
        </Suspense>
      </ErrorBoundary>
    </div>
  );
}
```

**出典**:
- [Next.js Docs: Error Handling with Suspense](https://nextjs.org/docs/app/building-your-application/routing/error-handling) (Next.js公式)
- [React Docs: Suspense](https://react.dev/reference/react/Suspense) (React公式)

**バージョン**: Next.js 13+, React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

## 関連プラクティス

- [`api-client/error-handling.md`](../api-client/error-handling.md) - HTTP/API エラーの分類・リトライ・ネットワークエラー処理
- [`observability/error-tracking.md`](../observability/error-tracking.md) - Sentry によるエラー追跡とアラート
