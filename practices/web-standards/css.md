# CSS のベストプラクティス

## ルール

### 1. Tailwind CSS ではユーティリティクラスを直接使い、カスタムCSSを最小化する

コンポーネントに直接ユーティリティクラスを記述し、
カスタム CSS は Tailwind のシステムで表現できない場合のみ書く。

**根拠**:
- ユーティリティファーストでCSSファイルの肖大化を防ぐ
- クラス名を読むだけでスタイルが把握できる
- 使われていないスタイルは PurgeCSS で自動削除される

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
**最終更新**: 2026-05-05

---

### 2. CSS カスタムプロパティ（変数）でデザイントークンを管理する

カラー・スペーシング・フォントサイズなどのデザイントークンは
CSS カスタムプロパティで定義し、テーマ切り替えに対応する。

**根拠**:
- CSS カスタムプロパティはJavaScriptで動的に変更でき、テーマ切り替えが容易
- プリプロセッサ（Sass）の変数と異なりカスケードとスコープが効く
- `prefers-color-scheme` メディアクエリと組み合わせてダークモードを実装できる

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

/* ダークモード */
@media (prefers-color-scheme: dark) {
  :root {
    --color-text: #f9fafb;
    --color-background: #111827;
    --color-surface: #1f2937;
  }
}

/* data 属性でのテーマ切り替え（JS制御）*/
[data-theme="dark"] {
  --color-text: #f9fafb;
  --color-background: #111827;
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

**バージョン**: CSS Living Standard
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. レスポンシブデザインはモバイルファーストで実装する

スタイルはモバイル向けをベースに書き、`min-width` メディアクエリで
画面が広くなるにつれてスタイルを上書きする。

**根拠**:
- モバイルユーザーが多い現代ではモバイルを優先することが自然
- `min-width`（モバイルファースト）は `max-width` より特異性の衝突が少ない
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
**最終更新**: 2026-05-05

---

### 4. CSS Container Queries でコンポーネント単位のレスポンシブを実装する

Viewport幅ではなく親コンテナの幅に応じてスタイルを変化させる `@container` クエリを活用する。
サイドバーに配置されるカードや、再利用コンポーネントのような「置かれる場所によって見た目が変わる」UIに最適。

**根拠**:
- メディアクエリはビューポート幅に依存するため、コンポーネントの再利用時にスタイルが崩れやすい
- Container Queries はコンポーネントが置かれたコンテキスト（親の幅）を基準にできる
- Chrome 105+、Firefox 110+、Safari 16+ でサポートされておりモダンブラウザでは実用段階

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

**バージョン**: Chrome 105+, Firefox 110+, Safari 16+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 5. CSS Nesting のネイティブサポートを活用する

プリプロセッサ（Sass/Less）を使わずに、CSS ネイティブのネスト構文で
セレクタを階層的に記述する。

**根拠**:
- Chrome 112+、Firefox 117+、Safari 17.2+ でサポート済み（2024年時点でモダンブラウザは全対応）
- Sass のネスト記法と互換性が高く、移行コストが低い
- ビルドステップなしでネスト構造が使え、コンポーネントスタイルの見通しが良くなる

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

**バージョン**: Chrome 112+, Firefox 117+, Safari 17.2+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 6. `:has()` 疑似クラスで親要素を条件スタイリングする

`:has()` を使い、子要素の状態に応じて親要素のスタイルを変更する。
「チェックされたチェックボックスを含むラベル」や「画像を含むカード」のような
従来 JavaScript が必要だったパターンを純粋な CSS で実現できる。

**根拠**:
- CSS の長年の課題だった「親セレクタ」がネイティブで実現された
- Chrome 105+、Firefox 121+、Safari 15.4+ でサポート済み
- フォームのバリデーション状態など、動的なスタイル変更をJSなしで実装できる

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

/* フォームが invalid な入力を含む場合にサブミットボタンを無効化 */
form:has(input:invalid) .submit-button {
  opacity: 0.5;
  pointer-events: none;
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

**バージョン**: Chrome 105+, Firefox 121+, Safari 15.4+
**確信度**: 高
**最終更新**: 2026-05-06
