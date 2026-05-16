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

**出典**:
- [Next.js Docs: Passing Props from Server to Client Components](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns#passing-props-from-server-to-client-components-serialization) (Next.js公式)
- [React Docs: Server Components - Serialization](https://react.dev/reference/rsc/server-components#serializable-types) (React公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05 / 補強 2026-05-16

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
