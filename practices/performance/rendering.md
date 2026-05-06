# レンダリングパフォーマンスのベストプラクティス

## ルール

### 1. 不要な再レンダリングを `React DevTools Profiler` で特定してから最適化する

最適化は計測なしに行わない。`React DevTools Profiler` でボトルネックを特定してから
`memo`・`useMemo`・`useCallback` を適用する。

**根拠**:
- 計測なしの最適化はコードを複雑にするだけで効果がないことが多い
- `React.memo` 自体もレンダリングコストを持つ（props比較のオーバーヘッド）
- 問題のある再レンダリングは一部のコンポーネントに集中していることが多い

**コード例**:
```tsx
// Step 1: Profiler で計測して問題のあるコンポーネントを特定
// React DevTools → Profiler → Record → 操作 → Stop

// Step 2: 問題のあるコンポーネントにのみ最適化を適用

// 親コンポーネントが再レンダリングされるたびに子も再レンダリングされる問題
// ↓ memo で解決
const ExpensiveChild = React.memo(function ExpensiveChild({
  data,
}: {
  data: ComplexData;
}) {
  return <ComplexVisualization data={data} />;
  // data が変わらなければ再レンダリングをスキップ
});

// 関数を props として渡す場合、参照が毎回変わる問題
// ↓ useCallback で解決
function Parent() {
  const [count, setCount] = useState(0);

  // useCallback なしでは毎回新しい関数参照 → ExpensiveChild が再レンダリング
  const handleAction = useCallback(() => {
    console.log('action');
  }, []);  // 依存なし → 一度だけ生成

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <ExpensiveChild data={data} onAction={handleAction} />
    </>
  );
}
```

