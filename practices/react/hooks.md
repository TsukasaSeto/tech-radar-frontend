# React Hooks のベストプラクティス

このファイルはフォーマットのサンプルです。
ブートストラップ実行時に上書き or 拡張されます。

## ルール

### 1. useEffect でデータ取得しない（Reactの公式立場）

データ取得は Server Components、ライブラリ（TanStack Query、SWR）、
またはフレームワークの仕組み（Next.js の Server Components / Server Actions）に委ねる。
useEffect でのfetchはレース条件、メモリリーク、ウォーターフォールを招く。

**根拠**:
- React公式が `useEffect` のデータ取得を推奨しないと明記
- レース条件の発生（古いリクエストの結果が新しい状態を上書き）
- StrictModeで二重実行される副作用への対処が複雑化

**コード例**:
```tsx
// Bad: useEffectでのfetch
function Profile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(setUser);
  }, [userId]);
  return user ? <div>{user.name}</div> : null;
}

// Good: ライブラリに委ねる
function Profile({ userId }: { userId: string }) {
  const { data: user } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });
  return user ? <div>{user.name}</div> : null;
}

// Better (Next.js): Server Componentでデータ取得
async function Profile({ userId }: { userId: string }) {
  const user = await fetchUser(userId);
  return <div>{user.name}</div>;
}
```

**出典**:
- [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) (React公式 / 2024)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. 派生状態は state ではなく計算で表現する

他のstateやpropsから導出できる値は、`useState` + `useEffect` で同期するのではなく、
レンダリング時に計算する。

**根拠**:
- 二重ソース・オブ・トゥルースの回避
- 不要な再レンダリングの削減
- バグの温床（同期漏れ）を排除

**コード例**:
```tsx
// Bad
function TodoList({ todos, filter }: Props) {
  const [visible, setVisible] = useState<Todo[]>([]);
  useEffect(() => {
    setVisible(todos.filter(t => t.status === filter));
  }, [todos, filter]);
  return <List items={visible} />;
}

// Good
function TodoList({ todos, filter }: Props) {
  const visible = todos.filter(t => t.status === filter);
  return <List items={visible} />;
}

// 計算が重い場合のみ useMemo
function TodoList({ todos, filter }: Props) {
  const visible = useMemo(
    () => expensiveFilter(todos, filter),
    [todos, filter]
  );
  return <List items={visible} />;
}
```

**出典**:
- [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) (React公式 / 2024)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

<!-- ここから先はブートストラップ実行で追加されます -->
