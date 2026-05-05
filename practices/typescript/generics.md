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
