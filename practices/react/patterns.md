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
