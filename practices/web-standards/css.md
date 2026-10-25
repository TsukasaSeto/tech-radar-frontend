# CSS のベストプラクティス

## ルール

### 1. Tailwind CSS ではユーティリティクラスを直接使い、カスタムCSSを最小化する

コンポーネントに直接ユーティリティクラスを記述し、
カスタム CSS は Tailwind のシステムで表現できない場合のみ書く。

**根拠**:
- ユーティリティファーストでCSSファイルの肥大化を防ぐ
- クラス名を読むだけでスタイルが把握できる
- Tailwind 3 以降は JIT エンジンがソース中で使われているクラスだけを生成するため、未使用のスタイルは出力されない

**コード例**:
```tsx
// Bad: カスタム CSS クラスを作成
// styles.module.css
.card {
  display: flex;
  flex-direction: column;
  padding: 1rem;
  border-radius: 0.5rem;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
  background-color: white;
}

// components/Card.tsx
<div className={styles.card}>...</div>

// Good: Tailwind ユーティリティを直接使用
<div className="flex flex-col p-4 rounded-lg shadow-sm bg-white">...</div>

// デザイントークンの再利用には @apply または CSS 変数（最小限に）
// globals.css
@layer components {
  /* 複数箇所で同じスタイルが必要な場合のみ */
  .btn-primary {
    @apply rounded-lg bg-blue-600 px-4 py-2 text-white hover:bg-blue-700;
  }
}

// cn() ユーティリティで条件付きクラスを管理（clsx + tailwind-merge）
import { cn } from '@/lib/utils';

function Button({ variant, className, ...props }: ButtonProps) {
  return (
    <button
      className={cn(
        'rounded-lg px-4 py-2 font-medium',
        variant === 'primary' && 'bg-blue-600 text-white',
        variant === 'outline' && 'border border-gray-300 text-gray-700',
        className  // 外部からの上書き
      )}
      {...props}
    />
  );
}
```

