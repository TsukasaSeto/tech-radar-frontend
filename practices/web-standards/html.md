# HTML のベストプラクティス

## ルール

### 1. セマンティックな HTML 要素を使う

`<div>` / `<span>` の代わりに意味を持つ要素（`<article>`・`<nav>`・`<section>` など）を使う。

**根拠**:
- スクリーンリーダーなどの支援技術がページ構造を正しく解釈できる
- 検索エンジンがコンテンツの重要度を判断しやすくなる（SEO）
- CSSのターゲティングが明確になり、`class` の乱用を防ぐ

**コード例**:
```tsx
// Bad: 汎用コンテナの乱用
<div className="header">
  <div className="nav">
    <div className="nav-item">ホーム</div>
  </div>
</div>
<div className="main">
  <div className="article">
    <div className="article-header">タイトル</div>
  </div>
</div>

// Good: セマンティックな要素
<header>
  <nav aria-label="メインナビゲーション">
    <ul>
      <li><a href="/">ホーム</a></li>
      <li><a href="/about">About</a></li>
    </ul>
  </nav>
</header>
<main>
  <article>
    <h1>記事タイトル</h1>
    <p>記事本文...</p>
  </article>
  <aside>
    <h2>関連記事</h2>
  </aside>
</main>
<footer>
  <p>© 2026 Company Name</p>
</footer>
```

**出典**:
- [MDN: HTML elements reference](https://developer.mozilla.org/en-US/docs/Web/HTML/Element) (MDN Web Docs)
- [HTML Living Standard: Semantics](https://html.spec.whatwg.org/multipage/semantics.html) (WHATWG)

**バージョン**: HTML Living Standard
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. フォームは適切な `input type` と `label` を使う

フォームの各入力フィールドに対応する `<label>` を紐付け、
`type` 属性でモバイルキーボードと入力検証を最適化する。

**根拠**:
- `<label>` はクリック領域を広げ、スクリーンリーダーがフィールドを正しく読み上げる
- 適切な `type` によりモバイルで専用キーボード（数字・メール・電話）が表示される
- `autocomplete` 属性でブラウザの自動補完を制御できる

**コード例**:
```tsx
// Bad: label なし、type 未指定
<div>
  <span>メールアドレス</span>
  <input name="email" />  {/* type="text" デフォルト */}
</div>

// Good: 正しい label と type
<form>
  <div>
    <label htmlFor="email">メールアドレス</label>
    <input
      id="email"
      name="email"
      type="email"           // メール専用キーボード表示
      autoComplete="email"   // ブラウザの自動補完
      required
    />
  </div>

  <div>
    <label htmlFor="phone">電話番号</label>
    <input
      id="phone"
      name="phone"
      type="tel"             // 電話番号キーボード表示
      autoComplete="tel"
      pattern="[0-9]{10,11}"
    />
  </div>

  <div>
    <label htmlFor="birthdate">生年月日</label>
    <input
      id="birthdate"
      name="birthdate"
      type="date"            // 日付ピッカー表示
      autoComplete="bday"
    />
  </div>
</form>
```

**出典**:
- [MDN: The input element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input) (MDN Web Docs)
- [web.dev: Create Amazing Forms](https://web.dev/learn/forms/) (web.dev)

**バージョン**: HTML Living Standard
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. `<head>` のメタタグを適切に設定する

`charset`・`viewport`・OGP などの必須メタタグを設定し、
Next.js の `metadata` API を使って管理する。

**根拠**:
- `viewport` の設定がないとモバイルで正しく表示されない
- OGP タグがないと SNS シェア時にプレビューが表示されない
- `charset` の未設定は文字化けの原因になる

**コード例**:
```tsx
// app/layout.tsx - Next.js の Metadata API
import type { Metadata } from 'next';

export const metadata: Metadata = {
  // 基本
  title: {
    template: '%s | サイト名',
    default: 'サイト名',
  },
  description: 'サイトの説明文',
  charset: 'utf-8',  // Next.js が自動設定

  // OGP
  openGraph: {
    type: 'website',
    locale: 'ja_JP',
    url: 'https://example.com',
    siteName: 'サイト名',
    images: [
      {
        url: 'https://example.com/og-image.jpg',
        width: 1200,
        height: 630,
        alt: 'OGP画像の説明',
      },
    ],
  },

  // Twitter Card
  twitter: {
    card: 'summary_large_image',
    creator: '@twitter_handle',
  },

  // robots
  robots: {
    index: true,
    follow: true,
  },
};

// 個別ページで上書き
// app/blog/[slug]/page.tsx
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const post = await fetchPost(params.slug);
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: { title: post.title, description: post.excerpt },
  };
}
```

**出典**:
- [Next.js Docs: Metadata](https://nextjs.org/docs/app/building-your-application/optimizing/metadata) (Next.js公式)
- [The Open Graph protocol](https://ogp.me/) (OGP公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05
