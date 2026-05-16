# 日時・数値・通貨ローカライズのベストプラクティス

ECMAScript Internationalization API（`Intl.*`）を活用して、日時・数値・通貨・相対時間・並び替えをロケール対応させる。
独自フォーマット関数を書かない。

## ルール

### 1. 日時は `Intl.DateTimeFormat` で locale に応じて表示する

`new Date().toLocaleString()` または `Intl.DateTimeFormat` を使い、文字列を手動で組み立てない。
ロケールごとの順序（YYYY/MM/DD vs MM/DD/YYYY vs DD/MM/YYYY）・区切り文字・月名を自動処理する。

**根拠**:
- 日付の表示順序は文化圏ごとに大きく異なる（米国: M/D/Y、欧州: D/M/Y、ISO: Y-M-D）
- 月名・曜日名は当然言語別、`Intl.DateTimeFormat` は Unicode CLDR データで全主要言語を提供
- タイムゾーンも `timeZone: 'Asia/Tokyo'` で明示できる
- `dateStyle` / `timeStyle` プリセットで「ロケール標準の表示」を簡単に得られる

**コード例**:
```ts
const date = new Date('2026-05-16T18:30:00Z');

// プリセット
new Intl.DateTimeFormat('ja-JP', { dateStyle: 'long' }).format(date);
// → "2026年5月17日"（タイムゾーン UTC→JST）

new Intl.DateTimeFormat('en-US', { dateStyle: 'long' }).format(date);
// → "May 16, 2026"

new Intl.DateTimeFormat('de-DE', { dateStyle: 'long' }).format(date);
// → "16. Mai 2026"

new Intl.DateTimeFormat('ar-EG', { dateStyle: 'long' }).format(date);
// → "١٦ مايو ٢٠٢٦"

// カスタム
new Intl.DateTimeFormat('ja-JP', {
  year: 'numeric',
  month: '2-digit',
  day: '2-digit',
  hour: '2-digit',
  minute: '2-digit',
  timeZone: 'Asia/Tokyo',
}).format(date);
// → "2026/05/17 03:30"

// 範囲
const start = new Date('2026-05-16');
const end = new Date('2026-05-18');
new Intl.DateTimeFormat('ja-JP', { dateStyle: 'long' }).formatRange(start, end);
// → "2026年5月16日～18日"
```

**`dateStyle` / `timeStyle` の値**:
| 値 | ja-JP 例 | en-US 例 |
|---|---|---|
| `full` | "2026年5月17日土曜日" | "Saturday, May 17, 2026" |
| `long` | "2026年5月17日" | "May 17, 2026" |
| `medium` | "2026/05/17" | "May 17, 2026" |
| `short` | "2026/5/17" | "5/17/26" |

**注意点**:
- `Date` オブジェクトは UTC 内部表現。`Intl.DateTimeFormat` は **クライアントのローカルタイムゾーン** で表示する
- サーバーレンダリング（SSR）と Hydration で `timeZone` を明示しないと表示がズレる
- SSR では必ず `timeZone: 'UTC'` または `timeZone: 'Asia/Tokyo'` 等を明示

**Next.js Server Component での SSR ハイドレーション対応**:
```tsx
// Bad: クライアントのタイムゾーンに依存（hydration mismatch）
const formatted = new Intl.DateTimeFormat('ja-JP', { dateStyle: 'long' }).format(date);

// Good: 明示的にタイムゾーンを指定
const formatted = new Intl.DateTimeFormat('ja-JP', {
  dateStyle: 'long',
  timeZone: 'Asia/Tokyo',
}).format(date);
```

**`next-intl` での format**:
```tsx
import { getFormatter } from 'next-intl/server';

const format = await getFormatter();
format.dateTime(date, { dateStyle: 'long' });
// → ロケール + Provider の設定済み timeZone で自動フォーマット
```

**よくある間違い**:
```ts
// Bad: 文字列スプリットで「年月日」を組み立てる
const formatted = `${date.getFullYear()}年${date.getMonth() + 1}月${date.getDate()}日`;
// → ロケール変更時に書き換えが必要、月名が出ない、UTC 計算で日付ズレ

// Bad: dayjs / date-fns で format string を書く
dayjs(date).format('YYYY年MM月DD日');
// → ロケール文字列が固定、別言語で出すには差し替えが必要
```

**dayjs / date-fns との関係**:
- `Intl.DateTimeFormat` は **表示** に特化
- `dayjs` / `date-fns` は **計算**（add / subtract / diff）に特化
- 両方使い分け：計算は date-fns、表示は `Intl`

