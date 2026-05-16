# RTL（右から左）言語対応のベストプラクティス

アラビア語・ヘブライ語・ペルシア語・ウルドゥー語に対応する場合、レイアウトを左右反転する必要がある。
論理プロパティと `dir` 属性を最初から使い、後付け対応の負債を回避する。

## ルール

### 1. 論理プロパティ（`margin-inline-start` 等）で方向に依存しない CSS を書く

`margin-left` / `padding-right` のような物理プロパティではなく、`margin-inline-start` / `padding-inline-end` のような論理プロパティを使う。
LTR / RTL の切り替えで自動的に反転される。

**根拠**:
- 物理プロパティ（`left` / `right`）は RTL で **そのまま** 残るため、`[dir="rtl"]` セレクタで全て上書きが必要
- 論理プロパティ（`inline-start` / `inline-end`）は writing-mode に従って自動的に反転する
- 縦書き（日本語縦組み）にも対応できる（`block-start` / `block-end` は writing-mode 依存）
- CSS Logical Properties Level 1 は全モダンブラウザでサポート済み

**対応表（物理 → 論理）**:

| 物理プロパティ | 論理プロパティ |
|---|---|
| `margin-left` | `margin-inline-start` |
| `margin-right` | `margin-inline-end` |
| `padding-left` | `padding-inline-start` |
| `padding-right` | `padding-inline-end` |
| `border-left` | `border-inline-start` |
| `border-right` | `border-inline-end` |
| `left: 0` | `inset-inline-start: 0` |
| `right: 0` | `inset-inline-end: 0` |
| `text-align: left` | `text-align: start` |
| `text-align: right` | `text-align: end` |
| `float: left` | `float: inline-start` |
| `border-top-left-radius` | `border-start-start-radius` |

**コード例**:
```css
/* Bad: 物理プロパティ */
.card {
  margin-left: 16px;
  padding-right: 24px;
  border-left: 2px solid blue;
  text-align: left;
}
/* RTL では右マージン・左パディングのつもりが、見た目が崩れる */

/* Good: 論理プロパティ */
.card {
  margin-inline-start: 16px;     /* LTR で左、RTL で右 */
  padding-inline-end: 24px;      /* LTR で右、RTL で左 */
  border-inline-start: 2px solid blue;
  text-align: start;             /* LTR で左、RTL で右 */
}
```

**Tailwind CSS v3+ の論理プロパティサポート**:
```html
<!-- ms = margin-inline-start, me = margin-inline-end -->
<div class="ms-4 me-2 ps-6 pe-3">
  <!-- LTR: ml-4 mr-2 pl-6 pr-3 / RTL: mr-4 ml-2 pr-6 pl-3 -->
</div>

<!-- start/end の text-align -->
<p class="text-start">...</p>

<!-- inset 系も -->
<div class="absolute start-0 top-0">...</div>
```

**`<html dir="rtl">` で自動反転される**:
```tsx
// app/[locale]/layout.tsx
const RTL_LOCALES = ['ar', 'he', 'fa', 'ur'];

export default async function LocaleLayout({ params }) {
  const { locale } = await params;
  const dir = RTL_LOCALES.includes(locale) ? 'rtl' : 'ltr';

  return (
    <html lang={locale} dir={dir}>
      <body>{children}</body>
    </html>
  );
}
```

**margin-block / padding-block（縦書き対応）**:
```css
.section {
  margin-block-start: 32px;  /* writing-mode horizontal なら top、vertical なら left or right */
  margin-block-end: 32px;
}
```

通常は `margin-block-start` ＝ `margin-top` だが、`writing-mode: vertical-rl` の場合は変わる。日本語縦書きを想定するなら使う価値がある。

