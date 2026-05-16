# 状態管理のベストプラクティス

## ルール

### 1. 状態は最も近い共通の親コンポーネントに置く（State Colocation）

状態はそれを必要とするコンポーネントのできるだけ近くに置く。
グローバル状態にする前に、ローカル状態で解決できないか検討する。

**根拠**:
- 不要なグローバル状態は再レンダリングの範囲を広げる
- ローカル状態は関連するコンポーネントが削除されれば自動的にクリーンアップされる
- テストが容易になる（プロバイダーのセットアップが不要）

**コード例**:
```tsx
// Bad: 不必要にグローバル状態に
const [isModalOpen, setIsModalOpen] = useAtom(modalOpenAtom);

// Good: ローカル状態で十分
function ParentComponent() {
  const [isModalOpen, setIsModalOpen] = useState(false);
  return (
    <>
      <Button onClick={() => setIsModalOpen(true)}>Open</Button>
      <Modal open={isModalOpen} onClose={() => setIsModalOpen(false)} />
    </>
  );
}
```

**出典**:
- [React Docs: Sharing State Between Components](https://react.dev/learn/sharing-state-between-components) (React公式)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. 複数の関連する state は `useReducer` でまとめる

3つ以上の state が連動して変化する場合、`useReducer` を使って状態遷移を一元管理する。

**根拠**:
- 複数の `setState` 呼び出しを一つのアクションにまとめられる
- 状態遷移ロジックをコンポーネント外でテストできる
- Discriminated Union で型安全に管理できる

**コード例**:
```tsx
type FormState =
  | { status: 'idle'; name: string; email: string }
  | { status: 'submitting'; name: string; email: string }
  | { status: 'error'; name: string; email: string; error: Error };

type FormAction =
  | { type: 'SET_FIELD'; field: 'name' | 'email'; value: string }
  | { type: 'SUBMIT' }
  | { type: 'SUBMIT_SUCCESS' }
  | { type: 'SUBMIT_ERROR'; error: Error };

function formReducer(state: FormState, action: FormAction): FormState {
  switch (action.type) {
    case 'SET_FIELD':
      return { ...state, [action.field]: action.value };
    case 'SUBMIT':
      return { ...state, status: 'submitting' };
    case 'SUBMIT_ERROR':
      return { ...state, status: 'error', error: action.error };
    default:
      return state;
  }
}
```

**出典**:
- [React Docs: Extracting State Logic into a Reducer](https://react.dev/learn/extracting-state-logic-into-a-reducer) (React公式)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. URL を状態として使う（URL State）

フィルター・ページネーション・検索クエリなど、共有・ブックマークできるべき状態は URL に保存する。

**根拠**:
- ブラウザの戻る/進む操作と連動する
- URL をコピーして他者と共有できる
- ページリロードで状態が失われない

**コード例**:
```tsx
// app/todos/page.tsx
export default function TodosPage({
  searchParams,
}: {
  searchParams: { filter?: string; page?: string };
}) {
  const filter = searchParams.filter ?? 'all';
  const page = Number(searchParams.page ?? 1);
  return <TodoList filter={filter} page={page} />;
}
```

**出典**:
- [React Docs: You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) (React公式)

**バージョン**: React 18+, Next.js 14+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. グローバル状態はサーバー状態・クライアント状態で分離する

外部からフェッチするデータ（サーバー状態）と UI の状態（クライアント状態）は別のツールで管理する。

**根拠**:
- サーバー状態にはキャッシュ・再検証・楽観的更新などのロジックが必要
- `useState` + `useEffect` での実装は再発明になる
- TanStack Query / SWR はキャッシュ管理を自動化し、バグを大幅に減らす

**コード例**:
```tsx
// Good: TanStack Query
function UserProfile({ userId }: { userId: string }) {
  const { data: user, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  if (isLoading) return <Spinner />;
  if (error) return <ErrorView error={error} />;
  return <Profile user={user!} />;
}
```

**状態管理ツール選定マトリクス**:

「グローバル状態が必要」と決めた後、何を使うかを以下の基準で選ぶ。**先に [Rule 1](#1-状態は最も近い共通の親コンポーネントに置くstate-colocation) で本当にグローバルが必要か検討する**ことが前提。

| 用途 | 推奨ツール | 理由 |
|---|---|---|
| サーバーから取得したデータ | **TanStack Query / SWR** | キャッシュ・再検証・楽観的更新・無限ロードを自動化 |
| URL 由来の状態（フィルタ・タブ・ページ） | **`useSearchParams` / `nuqs`** | ブックマーク・共有・戻る/進むに耐える |
| 単純なフラグ・テーマ・i18n locale | **React Context** | 1〜数個の値、変更頻度が低い、Provider 1 つで完結 |
| 中規模のクライアント状態（カート・フォーム下書き等） | **Zustand** | API が薄い、Provider 不要、persistence ミドルウェアあり |
| アトミックな状態（細粒度購読） | **Jotai** | useState ライクな API、各 atom 単位で購読、依存グラフを表現 |
| 大規模・複雑な状態遷移（admin / 業務 SaaS） | **Redux Toolkit** | DevTools・時間旅行デバッグ、明確な action ログ |
| ステートマシン（複雑な UI フロー） | **XState** | 状態と遷移を視覚化、不正遷移を型で防止 |
| フォーム状態 | **React Hook Form** + **zod** | 大量フィールドでも再レンダリングを抑制、検証を型と統合 |

**判断軸の優先順位**:

1. **URL に乗せられるか？** → 乗せる（[Rule 5 の URL State 参照](../react/suspense.md)）
2. **サーバー由来か？** → TanStack Query / SWR で扱う
3. **コンポーネントツリーで共有が必要か？** → No なら `useState`
4. **変更頻度が低く、Provider 1 つで済むか？** → Context
5. **複数の独立した state スライスが必要か？** → Zustand（フラットなストア）/ Jotai（atom グラフ）
6. **アクション履歴を追跡したいか？（admin・分析等）** → Redux Toolkit

**Context の落とし穴（よくある誤用）**:

```tsx
// Bad: 1 つの Context に大きなオブジェクトを詰める → 一部の更新で全消費者が再レンダリング
const AppContext = createContext<{
  user: User | null;
  theme: Theme;
  cart: Cart;
  notifications: Notification[];
}>(/* ... */);

// Good: 関心ごとに Context を分割（or Zustand で selector 購読）
const UserContext = createContext<User | null>(null);
const ThemeContext = createContext<Theme>('light');
const CartContext = createContext<Cart>(emptyCart);

// Good: state と setState を別 Context にして「読むだけ」のコンポーネントを再レンダリングから守る
const CartStateContext = createContext<Cart>(emptyCart);
const CartDispatchContext = createContext<Dispatch<CartAction>>(() => {});
```

**Zustand を選ぶ典型ケース**:
- 「Redux ほど大袈裟ではないが、Context だと再レンダリング問題が深刻」な中規模アプリ
- カスタム hook で関心ごとに分割しやすい（`useCart` / `useNotifications` 等）
- Server Components が増えて、Client 側に残る state が「数個のスライス」程度

**Jotai を選ぶ典型ケース**:
- 状態が細かく、互いに依存している（atom A → derive atom B → derive atom C）
- フォームの各フィールドを独立 atom にして再レンダリングを最小化
- React Suspense との統合が必要

**避けるべき選定**:
- 全部 Context に詰める（再レンダリング問題と Provider 地獄）
- 全部 Redux Toolkit（小規模アプリではボイラープレートが過剰）
- グローバル状態 + `useEffect` でサーバー同期を自作（TanStack Query で置換）

**出典**:
- [TanStack Query: Overview](https://tanstack.com/query/latest/docs/framework/react/overview) (TanStack公式)
- [Zustand Docs](https://zustand.docs.pmnd.rs/) (pmndrs)
- [Jotai Docs](https://jotai.org/docs/introduction) (jotai)
- [Redux Toolkit: When to use](https://redux.js.org/faq/general#when-should-i-use-redux) (Redux 公式)
- [React Docs: Passing Data Deeply with Context](https://react.dev/learn/passing-data-deeply-with-context) (React 公式)

**バージョン**: React 18+, TanStack Query v5+, Zustand v4+, Jotai v2+
**確信度**: 高
**最終更新**: 2026-05-05 / 補強 2026-05-16

---

#### 追加根拠 (2026-05-06) — ルール1「状態は最も近い共通の親コンポーネントに置く（State Colocation）」

新たに以下の記事/ドキュメントで同じプラクティスが推奨された:
- [You Might Not Need an Effect](https://raw.githubusercontent.com/reactjs/react.dev/main/src/content/learn/you-might-not-need-an-effect.md) (reactjs/react.dev / mainブランチ) ※2026-05-06に実際にfetch成功
- [Zustandで始める「一番シンプルな」React状態管理](https://zenn.dev/yasuhikasa/articles/48d93374b240db) (Zenn / 2026-05) ※2026-05-06に実際にfetch成功

You Might Not Need an Effect では「URL を状態として持つ」「store identifiers not derived values（IDを保持し、データは導出する）」など、状態の最小化と局所化の具体例を多数示している。Zustand 記事は「必要な状態や関数だけをセレクトして取り出す（Selective State Retrieval）」を推奨しており、グローバルストアを使う場合でも「使うスライスだけを購読する」ことで不必要な再レンダリングを防ぐパターンを解説。コロケーションの原則はグローバル状態ツールを使う際にも適用できる（ストアの購読範囲を最小化する形で）。

**確信度**: 既存（高）→ 高（公式文書 + コミュニティ記事で実証済み）
