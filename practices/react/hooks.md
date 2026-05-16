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

### 3. `useEffect` の依存配列は正確に記述する

ESLint の `exhaustive-deps` ルールに従い、`useEffect` 内で使用する全ての値を
依存配列に含める。依存配列を意図的に省略してエフェクトを無効化しない。

**根拠**:
- 依存配列が不正確だと古い値を参照する（Stale Closure）
- 意図的な省略はバグの原因になる
- `useEffectEvent`（React 19+）でエフェクトのイベントハンドラー部分を分離できる

**コード例**:
```tsx
// Bad: 依存配列に count が抜けている
useEffect(() => {
  document.title = `Count: ${count}`;
}, []); // count が変わっても更新されない

// Good: 依存配列に全て含める
useEffect(() => {
  document.title = `Count: ${count}`;
}, [count]);

// Bad: 無限ループ（毎回新しいオブジェクトが deps に入る）
useEffect(() => {
  fetchUser(options);
}, [{ userId, token }]); // 毎回新しいオブジェクト

// Good: プリミティブ値を依存配列に
useEffect(() => {
  fetchUser({ userId, token });
}, [userId, token]);
```

**Stale Closure 検出パターン**:

「依存配列を省略すれば古い値を参照する」のはバグ。意図的に「最新の値を読みたいが、それで再実行はしたくない」という要件は、以下のパターンで分離する。

```tsx
// 問題: 1秒ごとに count を console.log したいが、interval は1回だけ張りたい
// Bad: count を deps に入れると interval が毎回張り直されて重い
useEffect(() => {
  const id = setInterval(() => {
    console.log(count); // ← deps に count が必要だが、interval は再設定したくない
  }, 1000);
  return () => clearInterval(id);
}, [count]);

// Bad: deps を空にすると stale closure（count が常に初期値）
useEffect(() => {
  const id = setInterval(() => {
    console.log(count); // 常に初期値の 0 が出る
  }, 1000);
  return () => clearInterval(id);
}, []);

// Good (React 18 まで): useRef で最新値を保持
function useLatest<T>(value: T) {
  const ref = useRef(value);
  ref.current = value;       // レンダリングごとに最新値に更新
  return ref;
}

function Example({ count }: { count: number }) {
  const latestCount = useLatest(count);
  useEffect(() => {
    const id = setInterval(() => {
      console.log(latestCount.current); // 常に最新値
    }, 1000);
    return () => clearInterval(id);
  }, []); // interval は1回だけ張る
}

// Good (React 19+): useEffectEvent で「非反応的なロジック」を分離
import { useEffectEvent } from 'react';

function Example({ count }: { count: number }) {
  const logCount = useEffectEvent(() => {
    console.log(count); // 常に最新の count を読む（deps に含めなくて良い）
  });

  useEffect(() => {
    const id = setInterval(logCount, 1000);
    return () => clearInterval(id);
  }, []); // logCount は deps 不要
}
```

**Object / Array deps の対処**:

```tsx
// Bad: オブジェクトリテラルを deps に入れる（参照が毎回変わる）
useEffect(() => {
  fetcher.get('/users', { filter });
}, [{ filter }]); // 毎回新規オブジェクト → 無限ループ

// Bad: 親から渡された配列・オブジェクトを deps に（親の再レンダリングで参照変化）
function List({ items }: { items: Item[] }) {
  useEffect(() => {
    saveItems(items);
  }, [items]); // items の中身が同じでも参照が変われば再実行
}

// Good: プリミティブに展開する
useEffect(() => {
  fetcher.get('/users', { filter });
}, [filter]);

// Good: 中身で比較したい場合は useMemo / use-deep-compare-effect
const itemsKey = useMemo(() => items.map(i => i.id).join(','), [items]);
useEffect(() => {
  saveItems(items);
}, [itemsKey]);

// Good: そもそも Effect ではなくイベントハンドラーで実行する（推奨）
// 「items が変わったら保存」ではなく「ユーザー操作時に保存」が本来のセマンティクス
```

**React Compiler 移行への影響**:

- React Compiler（React 19 で beta）は自動的に memoization を行うため、`useMemo` / `useCallback` の手動使用が大幅に減る
- ただし **`useEffect` の依存配列は引き続き正確に記述する必要がある**。Compiler は副作用の依存関係を推論しない
- 移行前後で `exhaustive-deps` ルールを違反していたコードは、Compiler 後に動作差分が出る可能性がある（参照同一性が変わるため）。移行前に lint エラーをゼロにしておく

