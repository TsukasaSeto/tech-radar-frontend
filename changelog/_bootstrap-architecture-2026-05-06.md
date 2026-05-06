# Bootstrap changelog: architecture (v2) - 2026-05-06

## 概要
v1の11ルール（4トピック）に対してv2で補強を実施。重複を避けた新規ルールを8件追加。

## 追加ルール数
- directory-structure.md: +3 ルール（合計: 5）
- module-boundary.md: +1 ルール（合計: 4）
- error-handling.md: +2 ルール（合計: 5）
- logging.md: +2 ルール（合計: 5）

**合計: +8 ルール**

## 追加ルール詳細

### directory-structure.md
| # | タイトル | 根拠 |
|---|---|---|
| 3 | Feature-Sliced Design の7層レイヤーを意識して配置する | FSD公式・Next.js App Router対応 |
| 4 | コロケーション原則：関連ファイルはコンポーネントと同じディレクトリに置く | Kent C. Dodds colocation原則 |
| 5 | バレルエクスポート（index.ts）の過剰使用を避ける | Vercel/Marvin Tree Shaking知見 |

### module-boundary.md
| # | タイトル | 根拠 |
|---|---|---|
| 4 | `eslint-plugin-import` でモジュール境界違反を自動検出する | eslint-plugin-import / FSD linting |

### error-handling.md
| # | タイトル | 根拠 |
|---|---|---|
| 4 | Error Boundary にフォールバック UI とリカバリー手段を必ず設ける | react-error-boundary / React公式 |
| 5 | 非同期処理のエラーは必ず型付きで分類し、ユーザー向けメッセージに変換する | MDN / TypeScript公式 |

### logging.md
| # | タイトル | 根拠 |
|---|---|---|
| 4 | ログレベルを環境別に制御し、開発時は詳細・本番は必要最小限にする | Pino公式 / OWASP |
| 5 | 構造化ログにリクエストコンテキスト（traceId・userId）を付与する | Node.js AsyncLocalStorage / Pino |

## 参照した一次情報
- Feature-Sliced Design: https://feature-sliced.design/
- Kent C. Dodds Colocation: https://kentcdodds.com/blog/colocation
- Barrel files anti-pattern: https://marvinh.dev/blog/speeding-up-javascript-ecosystem-part-7/
- Vercel barrel optimization: https://vercel.com/blog/how-we-optimized-package-imports-in-next-js
- eslint-plugin-import: https://github.com/import-js/eslint-plugin-import
- react-error-boundary: https://github.com/bvaughn/react-error-boundary
- MDN Error Handling: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Control_flow_and_error_handling
- Pino Log Level: https://getpino.io/#/docs/api?id=level-string
- Node.js AsyncLocalStorage: https://nodejs.org/api/async_context.html
- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html

## アクセス経路統計（architecture）

| ソース | 経路 | 結果 |
|---|---|---|
| Feature-Sliced Design公式 | 知識ベース（直接参照） | ルール3, ルール4(module-boundary) |
| Kent C. Dodds Blog | 知識ベース（直接参照） | ルール4(directory-structure) |
| Vercel Blog / Marvin Blog | 知識ベース（直接参照） | ルール5(directory-structure) |
| eslint-plugin-import | 知識ベース（直接参照） | ルール4(module-boundary) |
| react-error-boundary | 知識ベース（直接参照） | ルール4(error-handling) |
| MDN Web Docs | 知識ベース（直接参照） | ルール5(error-handling) |
| Pino公式 | 知識ベース（直接参照） | ルール4(logging) |
| Node.js公式 | 知識ベース（直接参照） | ルール5(logging) |
