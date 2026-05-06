# ジェネリクスのベストプラクティス

## ルール

### 1. 型パラメータ名に意味のある名前を付ける（単文字は避ける）

`T`, `U`, `V` などの単文字名は最小限にとどめ、意味のある型パラメータ名を使う。
コレクション系のみ `T` を許容し、それ以外は `TKey`, `TValue`, `TEntity` などとする。

**根拠**:
- コードの読みやすさが向上し、意図が伝わりやすい
- 複数の型パラメータがある場合の混乱を防ぐ
- ジェネリクスのドキュメントとしての役割を果たす

**コード例**:
```tsx
// Bad: 意味不明な単文字
function merge<T, U>(a: T, b: U): T & U {
  return { ...a, ...b } as T & U;
}

// Good: 意味のある名前
function merge<TSource, TTarget>(
  source: TSource,
  target: TTarget
): TSource & TTarget {
  return { ...source, ...target } as TSource & TTarget;
}

// コレクション系は T でも可
function first<T>(items: T[]): T | undefined {
  return items[0];
}

// 複数パラメータには意味のある名前を
type Repository<TEntity, TId = string> = {
  findById(id: TId): Promise<TEntity | null>;
  save(entity: TEntity): Promise<TEntity>;
  delete(id: TId): Promise<void>;
};
```

**出典**:
- [TypeScript Handbook: Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html) (TypeScript公式)

**バージョン**: TypeScript 4.0+
**確信度**: 中
**最終更新**: 2026-05-05

---

### 2. 型パラメータに `extends` で制約を付ける

ジェネリクスを使う際は `extends` で型パラメータの制約を明示し、
型パラメータに対して安全にプロパティアクセスできるようにする。

**根拠**:
- 制約なしでは型パラメータに対してプロパティアクセスができない
- 呼び出し元で不正な型を渡した場合のエラーを早期に発見できる
- API の意図を明確に伝えられる

**コード例**:
```tsx
// Bad: 制約なしで keyof を使おうとする（エラーになる）
function getProperty<T>(obj: T, key: string) {
  return obj[key]; // error: Element implicitly has an 'any' type
}

// Good: extends で制約を付ける
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: 'Alice', age: 30 };
const name = getProperty(user, 'name'); // string
const age  = getProperty(user, 'age');  // number
// getProperty(user, 'email'); // コンパイルエラー

// 実用例: idを持つエンティティに制約
function findById<T extends { id: string }>(
  items: T[],
  id: string
): T | undefined {
  return items.find(item => item.id === id);
}
```

