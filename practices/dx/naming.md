# 命名規約のベストプラクティス

命名はコードの読解負荷を最も大きく左右する。チーム内で揃えれば、レビュー・debug・grep・ファイル探索の速度が直接向上する。

## ルール

### 1. コンポーネントは PascalCase、Props 型は `ComponentNameProps`

React コンポーネントは PascalCase で命名し、Props 型は `<ComponentName>Props` で統一する。
ファイル名もコンポーネント名と一致させ、`index.tsx` のような曖昧な名前を避ける。

**根拠**:
- React は識別子の大文字/小文字でコンポーネント / HTML 要素を区別する（`<button>` vs `<Button>`）
- ファイル名がコンポーネント名と一致するとファイル検索（VS Code の Cmd+P）で見つかりやすい
- `<Name>Props` のサフィックスで「これは Props 型」と一目で分かる
- 業界標準（React 公式 docs / 主要 OSS）と揃える

**コード例**:
```tsx
// Good
// UserAvatar.tsx
export type UserAvatarProps = {
  user: User;
  size?: 'sm' | 'md' | 'lg';
  showOnlineStatus?: boolean;
};

export function UserAvatar({ user, size = 'md', showOnlineStatus = false }: UserAvatarProps) {
  return <img src={user.avatarUrl} alt={user.name} className={`size-${size}`} />;
}

// 使う側
import { UserAvatar, type UserAvatarProps } from '@/components/UserAvatar';
```

**避けるべき**:
```tsx
// Bad: 小文字始まり
function userAvatar() { ... }  // ← React が HTML 要素として解釈

// Bad: index.tsx だらけ（ファイル検索でヒットしない）
// components/UserAvatar/index.tsx
// components/Header/index.tsx
// components/Footer/index.tsx

// Bad: Props 型の命名が一貫しない
type UserAvatarType = { ... }
type IUserAvatar = { ... }    // ハンガリアン記法
type Props = { ... }           // 曖昧
```

**`index.tsx` を使う場合の許容パターン**:
- フォルダで named export を集約する目的（`components/ui/index.ts` で再エクスポート）
- 単一コンポーネントのフォルダで `Card/index.tsx` + `Card/styles.css` + `Card/Card.test.tsx` の構成

ただし、開発者の検索体験を最優先するなら **named ファイル**（`Card.tsx`）を推奨。

**ファイル / フォルダ命名**:
```
# Good: ファイル名 = コンポーネント名
components/
├── UserAvatar.tsx
├── UserAvatar.test.tsx
└── UserAvatar.module.css

# Good: フォルダ構造の場合もファイル名を明示
components/UserAvatar/
├── UserAvatar.tsx
├── UserAvatar.test.tsx
└── UserAvatar.module.css

# Bad: index.tsx しか入っていないフォルダ
components/UserAvatar/
└── index.tsx    # ← VS Code で 100 個の index.tsx が並ぶ
```

**`'use client'` ファイルの命名**:
- 機能的に分ける必要はないが、`use-` プレフィックスや `.client.tsx` サフィックスで明示するチームもある
- 個人的には不要だが、チーム合意があれば従う

