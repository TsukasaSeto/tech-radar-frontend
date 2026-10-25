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

`react-dom` の `browser()` の戻り値を `use()` に渡す（`use(browser('理由'))`）と、ブラウザでしかレンダリングできないコンポーネントを明示的に宣言できる。サーバーではそのコンポーネントのレンダーを止めて最も近い `<Suspense>` のフォールバックを HTML に残し、ブラウザでは `use(browser())` が `undefined` を返して通常どおりレンダリングする。クライアントでレンダリングが成功しても、ハイドレーションミスマッチとしては扱われない。
`browser()` が受け取るのは SSR を止める理由（文字列か、理由を返す関数）で、Promise ではない。データ取得そのものは、キャッシュされた Promise を扱う Suspense 対応ライブラリに任せる。

**根拠**:
- 目視でフォールバックに見えても実際はハイドレーションミスマッチだった、という混同を防げる
- `use` は条件分岐の中でも呼べるので、「初期データがないときだけ SSR を止め、取得はクライアントのクエリライブラリに任せる」という条件付きの使い方ができる
- ルーターやライブラリ側で「pathless SSR モード」のような SSR 除外パターンを組み立てやすくなる
- 公式の制約: サーバーレンダリング時は `<Suspense>` の内側で呼ぶ（境界がないとサーバーレンダーが失敗する）。Client Component から呼ぶ。`browser()` を呼ぶだけでは何も起きず、`use` に渡す必要がある（throw もしない）

**コード例**:
```tsx
// Good: localStorage に依存するコンポーネントを SSR から外す
'use client';
import { Suspense, use, useState } from 'react';
import { browser } from 'react-dom';

function SavedDraft() {
  use(browser('下書きは localStorage に保存している'));
  const [draft] = useState(() => localStorage.getItem('draft') ?? '');
  return <textarea defaultValue={draft} />;
}

export function DraftSection() {
  return (
    <Suspense fallback={<p>下書きを読み込み中...</p>}>
      <SavedDraft />
    </Suspense>
  );
}
```

```tsx
// Good: 初期データがないときだけ SSR を止め、取得は Suspense 対応ライブラリに任せる
'use client';
import { use } from 'react';
import { browser } from 'react-dom';
import { useSuspenseQuery } from '@tanstack/react-query';

export function useBrowserQuery<T>(
  queryKey: readonly unknown[],
  queryFn: () => Promise<T>,
  initialData?: T,
) {
  if (initialData === undefined) {
    use(browser('useBrowserQuery: 初期データがない'));
  }
  return useSuspenseQuery({ queryKey, queryFn, initialData }).data;
}

// Bad: browser() に Promise を渡す
// browser() は理由（文字列 / 関数）しか受け取らず、ブラウザでは use(browser(...)) が undefined を返すので、
// このフックはクライアントで何も言わずに undefined を返す。fetcher() も SSR を含め毎レンダー実行され、キャッシュされない Promise を作る
function useBrowserQueryBad<T>(fetcher: () => Promise<T>) {
  return use(browser(fetcher()));
}
```

**出典引用**:
> "SSR時にSuspenseのフォールバックUIに落ちたところがクライアント側でレンダリング成功したとしてもハイドレーションミスマッチではないので、これにより従来挙動の問題を防ぐことができます。"
> ([React 19.3 browser() APIの使いみち〜FUNSTACK Routerの場合〜](https://zenn.dev/uhyo/articles/react-use-browser-usage), セクション "browser() APIの導入") ※2026-08-21に実際にfetch成功

**出典**:
- [React Docs: browser](https://react.dev/reference/react-dom/browser) (React公式、引数は理由の文字列か関数・サーバーとブラウザでの戻り値・Suspense 境界が必須などの制約・`useBrowserQuery` の公式パターン) ※2026-10-25 確認
- [React 19.3](https://react.dev/blog/2026/09/09/react-19-3) (React公式ブログ、19.3 で `browser()` が追加された) ※2026-10-25 確認
- [React 19.3 browser() APIの使いみち〜FUNSTACK Routerの場合〜](https://zenn.dev/uhyo/articles/react-use-browser-usage) (Zenn、react-dom の browser() API を用いた実装例) ※2026-08-21 fetch

**バージョン**: React 19.3+ / react-dom
**確信度**: 高
**最終更新**: 2026-10-25