**出典**:
- [MDN: Intl.DateTimeFormat](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat) (MDN Web Docs)
- [ECMA-402: DateTimeFormat](https://tc39.es/ecma402/#datetimeformat-objects) (TC39)
- [Unicode CLDR](https://cldr.unicode.org/) (Unicode)

**バージョン**: 全モダンブラウザ
**確信度**: 高
**最終更新**: 2026-05-16

---

### 2. 数値・通貨は `Intl.NumberFormat` で表示する

`Intl.NumberFormat` で桁区切り・小数点・通貨記号・パーセンテージをロケール対応させる。
通貨は ISO 4217 通貨コード（USD / JPY / EUR）を使う。

**根拠**:
- 桁区切り文字はロケールで異なる（`1,234.56` (en) / `1.234,56` (de) / `1 234,56` (fr) / `١٬٢٣٤٫٥٦` (ar)）
- 通貨記号の位置も異なる（`$1,234` (en) / `1.234 €` (de) / `1,234円` (ja)）
- 小数点表記も異なる（`.` / `,`）
- `compactDisplay: 'short'` で「1.2M / 1.2万」のような短縮表示が可能

**コード例**:
```ts
// 基本
new Intl.NumberFormat('ja-JP').format(1234567.89);  // → "1,234,567.89"
new Intl.NumberFormat('de-DE').format(1234567.89);  // → "1.234.567,89"
new Intl.NumberFormat('fr-FR').format(1234567.89);  // → "1 234 567,89"
new Intl.NumberFormat('hi-IN').format(1234567.89);  // → "12,34,567.89"（インド方式）

// 通貨
new Intl.NumberFormat('ja-JP', { style: 'currency', currency: 'JPY' }).format(1234);
// → "￥1,234"

new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(1234.5);
// → "$1,234.50"

new Intl.NumberFormat('de-DE', { style: 'currency', currency: 'EUR' }).format(1234.5);
// → "1.234,50 €"

// パーセンテージ
new Intl.NumberFormat('ja-JP', { style: 'percent', minimumFractionDigits: 1 }).format(0.1234);
// → "12.3%"

// 単位
new Intl.NumberFormat('ja-JP', { style: 'unit', unit: 'kilometer-per-hour' }).format(60);
// → "60 km/h"

// コンパクト表記
new Intl.NumberFormat('en-US', { notation: 'compact', compactDisplay: 'short' }).format(1234567);
// → "1.2M"

new Intl.NumberFormat('ja-JP', { notation: 'compact', compactDisplay: 'short' }).format(1234567);
// → "123万"

// 桁数制御
new Intl.NumberFormat('en-US', {
  minimumFractionDigits: 2,
  maximumFractionDigits: 4,
}).format(1.5);  // → "1.50"
```

**通貨の表示形式**:
```ts
// 標準表示
new Intl.NumberFormat('en-US', { style: 'currency', currency: 'JPY' }).format(1000);
// → "¥1,000"

// currencyDisplay
new Intl.NumberFormat('en-US', { style: 'currency', currency: 'JPY', currencyDisplay: 'code' }).format(1000);
// → "JPY 1,000"

new Intl.NumberFormat('en-US', { style: 'currency', currency: 'JPY', currencyDisplay: 'name' }).format(1000);
// → "1,000 Japanese yen"

new Intl.NumberFormat('en-US', { style: 'currency', currency: 'JPY', currencyDisplay: 'narrowSymbol' }).format(1000);
// → "¥1,000"
```

**「会計」モード（負数を括弧で）**:
```ts
new Intl.NumberFormat('en-US', {
  style: 'currency',
  currency: 'USD',
  currencySign: 'accounting',
}).format(-1234);
// → "($1,234.00)"
```

**Decimal / BigInt との組み合わせ**:
```ts
// 浮動小数点誤差を避けたい場合は decimal.js / bignumber.js で計算
import Decimal from 'decimal.js';

const value = new Decimal('1234.5678');
new Intl.NumberFormat('ja-JP').format(value.toNumber());
// → "1,234.568"

// BigInt
new Intl.NumberFormat('ja-JP').format(BigInt('12345678901234567890'));
// → "12,345,678,901,234,567,890"
```

**`next-intl` での format**:
```tsx
const format = useFormatter();
format.number(1234.5, { style: 'currency', currency: 'JPY' });
```

**メッセージ内での埋め込み（ICU MessageFormat）**:
```json
{
  "price": "価格: {amount, number, ::currency/JPY}",
  "discount": "割引: {rate, number, ::percent}"
}
```

**出典**:
- [MDN: Intl.NumberFormat](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat) (MDN Web Docs)
- [ECMA-402: NumberFormat](https://tc39.es/ecma402/#numberformat-objects) (TC39)
- [ISO 4217: Currency Codes](https://www.iso.org/iso-4217-currency-codes.html) (ISO)

**バージョン**: 全モダンブラウザ
**確信度**: 高
**最終更新**: 2026-05-16

---

### 3. 相対時間は `Intl.RelativeTimeFormat` で表示する

「3 日前」「2 時間後」「先週」のような相対時間は `Intl.RelativeTimeFormat` で表示する。
独自関数で「X 日前」を計算しない。

**根拠**:
- 相対時間表現は言語依存（複数形・性別・微妙な単位表現）
- 「now」「yesterday」「tomorrow」のような特殊表現も自動的に出る
- `numeric: 'auto'` で「1 日前」を「昨日」に自動置換する機能あり
- moment.js / dayjs の `fromNow()` は同等機能だがネイティブ API のほうが軽量

**コード例**:
```ts
// 過去
new Intl.RelativeTimeFormat('ja-JP', { numeric: 'auto' }).format(-1, 'day');
// → "昨日"

new Intl.RelativeTimeFormat('ja-JP', { numeric: 'auto' }).format(-3, 'day');
// → "3 日前"

new Intl.RelativeTimeFormat('en-US', { numeric: 'auto' }).format(-1, 'day');
// → "yesterday"

new Intl.RelativeTimeFormat('en-US', { numeric: 'auto' }).format(-3, 'day');
// → "3 days ago"

// 未来
new Intl.RelativeTimeFormat('ja-JP', { numeric: 'auto' }).format(1, 'hour');
// → "1 時間後"

new Intl.RelativeTimeFormat('en-US', { numeric: 'auto' }).format(2, 'week');
// → "in 2 weeks"

// numeric: 'always' で「昨日」ではなく「1 日前」を強制
new Intl.RelativeTimeFormat('ja-JP', { numeric: 'always' }).format(-1, 'day');
// → "1 日前"

// style
new Intl.RelativeTimeFormat('en-US', { style: 'long' }).format(-1, 'day');   // "1 day ago"
new Intl.RelativeTimeFormat('en-US', { style: 'short' }).format(-1, 'day');  // "1 day ago"
new Intl.RelativeTimeFormat('en-US', { style: 'narrow' }).format(-1, 'day'); // "1d ago"
```

**単位の選び方（時刻差から自動決定）**:
```ts
function formatRelative(date: Date, locale: string): string {
  const rtf = new Intl.RelativeTimeFormat(locale, { numeric: 'auto' });
  const diff = (date.getTime() - Date.now()) / 1000;  // 秒
  const absDiff = Math.abs(diff);

  if (absDiff < 60) return rtf.format(Math.round(diff), 'second');
  if (absDiff < 3600) return rtf.format(Math.round(diff / 60), 'minute');
  if (absDiff < 86400) return rtf.format(Math.round(diff / 3600), 'hour');
  if (absDiff < 86400 * 7) return rtf.format(Math.round(diff / 86400), 'day');
  if (absDiff < 86400 * 30) return rtf.format(Math.round(diff / (86400 * 7)), 'week');
  if (absDiff < 86400 * 365) return rtf.format(Math.round(diff / (86400 * 30)), 'month');
  return rtf.format(Math.round(diff / (86400 * 365)), 'year');
}

formatRelative(new Date(Date.now() - 30000), 'ja-JP');         // → "30 秒前"
formatRelative(new Date(Date.now() - 24 * 60 * 60 * 1000), 'ja-JP'); // → "昨日"
formatRelative(new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), 'ja-JP'); // → "来週"
```

**React コンポーネントとしての実装**:
```tsx
'use client';
import { useEffect, useState } from 'react';
import { useLocale } from 'next-intl';

function RelativeTime({ date }: { date: Date }) {
  const locale = useLocale();
  const [, refresh] = useState(0);

  // 1 分ごとに再計算（タイムリーな更新）
  useEffect(() => {
    const id = setInterval(() => refresh((n) => n + 1), 60000);
    return () => clearInterval(id);
  }, []);

  return <time dateTime={date.toISOString()}>{formatRelative(date, locale)}</time>;
}
```

**SSR で固定タイムスタンプを返す**:
```tsx
// Server Component
import { getTranslations } from 'next-intl/server';

export default async function CommentList({ comments }) {
  // SSR では絶対時刻、クライアントで相対時刻に置き換える
  return (
    <ul>
      {comments.map((c) => (
        <li key={c.id}>
          <RelativeTime date={new Date(c.createdAt)} fallback={c.createdAt} />
        </li>
      ))}
    </ul>
  );
}
```

**出典**:
- [MDN: Intl.RelativeTimeFormat](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/RelativeTimeFormat) (MDN Web Docs)
- [ECMA-402: RelativeTimeFormat](https://tc39.es/ecma402/#relativetimeformat-objects) (TC39)

**バージョン**: Chrome 71+, Firefox 65+, Safari 14+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 4. 並び替え・検索は `Intl.Collator` でロケール対応する

文字列の sort / 検索 / 比較は `Array.prototype.sort()` ではなく `Intl.Collator` を使う。
ロケールごとの文字順序（ドイツ語の ß, スウェーデン語の å, 日本語の濁音）を正しく扱う。

**根拠**:
- 標準の `String.prototype.localeCompare()` も `Intl.Collator` を内部で使うが、`Intl.Collator` のインスタンスを再利用するほうが高速
- ドイツ語の `ß` は他言語と扱いが異なる
- 日本語のかな・カタカナ・濁音半濁音の順序は CLDR データで規定
- 大文字小文字・アクセント・記号の扱いをオプションで制御可能

**コード例**:
```ts
const items = ['Äpfel', 'Apfel', 'Banane', 'Birne'];

// Bad: 単純な sort はバイナリ比較（言語規則に従わない）
items.sort();
// → ['Apfel', 'Banane', 'Birne', 'Äpfel'] （Ä が a の後扱い）

// Good: ドイツ語ルールで sort
const collator = new Intl.Collator('de-DE');
items.sort(collator.compare);
// → ['Äpfel', 'Apfel', 'Banane', 'Birne']（Ä と A が並列扱い）

// 日本語
const japaneseItems = ['さくら', 'カキ', 'りんご', 'みかん'];
const jaCollator = new Intl.Collator('ja-JP');
japaneseItems.sort(jaCollator.compare);
// → ['カキ', 'さくら', 'みかん', 'りんご']（あ行→か行→さ行 の順）

// オプション
new Intl.Collator('ja-JP', {
  sensitivity: 'base',           // base / accent / case / variant
  caseFirst: 'upper',            // 大文字を先に
  numeric: true,                  // "file2" < "file10"
  ignorePunctuation: true,        // 句読点を無視
}).compare('file10', 'file2');    // → 1（数値として比較）
```

**`sensitivity` の値**:
| 値 | 'a' vs 'A' | 'a' vs 'á' |
|---|---|---|
| `base` | 等しい | 等しい |
| `accent` | 等しい | 異なる |
| `case` | 異なる | 等しい |
| `variant` | 異なる | 異なる |

**実用例**:
```ts
// ユーザー名で大文字小文字を区別せずソート
const users = [{ name: 'Alice' }, { name: 'bob' }, { name: 'Charlie' }];
const collator = new Intl.Collator(undefined, { sensitivity: 'base' });
users.sort((a, b) => collator.compare(a.name, b.name));
// → [{name:'Alice'}, {name:'bob'}, {name:'Charlie'}]

// ファイル名で数値順
const files = ['file1.txt', 'file2.txt', 'file10.txt', 'file20.txt'];
const numericCollator = new Intl.Collator(undefined, { numeric: true });
files.sort(numericCollator.compare);
// → ['file1.txt', 'file2.txt', 'file10.txt', 'file20.txt']
```

**検索（部分一致）でも Collator を使う**:
```ts
// 「aécho」の中に「e」が含まれるか（accent insensitive）
function containsLocaleAware(haystack: string, needle: string, locale: string): boolean {
  const collator = new Intl.Collator(locale, { sensitivity: 'base' });
  // 簡易実装: substring を 1 文字ずつ比較
  for (let i = 0; i <= haystack.length - needle.length; i++) {
    if (collator.compare(haystack.slice(i, i + needle.length), needle) === 0) {
      return true;
    }
  }
  return false;
}

containsLocaleAware('café', 'cafe', 'fr-FR');  // → true
```

**サーバー側との一致**:
- DB の `ORDER BY` も locale-aware にする必要がある
- PostgreSQL: `COLLATE "ja_JP.UTF-8"` でロケール指定
- MySQL: `ORDER BY column COLLATE utf8mb4_ja_0900_as_cs`
- 検索・並び替えは「フロントで sort」と「DB で ORDER BY」を一致させる

**出典**:
- [MDN: Intl.Collator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Collator) (MDN Web Docs)
- [ECMA-402: Collator](https://tc39.es/ecma402/#collator-objects) (TC39)
- [Unicode UTS #10: Collation Algorithm](https://www.unicode.org/reports/tr10/) (Unicode)

**バージョン**: 全モダンブラウザ
**確信度**: 高
**最終更新**: 2026-05-16
