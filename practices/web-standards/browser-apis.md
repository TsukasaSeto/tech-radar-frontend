# ブラウザ API のベストプラクティス

## ルール

### 1. `IntersectionObserver` でスクロール連動処理を実装する

スクロール位置の監視には `scroll` イベントの代わりに `IntersectionObserver` を使う。

**根拠**:
- `scroll` イベントはスクロールのたびに発火しメインスレッドを圧迫する
- `IntersectionObserver` はブラウザが最適なタイミングで非同期に呼び出す
- 遅延読み込み・無限スクロール・アニメーショントリガーに適している

**コード例**:
```tsx
'use client';
import { useEffect, useRef } from 'react';

// 遅延ローディング（要素がビューポートに入ったら読み込む）
function LazyImage({ src, alt }: { src: string; alt: string }) {
  const imgRef = useRef<HTMLImageElement>(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            const img = entry.target as HTMLImageElement;
            img.src = img.dataset.src ?? '';
            observer.unobserve(img);  // 一度読み込んだら監視解除
          }
        });
      },
      { rootMargin: '200px' }  // 200px 手前から読み込み開始
    );

    if (imgRef.current) observer.observe(imgRef.current);

    return () => observer.disconnect();
  }, []);

  return <img ref={imgRef} data-src={src} alt={alt} />;
}

// 無限スクロールのトリガー
function InfiniteList() {
  const sentinelRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const observer = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting) {
        loadMoreItems();
      }
    });

    if (sentinelRef.current) observer.observe(sentinelRef.current);
    return () => observer.disconnect();
  }, []);

  return (
    <>
      <ItemList />
      <div ref={sentinelRef} />  {/* ここが見えたら次のページを読み込む */}
    </>
  );
}
```

**出典**:
- [MDN: Intersection Observer API](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API) (MDN Web Docs)

**バージョン**: Chrome 51+, Firefox 55+, Safari 12.1+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. `ResizeObserver` でコンポーネントのサイズ変化を監視する

コンテナの幅変化に応じたレイアウト調整には `window.resize` イベントの代わりに
`ResizeObserver` を使う。

**根拠**:
- `window.resize` はウィンドウサイズ変化にしか反応しない（コンテナリサイズには非対応）
- `ResizeObserver` は要素単位でサイズ変化を監視できる
- Container Queries のポリフィルや動的なチャートリサイズに適している

**コード例**:
```tsx
'use client';
import { useEffect, useRef, useState } from 'react';

function useElementSize<T extends HTMLElement>() {
  const ref = useRef<T>(null);
  const [size, setSize] = useState({ width: 0, height: 0 });

  useEffect(() => {
    const observer = new ResizeObserver(entries => {
      const entry = entries[0];
      if (entry) {
        setSize({
          width: entry.contentRect.width,
          height: entry.contentRect.height,
        });
      }
    });

    if (ref.current) observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);

  return { ref, ...size };
}

// チャートのコンテナサイズに応じてリサイズ
function ResponsiveChart({ data }: { data: DataPoint[] }) {
  const { ref, width, height } = useElementSize<HTMLDivElement>();

  return (
    <div ref={ref} className="w-full h-64">
      {width > 0 && (
        <Chart data={data} width={width} height={height} />
      )}
    </div>
  );
}
```

