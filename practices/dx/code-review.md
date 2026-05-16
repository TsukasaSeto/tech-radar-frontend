# コードレビューのベストプラクティス

PR レビューはチームの最重要コミュニケーション。レビュアー / 著者の負担を最小化しつつ、品質を担保するルールを設計する。

## ルール

### 1. PR タイトルは Conventional Commits 形式、ラベルでカテゴリを示す

PR タイトルは `<type>(<scope>): <summary>` 形式（Conventional Commits）で統一する。
`feat` / `fix` / `chore` / `docs` / `refactor` / `test` / `perf` / `build` / `ci` / `style` のいずれか。

**根拠**:
- 一目で「機能追加 / バグ修正 / リファクタ」が分かる
- 自動 changelog 生成・semantic-release との統合が容易
- レビュー優先度の判断材料になる（`fix(security)` は最優先 等）
- GitHub の検索で `type:bug` のように絞り込める

**フォーマット**:
```
<type>(<scope>)?: <summary>

[optional body]

[optional footer]
```

| type | 用途 |
|---|---|
| `feat` | 新機能（ユーザー向け） |
| `fix` | バグ修正 |
| `chore` | ビルド・依存・運用変更 |
| `docs` | ドキュメント |
| `refactor` | リファクタリング（機能変更なし） |
| `test` | テスト追加・修正 |
| `perf` | パフォーマンス改善 |
| `build` | ビルドシステム変更 |
| `ci` | CI 設定変更 |
| `style` | フォーマッタ・lint（機能変更なし） |
| `revert` | 過去コミットの取り消し |

**例**:
```
feat(auth): add password reset flow
fix(checkout): prevent double submission on slow networks
chore(deps): bump react to 18.3.1
refactor(api): extract retry logic into shared hook
perf(images): switch to AVIF for hero images
```

**Breaking change の表示**:
```
feat(api)!: change response format for /users endpoint

BREAKING CHANGE: Response is now { data: User[], pagination } instead of User[].
Migration: replace `data.map(...)` with `data.data.map(...)`.
```

**GitHub Actions で強制**:
```yaml
# .github/workflows/lint-pr-title.yml
name: Lint PR Title
on:
  pull_request:
    types: [opened, edited, reopened, synchronize]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: amannn/action-semantic-pull-request@v5
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          types: |
            feat
            fix
            chore
            docs
            refactor
            test
            perf
            build
            ci
            style
            revert
          scopes: |
            auth
            checkout
            api
            ui
            deps
          requireScope: false
          subjectPattern: ^[A-Z].+[^.]$
          subjectPatternError: |
            Subject must start with an uppercase letter and not end with a period.
```

**ラベル運用**:
```yaml
# .github/labeler.yml — pathベースで自動ラベリング
"area: auth":
  - changed-files:
      - any-glob-to-any-file: ['src/features/auth/**']

"area: ci":
  - changed-files:
      - any-glob-to-any-file: ['.github/workflows/**']

"dependencies":
  - changed-files:
      - any-glob-to-any-file: ['package.json', 'pnpm-lock.yaml']
```

```yaml
# .github/workflows/labeler.yml
name: PR Labeler
on:
  pull_request_target:
    types: [opened, synchronize]

jobs:
  label:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/labeler@v5
        with:
          repo-token: ${{ secrets.GITHUB_TOKEN }}
```

**判定軸**:
- PR タイトルが Conventional Commits ＝ release-notes 自動生成と直結
- ラベルは検索 / 優先度判定 / 自動ルーティング（CODEOWNERS）で活用
- scope は省略可だが、大規模リポジトリでは推奨

