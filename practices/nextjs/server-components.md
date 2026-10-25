# Server Components のベストプラクティス

## ルール

### 1. デフォルトは Server Component、必要な場合のみ `'use client'` を付ける

Next.js App Router では全コンポーネントがデフォルトで Server Component になる。
インタラクティビティ（イベントハンドラー・useState・useEffect）が必要な場合のみ
ファイル先頭に `'use client'` を付ける。

**根拠**:
- Server Components はバンドルサイズに含まれない（JS がゼロ）
- サーバーでデータを取得できる（認証情報をクライアントに渡さない）。取得はコンポーネントから DB を直接呼ばず、DAL 経由にする（`../architecture/data-access-layer.md` #1）
- クライアントに送信するデータ量を減らせる

**コード例**:
```tsx
// Server Component（デフォルト）: 'use client' なし
// app/users/page.tsx
import { getUsers } from '@/app/_lib/dal/user'; // 認可と DTO 変換は DAL の中で行う

export default async function UsersPage() {
  const users = await getUsers();
  return (
    <ul>
      {users.map(user => (
        <UserCard key={user.id} user={user} />
      ))}
    </ul>
  );
}

// Client Component: インタラクティビティが必要
// components/SearchInput.tsx
'use client';
import { useState } from 'react';

export function SearchInput({ onSearch }: { onSearch: (q: string) => void }) {
  const [query, setQuery] = useState('');
  return (
    <input
      value={query}
      onChange={e => setQuery(e.target.value)}
      onKeyDown={e => e.key === 'Enter' && onSearch(query)}
    />
  );
}
```

**出典**:
- [Next.js Docs: Server Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components) (Next.js公式)

**バージョン**: Next.js 13+, React 18+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 2. `'use client'` 境界はリーフコンポーネントの近くに置く

`'use client'` を付けると、そのモジュールと、そこから import されるモジュール（推移的な依存）がすべてクライアントコードになる。境界はレンダーツリーではなくモジュールの依存関係で決まる。できるだけツリーの末端（リーフ）のモジュールに置く。
`children` や props として渡した Server Component は、Client Component の中に表示されてもサーバーで実行されたままになる（#4）。

**根拠**:
- `'use client'` のモジュールが import するコンポーネントやユーティリティは、`'use client'` がなくてもクライアントで評価され、クライアントバンドルに含まれる。上位のモジュールに置くほど、不必要に多くのコードがバンドルに入る
- 親子の描画関係は実行環境を決めない。Client Component が `children` として受け取った Server Component は、Client Component のモジュールから import されていないのでサーバーのまま（React 公式）
- Server Component の恩恵（DBアクセス、バンドルサイズ削減）を最大化できる

**コード例**:
```tsx
// Good: Client Component を末端に限定
// app/page.tsx (Server Component)
import { Counter } from '@/components/Counter';

export default async function Page() {
  const data = await fetchHeavyData();
  return (
    <div>
      <StaticContent data={data} />
      <Counter />  {/* Client Component はリーフのみ */}
    </div>
  );
}

// components/Counter.tsx
'use client';
import { useState } from 'react';

export function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// Bad: ページのモジュールに 'use client' を付ける
// → このページが import する StaticContent なども含めてすべてクライアントコードになる
```