**チェックリスト（Code Review）**:
- [ ] `// eslint-disable-next-line react-hooks/exhaustive-deps` がない（あれば理由をコメント必須）
- [ ] deps にオブジェクト・配列リテラルが直接入っていない
- [ ] 「最新値を読みたいが再実行はしたくない」要件は `useEffectEvent` または `useRef` で分離
- [ ] Effect 内で setState する場合、無限ループにならないか確認
- [ ] そもそも `useEffect` が必要か（[Rule 5: イベントハンドラーに置く](#5-ユーザー操作に起因する処理は-useeffect-ではなくイベントハンドラーに置く) と [Rule 1: useEffect でデータ取得しない](#1-useeffect-でデータ取得しない-react-の公式立場) を再確認）

**出典**:
- [React Docs: Specifying Reactive Dependencies](https://react.dev/learn/lifecycle-of-reactive-effects#react-verifies-that-you-specified-every-reactive-value-as-a-dependency) (React公式)
- [React Docs: Separating Events from Effects](https://react.dev/learn/separating-events-from-effects) (React公式)
- [React Docs: useEffectEvent](https://react.dev/reference/react/experimental_useEffectEvent) (React公式)

**バージョン**: React 18+ / `useEffectEvent` は React 19+
**確信度**: 高
**最終更新**: 2026-05-05 / 補強 2026-05-16

---

### 4. カスタムフックは `use` プレフィックスを必ず付ける

カスタムフックの命名は必ず `use` で始める。
React はこの命名規則を元に、フックのルール（コンポーネント外での呼び出し禁止など）を強制する。

**根拠**:
- ESLint の `react-hooks/rules-of-hooks` ルールが `use` プレフィックスで判定する
- 呼び出し側がフックであることを視覚的に識別できる
- React DevTools がカスタムフックを適切に表示する

**コード例**:
```tsx
// Bad: フックのルールが適用されない
function fetchUserData(userId: string) {
  const [data, setData] = useState(null);
  // ...
}

// Good: use プレフィックスで命名
function useUserData(userId: string) {
  const { data } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });
  return data;
}
```

**出典**:
- [React Docs: Custom Hooks - Hook names always start with use](https://react.dev/learn/reusing-logic-with-custom-hooks#hook-names-always-start-with-use) (React公式)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 5. ユーザー操作に起因する処理は `useEffect` ではなくイベントハンドラーに置く

「ユーザーがボタンをクリックした」「フォームを送信した」など、特定のインタラクションに
応じた副作用（APIコール・通知表示など）は `useEffect` ではなくイベントハンドラーに記述する。
`useEffect` は「コンポーネントが表示されたことに同期する処理」に限定する。

**根拠**:
- イベントハンドラー内ではなぜ処理が発生したかが明確（ユーザー操作）
- `useEffect` に置くと誰が副作用を引き起こしたか不明になりデバッグが困難
- `useEffect` のチェーンは不要な中間レンダリングを生み、パフォーマンスを悪化させる

**コード例**:
```tsx
// Bad: ユーザー操作を useEffect で処理
function Form() {
  const [submitted, setSubmitted] = useState(false);
  useEffect(() => {
    if (submitted) {
      postForm(data);       // なぜ実行されたか不明
      showNotification();   // 条件を追うのが困難
    }
  }, [submitted]);

  return <button onClick={() => setSubmitted(true)}>送信</button>;
}

// Good: イベントハンドラーで直接処理
function Form() {
  const handleSubmit = async () => {
    await postForm(data);   // クリックが原因であることが明確
    showNotification();
  };
  return <button onClick={handleSubmit}>送信</button>;
}
```

**出典**:
- [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) (React公式 / 2024)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 6. `useEffectEvent` でエフェクトの「非反応的なロジック」を分離する（React 19+）

エフェクト内で「再実行のトリガーにしたくない値」を参照する場合、
`useEffectEvent` でその部分を切り出す。依存配列を意図的に省略する代わりに使うことで、
Stale Closure を回避しつつ再実行条件を正確に制御できる。

**根拠**:
- `eslint-plugin-react-hooks` の `exhaustive-deps` に準拠しながら意図を明確化できる
- テーマ・設定値など「最新値を読むが変更で再実行したくない」ケースに対応
- `useEffectEvent` はエフェクト内からのみ呼び出し可能で、他のHookへの引き渡しは禁止

**コード例**:
```tsx
import { useEffect, useEffectEvent } from 'react';

function ChatRoom({ roomId, theme }: { roomId: string; theme: string }) {
  // theme が変わっても接続を張り直したくない
  const onConnected = useEffectEvent(() => {
    showNotification(`${roomId} に接続しました`, theme); // 最新の theme を参照
  });

  useEffect(() => {
    const connection = createConnection(roomId);
    connection.on('connected', onConnected); // EffectEvent は deps に不要
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]); // roomId のみが再実行トリガー
}

// Bad: theme も deps に入れると theme 変更で再接続が走る
useEffect(() => {
  const connection = createConnection(roomId);
  connection.on('connected', () => showNotification(roomId, theme));
  connection.connect();
  return () => connection.disconnect();
}, [roomId, theme]); // theme 変更で不要な再接続
```

**出典**:
- [Separating Events from Effects](https://react.dev/learn/separating-events-from-effects) (React公式 / 2024)

**バージョン**: React 19+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 7. `key` プロパティでコンポーネントの state を完全リセットする

props の変化に応じてコンポーネントの state 全体をリセットしたい場合、
`useEffect` + `setState` の組み合わせではなく `key` プロパティを使う。
`key` が変わると React はコンポーネントをアンマウント→マウントし直し、state が完全にクリアされる。

**根拠**:
- `useEffect` でのリセットは「古い state でレンダリングされた後にリセットされる」という二重レンダリングを引き起こす
- `key` を使えばレンダリング1回で正しい初期 state から開始できる
- React公式が "resetting all state with a key" として明示推奨している（You Might Not Need an Effect より）

**コード例**:
```tsx
// Bad: useEffect で state をリセット（二重レンダリングが起きる）
function ProfileEditor({ userId }: { userId: string }) {
  const [name, setName] = useState('');
  useEffect(() => {
    setName(''); // userId が変わるたびにリセット（古い値で1回レンダリングされる）
  }, [userId]);
  return <input value={name} onChange={e => setName(e.target.value)} />;
}

// Good: key で完全リセット（マウント時に初期 state から始まる）
function ProfileEditorPage({ userId }: { userId: string }) {
  return <ProfileEditor key={userId} />;
}

function ProfileEditor() {
  const [name, setName] = useState(''); // userId が変わると完全に初期化される
  return <input value={name} onChange={e => setName(e.target.value)} />;
}

// 実用例: タブ切り替えでフォームをリセット
function TabPanel({ activeTab }: { activeTab: string }) {
  return <TabContent key={activeTab} tab={activeTab} />;
}
```

**出典**:
- [You Might Not Need an Effect: Resetting all state when a prop changes](https://raw.githubusercontent.com/reactjs/react.dev/main/src/content/learn/you-might-not-need-an-effect.md) (reactjs/react.dev / mainブランチ) ※2026-05-06に実際にfetch成功

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-06

---

#### 追加根拠 (2026-05-06) — ルール1「useEffect でデータ取得しない」

新たに以下の記事/ドキュメントで同じプラクティスが推奨された:
- [Synchronizing with Effects](https://raw.githubusercontent.com/reactjs/react.dev/main/src/content/learn/synchronizing-with-effects.md) (reactjs/react.dev / mainブランチ) ※2026-05-06に実際にfetch成功

同ドキュメントは「Effects are an escape hatch from the React paradigm」と明記し、Effects は React の外部システム（ブラウザ API、サードパーティウィジェット、ネットワーク接続）との同期にのみ使うべきと定義している。特にデータフェッチは「Effects alone create architectural problems: waterfalls, missing caches, and complex cleanup requirements」として問題視されており、フレームワーク（Next.js Server Components）・SWR・TanStack Query などの専用ツールへの委譲を強く推奨している。

**確信度**: 既存（高）→ 高（公式文書で実証済み）

---

#### 追加根拠 (2026-05-06) — ルール5「ユーザー操作はイベントハンドラーに置く」

新たに以下の記事/ドキュメントで同じプラクティスが推奨された:
- [You Might Not Need an Effect](https://raw.githubusercontent.com/reactjs/react.dev/main/src/content/learn/you-might-not-need-an-effect.md) (reactjs/react.dev / mainブランチ) ※2026-05-06に実際にfetch成功

公式ドキュメントは「Effects are caused by rendering itself, not by a particular event」と明確に定義し、ユーザーのボタンクリック・フォーム送信などのインタラクションに伴う副作用（通知表示・APIコール）は必ずイベントハンドラーに置くべきと述べている。Effects チェーン（state 変化 → Effect → state 変化 → Effect...）は中間レンダリングが増え、デバッグが困難になるため避けるべきパターンとして明示されている。

**確信度**: 既存（高）→ 高（公式文書で実証済み）
