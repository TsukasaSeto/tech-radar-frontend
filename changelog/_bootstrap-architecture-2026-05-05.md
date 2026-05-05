# Bootstrap Changelog: Architecture カテゴリ

**実行日**: 2026-05-05
**ブランチ**: bootstrap/architecture-2026-05-05
**処理カテゴリ**: architecture

## 作成ファイル

| ファイル | ルール数 | 内容 |
|---|---|---|
| `practices/architecture/directory-structure.md` | 2 | Feature-Slicedディレクトリ構造・index.tsバレルファイルによるパブリックAPI定義 |
| `practices/architecture/module-boundary.md` | 3 | 一方向依存ルール・型定義の配置・環境変数のZodスキーマ検証 |
| `practices/architecture/error-handling.md` | 3 | Next.js error.tsx/not-found.tsx・Result型パターン・ErrorBoundary+Suspense統合 |
| `practices/architecture/logging.md` | 3 | 構造化ログ(Pino)・Sentry統合・クライアントログの機密情報サニタイズ |

**合計**: 11 ルール / 4 トピック

## WebFetch 結果

全URLで 403 エラーが発生したため、モデルの学習知識からルールを作成した。
参照 URL は各ファイルの出典セクションに記載済み。

## 処理ノート

- Feature-Sliced Design を推奨アーキテクチャとして採用
- Result 型パターンは TypeScript の Discriminated Union で型安全なエラー処理を実現
- 環境変数管理は t3-env（Zod ベース）のアプローチを採用
- ロギングは Pino を推奨（サーバーサイド）、クライアントはSentryのbeforeSendでサニタイズ
