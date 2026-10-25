# Developer Experience (DX) Practices

開発者体験に関するベストプラクティス。
命名規約・コードレビュー・モノレポ運用・依存管理を扱う。

## ファイル構成

- `naming.md` — コンポーネント / hook / 型 / boolean / ハンドラーの命名規約
- `code-review.md` — レビュー観点・PR ラベル・ファシリテーション
- `monorepo.md` — pnpm workspace / Turborepo / パッケージ境界
- `dependency-management.md` — lockfile / packageManager / range 指定と major 更新の進め方（更新ポリシーは `security/dependency-security.md`）

## スコープ

- 技術的な実装パターンよりも、チームの開発生産性・コラボレーションに直結する規約や運用ルール
- 「人とコードのインターフェース」を扱う（コードと外部システムのインターフェースは他カテゴリ）
- 「うちのチームは別ルールでやっている」場合は遠慮なくプロジェクト独自に上書きする