**出典**:
- [Next.js Docs: Moving Client Components Down the Tree](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns#moving-client-components-down-the-tree) (Next.js公式)
- [React Docs: 'use client'](https://react.dev/reference/rsc/use-client) (React 公式、境界はモジュールの依存ツリーで決まり、children で渡したコンポーネントはサーバーのまま) ※2026-10-25 確認
  > "Recall that `'use client'` defines the boundary between server and client code on the _module dependency tree_, not the render tree."

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 3. Server Components から Client Components に props を渡す際はシリアライズ可能な値のみ

Server Components から Client Components への props は、React がシリアライズできる値に限る（JSON より広く、`Date` / `Map` / `Set` / `BigInt` / `Promise` なども渡せる）。関数は Server Actions 経由か Client Component 内で定義する。

**根拠**:
- Server と Client の境界を越えるデータは React によってシリアライズされる
- シリアライズできない値を渡すとエラーになる
- **よくある誤解**: `Event handlers cannot be passed to Client Component props` エラーが出た際、受け取り側の Client Component に既に `'use client'` を付けているのに直らないケースがある。原因はコンポーネントではなく、渡している**関数がどこで生成されたか**。Server Component の中で定義したインライン関数（`onClick={() => ...}`）はその時点で「サーバー側の値」であり、`'use client'` は境界を越えた後の話でしかない。「文字に書き起こせる値だけが渡せる」と覚え、関数は Client Component 側で定義するか Server Actions（`'use server'`）にする

**コード例**:
```tsx
// Good: Server Actions を使う（インライン定義でも公開エンドポイントなので、認証・検証は省かない。server-actions.md #5）
import { redirect } from 'next/navigation';
import { verifySession } from '@/app/_lib/dal/auth';
import { saveNote } from '@/app/_lib/dal/note.mutations';

export default async function Page() {
  async function handleAction(formData: FormData) {
    'use server';
    const session = await verifySession();
    if (!session) redirect('/login');
    const note = String(formData.get('note') ?? '').trim();
    if (note) await saveNote(session.userId, note);
  }
  return (
    <form action={handleAction}>
      <input name="note" />
      <button>Submit</button>
    </form>
  );
}
```

**境界を越えられる値・越えられない値（React 公式の一覧）**:

Server Component から Client Component に渡す props の対応型は、React 公式ドキュメントの一覧で決まる。
対応していない値はエラーになるので、props 設計時にチェックする。

| 値 | 境界を越えられるか | 対処 |
|---|---|---|
| プリミティブ（string / number / bigint / boolean / undefined / null） | 越えられる | — |
| `Date` | 越えられる（`Date` のまま届く） | — |
| `Map` / `Set` / Array / TypedArray / ArrayBuffer | 越えられる（中身も対応型であること） | — |
| プレーンオブジェクト（オブジェクトリテラルで作ったもの） | 越えられる（プロパティも対応型であること） | — |
| `Promise` | 越えられる | Client 側で `use()` で解決する |
| Server Function（`'use server'`） | 越えられる | — |
| JSX（Server / Client Component の要素） | 越えられる | — |
| `Symbol` | `Symbol.for` でグローバル登録したものだけ越えられる。`Symbol('x')` はエラー | 文字列・列挙型に置き換え |
| 関数（Server Function でも Client モジュールの export でもないもの） | エラー | Server Action（`'use server'`）として渡す or Client 側で定義 |
| クラス、組み込み以外のクラスのインスタンス（Prisma の `Decimal` など）、null プロトタイプのオブジェクト | エラー | DTO（プレーンオブジェクト）に変換する |
| `RegExp` / `Error` など上記の一覧にない組み込みのインスタンス | 対応型に含まれない | 文字列で送る / Result 型でエラーを構造化（`architecture/error-handling.md` Rule 2 参照） |

**検出方法**:
1. **TypeScript 型レベル**: `serializeable` 型ヘルパーや [`Serializable<T>`](https://github.com/sindresorhus/type-fest) ユーティリティで境界 props の型を制約する
2. **ランタイム検出**: Next.js は dev mode で `Only plain objects, and a few built-ins, can be passed to Client Components from Server Components` という警告を出す。CI で Build ログを grep して落とす運用
3. **境界コンポーネントの命名規約**: Client Component の props 型を `XxxClientProps` のように命名し、レビューで重点チェック
4. **設計レベル**: Server Component の戻り値を **DTO（Data Transfer Object）** として明示的に定義し、Server 側で `toDTO()` 変換を挟む。DTO 化の目的は、機密フィールド（仕入れ値・内部メモなど）を Client に渡さないことと、Prisma の `Decimal` などクラスのインスタンスをプレーンな値に変換すること（`Date` や `Set` は変換しなくても渡せる）

**コード例（DTO パターン）**:
```tsx
// Bad: Prisma の戻り値をそのまま渡す
async function ProductPage({ id }: { id: string }) {
  const product = await prisma.product.findUnique({ where: { id } });
  return <ProductDetail product={product!} />; // Decimal（クラスのインスタンス）でエラーになり、仕入れ値などの内部フィールドも渡ってしまう
}

// Good: Server 側で DTO に変換してから渡す
type ProductDTO = {
  id: string;
  name: string;
  priceJpy: number;          // Prisma Decimal（クラスのインスタンス）→ number
  publishedAt: Date;          // Date はそのまま渡せる
  // costPriceJpy などの内部フィールドは含めない
};

function toProductDTO(p: Product): ProductDTO {
  return {
    id: p.id,
    name: p.name,
    priceJpy: p.priceJpy.toNumber(),
    publishedAt: p.publishedAt,
  };
}

async function ProductPage({ id }: { id: string }) {
  const product = await prisma.product.findUnique({ where: { id } });
  return <ProductDetail product={toProductDTO(product!)} />;
}
```

**出典引用**:
> "関数は書き起こせません" / "渡せるのは「文字に書き起こせる値」だけと覚えるのが実用的です"
> ([「Event handlers cannot be passed to Client Component props」は 'use client' を付けても直らない](https://qiita.com/kai_kou/items/31edab0b35b2ce93f1e1), セクション "なぜ関数だけ渡せないのか") ※2026-08-19に実際にfetch成功

**出典**:
- [Next.js Docs: Passing Props from Server to Client Components](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns#passing-props-from-server-to-client-components-serialization) (Next.js公式)
- [React Docs: Server Components - Serialization](https://react.dev/reference/rsc/server-components#serializable-types) (React公式)
- [React Docs: 'use client' — Serializable types returned by Server Components](https://react.dev/reference/rsc/use-client#serializable-types) (React 公式、Server から Client へ渡せる型と渡せない型の一覧) ※2026-10-25 確認
- [「Event handlers cannot be passed to Client Component props」は 'use client' を付けても直らない](https://qiita.com/kai_kou/items/31edab0b35b2ce93f1e1) (Qiita、'use client' を付けても直らない理由と2つの修正方法の実機検証) ※2026-08-19 fetch

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 4. `"use server"` と `"use client"` の境界を意識したコンポーネント設計をする

Server/Client 境界は「どこでデータを取得し、どこでインタラクションを処理するか」を明確に分割する設計上の境界線として扱う。境界をまたぐ際のパターンを統一することで、予期しないクライアントバンドル肥大化や、サーバー専用コードの漏洩を防ぐ。

**根拠**:
- `'use client'` はモジュールの依存グラフの境界であり、そのモジュールが import するモジュールはすべてクライアントバンドルに含まれる
- Client Component のモジュールから import したコンポーネントは、`'use client'` がなくてもクライアントで評価される（Server Component としては動かない）。サーバー専用のモジュール（`server-only` を付けた DAL など）は Client から import するとビルドエラーになる
- サーバーで実行したいコンポーネントは import せず、`children` や `props` 経由で Client Component に渡す「サーバーコンポーネントの合成」パターンを使う。Context Provider のようにツリーの上位に置かざるを得ない Client Component でも、`children` で Server Component を組み込める

**コード例**:
```tsx
// Good: Server Component を children として Client Component に渡す
// Server Component はクライアントバンドルに含まれない

// components/ClientShell.tsx
'use client';
import { useState } from 'react';

export function ClientShell({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false);
  return (
    <div>
      <button onClick={() => setOpen(o => !o)}>Toggle</button>
      {open && children}
    </div>
  );
}

// app/page.tsx (Server Component)
import { ClientShell } from '@/components/ClientShell';
import { HeavyServerContent } from '@/components/HeavyServerContent'; // Server Component

export default async function Page() {
  return (
    <ClientShell>
      <HeavyServerContent />  {/* Server Component を children で渡す */}
    </ClientShell>
  );
}

// Bad: Client Component 内で Server-only モジュールをインポート
// 'use client'
// import { db } from '@/lib/db'; // サーバー専用コードがクライアントに入る（server-only を付けていればビルドエラーで検出できる）

// Bad: Client Component から Server Component のつもりのコンポーネントを import する
// 'use client'
// import { HeavyServerContent } from './HeavyServerContent'; // クライアントで評価され、サーバーでは動かない
```

**出典**:
- [Next.js Docs: Composition Patterns](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns) (Next.js公式 / 2024)
- [Next.js Docs: Server and Client Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components) (Next.js公式 / 2024)
- [React Docs: 'use client'](https://react.dev/reference/rsc/use-client) (React 公式、`'use client'` のモジュールの推移的な依存はクライアントで評価され、children で渡したコンポーネントはサーバーのまま) ※2026-10-25 確認
  > "A component usage is considered a Client Component if it is defined in module with `'use client'` directive or when it is a transitive dependency of a module that contains a `'use client'` directive. Otherwise, it is a Server Component."

**バージョン**: Next.js 15+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 5. Streaming with Suspense でページの初期表示を高速化する

重いデータを取得するコンポーネントを `<Suspense>` でラップし、フォールバックUIを表示しながらコンテンツをストリームする。ページ全体を待たせるのではなく、準備できたコンテンツから順次表示する。
Suspense でのデータ取得の分け方・LCP 要素の置き場所・コード例は `practices/nextjs/data-fetching.md` #3 を正本とする。

**根拠**:
- Server Component の観点では、データを取得するコンポーネントを `async` の Server Component として切り出し、その単位で `<Suspense>` を張るのが基本になる（詳細は `data-fetching.md` #3）
- ストリーミングを開始するとステータスコードが確定するので、`notFound()` などで 404 を返したい場合は境界の外で先に判定する（`routing.md` #9）

**出典**:
- [Next.js Docs: Streaming with Suspense](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming) (Next.js公式 / 2024)
- [Next.js Docs: Data Fetching Patterns](https://nextjs.org/docs/app/building-your-application/data-fetching/patterns) (Next.js公式 / 2024)

**バージョン**: Next.js 15+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 6. レンダー中にブラウザ専用 API へ直接アクセスしない — サーバーでは `window` は存在せず、失敗は静かに起きる

Server Component / SSR 実行パスのレンダー本体で `window` / `document` 等のブラウザ専用 API に直接触れると、サーバーには存在しないためレンダーが失敗する。しかもブラウザで検証しても再現しないため気づきにくい。ブラウザ API が必要な値は `useEffect` 内でのみ読み、初期値は SSR で決定可能な値（またはサーバーから渡された値）にする。

**根拠**:
- サーバーには `window` オブジェクトが存在しない。レンダー本体（コンポーネント関数のトップレベル）で参照すると `ReferenceError: window is not defined` でそのページのサーバーレンダリングが失敗する
- ブラウザで動作確認しても「ブラウザには `window` がある」ため問題が再現せず、CI やローカル開発では見つかりにくい。本番のクローラー/SSR 経路でのみ症状が出る
- 症状は派手なエラー画面ではなく「サーバーレンダリングが黙って無効化され、ソフト404やインデックス漏れとして現れる」形を取りうるため、影響範囲に気づくまでに時間がかかる
- `useEffect` はマウント後（クライアント側）にのみ実行されるため、ブラウザ API を安全に読める。初期値は SSR で決定できる値にしておき、マウント後に実測値で更新する

**コード例**:
```tsx
// Bad: レンダー本体で window に直接アクセス（サーバーで ReferenceError）
function Banner() {
  const isMobile = window.innerWidth < 640;
  return <div>{isMobile ? <MobileBanner /> : <DesktopBanner />}</div>;
}

// Good: useEffect 内でのみ window を読み、初期値はSSRで決定可能な値にする
function Banner() {
  const [isMobile, setIsMobile] = useState(false);

  useEffect(() => {
    const sync = () => setIsMobile(window.innerWidth < 640);
    sync();
    window.addEventListener('resize', sync);
    return () => window.removeEventListener('resize', sync);
  }, []);

  return <div>{isMobile ? <MobileBanner /> : <DesktopBanner />}</div>;
}

// Good: サーバー側で取得したデータを Client Component に渡す構成にする
export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const show = await getShow(id);
  return <ShowDetailsClient initialShow={show} />;
}
```

**出典引用**:
> "On the server there is no `window`. That line throws `ReferenceError: window is not defined`"
> ([One line of JavaScript disabled server rendering on 190 pages](https://dev.to/eugen_taranowski/one-line-of-javascript-disabled-server-rendering-on-190-pages-5akb), セクション本文) ※2026-08-19に実際にfetch成功

> "Touching a browser API during render doesn't fail loudly. It fails by turning off the thing you can't see from a browser."
> ([One line of JavaScript disabled server rendering on 190 pages](https://dev.to/eugen_taranowski/one-line-of-javascript-disabled-server-rendering-on-190-pages-5akb), セクション本文) ※2026-08-19に実際にfetch成功

**出典**:
- [One line of JavaScript disabled server rendering on 190 pages](https://dev.to/eugen_taranowski/one-line-of-javascript-disabled-server-rendering-on-190-pages-5akb) (dev.to、`window.innerWidth` の直接参照が190ページのSSRを無効化した実例と `useEffect` への修正) ※2026-08-19 fetch

**バージョン**: Next.js 13+（App Router SSR 全般）
**確信度**: 中（単一記事だが Next.js/React 公式の SSR 実行モデル — サーバーにブラウザグローバルが存在しない — を直接示す再現コード付き検証のためパターン1c扱い）
**最終更新**: 2026-08-19

---

### 7. Hydration Error は「window が存在しない」以外にも原因が複数ある — 非決定的な値・不正な HTML ネスト・クライアント限定状態を疑う

Rule #6 の「レンダー本体でブラウザ専用 API に直接触れるとサーバーが `ReferenceError` で落ちる」は Hydration 問題の一部でしかない。より頻度が高いのは、SSR 自体は成功するがサーバーが出力した HTML とクライアントが再構築する DOM が一致せず、React が Hydration を中断する「Hydration Mismatch」で、これは別の原因群を持つ。代表的な原因は (1) `Math.random()` / `Date.now()` など非決定的な値をレンダー本体で使う、(2) `<p>` の中に `<div>` を置くなど無効な HTML ネスト（ブラウザが自動補正した結果クライアント側DOMだけ構造が変わる）、(3) `localStorage` の値や認証状態などクライアントでしか分からない値で条件分岐する、の3つ。対処は「非決定的な値は `useId()` で安定化する」「クライアント限定の値はマウント済みフラグを経由して `useEffect` 内でのみ読む」「どうしても揃えられない場合のみ `suppressHydrationWarning` で対象要素に限定して黙らせる（乱用しない）」。

**根拠**:
- Hydration Mismatch は React が「サーバー出力とクライアント出力が食い違ったため Hydration を中断した」際に起きるもので、Rule #6 の `ReferenceError` によるサーバー側の即時クラッシュとは別の失敗モードである
- `useId()` は React 18 で導入された安定 ID 生成 API で、`Math.random()` 等の代わりに使うとサーバー/クライアントで同じ値を生成できる
- クライアント専用の値（`localStorage` 等）はマウント済みフラグを経由して読むことで、初回レンダーはサーバーと同じ「未確定値」で揃い、マウント後の再レンダーで実値に更新される
- `suppressHydrationWarning` は差分そのものを解消するわけではなく警告を黙らせるだけなので、タイムスタンプ表示など「差分が原理的に避けられない」要素にのみ限定して使う
- マウント済みフラグ経由の `useEffect` はクライアント専用の値を扱う代表的な回避策だが、初回レンダー後にもう1回の再レンダーを挟む。値がレンダー前から DOM 上に実在する場合（`<html data-theme>` など、インラインスクリプトで先に設定済みの属性）は、`useSyncExternalStore` で DOM を「外部ストア」として直接購読する方が、React state に値を二重管理させずに済む

**コード例**:
```tsx
// Bad: 非決定的な値をそのままレンダーに使う（サーバー/クライアントでIDがずれる）
function RandomBadge() {
  const id = Math.random().toString(36);
  return <span id={`badge-${id}`}>Random ID: {id}</span>;
}

// Good: useId() で安定した ID を生成する
'use client';
import { useId } from 'react';

function RandomBadge() {
  const id = useId();
  return <span id={`badge-${id}`}>Random ID: {id}</span>;
}

// Good: クライアント専用の値はマウント済みフラグ経由で useEffect 内でのみ読む
'use client';
import { useEffect, useState } from 'react';

function FavoriteButton() {
  const [isFavorited, setIsFavorited] = useState(false);
  const [isMounted, setIsMounted] = useState(false);

  useEffect(() => setIsMounted(true), []);
  useEffect(() => {
    if (isMounted) {
      setIsFavorited(localStorage.getItem('favorite') === 'true');
    }
  }, [isMounted]);

  return <button>{isFavorited ? 'Unfavorite' : 'Favorite'}</button>;
}

// Good: 値が既に DOM 上にある場合は useSyncExternalStore で外部ストアとして直接購読する
'use client';
import { useSyncExternalStore } from 'react';

function subscribe(callback: () => void) {
  const observer = new MutationObserver(callback);
  observer.observe(document.documentElement, {
    attributes: true,
    attributeFilter: ['data-theme'],
  });
  return () => observer.disconnect();
}

function getSnapshot() {
  return document.documentElement.getAttribute('data-theme') === 'dark' ? 'dark' : 'light';
}

function getServerSnapshot() {
  return 'pending'; // SSR 時点の初期値（マウント後に実値へ更新される）
}

function useDisplayedTheme() {
  return useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
}
```

**出典引用**:
> "The article demonstrates how to replace the nondeterministic call with a stable API (`useId`)"
> ([Fix React 18 Hydration Mismatch in Next.js](https://dev.to/mahdi_benrhouma_fe1c6005/fix-react-18-hydration-mismatch-in-nextjs-371m), セクション "How React 18 decides your page is broken") ※2026-09-09に実際にfetch成功

> "The theme lives in the DOM. React just doesn't know that...React needs to _read_ an external source, not maintain its own parallel copy."
> ([Stop Using a mounted Flag to Fix Theme Flash: Use useSyncExternalStore Instead](https://dev.to/udlxix/stop-using-a-mounted-flag-to-fix-theme-flash-use-usesyncexternalstore-instead-4emo), セクション "Where the theme actually lives") ※2026-09-10に実際にfetch成功

> "サーバーでレンダリングされたHTML構造やコンテンツと、クライアントでReactが生成するDOMツリーが一致しない場合に発生します"
> ([Next.js Hydration Errorを徹底解剖！堅牢な解決策を実装する手順](https://zenn.dev/fd_ai_teacher/articles/tech-20260908172014-1), セクション "Hydration Errorの一般的な発生原因") ※2026-09-09に実際にfetch成功

**出典**:
- [Fix React 18 Hydration Mismatch in Next.js](https://dev.to/mahdi_benrhouma_fe1c6005/fix-react-18-hydration-mismatch-in-nextjs-371m) (dev.to、`useId()` による安定 ID 生成のコード例) ※2026-09-09に実際にfetch成功
- [Next.js Hydration Errorを徹底解剖！堅牢な解決策を実装する手順](https://zenn.dev/fd_ai_teacher/articles/tech-20260908172014-1) (Zenn、マウント済みフラグ経由の `useEffect` パターンと `suppressHydrationWarning` の適用範囲限定というコード例) ※2026-09-09に実際にfetch成功
- 参考として同日 Medium にも同テーマの記事（entepazhe 名義）が見つかったが、本文取得は403/500/408で全段階失敗したため出典に含めていない（取得統計参照）
- [Stop Using a mounted Flag to Fix Theme Flash: Use useSyncExternalStore Instead](https://dev.to/udlxix/stop-using-a-mounted-flag-to-fix-theme-flash-use-usesyncexternalstore-instead-4emo) (dev.to、マウント済みフラグの代替として `useSyncExternalStore` で DOM を外部ストアとして直接購読するコード例) ※2026-09-10 fetch

**バージョン**: Next.js 13+ / React 18+（`useId` は React 18 で導入。`useSyncExternalStore` も React 18 で導入）
**確信度**: 高（パターン2: 異なる著者2名〔dev.to / Zenn〕が同時期に同じ知見を独立に記事化し、両方に具体的なコード例あり。ただし Zenn 側の著者 `fd_ai_teacher` は過去 changelog（2026-08-11）で「AI生成の "tech-daily" 定型記事で品質に疑義」と記録された前歴があるため、dev.to 側のコード例を一次根拠とし Zenn 側は裏付けとして扱う）
**最終更新**: 2026-09-10

---
