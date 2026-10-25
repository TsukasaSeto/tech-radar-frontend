# レンダリングパフォーマンスのベストプラクティス

> **役割分担**: このファイルは **計測駆動の最適化テクニック**（Profiler での再レンダー特定、仮想スクロール、コンテキスト分割、Core Web Vitals 連携）を扱う。
> **React API の基本パターン**（`memo` / `useMemo` / `startTransition` の使いどころと意味論）は [`react/rendering.md`](../react/rendering.md) を参照。
> 「いつどう書くか」は react 側、「測ってどう調整するか」はこのファイル の分担。

## ルール

### 1. 不要な再レンダリングを `React DevTools Profiler` で特定してから最適化する

最適化は計測なしに行わない。`React DevTools Profiler` でボトルネックを特定してから
`memo`・`useMemo`・`useCallback` を適用する。予防的な過剰適用を避ける具体例は #4 を参照。
React Compiler を導入しているコンポーネントは、コンパイラが自動でメモ化するので手動の `memo` / `useMemo` / `useCallback` は原則書かない。Profiler で計測して問題を特定する手順は同じで、対処はコンパイラの対象範囲の見直し（`practices/react/rendering.md` #7）から始める。

**根拠**:
- 計測なしの最適化はコードを複雑にするだけで効果がないことが多い
- `React.memo` 自体もレンダリングコストを持つ（props比較のオーバーヘッド）
- 問題のある再レンダリングは一部のコンポーネントに集中していることが多い

**コード例**:
```tsx
import { memo, useCallback, useState } from 'react';

// Step 1: Profiler で計測して問題のあるコンポーネントを特定
// React DevTools → Profiler → Record → 操作 → Stop

// Step 2: 問題のあるコンポーネントにのみ最適化を適用

// 親コンポーネントが再レンダリングされるたびに子も再レンダリングされる問題
// ↓ memo で解決
const ExpensiveChild = memo(function ExpensiveChild({
  data,
  onAction,
}: {
  data: ComplexData;
  onAction: () => void;
}) {
  return <ComplexVisualization data={data} onAction={onAction} />;
  // data と onAction の参照が変わらなければ再レンダリングをスキップ
});

// 関数を props として渡す場合、参照が毎回変わる問題
// ↓ useCallback で解決
function Parent({ data }: { data: ComplexData }) {
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
- [React Docs: React Compiler](https://react.dev/learn/react-compiler) (React公式)

**バージョン**: React 18+（React Compiler は 1.0+）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 2. 大量リストは仮想スクロール（Virtualization）で最適化する

1000件以上の要素を一度にレンダリングするのを避け、
`@tanstack/react-virtual` などの仮想スクロールライブラリを使う。

**根拠**:
- 大量の DOM ノードはブラウザのレイアウト・ペイントコストを大幅に増加させる
- 仮想スクロールはビューポート内の要素のみをレンダリングする
- 10,000件のリストでも 50件分の DOM ノードしか存在しない
- 行の高さが固定値で見積もれない場合（コメントスレッドのような可変高さコンテンツが混在する等）は、高さの見積もりを「行の種類」で2系統に分ける設計が有効: 決定論的に計算できる要素（コード行など）は事前に正確な高さを積み上げ、動的な要素（コメント等）はビューポート付近（実測で約2400px手前）に来たときだけ実測してから合計する。GitHub Copilot アプリでは 2,200ファイル・100万行超・400件超のインラインコメントを含む PR でこの2系統設計により、常時マウントする行数を約100行に抑えつつ描画を成立させている

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

**出典引用**:
> "total height = deterministic code height (exact, known up front) + Σ dynamic block effective heights (estimated, then measured)"
> ([Rendering huge pull requests in the GitHub Copilot app](https://github.blog/engineering/user-experience/rendering-huge-pull-requests-in-the-github-copilot-app/), GitHub Blog, セクション "Two geometries instead of one") ※2026-09-23に実際にfetch成功

**出典**:
- [TanStack Virtual Docs](https://tanstack.com/virtual/latest) (TanStack公式)
- [Rendering huge pull requests in the GitHub Copilot app](https://github.blog/engineering/user-experience/rendering-huge-pull-requests-in-the-github-copilot-app/) (GitHub Blog 公式、固定高さ行と可変高さブロックを分離する仮想スクロール設計) ※2026-09-23 fetch

**バージョン**: @tanstack/react-virtual 3+
**確信度**: 高
**最終更新**: 2026-09-23

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

「計測してから適用する」という判断基準と React Compiler 導入時の扱いは #1 を正とする。このルールでは、全コンポーネントへの予防的な適用がコードの複雑さを増すだけで逆効果になる典型例を扱う。

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
**最終更新**: 2026-10-25

---

### 5. `startTransition` でレンダリング優先度を制御し UI の応答性を保つ

ユーザーの即時フィードバックを必要とする更新と、時間のかかる二次的な更新を
分離し、入力遅延をなくす。`startTransition` / `useTransition` / `useDeferredValue` の基本パターンと意味論は `practices/react/rendering.md` #5 / #6 を正とする。このルールでは、Profiler や INP の計測で入力遅延が確認された画面への適用例を扱う。

**根拠**:
- React 18 の Concurrent Features により、更新に優先度をつけられるようになった
- `startTransition` で囲まれた更新は「非緊急」として扱われ、緊急な更新（入力反映など）が優先される
- タブ切り替え・検索フィルタリング・ページ遷移などの重い UI 更新に有効
- INP（Interaction to Next Paint）の改善に直接寄与する
- 選択状態そのもの（`aria-selected` の元になる state）を transition に入れると、タブの選択表示まで遅れる。選択状態は即座に更新し、重いコンテンツだけを `useDeferredValue` で遅らせる

**コード例**:
```tsx
'use client';
import { memo, useDeferredValue, useState } from 'react';
import { HeavyTabContent } from './HeavyTabContent';

// deferredTab が変わるまで再レンダリングをスキップさせるため memo で包む
const TabContent = memo(HeavyTabContent);

// パターン1: 選択状態は即座に、重いタブコンテンツだけ遅れて追従させる
function TabSwitcher({ tabs }: { tabs: Tab[] }) {
  const [activeTab, setActiveTab] = useState(tabs[0].id);
  const deferredTab = useDeferredValue(activeTab);
  const isPending = activeTab !== deferredTab;

  return (
    <div>
      <div role="tablist">
        {tabs.map(tab => (
          <button
            key={tab.id}
            role="tab"
            aria-selected={tab.id === activeTab}  // 選択表示はクリック直後に切り替わる
            onClick={() => setActiveTab(tab.id)}
          >
            {tab.label}
          </button>
        ))}
      </div>
      {/* 古いコンテンツを表示している間は薄く表示（UIがブロックされない） */}
      <div style={{ opacity: isPending ? 0.6 : 1 }}>
        <TabContent tabId={deferredTab} />
      </div>
    </div>
  );
}
```

```tsx
// パターン2: startTransition（インポート版）でルーター遷移を最適化
'use client';
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
- [React Docs: useDeferredValue](https://react.dev/reference/react/useDeferredValue) (React公式)

**バージョン**: React 18+, Next.js 13+
**確信度**: 高
**最終更新**: 2026-10-25

---