**出典**:
- [MDN: CSS Logical Properties](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_logical_properties_and_values) (MDN Web Docs)
- [W3C: CSS Logical Properties Level 1](https://www.w3.org/TR/css-logical-1/) (W3C)
- [Tailwind CSS: Logical Properties](https://tailwindcss.com/docs/margin#using-logical-properties) (Tailwind)
- [web.dev: Building inclusive forms](https://web.dev/articles/inclusive-forms) (web.dev)

**バージョン**: 全モダンブラウザ
**確信度**: 高
**最終更新**: 2026-05-16

---

### 2. `<html dir>` で文書全体の方向を設定し、locale から推論する

ロケールから RTL/LTR を判定し、`<html>` 要素の `dir` 属性に設定する。
個別要素で `dir` を上書きすると影響範囲が限定される（例: 引用ブロック）。

**根拠**:
- `<html dir>` でドキュメント全体のフローを決めるのが標準的アプローチ
- ブラウザは `dir` に応じて scrollbar 位置・テキスト選択方向・select の dropdown 方向まで自動調整する
- 個別の方向上書きは `<bdi>` / `<bdo>` / `dir` 属性で局所的に行える
- `Intl.Locale` の `textInfo.direction` で BCP 47 タグから direction を取得できる

**コード例**:
```tsx
// 標準的な実装
const RTL_LOCALES = new Set(['ar', 'he', 'fa', 'ur', 'yi']);

export default async function LocaleLayout({ children, params }) {
  const { locale } = await params;
  const dir = RTL_LOCALES.has(locale) ? 'rtl' : 'ltr';

  return (
    <html lang={locale} dir={dir}>
      <body>{children}</body>
    </html>
  );
}

// Intl.Locale API での判定（推奨、将来は本格活用）
function getDirectionForLocale(locale: string): 'ltr' | 'rtl' {
  try {
    // @ts-expect-error — textInfo は ECMAScript 2024 提案
    const direction = new Intl.Locale(locale).textInfo?.direction;
    return direction ?? 'ltr';
  } catch {
    return 'ltr';
  }
}
```

**個別要素での dir 制御**:
```html
<!-- ページは LTR、引用部分のみ RTL -->
<html lang="en" dir="ltr">
  <body>
    <p>This is an English page.</p>

    <blockquote lang="ar" dir="rtl">
      هذا اقتباس باللغة العربية.
    </blockquote>

    <!-- 双方向テキスト埋め込み: ユーザー名等 -->
    <p>The user <bdi>إيان</bdi> commented on this post.</p>

    <!-- 強制的に LTR で表示（数字の混在対応） -->
    <p dir="ltr">電話番号: <bdo dir="ltr">+1 (555) 123-4567</bdo></p>
  </body>
</html>
```

**`<bdi>` vs `<bdo>` の使い分け**:
- `<bdi>`: 周囲のテキストと方向が異なる可能性がある場合に「分離（isolation）」する。ユーザー入力等
- `<bdo>`: 明示的に方向を指定する（override）。特定の表記法に従う必要がある場合

**RTL 切り替えの落とし穴**:
- アイコンの読み込み順序（戻る→進むではなく進む→戻るになる）
- アニメーション方向（slide-in-from-right が右からではなく左から来るべき）
- 数字は LTR のままレンダリングされる（アラビア語でも 123 は LTR）
- `:nth-child` セレクタは依然として「DOM 順」で動作。視覚順とは別

**Cypress / Playwright でのテスト**:
```ts
// playwright-rtl.spec.ts
test.describe('RTL layout', () => {
  test.use({ locale: 'ar' });

  test('navigation aligns to the right', async ({ page }) => {
    await page.goto('/ar/');
    const html = page.locator('html');
    await expect(html).toHaveAttribute('dir', 'rtl');

    const nav = page.locator('nav');
    const box = await nav.boundingBox();
    // RTL では nav の終端（end）が左にあるはず
    expect(box!.x).toBeLessThan(100);
  });
});
```

**出典**:
- [MDN: dir attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/dir) (MDN Web Docs)
- [MDN: bdi element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/bdi) (MDN Web Docs)
- [W3C: Internationalization - Setting the document base direction](https://www.w3.org/International/questions/qa-html-dir) (W3C i18n)
- [TC39: Intl.Locale textInfo proposal](https://github.com/tc39/proposal-intl-locale-info) (TC39)

**バージョン**: HTML Living Standard
**確信度**: 高
**最終更新**: 2026-05-16

---

### 3. 方向依存アイコンは RTL で反転、方向不変アイコンは反転しない

「戻る」「進む」「インデント」など方向の意味を持つアイコンは RTL で水平反転（`transform: scaleX(-1)`）する。
「検索」「設定」「✓」「✗」など方向の意味を持たないアイコンは反転しない。

**根拠**:
- 戻るアイコン（←）は LTR では「戻る」、RTL では「進む」を意味する（読み始める方向と逆）
- 反転する/しないの判断は文化的・言語的なルールに従う
- 大量のアイコンを個別ハンドリングするのは現実的でないため、`[dir="rtl"]` セレクタで一括反転 + 例外を class で除外する運用が標準
- Material Design / Apple HIG とも RTL ガイドラインで明示している

**反転するアイコン**:
- 矢印系: `<` `>` `→` `←` `▶` `◀`
- 戻る・進む: history navigation
- インデント: list indent / outdent
- リスト・順序: rank, ordering
- メディアコントロール: rewind / fast-forward（**例外**: play/pause は反転しない）
- スワイプ・スライド方向

**反転しないアイコン**:
- 検索 🔍 / 設定 ⚙ / 通知 🔔 / ホーム 🏠 / メール ✉
- ✓ / ✗ / +/-
- 時計 ⏰（数字は LTR）
- 音量 🔊
- play / pause（メディア国際標準）

**コード例（CSS で一括反転）**:
```css
/* RTL でも反転しない例外を除き、方向依存アイコンは反転 */
[dir="rtl"] .icon-directional {
  transform: scaleX(-1);
}

/* または個別アイコンクラスで */
[dir="rtl"] .icon-arrow,
[dir="rtl"] .icon-back,
[dir="rtl"] .icon-forward {
  transform: scaleX(-1);
}

/* Tailwind */
<svg class="rtl:scale-x-[-1]">...</svg>

/* Lucide アイコンの ChevronLeft → 反転して ChevronRight 相当に */
import { ChevronLeft } from 'lucide-react';
<ChevronLeft className="rtl:rotate-180" />
```

**判定ヘルパー関数**:
```tsx
const directionalIcons = new Set(['arrow-back', 'arrow-forward', 'chevron-left', 'chevron-right', 'rewind', 'fast-forward']);

function Icon({ name, ...props }: { name: string }) {
  const className = directionalIcons.has(name) ? 'rtl:scale-x-[-1]' : '';
  return <svg className={className} {...props}>...</svg>;
}
```

**Material Design / Apple のガイドライン**:
- Material Design: [RTL Mirroring Guide](https://m2.material.io/design/usability/bidirectionality.html)
- Apple HIG: [Right-to-Left](https://developer.apple.com/design/human-interface-guidelines/right-to-left)

両者とも明示的に「反転すべきアイコン」「すべきでないアイコン」のカタログを提供している。

**翻訳テキスト内の数字**:
```html
<!-- アラビア語の中でも数字は LTR のまま -->
<p dir="rtl">عدد المستخدمين هو 1,234</p>
<!-- "Number of users is 1,234" -->
<!-- レンダリング: 1,234 هو المستخدمين عدد -->
```

数字は Unicode の bidirectional algorithm により自動的に LTR で表示される。手動操作は不要だが、混在テキストでの順序問題は `<bdi>` で対処する。

**出典**:
- [Material Design: Bidirectionality](https://m2.material.io/design/usability/bidirectionality.html) (Google)
- [Apple HIG: Right-to-Left](https://developer.apple.com/design/human-interface-guidelines/right-to-left) (Apple)
- [W3C: When to mirror UI design and icons](https://www.w3.org/International/articles/inline-bidi-markup/) (W3C i18n)

**バージョン**: パターン（一般原則）
**確信度**: 高
**最終更新**: 2026-05-16

---

### 4. Tailwind の `rtl:` / `ltr:` 修飾子で方向別スタイルを書く

論理プロパティでカバーできない部分（grid template / アイコンの transform 等）は Tailwind の `rtl:` / `ltr:` 修飾子で個別対応する。
Tailwind v3 から RTL モディファイアが標準サポート。

**根拠**:
- 論理プロパティは margin / padding / border / text-align 等の「方向対応のあるプロパティ」だけ自動反転する
- transform / grid-template-columns / animation 等は手動で RTL 別スタイルが必要
- `rtl:` 修飾子は `dark:` と同様にプレフィックスで適用、保守性が高い
- Tailwind 設定で `darkMode: 'class'` ではなく `dir-based` で動かす

**Tailwind 設定**:
```ts
// tailwind.config.ts
import type { Config } from 'tailwindcss';

const config: Config = {
  // ... ほかの設定

  // RTL は dir="rtl" 属性で動作（標準）
  // 設定変更不要、デフォルトで rtl: 修飾子が使える
};
```

**実装例**:
```tsx
// アイコンの反転
<svg className="rtl:scale-x-[-1] rtl:rotate-180">...</svg>

// 異なる Grid レイアウト
<div className="grid grid-cols-[200px_1fr] rtl:grid-cols-[1fr_200px]">
  <Sidebar />
  <MainContent />
</div>

// アニメーション方向（スライドイン）
<div className="
  animate-slide-in-from-right
  rtl:animate-slide-in-from-left
">
  ...
</div>

// 数値ベースの transform
<button className="
  translate-x-2
  rtl:-translate-x-2
">
  ...
</button>

// アイコン + テキストの順序（gap も反転対応）
<div className="flex items-center gap-2">
  <Icon className="rtl:order-2" />
  <span>ラベル</span>
</div>
```

**論理プロパティでカバーされる場合は使わない**:
```tsx
// Bad: 既に論理プロパティでカバーされるのに rtl: を使う
<div className="ml-4 rtl:ml-0 rtl:mr-4">  // 二重メンテ
  ...
</div>

// Good: 論理プロパティで自動的に反転
<div className="ms-4">
  ...
</div>
```

**カスタム RTL クラスの作成**:
```css
/* globals.css */
@layer utilities {
  .text-start-rtl-end {
    text-align: start;
  }
  [dir="rtl"] .text-start-rtl-end {
    text-align: end;
  }
}
```

**Tailwind 以外のフレームワーク（CSS Modules / styled-components）**:
```css
/* CSS Modules */
.card {
  margin-inline-start: 16px;
}

[dir="rtl"] .card .icon {
  transform: scaleX(-1);
}
```

```tsx
// styled-components
import styled from 'styled-components';

const Card = styled.div`
  margin-inline-start: 16px;

  [dir="rtl"] & .icon {
    transform: scaleX(-1);
  }
`;
```

**テストでの確認方法**:
```tsx
// vitest + testing-library
test('RTL: arrow icon is mirrored', () => {
  document.documentElement.setAttribute('dir', 'rtl');
  render(<BackButton />);
  const icon = screen.getByRole('img', { name: /back/i });
  expect(icon).toHaveClass('rtl:scale-x-[-1]');
});
```

**出典**:
- [Tailwind CSS: Right-to-Left Support](https://tailwindcss.com/docs/hover-focus-and-other-states#rtl-support) (Tailwind)
- [CSS Logical Properties](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_logical_properties_and_values) (MDN Web Docs)

**バージョン**: Tailwind CSS 3.3+
**確信度**: 高
**最終更新**: 2026-05-16
