# メッセージ管理のベストプラクティス

翻訳メッセージは「文字列ハードコード」と「翻訳者ワークフロー」を両立するため、ICU MessageFormat + 階層化 namespace + 型生成で扱う。

## ルール

### 1. メッセージは ICU MessageFormat で書き、複数形・性別を扱う

「1 件のメッセージ」「2 件のメッセージ」「3 件のメッセージ」のような複数形分岐は、メッセージ自体に `{count, plural, ...}` 構文を埋め込む。
JavaScript 側で if-else しない。

**根拠**:
- 複数形は言語によって規則が大きく異なる（英語: 1 / 2-many、日本語: 区別なし、アラビア語: 0/1/2/few/many/other 6 形式、ロシア語: 3 形式）
- if-else でロジックを組むと多言語化で必ず破綻する
- ICU MessageFormat は Unicode CLDR の plural rules を準拠して扱う標準フォーマット
- FormatJS / next-intl / i18next-icu などほぼ全ての主要 i18n ライブラリが対応

**コード例（ICU MessageFormat）**:
```json
// messages/en.json
{
  "inbox.unread": "{count, plural, =0 {No unread} =1 {1 unread message} other {# unread messages}}",
  "greeting.user": "Hello, {name}!",
  "greeting.gender": "{gender, select, female {She wrote} male {He wrote} other {They wrote}} this article.",
  "delete.confirm": "Delete {count, plural, =1 {this item} other {these # items}}?",
  "date.format": "Published on {date, date, long}",
  "price.format": "{price, number, ::currency/USD}"
}
```

```json
// messages/ja.json
{
  "inbox.unread": "{count, plural, =0 {未読なし} other {未読 #件}}",
  "greeting.user": "{name}さん、こんにちは！",
  "greeting.gender": "{gender, select, other {この人が}}この記事を書きました。",
  "delete.confirm": "{count, plural, other {これら #件}}を削除しますか？",
  "date.format": "{date, date, long} 公開",
  "price.format": "{price, number, ::currency/JPY}"
}
```

**Server Component での使用（next-intl）**:
```tsx
import { getTranslations, getFormatter } from 'next-intl/server';

export default async function InboxPage({ unreadCount }: { unreadCount: number }) {
  const t = await getTranslations('inbox');
  const format = await getFormatter();

  return (
    <div>
      <h2>{t('unread', { count: unreadCount })}</h2>
      {/* unreadCount=0 → "未読なし" / unreadCount=5 → "未読 5件" */}

      <p>{format.dateTime(new Date(), { dateStyle: 'long' })}</p>
    </div>
  );
}
```

**避けるべきパターン**:
```tsx
// Bad: ロジックを JS 側で組む
const message = count === 0
  ? t('no_messages')
  : count === 1
  ? t('one_message')
  : t('many_messages', { count });
// 言語ごとに必要な分岐が違うので破綻する

// Bad: 翻訳キーを動的に組み立てる
t(`messages.count_${count}`);  // count に応じてキーが変わる → 抽出ツールが認識できない

// Good: ICU MessageFormat で 1 キーに集約
t('messages.unread', { count });
```

**MessageFormat の構文サマリ**:
- `{name}` — 単純置換
- `{count, plural, ...}` — 複数形（CLDR 規則）
- `{value, select, ...}` — 値ベースの分岐（性別など）
- `{date, date, long|short|medium}` — 日付フォーマット
- `{n, number, ::currency/USD}` — 数値・通貨フォーマット
- `{n, number, percent}` — パーセンテージ

