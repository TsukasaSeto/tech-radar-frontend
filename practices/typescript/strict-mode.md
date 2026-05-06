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
