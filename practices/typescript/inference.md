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

関数宣言のパラメータは、コールバックのように文脈から型が決まる場合（コンテキストによる型付け）を除いて推論されないため、必ず型を明示する。
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
**最終更新**: 2026-10-25

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

**バージョン**: TypeScript 2.8+（コード例の `Awaited` は 4.5+）
**確信度**: 高
**最終更新**: 2026-10-25

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

// Good: 実際のイベントオブジェクトなら instanceof で判定する
function handleEvent(event: unknown) {
  if (event instanceof MouseEvent) {
    console.log(event.clientX); // 安全
  }
}

// Good: 構造で判定するユーザー定義型ガードは、プロパティの有無だけでなく値の型まで確かめる
// （'clientX' in value だけだと { clientX: 'x', clientY: null } も通ってしまう）
type Point = { clientX: number; clientY: number };

function isPoint(value: unknown): value is Point {
  return (
    typeof value === 'object' &&
    value !== null &&
    'clientX' in value &&
    'clientY' in value &&
    typeof value.clientX === 'number' &&
    typeof value.clientY === 'number'
  );
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

**バージョン**: TypeScript 4.0+（`in` で絞り込んだ後に `value.clientX` を参照できるのは 4.9+）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 5. `Array.filter` に型述語を渡して絞り込み結果の型を正確にする

TypeScript 5.5 以降は、`filter(x => x !== null)` のような単純な比較なら型述語が推論され、結果は `number[]` になる。
5.4 以前と、5.5 以降でも推論されないケース（truthiness 判定 `x => !!x`、戻り値を `boolean` と注釈した判定関数、複数の `return` を持つ関数など）では、型述語 `(x): x is T` を明示して戻り値の配列要素型を絞り込む。
再利用する判定関数は、型述語を明示しておくと推論の条件に左右されない。

**根拠**:
- TS 5.4 以前のコントロールフロー解析は `Array.filter` のコールバックから型述語を推論しない
- 型述語が推論されるのは、戻り値の注釈がなく、`return` が 1 つで、引数を変更せず、引数の絞り込みに結び付いた `boolean` を返す関数に限られる（TS 5.5 リリースノート）
- truthiness 判定は `0` や `''` も除外するので「`true` なら `T`、`false` なら `T` ではない」が成り立たず、型述語は推論されない。意図せず `0` を落とすバグにもなる

**コード例**:
```tsx
const items: (number | null)[] = [1, null, 2, null, 3];

// TS 5.5+: 単純な比較なら型述語が推論される
const nums = items.filter(x => x !== null);
// 型: number[]（5.4 以前は (number | null)[] のまま）

// Bad: 戻り値を boolean と注釈した判定関数は型述語として扱われない
function isPresent(x: number | null): boolean {
  return x !== null;
}
const a = items.filter(isPresent);
// 型: (number | null)[]

// Good: 再利用する判定関数は型述語を明示する
function isNumber(x: number | null): x is number {
  return x !== null;
}
const b = items.filter(isNumber);
// 型: number[]

// Bad: truthiness 判定では推論されず、0 も除外される
const scores: (number | undefined)[] = [0, 80, undefined];
const truthy = scores.filter(s => !!s);
// 型: (number | undefined)[]、値: [80]

// Good: 意図どおりの比較を書く（TS 5.5+ では型述語が推論される）
const defined = scores.filter(s => s !== undefined);
// 型: number[]、値: [0, 80]
```

**出典**:
- [TypeScript Handbook: Narrowing - Using type predicates](https://raw.githubusercontent.com/microsoft/TypeScript-Website/v2/packages/documentation/copy/en/handbook-v2/Narrowing.md) (microsoft/TypeScript-Website / v2ブランチ) ※2026-05-06に実際にfetch成功
- [TypeScript 5.5 Release Notes: Inferred Type Predicates](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#inferred-type-predicates) (TypeScript公式、型述語が推論される条件と truthiness 判定の例) ※2026-10-25 確認

**バージョン**: TypeScript 2.0+（型述語）/ 5.5+（型述語の推論）
**確信度**: 高
**最終更新**: 2026-10-25

---

#### 追加根拠 (2026-05-06) — ルール1「推論できる型は明示的に書かない」

新たに以下の記事/ドキュメントで同じプラクティスが推奨された:
- [TypeScript Handbook: Everyday Types](https://raw.githubusercontent.com/microsoft/TypeScript-Website/v2/packages/documentation/copy/en/handbook-v2/Everyday%20Types.md) (microsoft/TypeScript-Website / v2ブランチ) ※2026-05-06に実際にfetch成功

公式ハンドブックは「コンテキスト型付け（Contextual Typing）」を明示的に解説している: `names.forEach(s => s.toUpperCase())` の引数 `s` は、`names` が `string[]` であるコンテキストから自動的に `string` と推論されるため注釈不要。`map` / `filter` / `reduce` などの配列メソッドのコールバック引数は原則として型注釈を書く必要がなく、逆に書くと「冗長な注釈は保守コストを増やす」として公式ドキュメントで避けるよう促されている。

**確信度**: 既存（高）→ 高（公式文書で実証済み）

---

#### 追加根拠 (2026-05-06) — ルール4「型ガードで unknown / Union 型を安全に絞り込む」

新たに以下の記事/ドキュメントで同じプラクティスが推奨された:
- [TypeScript Handbook: Narrowing](https://raw.githubusercontent.com/microsoft/TypeScript-Website/v2/packages/documentation/copy/en/handbook-v2/Narrowing.md) (microsoft/TypeScript-Website / v2ブランチ) ※2026-05-06に実際にfetch成功

Narrowing.md では型絞り込み技法の完全なカタログを提供: `typeof`（string/number/boolean/symbol）, truthiness（null/undefined フィルタ）, 等値チェック（`===`/`!==`）, `in` 演算子（プロパティ存在確認）, `instanceof`（プロトタイプチェーン確認）, ユーザー定義型述語（`x is T`）。公式として「ユーザー定義型述語は再利用可能な型チェックロジックを関数として共有する手段」であることが強調されており、`src/utils/typeGuards.ts` のような共有ユーティリティとして集約するパターンを推奨。また never 型を使った exhaustiveness check も同ドキュメントで解説されている。

**確信度**: 既存（高）→ 高（公式文書で実証済み）