**出典**:
- [React Docs: Profiling React App Performance](https://react.dev/learn/react-developer-tools) (React公式)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. 大量リストは仮想スクロール（Virtualization）で最適化する

1000件以上の要素を一度にレンダリングするのを避け、
`@tanstack/react-virtual` などの仮想スクロールライブラリを使う。

**根拠**:
- 大量の DOM ノードはブラウザのレイアウト・ペイントコストを大幅に増加させる
- 仮想スクロールはビューポート内の要素のみをレンダリングする
- 10,000件のリストでも 50件分の DOM ノードしか存在しない

**コード例**:
```tsx
'use client';
import { useVirtualizer } from '@tanstack/react-virtual';
import { useRef } from 'react';

function VirtualList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const rowVirtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 56,  // 各行の推定高さ (px)
    overscan: 5,             // ビューポート外を少し先読み
  });

  return (
    <div
      ref={parentRef}
      style={{ height: '600px', overflow: 'auto' }}
    >
      {/* 全アイテムの合計高さで スクロール領域を確保 */}
      <div style={{ height: rowVirtualizer.getTotalSize(), position: 'relative' }}>
        {rowVirtualizer.getVirtualItems().map(virtualItem => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              transform: `translateY(${virtualItem.start}px)`,
            }}
          >
            <ListItem item={items[virtualItem.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

**出典**:
- [TanStack Virtual Docs](https://tanstack.com/virtual/latest) (TanStack公式)

**バージョン**: @tanstack/react-virtual 3+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. コンテキストの過剰な更新を分割・メモ化で防ぐ

React Context はすべての Consumer を再レンダリングする。
頻繁に更新される値と静的な値を別々の Context に分割する。

**根拠**:
- `Context.Provider` の value が変わると、すべての Consumer が再レンダリングされる
- グローバルな状態管理に Context を使いすぎるとパフォーマンス問題が起きやすい
- 更新頻度の異なる値を分離することで再レンダリングの範囲を最小化できる

**コード例**:
```tsx
// Bad: 頻繁に更新されるカウンターと静的なユーザー情報を同じ Context に入れる
const AppContext = createContext<{
  user: User;
  cartCount: number;  // 頻繁に変わる
  setCartCount: (n: number) => void;
}>({ ... });

// Good: 更新頻度で Context を分割
const UserContext = createContext<User | null>(null);  // 静的

type CartContextType = { count: number; setCount: (n: number) => void };
const CartContext = createContext<CartContextType>({ count: 0, setCount: () => {} });  // 動的

function AppProviders({ children }: { children: React.ReactNode }) {
  const [cartCount, setCartCount] = useState(0);
  const user = useUser();  // 変わらないユーザー情報

  // cartCount の変化は CartContext の Consumer のみを再レンダリング
  return (
    <UserContext.Provider value={user}>
      <CartContext.Provider value={{ count: cartCount, setCount: setCartCount }}>
        {children}
      </CartContext.Provider>
    </UserContext.Provider>
  );
}
```

**出典**:
- [React Docs: Scaling Up with Reducer and Context](https://react.dev/learn/scaling-up-with-reducer-and-context) (React公式)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. `memo`・`useMemo`・`useCallback` の過剰適用を避ける

最適化フックは「計測で問題が確認されたコンポーネント」にのみ適用する。
全コンポーネントへの予防的な適用はコードの複雑さを増すだけで逆効果になる。

**根拠**:
- `React.memo` は毎回 props のシャロー比較を行うため、それ自体がコストを持つ
- `useMemo`・`useCallback` もクロージャとキャッシュのメモリコストがある
- Reactの再レンダリング自体は軽量であり、DOM差分計算がボトルネックになることは少ない
- 誤った依存配列（空配列・過剰な依存）はバグの原因になる

**コード例**:
```tsx
// Bad: 全コンポーネントを予防的に memo でラップする
const SimpleLabel = React.memo(function SimpleLabel({ text }: { text: string }) {
  // 軽量なコンポーネントのメモ化は props 比較のオーバーヘッドが再レンダリングコストを上回る
  return <span>{text}</span>;
});

// Bad: 単純な値の計算に useMemo を使う
function Component({ items }: { items: string[] }) {
  // 単純な配列操作に useMemo は不要（メモ化コスト > 計算コスト）
  const count = useMemo(() => items.length, [items]);
  return <div>{count}</div>;
}

// Good: 高コストな計算（重いフィルタリング・ソート）にのみ useMemo を適用
function ExpensiveFilterList({ items, filter }: { items: Item[]; filter: string }) {
  // Profiler で計測後、実際にボトルネックと確認された場合のみ適用
  const filtered = useMemo(
    () => items.filter(item => heavyMatchFunction(item, filter)),
    [items, filter]
  );
  return <ItemList items={filtered} />;
}

// Good: memo と useCallback の組み合わせは、子が重い場合のみ意味がある
const HeavyChild = React.memo(HeavyChildComponent); // Profilerで重いと確認済み

function Parent() {
  // HeavyChild が memo されていて初めて useCallback が意味を持つ
  const handleChange = useCallback((value: string) => {
    // ...
  }, []);
  return <HeavyChild onChange={handleChange} />;
}
```

**出典**:
- [React Docs: You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) (React公式)
- [React Docs: memo](https://react.dev/reference/react/memo) (React公式)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 5. `startTransition` でレンダリング優先度を制御し UI の応答性を保つ

ユーザーの即時フィードバックを必要とする更新と、時間のかかる二次的な更新を
`startTransition` で分離し、入力遅延をなくす。

**根拠**:
- React 18 の Concurrent Features により、更新に優先度をつけられるようになった
- `startTransition` で囲まれた更新は「非緊急」として扱われ、緊急な更新（入力反映など）が優先される
- タブ切り替え・検索フィルタリング・ページ遷移などの重い UI 更新に有効
- INP（Interaction to Next Paint）の改善に直接寄与する

**コード例**:
```tsx
'use client';
import { startTransition, useTransition, useState } from 'react';

// パターン1: useTransition でペンディング状態を表示
function TabSwitcher({ tabs }: { tabs: Tab[] }) {
  const [activeTab, setActiveTab] = useState(tabs[0].id);
  const [isPending, startTransition] = useTransition();

  function handleTabChange(tabId: string) {
    // タブの見た目の変更は即座に（緊急）
    // 重いコンテンツレンダリングは非緊急として後回し
    startTransition(() => {
      setActiveTab(tabId);
    });
  }

  return (
    <div>
      <div role="tablist">
        {tabs.map(tab => (
          <button
            key={tab.id}
            role="tab"
            aria-selected={tab.id === activeTab}
            onClick={() => handleTabChange(tab.id)}
          >
            {tab.label}
          </button>
        ))}
      </div>
      {/* isPending 中はタブコンテンツを薄く表示（UIがブロックされない） */}
      <div style={{ opacity: isPending ? 0.6 : 1 }}>
        <TabContent tabId={activeTab} />
      </div>
    </div>
  );
}

// パターン2: startTransition（インポート版）でルーター遷移を最適化
import { startTransition } from 'react';
import { useRouter } from 'next/navigation';

function NavigationButton({ href, children }: { href: string; children: React.ReactNode }) {
  const router = useRouter();

  return (
    <button
      onClick={() => {
        startTransition(() => {
          router.push(href);  // ページ遷移を非緊急として処理
        });
      }}
    >
      {children}
    </button>
  );
}
```

**出典**:
- [React Docs: startTransition](https://react.dev/reference/react/startTransition) (React公式)
- [React Docs: useTransition](https://react.dev/reference/react/useTransition) (React公式)

**バージョン**: React 18+, Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-06

---
