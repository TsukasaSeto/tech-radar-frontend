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

**出典**:
- [Next.js Docs: Passing Props from Server to Client Components](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns#passing-props-from-server-to-client-components-serialization) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05
