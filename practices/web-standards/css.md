# CSS のベストプラクティス

## ルール

### 1. Tailwind CSS ではユーティリティクラスを直接使い、カスタムCSSを最小化する

コンポーネントに直接ユーティリティクラスを記述し、
カスタム CSS は Tailwind のシステムで表現できない場合のみ書く。

**根拠**:
- ユーティリティファースでCSSファイルの肢大化を防ぐ
- クラス名を読むだけでスタイルが把握できる
- 使われていないスタイルは PurgeCSS で自動削除される

**コード例**:
```tsx
// Bad: カスタム CSS クラスを作成
.card {
  display: flex;
  flex-direction: column;
  padding: 1rem;
  border-radius: 0.5rem;
}
<div className={styles.card}>...</div>

// Good: Tailwind ユーティリティを直接使用
<div className="flex flex-col p-4 rounded-lg shadow-sm bg-white">...</div>

// cn() ユーティリティで条件付きクラスを管理
function Button({ variant, className, ...props }: ButtonProps) {
  return (
    <button
      className={cn(
        'rounded-lg px-4 py-2 font-medium',
        variant === 'primary' && 'bg-blue-600 text-white',
        variant === 'outline' && 'border border-gray-300 text-gray-700',
        className
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
- `prefers-color-scheme` メディアクエリと組み合わせてダークモードを実装できる

**コード例**:
```css
:root {
  --color-primary: #2563eb;
  --color-text: #111827;
  --color-background: #ffffff;
}

@media (prefers-color-scheme: dark) {
  :root {
    --color-text: #f9fafb;
    --color-background: #111827;
  }
}
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
- `min-width`（モバイルファースト）は `max-width` より特異性の衝突が少ない
- Tailwind CSS のブレークポイントがモバイルファーストを前提としている

**コード例**:
```tsx
<div className="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-4">
  {products.map(product => <ProductCard key={product.id} />)}
</div>
```

**出典**:
- [Tailwind CSS Docs: Responsive Design](https://tailwindcss.com/docs/responsive-design) (Tailwind CSS公式)

**バージョン**: Tailwind CSS 3+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. CSS Container Queries でコンポーネント単位のレスポンシブを実装する

Viewport幅ではなく親コンテナの幅に応じてスタイルを変化させる `@container` クエリを活用する。

**根拠**:
- メディアクエリはビューポート幅に依存するため、コンポーネントの再利用時にスタイルが崩れやすい
- Chrome 105+, Firefox 110+, Safari 16+ でサポート済み

**コード例**:
```css
.card-wrapper {
  container-type: inline-size;
}

@container (max-width: 400px) {
  .card { flex-direction: column; }
}

@container (min-width: 401px) {
  .card { flex-direction: row; }
}
```

**出典**:
- [CSS container queries - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_containment/Container_queries) (MDN Web Docs)

**バージョン**: Chrome 105+, Firefox 110+, Safari 16+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 5. CSS Nesting のネイティブサポートを活用する

プリプロセッサを使わずに、CSS ネイティブのネスト構文でセレクタを階層的に記述する。

**出典**:
- [CSS Nesting - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_nesting) (MDN Web Docs)

**バージョン**: Chrome 112+, Firefox 117+, Safari 17.2+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 6. `:has()` 疑似クラスで親要素を条件スタイリングする

`:has()` を使い、子要素の状態に応じて親要素のスタイルを変更する。

**出典**:
- [:has() CSS pseudo-class - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/CSS/:has) (MDN Web Docs)

**バージョン**: Chrome 105+, Firefox 121+, Safari 15.4+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 7. `@layer` でCSSカスケードの優先順位を明示的に管理する

`@layer` でスタイルをレイヤーに分割し、CSSの詳細度（specificity）や `!important` に頼らずに
スタイルの適用優先順位をコードで明示する。

**出典**:
- [最近のCSS、全然追えてなかった。こここ1、ዲ1年で使えるようになった機能10選](https://zenn.dev/seekseek/articles/css-new-features-catch-up-2026) (Zenn seekseek / 2026)
- [MDN: @layer](https://developer.mozilla.org/en-US/docs/Web/CSS/@layer) (MDN Web Docs)

**バージョン**: Chrome 99+, Firefox 97+, Safari 15.4+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 8. `scrollbar-gutter: stable` でスクロールバー出現によるレイアウトシフトを防ぐ

`scrollbar-gutter: stable` を `:root` に設定し、スクロールバーが出現・消失する際にページレイアウトがガタつく CLS を防ぐ。

**コード例**:
```css
:root {
  scrollbar-gutter: stable;
}
```

**出典**:
- [The Most Underrated CSS Property Nobody Talks About: scrollbar-gutter](https://medium.com/@konstantinkeylin/the-most-underrated-css-property-nobody-talks-about-scrollbar-gutter-2ca598352675) (Medium / 2026-05)
- [MDN: scrollbar-gutter](https://developer.mozilla.org/en-US/docs/Web/CSS/scrollbar-gutter) (MDN Web Docs)

**バージョン**: Chrome 94+, Firefox 97+, Safari 15.8+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 9. UIライブラリ配布時は CSSをJavaScriptにバンドルせず、明示的インポートで提供する

npm パッケージとして配布する UIライブラリでは、CSS を JavaScript ファイルに自動インポートする形式を避け、利用者がアプリ側で明示的に CSS ファイルをインポートする設計にする。

**根拠**:
- `import './button.css'` のような静的 CSS インポートが JavaScript に含まれると、SSR 環境（Node.js）で `Unknown file extension ".css"` エラーが発生する
- Next.js の App Router は Server Components でのスタイルシートのインポートを制限している
- 利用者がアプリの `globals.css` で `@import` するか `<link>` タグで読み込む設計にすることで、CSS の適用タイミングとバンドル方式を利用者が制御できる
- `package.json` の `exports` フィールドで CSS ファイルの存在を明示する

**コード例**:
```tsx
// Bad: ライブラリコンポーネント内で CSS を静的インポート（SSR でクラッシュ）
// packages/my-ui-lib/src/Button.tsx
import './button.css'; // ❌ Node.js: Unknown file extension ".css"
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

// 利用側アプリで明示的にインポート
// app/globals.css
// @import '@my-ui-lib/dist/styles.css';

// または layout.tsx でインポート
import '@my-ui-lib/dist/styles.css';
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return <html><body>{children}</body></html>;
}
```

**出典**:
- [Why UI Libraries Still Need Explicit CSS Imports](https://dev.to/ddtamn/why-ui-libraries-still-need-explicit-css-imports-5b6o) (dev.to ddtamn / 2026-05) ※2026-05-07に実際にfetch成功

**バージョン**: Next.js 14+, Node.js 20+
**確信度**: 高
**最終更新**: 2026-05-07
