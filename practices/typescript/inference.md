# 型推論のベストプラクティス

## ルール

### 1. 推論できる型は明示的に書かない

TypeScript が型を推論できる箇所に冗長な型注釈を書かない。
変数の初期化時・関数の戻り値（内部実装）・配列リテラルなどは推論に任せる。

**根拠**:
- 冗長な型注釈はコードを読みにくくし、変更コストを増やす
- 推論された型は実装と常に一致するため、アノテーションとの乖離が起きない
- TypeScript の型推論は非常に強力で、ほとんどのケースで正確

**コード例**:
```tsx
// Bad: 推論できるのに明示
const count: number = 0;
const name: string = 'Alice';
const arr: number[] = [1, 2, 3];

// Good: 推論に任せる
const count = 0;
const name = 'Alice';
const arr = [1, 2, 3];

// Good: 境界（公開API・関数シグネチャ）では明示する
function fetchUser(id: string): Promise<User> {
  // 戻り値の型を公開APIでは明示することが多い
}

// Bad: 戻り値の型が複雑で推論困難な場合でも不明瞭なまま
function getConfig() {
  return { host: 'localhost', port: 3000, ssl: false };
  // 呼び出し元で型が分かりにくい
}
```

**出典**:
- [TypeScript Handbook: Type Inference](https://www.typescriptlang.org/docs/handbook/type-inference.html) (TypeScript公式)

**バージョン**: TypeScript 4.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. 関数のパラメータには型注釈を付ける（境界では明示）

関数のパラメータは TypeScript が推論できないため必ず型を明示する。
外部に公開する関数（エクスポートする関数）の戻り値も明示的に型付けする。

**根拠**:
- パラメータの型はコールサイトの型安全の根拠になる
- 公開関数の戻り値型を明示することで、意図しない型変更を検出できる
- ドキュメントとしての役割も果たす

**コード例**:
```tsx
// Bad: パラメータ型なし
function formatPrice(amount, currency) {
  return `${currency}${amount.toFixed(2)}`;
}

// Good: パラメータ型あり
function formatPrice(amount: number, currency: string): string {
  return `${currency}${amount.toFixed(2)}`;
}

// Good: コールバックの型も定義
function processItems<T>(
  items: T[],
  transform: (item: T, index: number) => T
): T[] {
  return items.map(transform);
}
```

**出典**:
- [TypeScript Handbook: More on Functions](https://www.typescriptlang.org/docs/handbook/2/functions.html) (TypeScript公式)

**バージョン**: TypeScript 4.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. `ReturnType` / `Parameters` で既存関数から型を抽出する

関数の戻り値型やパラメータ型を手動で定義するのではなく、
`ReturnType<typeof fn>` や `Parameters<typeof fn>` で自動抽出する。
関数定義が変われば型定義も自動更新される。

**根拠**:
- 型定義と実装の二重メンテナンスを避けられる
- 外部ライブラリの関数から型情報を取り出す際に特に有用
- Single Source of Truth の原則に沿う

**コード例**:
```tsx
// 外部APIクライアント関数から戻り値型を抽出
async function fetchUserProfile(id: string) {
  const response = await api.get(`/users/${id}`);
  return response.data as { name: string; email: string; role: string };
}

type UserProfile = Awaited<ReturnType<typeof fetchUserProfile>>;
// { name: string; email: string; role: string }

// パラメータ型の抽出
function createQueryString(params: Record<string, string | number>) {
  return new URLSearchParams(
    Object.entries(params).map(([k, v]) => [k, String(v)])
  ).toString();
}

type QueryParams = Parameters<typeof createQueryString>[0];
// Record<string, string | number>
```

**出典**:
- [TypeScript Handbook: Utility Types - ReturnType](https://www.typescriptlang.org/docs/handbook/utility-types.html#returntypetype) (TypeScript公式)

**バージョン**: TypeScript 2.8+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. 型ガードで `unknown` / Union 型を安全に絞り込む

`typeof`、`instanceof`、`in` 演算子、あるいはカスタム型ガード関数を使って型を絞り込む。
型アサーション（`as`）の代わりに型ガードを優先する。

**根拠**:
- 型アサーション（`as`）はランタイムエラーのリスクを隠蔽する
- 型ガードはランタイムの実際の値と型システムを一致させる
- ユーザー定義型ガードで再利用可能な型チェックロジックを作れる

**コード例**:
```tsx
// Bad: 型アサーションで無理やり型付け
function handleEvent(event: unknown) {
  const e = event as MouseEvent;
  console.log(e.clientX); // ランタイムエラーの危険
}

// Good: ユーザー定義型ガード
function isMouseEvent(event: unknown): event is MouseEvent {
  return (
    typeof event === 'object' &&
    event !== null &&
    'clientX' in event &&
    'clientY' in event
  );
}

function handleEvent(event: unknown) {
  if (isMouseEvent(event)) {
    console.log(event.clientX); // 安全
  }
}

// Good: instanceof による絞り込み
function handleError(err: unknown) {
  if (err instanceof Error) {
    console.error(err.message);
  } else {
    console.error(String(err));
  }
}
```

**出典**:
- [TypeScript Handbook: Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html) (TypeScript公式)

**バージョン**: TypeScript 4.0+
**確信度**: 高
**最終更新**: 2026-05-05