**出典**:
- [ICU MessageFormat](https://unicode-org.github.io/icu/userguide/format_parse/messages/) (Unicode)
- [next-intl: Messages](https://next-intl-docs.vercel.app/docs/usage/messages) (next-intl)
- [FormatJS: ICU Message Syntax](https://formatjs.io/docs/core-concepts/icu-syntax) (FormatJS)
- [Unicode CLDR: Plural Rules](https://cldr.unicode.org/index/cldr-spec/plural-rules) (Unicode CLDR)

**バージョン**: ICU MessageFormat 標準, next-intl 3+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 2. メッセージキーは階層 namespace（`feature.component.action`）で構造化する

メッセージキーは平坦な文字列ではなく、機能・コンポーネント・アクションの 3 階層で namespace を切る。
フラットな辞書はキー衝突と探しにくさを生む。

**根拠**:
- フラットなキー（`button_submit`）はアプリが大きくなると衝突する
- 階層化（`form.submit`, `dialog.confirm`, `auth.login.title`）で feature ごとに整理できる
- 翻訳者にとっても「どこで使われるか」が推測しやすい
- 型生成で `t('foo.bar.baz')` を補完できるようになる

**推奨構造**:
```json
{
  "common": {
    "buttons": {
      "submit": "送信",
      "cancel": "キャンセル",
      "delete": "削除"
    },
    "validation": {
      "required": "必須項目です",
      "email": "メールアドレスを正しく入力してください"
    }
  },
  "auth": {
    "login": {
      "title": "ログイン",
      "submit": "ログインする",
      "forgotPassword": "パスワードをお忘れですか？"
    },
    "signup": {
      "title": "新規登録",
      "agreeToTerms": "利用規約に同意します"
    }
  },
  "products": {
    "list": {
      "title": "商品一覧",
      "empty": "商品がありません",
      "filter": {
        "all": "すべて",
        "inStock": "在庫あり"
      }
    },
    "detail": {
      "addToCart": "カートに追加",
      "outOfStock": "在庫切れ"
    }
  }
}
```

**避けるべき構造**:
```json
// Bad: フラット（衝突しやすい）
{
  "title": "...",            // どこのタイトル？
  "submit": "...",           // どこの送信？
  "products_list_title": "...",  // _ 区切りでは namespace の意味が薄い
  "PRODUCTS_LIST_TITLE": "..."   // 定数風はキー設計を窮屈にする
}

// Bad: 1 文字列にコンテキストを詰める
{
  "delete.product.from.cart.confirmation.title": "..."  // 階層が深すぎ
}
```

**namespace 切り方の指針**:
- **第 1 階層**: `common` / `auth` / `products` 等の業務ドメイン
- **第 2 階層**: コンポーネント名 / ページ名 / セクション名
- **第 3 階層**: 個別のテキスト（短い名詞 / 動詞）

**next-intl での namespace 使用**:
```tsx
// 階層を namespace で指定して取得
const t = await getTranslations('auth.login');
t('title');           // → 'auth.login.title'
t('forgotPassword');  // → 'auth.login.forgotPassword'

// 必要な namespace だけ Provider に渡して bundle 削減
<NextIntlClientProvider
  messages={pick(messages, ['common.buttons', 'auth.login'])}
>
  <LoginForm />
</NextIntlClientProvider>
```

**翻訳の分割（モノレポでの運用）**:
```
i18n/
├── messages/
│   ├── ja/
│   │   ├── common.json
│   │   ├── auth.json
│   │   └── products.json
│   └── en/
│       ├── common.json
│       ├── auth.json
│       └── products.json
└── request.ts  # 動的に namespace を読み込んで merge
```

**出典**:
- [next-intl: Message structure](https://next-intl-docs.vercel.app/docs/usage/messages#nested-messages) (next-intl)
- [i18next: Namespace](https://www.i18next.com/principles/namespaces) (i18next)

**バージョン**: パターン（実装依存）
**確信度**: 高
**最終更新**: 2026-05-16

---

### 3. メッセージファイルは外部翻訳ツール（Crowdin / Lokalise / Phrase）連携を前提に設計する

翻訳ファイルは `JSON` または `Format.js JSON / ARB`（Flutter 系）で書き、CAT ツール（Computer-Assisted Translation）と連携できる構造にする。
翻訳者は Git を使えない前提で、外部 UI で編集できるパスを用意する。

**根拠**:
- 翻訳者は技術者ではない。直接 JSON を編集させると JSON 構文エラーが頻発
- CAT ツール（Crowdin / Lokalise / Phrase / Localazy）は翻訳メモリ・用語集・進捗管理を提供
- ツール側で承認された翻訳が PR で自動的に降りてくる運用が業界標準
- ARB / XLIFF など標準フォーマットを使えばツール乗り換えコストが下がる

**主要な選択肢**:

| ツール | 強み | 料金 |
|---|---|---|
| **Crowdin** | 無料枠あり、GitHub 統合、業界スタンダード | 無料〜 |
| **Lokalise** | UI 良好、TypeScript 型生成サポート | 有料（無料枠なし） |
| **Phrase** | 大企業向け、用語管理が強力 | 有料 |
| **Localazy** | 開発者向け、無料枠が大きい | 無料〜 |
| **Tolgee** | OSS / self-host 可、in-context 編集 | OSS 無料 |
| **Transifex** | 長年の実績、CLI 強力 | 有料 |
| **Weblate** | OSS / self-host、Linux Foundation 採用 | OSS 無料 |

**運用フロー（Crowdin の場合）**:
```yaml
# crowdin.yml
project_id_env: CROWDIN_PROJECT_ID
api_token_env: CROWDIN_API_TOKEN

files:
  - source: /messages/en.json
    translation: /messages/%two_letters_code%.json
    update_option: update_as_unapproved
```

```yaml
# .github/workflows/crowdin.yml
name: Crowdin Sync
on:
  push:
    branches: [main]
    paths: ['messages/en.json']
  schedule:
    - cron: '0 0 * * *'  # 翻訳完了を毎日取り込む

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: crowdin/github-action@v1
        with:
          upload_sources: true
          download_translations: true
          create_pull_request: true
          localization_branch_name: i18n/crowdin
        env:
          CROWDIN_PROJECT_ID: ${{ secrets.CROWDIN_PROJECT_ID }}
          CROWDIN_API_TOKEN: ${{ secrets.CROWDIN_API_TOKEN }}
```

**翻訳ワークフロー**:
1. 開発者が英語（master）でメッセージを追加 → push
2. Crowdin が自動で取り込み、未翻訳項目として表示
3. 翻訳者が Crowdin UI で翻訳
4. 承認された翻訳が PR で main に降りてくる
5. レビュー後マージ

**メッセージ追加のチェックリスト**:
- [ ] master 言語（通常 en）で書く
- [ ] ICU 構文を使う（複数形・性別）
- [ ] 翻訳者向けコメントを付ける（CAT ツールに表示される）
- [ ] スクリーンショット URL を付ける（context が伝わる）
- [ ] 文字数制限を明記（UI 制約がある場合）

**翻訳者向けコメントの埋め込み**（ICU メッセージのコメント）:
```json
// 多くのツールが拡張属性として扱う
{
  "products.detail.addToCart": {
    "defaultMessage": "Add to Cart",
    "description": "Button label on product detail page",
    "screenshots": ["https://figma.com/proto/..."],
    "maxLength": 20
  }
}
```

**出典**:
- [Crowdin Docs](https://support.crowdin.com/) (Crowdin)
- [Lokalise Docs](https://docs.lokalise.com/) (Lokalise)
- [Tolgee](https://tolgee.io/) (Tolgee)
- [FormatJS: Translation Management](https://formatjs.io/docs/getting-started/message-distribution) (FormatJS)

**バージョン**: パターン（ツール依存）
**確信度**: 高
**最終更新**: 2026-05-16

---

### 4. メッセージキーは TypeScript 型レベルで補完・検証する

`useTranslations('home')` の引数や `t('title')` のキーを TypeScript で補完・検証可能にする。
キーの typo / 削除済みキーへの参照を実行時ではなくコンパイル時に検出する。

**根拠**:
- メッセージキーの typo は実行時まで気付かない（i18n ライブラリは undefined キーをサイレントに `key` 表示する）
- `next-intl` / `react-i18next` のどちらも TypeScript declaration で型補完できる
- 不要になったキーを検出する `unused messages` linter でメッセージファイルが肥大化するのを防ぐ
- 型生成は CI に組み込めば翻訳ファイル変更時に自動更新できる

**next-intl での型補完**:
```ts
// global.d.ts または next-intl.d.ts
import { type messages } from './i18n/messages/en.json';

type Messages = typeof messages;

declare module 'next-intl' {
  interface AppConfig {
    Messages: Messages;
  }
}
```

これで以下のように補完が効く:
```tsx
const t = useTranslations('home');
t('titl');  // ← compile error: 'titl' は 'home' namespace に存在しない
t('title'); // ← OK
```

**`paraglide-js` の場合**（型生成が標準）:
```ts
// 翻訳キーごとに関数が生成される
import * as m from '$paraglide/messages';

<h1>{m.home_title()}</h1>
<p>{m.inbox_unread({ count: 5 })}</p>
// 型エラー: m.home_titl() は存在しない
```

**未使用メッセージ検出**:
```bash
# next-intl
npx next-intl-cli detect-unused

# react-i18next
npx i18next-parser --check

# 自前スクリプト
npm run build  # bundle 内のメッセージキー使用を検証
```

**実装例（自前スクリプト）**:
```ts
// scripts/check-i18n.ts
import { readFile, glob } from 'node:fs/promises';

const messages = JSON.parse(await readFile('messages/en.json', 'utf-8'));
const usedKeys = new Set<string>();

// .tsx ファイルを grep してキー使用箇所を集める
for await (const file of glob('**/*.{ts,tsx}')) {
  const content = await readFile(file, 'utf-8');
  for (const match of content.matchAll(/t\(['"]([^'"]+)['"]/g)) {
    usedKeys.add(match[1]);
  }
}

// 定義済みだが未使用のキーを列挙
function flatten(obj: any, prefix = ''): string[] {
  return Object.entries(obj).flatMap(([k, v]) =>
    typeof v === 'object' ? flatten(v, `${prefix}${k}.`) : [`${prefix}${k}`],
  );
}

const definedKeys = flatten(messages);
const unused = definedKeys.filter((k) => !usedKeys.has(k));
console.log('Unused:', unused);
```

**翻訳カバレッジの可視化**:
```bash
# 各言語の翻訳カバレッジを数える
for lang in messages/*.json; do
  total=$(jq '[paths(scalars)] | length' messages/en.json)
  translated=$(jq '[paths(scalars)] | length' "$lang")
  echo "$lang: $translated/$total ($(echo "scale=2; $translated*100/$total" | bc)%)"
done
```

**`Intl.Locale` API で BCP 47 を検証**:
```ts
function isValidLocale(tag: string): boolean {
  try {
    new Intl.Locale(tag);
    return true;
  } catch {
    return false;
  }
}
```

**出典**:
- [next-intl: TypeScript integration](https://next-intl-docs.vercel.app/docs/workflows/typescript) (next-intl)
- [react-i18next: TypeScript](https://react.i18next.com/latest/typescript) (i18next)
- [paraglide-js](https://inlang.com/m/gerre34r/library-inlang-paraglideJs) (inlang)

**バージョン**: TypeScript 5+, next-intl 3+
**確信度**: 高
**最終更新**: 2026-05-16
