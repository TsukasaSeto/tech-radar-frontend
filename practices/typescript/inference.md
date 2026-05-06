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

---

### 5. `Array.filter` に型述語を渡して絞り込み結果の型を正確にする

`array.filter(x => x !== null)` の結果型は TypeScript 5.4 以前では `(T | null)[]` のまま narrowing されない。
型述語 `(x): x is T` を filter のコールバックに使うことで、戻り値の配列要素型を正確に絞り込める。
null/undefined の除去・Discriminated Union の特定メンバー抽出などで必須のパターン。

**根拠**:
- TypeScript のコントロールフロー解析は `Array.filter` のコールバック内まで追従しない（TS 5.4 以前）
- 型述語なしで filter した配列は後続の `map` などで型エラーが頻発する
- TS 5.5 以降は一部自動推論が改善されたが、複雑な述語には型述語が引き続き必要（TypeScript公式 Narrowing.md より）

**コード例**:
```tsx
// Bad: filter 後も null が型に残る
const items: (number | null)[] = [1, null, 2, null, 3];
const nums = items.filter(x => x !== null);
// 型: (number | null)[] ← narrowing されない
nums.map(x => x.toFixed(2)); // エラー: null の可能性がある

// Good: 型述語で戻り値型を narrowing
const nums = items.filter((x): x is number => x !== null);
// 型: number[]
nums.map(x => x.toFixed(2)); // OK

// 実用例: Discriminated Union のフィルタリング
type Event =
  | { type: 'click'; target: Element }
  | { type: 'keydown'; key: string };
const events: Event[] = getEvents();
const clicks = events.filter(
  (e): e is Extract<Event, { type: 'click' }> => e.type === 'click'
);
// clicks: { type: 'click'; target: Element }[]
```

**出典**:
- [TypeScript Handbook: Narrowing - Using type predicates](https://raw.githubusercontent.com/microsoft/TypeScript-Website/v2/packages/documentation/copy/en/handbook-v2/Narrowing.md) (microsoft/TypeScript-Website / v2ブランチ) ※2026-05-06に実際にfetch成功

**バージョン**: TypeScript 2.0+（型述語）
**確信度**: 高
**最終更新**: 2026-05-06

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
