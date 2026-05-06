# コンポーネント設計パターンのベストプラクティス

## ルール

### 1. コンポジションを継承より優先する（Composition over Inheritance）

**根拠**:
- React はクラス継承よりコンポジションを推奨している
- `children` パターンにより、内部実装を知らずに拡張できる

**コード例**:
```tsx
// Good: コンポジションで拡張
function FancyButton({
  children,
  ...props
}: React.ButtonHTMLAttributes<HTMLButtonElement>) {
  return (
    <button
      {...props}
      className={`fancy-button ${props.className ?? ''}`}
    >
      {children}
    </button>
  );
}

// Good: children で柔軟なコンポジション
function Card({ children, title }: { children: React.ReactNode; title: string }) {
  return (
    <div className="card">
      <h2>{title}</h2>
      {children}
    </div>
  );
}
```

**出典**:
- [React Docs: Thinking in React](https://react.dev/learn/thinking-in-react) (React公式)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. ロジックをカスタムフックに抽出する

コンポーネント内のロジックが複雑になったらカスタムフックに抽出する。コンポーネントは UI の描画に集中する。

**根拠**:
- カスタムフックでロジックを再利用できる
- コンポーネントが薄くなりテストが容易になる

**コード例**:
```tsx
function useProduct(productId: string) {
  return useQuery({
    queryKey: ['product', productId],
    queryFn: () => fetchProduct(productId),
  });
}

function ProductPage({ productId }: { productId: string }) {
  const { data: product, isLoading } = useProduct(productId);
  const [quantity, setQuantity] = useState(1);

  if (isLoading) return <Spinner />;
  return <div>...</div>;
}
```

**出典**:
- [React Docs: Reusing Logic with Custom Hooks](https://react.dev/learn/reusing-logic-with-custom-hooks) (React公式)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. コンポーネントは小さく保ち、単一責任を守る

1つのコンポーネントは1つの責任だけを持つ。

**根拠**:
- 小さなコンポーネントはテストが容易
- 責任が明確なため変更の影響範囲が限定される

**コード例**:
```tsx
// Good: 責任を分離
function UserDashboard({ userId }: { userId: string }) {
  return (
    <div className="dashboard">
      <UserProfile userId={userId} />
      <UserActivityFeed userId={userId} />
      <UserSettingsPanel userId={userId} />
    </div>
  );
}
```

**出典**:
- [React Docs: Thinking in React](https://react.dev/learn/thinking-in-react) (React公式)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. Render Props パターンより Custom Hooks を優先する

Render Props や HOC でのロジック共有は Custom Hooks で代替できる。

**根拠**:
- Custom Hooks の方が読みやすくシンプル
- ネストが深くならない（Wrapper Hell を回避）

**コード例**:
```tsx
// Good: Custom Hooks で代替
function useMousePosition() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  useEffect(() => {
    const handler = (e: MouseEvent) =>
      setPosition({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', handler);
    return () => window.removeEventListener('mousemove', handler);
  }, []);
  return position;
}

function FloatingWidget() {
  const { x, y } = useMousePosition();
  const { data, isLoading } = useQuery({ queryKey: ['/api/data'], queryFn: fetchData });
  return (
    <div style={{ position: 'fixed', left: x, top: y }}>
      {isLoading ? 'Loading...' : data}
    </div>
  );
}
```

**出典**:
- [React Docs: Custom Hooks](https://react.dev/learn/reusing-logic-with-custom-hooks) (React公式)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 5. Server Components と Client Components の境界を意識して設計する

Next.js App Router などでは、コンポーネントをデフォルトで Server Component として設計し、
インタラクティブ性が必要な最小限の部分のみを Client Component に切り出す。
`"use client"` ディレクティブは境界の葉（leaf）に近いコンポーネントに限定し、
境界より上にあるコンポーネントツリー全体がクライアントバンドルに含まれることを防ぐ。

**根拠**:
- Server Components はバンドルサイズに影響しない（サーバーのみで実行）
- `"use client"` は宣言されたコンポーネントとその子を全てクライアントバンドルに含める
- データフェッチを Server Component に集中させることでウォーターフォールを回避できる

**コード例**:
```tsx
// Good: Server Component がデータ取得、Client Component はインタラクションのみ
// app/dashboard/page.tsx (Server Component)
async function DashboardPage() {
  const data = await fetchDashboardData(); // サーバーサイドで解決
  return (
    <div>
      <h1>Dashboard</h1>
      <StaticSummary data={data} />      {/* Server Component */}
      <InteractiveChart data={data} />   {/* Client Component */}
    </div>
  );
}

// components/InteractiveChart.tsx
'use client'; // ここだけ Client Component
import { useState } from 'react';

export function InteractiveChart({ data }: { data: ChartData }) {
  const [activeIndex, setActiveIndex] = useState(0);
  return <Chart data={data} activeIndex={activeIndex} onChange={setActiveIndex} />;
}

// Bad: ページトップに 'use client' を置くとツリー全体がクライアント化
'use client';
async function DashboardPage() { // Server Componentの利点がゼロになる
  ...
}
```

**出典**:
- [Next.js Docs: Server and Client Components](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns) (Next.js公式)
- [React Docs: "use client"](https://react.dev/reference/rsc/use-client) (React公式)

**バージョン**: React 19+, Next.js 14+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 6. `"use client"` ディレクティブは境界の葉ノードに最小化し、Serverコンポーネントを Props で貫通させる

`"use client"` 境界の中に Server Component を直接ネストすることはできないが、
`children` や他の props として渡す（Props drilling）ことで Server Component の出力を
Client Component ツリー内に埋め込める。これにより `"use client"` の影響範囲を限定できる。

**根拠**:
- `children` として渡された Server Component はサーバー側でレンダリングされ、クライアントバンドルに含まれない
- Context Provider など「ツリー上位に置かざるを得ない」Client Component でも、children を通じて Server Component を組み込める
- `"use client"` の境界を細かくするほどバンドルサイズを抑制できる

**コード例**:
```tsx
// Good: ClientProvider の children に Server Component を渡す
// app/providers.tsx
'use client';
import { ThemeProvider } from 'some-library';

export function Providers({ children }: { children: React.ReactNode }) {
  return <ThemeProvider>{children}</ThemeProvider>; // children は Server Component でも可
}

// app/layout.tsx (Server Component)
import { Providers } from './providers';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <Providers>
          {children} {/* Server Component がそのまま流れる */}
        </Providers>
      </body>
    </html>
  );
}

// Bad: Client Component 内で直接 Server Component をインポートする
'use client';
import { HeavyServerComponent } from './HeavyServerComponent'; // エラーになる
export function ClientWrapper() {
  return <HeavyServerComponent />; // Server Component を直接ネストは不可
}
```

**出典**:
- [Next.js Docs: Composing Client and Server Components](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns#supported-pattern-passing-server-components-to-client-components-as-props) (Next.js公式)
- [React Docs: "use client"](https://react.dev/reference/rsc/use-client) (React公式)

**バージョン**: React 19+, Next.js 14+
**確信度**: 高
**最終更新**: 2026-05-06

---
