# Bootstrap changelog: typescript (v2) - 2026-05-06

## 概要
v1の24ルール（5トピック）に対してv2で補強を実施。
type-design.md に Template Literal Types・`satisfies` 演算子、
generics.md に `infer` キーワード・分配的条件型、
strict-mode.md に `useUnknownInCatchVariables`・`noPropertyAccessFromIndexSignature` を追加。

## 追加ルール数
- type-design.md: +2 ルール（合計: 8）
- generics.md: +2 ルール（合計: 6）
- strict-mode.md: +2 ルール（合計: 7）

## 参照した一次情報
- [TypeScript Handbook: Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html) (TypeScript公式)
- [TypeScript Handbook: Template Literal Types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html) (TypeScript公式)
- [TypeScript 4.9 Release Notes: The satisfies Operator](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html#the-satisfies-operator) (TypeScript公式)
- [TypeScript tsconfig reference: useUnknownInCatchVariables](https://www.typescriptlang.org/tsconfig#useUnknownInCatchVariables) (TypeScript公式)
- [TypeScript tsconfig reference: noPropertyAccessFromIndexSignature](https://www.typescriptlang.org/tsconfig#noPropertyAccessFromIndexSignature) (TypeScript公式)

## アクセス経路統計（typescript）

| ソース | 経路1 (GitHub MCP) | 経路2 (WebFetch) | 結果 |
|---|---|---|---|
| TypeScript-Website (microsoft/TypeScript-Website) | ❌ アクセス拒否 | ❌ 403 Forbidden | フォールバック |
| TypeScript公式サイト (typescriptlang.org) | N/A | ❌ 403 Forbidden | フォールバック |
| TypeScript devblog / 3rd party | N/A | ❌ 403 Forbidden | フォールバック |
| 知識ベース（モデル学習済みTS仕様） | N/A | N/A | ✅ 使用 |

外部ドキュメントへのアクセスがすべて失敗したため、TypeScript仕様の学習済み知識をもとにルールを作成。
出典URLは各ルールに記載した公式ハンドブックの正規URLを使用。

## スキップしたトピック
- inference.md: 既存4ルールでカバー範囲が十分（型推論・型ガード・ReturnType/Parameters・境界での型注釈）
- utility-types.md: 既存5ルールでカバー範囲が十分（Pick/Omit・Partial/Required・Record・Exclude/Extract・Awaited）
- strict-mode.md の noUncheckedIndexedAccess / exactOptionalPropertyTypes: **v1で既にカバー済み**（ルール2・3として存在）
