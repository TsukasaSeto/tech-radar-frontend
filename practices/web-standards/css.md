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
