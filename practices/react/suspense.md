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

---

### 4. `browser()` で SSR 不可なコンポーネントを明示し、ハイドレーションミスマッチではなく Suspense フォールバックに落とす

`react-dom` の `browser()` と `use()` を組み合わせると、ブラウザでしかレンダリングできないコンポーネントを明示的に宣言できる。SSR 時は自動的に Suspense のフォールバック UI へ落ち、クライアントでレンダリングが成功してもハイドレーションミスマッチとして扱われない。

**根拠**:
- 目視でフォールバックに見えても実際はハイドレーションミスマッチだった、という混同を防げる
- クライアント専用データ取得（`useBrowserQuery` 等）を hydration 後まで安全に遅延できる
- ルーターやライブラリ側で「pathless SSR モード」のような SSR 除外パターンを組み立てやすくなる

**コード例**:
```tsx
import { browser } from 'react-dom';
import { use } from 'react';

function useBrowserQuery<T>(fetcher: () => Promise<T>) {
  // browser() はブラウザ環境でのみ解決される Promise を返し、
  // SSR 時は use() が Suspense フォールバックへフォールスルーする
  return use(browser(fetcher()));
}
```

**出典引用**:
> "SSR時にSuspenseのフォールバックUIに落ちたところがクライアント側でレンダリング成功したとしてもハイドレーションミスマッチではないので、これにより従来挙動の問題を防ぐことができます。"
> ([React 19.3 browser() APIの使いみち〜FUNSTACK Routerの場合〜](https://zenn.dev/uhyo/articles/react-use-browser-usage), セクション "browser() APIの導入") ※2026-08-21に実際にfetch成功

**出典**:
- [React 19.3 browser() APIの使いみち〜FUNSTACK Routerの場合〜](https://zenn.dev/uhyo/articles/react-use-browser-usage) (Zenn、react-dom の browser() API を用いた実装例) ※2026-08-21 fetch

**バージョン**: React 19.3+ / react-dom
**確信度**: 中（公式ドキュメントでの直接確認ができておらず、単独のコミュニティ記事による報告）
**最終更新**: 2026-08-21