**出典**:
- [TypeScript Handbook: Generic Constraints](https://www.typescriptlang.org/docs/handbook/2/generics.html#generic-constraints) (TypeScript公式)

**バージョン**: TypeScript 4.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. 過度なジェネリクスを避け、Union 型で代替できないか検討する

ジェネリクスは強力だが、Union 型で十分な場合は Union 型を使う。
型パラメータが1箇所でしか使われていない場合はジェネリクスにする意味がない。

**根拠**:
- 不要なジェネリクスはコードの複雑さを増すだけで利益がない
- 型パラメータは2箇所以上で使われて初めて価値を持つ
- Union 型の方がコードの見通しが良いケースが多い

**コード例**:
```tsx
// Bad: 型パラメータが1箇所しか使われていない（意味がない）
function identity<T>(value: T): string {
  return String(value);
}

// Good: ジェネリクス不要
function identity(value: unknown): string {
  return String(value);
}

// Bad: Union 型で十分
function process<T extends string | number>(value: T): T {
  return value;
}

// Good: Union 型で表現
function process(value: string | number): string | number {
  return value;
}

// Good: ジェネリクスが真に有用なケース（入出力の型が連動する）
function mapRecord<K extends string, V, U>(
  record: Record<K, V>,
  transform: (value: V) => U
): Record<K, U> {
  return Object.fromEntries(
    Object.entries(record).map(([k, v]) => [k, transform(v as V)])
  ) as Record<K, U>;
}
```

**出典**:
- [TypeScript Handbook: Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html) (TypeScript公式)

**バージョン**: TypeScript 4.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. 条件型（Conditional Types）は複雑にしすぎない

`T extends U ? X : Y` の条件型は型レベルのプログラミングを可能にするが、
深いネストは型エラーメッセージを解読不能にする。
標準ユーティリティ型で代替できる場合はそちらを優先する。

**根拠**:
- 条件型のネストはデバッグが困難になる
- TypeScript の型エラーメッセージが著しく読みにくくなる
- 標準ユーティリティ型は広く知られており、意図が伝わりやすい

**コード例**:
```tsx
// Bad: 深すぎる条件型
type DeepReadonly<T> = T extends (infer U)[]
  ? DeepReadonlyArray<U>
  : T extends object
  ? DeepReadonlyObject<T>
  : T;
// このような実装は型エラー時のデバッグが難しい

// Good: まず標準ユーティリティ型で実現できないか確認
type ReadonlyUser = Readonly<User>;
type PartialConfig = Partial<Config>;

// 条件型が必要な場合は1段階までにとどめる
type Nullable<T> = T | null;
type NonNullable<T> = T extends null | undefined ? never : T; // 標準に存在
```

**出典**:
- [TypeScript Handbook: Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html) (TypeScript公式)

**バージョン**: TypeScript 2.8+
**確信度**: 中
**最終更新**: 2026-05-05

---

### 5. `infer` キーワードで条件型の中から型を抽出する

条件型の `extends` 節の中で `infer` を使うと、型パターンマッチング時に
部分的な型を変数として束縛して取り出せる。
配列の要素型・Promise の解決型・関数の引数型など「型の内側にある型」を抽出するのに使う。
`ReturnType` や `Parameters` などの標準ユーティリティ型は `infer` を使って実装されている。

**根拠**:
- `infer` なしでは型の「分解」が難しく、型ユーティリティの表現力が著しく制限される
- 標準ユーティリティ型を自作・拡張する際の基礎知識として必須
- 再帰的な型定義と組み合わせることで深い型の分解が可能になる

**コード例**:
```tsx
// 配列の要素型を抽出
type ElementOf<T> = T extends (infer U)[] ? U : never;
type StrEl = ElementOf<string[]>; // string
type NumEl = ElementOf<number[]>; // number

// Promiseの解決型を抽出（Awaited の自前実装）
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;
type Resolved = UnwrapPromise<Promise<{ id: string }>>; // { id: string }

// 関数の第一引数の型を抽出
type FirstArg<T> = T extends (first: infer A, ...rest: any[]) => any ? A : never;
type F = FirstArg<(id: string, name: string) => void>; // string

// 実用例: React コンポーネントの Props 型を取り出す
type PropsOf<T> = T extends React.ComponentType<infer P> ? P : never;
type ButtonProps = PropsOf<typeof Button>; // Button コンポーネントの props 型
```

**出典**:
- [TypeScript Handbook: Inferring Within Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html#inferring-within-conditional-types) (TypeScript公式 / 2021-11)

**バージョン**: TypeScript 2.8+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 6. 分配的条件型（Distributive Conditional Types）の挙動を理解して活用する

条件型の型パラメータに Union 型を渡すと、Union の各メンバーに対して個別に条件型が適用される（分配）。
この性質を利用すると、Union 型のフィルタリングや変換を簡潔に記述できる。
意図しない分配を防ぐには型パラメータを `[T]` のようにタプルで包む。

**根拠**:
- `Extract` / `Exclude` などの標準ユーティリティ型は分配的条件型で実装されている
- Union に対するマッピングを簡潔に書くための重要なパターン
- 分配を意図しない場合（`T extends T` のような自己比較）はタプルで防げる

**コード例**:
```tsx
// 分配の例: Union の各メンバーに個別適用
type ToArray<T> = T extends any ? T[] : never;
type StrOrNumArr = ToArray<string | number>;
// string[] | number[]（分配される）

// 分配を抑制する例: タプルで包む
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;
type StrOrNumArr2 = ToArrayNonDist<string | number>;
// (string | number)[]（分配されない）

// 実用例: オブジェクト型のプロパティから null を除去
type NonNullableProperties<T> = {
  [K in keyof T]: NonNullable<T[K]>;
};
type SafeUser = NonNullableProperties<{
  name: string | null;
  age: number | undefined;
}>;
// { name: string; age: number }
```

**出典**:
- [TypeScript Handbook: Distributive Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html#distributive-conditional-types) (TypeScript公式 / 2021-11)

**バージョン**: TypeScript 2.8+
**確信度**: 中
**最終更新**: 2026-05-06

#### 追加根拠 (2026-05-06)

新たに以下のドキュメントで同じプラクティスが推奨された:
- [TypeScript Handbook: Conditional Types](https://raw.githubusercontent.com/microsoft/TypeScript-Website/v2/packages/documentation/copy/en/handbook-v2/Type%20Manipulation/Conditional%20Types.md) (TypeScript公式ドキュメント) ※2026-05-06に実際にfetch成功

公式ドキュメントで分配的条件型の制御方法が明示的に記述されており、「`extends` の両側を角括弧で囲む（`[T] extends [any]`）」ことで非分配化できることが示されている。また `Extract`/`Exclude` などの標準ユーティリティ型が分配的条件型で実装されていることも言及されており、Union 型フィルタリングの基盤知識として重要であることが裏付けられた。さらに `infer` キーワードとの組み合わせで「宣言的に新しいジェネリック型変数を導入できる」という原則も強調されている。

**確信度**: 既存（中）→ 高（実証付き）

---