**出典**:
- [Conventional Commits](https://www.conventionalcommits.org/) (Conventional Commits)
- [amannn/action-semantic-pull-request](https://github.com/amannn/action-semantic-pull-request) (amannn)
- [actions/labeler](https://github.com/actions/labeler) (GitHub)

**バージョン**: パターン
**確信度**: 高
**最終更新**: 2026-05-16

---

### 2. レビュー観点を「正しさ・可読性・保守性・性能」の 4 軸で構造化する

PR レビューは思いつきで指摘せず、4 つの観点を順番にチェックする。
これにより重要な抜けを防ぎ、レビューの粒度が安定する。

**根拠**:
- レビューを「読み流して気付いた点を書く」だけだと観点が偏る
- 4 軸でチェックリスト化すれば、新メンバーのレビュー品質も均一化できる
- 「正しさ」が最優先、その後に他の軸を見ることで重要な指摘を見逃さない
- 1 PR で全観点を満たすのが理想だが、scope を限定するなら "正しさ + 可読性" だけでも OK

**4 軸チェックリスト**:

#### 1. 正しさ（Correctness）
- [ ] 仕様通りに動作するか（issue / spec と一致）
- [ ] エッジケース（null / undefined / empty / 大量データ）を考慮しているか
- [ ] エラーハンドリングが妥当か（throw / Result / silent fail の選択）
- [ ] 型が緩すぎないか（`any` / `as` の濫用）
- [ ] テストカバレッジが十分か（happy path + 主要なエラーパス）
- [ ] レース条件・無限ループ・メモリリークがないか

#### 2. 可読性（Readability）
- [ ] 命名が直感的か（`naming.md` ルール準拠）
- [ ] 関数 / コンポーネントの責任が単一か
- [ ] ネストが深すぎないか（3 階層を超えたら抽出を検討）
- [ ] 1 ファイル / 1 関数が大きすぎないか
- [ ] コメントは「なぜ」を説明しているか（「何を」はコードで自明にする）

#### 3. 保守性（Maintainability）
- [ ] 既存パターンと整合しているか（既存コードとの一貫性）
- [ ] 依存関係の方向が正しいか（layer 違反がないか）
- [ ] 暗黙的な前提（implicit knowledge）が文書化されているか
- [ ] 「次の人が変更しやすい」構造か
- [ ] 不要な抽象化がないか（YAGNI 違反）

#### 4. 性能（Performance）
- [ ] N+1 クエリ / ウォーターフォール fetch がないか
- [ ] 不要な re-render（`useMemo` / `useCallback` の使い所）がないか
- [ ] バンドルサイズへの影響（大きなライブラリの追加）
- [ ] Core Web Vitals への影響（LCP 画像・INP の重い処理）

**運用ルール**:
```markdown
## PR レビューフロー

1. **PR 著者**: PR 説明欄に "Self-review" 項目を入れて自己チェック
2. **レビュアー**: 4 軸の順番でチェック
   - 「正しさ」で blocker があれば、他の軸は見ずに即座に指摘
   - 「可読性」「保守性」「性能」は nit / suggestion / blocker のいずれかで marking
3. **PR 著者**: 全 blocker を解消 → re-request review
4. **レビュアー**: blocker 解消を確認 → approve
```

**指摘の粒度**（後述 Rule 3）と組み合わせて運用。

**1 PR あたりの目安サイズ**:
- < 200 行: 高速レビュー、blocker なしで approve しやすい
- 200-500 行: 通常のレビュー
- 500-1000 行: レビュー疲れが出る、分割を推奨
- > 1000 行: 分割必須（テスト・スタイル変更を別 PR に）

**著者のセルフレビュー（PR 作成前）**:
```markdown
## Self-review checklist (in PR description)
- [ ] テストが追加 / 更新されている
- [ ] 型エラーが解消している
- [ ] CI が green
- [ ] 関連ドキュメント（CLAUDE.md / README）を更新した
- [ ] スクリーンショット / 動画を添付した（UI 変更時）
- [ ] レビュアーがチェックすべき箇所をコメントで明示した
```

**出典**:
- [Google: Engineering Practices - Code Review](https://google.github.io/eng-practices/review/) (Google)
- [Conventional Comments](https://conventionalcomments.org/) (Conventional Comments)

**バージョン**: パターン
**確信度**: 高
**最終更新**: 2026-05-16

---

### 3. コメントは `nit:` / `suggestion:` / `blocker:` で重要度を明示する

レビューコメントの先頭にラベル（`nit:` / `suggestion:` / `blocker:`）を付け、対応の緊急度を伝える。
すべての指摘を merge 阻害要因と扱うとレビュースピードが落ちる。

**根拠**:
- 「`button` のクラス名を変えてほしい」と「`unauthorized access` が通る」は緊急度がまったく違う
- ラベルなしだと著者は全コメントを blocker と解釈しがち
- Conventional Comments という業界スタンダードがあり、いくつかのチームで実証済み
- 過剰な fix 要求はレビュー疲れを招き、本質的な指摘の質を下げる

**ラベル一覧**:

| ラベル | 意味 | 対応 |
|---|---|---|
| `blocker:` | merge 前に必須対応 | 修正 or 議論 |
| `issue:` | 重要だが議論余地あり | 議論 → 修正 or 棚上げ |
| `suggestion:` | こうしたほうが良い | 検討して採否 |
| `nit:` | 些細な改善（ピックアップ） | 著者の判断で対応 |
| `question:` | 確認 / 理解のため | 回答必須、コード変更は任意 |
| `praise:` | 良い実装への賞賛 | 対応不要、士気向上 |
| `thought:` | レビュアーの思考共有 | 対応不要 |

**コメント例**:
```markdown
blocker: ここで `await` が抜けています。Promise が return されないため
exception がレース条件で消えます。

suggestion: この関数は 50 行を超えていて 3 つの責任を担っています。
`validateInput` / `transformData` / `savePost` に分割すると test が書きやすくなりそうです。

nit: `userdata` のスペースを `userData` にしませんか？このファイル
内では camelCase で揃っているようです。

question: この feature flag のロールアウト計画はありますか？
PR 説明には書いてなかったので確認です。

praise: この `useCallback` の依存配列、めちゃくちゃ整理されていて
読みやすいです。

thought: 似たような pagination logic を `<UserList>` でも書いていた
気がします。共通化は次の PR で検討しても良さそう（今 PR ではスキップ OK）。
```

**「決定」の明示**（議論が紛糾した時の収束）:
```markdown
decision: Tech Lead として、ここは `Result<T, E>` パターンで統一する
方針にします（[architecture/error-handling.md](.../error-handling.md) 参照）。
今 PR ではこの 1 箇所だけ修正、残りは別 PR で。
```

**過剰なレビュー疲れを防ぐルール**:
- 1 PR で `blocker:` が 5 個以上 → PR 分割の合図
- `nit:` だらけのレビュー → ESLint / Prettier で自動化
- 同じ指摘が複数の PR で繰り返される → ドキュメント化 / 自動化を検討

**著者の対応**:
```markdown
> blocker: ここで `await` が抜けています...

ご指摘ありがとうございます。修正しました（commit abc1234）。

> suggestion: この関数は 50 行を超えていて...

✓ 分割しました（commit def5678）。

> nit: `userdata` のスペースを `userData` に...

✓ 直しました。

> question: feature flag のロールアウト計画は...

✓ 1% → 10% → 50% → 100% の段階展開を予定しています。詳細は
[#1234](https://...) を参照。今 PR では flag を `default: off` で
merge し、来週ロールアウトを開始します。
```

**自動チェックを優先する**:
- スペース / インデント / セミコロン → Prettier
- import 順序 / 未使用変数 → ESLint
- 型エラー → TypeScript
- 命名規約 → ESLint plugin

これらが自動化されていればレビュアーが指摘する手間が減り、人間は本質的な判断に集中できる。

**出典**:
- [Conventional Comments](https://conventionalcomments.org/) (Conventional Comments)
- [Google: Code Review Comments](https://google.github.io/eng-practices/review/reviewer/comments.html) (Google)

**バージョン**: パターン
**確信度**: 高
**最終更新**: 2026-05-16

---

### 4. レビュー自動化（reviewdog / danger）で ルーチン指摘を機械化する

「lint 失敗」「テスト不足」「PR description 未記入」のような機械的に判定できる指摘は自動化する。
人間のレビュアーは「判断が必要な部分」だけに集中させる。

**根拠**:
- 同じ指摘を毎 PR で書くのはレビュアー疲れと著者ストレスを両方招く
- 自動化できる指摘を機械的にコメントすれば、人間レビュアーの認知負荷が下がる
- `reviewdog` / `danger` は GitHub PR コメントとして lint 結果を投稿できる
- ルールが明文化される（pre-commit hook より「PR で見える」のが大事）

**reviewdog の設定**:
```yaml
# .github/workflows/reviewdog.yml
name: reviewdog
on: [pull_request]

jobs:
  eslint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: pnpm }
      - run: pnpm install --frozen-lockfile

      - uses: reviewdog/action-eslint@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          reporter: github-pr-review  # PR にインラインコメント
          eslint_flags: 'src/**/*.{ts,tsx}'
          fail_on_error: true

  textlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: tsuyoshicho/action-textlint@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          reporter: github-pr-review
          filter_mode: added  # 新規追加行のみ
          textlint_flags: 'docs/**/*.md'
```

**Danger.js の設定**（より柔軟な PR チェック）:
```ts
// dangerfile.ts
import { danger, warn, fail, message } from 'danger';

// PR description が空ならエラー
if (!danger.github.pr.body || danger.github.pr.body.length < 30) {
  fail('PR description が短すぎます。何を / なぜ / どのようにテストしたかを書いてください。');
}

// 大きすぎる PR は warn
const totalChanges = danger.github.pr.additions + danger.github.pr.deletions;
if (totalChanges > 600) {
  warn(`この PR は ${totalChanges} 行の変更を含んでいます。分割を検討してください。`);
}

// テストが追加されていない feature PR は warn
const hasSourceChanges = danger.git.modified_files.some((f) => f.startsWith('src/') && !f.includes('.test.'));
const hasTestChanges = danger.git.modified_files.some((f) => f.includes('.test.'));
if (hasSourceChanges && !hasTestChanges) {
  warn('ソースファイルが変更されていますがテストが更新されていません。');
}

// package.json の変更があれば必ず lockfile も更新されていること
if (danger.git.modified_files.includes('package.json') && !danger.git.modified_files.includes('pnpm-lock.yaml')) {
  fail('package.json が変更されていますが pnpm-lock.yaml が更新されていません。`pnpm install` を実行してください。');
}

// .env.example の変更時はリリースノートに記載が必要
if (danger.git.modified_files.includes('.env.example')) {
  message('🔔 環境変数が変更されました。リリースノートとデプロイ手順を確認してください。');
}

// CHANGELOG / RELEASE NOTES が更新されているか
if (hasSourceChanges && !danger.git.modified_files.some((f) => f.includes('changeset'))) {
  warn('機能変更には changeset の追加が必要です。`pnpm changeset` を実行してください。');
}
```

```yaml
# .github/workflows/danger.yml
- uses: danger/danger-js@v12
  env:
    DANGER_GITHUB_API_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**よく自動化される指摘**:

| 指摘 | 自動化手段 |
|---|---|
| ESLint / Prettier 違反 | reviewdog + ESLint |
| TypeScript エラー | reviewdog + tsc |
| 大きすぎる PR | Danger.js |
| テスト未追加 | Danger.js |
| Conventional Commits 違反 | action-semantic-pull-request |
| TODO コメント残し | Danger.js（regex） |
| `console.log` 残し | ESLint (`no-console`) |
| 翻訳キーの typo | textlint + custom rule |
| package.json + lockfile 不整合 | Danger.js |
| 機微情報の commit | gitleaks + reviewdog |
| 視覚的差分 | Chromatic / Percy（PR コメントに自動投稿） |

**指針**:
- 同じ指摘を 3 回以上したら自動化を検討する
- 機械的に検出できる指摘は人間が書かない（時間の無駄）
- ただし「自動化のためのスクリプトメンテナンス」が割に合うか定期的に評価

**出典**:
- [reviewdog](https://github.com/reviewdog/reviewdog) (reviewdog)
- [Danger.js](https://danger.systems/js/) (Danger)

**バージョン**: reviewdog v0.20+, Danger.js v12+
**確信度**: 高
**最終更新**: 2026-05-16