**出典**:
- [MDN: ResizeObserver API](https://developer.mozilla.org/en-US/docs/Web/API/ResizeObserver) (MDN Web Docs)

**バージョン**: Chrome 64+, Firefox 69+, Safari 13.1+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. Web Storage は `localStorage` / `sessionStorage` を適切に使い分ける

クライアントサイドの永続化は `localStorage`（永続）と `sessionStorage`（タブスコープ）を
用途に応じて使い分け、機密情報は保存しない。

**根拠**:
- `localStorage` と `sessionStorage` はどちらもオリジン単位で隔離されている
- `sessionStorage` はタブを閉じると消えるため一時的なデータに適している
- アクセストークンなどの機密情報は Web Storage ではなく HttpOnly Cookie に保存すべき

**コード例**:
```tsx
// Good: ユーザー設定（永続的に保持する非機密データ）
const THEME_KEY = 'user-theme';

function useThemePreference() {
  const [theme, setTheme] = useState<'light' | 'dark'>(() => {
    // SSR では window が存在しないため typeof でチェック
    if (typeof window === 'undefined') return 'light';
    return (localStorage.getItem(THEME_KEY) as 'light' | 'dark') ?? 'light';
  });

  const updateTheme = (newTheme: 'light' | 'dark') => {
    setTheme(newTheme);
    localStorage.setItem(THEME_KEY, newTheme);
  };

  return { theme, updateTheme };
}

// Good: フォームの下書き（タブを閉じたら捨てる）
function useDraftSave(key: string) {
  const save = (value: string) => sessionStorage.setItem(key, value);
  const load = () => sessionStorage.getItem(key) ?? '';
  const clear = () => sessionStorage.removeItem(key);
  return { save, load, clear };
}

// Bad: 機密情報を localStorage に保存
localStorage.setItem('authToken', token);  // XSS で盗まれる可能性がある

// Good: 機密情報は HttpOnly Cookie に保存（JS からアクセスできない）
// サーバーサイドで Set-Cookie: token=xxx; HttpOnly; Secure; SameSite=Strict
```

**出典**:
- [MDN: Web Storage API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API) (MDN Web Docs)
- [OWASP: HTML5 Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html) (OWASP)

**バージョン**: すべてのモダンブラウザ
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. View Transitions API でページ・状態遷移にアニメーションを付ける

SPA でのルーティング遷移やUI状態の切り替えには `document.startViewTransition()` を使い、
宣言的なCSSアニメーションで滑らかな遷移を実装する。

**根拠**:
- ブラウザがスナップショットのキャプチャと合成を自動で行い、実装コストが低い
- CSS の `view-transition-name` で要素ごとにヒーローアニメーションが実現できる
- Chrome 111+、Safari 18+（Same-Document）でサポート済み；Cross-Document は Chrome 126+

**コード例**:
```tsx
// Good: Next.js App Router + View Transitions
// ルーター遷移にラップする（Next.js の場合は experimental.viewTransition が必要）

// シンプルな状態遷移の例
function TabPanel({ tabs }: { tabs: Tab[] }) {
  const [active, setActive] = useState(0);

  const switchTab = (index: number) => {
    // View Transitions API でアニメーション
    if (!document.startViewTransition) {
      setActive(index);  // 非対応ブラウザは即時切り替え
      return;
    }
    document.startViewTransition(() => {
      setActive(index);  // DOM の更新を transition の中で行う
    });
  };

  return (
    <div>
      <div role="tablist">
        {tabs.map((tab, i) => (
          <button
            key={tab.id}
            role="tab"
            aria-selected={active === i}
            onClick={() => switchTab(i)}
          >
            {tab.label}
          </button>
        ))}
      </div>
      {/* view-transition-name でヒーロー要素を指定 */}
      <div
        style={{ viewTransitionName: 'tab-content' }}
        role="tabpanel"
      >
        {tabs[active].content}
      </div>
    </div>
  );
}
```

```css
/* デフォルトのクロスフェードをカスタマイズ */
::view-transition-old(tab-content) {
  animation: 200ms ease-out both fade-out;
}
::view-transition-new(tab-content) {
  animation: 200ms ease-in both fade-in;
}

/* アクセシビリティ: モーション低減設定を尊重 */
@media (prefers-reduced-motion: reduce) {
  ::view-transition-old(*),
  ::view-transition-new(*) {
    animation: none;
  }
}

@keyframes fade-out { from { opacity: 1; } to { opacity: 0; } }
@keyframes fade-in  { from { opacity: 0; } to { opacity: 1; } }
```

**出典**:
- [View Transitions API - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/API/View_Transitions_API) (MDN Web Docs)
- [Smooth transitions with the View Transition API](https://developer.chrome.com/docs/web-platform/view-transitions/) (Chrome for Developers)

**バージョン**: Chrome 111+ (Same-Document), Chrome 126+ (Cross-Document), Safari 18+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 5. `AbortController` で fetch リクエストをキャンセルする

コンポーネントのアンマウントやユーザーの操作によって不要になった fetch は
`AbortController` でキャンセルし、不要なネットワーク処理とメモリリークを防ぐ。

**根拠**:
- コンポーネントがアンマウントされても fetch が完了すると `setState` が呼ばれ、メモリリークや警告が発生する
- `AbortController` は `fetch`・`axios`・`EventSource` など複数のAPIに対応する
- React では `useEffect` のクリーンアップ関数でキャンセルするパターンが標準的

**コード例**:
```tsx
// Good: useEffect + AbortController でリクエストをキャンセル
'use client';
import { useEffect, useState } from 'react';

function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const controller = new AbortController();
    const { signal } = controller;

    async function fetchUser() {
      try {
        const res = await fetch(`/api/users/${userId}`, { signal });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const data = await res.json();
        setUser(data);
      } catch (err) {
        if ((err as Error).name === 'AbortError') {
          return;  // キャンセルは正常系なので無視
        }
        setError('データの取得に失敗しました');
      }
    }

    fetchUser();

    // クリーンアップ: userId が変わるかアンマウント時にキャンセル
    return () => controller.abort();
  }, [userId]);

  if (error) return <p role="alert">{error}</p>;
  if (!user) return <p>読み込み中...</p>;
  return <div>{user.name}</div>;
}

// Bad: クリーンアップなし（メモリリーク・競合状態の原因）
useEffect(() => {
  fetch(`/api/users/${userId}`)
    .then(r => r.json())
    .then(data => setUser(data));  // アンマウント後も setState が呼ばれる
}, [userId]);

// 複数リクエストを同時キャンセルする場合も同一 signal を渡すだけ
const controller = new AbortController();
await Promise.all([
  fetch('/api/a', { signal: controller.signal }),
  fetch('/api/b', { signal: controller.signal }),
]);
controller.abort();  // 両方を一度でキャンセル
```

**出典**:
- [MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) (MDN Web Docs)
- [MDN: Fetch API - Canceling a request](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch#canceling_a_request) (MDN Web Docs)

**バージョン**: Chrome 66+, Firefox 57+, Safari 12.1+
**確信度**: 高
**最終更新**: 2026-05-06
