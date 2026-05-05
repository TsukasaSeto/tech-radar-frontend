# ユーティリティ型のベストプラクティス

## ルール

### 1. `Pick` / `Omit` でオブジェクト型のサブセットを作る

大きな型から特定のプロパティだけを使いたい場合は `Pick`、
特定のプロパティを除外したい場合は `Omit` を使い、手動で型を再定義しない。

**根拠**:
- 元の型と同期が保たれる（元が変われば派生型も追従）
- 型定義の重複を避けられる
- 意図が明確に伝わる（「Userのname, emailだけ」など）

**コード例**:
```tsx
type User = {
  id: string;
  name: string;
  email: string;
  passwordHash: string;
  createdAt: Date;
};

// Bad: 手動で型を再定義（元の型と乖離しやすい）
type UserPublic = {
  id: string;
  name: string;
  email: string;
};

// Good: Pick で必要なプロパティだけ選択
type UserPublic = Pick<User, 'id' | 'name' | 'email'>;

// Good: Omit で不要なプロパティを除外
type UserWithoutPassword = Omit<User, 'passwordHash'>;

// 実用例: フォームの型
type CreateUserForm = Omit<User, 'id' | 'createdAt'>;
type UpdateUserForm = Partial<Omit<User, 'id' | 'createdAt'>>;
```

**出典**:
- [TypeScript Handbook: Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html) (TypeScript公式)

**バージョン**: TypeScript 2.1+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. `Partial` / `Required` でオプショナリティを制御する

更新フォームや設定のオーバーライドには `Partial` を使い、
全プロパティを必須にする必要がある場合は `Required` を使う。

**根拠**:
- `Partial` は全プロパティをオプショナルにし、パッチ更新の型定義を簡潔にする
- `Required` は設定のデフォルト値マージ後など、全プロパティが確実に存在する場合に使う
- 型の意図が明確になる

**コード例**:
```tsx
type Config = {
  apiUrl: string;
  timeout?: number;
  retries?: number;
  headers?: Record<string, string>;
};

// Partial: ユーザーが設定を上書きする際の型
function mergeConfig(defaults: Config, overrides: Partial<Config>): Config {
  return { ...defaults, ...overrides };
}

// Required: デフォルト値マージ後は全プロパティが存在する
const defaultConfig: Required<Config> = {
  apiUrl: 'https://api.example.com',
  timeout: 5000,
  retries: 3,
  headers: {},
};

// PATCH リクエストの型
type PatchUserRequest = Partial<Pick<User, 'name' | 'email'>>;
```

**出典**:
- [TypeScript Handbook: Partial](https://www.typescriptlang.org/docs/handbook/utility-types.html#partialtype) (TypeScript公式)

**バージョン**: TypeScript 2.1+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. `Record` でキー・バリュー型のマップを表現する

`{ [key: string]: T }` の代わりに `Record<K, V>` を使い、
キーの型を明示的に制限する。

**根拠**:
- `Record` はキーの型を Union 型で制限できる
- インデックスシグネチャより意図が明確
- 全てのキーが揃っているかをコンパイル時に検証できる

**コード例**:
```tsx
// Bad: インデックスシグネチャ（キー型が広すぎる）
const translations: { [key: string]: string } = {
  hello: 'こんにちは',
  goodbye: 'さようなら',
};

// Good: Record でキーを Union 型に制限
type Locale = 'ja' | 'en' | 'zh';
type Translations = Record<Locale, string>;

const translations: Translations = {
  ja: 'こんにちは',
  en: 'Hello',
  zh: '你好',
  // fr: 'Bonjour' // コンパイルエラー（Locale に fr がない）
};

// 実用例: ステータスごとのスタイル定義
type Status = 'active' | 'inactive' | 'pending';
const statusColors: Record<Status, string> = {
  active: 'green',
  inactive: 'gray',
  pending: 'yellow',
};
```

**出典**:
- [TypeScript Handbook: Record](https://www.typescriptlang.org/docs/handbook/utility-types.html#recordkeys-type) (TypeScript公式)

**バージョン**: TypeScript 2.1+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. `Exclude` / `Extract` で Union 型を絞り込む

Union 型から特定の型を除外するには `Exclude`、特定の型だけを残すには `Extract` を使う。

**根拠**:
- 手動で Union 型を書き直す必要がなくなる
- 元の型が変わっても追従する
- 型の意図が明確になる

**コード例**:
```tsx
type Status = 'draft' | 'published' | 'archived' | 'deleted';

// Exclude: 特定の型を除外
type ActiveStatus = Exclude<Status, 'deleted'>;
// 'draft' | 'published' | 'archived'

// Extract: 特定の型だけ残す
type PublishableStatus = Extract<Status, 'draft' | 'published'>;
// 'draft' | 'published'

// NonNullable は Exclude<T, null | undefined> のエイリアス
type RequiredValue = NonNullable<string | null | undefined>;
// string

// 実用例: イベントハンドラーのサブセット
type MouseEvents = Extract<keyof HTMLElementEventMap, `mouse${string}`>;
// 'mousedown' | 'mouseenter' | 'mouseleave' | 'mousemove' | etc.
```

**出典**:
- [TypeScript Handbook: Exclude](https://www.typescriptlang.org/docs/handbook/utility-types.html#excludeuniontype-excludedmembers) (TypeScript公式)

**バージョン**: TypeScript 2.8+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 5. `Awaited` で非同期関数の解決型を取得する

`Promise` や非同期関数の戻り値型から、解決後の型を取り出すには `Awaited` を使う。

**根拠**:
- `ReturnType<typeof asyncFn>` だと `Promise<T>` になり、実際の値の型が取れない
- `Awaited` はネストした `Promise` も再帰的に解決する
- 非同期処理を含むコードの型定義が簡潔になる

**コード例**:
```tsx
async function fetchUser(id: string) {
  const res = await fetch(`/api/users/${id}`);
  return res.json() as Promise<{ name: string; email: string }>;
}

// Bad: Promise<T> が残る
type FetchUserReturn = ReturnType<typeof fetchUser>;
// Promise<{ name: string; email: string }>

// Good: 解決後の型を取得
type User = Awaited<ReturnType<typeof fetchUser>>;
// { name: string; email: string }

// Promiseのネストにも対応
type Nested = Awaited<Promise<Promise<string>>>;
// string
```

**出典**:
- [TypeScript Handbook: Awaited](https://www.typescriptlang.org/docs/handbook/utility-types.html#awaitedtype) (TypeScript公式)

**バージョン**: TypeScript 4.5+
**確信度**: 高
**最終更新**: 2026-05-05
