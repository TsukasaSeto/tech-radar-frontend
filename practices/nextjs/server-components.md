# Server Components のベストプラクティス

## ルール

### 1. デフォルトは Server Component、必要な場合のみ `'use client'` を付ける

Next.js App Router では全コンポーネントがデフォルトで Server Component になる。
インタラクティビティ（イベントハンドラー・useState・useEffect）が必要な場合のみ
ファイル先頭に `'use client'` を付ける。

**根拠**:
- Server Components はバンドルサイズに含まれない（JS がゼロ）
- データベースや内部 API に直接アクセスできる（認証情報を漏らさない）
- クライアントに送信するデータ量を減らせる

**コード例**:
```tsx
// Server Component（デフォルト）: 'use client' なし
// app/users/page.tsx
import { db } from '@/lib/db';

export default async function UsersPage() {
  const users = await db.user.findMany();
  return (
    <ul>
      {users.map(user => (
        <UserCard key={user.id} user={user} />
      ))}
    </ul>
  );
}

// Client Component: インタラクティビティが必要
'use client';

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
**最終更新**: 2026-05-05

---

### 2. `'use client'` 境界はリーフコンポーネントの近くに置く

`'use client'` を付けると、そのコンポーネント以下の子コンポーネントも
全てクライアントコンポーネントになる。できるだけツリーの末端（リーフ）に置く。

**根拠**:
- `'use client'` が上位にあると、不必要に多くのコンポーネントがクライアントバンドルに含まれる
- Server Component の恩恵（DBアクセス、バンドルサイズ削減）を最大化できる

**コード例**:
```tsx
// Good: Client Component を末端に限定
// app/page.tsx (Server Component)
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
export function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

