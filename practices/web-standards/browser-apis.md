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