**出典**:
- [Tailwind CSS Docs: Utility-First Fundamentals](https://tailwindcss.com/docs/utility-first) (Tailwind CSS公式)

**バージョン**: Tailwind CSS 3+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 2. CSS カスタムプロパティ（変数）でデザイントークンを管理する

カラー・スペーシング・フォントサイズなどのデザイントークンは
CSS カスタムプロパティで定義し、テーマ切り替えに対応する。

**根拠**:
- CSS カスタムプロパティはJavaScriptで動的に変更でき、テーマ切り替えが容易
- プリプロセッサ（Sass）の変数と異なりカスケードとスコープが効く
- `prefers-color-scheme` メディアクエリと組み合わせてダークモードを実装できる
- トークンを「色そのもの」ではなく「関係」として設計する。ライトモードで決めた「どちらが手前か」「どちらが暗いか」という視覚的関係は、ダークモードで色を反転すると崩れることがある。個別の例外を色ごとに列挙する設計は、想定していない組み合わせで漏れが発生しやすい

> "トークンに持たせるべきは『色』ではなく『関係』"
> ([ダークモード対応で同じ穴を2回踏んだ——ライトで決めた「明暗の関係」は逆転する](https://zenn.dev/matsutake_prgrm/articles/dark-mode-contrast-inversion), セクション "一般則") ※2026-08-15に実際にfetch成功

**コード例**:
```css
/* globals.css */
:root {
  /* カラートークン */
  --color-primary: #2563eb;
  --color-primary-hover: #1d4ed8;
  --color-text: #111827;
  --color-text-muted: #6b7280;
  --color-background: #ffffff;
  --color-surface: #f9fafb;

  /* スペーシング */
  --spacing-xs: 0.25rem;
  --spacing-sm: 0.5rem;
  --spacing-md: 1rem;
  --spacing-lg: 1.5rem;

  /* フォント */
  --font-size-sm: 0.875rem;
  --font-size-base: 1rem;
  --font-size-lg: 1.125rem;
}

/* ダークモード（OS 設定）。background / surface / text / muted は必ず組で上書きする
   （一部だけ上書きすると、ライト用の text とダーク用の surface のような組み合わせが残る） */
@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    --color-text: #f9fafb;
    --color-text-muted: #9ca3af;  /* #111827 に対して 6.99:1、#1f2937 に対して 5.78:1 */
    --color-background: #111827;
    --color-surface: #1f2937;
  }
}

/* data 属性でのテーマ切り替え（JS 制御）。値は上の @media 版と同じ組にそろえる */
[data-theme="dark"] {
  --color-text: #f9fafb;
  --color-text-muted: #9ca3af;
  --color-background: #111827;
  --color-surface: #1f2937;
}
```

```tsx
// Tailwind v3 での CSS 変数活用
// tailwind.config.ts
export default {
  theme: {
    extend: {
      colors: {
        primary: 'var(--color-primary)',
        background: 'var(--color-background)',
      },
    },
  },
};
```

**出典**:
- [MDN: CSS Custom Properties](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties) (MDN Web Docs)
- [ダークモード対応で同じ穴を2回踏んだ——ライトで決めた「明暗の関係」は逆転する](https://zenn.dev/matsutake_prgrm/articles/dark-mode-contrast-inversion) (Zenn、色の関係を列挙ではなく相対関係としてトークン化する設計指針) ※2026-08-15に実際にfetch成功

**バージョン**: CSS Living Standard
**確信度**: 高
**最終更新**: 2026-10-25

---

### 3. レスポンシブデザインはモバイルファーストで実装する

スタイルはモバイル向けをベースに書き、`min-width` メディアクエリで
画面が広くなるにつれてスタイルを上書きする。

**根拠**:
- モバイルユーザーが多い現代ではモバイルを優先することが自然
- メディアクエリは詳細度を変えないので、上書きはソース順で決まる。`min-width`（モバイルファースト）で狭い順に書けば、上書きの向きが「広い画面ほど後から足す」の一方向にそろう
- Tailwind CSS のブレークポイントがモバイルファーストを前提としている（`sm:` = 640px以上）

**コード例**:
```tsx
// Tailwind CSS でモバイルファースト
function ProductGrid({ products }: { products: Product[] }) {
  return (
    <div className={`
      grid
      grid-cols-1      /* モバイル: 1列 */
      sm:grid-cols-2   /* 640px以上: 2列 */
      md:grid-cols-3   /* 768px以上: 3列 */
      lg:grid-cols-4   /* 1024px以上: 4列 */
      gap-4
    `}>
      {products.map(product => (
        <ProductCard
          key={product.id}
          className="
            p-3 sm:p-4      /* モバイルは小さいパディング */
            text-sm sm:text-base  /* モバイルは小さいフォント */
          "
        />
      ))}
    </div>
  );
}

// CSS でモバイルファースト
// Bad: max-width（デスクトップファースト）
@media (max-width: 768px) {
  .grid { grid-template-columns: 1fr; }
}

// Good: min-width（モバイルファースト）
.grid { grid-template-columns: 1fr; }  /* デフォルト: モバイル */

@media (min-width: 640px) {
  .grid { grid-template-columns: repeat(2, 1fr); }
}
@media (min-width: 1024px) {
  .grid { grid-template-columns: repeat(4, 1fr); }
}
```

**出典**:
- [MDN: Responsive design](https://developer.mozilla.org/en-US/docs/Learn/CSS/CSS_layout/Responsive_Design) (MDN Web Docs)
- [Tailwind CSS Docs: Responsive Design](https://tailwindcss.com/docs/responsive-design) (Tailwind CSS公式)

**バージョン**: CSS Living Standard, Tailwind CSS 3+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 4. CSS Container Queries でコンポーネント単位のレスポンシブを実装する

Viewport幅ではなく親コンテナの幅に応じてスタイルを変化させる `@container` クエリを活用する。
サイドバーに配置されるカードや、再利用コンポーネントのような「置かれる場所によって見た目が変わる」UIに最適。

**根拠**:
- メディアクエリはビューポート幅に依存するため、コンポーネントの再利用時にスタイルが崩れやすい
- Container Queries はコンポーネントが置かれたコンテキスト（親の幅）を基準にできる
- Chrome 105+、Firefox 110+、Safari 16+ でサポートされておりモダンブラウザでは実用段階
- Container Query Units（`cqi` / `cqw` 等）を使うと、コンテナ幅に対して相対的にフォントサイズ等をフルード化できる（例: `clamp(1rem, .5rem + 3cqi, 2rem)`）。認知度に対して実採用率が低いというギャップがあり、メディアクエリのブレークポイントをそのまま流用してしまう書き方の慣性が主因

**コード例**:
```css
/* Good: コンテナを定義し、子要素からクエリする */
.card-wrapper {
  container-type: inline-size;  /* 横幅をクエリ対象に */
  container-name: card;         /* 名前付きコンテナ（省略可） */
}

/* コンテナ幅が 400px 以下のとき縦積みレイアウト */
@container card (max-width: 400px) {
  .card {
    flex-direction: column;
  }
  .card__image {
    width: 100%;
  }
}

/* コンテナ幅が 401px 以上のとき横並びレイアウト */
@container card (min-width: 401px) {
  .card {
    flex-direction: row;
  }
  .card__image {
    width: 200px;
    flex-shrink: 0;
  }
}

/* Bad: ビューポート幅でのメディアクエリ（コンテキスト依存で崩れる） */
@media (max-width: 768px) {
  .card {
    flex-direction: column;  /* サイドバー配置時も誤発動する */
  }
}
```

```tsx
// React コンポーネントでの使用例
function ProductCard() {
  return (
    // wrapper に container-type を付与
    <div className="[container-type:inline-size]">  {/* Tailwind arbitrary value */}
      <div className="flex @[400px]:flex-row flex-col">
        <img className="@[400px]:w-48 w-full" src="..." alt="..." />
        <div className="p-4">
          <h2>商品名</h2>
          <p>説明文</p>
        </div>
      </div>
    </div>
  );
}
// ※ Tailwind v3.2+ では @tailwindcss/container-queries プラグインで @[] 構文を使用
```

**出典**:
- [CSS Containment Module Level 3: Container queries](https://www.w3.org/TR/css-contain-3/#container-queries) (W3C Spec)
- [CSS container queries - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_containment/Container_queries) (MDN Web Docs / 2023)
- [Container queries observe the component, not the viewport](https://dev.to/leobaniak/container-queries-observe-the-component-not-the-viewport-3chn) (dev.to、Container Query Units によるフルードタイポグラフィと実採用率ギャップの指摘) ※2026-09-19 fetch
  > "Media queries answer the browser. Container queries answer the box."

**バージョン**: Chrome 105+, Firefox 110+, Safari 16+
**確信度**: 高
**最終更新**: 2026-09-19

---

### 5. CSS Nesting のネイティブサポートを活用する

プリプロセッサ（Sass/Less）を使わずに、CSS ネイティブのネスト構文で
セレクタを階層的に記述する。

**根拠**:
- Chrome 112+、Firefox 117+、Safari 17.2+ でサポート済み（2024年時点でモダンブラウザは全対応）
- **2026年6月11日に Baseline Widely Available に到達**。すべてのモダンブラウザで「広く安定して使える」と公式認定されたため、プロジェクトへの採用判断が容易になった
- Sass のネスト記法と互換性が高く、移行コストが低い
- ビルドステップなしでネスト構造が使え、コンポーネントスタイルの見通しが良くなる
- **`&` の省略ルール**: 子セレクタ（`>`/`+`/`~`）や型セレクタは `&` なしで書ける。修飾子（`.card.-subtle`）・擬似クラス（`:hover`）・属性セレクタは `&` が必須。文字列連結（`&__title` のような BEM 構文）は不可——フラットなクラス名（`.card__title`）で代替する
- **Sass との共存**: ネスト目的のみで Sass を使っていた場合、ネイティブ CSS への移行で Sass のプリプロセスが不要になる。ただし Sass の変数・ミックスイン・ロジック機能は引き続き価値があるため「脱 Sass」は目的ではなく「脱ネスト用途 Sass」が正確

**コード例**:
```css
/* Good: CSS ネイティブネスト */
.card {
  padding: 1rem;
  border-radius: 0.5rem;
  background: white;

  /* 子要素のスタイル */
  & .card__title {
    font-size: 1.25rem;
    font-weight: bold;
  }

  /* 擬似クラス */
  &:hover {
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  }

  /* メディアクエリもネスト可能 */
  @media (max-width: 640px) {
    padding: 0.75rem;
  }

  /* 状態バリアント */
  &.is-featured {
    border: 2px solid var(--color-primary);
  }
}

/* Bad: フラットな記述（Sass なしの従来手法） */
.card { padding: 1rem; }
.card .card__title { font-size: 1.25rem; }
.card:hover { box-shadow: 0 4px 12px rgba(0,0,0,0.1); }
.card.is-featured { border: 2px solid var(--color-primary); }
```

**出典**:
- [CSS Nesting - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_nesting) (MDN Web Docs)
- [CSS Nesting Module Level 1](https://www.w3.org/TR/css-nesting-1/) (W3C Spec)
- [CSS Nesting が Baseline WA に到達](https://zenn.dev/shunei/articles/css-nesting-baseline-scss) (Zenn、Baseline WA 到達・`&` 省略ルール・Sass 移行判断) ※2026-06-10 fetch

> "ネスト目的だけで Sass を使っていたなら、要らなくなりつつあります。ただし Sass の真価は、ネストではありません。"
> ([CSS Nesting が Baseline WA に到達](https://zenn.dev/shunei/articles/css-nesting-baseline-scss), セクション "脱 Sass の判断基準") ※2026-06-10に実際にfetch成功

**バージョン**: Chrome 112+, Firefox 117+, Safari 17.2+（Baseline Widely Available: 2026-06-11）
**確信度**: 高
**最終更新**: 2026-06-10

---

### 6. `:has()` 疑似クラスで親要素を条件スタイリングする

`:has()` を使い、子要素の状態に応じて親要素のスタイルを変更する。
「チェックされたチェックボックスを含むラベル」や「画像を含むカード」のような
従来 JavaScript が必要だったパターンを純粋な CSS で実現できる。
ただし置き換えてよいのは「見た目の同期」であり、フォームの送信可否判定や
他ロジックが参照する「状態そのもの」の管理までは `:has()` に置き換えるべきではない。

**根拠**:
- CSS の長年の課題だった「親セレクタ」がネイティブで実現された
- Chrome 105+、Firefox 121+、Safari 15.4+ でサポート済み
- フォームのバリデーション状態など、動的なスタイル変更をJSなしで実装できる
- `:has()` はあくまで見た目の同期に使うべきで、送信可否の判定結果を他のロジックが参照するような「状態そのもの」の管理まで CSS に寄せると、状態の単一の真実源が UI 側に分散してしまう

**コード例**:
```css
/* Good: チェックボックスが checked の親 label をハイライト */
.option-label:has(input[type="checkbox"]:checked) {
  background-color: var(--color-primary-light);
  border-color: var(--color-primary);
  font-weight: bold;
}

/* 画像を含む場合と含まない場合でカードのレイアウトを切り替え */
.card:has(img) {
  grid-template-columns: 200px 1fr;
}
.card:not(:has(img)) {
  grid-template-columns: 1fr;
}

/* フォームが不正な入力を含む間、送信ボタンを薄く表示する（見た目の同期だけ）
   - pointer-events: none ではポインタ操作しか止まらず、Enter キーでの送信やフォーカスは止まらない。
     送信の無効化は disabled 属性やフォームの状態（ネイティブバリデーション）で行う
   - :invalid は未入力の required 項目にも初期表示から一致するので、操作後だけ一致する :user-invalid を使う */
form:has(input:user-invalid) .submit-button {
  opacity: 0.5;
}

/* 入力済みの input を持つフィールドのラベルを浮かせる（Floating Label） */
.field:has(input:not(:placeholder-shown)) .field__label {
  transform: translateY(-1.5rem) scale(0.85);
  color: var(--color-primary);
}

/* Bad: 同等の処理を JS で行う（避けるべき） */
// checkbox.addEventListener('change', e => {
//   label.classList.toggle('is-checked', e.target.checked);
// });
```

**出典**:
- [:has() CSS pseudo-class - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/CSS/:has) (MDN Web Docs)
- [CSS Selectors Level 4: :has()](https://www.w3.org/TR/selectors-4/#relational) (W3C Spec)

**バージョン**: Chrome 105+, Firefox 121+, Safari 15.4+（`:user-invalid` は Chrome 119+, Firefox 88+, Safari 16.5+）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 7. `@layer` でCSSカスケードの優先順位を明示的に管理する

`@layer` でスタイルをレイヤーに分割し、CSSの詳細度（specificity）や `!important` に頼らずに
スタイルの適用優先順位をコードで明示する。

**根拠**:
- CSS 詳細度の衝突は大規模プロジェクトで管理困難になり `!important` の連鎖を招く
- `@layer` はレイヤーの宣言順序で優先順位が決まるため、詳細度ハックが不要になる
- Tailwind CSS v4 は内部的にカスケードレイヤーを使用しており、サードパーティCSSとの競合を防ぐ構造が標準化された
- 通常の宣言では、`@layer` 外に書かれたスタイルがすべてのレイヤーより優先されるため、緊急の上書きもシンプルに書ける
- `!important` を付けた宣言では順序が逆転する。レイヤー内の `!important` はレイヤー外の `!important` に勝ち、レイヤー同士では先に宣言したレイヤーの `!important` ほど強い。サードパーティ CSS の `!important` は、レイヤーに入れると自前の `!important` より強くなる点に注意する
- サードパーティ CSS は `@import url(...) layer(vendor);` でレイヤーに読み込む。`@import` は `@charset` と `@layer` の宣言文以外のすべてのルールより前に書く必要があり、`@layer vendor { ... }` のブロック内に書いた `@import` は無効として無視される

**コード例**:
```css
/* レイヤーの優先順位を最初に宣言（先に宣言したレイヤーほど優先度が低い）。
   vendor を先頭に置き、サードパーティ CSS を自前のスタイルに負けさせる */
@layer vendor, base, components, utilities;

/* サードパーティ CSS を vendor レイヤーに読み込む。
   @import は @layer 宣言文の直後、ほかのルールより前のトップレベルに書く */
@import url('vendor-library.css') layer(vendor);

/* base レイヤー: リセット・デフォルトスタイル */
@layer base {
  *, *::before, *::after {
    box-sizing: border-box;
  }
  body {
    font-family: var(--font-sans);
    color: var(--color-text);
  }
}

/* components レイヤー: UIコンポーネント */
@layer components {
  .btn {
    padding: 0.5rem 1rem;
    border-radius: 0.375rem;
    background-color: var(--color-primary);
    color: white;
  }
}

/* utilities レイヤー: ユーティリティクラス（最も優先度が高いレイヤー） */
@layer utilities {
  .hidden { display: none; }
  .sr-only { /* スクリーンリーダーのみ表示 */ }
}

/* レイヤー外は全レイヤーより優先（通常の宣言の場合。例外的な上書きに使用） */
.critical-override {
  color: red;
}
```

```css
/* Bad: @import はブロック内に書けないので無視され、vendor CSS は読み込まれない。
   また先頭の宣言に vendor がないため、このブロックで初めて作られる vendor は
   最後尾（最優先）のレイヤーになり、「自前スタイルに負けさせる」意図と逆になる */
@layer base, components, utilities;

@layer vendor {
  @import url('vendor-library.css');
}
```

**出典**:
- [最近のCSS、全然追えてなかった。ここ1、2年で使えるようになった機能10選](https://zenn.dev/seekseek/articles/css-new-features-catch-up-2026) (Zenn seekseek / 2026) ※2026-05-06に実際にfetch成功
- [MDN: @layer](https://developer.mozilla.org/en-US/docs/Web/CSS/@layer) (MDN Web Docs、宣言順による優先度・レイヤー外の宣言・`!important` での逆転) ※2026-10-25 確認
- [MDN: @import](https://developer.mozilla.org/en-US/docs/Web/CSS/@import) (MDN Web Docs、`@import` を置ける位置と `layer()` 構文) ※2026-10-25 確認

**バージョン**: Chrome 99+, Firefox 97+, Safari 15.4+（`@layer` と `@import ... layer()` の両方。Widely Available）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 8. `scrollbar-gutter: stable` でスクロールバー出現によるレイアウトシフトを防ぐ

`scrollbar-gutter: stable` を `:root` に設定し、スクロールバーが出現・消失する際にページレイアウトがガタつく CLS を防ぐ。

**根拠**:
- モーダル表示時の `overflow: hidden` 切り替えなど、スクロールバーの表示・非表示でページ幅が変化し CLS が発生する
- `scrollbar-gutter: stable` はコンテンツが overflow しない場合でもスクロールバー分のスペースを確保し、幅の変動を防ぐ
- CSS 1行で解決でき、JavaScript での `padding-right` 補正など回避策のハックが不要になる
- Chrome 94+、Firefox 97+、Safari 18.2+ でサポート済み（Baseline Newly available: 2024-12。Safari 18.2 未満では無視されるだけで害はない）

**コード例**:
```css
/* globals.css */
:root {
  scrollbar-gutter: stable;
}
```

```css
/* モーダル表示時もレイアウトシフトが発生しない */

/* Bad: overflow: hidden でスクロールバーが消えてページ幅が変化する */
body.modal-open {
  overflow: hidden;
  /* スクロールバー分（約17px）ページ幅が広がり、コンテンツが右にシフト */
}

/* Good: scrollbar-gutter: stable でスクロールバー分のスペースを常に確保 */
:root {
  scrollbar-gutter: stable;
}

body.modal-open {
  overflow: hidden;
  /* スクロールバーが消えても gutter（余白）が維持されるためレイアウト不変 */
}
```

```tsx
// Next.js での推奨設定: app/globals.css に追加するだけ
// layout.tsx で globals.css をインポートすれば全ページに適用される

// useEffect での padding 補正（アンチパターン）が不要になる
// Bad:
// useEffect(() => {
//   document.body.style.paddingRight =
//     `${window.innerWidth - document.documentElement.clientWidth}px`;
// }, [isModalOpen]);
```

**出典**:
- [The Most Underrated CSS Property Nobody Talks About: scrollbar-gutter](https://medium.com/@konstantinkeylin/the-most-underrated-css-property-nobody-talks-about-scrollbar-gutter-2ca598352675) (Medium konstantinkeylin / 2026-05) ※2026-05-06に実際にfetch成功
- [MDN: scrollbar-gutter](https://developer.mozilla.org/en-US/docs/Web/CSS/scrollbar-gutter) (MDN Web Docs、Baseline 2024 / Newly available と各ブラウザの対応版) ※2026-10-25 確認

**バージョン**: Chrome 94+, Firefox 97+, Safari 18.2+（Baseline Newly available: 2024-12）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 9. UIライブラリ配布時は CSSをJavaScriptにバンドルせず、明示的インポートで提供する

npm パッケージとして配布する UIライブラリでは、CSS を JavaScript ファイルに自動インポートする形式を避け、利用者がアプリ側で明示的に CSS ファイルをインポートする設計にする。

**根拠**:
- `import './button.css'` のような静的 CSS インポートが JavaScript に含まれると、バンドラを通さずに Node.js がライブラリの JS を直接読み込む環境（Node 単体での SSR やテスト実行など）で `Unknown file extension ".css"` エラーが発生する
- Next.js の App Router 自体は、`app` 内のどのコンポーネントからでも CSS（npm パッケージが配布する CSS を含む）を import できる。問題になるのは上記のようにバンドラを通らない実行環境に限られる
- 利用者がアプリの `globals.css` で `@import` するか `<link>` タグで読み込む設計にすることで、CSS の適用タイミングとバンドル方式を利用者が制御できる（Next.js では CSS の順序が import の順序で決まる）
- `package.json` の `exports` フィールドで CSS ファイルの存在を明示する

**コード例**:
```tsx
// Bad: ライブラリコンポーネント内で CSS を静的インポート（バンドラを通さない Node.js 実行でクラッシュ）
// packages/my-ui-lib/src/Button.tsx
import './button.css'; // Node.js: Unknown file extension ".css"
export function Button({ children }: { children: React.ReactNode }) {
  return <button className="btn">{children}</button>;
}

// Good: CSS をビルド成果物として別ファイルで提供
// packages/my-ui-lib/dist/styles.css （ビルド出力）

// package.json の exports フィールドで明示
// {
//   "exports": {
//     ".": "./dist/index.js",
//     "./styles.css": "./dist/styles.css"
//   }
// }

// 利用側アプリで明示的にインポート（exports で公開したパスを使う。dist/ を直接指すと exports を尊重するリゾルバで解決できない）
// app/globals.css
// @import '@my-ui-lib/styles.css';

// または layout.tsx でインポート
import '@my-ui-lib/styles.css';
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return <html><body>{children}</body></html>;
}
```

**出典**:
- [Why UI Libraries Still Need Explicit CSS Imports](https://dev.to/ddtamn/why-ui-libraries-still-need-explicit-css-imports-5b6o) (dev.to ddtamn / 2026-05)
- [Next.js Docs: CSS](https://nextjs.org/docs/app/getting-started/css) (Next.js 公式、global CSS と外部パッケージの CSS は `app` 内のどこからでも import できること、CSS の順序は import 順で決まること) ※2026-10-25 確認

**バージョン**: Next.js 14+, Node.js 20+
**確信度**: 中（ライブラリ設計の主張は 1 記事。Next.js の挙動は公式で確認）
**最終更新**: 2026-10-25

---

### 10. Modern CSS の3大原則（論理プロパティ・`:focus-visible`・opacity/transform アニメーション）を適用する

Google の Modern Web Guidance（2026）に基づき、RTL/i18n 対応のための論理プロパティ、フォーカスリングの `:focus-visible`、リフローを起こさないコンポジタレイヤアニメーション（opacity/transform）の3つを一貫して使う。
論理プロパティの詳細は `practices/i18n/rtl.md` #1、`:focus-visible` の詳細は `practices/web-standards/accessibility.md` #5 を参照。このルールでは opacity/transform アニメーションを扱う。

**根拠**:
- **opacity/transform アニメーション**: `height`・`width`・`top` 等はアニメーション時にレイアウト再計算（リフロー）が発生する。`opacity`・`transform` はコンポジタレイヤで処理されリフローが不要。GPU 加速により 60fps を維持しやすい

**コード例**:
```css
/* Good: コンポジタレイヤのアニメーション（リフローなし） */
.modal {
  opacity: 0;
  transform: translateY(-8px);
  transition: opacity 200ms ease, transform 200ms ease;
}
.modal.is-visible {
  opacity: 1;
  transform: translateY(0);
}

/* Bad: リフローが発生するアニメーション */
.modal {
  height: 0;
  overflow: hidden;
  transition: height 200ms ease;  /* レイアウト再計算が毎フレーム発生 */
}
```

**アンチパターン**:
- `top` / `height` をアニメーション対象にする → パフォーマンス問題が発生する

**出典引用**:
> "論理プロパティを使う" / "フォーカスリングは`:focus-visible`で定義する" / "OpacityとTransformを優先する"
> ([Googleの「Modern Web Guidance」で学ぶModern CSSのDo's and Don'ts](https://zenn.dev/ubie_dev/articles/modern-css-dos-donts), セクション "Modern CSSのDo's and Don'ts") ※2026-05-21に実際にfetch成功

**バージョン**: CSS Living Standard（全モダンブラウザで対応済み）
**確信度**: 中（Google の Modern Web Guidance を紹介する 1 記事が出典）
**最終更新**: 2026-10-25

---

### 11. CSS Anchor Positioning でツールチップ・ポップオーバーを JavaScript なしで配置する

`anchor-name` / `position-anchor` / `anchor()` / `position-try-fallbacks` を使い、
ツールチップ・ドロップダウン・ポップオーバーの配置計算を JavaScript ライブラリに頼らず CSS ネイティブで実現する。

**根拠**:
- Floating UI / Popper.js のような外部ライブラリが不要になり、バンドルサイズと実行時オーバーヘッドが削減できる
- `position-try-fallbacks` によりビューポート端でのはみ出しを CSS だけで自動回避できる
- 対応版: `anchor-name` / `anchor()` / `@position-try` は Chrome / Edge 125+、`position-try-fallbacks` は Chrome / Edge 128+（125〜127 は旧名 `position-try-options`）、Firefox 147+、Safari 26+（W3C Candidate Recommendation）
- Baseline: web-features のデータでは、主要な個別機能が 2026-01 に Newly available になり、2026-09 に `position-anchor` の全値（Chrome / Edge 151・Firefox 151・Safari 27）がそろった。ただし `position-visibility` の一部の値が Safari だけの対応のため、「Anchor positioning」機能全体はまだ Baseline ではない（caniuse でも Safari 27 以外は部分対応の扱い）
- `position-area` と論理値（`block-end span-inline-end`）を使えば RTL 対応も追加 CSS なしで得られる（古い環境ではフォールバック配置に劣化する）
- `@position-try` に書けるのは inset・margin・サイズ・self-alignment・`position-anchor` / `position-area` だけで、`transform` は上書きできない。中央寄せを `transform: translateX(-50%)` で行うと、すべての候補に同じ transform が残って位置がずれるので、`anchor-center` で揃える

**コード例**:
```css
/* Good: CSS Anchor Positioning */
.trigger {
  anchor-name: --tooltip-anchor;
}

.tooltip {
  position: absolute;
  position-anchor: --tooltip-anchor;
  /* 既定: アンカーの上に置き、横方向はアンカーの中央に揃える */
  bottom: calc(anchor(top) + 8px);
  justify-self: anchor-center;

  /* ビューポート端に当たったら 下 → 右 → 左 の順で試す
     （flip-block は上下の inset と anchor() の辺を入れ替える組み込みの候補） */
  position-try-fallbacks: flip-block, --right, --left;
}

/* 候補ごとに、既定の配置で使った bottom / justify-self を打ち消してから位置を指定する */
@position-try --right {
  bottom: auto;
  left: calc(anchor(right) + 8px);
  justify-self: normal;
  align-self: anchor-center;
}

@position-try --left {
  bottom: auto;
  right: calc(anchor(left) + 8px);
  justify-self: normal;
  align-self: anchor-center;
}
```

**アンチパターン**:
- `position: fixed` + JavaScript で `getBoundingClientRect()` を使った手動配置 → スクロール・リサイズのたびに再計算が必要
- Floating UI / Popper.js を CSS で代替できる場面でも使い続ける → 不要なバンドルサイズ増加

**出典引用**:
> "CSS Anchor PositioningとはJavaScriptを使わず、CSSのみでツールチップやドロップダウンメニューなどの配置を実現する機能"
> ([CSS Anchor Positioningでカスタムツールチップを実装する](https://zenn.dev/mk668a/articles/980bc733fd9e0d), Zenn) ※2026-06-04に実際にfetch成功

**出典**:
- [Dropdowns That Position Themselves: CSS Anchor Positioning Is Baseline Now](https://dev.to/rashadmehtab/dropdowns-that-position-themselves-css-anchor-positioning-is-baseline-now-450d) (dev.to、Baseline 到達と `position-area` の論理値記法) ※2026-10-05 fetch
  > "CSS anchor positioning reached Baseline in September 2026... the current release of every major browser supports it, while older installs do not." (セクション "Browser Support")
- [MDN: position-try-fallbacks](https://developer.mozilla.org/en-US/docs/Web/CSS/position-try-fallbacks) (MDN Web Docs、try-tactic と `@position-try` の参照方法、ブラウザ対応版) ※2026-10-25 確認
- [MDN: @position-try](https://developer.mozilla.org/en-US/docs/Web/CSS/@position-try) (MDN Web Docs、`@position-try` に書けるプロパティ) ※2026-10-25 確認
- [web-features: anchor-positioning](https://github.com/web-platform-dx/web-features/blob/main/features/anchor-positioning.yml) (W3C WebDX CG、個別機能ごとの Baseline 到達日) ※2026-10-25 確認

**バージョン**: Chrome / Edge 128+（`position-try-fallbacks`。125〜127 は旧名 `position-try-options`）, Firefox 147+, Safari 26+（W3C CSS Anchor Positioning Module Level 1 CR）
**確信度**: 高（W3C Candidate Recommendation + 主要ブラウザの現行版が対応）
**最終更新**: 2026-10-25

---

### 12. `will-change` / `transform: translateZ(0)` は子要素の `position: fixed` を壊すことを踏まえて配置する

パフォーマンス最適化のために親要素へ `will-change: transform`（`transform` や `filter` など、指定すると包含ブロックを作るプロパティを挙げた `will-change`）や `transform: translateZ(0)` を指定すると、その要素は子孫要素にとっての containing block（包含ブロック）になる。結果として子要素の `position: fixed` がビューポート基準ではなく、その親要素基準で配置されてしまい、意図せず固定表示が壊れる。

**根拠**:
- CSS 仕様上、`transform` に `none` 以外の値を指定した要素は、すべての子孫要素に対して containing block となり、新しいスタッキングコンテキストを作成する
- `will-change: transform` も同様の副作用（先行してスタッキングコンテキストを作る挙動）を持ちうるため、GPU アクセラレーションの意図しか無くても `position: fixed` の基準がずれる
- 対策は3通り: (1) 最適化を本当に必要な要素にのみ適用しコンテナ全体には貼らない、(2) 最適化を `::before` / `::after` 疑似要素に移し子要素の配置に影響させない、(3) アニメーション層と `fixed` 要素を別の DOM 階層に分離する
- パフォーマンス最適化は「とりあえず全部に付ける」デフォルト運用にせず、計測して必要な箇所にだけ適用する

**コード例**:
```css
/* Bad: 親要素全体に will-change を貼り、子要素の position: fixed が親基準になる */
.card {
  will-change: transform;
}
.card .fixed-badge {
  position: fixed; /* ビューポートではなく .card 基準で配置されてしまう */
  top: 8px;
  right: 8px;
}

/* Good: 最適化を疑似要素に移し、子要素の配置に影響させない */
.card {
  position: relative;
}
.card::before {
  content: "";
  position: absolute;
  inset: 0;
  will-change: transform;
  /* アニメーション対象はこの疑似要素側にまとめる */
}
.card .fixed-badge {
  position: fixed; /* ビューポート基準のまま */
  top: 8px;
  right: 8px;
}
```

**アンチパターン**:
- 最適化のためだけに親要素全体へ `will-change` や `translateZ(0)` を貼り、意図せず子孫の `fixed` 配置を壊す
- `position: fixed` が効かなくなった原因調査をせず、`position: absolute` に変更してレイアウトの意味を変えてしまう

**出典引用**:
> "transformにnone以外の値を指定すると、その要素はすべての子孫要素に対して「包含ブロック（containing block）」となる"
> ([will-changeとtransform: translateZ(0)の落とし穴 - position:fixedが効かなくなる理由と対策](https://zenn.dev/sarap422/articles/cbdf96f215bbda), セクション 原因の解説) ※2026-08-07に実際にfetch成功

**出典**:
- [will-changeとtransform: translateZ(0)の落とし穴 - position:fixedが効かなくなる理由と対策](https://zenn.dev/sarap422/articles/cbdf96f215bbda) (Zenn) ※2026-08-07に実際にfetch成功

**バージョン**: CSS仕様（Stacking Context / Containing Block、全モダンブラウザ共通の挙動）
**確信度**: 中（CSS仕様上の一般的挙動の解説だが、出典は1記事）
**最終更新**: 2026-10-25

---

### 13. `oklch()` などの CSS カスタムプロパティに対するフォールバックは `@supports` で分岐する（`var()` 側では失敗しない）

CSS カスタムプロパティ（`--foo: oklch(...)`）はほぼ任意のトークン列を受理し、ブラウザが `oklch()` を解釈できなくても文字列としてそのまま保持してしまう。値を実際に使う `color` などのプロパティで初めて無効値として扱われるため、`var()` 側にフォールバック値を書いても機能しない。`@supports (color: oklch(...))` でブロックごと分岐させる必要がある。

**根拠**:
- カスタムプロパティは型を持たず、パース可能かどうかのチェックは「値を消費する側」で初めて行われる
- そのため「未対応ブラウザでは前の宣言にフォールバックする」という期待通りの挙動にならず、無効値が保持されたまま `color` に渡って描画が壊れる
- `@supports` でカスタムプロパティの再定義自体を丸ごとガードすれば、未対応ブラウザには不正なトークン列を渡さずに済む

**コード例**:
```css
/* Bad: var() 側にフォールバックを書いても機能しない */
:root {
  --brand: #6d28d9;
}
:root {
  --brand: oklch(50% 0.2 300); /* 未対応ブラウザでも文字列として保持されてしまう */
}
.button {
  background: var(--brand, #6d28d9); /* フォールバックは発火しない */
}

/* Good: @supports でブロックごと分岐する */
:root {
  --brand: #6d28d9;
}
@supports (color: oklch(50% 0 0)) {
  :root {
    --brand: oklch(50% 0.2 300);
  }
}
```

**出典引用**:
> "A custom property accepts almost any token sequence. A browser can preserve `oklch(...)` as the value of `--brand` even if it cannot use that function as a color."
> ([Why OKLCH CSS variables need @supports for a real fallback](https://dev.to/ivan_kulkin_1522025957eee/why-oklch-css-variables-need-supports-for-a-real-fallback-2fgp), 本文) ※2026-08-21に実際にfetch成功

**出典**:
- [Why OKLCH CSS variables need @supports for a real fallback](https://dev.to/ivan_kulkin_1522025957eee/why-oklch-css-variables-need-supports-for-a-real-fallback-2fgp) (dev.to) ※2026-08-21 fetch

**バージョン**: CSS Custom Properties（CSS Variables）仕様、全モダンブラウザ共通の挙動
**確信度**: 中（CSS仕様上の一般的挙動の解説だが、出典は1記事）
**最終更新**: 2026-08-21

---

### 14. Tailwind CSS v4 では `leading-*` がレスポンシブな `text-*` に勝つ（v3 とカスケード優先順位が逆転している）

Tailwind CSS v3 では `text-{size}` ユーティリティが行間（line-height）を直接 CSS プロパティとして出力していたため、レスポンシブバリアント（`md:text-5xl` 等）が後から出力されて `leading-*` を上書きすることがあった。v4 では `text-*` の line-height が `var(--tw-leading, var(--text-*--line-height))` というカスタムプロパティ経由のフォールバック形式に変わり、`leading-*` が明示的に `--tw-leading` を設定するため、宣言順に関わらず `leading-*` が常に優先される。バージョンアップ時にこの優先順位の逆転に気づかず「効かなくなった」と誤診しないよう注意する。

**根拠**:
- v3 は `text-{size}` が `line-height` を直接代入するため、CSSの宣言順（レスポンシブバリアントが後に出力される）で上書きが発生していた
- v4 は `text-{size}` の line-height が `var(--tw-leading, var(--text-{size}--line-height))` というフォールバック形式になり、`leading-*` が `--tw-leading` カスタムプロパティを明示的に設定することで、宣言順ではなく変数の有無で優先順位が決まる構造に変わった

**コード例**:
```html
<h1 class="text-4xl font-bold leading-tight md:text-5xl lg:text-6xl">…</h1>
```
```css
/* Tailwind v4 が生成する CSS（抜粋） */
.text-6xl {
  font-size: var(--text-6xl);
  line-height: var(--tw-leading, var(--text-6xl--line-height));
}
.leading-tight {
  --tw-leading: var(--leading-tight);
  line-height: var(--leading-tight);
}
```

**アンチパターン**:
- Tailwind v3→v4 移行時に「`leading-*` が効かなくなった／急に効くようになった」を Tailwind のバグと誤診し、実際にはカスケード優先順位の仕様変更であることに気づかない

**出典引用**:
> "v4 では `leading-*` がレスポンシブな `text-*` に勝つ。v3 は逆で、後から出力されるレスポンシブ変種が行間を上書きしていた"
> ([Tailwind v4 で leading-* が勝つようになった](https://zenn.dev/ukintech/articles/tailwind-v4-leading-vs-responsive-text), セクション "なぜ変わったのか — 生成 CSS を見ると分かる") ※2026-08-23に実際にfetch成功

**バージョン**: Tailwind CSS v4+
**確信度**: 中（公式ツールのバージョン間挙動差を検証した単独記事、パターン1c採用）
**最終更新**: 2026-08-23

---

### 15. Tailwind CSS の Preflight は「タグ名で見た目を決める」習慣を意図的に壊す — 何がリセットされるかを把握してから外す

Tailwind CSS の Preflight（base layer）は `h1`〜`h6` の `font-size`/`font-weight` を `inherit` にリセットし、`ol`/`ul`/`menu` の `list-style` を `none` にし、全要素の `margin`/`padding` を 0 にする。これは「バグでリセットされている」のではなく、タグ名だけで見た目が決まる状態を防ぎ、スタイリングを常にユーティリティクラスで明示させるための設計。ただし `list-style: none` は VoiceOver 等のスクリーンリーダーからリスト要素の意味論を奪うため、`role="list"` を明示しないとアクセシビリティが壊れる。

**根拠**:
- ブラウザ既定の `h1` サイズ（2em 等）は多くの場合デザイナーのタイプスケールに存在せず、無自覚に使うとスケール体系が崩れる
- 「大きく見せたいから `h1` を使う」という動機は見た目上正しくても文書構造を破壊し、支援技術の利用者に影響する
- Preflight は margin/padding も含めて全称リセットするため、コンポーネント側は常に明示的なユーティリティクラスでスタイルを当てる前提になる
- `list-style: none` は見た目のリセットである一方、一部のスクリーンリーダー（VoiceOver など）ではリストの意味論も同時に失われるため、意味を残したい場合は `role="list"` の追加が必要

**コード例**:
```css
/* Tailwind CSS Preflight が実際に出力するリセット（抜粋） */
h1, h2, h3, h4, h5, h6 {
  font-size: inherit;
  font-weight: inherit;
}
ol, ul, menu {
  list-style: none;
}
*, ::after, ::before, ::backdrop, ::file-selector-button {
  margin: 0;
  padding: 0;
}
```
```html
<!-- list-style: none でVoiceOverのリスト読み上げが外れるため、意味を残すなら role="list" を明示する -->
<ul role="list">
  <li>One</li>
  <li>Two</li>
  <li>Three</li>
</ul>
```

**アンチパターン**:
- `<h1>` を「大きな見出しを出すタグ」として使い、Preflight のリセットに気づかず「なぜか大きくならない」と誤診する
- `list-style: none` だけ適用して `role="list"` を付け忘れ、スクリーンリーダー利用者からリストの構造情報を奪う

**出典引用**:
> "list-style: none はVoiceOverでの"リスト扱い"を外す"
> ([h1が大きくならないのはバグではなく、タグで見た目を決める癖を折る設計だった](https://zenn.dev/matsutake_prgrm/articles/preflight-breaks-the-habit-of-styling-by-tag), Zenn, セクション "ただし、意味まで無傷ではなかった") ※2026-09-05に実際にfetch成功

**バージョン**: Tailwind CSS v3+ (Preflight)
**確信度**: 中（公式ツールの base layer 挙動を検証した単独記事、パターン1c採用）
**最終更新**: 2026-09-05

---

### 16. CSS-in-JS をランタイムで解決させず、CSS Modules 等の静的抽出方式に移行する

CSS-in-JS ライブラリ（styled-components 等）はスタイルを実行時に生成・注入する（props に依存する動的なスタイルはレンダーごとに計算し、静的なスタイルも初回に生成してキャッシュする）ため、SSR とクライアント初期化の両方に実行時コストを持ち込む。GitHub は本番の CSS-in-JS を CSS Modules へ段階移行し、SSR 時間 55%減・コンポーネント初期化 25%高速化を達成した。

**根拠**:
- CSS-in-JS はスタイルを実行時に生成・注入する（動的なスタイルはレンダーのたびに計算する）ため、SSR・クライアント初期化の双方でJS実行コストが発生する
- CSS Modules はビルド時にスタイルを静的なCSSファイルへ抽出するため、ランタイムでのスタイル計算が不要になる
- 大規模移行は feature flag による段階的ロールアウト（社内→社員→一般ユーザー）と Visual Regression Testing での出力比較を組み合わせることで安全に進められる
- codemod や AI支援ツールによる自動書き換えで、数千箇所規模の移行も現実的な工数で実行できる
- 移行期間中はブリッジパッケージ（GitHub の例では `@primer/styled-react`）で移行済み・未移行のコンポーネントを共存させると、画面単位で段階的に切り替えられる

**コード例**:
```tsx
// Bad: CSS-in-JS は実行時にスタイルを生成・注入する（props に依存する場合はレンダーごとに計算する）
const Button = styled.button`
  padding: 8px 16px;
  border-radius: 4px;
`;

// Good: CSS Modules はビルド時に静的CSSへ抽出され、ランタイムコストがゼロになる
// Button.module.css
// .button { padding: 8px 16px; border-radius: 4px; }
import styles from './Button.module.css';
function Button() {
  return <button className={styles.button}>...</button>;
}
```

**アンチパターン**:
- 大規模な CSS-in-JS 移行を一括切り替えで行い、Visual Regression Testing なしでデプロイする（本番で見た目の差分に気づけない）
- feature flag を使わず全ユーザーに一斉ロールアウトし、問題発生時に切り戻せない状態を作る
- 性能上の問題が出ていない小規模なコードベースで既存の CSS-in-JS 基盤を書き換える（移行コストも大きく、投資対効果が出るのは大規模化してから）

**出典引用**:
> "With CSS Modules, styles would be authored in a CSS file alongside the JavaScript source for the component."
> ([Improving site performance by shipping more CSS](https://github.blog/engineering/architecture-optimization/improving-site-performance-by-shipping-more-css/), GitHub Blog, セクション "Introducing CSS (Modules)") ※2026-09-25に実際にfetch成功

**バージョン**: フレームワーク非依存（CSS Modules は Webpack/Vite 等のバンドラでネイティブサポート）
**確信度**: 中（公式 GitHub Engineering Blog の 1 記事）
**最終更新**: 2026-10-25

---
