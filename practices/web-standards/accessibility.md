# アクセシビリティのベストプラクティス

## ルール

### 1. インタラクティブ要素には適切な ARIA 属性を付与する

ネイティブ HTML 要素で表現できないインタラクションには ARIA 属性を使い、
スクリーンリーダーに意味を伝える。ただし「No ARIA is better than bad ARIA」。

**根拠**:
- スクリーンリーダーユーザーがUIの状態（展開/折りたたみ・選択中など）を知れる
- ARIA ロールと属性はアクセシビリティツリーを通じてユーザーに伝わる
- 誤った ARIA の使用はかえってアクセシビリティを損なう

**コード例**:
```tsx
// Accordion（折りたたみパネル）
function AccordionItem({ title, content }: { title: string; content: string }) {
  const [isOpen, setIsOpen] = useState(false);
  const contentId = useId();

  return (
    <div>
      <button
        aria-expanded={isOpen}       // 展開状態をスクリーンリーダーに伝える
        aria-controls={contentId}    // 制御する要素を指定
        onClick={() => setIsOpen(v => !v)}
      >
        {title}
      </button>
      <div
        id={contentId}
        role="region"
        aria-labelledby={/* button の id */undefined}
        hidden={!isOpen}             // hidden で非表示時にスクリーンリーダーからも隠す
      >
        {content}
      </div>
    </div>
  );
}

// ライブリージョン（動的なコンテンツ更新を通知）
function StatusMessage({ message }: { message: string }) {
  return (
    <div
      role="status"           // 割り込まない通知
      aria-live="polite"      // 現在の読み上げが終わったら通知
      aria-atomic="true"      // メッセージ全体を読み上げる
    >
      {message}
    </div>
  );
}

// エラーメッセージ（即座に通知）
<div role="alert" aria-live="assertive">
  フォームの送信に失敗しました
</div>
```

