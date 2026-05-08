# Strict モードのベストプラクティス

## ルール

### 1. `strict: true` を必ず有効にする

`tsconfig.json` で `strict: true` を設定し、TypeScript の全 strict 系オプションを有効にする。
新規プロジェクトでは最初から有効にし、既存プロジェクトは段階的に対応する。

**根拠**:
- `strict: true` は `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes` などを一括有効化
- null 参照エラー、暗黙の `any` など実行時バグの大多数を型チェックで防げる
- TypeScript を使う最大のメリットを享受できる

**コード例**:
```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true
  }
}
```

```tsx
// strict: false の場合はエラーにならない（危険）
function greet(name) { // noImplicitAny: name は暗黙の any
  console.log(name.toUpperCase());
}

let user: User | null = null;
user.name; // strictNullChecks: null かもしれないのにアクセスできる

// strict: true なら上記はコンパイルエラー
```

**出典**:
- [TypeScript tsconfig reference: strict](https://www.typescriptlang.org/tsconfig#strict) (TypeScript公式)

**バージョン**: TypeScript 2.3+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. `noUncheckedIndexedAccess: true` を追加で有効にする

`strict: true` に含まれないが重要な `noUncheckedIndexedAccess` も有効にする。
配列・オブジェクトへのインデックスアクセスの戻り値が `T | undefined` になり、
境界外アクセスを型チェックで検出できる。

**根拠**:
- `arr[0]` は配列が空の場合 `undefined` になるが、`strict: true` だけでは型が `T` のまま
- 境界外アクセスは実行時エラーの主な原因のひとつ
- 配列アクセス後に undefined チェックが強制され、バグを防げる

**コード例**:
```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true
  }
}
```

```tsx
const items = ['a', 'b', 'c'];

// noUncheckedIndexedAccess なしの場合
const first: string = items[0]; // OK（でも実行時に undefined の可能性がある）

// noUncheckedIndexedAccess ありの場合
const first: string | undefined = items[0];
if (first !== undefined) {
  console.log(first.toUpperCase()); // 安全
}
```

**出典**:
- [TypeScript tsconfig reference: noUncheckedIndexedAccess](https://www.typescriptlang.org/tsconfig#noUncheckedIndexedAccess) (TypeScript公式)

**バージョン**: TypeScript 4.1+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. `exactOptionalPropertyTypes: true` でオプショナルプロパティを厳密に扱う

`exactOptionalPropertyTypes: true` を有効にすると、オプショナルプロパティに
`undefined` を明示的に代入しようとした場合にエラーになる。

**根拠**:
- オプショナル（プロパティが存在しない）と `undefined` 代入を区別できる
- `in` 演算子での存在チェックが正確になる
- APIのオプショナルフィールドの扱いが厳密になる

**コード例**:
```json
{
  "compilerOptions": {
    "strict": true,
    "exactOptionalPropertyTypes": true
  }
}
```

```tsx
type Config = {
  theme?: 'light' | 'dark';
};

// exactOptionalPropertyTypes あり
const config: Config = { theme: undefined }; // エラー
// Type 'undefined' is not assignable to type 'light' | 'dark'

// 正しい書き方
const config: Config = {}; // OK
const config: Config = { theme: 'dark' }; // OK
```

**出典**:
- [TypeScript tsconfig reference: exactOptionalPropertyTypes](https://www.typescriptlang.org/tsconfig#exactOptionalPropertyTypes) (TypeScript公式)

**バージョン**: TypeScript 4.4+
**確信度**: 中
**最終更新**: 2026-05-05

---

### 4. `paths` エイリアスと `baseUrl` でインポートを整理する

相対パスの深いインポート（`../../../components/Button`）を避けるため、
`paths` でエイリアスを設定する。

**根拠**:
- ファイル移動時のインポートパス修正が不要になる
- コードの可読性が向上する
- `@/` プレフィックスは Next.js や Vite の標準的な慣習

**コード例**:
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

```tsx
// Bad: 深い相対パス
import { Button } from '../../../components/ui/Button';

// Good: エイリアスを使用
import { Button } from '@/components/ui/Button';
```

**出典**:
- [TypeScript tsconfig reference: paths](https://www.typescriptlang.org/tsconfig#paths) (TypeScript公式)

**バージョン**: TypeScript 2.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 5. `skipLibCheck: true` でビルド速度を改善する

`skipLibCheck: true` は型定義ファイル（`.d.ts`）のチェックをスキップし、
ビルド速度を改善する。

**根拠**:
- サードパーティの型定義には互換性問題がある場合があり、不要なエラーを抑制できる
- ビルド時間の短縮（大規模プロジェクトで顕著）
- Next.js / Vite のデフォルト設定でも推奨されている

**コード例**:
```json
{
  "compilerOptions": {
    "strict": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "moduleResolution": "bundler",
    "module": "ESNext",
    "target": "ES2017"
  }
}
```

**出典**:
- [TypeScript tsconfig reference: skipLibCheck](https://www.typescriptlang.org/tsconfig#skipLibCheck) (TypeScript公式)

**バージョン**: TypeScript 2.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 6. `useUnknownInCatchVariables: true` で catch 節の変数を `unknown` にする

`strict: true` に含まれる `useUnknownInCatchVariables`（TS4.0以降で `strict` に内包）を確認し、
catch 節のエラー変数が `any` ではなく `unknown` として扱われることを活用する。
エラー処理時に型ガードで安全に絞り込む習慣をチームに定着させる。

**根拠**:
- catch 節の `e` が `any` の場合、`e.message` などへの無検証アクセスが通ってしまう
- `unknown` にすることで型ガードが強制され、エラーハンドリングの品質が上がる
- `instanceof Error` チェックがベストプラクティスとして定着する

**コード例**:
```json
{
  "compilerOptions": {
    "strict": true
    // useUnknownInCatchVariables は strict: true に含まれる（TS4.4+）
  }
}
```

```tsx
// Bad: e が any として扱われ、型チェックなしで .message にアクセス
try {
  await fetchData();
} catch (e) {
  console.error(e.message); // useUnknownInCatchVariables が false だと通る
}

// Good: unknown として正しく型ガードして処理
try {
  await fetchData();
} catch (e) {
  if (e instanceof Error) {
    console.error(e.message); // 安全
  } else {
    console.error('Unknown error:', String(e));
  }
}

// さらに良い: ユーティリティ関数で共通化
function toErrorMessage(e: unknown): string {
  return e instanceof Error ? e.message : String(e);
}

try {
  await fetchData();
} catch (e) {
  reportError(toErrorMessage(e));
}
```

**出典**:
- [TypeScript tsconfig reference: useUnknownInCatchVariables](https://www.typescriptlang.org/tsconfig#useUnknownInCatchVariables) (TypeScript公式 / 2021-09)

**バージョン**: TypeScript 4.4+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 7. `noPropertyAccessFromIndexSignature: true` でインデックスシグネチャのプロパティアクセスを厳格化する

`noPropertyAccessFromIndexSignature: true` を有効にすると、インデックスシグネチャを持つ型に対して
ドット記法（`obj.key`）でのアクセスがエラーになる。
ブラケット記法（`obj['key']`）を強制することで、コードを読む人がそのプロパティが
確定的に存在するものか、インデックスアクセスかを視覚的に区別できる。

**根拠**:
- ドット記法は「確実に存在するプロパティ」、ブラケット記法は「存在するかもしれないキー」という意図を分離できる
- インデックスシグネチャから取得した値が `undefined` になり得ることを明示的に扱わせる
- コードレビュー時にインデックスアクセスのリスク箇所を一目で識別できる

**コード例**:
```json
{
  "compilerOptions": {
    "strict": true,
    "noPropertyAccessFromIndexSignature": true
  }
}
```

```tsx
type Options = {
  timeout: number;  // 明示的なプロパティ
  [key: string]: unknown;  // インデックスシグネチャ
};

declare const opts: Options;

// OK: 明示的なプロパティはドット記法でアクセス可
console.log(opts.timeout);

// エラー: noPropertyAccessFromIndexSignature が true の場合
console.log(opts.someFlag); // ドット記法は不可

// OK: インデックスシグネチャ経由はブラケット記法を使う
console.log(opts['someFlag']); // 型は unknown
```

**出典**:
- [TypeScript tsconfig reference: noPropertyAccessFromIndexSignature](https://www.typescriptlang.org/tsconfig#noPropertyAccessFromIndexSignature) (TypeScript公式 / 2021-08)

**バージョン**: TypeScript 4.2+
**確信度**: 中
**最終更新**: 2026-05-06

---

#### 追加根拠 (2026-05-06) — ルール6「`useUnknownInCatchVariables: true` で catch 節の変数を `unknown` にする」

新たに以下の記事で `unknown` の適用範囲を catch 節の外（関数パラメータ・API レスポンス等）に広げるプラクティスが推奨された:
- [Why `unknown` is Safer Than `any` in TypeScript: How Type Narrowing Makes It Practical](https://medium.com/@najmul.myself/why-unknown-is-safer-than-any-in-typescript-how-type-narrowing-makes-it-practical-25c67a6bd0c8) (Medium najmul.myself / 2026-05) ※2026-05-06に実際にfetch成功

`unknown` はシステム境界での防御的型付けとして catch 節以外にも有効: (1) **API レスポンスの型付け** — `const data: unknown = await res.json()` として受け取り、型ガード関数で絞り込む。`any` で受け取ると型チェックが無効化され、実行時エラーを招く。(2) **汎用関数パラメータ** — 受け入れる値の型が不明な場合は `any` ではなく `unknown` を使い、型ガードによる絞り込みを関数内で強制する。型ガード関数は `is` 型述語（type predicate）を返す独立した純粋関数として定義すると再利用性が高い。

```typescript
// unknown を catch 節の外でも活用するパターン

// 1. API レスポンスを unknown で受け取り型ガードで絞り込む
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value &&
    typeof (value as Record<string, unknown>).id === 'string' &&
    typeof (value as Record<string, unknown>).name === 'string'
  );
}

async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  const data: unknown = await res.json();  // any ではなく unknown
  if (!isUser(data)) throw new Error('Invalid user response shape');
  return data;  // ここで User 型として安全に扱える
}

// 2. 汎用関数パラメータに unknown を使う
function processInput(value: unknown): string {
  if (typeof value === 'string') return value.toUpperCase();
  if (typeof value === 'number') return value.toFixed(2);
  if (value instanceof Date) return value.toISOString();
  return String(value);
}
```

**確信度**: 既存（高）→ 高（catch 節外への適用拡大が Medium 実証記事で確認済み）

---

### 8. `strict: true` を段階的に導入する際は8フラグを難易度順に有効化する

既存のコードベースに `strict: true` を一括で適用するのではなく、
内包される8つのフラグを難易度（発生エラー数）が低い順に個別に有効化する。
`strictNullChecks` は最後に適用する。

**根拠**:
- `strict: true` は実際には8つの独立したフラグの束であり、一括適用すると数千件のエラーが一度に発生する
- フラグを難易度順に有効化することで、段階的なPRで進められ、レビューが可能になる
- `strictNullChecks` だけで全エラーの60〜80%を占めるため（30kライン換算で1000〜4000件）、最後に予算を確保して臨む

**有効化推奨順序**（エラー影響が少ない順）:
1. `alwaysStrict` — ほぼコストゼロ（`"use strict"` 付与のみ）
2. `noImplicitThis` — 少数（`this` パラメータ宣言で修正）
3. `useUnknownInCatchVariables` — 少数（catch節のe型ガード追加）
4. `strictBindCallApply` — 少数〜中（bind/call/apply の引数型修正）
5. `strictFunctionTypes` — 中（関数パラメータの反変性）
6. `noImplicitAny` — 中〜多（型アノテーション追加）
7. `strictPropertyInitialization` — 多（クラスプロパティ初期化）
8. `strictNullChecks` — **最多**（全体の60〜80%、複数週分の作業）

**コード例**:
```json
// 段階1: まず低コストのフラグだけ有効化
{
  "compilerOptions": {
    "alwaysStrict": true,
    "noImplicitThis": true,
    "useUnknownInCatchVariables": true
  }
}

// 最終段階: strictNullChecks を含む全フラグを有効化
{
  "compilerOptions": {
    "strict": true
  }
}
```

```typescript
// ディレクトリ単位のマイグレーション（tsconfig でスコープ分割）
// tsconfig.auth.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "strictNullChecks": true  // このディレクトリのみ先行有効化
  },
  "include": ["./src/features/auth/**/*"]
}
```

**出典**:
- [TypeScript strict Mode Is 8 Flags. Turn strictNullChecks On Last.](https://dev.to/gabrielanhaia/typescript-strict-mode-is-8-flags-turn-strictnullchecks-on-last-52mj) (dev.to gabrielanhaia / 2026-05-07) ※2026-05-07に実際にfetch成功

**バージョン**: TypeScript 4.0+
**確信度**: 高
**最終更新**: 2026-05-07

---
