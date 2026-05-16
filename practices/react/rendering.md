# React レンダリングの基本パターン

> **役割分担**: このファイルは **React の基本パターン**（`memo` / `useMemo` / `useCallback` の使いどころ、`startTransition` / `useDeferredValue` の意味論）を扱う。
> **計測駆動の最適化テクニック**（Profiler での再レンダー特定、仮想スクロール、コンテキスト分割、Core Web Vitals 連携）は [`performance/rendering.md`](../performance/rendering.md) を参照。
> 両者は重なる API（`memo`, `startTransition`）を扱うが、**「いつどう書くか」は react 側、「測ってどう調整するか」は performance 側** の分担。

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

---

### 5. `startTransition` / `useTransition` で緊急度の低い更新を後回しにする

ユーザー入力（タイピング）などの緊急な更新と、検索結果一覧の再描画などの非緊急な更新を
`startTransition` で分離する。非緊急な更新はより優先度の低いレーンでスケジュールされ、
ユーザーインタラクションの応答性が向上する。

**根拠**:
- `startTransition` で包まれた更新は「Transition」扱いとなり、入力など緊急更新に割り込まれる
- `useTransition` の `isPending` フラグでローディングインジケーターを表示できる
- `Suspense` と組み合わせると、Transition中は古いUIを維持しつつ新しいUIを準備できる

**コード例**:
```tsx
import { useState, useTransition } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    // 入力の更新は緊急 → startTransition の外
    setQuery(e.target.value);

    // 検索結果の更新は非緊急 → startTransition の中
    startTransition(() => {
      setResults(searchItems(e.target.value));
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending ? <Spinner /> : <ResultList items={results} />}
    </>
  );
}

// Bad: 全て同じ優先度で更新するとタイピングがもたつく
function SearchPageBad() {
  const [query, setQuery] = useState('');
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(e.target.value);
    setResults(searchItems(e.target.value)); // 重い処理がブロック
  };
}
```

**出典**:
- [React Docs: startTransition](https://react.dev/reference/react/startTransition) (React公式)
- [React Docs: useTransition](https://react.dev/reference/react/useTransition) (React公式)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 6. `useDeferredValue` で重い子コンポーネントの再レンダリングを遅延させる

`useDeferredValue` は特定の値の「遅延コピー」を返す。遅延コピーは緊急な更新が
完了した後に非同期で更新されるため、入力フィールドなどの応答性を維持しながら
重い描画処理（大きなリストなど）を後回しにできる。

**根拠**:
- `useTransition` が使えない外部ライブラリのstateに対しても適用できる
- `memo` と組み合わせることで、遅延値が変わるまで子コンポーネントの再レンダリングをスキップできる
- Concurrent Rendering の仕組みを活用し、UI のジャンクを削減する

**コード例**:
```tsx
import { useState, useDeferredValue, memo } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  // query の遅延コピー。タイピング中は古い値を保持
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  return (
    <>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
      />
      <div style={{ opacity: isStale ? 0.5 : 1 }}>
        {/* deferredQuery が変わったときだけ再レンダリング */}
        <HeavyResultList query={deferredQuery} />
      </div>
    </>
  );
}

// memo と組み合わせて deferredQuery が変わるまでスキップ
const HeavyResultList = memo(function HeavyResultList({ query }: { query: string }) {
  const results = expensiveSearch(query);
  return <ul>{results.map(r => <li key={r.id}>{r.name}</li>)}</ul>;
});
```

**出典**:
- [React Docs: useDeferredValue](https://react.dev/reference/react/useDeferredValue) (React公式)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-06

---
