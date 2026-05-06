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

function ConfirmDialog({
  open,
  onClose,
  onConfirm,
  title,
  description,
}: {
  open: boolean;
  onClose: () => void;
  onConfirm: () => void;
  title: string;
  description: string;
}) {
  const dialogRef = useRef<HTMLDialogElement>(null);

  useEffect(() => {
    const dialog = dialogRef.current;
    if (!dialog) return;
    if (open) {
      dialog.showModal();  // top-layer に表示、フォーカストラップ自動適用
    } else {
      dialog.close();
    }
  }, [open]);

  // dialog の close イベント（Escape キーでも発火）を親に通知
  return (
    <dialog
      ref={dialogRef}
      onClose={onClose}
      className="rounded-lg p-6 backdrop:bg-black/50"
      aria-labelledby="dialog-title"
    >
      <h2 id="dialog-title" className="text-lg font-bold">{title}</h2>
      <p className="mt-2 text-gray-600">{description}</p>
      <div className="mt-4 flex justify-end gap-2">
        <button onClick={onClose} className="btn-secondary">キャンセル</button>
        <button onClick={onConfirm} className="btn-danger">確認</button>
      </div>
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
