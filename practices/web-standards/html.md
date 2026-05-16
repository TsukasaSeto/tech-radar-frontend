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

---

### 4. `<dialog>` 要素でモーダルをネイティブ実装する

モーダルダイアログの実装には `<div>` + z-index の手動制御ではなく、
HTMLネイティブの `<dialog>` 要素と `showModal()` メソッドを使う。

**根拠**:
- ブラウザがフォーカストラップ・Escapeキー・`::backdrop` を自動的に提供する
- `aria-modal` などのARIA属性を手動管理する必要がなく、セマンティクスが保証される
- Chrome 37+、Firefox 98+、Safari 15.4+ で完全サポート済み（top-layer に配置され z-index 不要）

**コード例**:
```html
<!-- Good: ネイティブ dialog 要素 -->
<dialog id="confirm-dialog" aria-labelledby="dialog-title">
  <h2 id="dialog-title">削除の確認</h2>
  <p>このアイテムを削除してもよろしいですか？</p>
  <div class="dialog-actions">
    <!-- type="submit" は form[method="dialog"] と連動して dialog を閉じる -->
    <form method="dialog">
      <button type="button" id="cancel-btn">キャンセル</button>
      <button type="submit" value="confirm">削除する</button>
    </form>
  </div>
</dialog>

<!-- Bad: div でモーダルを自作（フォーカス管理・z-index・ARIA を全手動） -->
<div class="modal-overlay" style="z-index: 9999;" role="dialog" aria-modal="true">
  <div class="modal-content">...</div>
</div>
```

```tsx
// React での使用例
'use client';
import { useEffect, useRef } from 'react';

function ConfirmDialog({ open, onClose, ...rest }: Props) {
  const dialogRef = useRef<HTMLDialogElement>(null);

  useEffect(() => {
    const dialog = dialogRef.current;
    if (!dialog) return;
    if (open) dialog.showModal();   // top-layer 表示 + フォーカストラップ自動
    else dialog.close();
  }, [open]);

  // close イベントは Escape キーでも発火する
  return (
    <dialog ref={dialogRef} onClose={onClose} className="backdrop:bg-black/50">
      {/* ...content */}
    </dialog>
  );
}
```