**出典**:
- [MDN: ARIA](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA) (MDN Web Docs)
- [WAI-ARIA Authoring Practices Guide](https://www.w3.org/WAI/ARIA/apg/) (W3C WAI)

**バージョン**: ARIA 1.2
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. キーボードナビゲーションを保証する

すべてのインタラクティブ要素はキーボードで操作できる必要がある。
フォーカス管理と `tabIndex` を適切に使う。

**根拠**:
- キーボードのみで操作するユーザー（運動障害を持つ方など）が利用できる
- スクリーンリーダーはキーボードナビゲーションに依存している
- WCAG 2.1 の Success Criterion 2.1.1（Keyboard）で要求されている

**コード例**:
```tsx
// Bad: onClick のみでキーボード非対応
<div onClick={handleClick} className="card">
  クリックしてください
</div>

// Good: button を使う（Enter/Space でフォーカス時に起動）
<button onClick={handleClick} className="card">
  クリックしてください
</button>

// モーダルのフォーカス管理
'use client';
import { useEffect, useRef } from 'react';

function Modal({ isOpen, onClose, children }: ModalProps) {
  const firstFocusableRef = useRef<HTMLButtonElement>(null);
  const modalRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (isOpen) {
      // モーダルが開いたら最初のフォーカス可能要素にフォーカス
      firstFocusableRef.current?.focus();
    }
  }, [isOpen]);

  // Escape キーで閉じる
  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    if (isOpen) document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return (
    <div
      ref={modalRef}
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
    >
      <h2 id="modal-title">タイトル</h2>
      {children}
      <button ref={firstFocusableRef} onClick={onClose}>閉じる</button>
    </div>
  );
}

// tabIndex={0}: タブ順序に追加（div を focusable にする最終手段）
// tabIndex={-1}: プログラム的なフォーカスのみ可能（タブ順序から除外）
```

**出典**:
- [WCAG 2.1: Keyboard Accessible](https://www.w3.org/TR/WCAG21/#keyboard-accessible) (W3C WCAG)

**バージョン**: WCAG 2.1
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. 色のみで情報を伝えない。十分なコントラスト比を確保する

情報の伝達に色だけを使わず、テキスト・アイコン・パターンを組み合わせる。
テキストのコントラスト比は WCAG AA 基準（4.5:1）以上を維持する。

**根拠**:
- 色覚特性（色盲）があるユーザーは色の区別が困難な場合がある
- コントラスト比が低いと視力の弱いユーザーがテキストを読めない
- WCAG 2.1 Success Criterion 1.4.1（Use of Color）と 1.4.3（Contrast）で要求される

**コード例**:
```tsx
// Bad: 色のみでエラーを示す
<input className="border-red-500" />  // 色盲のユーザーには分からない

// Good: 色 + アイコン + テキストを組み合わせる
<div>
  <input
    className="border-red-500"
    aria-invalid="true"
    aria-describedby="email-error"
  />
  <p id="email-error" className="text-red-600 flex items-center gap-1">
    <ExclamationIcon aria-hidden="true" />  {/* 装飾的なアイコンは aria-hidden */}
    <span>有効なメールアドレスを入力してください</span>
  </p>
</div>

// 白背景（#ffffff）に対する Tailwind カラーの目安:
// gray-700 (#374151) → 10.7:1 ✓（通常テキストに使用可）
// gray-500 (#6b7280) → 5.9:1 ✓（通常テキストに使用可）
// gray-400 (#9ca3af) → 2.9:1 ✗（装飾的用途のみ）
// blue-600 (#2563eb) → 5.9:1 ✓（リンクに使用可）
```

**出典**:
- [WCAG 2.1: Use of Color](https://www.w3.org/TR/WCAG21/#use-of-color) (W3C WCAG)
- [WCAG 2.1: Contrast (Minimum)](https://www.w3.org/TR/WCAG21/#contrast-minimum) (W3C WCAG)
- [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/) (WebAIM)

**バージョン**: WCAG 2.1
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. `aria-live` リージョンで動的コンテンツの変化をスクリーンリーダーに通知する

Ajax通信結果・フォームバリデーション・進捗状況など、DOM が動的に更新される箇所には
`aria-live` リージョンを適切に配置し、視覚的変化をスクリーンリーダーに伝える。

**根拠**:
- スクリーンリーダーはフォーカスが当たった要素しか読まないため、フォーカス外の更新は読み上げられない
- `aria-live="polite"` は現在の読み上げ完了後に通知し、`aria-live="assertive"` は即座に割り込む
- `aria-atomic="true"` を設定するとリージョン全体を一括で読み上げ、部分更新の誤読を防ぐ
- ライブリージョンはページ初期読み込み時から DOM に存在させ、後から追加しない

**根拠**:
- スクリーンリーダーはフォーカスのない DOM 更新を自動的には読み上げない
- WCAG 2.1 SC 4.1.3（Status Messages）は状態メッセージのプログラム的提供を要求する
- `assertive` の過剰使用は読み上げの中断を引き起こし体験を損なう

**コード例**:
```tsx
// Good: ライブリージョンをレイアウト時から DOM に配置
function App() {
  const [announcement, setAnnouncement] = useState('');

  return (
    <>
      {/* 常に DOM に存在させる（後から追加すると読み上げられないことがある） */}
      <div
        role="status"
        aria-live="polite"
        aria-atomic="true"
        className="sr-only"  {/* 視覚的には非表示 */}
      >
        {announcement}
      </div>
      <SearchPage onSearchComplete={(count) =>
        setAnnouncement(`${count}件の結果が見つかりました`)
      } />
    </>
  );
}

// 使い分けの指針
// aria-live="polite"    : 検索結果・保存完了・ページ遷移完了など（割り込み不要）
// aria-live="assertive" : エラー・タイムアウト警告など（即座の通知が必要）
// role="status"         : polite の省略形（aria-live="polite" + aria-atomic="true"）
// role="alert"          : assertive の省略形（aria-live="assertive" + aria-atomic="true"）

// Bad: setTimeout で強引に DOM に追加（初期配置されていないため読まれないことがある）
function BadToast({ message }: { message: string }) {
  return (
    <div aria-live="polite">  {/* 動的に追加されたリージョンは無視されるブラウザがある */}
      {message}
    </div>
  );
}

// フォームバリデーションの例
function EmailField() {
  const [error, setError] = useState('');
  return (
    <div>
      <label htmlFor="email">メールアドレス</label>
      <input
        id="email"
        type="email"
        aria-describedby="email-error"
        aria-invalid={error ? 'true' : 'false'}
        onBlur={e => {
          setError(e.target.validity.valid ? '' : '有効なメールアドレスを入力してください');
        }}
      />
      {/* role="alert" で即座に読み上げ */}
      <p id="email-error" role="alert" aria-live="assertive">
        {error}
      </p>
    </div>
  );
}
```

**出典**:
- [MDN: ARIA live regions](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/ARIA_Live_Regions) (MDN Web Docs)
- [WCAG 2.1 SC 4.1.3: Status Messages](https://www.w3.org/TR/WCAG21/#status-messages) (W3C WCAG)
- [WAI-ARIA 1.2: aria-live](https://www.w3.org/TR/wai-aria-1.2/#aria-live) (W3C WAI)

**バージョン**: ARIA 1.2 / すべてのモダンブラウザ
**確信度**: 高
**最終更新**: 2026-05-06

---

### 5. `focus-visible` と Focus Trap でフォーカス体験を最適化する

フォーカスリングはキーボードユーザーに見せ、マウスユーザーには不要な場合は非表示にする。
モーダル・ドロワーなどの重畳UIでは Focus Trap でフォーカスをコンテナ内に閉じ込める。

**根拠**:
- `:focus-visible` はキーボード操作時のみフォーカスリングを表示し、マウス操作時は非表示にできる
- Focus Trap がないモーダルではユーザーが Tab キーでモーダル外に出てしまい操作不能になる
- WCAG 2.1 SC 2.4.7（Focus Visible）はフォーカスインジケータの視認性を要求する

**コード例**:
```css
/* Good: :focus-visible でキーボードフォーカスリングを制御 */

/* デフォルトのフォーカスリングを除去し、:focus-visible のみに適用 */
:focus {
  outline: none;  /* マウスクリック時はリングなし */
}

:focus-visible {
  outline: 2px solid var(--color-primary);
  outline-offset: 2px;
  border-radius: 2px;
}

/* Tailwind CSS での設定 */
/* tailwind.config.ts */
/* focus-visible: をユーティリティクラスで使用 */
```

```tsx
/* タイロウィンドで focus-visible を使う */
<button className="focus:outline-none focus-visible:ring-2 focus-visible:ring-blue-500 focus-visible:ring-offset-2">
  送信
</button>

// Focus Trap の実装（モーダル・ドロワー等）
'use client';
import { useEffect, useRef } from 'react';

const FOCUSABLE_SELECTORS = [
  'a[href]',
  'button:not([disabled])',
  'input:not([disabled])',
  'select:not([disabled])',
  'textarea:not([disabled])',
  '[tabindex]:not([tabindex="-1"])',
].join(', ');

function useFocusTrap(isActive: boolean) {
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!isActive || !containerRef.current) return;

    const container = containerRef.current;
    const focusableElements = Array.from(
      container.querySelectorAll<HTMLElement>(FOCUSABLE_SELECTORS)
    );
    const firstEl = focusableElements[0];
    const lastEl = focusableElements[focusableElements.length - 1];

    // コンテナ内の最初の要素にフォーカスを移す
    firstEl?.focus();

    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;
      if (e.shiftKey) {
        // Shift+Tab: 最初の要素にいたら最後の要素へ
        if (document.activeElement === firstEl) {
          e.preventDefault();
          lastEl?.focus();
        }
      } else {
        // Tab: 最後の要素にいたら最初の要素へ
        if (document.activeElement === lastEl) {
          e.preventDefault();
          firstEl?.focus();
        }
      }
    };

    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [isActive]);

  return containerRef;
}

// 使用例（<dialog> を使う場合は Focus Trap は不要。div ベースのモーダルに使う）
function Drawer({ isOpen, onClose, children }: DrawerProps) {
  const containerRef = useFocusTrap(isOpen);

  return (
    <div
      ref={containerRef}
      role="dialog"
      aria-modal="true"
      aria-label="メニュー"
      className={`drawer ${isOpen ? 'drawer--open' : ''}`}
    >
      {children}
      <button onClick={onClose}>閉じる</button>
    </div>
  );
}
```

**出典**:
- [MDN: :focus-visible](https://developer.mozilla.org/en-US/docs/Web/CSS/:focus-visible) (MDN Web Docs)
- [WCAG 2.1 SC 2.4.7: Focus Visible](https://www.w3.org/TR/WCAG21/#focus-visible) (W3C WCAG)
- [WAI-ARIA Authoring Practices: Managing Focus](https://www.w3.org/WAI/ARIA/apg/practices/managing-focus/) (W3C WAI)

**バージョン**: Chrome 86+, Firefox 85+, Safari 15.4+（:focus-visible）
**確信度**: 高
**最終更新**: 2026-05-06

---

### 6. Skip Link でメインコンテンツへの直接ナビゲーションを提供する

ページ冒頭にメインコンテンツへジャンプする "Skip to main content" リンクを配置する。
通常は視覚的に隠し、Tab キーでフォーカスされた時のみ表示する。

**根拠**:
- キーボードユーザー・スクリーンリーダーユーザーは、ナビゲーションを毎回タブで通り抜けるとページ閲覧コストが極めて高い
- WCAG 2.1 SC 2.4.1（Bypass Blocks）は繰り返しコンテンツをスキップする手段の提供を要求する
- スクリーンリーダーには landmark navigation 機能があるが、すべてのユーザーが使えるわけではないため Skip Link との併用が推奨される
- 視覚的に常時表示する必要はなく、フォーカス時のみ表示することでデザインを損なわない

**コード例**:
```tsx
// app/layout.tsx (Next.js)
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja">
      <body>
        {/* Skip Link はページ最上部の最初のフォーカス可能要素 */}
        <a
          href="#main-content"
          className="
            sr-only
            focus:not-sr-only
            focus:fixed focus:top-2 focus:left-2 focus:z-50
            focus:bg-white focus:text-black focus:px-4 focus:py-2
            focus:ring-2 focus:ring-blue-500 focus:rounded
          "
        >
          メインコンテンツへスキップ
        </a>

        <header>
          <nav aria-label="メインナビゲーション">
            {/* 多数のリンク */}
          </nav>
        </header>

        {/* tabIndex={-1} で プログラム的にフォーカスを移せるようにする */}
        <main id="main-content" tabIndex={-1}>
          {children}
        </main>
      </body>
    </html>
  );
}

// Tailwind の sr-only ユーティリティ相当（自前で書く場合）
/*
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border-width: 0;
}
*/
```

**配置原則**:
- 最初のフォーカス可能要素として配置する（`<body>` 直下が望ましい）
- 複数の主要ランドマーク（メインコンテンツ・検索・主要ナビ）がある場合は Skip Link も複数置く
- ターゲット要素には `tabIndex={-1}` を付け、`#main-content` クリック後に確実にフォーカスを受けられるようにする

**出典**:
- [WCAG 2.1 SC 2.4.1: Bypass Blocks](https://www.w3.org/TR/WCAG21/#bypass-blocks) (W3C WCAG)
- [WebAIM: "Skip Navigation" Links](https://webaim.org/techniques/skipnav/) (WebAIM)
- [MDN: Keyboard accessibility - Skip links](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/Accessibility/HTML#skip_links) (MDN Web Docs)

**バージョン**: WCAG 2.1
**確信度**: 高
**最終更新**: 2026-05-16

---

### 7. タッチターゲットの最小サイズ（24-44 CSS px）を確保する

ボタン・リンク等のインタラクティブ要素は最小 24×24 CSS px（WCAG 2.2 AA）、推奨 44×44 CSS px（WCAG 2.1 AAA / Apple HIG）を確保する。
視覚的なアイコン自体が小さくても、`padding` や `::before` 疑似要素でタップ領域を拡張する。

**根拠**:
- 小さいタップ領域はモバイル・運動機能に制限のあるユーザーが誤タップする原因になる
- WCAG 2.2 SC 2.5.8（Target Size, Minimum）は 24×24 CSS px の最低限を要求する
- WCAG 2.1 SC 2.5.5（Target Size, Enhanced AAA）と Apple HIG は 44×44 CSS px を推奨する
- 例外: インラインテキストリンク、ブラウザのネイティブコントロール、隣接ターゲットに十分な間隔がある場合

**コード例**:
```tsx
// Bad: アイコンボタンが 16×16 px しかなく押しづらい
<button className="size-4">
  <CloseIcon className="size-4" />
</button>

// Good: アイコンは小さくても padding で 44×44 px を確保
<button className="
  inline-flex items-center justify-center
  size-11    {/* 44px × 44px */}
  rounded
  hover:bg-gray-100
">
  <CloseIcon className="size-4" aria-hidden="true" />
  <span className="sr-only">閉じる</span>
</button>

// 視覚的に小さくしたい場合は ::before で見えないタップ領域を拡張
<button className="
  relative size-6 inline-flex items-center justify-center
  before:absolute before:inset-[-10px]    {/* 上下左右に 10px 拡張 → 44px */}
  before:content-['']
">
  <CloseIcon className="size-6" />
</button>

// リンク同士の最小間隔（8px 以上が望ましい）
<nav>
  <ul className="flex gap-2">  {/* gap-2 = 8px */}
    {items.map(item => (
      <li key={item.id}>
        <a href={item.href} className="block px-4 py-3">
          {item.label}
        </a>
      </li>
    ))}
  </ul>
</nav>
```

**配色・コントラストとは別の独立した要件**:
- ボタンが見やすいだけでは不十分。物理的に押せるサイズが必要
- フォーム送信ボタン・モバイル UI で特に重要
- 例外でアイコンボタンを 24px にする場合は、隣接ターゲットとの間隔を 24px 以上空ける

**出典**:
- [WCAG 2.2 SC 2.5.8: Target Size (Minimum)](https://www.w3.org/TR/WCAG22/#target-size-minimum) (W3C WCAG)
- [WCAG 2.1 SC 2.5.5: Target Size (Enhanced)](https://www.w3.org/TR/WCAG21/#target-size-enhanced) (W3C WCAG)
- [Apple Human Interface Guidelines: Layout](https://developer.apple.com/design/human-interface-guidelines/layout) (Apple)
- [Material Design: Accessibility - Touch targets](https://m3.material.io/foundations/designing/structure#touch-targets) (Google)

**バージョン**: WCAG 2.2
**確信度**: 高
**最終更新**: 2026-05-16

---

### 8. `prefers-reduced-motion` を尊重してアニメーションを抑制する

CSS の `@media (prefers-reduced-motion: reduce)` または JS の `matchMedia` でユーザーの「視覚効果を控えめに」設定を検知し、
パララックス・大きなトランジション・自動再生アニメーションを停止または最小化する。

**根拠**:
- 前庭障害（vestibular disorder）を持つユーザーは動きの大きいアニメーションで吐き気・めまいを誘発する
- WCAG 2.1 SC 2.3.3（Animation from Interactions）はインタラクション起因のアニメーションを無効化する手段を要求する
- macOS / iOS / Windows / Android すべてに OS レベルの reduce motion 設定が存在し、ブラウザに伝播する
- ホバーフィードバック等の小さなアニメーションは保持してよい。「動きの大きさ」と「持続時間」を抑える

**コード例**:
```tsx
// CSS で全アニメーションを無効化
/* globals.css */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}

// より細やかに：特定の演出のみ無効化
.parallax {
  transform: translateY(var(--scroll-y));
  transition: transform 0.6s ease-out;
}
@media (prefers-reduced-motion: reduce) {
  .parallax {
    transform: none;
    transition: none;
  }
}

// Tailwind v3+ の motion-safe / motion-reduce 修飾子
<div className="motion-safe:animate-bounce motion-reduce:animate-none">
  ...
</div>

<button className="
  motion-safe:transition-transform motion-safe:hover:scale-105
  motion-reduce:transition-none
">
  押す
</button>

// JS で検知して制御（Framer Motion 等）
'use client';
import { useReducedMotion, motion } from 'framer-motion';

function AnimatedHero() {
  const shouldReduceMotion = useReducedMotion();

  return (
    <motion.section
      initial={{ opacity: 0, y: shouldReduceMotion ? 0 : 40 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: shouldReduceMotion ? 0 : 0.6 }}
    >
      ...
    </motion.section>
  );
}

// matchMedia で同等処理
const mq = window.matchMedia('(prefers-reduced-motion: reduce)');
if (!mq.matches) {
  // 通常アニメーションを再生
}
mq.addEventListener('change', (e) => {
  // 設定変更時の再描画
});
```

**判断基準（何を「動き」とみなすか）**:
- 大きな移動（>30 CSS px）、回転、スケール、パララックス → 抑制
- 不透明度のフェード（移動なし）、色相変化、3D 効果なしの小さなアイコン揺れ → 保持してよい
- 自動再生動画・GIF → コントロール（一時停止ボタン）を提供

**出典**:
- [MDN: prefers-reduced-motion](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion) (MDN Web Docs)
- [WCAG 2.1 SC 2.3.3: Animation from Interactions (AAA)](https://www.w3.org/TR/WCAG21/#animation-from-interactions) (W3C WCAG)
- [WebAIM: Designing for Accessibility - Animations](https://webaim.org/articles/seizure/) (WebAIM)
- [web.dev: prefers-reduced-motion](https://web.dev/articles/prefers-reduced-motion) (web.dev)

**バージョン**: Chrome 74+, Firefox 63+, Safari 10.1+
**確信度**: 高
**最終更新**: 2026-05-16

---

#### 追加根拠 (2026-05-06) — ルール1「インタラクティブ要素には適切な ARIA 属性を付与する」

新たに以下のドキュメントで「No ARIA is better than bad ARIA」の根拠が定量データで示された:
- [MDN Web Docs: ARIA index](https://raw.githubusercontent.com/mdn/content/main/files/en-us/web/accessibility/aria/index.md) (mdn/content / mainブランチ) ※2026-05-06に実際にfetch成功

WebAIM の調査データが MDN ARIA ページで引用: 「ARIA を使用しているページは ARIA を使用していないページと比較して、41% 多くのアクセシビリティエラーが検出された」。この統計は ARIA 誤用がアクセシビリティを損なうというルールの根拠を定量的に裏付けている。MDN も「まず可能な限りネイティブ HTML 要素を使用し、ARIA はネイティブで表現できないインタラクションのみに適用する」という同じ優先原則を明示している。

**確信度**: 既存（高）→ 高（WebAIM定量データで根拠強化）

---

#### 追加根拠 (2026-05-06) — ルール3「色のみで情報を伝えない。十分なコントラスト比を確保する」

新たに以下の記事でOSレベルのハイコントラスト設定への CSS 対応パターンが示された:
- [モダンCSSでWebの課題をスマートに解決！一歩進んだアクセシビリティ対応テクニック3選](https://zenn.dev/gemcook/articles/css-accessibility-tips3) (Zenn gemcook / 2026) ※2026-05-06に実際にfetch成功

`@media (prefers-contrast: more)` で OS のハイコントラストモードを検知し、テキストカラー・リンクカラー・プレースホルダーを強制的に高コントラスト値に上書きするパターン。`prefers-contrast: more` 対応は WCAG 最低基準（4.5:1）の達成後に行う補助的対応として位置づけられ、低視力ユーザーの OS 設定を尊重する実装になる。WCAG 最低基準を満たしたうえで、さらに高コントラストが必要なユーザー（弱視・老眼等）のために上乗せ対応できる。

**確信度**: 既存（高）→ 高（CSS実装パターン追加）
