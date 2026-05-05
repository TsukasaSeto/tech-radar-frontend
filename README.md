# Tech Radar

フロントエンド技術のベストプラクティスを継続的に蓄積するリポジトリ。
Claude Code Routines が自動的に最新記事から実用的なプラクティスを抽出し、
プロジェクトに採用可能な形で集約する。

## 目的

このリポジトリは「読み物」ではなく「実プロジェクトに適用するルールブック」である。

- 既存プロジェクトには差分提案として適用
- 新規プロジェクトでは初期スタイルガイドとして利用
- `AGENTS.md` / `CLAUDE.md` に取り込んで Claude Code セッションで参照

## ディレクトリ構造

```
.
├── practices/              # カテゴリ別のベストプラクティス集
│   ├── api-client/         # API通信全般
│   ├── observability/      # 監視・計測・エラー追跡
│   ├── typescript/         # 型設計、推論、ジェネリクス等
│   ├── react/              # hooks、状態管理、レンダリング等
│   ├── nextjs/             # App Router、Server Components等
│   ├── testing/            # Unit/Component/E2Eテスト戦略
│   ├── performance/        # Core Web Vitals、最適化
│   ├── web-standards/      # HTML/CSS/JS、ブラウザAPI、a11y
│   └── architecture/       # 設計、ディレクトリ構造、エラー処理
├── changelog/              # 日次の変更ログ
├── _seen.json              # 取得済み記事URLの記録（重複防止）
└── README.md
```

## 運用フェーズ

### Phase 0: ブートストラップ（1回限り）

公式ドキュメント・企業ブログ・コミュニティ記事から、
既に確立されたベストプラクティスを集中投入する。

実行：`routines/phase-0-bootstrap.md` を Routine に登録して手動起動

### Phase 1: 日次更新（継続運用）

毎日 06:00 に直近48時間の新着記事を読み、
ベースラインを差分更新する。

実行：`routines/phase-1-daily.md` を Routine に登録、cron `0 6 * * *`

## プラクティスファイルの読み方

各ファイルは「ルール + 根拠 + コード例 + 出典」のセットで構成される。

### 確信度

- **高**: 公式ドキュメント or 複数の独立ソースで確認済み
- **中**: 1つの権威ある記事で確認
- **低**: 1記事の主張、追加検証推奨

### バージョン

各ルールには対象バージョンが記載される。
例: `Next.js 15+`, `React 19+`, `TypeScript 5.4+`

## PR運用

- 自動起票されるPRは全てドラフト
- レビュー後にマージ
- ラベル: `bootstrap` / `practice-update` / `auto-generated`

## 注意

このリポジトリの内容は自動収集された情報を元にしている。
実プロジェクトへの適用前には、対象プロジェクトの文脈で
妥当性を判断すること。
