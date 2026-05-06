# 型設計のベストプラクティス

## ルール

### 1. `interface` より `type` を基本とし、拡張が必要な場合に `interface` を選ぶ

オブジェクト型の定義には `type` を基本として使用する。
ただし、外部ライブラリの型をマージ（Declaration Merging）する必要がある場合や、
クラスが実装する契約を表す場合は `interface` を使用する。
プロジェクト内で一方に統一し、混在させない。

**根拠**:
- `type` は Union 型・交差型・条件型など表現力が高い
- `interface` の Declaration Merging はサードパーティ型の拡張に有効
- 統一することで読み手の認知負荷を下げる

**コード例**:
```tsx
// Good: type で統一（コンポーネントのpropsなど）
type ButtonProps = {
  label: string;
  onClick: () => void;
  disabled?: boolean;
};

// Good: Declaration Merging が必要なケースは interface
interface Window {
  myCustomProperty: string;
}

// Bad: 同じプロジェクト内で混在
interface UserProfile {
  name: string;
}
type UserProfile = { name: string }; // 重複・混在
```

**出典**:
- [TypeScript Handbook: Differences Between Type Aliases and Interfaces](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#differences-between-type-aliases-and-interfaces) (TypeScript公式)

**バージョン**: TypeScript 4.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. Union 型で網羅的な分岐を強制する（Discriminated Union）

状態を複数の `state` で管理するのではなく、判別子（discriminant）を持つ Union 型で表現する。
`switch` 文や `if/else` でのハンドリング漏れをコンパイル時に検出できる。

**根拠**:
- 不正な状態（`loading: true` かつ `data: User`）をモデリングレベルで排除できる
- `never` チェックで網羅性を担保できる
- リファクタリング時の変更漏れをコンパイルエラーで検出

**コード例**:
```tsx
// Bad: 独立した state フラグで管理
type State = {
  isLoading: boolean;
  error: Error | null;
  data: User | null;
};
// isLoading=true かつ data=User という不正状態が型上は可能

// Good: Discriminated Union
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

// 網羅性チェック
function render(state: AsyncState<User>) {
  switch (state.status) {
    case 'idle':    return <Idle />;
    case 'loading': return <Spinner />;
    case 'success': return <Profile user={state.data} />;
    case 'error':   return <ErrorView error={state.error} />;
    default:
      const _exhaustive: never = state;
      throw new Error(`Unhandled: ${_exhaustive}`);
  }
}
```

**出典**:
- [TypeScript Handbook: Narrowing - Discriminated Unions](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#discriminated-unions) (TypeScript公式)

**バージョン**: TypeScript 4.0+
**確信度**: 高
**最終更新**: 2026-05-05

#### 追加根拠 (2026-05-06)

新たに以下の記事/ドキュメントで同じプラクティスが推奨された:
- [Aljabr: 0-deps TypeScript lib that fuses tagged unions / exhaustive matching + schema validation + reactive state + reactive UI](https://dev.to/jasuperior/aljabr-ljbr-0-deps-typescript-lib-that-fuses-tagged-unions-exhaustive-matching-schema-3pe7) (dev.to / 2026-05-06) ※2026-05-06に実際にfetch成功

Discriminated Union（識別可能なユニオン型）を「コンパイラが検証する有限状態機械（compiler-checked finite state machine）」として捉える観点がコミュニティで強調されている。
バリアント（`Circle`, `Rect` など）を明示的に定義し `match()` 関数で網羅的にパターンマッチングすることで、コンパイル時に全ケースの処理が強制される。
この設計パターンはライブラリとしても実装されており、「不可能な状態（impossible state bugs）を構造レベルで排除する」という考え方がエコシステム全体で支持されていることが確認された。

**確信度**: 既存（高）→ 高（コミュニティ実証付き）

---

### 3. `any` を禁止し、型が不明な場合は `unknown` を使う

`any` は TypeScript の型安全保証を完全に無効化する。
型が不明な外部入力（APIレスポンス、ユーザー入力など）には `unknown` を使い、
型ガードで絞り込んでから使用する。

**根拠**:
- `any` を使った変数は型エラーを伝播させず、バグを埋め込みやすい
- `unknown` は使う前に型チェックを強制するため安全
- `noImplicitAny: true` で暗黙の `any` をコンパイルエラーにできる

**コード例**:
```tsx
// Bad
function processResponse(data: any) {
  return data.user.name.toUpperCase(); // ランタイムエラーの可能性あり
}

// Good
function processResponse(data: unknown) {
  if (
    typeof data === 'object' &&
    data !== null &&
    'user' in data &&
    typeof (data as { user: unknown }).user === 'object'
  ) {
    // 型が絞り込まれた後に使用
  }
}

// zodなどのバリデーションライブラリを使う方法も推奨
import { z } from 'zod';
const UserSchema = z.object({ name: z.string() });
const user = UserSchema.parse(unknownData); // 型安全
```

**出典**:
- [TypeScript Handbook: The unknown type](https://www.typescriptlang.org/docs/handbook/2/functions.html#unknown) (TypeScript公式)

**バージョン**: TypeScript 3.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. 型に `null` / `undefined` を明示的に含めて安全に扱う

`strictNullChecks: true` を有効にし、`null` / `undefined` の可能性がある値を
必ず型に含める。オプショナルチェーン `?.` や Nullish Coalescing `??` を活用する。

**根拠**:
- `strictNullChecks` なしでは null 参照エラーが型チェックで防げない
- `?.` と `??` により冗長な null チェックを簡潔に書ける
- Non-null assertion `!` の乱用は避ける（実質 `any` と同等のリスク）

**コード例**:
```tsx
// Bad: Non-null assertion の乱用
function getUsername(user: User | null) {
  return user!.name; // ランタイムエラーの危険
}

// Good: オプショナルチェーンと Nullish Coalescing
function getUsername(user: User | null) {
  return user?.name ?? 'Anonymous';
}

// Good: 早期リターンで絞り込み
function processUser(user: User | null) {
  if (!user) return;
  // この下では user は User 型
  console.log(user.name);
}
```

**出典**:
- [TypeScript Handbook: Strictness](https://www.typescriptlang.org/docs/handbook/2/basic-types.html#strictness) (TypeScript公式)

**バージョン**: TypeScript 2.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 5. 型の幅を最小化する（最小権限の型）

変数・パラメータの型は、実際に必要な形状だけを受け付けるように絞る。
巨大なオブジェクト型全体を渡すのではなく、必要なプロパティだけ受け取る型を定義する。

**根拠**:
- テスタビリティの向上（モックの作成が容易になる）
- 関数の依存範囲が明確になり、再利用性が上がる
- 型変更の影響範囲を局所化できる

**コード例**:
```tsx
// Bad: 巨大な型全体を要求
function displayUserBadge(user: User) {
  return <Badge name={user.name} avatar={user.avatarUrl} />;
}

// Good: 必要なプロパティだけを持つ型
type UserBadgeProps = Pick<User, 'name' | 'avatarUrl'>;
function displayUserBadge(user: UserBadgeProps) {
  return <Badge name={user.name} avatar={user.avatarUrl} />;
}

// または型をインラインで定義
function displayUserBadge(user: { name: string; avatarUrl: string }) {
  return <Badge name={user.name} avatar={user.avatarUrl} />;
}
```

**出典**:
- [TypeScript Handbook: Structural Type System](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes.html#structural-type-system) (TypeScript公式)

**バージョン**: TypeScript 4.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 6. `as const` でリテラル型の配列・オブジェクトを定義する

定数として扱いたい配列やオブジェクトには `as const` を付与し、
型を widening させないようにする。

**根拠**:
- `as const` なしでは `string[]` に widening され、リテラル型の情報が失われる
- `typeof ARRAY[number]` で Union 型を自動生成できる
- enum の代替として、より TypeScript らしい定数定義が可能

**コード例**:
```tsx
// Bad: 型が string[] に widening される
const ROLES = ['admin', 'editor', 'viewer'];
type Role = typeof ROLES[number]; // string（意図と違う）

// Good: as const でリテラル型を保持
const ROLES = ['admin', 'editor', 'viewer'] as const;
type Role = typeof ROLES[number]; // 'admin' | 'editor' | 'viewer'

// オブジェクトでも同様
const ROUTES = {
  home: '/',
  users: '/users',
} as const;
type Route = typeof ROUTES[keyof typeof ROUTES]; // '/' | '/users'
```

**出典**:
- [TypeScript Handbook: Const Assertion](https://www.typescriptlang.org/docs/handbook/2/types-from-types.html#the-keyof-type-operator) (TypeScript公式)

**バージョン**: TypeScript 3.4+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 7. Template Literal Types でイベント名・CSSプロパティ名などの文字列型を精密に定義する

テンプレートリテラル型（`` `prefix_${Union}` ``）を使い、文字列のパターンを型として表現する。
イベント名・メソッド名・CSSプロパティのような「規則性のある文字列」を Union 型として自動生成できる。
手書きの Union 型と異なり、ベースとなる型が変われば自動追従する。

**根拠**:
- 文字列の命名規則をコンパイル時に強制できる
- Union 型のベース（例：イベント名）と派生型（例：`on${EventName}`）を DRY に保てる
- `Capitalize` / `Uppercase` などの組み込み文字列操作型と組み合わせて表現力が増す

**コード例**:
```tsx
// Good: イベントハンドラー名を自動生成
type EventName = 'click' | 'focus' | 'blur' | 'change';
type HandlerName = `on${Capitalize<EventName>}`;
// 'onClick' | 'onFocus' | 'onBlur' | 'onChange'

// Good: APIエンドポイントの命名をパターン化
type HttpMethod = 'get' | 'post' | 'put' | 'delete';
type ApiAction = `${HttpMethod}${Capitalize<string>}`; // 例: 'getUser', 'postUser'

// Good: CSS変数名の型安全なアクセス
type ColorScale = 100 | 200 | 300 | 400 | 500;
type ColorName = 'blue' | 'red' | 'green';
type CSSColorVar = `--color-${ColorName}-${ColorScale}`;
// '--color-blue-100' | '--color-blue-200' | ... など全組み合わせ

// Bad: 手書きで Union 型を列挙（ベース型の変更に追従しない）
type HandlerName = 'onClick' | 'onFocus' | 'onBlur' | 'onChange';
```

**出典**:
- [TypeScript Handbook: Template Literal Types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html) (TypeScript公式 / 2021-11)

**バージョン**: TypeScript 4.1+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 8. `satisfies` 演算子で型チェックと型推論を両立する

`satisfies` 演算子は、値が特定の型を満たすことを検証しつつ、
推論される型を変数の宣言型に「昇格」させずに保持する。
型注釈（`: Type`）では推論型が広くなりすぎる問題と、型アサーション（`as Type`）で型チェックが無効化される問題を同時に解決する。

**根拠**:
- 型注釈だと具体的なリテラル型が失われ、メソッドの補完精度が下がる
- 型アサーション（`as`）はコンパイラのチェックを回避してしまう
- `satisfies` は「型チェックはするが型推論は維持する」というバランスを実現する

**コード例**:
```tsx
type Palette = Record<string, string | [number, number, number]>;

// Bad: 型注釈だと 'red' の値が string | [number, number, number] に広がり、
// .toUpperCase() などメソッドが補完されない
const palette: Palette = {
  red: '#ff0000',
  green: [0, 128, 0],
};
palette.red.toUpperCase(); // エラー: string | [number, number, number] にはない

// Good: satisfies で型チェックしつつリテラル型を保持
const palette = {
  red: '#ff0000',
  green: [0, 128, 0],
} satisfies Palette;
palette.red.toUpperCase(); // OK: 推論型は string のまま
palette.green.map(v => v * 2); // OK: 推論型は [number, number, number] のまま

// 型制約違反はコンパイルエラーになる
const bad = {
  red: 123, // エラー: number は string | [number, number, number] に代入不可
} satisfies Palette;
```

**出典**:
- [TypeScript 4.9 Release Notes: The `satisfies` Operator](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html#the-satisfies-operator) (TypeScript公式 / 2022-11)

**バージョン**: TypeScript 4.9+
**確信度**: 高
**最終更新**: 2026-05-06

---