**出典**:
- [The dialog element - HTML Living Standard](https://html.spec.whatwg.org/multipage/interactive-elements.html#the-dialog-element) (WHATWG)
- [MDN: `<dialog>`: The Dialog element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/dialog) (MDN Web Docs)

**バージョン**: Chrome 37+, Firefox 98+, Safari 15.4+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 5. Popover API でポップオーバーUIをネイティブ実装する

ツールチップ・ドロップダウン・通知バナーなどのポップオーバーUIには
`popover` 属性と `popovertarget` 属性を使い、JavaScript を最小化する。

**根拠**:
- ブラウザが top-layer への配置・Escape 閉じ・Light Dismiss（外クリックで閉じる）を自動提供
- `<dialog>` と異なりフォーカストラップを行わず、ページとのインタラクションを維持できる
- Chrome 114+、Firefox 125+、Safari 17+ でサポート済み

**コード例**:
```html
<!-- Good: Popover API をネイティブ使用（JS 不要） -->
<button popovertarget="user-menu">メニューを開く</button>

<div id="user-menu" popover>
  <ul>
    <li><a href="/profile">プロフィール</a></li>
    <li><a href="/settings">設定</a></li>
    <li><button popovertarget="user-menu" popovertargetaction="hide">閉じる</button></li>
  </ul>
</div>

<!-- popover="manual": Light Dismiss を無効化（JS で明示的に制御） -->
<div id="notification" popover="manual">
  <p>保存しました</p>
</div>

<!-- Bad: JS で手動実装（外クリック監視・z-index・ARIA を全手動） -->
<div id="dropdown" class="dropdown hidden" style="z-index: 1000;">
  ...
</div>
```

```tsx
// React での使用例（型定義は現時点で手動拡張が必要な場合あり）
function NotificationToast({ message }: { message: string }) {
  const popoverRef = useRef<HTMLDivElement>(null);

  const show = () => (popoverRef.current as any)?.showPopover();
  const hide = () => (popoverRef.current as any)?.hidePopover();

  useEffect(() => {
    show();
    const timer = setTimeout(hide, 3000);  // 3秒後に自動非表示
    return () => clearTimeout(timer);
  }, [message]);

  return (
    <div
      ref={popoverRef}
      // @ts-expect-error -- popover is not yet in React's JSX types
      popover="manual"
      className="fixed bottom-4 right-4 rounded-lg bg-gray-800 px-4 py-2 text-white"
    >
      {message}
    </div>
  );
}
```

**出典**:
- [Popover API - HTML Living Standard](https://html.spec.whatwg.org/multipage/popover.html) (WHATWG)
- [MDN: Popover API](https://developer.mozilla.org/en-US/docs/Web/API/Popover_API) (MDN Web Docs)

**バージョン**: Chrome 114+, Firefox 125+, Safari 17+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 6. `<details name>` で排他的アコーディオンをネイティブ実装する

複数の `<details>` 要素に同じ `name` 属性を設定すると、グループ内で1つだけが開く
排他的アコーディオンを JavaScript なしで実現できる。

**根拠**:
- `name` 属性を共有した `<details>` は「ラジオグループ」と同様に排他動作する
- フォーカス管理・キーボード操作・スクリーンリーダー読み上げはブラウザが担保
- Chrome 120+、Firefox 130+、Safari 17.2+ でサポート済み
- ブラウザネイティブの3API（`<dialog>`・`popover`・`<details name>`）を目的別に使い分けることで
  ライブラリ依存を減らしバンドルサイズを削減できる

**コード例**:
```html
<!-- JS 不要: 同一 name で排他的アコーディオン -->
<details name="faq">
  <summary>Q: 返品できますか？</summary>
  <p>購入から30日以内であれば返品可能です。</p>
</details>
<details name="faq">
  <summary>Q: 配送にかかる日数は？</summary>
  <p>通常2〜3営業日でお届けします。</p>
</details>
<details name="faq">
  <summary>Q: 支払い方法は？</summary>
  <p>クレジットカードおよびコンビニ払いに対応しています。</p>
</details>
```

**使い分け早見表**（`<dialog>` / `popover` / `<details name>` の選択）:
| 用途 | 推奨API | 判断軸 |
|------|---------|--------|
| 確認ダイアログ・入力フォームを含むモーダル | `<dialog>` | ユーザーに必ず応答が必要 |
| ツールチップ・ドロップダウン・トースト | `popover` | ページ操作をブロックしない |
| 排他的アコーディオン・FAQ | `<details name>` | グループ内で1つだけ開く |

**出典引用**:
> "ブラウザネイティブのHTML機能だけで実装できるようになっています。`<dialog>`・`popover`・`<details name>` の3つを使いこなせば、JavaScriptをほとんど書かずにインタラクティブなUIが作れます。"
> ([JSなしで実装できる！`<dialog>`・`popover`・`<details name>` 使い分けガイド【2025年版】](https://zenn.dev/ui_memo/articles/efec949e5cd9d1), セクション "まとめ") ※2026-05-13に実際にfetch成功

**バージョン**: Chrome 120+, Firefox 130+, Safari 17.2+
**確信度**: 中
**最終更新**: 2026-05-13

---

### 7. 見出し階層（h1-h6）を文書構造に従って使う

`h1` から `h6` は文書のアウトラインを表現する。視覚的な大きさで選ばず、論理階層をスキップせずに使う。
ページに `h1` は通常 1 つ、セクションごとに 1 つ階層下げる。

**根拠**:
- スクリーンリーダーは見出しジャンプ機能で文書内を移動するため、階層が崩れるとナビゲーション不能になる
- WCAG 2.1 SC 1.3.1（Info and Relationships）は文書構造を機械可読にすることを要求する
- 検索エンジンは見出し階層をコンテンツ構造の手がかりとして利用する
- HTML5 のアウトラインアルゴリズム（`<section>` ベースの暗黙的 h1 リセット）はブラウザに実装されなかったため、明示的な `h1-h6` 番号で階層を表現する必要がある

**コード例**:
```tsx
// Bad: 視覚的な大きさで見出しレベルを選ぶ
<main>
  <h1>ページタイトル</h1>
  <h4>セクション 1</h4>   {/* h2 をスキップしている */}
  <h2>セクション 1.1</h2> {/* 親より深いレベルが浅い見出し */}
</main>

// Bad: ページに h1 が複数（or 不在）
<main>
  <h1>ナビゲーション</h1>
  <h1>記事タイトル</h1>
  <h1>サイドバー</h1>
</main>

// Good: 階層を守る
<main>
  <h1>記事タイトル</h1>
  <section>
    <h2>セクション 1</h2>
    <h3>サブセクション 1.1</h3>
    <h3>サブセクション 1.2</h3>
  </section>
  <section>
    <h2>セクション 2</h2>
  </section>
</main>

// 「視覚的な大きさ」と「論理的なレベル」を分けたい場合は class で見た目を制御
<h2 className="text-sm font-normal text-gray-500">関連記事</h2>
```

**ツールでの検証**:
- axe DevTools・Lighthouse の Accessibility 監査で見出しスキップを検出できる
- `eslint-plugin-jsx-a11y` の `heading-has-content` で空見出しを禁止

**出典**:
- [MDN: Heading Elements](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/Heading_Elements) (MDN Web Docs)
- [WCAG 2.1 SC 1.3.1: Info and Relationships](https://www.w3.org/TR/WCAG21/#info-and-relationships) (W3C WCAG)
- [HTML Living Standard: Headings and sections](https://html.spec.whatwg.org/multipage/sections.html#headings-and-sections) (WHATWG)

**バージョン**: HTML Living Standard / WCAG 2.1
**確信度**: 高
**最終更新**: 2026-05-16

---

### 8. ネイティブフォームバリデーションを起点に組み、JS は補強として使う

`required` / `pattern` / `minlength` / `type="email"` 等の HTML 属性でブラウザのバリデーションをまず効かせ、
`:user-invalid` 疑似クラスでユーザー操作後のみエラースタイルを当てる。
JS バリデーション（zod 等）はサーバー送信前の最終チェック・複雑なクロスフィールド検証に絞る。

**根拠**:
- ネイティブ属性は JS を読み込む前から動作するため、JS 失敗・遅延時もフォーム送信を防げる
- `:invalid` は初期表示時にも発火するため UX を損なうが、`:user-invalid` はユーザーが操作した後のみ発火する（Chrome 119+, Firefox 88+, Safari 16.5+）
- `ValidityState` API でフィールドごとのエラー種別（`valueMissing` / `typeMismatch` / `patternMismatch` 等）にアクセスできる
- スクリーンリーダーはネイティブ制約違反を自動的に通知する

**コード例**:
```tsx
// Good: ネイティブ属性 + :user-invalid でエラースタイル
<form>
  <label htmlFor="email">メールアドレス</label>
  <input id="email" name="email" type="email" required autoComplete="email" className="border peer" />
  {/* ユーザーが触った後のみ表示。:invalid だと初期表示で発火して UX を損なう */}
  <p className="hidden peer-[&:user-invalid]:block text-red-600">
    有効なメールアドレスを入力してください
  </p>
  <button type="submit">送信</button>
</form>

// ValidityState でカスタムメッセージを切り替え
<input
  type="email"
  required
  onInvalid={(e) => {
    const t = e.currentTarget;
    if (t.validity.valueMissing) t.setCustomValidity('メールアドレスは必須です');
    else if (t.validity.typeMismatch) t.setCustomValidity('メールアドレス形式で入力してください');
    else t.setCustomValidity('');
  }}
  onInput={(e) => e.currentTarget.setCustomValidity('')}
/>

// Bad: type="text" + JS で全部実装（JS 失敗時に何も検証されない）
<input type="text" onBlur={(e) => validateEmail(e.target.value)} />
```

`number` / `password` / `tel` / `url` 等の他 type も同パターン。`min` / `max` / `minLength` / `pattern` 属性を組み合わせる。

**JS バリデーションとの併用**:
- ネイティブ属性 → 第 1 層（ブラウザ）
- React Hook Form + zod 等 → 第 2 層（クロスフィールド検証・サーバーエラー表示）
- サーバー側 → 第 3 層（最終 source of truth）

**出典**:
- [MDN: Client-side form validation](https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Forms/Form_validation) (MDN Web Docs)
- [MDN: :user-invalid](https://developer.mozilla.org/en-US/docs/Web/CSS/:user-invalid) (MDN Web Docs)
- [MDN: ValidityState](https://developer.mozilla.org/en-US/docs/Web/API/ValidityState) (MDN Web Docs)
- [web.dev: Use JavaScript for more complex real-time validation](https://web.dev/learn/forms/validation) (web.dev)

**バージョン**: HTML Living Standard / `:user-invalid` は Chrome 119+, Firefox 88+, Safari 16.5+
**確信度**: 高
**最終更新**: 2026-05-16

---

### 9. `<time>` `<address>` `<abbr>` 等の意味要素で機械可読性を上げる

日時・住所・略語・引用などには対応する意味要素を使う。
特に `<time datetime="...">` は機械可読な ISO 8601 形式を `datetime` 属性に持たせ、表示テキストとは独立させる。

**根拠**:
- スクリーンリーダーは `<time datetime>` を「2026年5月16日」のように読み下す
- 検索エンジン・SNS のリッチプレビューは `<time>` `<address>` を構造化情報として解釈する
- `<abbr title>` で略語を展開できるため、スクリーンリーダーが読み上げ可能
- `<q>` `<blockquote cite>` で引用元 URL を機械可読に表現できる

**コード例**:
```tsx
// Bad: 単なる span / div
<div>2026年5月16日 18:30</div>
<div>東京都千代田区 1-1-1</div>
<div>HTML（HyperText Markup Language）</div>

// Good: 意味要素を使う

// 日時：datetime 属性は ISO 8601 形式
<time dateTime="2026-05-16T18:30:00+09:00">2026年5月16日 18:30</time>

// 期間
<time dateTime="P2D">2日間</time>

// 投稿日 + 更新日
<article>
  <h1>記事タイトル</h1>
  <p>
    投稿日：<time dateTime="2026-05-10">2026年5月10日</time>
    （更新：<time dateTime="2026-05-16">5月16日</time>）
  </p>
</article>

// 住所（連絡先の文脈で使う。本文中の地名には使わない）
<address>
  株式会社 Example<br />
  〒100-0001 東京都千代田区 1-1-1<br />
  <a href="mailto:info@example.com">info@example.com</a>
</address>

// 略語（初出時に展開）
<p>
  <abbr title="HyperText Markup Language">HTML</abbr> は
  <abbr title="World Wide Web Consortium">W3C</abbr> によって標準化されている。
</p>

// 引用
<blockquote cite="https://example.com/article">
  <p>引用本文…</p>
  <footer>— <cite>記事タイトル</cite></footer>
</blockquote>

// インライン引用
<p>彼は<q cite="https://example.com">設計は妥協の科学だ</q>と述べた。</p>

// 計測値
<p>
  メモリ使用量：<data value="1024">1 KB</data>
</p>
```

**注意点**:
- `<address>` は連絡先の文脈に限定する。本文中の地名・住所には使わない（HTML Living Standard 仕様）
- `<time>` の `datetime` 値は ISO 8601 準拠（`YYYY-MM-DD`, `YYYY-MM-DDThh:mm:ssZ`, `PnDTnHnMnS` 等）
- `<i>` `<b>` `<small>` も意味タグ（i=技術用語/外国語、b=製品名等の強調、small=副次的注釈）として復権しているが、強調は `<em>` `<strong>` を優先

**出典**:
- [MDN: `<time>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/time) (MDN Web Docs)
- [MDN: `<address>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/address) (MDN Web Docs)
- [MDN: `<abbr>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/abbr) (MDN Web Docs)
- [HTML Living Standard: Text-level semantics](https://html.spec.whatwg.org/multipage/text-level-semantics.html) (WHATWG)

**バージョン**: HTML Living Standard
**確信度**: 高
**最終更新**: 2026-05-16

---

### 10. `<html lang>` と部分要素の `lang` で言語境界を明示する

ルート `<html>` には必ず `lang` 属性を設定し、本文中の他言語部分にも個別に `lang` を付与する。
これによりスクリーンリーダーの発音切り替え・自動翻訳の精度・検索インデックスの言語判別が正しく機能する。

**根拠**:
- スクリーンリーダーは `lang` 属性に応じて発音エンジンを切り替える（日本語の中の英語名を英語発音で読む）
- WCAG 2.1 SC 3.1.1（Language of Page）と SC 3.1.2（Language of Parts）で要求される
- ブラウザの自動翻訳機能（Chrome Translate 等）が誤訳を避けるために言語境界を参照する
- 検索エンジンは `lang` を国・言語別インデックスの手がかりにする

**コード例**:
```tsx
// Bad: lang 属性なし、または英語固定
<html>
<html lang="en">  {/* 日本語サイトなのに en */}

// Good: ルートに正しい lang
// app/layout.tsx (Next.js)
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja">
      <body>{children}</body>
    </html>
  );
}

// 多言語サイトでは locale から動的に
import { getLocale } from 'next-intl/server';

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const locale = await getLocale();
  return (
    <html lang={locale}>
      <body>{children}</body>
    </html>
  );
}

// 本文中の他言語部分
<p>
  この記事は <span lang="en">Server Components</span>（サーバーコンポーネント）について解説する。
</p>

<p>
  ドイツ語で <q lang="de">Schadenfreude</q> は「他人の不幸を喜ぶ感情」を意味する。
</p>

// 引用は cite + lang で
<blockquote lang="en" cite="https://example.com">
  <p>The best way to predict the future is to invent it.</p>
</blockquote>
```

**BCP 47 形式の値**:
- 言語のみ: `ja`, `en`, `de`
- 地域込み: `en-US`, `en-GB`, `zh-CN`, `zh-TW`, `pt-BR`
- スクリプト指定: `zh-Hans`, `zh-Hant`（簡体字 / 繁体字）

**RTL 言語（右から左）の場合は `dir` も**:
```html
<html lang="ar" dir="rtl">
<p lang="he" dir="rtl">…</p>
```

**出典**:
- [MDN: lang attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/lang) (MDN Web Docs)
- [WCAG 2.1 SC 3.1.1: Language of Page](https://www.w3.org/TR/WCAG21/#language-of-page) (W3C WCAG)
- [WCAG 2.1 SC 3.1.2: Language of Parts](https://www.w3.org/TR/WCAG21/#language-of-parts) (W3C WCAG)
- [BCP 47: Tags for Identifying Languages](https://www.rfc-editor.org/info/bcp47) (IETF)

**バージョン**: HTML Living Standard / WCAG 2.1
**確信度**: 高
**最終更新**: 2026-05-16

---

### 11. リソースヒント（`preload` / `preconnect` / `dns-prefetch`）はクリティカルリソース限定で使う

`<link rel="preload">` `<link rel="preconnect">` 等のリソースヒントは強力な反面、誤用するとメインリソースの帯域を奪う。
LCP に直接寄与するヒーロー画像・クリティカル CSS・必須サードパーティドメイン以外には使わない。

**根拠**:
- `preload` はリソースを高優先度で先読みするが、過剰使用するとブラウザの優先度判定を上書きしてクリティカルパスを遅らせる
- `preconnect` は DNS + TCP + TLS を事前に確立する。ただし接続は維持コストがあるため 4-6 ドメイン程度が上限
- `dns-prefetch` は DNS 解決のみで軽量。`preconnect` をサポートしない古いブラウザ用のフォールバック
- `prefetch` は次ページ用のアイドル時取得。LCP には使わない

**コード例**:
```tsx
// Next.js App Router での実装
// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja">
      <head>
        {/* 同一オリジンの重要画像（LCP 対象） */}
        <link
          rel="preload"
          as="image"
          href="/hero.webp"
          fetchPriority="high"
          imageSrcSet="/hero-800.webp 800w, /hero-1600.webp 1600w"
          imageSizes="100vw"
        />

        {/* CDN への事前接続（クリティカルなアセットがある場合のみ） */}
        <link rel="preconnect" href="https://cdn.example.com" crossOrigin="anonymous" />

        {/* preconnect のフォールバック（古いブラウザ） */}
        <link rel="dns-prefetch" href="https://cdn.example.com" />

        {/* フォント preload は woff2 のみ・crossorigin 必須 */}
        <link
          rel="preload"
          as="font"
          type="font/woff2"
          href="/fonts/main.woff2"
          crossOrigin="anonymous"
        />
      </head>
      <body>{children}</body>
    </html>
  );
}

// Bad: 全画像を preload（優先度判定を破壊）
<link rel="preload" as="image" href="/photo-1.jpg" />
<link rel="preload" as="image" href="/photo-2.jpg" />
<link rel="preload" as="image" href="/photo-3.jpg" />
{/* …続く */}

// Bad: 不要な preconnect でハンドシェイクコネクションを浪費
<link rel="preconnect" href="https://analytics.example.com" />
<link rel="preconnect" href="https://ads.example.com" />
<link rel="preconnect" href="https://chat.example.com" />
{/* …続く（実際にはサードパーティスクリプト遅延化と組み合わせるべき） */}
```

**使い分け基準**:
| 用途 | リソースヒント | 適用条件 |
|------|---------------|----------|
| LCP 画像 | `preload` + `fetchpriority="high"` | 計測で LCP ボトルネックを確認 |
| Web フォント | `preload as="font" crossorigin` | woff2、`font-display: optional` 推奨 |
| 重要なサードパーティドメイン | `preconnect` | 1ドメイン1回、4-6 個まで |
| 古ブラウザ用フォールバック | `dns-prefetch` | `preconnect` の補助 |
| 次ページ予測取得 | `prefetch` | アイドル時、UX 重視のページ遷移 |

**出典**:
- [MDN: rel="preload"](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/preload) (MDN Web Docs)
- [MDN: rel="preconnect"](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/preconnect) (MDN Web Docs)
- [web.dev: Preload critical assets](https://web.dev/articles/preload-critical-assets) (web.dev)
- [web.dev: Establish network connections early](https://web.dev/articles/preconnect-and-dns-prefetch) (web.dev)

**バージョン**: HTML Living Standard / すべてのモダンブラウザ
**確信度**: 高
**最終更新**: 2026-05-16

---

### 12. 画像のデフォルトを `loading="lazy"` + `decoding="async"` にし、LCP 画像のみ例外扱いにする

ファーストビュー外の画像には `loading="lazy"` と `decoding="async"` を付与してメインスレッド・ネットワーク帯域を節約する。
LCP 候補となるヒーロー画像のみ `loading="eager"` + `fetchpriority="high"` を明示する。

**根拠**:
- `loading="lazy"` はビューポート近くまでスクロールされるまで画像取得を遅延する（Chrome 77+, Firefox 75+, Safari 15.4+）
- `decoding="async"` は画像デコードをメインスレッドから外し、レンダリングブロックを避ける
- ファーストビュー画像に `loading="lazy"` を付けると LCP が遅延するため除外が必要
- Next.js の `<Image>` コンポーネントは `priority` 属性で内部的にこれらを切り替える

**コード例**:
```tsx
// Plain HTML / JSX
// Good: ファーストビュー外はデフォルト lazy
<img
  src="/photo.jpg"
  alt="商品写真"
  width={800}
  height={600}
  loading="lazy"
  decoding="async"
/>

// Good: LCP 候補（ファーストビュー上部の主要画像）は eager + 高優先度
<img
  src="/hero.webp"
  alt="ヒーロー画像"
  width={1600}
  height={800}
  loading="eager"
  decoding="sync"
  fetchPriority="high"
/>

// Next.js Image での同等表現
import Image from 'next/image';

// LCP 候補
<Image src="/hero.webp" alt="..." width={1600} height={800} priority />

// それ以外（デフォルトで lazy + async）
<Image src="/thumbnail.jpg" alt="..." width={400} height={300} />

// Bad: ファーストビュー画像が lazy で LCP が遅延
<Image src="/hero.jpg" alt="..." width={1600} height={800} loading="lazy" />

// Bad: 全画像 eager（帯域を浪費）
{photos.map(p => <img key={p.id} src={p.url} loading="eager" />)}
```

**判定基準（どの画像が "LCP 候補" か）**:
1. ビューポート上部 600px 以内に表示される
2. 視覚的に大きい（画面の 20% 以上を占める）
3. ページの主題を表現している

該当する画像は通常 1 枚（多くて 2-3 枚）。残りはすべて lazy で良い。

**ブラウザ互換**:
- `loading="lazy"`: Chrome 77+, Firefox 75+, Safari 15.4+ — 非対応ブラウザでは無視される（フォールバック不要）
- `decoding="async"`: 全モダンブラウザでサポート
- `fetchpriority`: Chrome 101+, Safari 17.2+ — 非対応ブラウザは `auto` 扱い

**出典**:
- [MDN: HTMLImageElement.loading](https://developer.mozilla.org/en-US/docs/Web/API/HTMLImageElement/loading) (MDN Web Docs)
- [MDN: HTMLImageElement.decoding](https://developer.mozilla.org/en-US/docs/Web/API/HTMLImageElement/decoding) (MDN Web Docs)
- [web.dev: Browser-level image lazy-loading](https://web.dev/articles/browser-level-image-lazy-loading) (web.dev)
- [web.dev: Optimizing Largest Contentful Paint](https://web.dev/articles/optimize-lcp) (web.dev)

**バージョン**: HTML Living Standard / `fetchpriority` は Chrome 101+, Safari 17.2+
**確信度**: 高
**最終更新**: 2026-05-16

---
