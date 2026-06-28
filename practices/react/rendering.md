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
- React Compiler 1.0（2025年10月 stable リリース）が `memo` の最適化を自動化するため、Compiler 有効プロジェクトでは手動 `memo` はほぼ不要になった

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
- [React Compiler 1.0 Is Here — And It Changes Everything You Know About Performance](https://medium.com/@Ardhendu_init_/react-compiler-1-0-is-here-and-it-changes-everything-you-know-about-performance-59a0f5554a36) (Medium、Compiler 1.0 stable と自動 memo 化) ※2026-06-17 fetch

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-06-17

---

### 2. `useMemo` / `useCallback` は高コストな計算にのみ使う

**根拠**:
- メモ化は「記憶する」コストがかかる（メモリ使用量増加）
- 依存配列の比較もコストがかかる
- プリミティブ値の計算ならメモ化は不要
- React Compiler 1.0（2025年10月 stable）有効環境では `useMemo` / `useCallback` の手動記述はほぼ不要。「新しいコードはコンパイラに任せ、詳細な制御が必要な場合にのみ使用することが推奨」（Qiita, 2026-06-17）

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
- [【React Compiler v1.0】useMemo・useCallback はもう書かなくていい？導入手順と実測結果まで解説](https://qiita.com/yuuue/items/c8a395c17ea3ccd995ec) (Qiita、Compiler で手動 memo 化が不要になる根拠と実測) ※2026-06-17 fetch

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-06-17

---

### 3. リストには必ず安定した `key` を使う

`key` には配列インデックスではなく、データの一意のIDを使う。
レンダリング中に `Math.random()` や `crypto.randomUUID()` を呼ぶと、再レンダリングのたびに key が変わりコンポーネントが毎回マウントし直される。

**根拠**:
- React は `key` を使ってコンポーネントのidentityを判断する
- インデックスが key だと並び替え時に誤ったコンポーネントを再利用する
- `Math.random()` を key に使うとレンダリングのたびに全リストアイテムが再マウントされ、入力状態・フォーカス・アニメーションがリセットされる
- UUID を使う場合はオブジェクト生成時に一度だけ払い出し、render 関数内では生成しない

**コード例**:
```tsx
// Bad: インデックスをkeyに使用
{items.map((item, index) => (
  <TodoItem key={index} item={item} />
))}

// Bad: renderごとにランダム生成 → 毎回再マウント
{items.map(item => (
  <TodoItem key={Math.random()} item={item} />
))}

// Good: 安定したIDをkeyに使用
{items.map(item => (
  <TodoItem key={item.id} item={item} />
))}

// Good: UUIDを使う場合は生成をrender外で行う
const newItem = { id: crypto.randomUUID(), name: '...' }; // 追加時に一度だけ
```

**出典引用**:
> "The key is React's way to identify list items; without it, React cannot properly match previous items to current ones when reordering occurs."
> ([なんとなく付けていたReactのkeyの役割を理解する](https://zenn.dev/uraaaa24/articles/bfeb6d3dcb01ba), Zenn, セクション "key がないと何が起きるか") ※2026-05-31に実際にfetch成功

**出典**:
- [React Docs: Rendering Lists](https://react.dev/learn/rendering-lists#keeping-list-items-in-order-with-key) (React公式)
- [なんとなく付けていたReactのkeyの役割を理解する](https://zenn.dev/uraaaa24/articles/bfeb6d3dcb01ba) (Zenn、Math.random anti-pattern と UUID 生成タイミング) ※2026-05-31 fetch

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-31

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

### 7. 大規模コードベースへの React Compiler 導入は `compilationMode: "annotation"` で漸進的に行う

7,000 ファイル超の大規模プロジェクトへ React Compiler を全ファイル一括適用するのではなく、`compilationMode: "annotation"` でファイル単位のオプトインに限定し、Oxlint のデュアルルールと組み合わせてリスクを抑えながら段階的に導入する。

**根拠**:
- React Compiler 1.0 は 2025年10月に stable リリース。実測で**総レンダリング時間 17.2%、平均レンダリング時間 17.2%、最大 12.4%** の削減（既存の手動 memo と共存した状態でも改善）
- 新規プロジェクトでは `next.config.ts` に `reactCompiler: true` のワンライン設定のみで十分。大規模レガシーコードベースでは `compilationMode: "annotation"` での段階導入が安全
- 全ファイル一括（`compilationMode: "all"`）は TanStack Table v8・react-hook-form 等の非互換ライブラリが混在する場合に高リスク
- Oxlint の `react-hooks-js/*` ルール（eslint-plugin-react-hooks v7 経由）でコンパイラ非互換パターンをデプロイ前に静的検出できる
- `"use no memo";` ディレクティブで非互換ファイルを明示的にオプトアウトでき、互換性の問題範囲を局所化できる
- **Interior Mutability 問題**: TanStack Table など「参照は安定しているが内部状態を変更する」ライブラリのオブジェクトを子コンポーネントに props として渡すと、Compiler が参照変化を検出できず再レンダリングをスキップする。修正はオブジェクト自体を渡すのをやめ `table.getHeaderGroups()` のような**派生値（新しい参照が生まれる）を props に渡す**こと。`"use no memo"` はこの問題の根本解決にならない

**コード例**:
```ts
// react-compiler.config.ts
export const ReactCompilerConfig = {
  target: "18",
  compilationMode: "annotation",
} as const;
```

```tsx
// Good: コンパイラ対象ファイルには "use memo"; を冒頭に追加
"use memo";
export function ProductCard({ product }: { product: Product }) {
  return <div>{product.name}</div>;
}

// ライブラリ非互換ファイルには "use no memo"; で明示的にオプトアウト
"use no memo";
import { useForm } from 'react-hook-form'; // Compiler 非互換
```

```jsonc
// .oxlintrc.json — Compiler 非互換パターンを事前検出するデュアルルール
{
  "plugins": ["react-hooks"],
  "rules": {
    "react-hooks/rules-of-hooks": "error",     // 従来のフックルール
    "react-hooks-js/react-compiler": "error"   // Compiler 固有の非互換検出
  }
}
```

**コード例（Interior Mutability の修正パターン）**:
```tsx
// Bad: ミュータブルオブジェクトをそのまま渡す → Compiler が変化を検出できない
<TableHeader table={table} />

// Good: 派生値を渡す → 列変更時に新しい配列参照が生まれ Compiler が再レンダリングできる
<TableHeader headerGroups={table.getHeaderGroups()} />

// Bad: ステート値をオブジェクト経由で読み取る
export function Panel({ table }) {
  const { columnVisibility } = table.getState(); // stale になる
}

// Good: tracked な値を親から明示的に props に渡す
<Panel table={table} columnVisibility={columnVisibility} columnOrder={columnOrder} />
```

**アンチパターン**:
- 非互換ライブラリの確認なしに `compilationMode: "all"` で全ファイルを一括最適化する
- Compiler 非互換ライブラリを使うファイルに `"use memo";` を付けてビルドエラーを発生させる
- `oxlint-disable` をファイル単位で無効化して非互換を長期放置する
- Interior Mutability 問題に対して `"use no memo"` で回避する → 最適化を全オフにするだけで根本原因（不安定な参照渡し）が残る

**出典引用**:
> "全ファイル一括は早々に諦め、**`compilationMode: "annotation"`**（annotationモード）でファイル単位のオプトインに切り替えることにしました。"
> ([React CompilerをannotationモードとOxlintで漸進的に導入する](https://zenn.dev/dress_code/articles/92dfb9206f50f3), セクション "annotationモードの採用") ※2026-05-20に実際にfetch成功

> "新しいコードにおいてはメモ化をコンパイラに任せ、詳細な制御が必要な場合にのみ使用することが推奨されています"
> ([【React Compiler v1.0】useMemo・useCallback はもう書かなくていい？導入手順と実測結果まで解説](https://qiita.com/yuuue/items/c8a395c17ea3ccd995ec), セクション "これからの useMemo・useCallback との付き合い方") ※2026-06-17に実際にfetch成功

**追加出典**:
- [React Compiler 1.0 Is Here — And It Changes Everything You Know About Performance](https://medium.com/@Ardhendu_init_/react-compiler-1-0-is-here-and-it-changes-everything-you-know-about-performance-59a0f5554a36) (Medium、v1.0 stable と新規プロジェクト setup) ※2026-06-17 fetch
- [React Compiler Broke Our Tables, And "use no memo" Is Not the Fix](https://medium.com/@kshahbaghi/react-compiler-broke-our-tables-and-use-no-memo-is-not-the-fix-f8d849f6eb79) (Medium kshahbaghi、Interior Mutability 問題・派生値を渡すパターン・TanStack Table / react-hook-form 事例) ※2026-06-23 fetch

> "if a library returns a mutable object with a stable reference, do not pass that object as a prop to a child component"
> ([React Compiler Broke Our Tables, And "use no memo" Is Not the Fix](https://medium.com/@kshahbaghi/react-compiler-broke-our-tables-and-use-no-memo-is-not-the-fix-f8d849f6eb79), セクション "Key Rule") ※2026-06-23に実際にfetch成功

**バージョン**: React 19+, React Compiler 1.0 (stable), Oxlint 0.x
**確信度**: 高
**最終更新**: 2026-06-23

---