**出典**:
- [React Docs: Your First Component](https://react.dev/learn/your-first-component) (React 公式)
- [Airbnb React/JSX Style Guide](https://github.com/airbnb/javascript/tree/master/react) (Airbnb)

**バージョン**: パターン
**確信度**: 高
**最終更新**: 2026-05-16

---

### 2. カスタムフックは `use` プレフィックス + ドメイン名（`useAuth` / `useCartCount`）

React のカスタムフックは必ず `use` から始める。
ドメイン用語を含めて `useUser` ではなく `useAuthUser` / `useProfile` のように具体化する。

**根拠**:
- React の Lint ルール（`react-hooks/rules-of-hooks`）は `use` プレフィックスで判定する
- プレフィックスがないと「これは hook かどうか」を読み手が判断できない
- `use*` だけだと汎用すぎてコード再利用性が落ちる（`useUser` が複数あると混乱）
- ドメイン用語を入れることでファイル検索・grep が容易

**コード例**:
```tsx
// Good: ドメイン文脈を含む
function useAuthUser() { ... }              // 認証ユーザー
function useShoppingCart() { ... }          // ショッピングカート
function useOrderHistory() { ... }          // 注文履歴
function useFeatureFlag(name: string) { ... }
function useDebounce<T>(value: T, ms: number) { ... }  // 汎用ユーティリティはこの形で OK

// Bad: 汎用すぎ
function useData() { ... }                   // 何のデータ？
function useState2() { ... }                  // useState と紛らわしい
function getUser() { ... }                    // use プレフィックスなし → Lint エラー
function useUserAndProfileAndPreferences() { ... } // 1 hook の責任が広すぎ
```

**命名のヒント**:
- 「何を返すか」を hook 名で表現する
- 動詞より名詞ベース（`getUser` ではなく `useUser`）
- 副作用が強い hook はそれを示す（`useAutoSave` / `useTrackPageView`）

**hook の責任分割の指針**:
| Hook 名 | 戻り値の型 | 責任 |
|---|---|---|
| `useAuthUser()` | `User \| null` | 認証されたユーザー情報 |
| `useAuthActions()` | `{ login, logout, signup }` | 認証アクション |
| `useAuth()` | 上記両方をまとめる | 簡易インターフェース |

両方提供するか、分離するかはチームで決める。

**hook の戻り値の型推奨**:
```tsx
// Good: オブジェクトで返す（順序依存を避ける）
function useShoppingCart() {
  return { items, addItem, removeItem, total, isEmpty };
}

const { items, addItem } = useShoppingCart();

// Acceptable: 順序依存だが、ペアが明確な場合
function useToggle(initial = false): [boolean, () => void] {
  const [value, setValue] = useState(initial);
  return [value, useCallback(() => setValue(v => !v), [])];
}

// Bad: 3 つ以上の値を tuple で返す → 順序の覚え方が混乱
function useFormState(): [Form, SetForm, Validate, Reset] { ... }
```

**`useEffectEvent` / `useDeferredValue` のような React 19+ hook を踏襲**:
- React の組み込み hook と命名が衝突しないよう、独自 hook 名は **動詞 / 副詞 + 名詞** を意識する
  - `useDebouncedValue` ○ vs `useDeferred` × （`useDeferredValue` と紛らわしい）
  - `usePrevious` ○

**出典**:
- [React Docs: Reusing Logic with Custom Hooks](https://react.dev/learn/reusing-logic-with-custom-hooks) (React 公式)
- [react-hooks/rules-of-hooks](https://www.npmjs.com/package/eslint-plugin-react-hooks) (Meta)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 3. boolean は `is*` / `has*` / `can*` / `should*` で命名する

boolean 型の変数・関数戻り値は接頭辞でその意味を明示する。
`active`（形容詞単独）や `flag` のような汎用名を避ける。

**根拠**:
- if 文の可読性が向上する（`if (user.isAdmin)` vs `if (user.admin)`）
- IDE の補完で同じ接頭辞のフラグが一覧できる
- 反転条件も明確（`!isLoading` vs `!loading`）
- TypeScript の型ガードでも同じ規約が活きる（`function isAdmin(u): u is Admin`）

**プレフィックス一覧**:
| プレフィックス | 用途 | 例 |
|---|---|---|
| `is` | 状態・属性 | `isLoading`, `isAdmin`, `isOpen` |
| `has` | 所有・存在 | `hasError`, `hasPermission`, `hasChildren` |
| `can` | 権限・能力 | `canEdit`, `canDelete`, `canPublish` |
| `should` | 推奨・期待 | `shouldRetry`, `shouldSkipValidation` |
| `did` / `was` | 過去の状態 | `didMount`, `wasModified` |
| `will` | 未来の予定 | `willUnmount`, `willExpire` |

**コード例**:
```tsx
// Good
type UserState = {
  isLoading: boolean;
  hasUnreadMessages: boolean;
  canEditProfile: boolean;
  shouldShowOnboarding: boolean;
};

// Bad
type UserState = {
  loading: boolean;        // ← 動詞 / 形容詞？
  unreadMessages: boolean;  // ← number に見える
  edit: boolean;            // ← 動詞か boolean か曖昧
  onboarding: boolean;      // ← 何の状態？
};

// 否定形は避ける（二重否定の温床）
// Bad: const isNotLoaded = !loaded;
// → if (!isNotLoaded) { ... } で混乱

// Good: ポジティブな名前
const isLoaded = loaded;
```

**React props での typical な命名**:
```tsx
// Good
<Button disabled />
<Modal isOpen />
<Input required readOnly />
<Form noValidate />

// Convention: HTML 標準属性は HTML の命名に合わせる
// disabled / readonly / required / hidden / checked / selected
// → これらはそのまま使う（is プレフィックスを付けない）

// Custom Props は is プレフィックス
type Props = {
  isCompact?: boolean;       // カスタム
  hasError?: boolean;        // カスタム
  disabled?: boolean;        // HTML 標準
};
```

**TypeScript 型ガード**:
```ts
// 型述語（type predicate）も同じ規約
function isAdmin(user: User): user is AdminUser {
  return user.role === 'admin';
}

function hasPermission(user: User, permission: string): boolean {
  return user.permissions.includes(permission);
}

// 使用
if (isAdmin(user)) {
  user.adminOnlyMethod();  // 型が narrowing される
}
```

**関数戻り値**:
```ts
// Good
function canUserEdit(user: User, resource: Resource): boolean { ... }
function hasValidSession(): boolean { ... }
function shouldRefetch(query: Query): boolean { ... }

// Bad
function userEdit(user, resource): boolean { ... }      // 動詞？プロパティ？
function valid(): boolean { ... }                         // 何の検証？
```

**出典**:
- [Clean Code (Robert C. Martin)](https://www.oreilly.com/library/view/clean-code-a/9780136083238/) (Prentice Hall)
- [Airbnb JavaScript Style Guide](https://github.com/airbnb/javascript#naming-conventions) (Airbnb)

**バージョン**: パターン
**確信度**: 高
**最終更新**: 2026-05-16

---

### 4. イベントハンドラーは `on*`（prop）/ `handle*`（内部関数）で区別する

React コンポーネントで「ハンドラーを受け取る props」と「内部で定義するハンドラー」を `on*` / `handle*` で命名分離する。

**根拠**:
- `onClick={handleClick}` のように外向き API と内部実装で命名を分けると役割が明確
- 業界デフォルトのパターン（React 公式 docs も同じ規約）
- `onClick={onClick}` のように同じ名前だと混乱する
- 「これは prop か、コンポーネント内部関数か」を一目で判定できる

**コード例**:
```tsx
// Good
type ButtonProps = {
  label: string;
  onClick?: (event: React.MouseEvent<HTMLButtonElement>) => void;
  onHover?: () => void;
};

function Button({ label, onClick, onHover }: ButtonProps) {
  // 内部のハンドラーは handle*
  const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
    console.log('clicked');
    onClick?.(e);  // prop に flow through
  };

  return (
    <button onClick={handleClick} onMouseEnter={onHover}>
      {label}
    </button>
  );
}

// 使う側
<Button label="送信" onClick={() => console.log('submit')} />
```

**ハンドラー名のイベント部分**:
- `onClick` / `onChange` / `onSubmit` — DOM イベントに対応
- `onClose` / `onOpen` / `onSelect` — 抽象的なコンポーネントイベント
- `onSuccess` / `onError` — 非同期処理の結果

**よくある間違い**:
```tsx
// Bad: prop 名と内部関数が同じで紛らわしい
function Modal({ onClose }) {
  const onClose = () => { ... };  // 同名で上書きしている
  return <button onClick={onClose}>...</button>;
}

// Bad: handle / on の使い分けが逆
type Props = {
  handleClick: () => void;  // ← prop なら on*
};
function Button({ handleClick }: Props) {
  const onClick = () => handleClick();  // ← 内部なら handle*
  return <button onClick={onClick}>...</button>;
}
```

**コンテキスト依存の命名**:
```tsx
// フォームコンポーネント
type FormProps = {
  onSubmit: (data: FormData) => void;       // submit を受け取る
  onChange?: (field: string, value: string) => void;  // 変更通知
  onValidationError?: (errors: string[]) => void;
};

// データ取得コンポーネント
type FetcherProps = {
  onSuccess: (data: Data) => void;
  onError: (error: Error) => void;
  onSettled?: () => void;
};

// 関数のセマンティクス + イベント名
type CartProps = {
  onAddItem: (item: Item) => void;          // アクション
  onRemoveItem: (itemId: string) => void;
  onCheckout: () => void;
};
```

**`async` イベントハンドラー**:
```tsx
// Good: async ハンドラーは即時 awaitable な値を返さない
const handleSubmit = async (e: React.FormEvent) => {
  e.preventDefault();
  try {
    await api.save(data);
  } catch (err) {
    setError(err.message);
  }
};

// async-await を JSX 内で書かない（戻り値が unfulfilled promise になる）
<form onSubmit={handleSubmit}>...</form>
```

**チームでの徹底**:
ESLint rule で強制可能:
```js
// .eslintrc.js
'naming-convention': ['error', {
  selector: 'parameter',
  filter: { regex: '^(handle|on)[A-Z].*$', match: false },
  format: ['camelCase'],
}]
```

ただし naming-convention は厳しすぎると開発体験を損なうので、PR レビューで指摘する程度の運用が現実的。

**出典**:
- [React Docs: Responding to Events](https://react.dev/learn/responding-to-events#naming-event-handler-props) (React 公式)
- [Airbnb React Style Guide: Naming](https://github.com/airbnb/javascript/tree/master/react#naming) (Airbnb)

**バージョン**: React 18+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 5. 型・enum・定数は PascalCase / SCREAMING_SNAKE_CASE で命名する

`type` / `interface` / `enum` は PascalCase、定数（モジュールレベル）は SCREAMING_SNAKE_CASE で書く。
小文字定数や camelCase 型は読み手の認知負荷を上げる。

**根拠**:
- 型と値の区別が PascalCase / 小文字始まりで判別可能（業界標準）
- 定数の SCREAMING_SNAKE_CASE は「再代入されない」「アプリ全体で参照される」の signal
- TypeScript の `as const` で literal type を定数化する際にも有効
- ESLint の `naming-convention` でルール化できる

**コード例**:
```ts
// Good: 型 = PascalCase
type User = { name: string; email: string };
type UserId = string;
interface ApiResponse<T> { data: T; error: null }
enum Status { Active, Inactive, Banned }

// Good: 関数 / 変数 = camelCase
const currentUser = await fetchUser(id);
function calculateTotal(items: Item[]): number { ... }

// Good: モジュール定数 = SCREAMING_SNAKE_CASE
const MAX_ITEMS = 100;
const API_BASE_URL = 'https://api.example.com';
const RATE_LIMIT_MS = 60 * 1000;

// Good: enum-like なオブジェクト
const Status = {
  ACTIVE: 'active',
  INACTIVE: 'inactive',
  BANNED: 'banned',
} as const;
type Status = typeof Status[keyof typeof Status];

// Bad
type user = { name: string };       // 型は PascalCase
interface IUserService { ... }       // ハンガリアン (I) は TS では不要
const maxItems = 100;                // モジュール定数は SCREAMING
const API_URL = computeBaseUrl();    // 動的に決まる値は const 命名でなく camelCase
```

**enum vs `as const` の選択**:
```ts
// 推奨: as const + union type
const Role = {
  ADMIN: 'admin',
  USER: 'user',
  GUEST: 'guest',
} as const;
type Role = typeof Role[keyof typeof Role];

// 理由:
// - Tree-shake friendly（enum はランタイムオブジェクトを生成、as const は型のみ）
// - 文字列リテラル型として扱える
// - JSON にそのまま乗る
// - TypeScript 5.0+ では enum も改善されているが、as const のほうがコード生成が軽量

// 例外: enum を使ってよいケース
// - 古い TS バージョンとの互換性
// - 既に enum で統一されたコードベース
// - reverse mapping が必要（数値 → 名前）
```

**ローカル変数の定数**:
```ts
function foo() {
  const MAX_RETRIES = 3;  // 関数内でも全大文字でよい
  // ↑ 「local だが意味的な定数」を強調できる

  // ただし「ただの初期値」は camelCase で十分
  const initialValue = 0;
}
```

**型エイリアス vs Interface**:
```ts
// type を基本とし、宣言マージが必要なら interface（typescript/utility-types.md Rule 1 参照）
type ButtonProps = { ... };       // 推奨デフォルト
interface Window { customProp: string; }  // declaration merging
```

**ESLint で強制**:
```js
// .eslintrc.js
'@typescript-eslint/naming-convention': [
  'error',
  {
    selector: 'typeAlias',
    format: ['PascalCase'],
  },
  {
    selector: 'interface',
    format: ['PascalCase'],
    custom: { regex: '^I[A-Z]', match: false },  // I プレフィックス禁止
  },
  {
    selector: 'enumMember',
    format: ['UPPER_CASE'],
  },
  {
    selector: 'variable',
    modifiers: ['const', 'global'],
    format: ['UPPER_CASE', 'camelCase'],  // 両方許容
  },
]
```

**ファイル名規約との関係**:
- コンポーネントファイル: `UserAvatar.tsx`（PascalCase）
- ユーティリティ・hook ファイル: `formatDate.ts` / `useAuth.ts`（camelCase）
- 定数モジュール: `constants.ts` または `apiEndpoints.ts`（camelCase）

**出典**:
- [TypeScript Handbook: Type Aliases](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#type-aliases) (TypeScript 公式)
- [Google TypeScript Style Guide](https://google.github.io/styleguide/tsguide.html#naming-style) (Google)
- [Airbnb JavaScript Style Guide: Naming](https://github.com/airbnb/javascript#naming-conventions) (Airbnb)

**バージョン**: TypeScript 5+
**確信度**: 高
**最終更新**: 2026-05-16
