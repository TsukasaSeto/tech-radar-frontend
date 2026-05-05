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
