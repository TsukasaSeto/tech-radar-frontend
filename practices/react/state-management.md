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

**出典**:
- [TanStack Query: Overview](https://tanstack.com/query/latest/docs/framework/react/overview) (TanStack公式)

**バージョン**: React 18+, TanStack Query v5+
**確信度**: 高
**最終更新**: 2026-05-05

#### 追加根拠 (2026-05-06)

新たに以下の記事/ドキュメントで同じプラクティスが推奨された:
- [Zustandで始める「一番シンプルな」React状態管理](https://zenn.dev/yasuhikasa/articles/48d93374b240db) (Zenn / 2026-05-06) ※2026-05-06に実際にfetch成功
- [【SWR】SWRの使い方を超絶シンプルに整理する](https://zenn.dev/yuitonn/articles/5018829bfc6205) (Zenn / 2026-05-04) ※2026-05-06に実際にfetch成功

サーバー状態（データフェッチ）には SWR、クライアント状態（UIのグローバル状態）には Zustand という分離パターンがコミュニティで実践されていることが確認された。
Zustand は `create()` 一行でストアを定義し、Provider 不要の軽量設計でクライアントUI状態を管理する。
SWR は `useSWR()` 一行でサーバーデータの取得・キャッシュ・再検証を全て控え、`useState` + `useEffect` による手動実装を不要にする。
セレクターパターン `useStore((state) => state.count)` により、読んでいる値のみに反応して再レンダリングするためパフォーマンスの最適化も容易。

**確信度**: 既存（高）→ 高（コミュニティ実証付き）
