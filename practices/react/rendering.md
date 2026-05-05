# レンダリング最適化のベストプラクティス

## ルール

### 1. `memo` は実際にパフォーマンス問題が発生してから使う

`React.memo` はプロファイラーで問題を確認する前に使うのは早すぎる。

**根拠**:
- `memo` 自体も props の比較コストがかかる
- 無駄な最適化はコードを複雑にする
- React Compiler (React 19+) が自動的に最適化する方向に向かっている

**コード例**:
```tsx
// 適切: プロファイラーで重い再レンダリングが確認された後に使用
const HeavyList = memo(function HeavyList({
  items,
  onItemClick,
}: {
  items: Item[];
  onItemClick: (id: string) => void;
}) {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id} onClick={() => onItemClick(item.id)}>
          {item.name}
        </li>
      ))}
    </ul>
  );
});

// memo の効果を出すには props も安定させる
function Parent() {
  const handleClick = useCallback((id: string) => {
    console.log(id);
  }, []);
  return <HeavyList items={items} onItemClick={handleClick} />;
}
```

**出典**:
- [React Docs: memo](https://react.dev/reference/react/memo) (React公式)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. `useMemo` / `useCallback` は高コストな計算にのみ使う

**根拠**:
- メモ化は「記憶する」コストがかかる（メモリ使用量増加）
- 依存配列の比較もコストがかかる
- プリミティブ値の計算ならメモ化は不要

**コード例**:
```tsx
// Bad: 不要な useMemo
const doubled = useMemo(() => count * 2, [count]);

// Good: メモ化なしで十分
const doubled = count * 2;

// Good: useMemo が有効なケース
const sortedItems = useMemo(
  () => items.slice().sort((a, b) => a.price - b.price),
  [items]
);
```

**出典**:
- [React Docs: useMemo](https://react.dev/reference/react/useMemo) (React公式)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. リストには必ず安定した `key` を使う

`key` には配列インデックスではなく、データの一意のIDを使う。

**根拠**:
- React は `key` を使ってコンポーネントのidentityを判断する
- インデックスが key だと並び替え時に誤ったコンポーネントを再利用する

**コード例**:
```tsx
// Bad: インデックスをkeyに使用
{items.map((item, index) => (
  <TodoItem key={index} item={item} />
))}

// Good: 安定したIDをkeyに使用
{items.map(item => (
  <TodoItem key={item.id} item={item} />
))}
```

**出典**:
- [React Docs: Rendering Lists](https://react.dev/learn/rendering-lists#keeping-list-items-in-order-with-key) (React公式)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. 状態更新はできるだけバッチ処理される（React 18以降）

React 18 では、全ての状態更新が自動的にバッチ処理される（Automatic Batching）。

**根拠**:
- 複数の `setState` 呼び出しが一回のレンダリングにまとめられる
- React 17以前は非同期コールバック内の更新はバッチされなかった

**コード例**:
```tsx
// React 18+: 全て自動バッチ（1回のレンダリング）
function handleClick() {
  setCount(c => c + 1);
  setFlag(f => !f); // ここでまとめて1回再レンダリング
}

// 強制同期が必要な場合
import { flushSync } from 'react-dom';
flushSync(() => setCount(c => c + 1));
const height = divRef.current!.offsetHeight;
```

**出典**:
- [React 18 Automatic Batching](https://react.dev/blog/2022/03/29/react-v18#new-feature-automatic-batching) (React公式ブログ)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-05