**出典**:
- [Next.js Docs: Moving Client Components Down the Tree](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns#moving-client-components-down-the-tree) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. Server Components から Client Components に props を渡す際はシリアライズ可能な値のみ

Server Components から Client Components への props は JSON シリアライズ可能な値に限る。関数は Server Actions 経由か Client Component 内で定義する。

**根拠**:
- Server と Client の境界を越えるデータは React によってシリアライズされる
- シリアライズできない値を渡すとエラーになる
- **よくある誤解**: `Event handlers cannot be passed to Client Component props` エラーが出た際、受け取り側の Client Component に既に `'use client'` を付けているのに直らないケースがある。原因はコンポーネントではなく、渡している**関数がどこで生成されたか**。Server Component の中で定義したインライン関数（`onClick={() => ...}`）はその時点で「サーバー側の値」であり、`'use client'` は境界を越えた後の話でしかない。「文字に書き起こせる値だけが渡せる」と覚え、関数は Client Component 側で定義するか Server Actions（`'use server'`）にする

**コード例**:
```tsx
// Good: Server Actions を使う
export default async function Page() {
  async function handleAction(formData: FormData) {
    'use server';
    await saveData(formData);
  }
  return <form action={handleAction}><button>Submit</button></form>;
}
```

**非シリアライズ値カタログ（よくある落とし穴）**:

React の Server/Client 境界では以下の値が **silent に破壊** されるか **明示的にエラー** になる。
ビルドは通っても実行時に壊れるケースが多いため、props 設計時にチェックする。

| 値 | 境界越え時の挙動 | 対処 |
|---|---|---|
| `Date` | プレーン object に変換、`.getTime()` 等が消える | Client 側で `new Date(isoString)` に再変換 / `toISOString()` で送る |
| `Map` / `Set` | プレーン object / array に変換、メソッド消失 | `Array.from(map.entries())` で送り Client で再構築 |
| `Function`（任意の関数） | **エラー** | Server Action（`'use server'`）として渡す or Client 側で定義 |
| `Class instance`（Prisma 等） | プロトタイプ消失、メソッド呼び出し不可 | プレーン object に変換（`{ ...instance }` や DTO 化） |
| `RegExp` | プレーン object に変換、`.test()` 不可 | 文字列で送り Client 側で `new RegExp()` |
| `Symbol`（非 well-known） | **エラー** | 文字列・列挙型に置き換え |
| `BigInt` | **エラー**（JSON.stringify 不可） | 文字列化して送信、Client で `BigInt()` |
| `Error` インスタンス | プレーン object 化、stack/cause が部分的に欠落 | Result 型でエラーを構造化（`architecture/error-handling.md` Rule 2 参照） |
| `Promise` | Server で await される（React 19+）/ 未解決時 Suspense | `use()` フックで Client から扱うパターンを採用 |
| 循環参照 | **エラー** | DTO 化で循環を切る |

**検出方法**:
1. **TypeScript 型レベル**: `serializeable` 型ヘルパーや [`Serializable<T>`](https://github.com/sindresorhus/type-fest) ユーティリティで境界 props の型を制約する
2. **ランタイム検出**: Next.js は dev mode で `Only plain objects, and a few built-ins, can be passed to Client Components from Server Components` という警告を出す。CI で Build ログを grep して落とす運用
3. **境界コンポーネントの命名規約**: Client Component の props 型を `XxxClientProps` のように命名し、レビューで重点チェック
4. **設計レベル**: Server Component の戻り値を **DTO（Data Transfer Object）** として明示的に定義し、Server 側で `toDTO()` 変換を挟む

**コード例（DTO パターン）**:
```tsx
// Bad: Prisma の戻り値（class instance）をそのまま渡す
async function ProductPage({ id }: { id: string }) {
  const product = await prisma.product.findUnique({ where: { id } });
  return <ProductDetail product={product!} />; // Date / Decimal で問題発生
}

// Good: Server 側で DTO に変換してから渡す
type ProductDTO = {
  id: string;
  name: string;
  priceJpy: number;          // Prisma Decimal → number
  publishedAt: string;        // Date → ISO string
  tags: string[];             // Set → Array
};

function toProductDTO(p: Product): ProductDTO {
  return {
    id: p.id,
    name: p.name,
    priceJpy: p.priceJpy.toNumber(),
    publishedAt: p.publishedAt.toISOString(),
    tags: Array.from(p.tagSet),
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
- [「Event handlers cannot be passed to Client Component props」は 'use client' を付けても直らない](https://qiita.com/kai_kou/items/31edab0b35b2ce93f1e1) (Qiita、'use client' を付けても直らない理由と2つの修正方法の実機検証) ※2026-08-19 fetch

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05 / 補強 2026-05-16 / 2026-08-19

---

### 4. `"use server"` と `"use client"` の境界を意識したコンポーネント設計をする

Server/Client 境界は「どこでデータを取得し、どこでインタラクションを処理するか」を明確に分割する設計上の境界線として扱う。境界をまたぐ際のパターンを統一することで、予期しないクライアントバンドル肥大化や、サーバー専用コードの漏洩を防ぐ。

**根拠**:
- `'use client'` はモジュールグラフの「切断点」であり、その配下のすべてがクライアントバンドルに含まれる
- Server Component は `'use client'` のコンポーネントを直接インポートできるが、逆は不可
- Server Component を `children` や `props` 経由で Client Component に渡す「サーバーコンポーネントの合成」パターンを活用する

**コード例**:
```tsx
// Good: Server Component を children として Client Component に渡す
// Server Component はクライアントバンドルに含まれない

// components/ClientShell.tsx
'use client';
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
// import { db } from '@/lib/db'; // エラー: サーバー専用コードがクライアントに漏洩
```

**出典**:
- [Next.js Docs: Composition Patterns](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns) (Next.js公式 / 2024)
- [Next.js Docs: Server and Client Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components) (Next.js公式 / 2024)

**バージョン**: Next.js 15+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 5. Streaming with Suspense でページの初期表示を高速化する

重いデータを取得するコンポーネントを `<Suspense>` でラップし、フォールバックUIを表示しながらコンテンツをストリームする。ページ全体を待たせるのではなく、準備できたコンテンツから順次表示する。

**根拠**:
- サーバーはHTMLを段階的にストリーミングし、ブラウザは受け取り次第レンダリングできる
- Time To First Byte（TTFB）を下げ、ユーザーの体感速度を向上させる
- `loading.tsx` はルート全体にSuspenseを張るショートカットだが、コンポーネント粒度でラップすればより細かい制御が可能

**コード例**:
```tsx
// Good: 重いコンポーネントだけを Suspense でラップし、残りは即時表示
// app/dashboard/page.tsx
import { Suspense } from 'react';

export default function DashboardPage() {
  return (
    <main>
      <PageHeader />  {/* 即時表示（データフェッチなし）*/}
      <Suspense fallback={<StatsSkeleton />}>
        <StatsPanel />  {/* 重いデータ取得 → ストリーミング */}
      </Suspense>
      <Suspense fallback={<FeedSkeleton />}>
        <ActivityFeed />  {/* 別の重いデータ取得 → 並列ストリーミング */}
      </Suspense>
    </main>
  );
}

// components/StatsPanel.tsx (Server Component)
export async function StatsPanel() {
  const stats = await fetchHeavyStats(); // 時間のかかる処理
  return <div>{/* stats を表示 */}</div>;
}

// Bad: ページ全体を一つの async Server Component にまとめて全データを待つ
export default async function DashboardPage() {
  const [stats, feed] = await Promise.all([fetchHeavyStats(), fetchActivityFeed()]);
  // 両方の取得が完了するまでユーザーは何も見えない
  return <div>...</div>;
}
```

**出典**:
- [Next.js Docs: Streaming with Suspense](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming) (Next.js公式 / 2024)
- [Next.js Docs: Data Fetching Patterns](https://nextjs.org/docs/app/building-your-application/data-fetching/patterns) (Next.js公式 / 2024)

**バージョン**: Next.js 15+
**確信度**: 高
**最終更新**: 2026-05-06

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
