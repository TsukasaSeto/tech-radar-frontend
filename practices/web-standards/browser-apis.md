# ブラウザ API のベストプラクティス

## ルール

### 1. `IntersectionObserver` でスクロール連動処理を実装する

要素がビューポート（や指定した領域）に入った・出たことの検出（無限スクロールのトリガー、表示時のアニメーション、既読判定など）には、`scroll` イベントでの位置計算の代わりに `IntersectionObserver` を使う。
スクロール量そのものに連動する処理（パララックス・進捗バーなど）は対象外。画像の遅延読み込みは IO で自作せず、ネイティブの `loading="lazy"` を使う（`practices/web-standards/html.md` #12）。

**根拠**:
- `scroll` イベントはスクロールのたびに発火しメインスレッドを圧迫する
- `IntersectionObserver` はブラウザが最適なタイミングで非同期に呼び出す
- 無限スクロール・アニメーショントリガー・表示判定に適している
- `data-src` を IO で `src` に差し替える自前の遅延読み込みは、`width` / `height` を持たない `<img>` になりやすく CLS を起こし、JS が動くまで画像が読み込まれない。ネイティブの `loading="lazy"` なら不要

**コード例**:
```tsx
'use client';
import { useEffect, useRef } from 'react';

// 無限スクロールのトリガー（番兵要素が見えたら次のページを読み込む）
function InfiniteList({
  onLoadMore,
  children,
}: {
  onLoadMore: () => void;
  children: React.ReactNode;
}) {
  const sentinelRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const sentinel = sentinelRef.current;
    if (!sentinel) return;

    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0]?.isIntersecting) onLoadMore();
      },
      { rootMargin: '200px' },  // 200px 手前で読み込みを始める
    );
    observer.observe(sentinel);
    return () => observer.disconnect();
  }, [onLoadMore]);

  return (
    <>
      {children}
      <div ref={sentinelRef} />
    </>
  );
}

// Bad: 画像の遅延読み込みを IO で自作する（width / height なしで CLS、src prop の変更にも追従しない）
// <img ref={imgRef} data-src={src} alt={alt} />

// Good: 画像はネイティブの loading="lazy" に任せる
// <img src={src} alt={alt} width={800} height={600} loading="lazy" decoding="async" />
```

**出典**:
- [MDN: Intersection Observer API](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API) (MDN Web Docs)

**バージョン**: Chrome 51+, Firefox 55+, Safari 12.1+
**確信度**: 高
**最終更新**: 2026-10-25

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
'use client';
import { useSyncExternalStore } from 'react';

type Theme = 'light' | 'dark';
const THEME_KEY = 'user-theme';

function subscribe(callback: () => void) {
  // 他のタブでの変更は storage イベントで届く
  window.addEventListener('storage', callback);
  return () => window.removeEventListener('storage', callback);
}

function getSnapshot(): Theme {
  return localStorage.getItem(THEME_KEY) === 'dark' ? 'dark' : 'light';
}

// サーバーと hydration 直後は固定値を返し、初回描画をサーバーと一致させる（hydration mismatch を防ぐ）。
// useState(() => localStorage...) の初期化はサーバーとクライアントで値が変わるので使わない。
// 初回表示から保存済みのテーマで描画したい場合は、cookie に保存してサーバーで読む
function getServerSnapshot(): Theme {
  return 'light';
}

function useThemePreference() {
  const theme = useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);

  const updateTheme = (newTheme: Theme) => {
    localStorage.setItem(THEME_KEY, newTheme);
    // storage イベントは書き込んだタブ自身には届かないので、自分で通知する
    window.dispatchEvent(new StorageEvent('storage', { key: THEME_KEY }));
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
- [React: useSyncExternalStore](https://react.dev/reference/react/useSyncExternalStore) (React 公式、`getServerSnapshot` はサーバーと hydration で同じ値を返す必要があること) ※2026-10-25 確認

**バージョン**: すべてのモダンブラウザ
**確信度**: 高
**最終更新**: 2026-10-25

---

### 4. View Transitions API でページ・状態遷移にアニメーションを付ける

SPA でのルーティング遷移やUI状態の切り替えには `document.startViewTransition()` を使い、
宣言的なCSSアニメーションで滑らかな遷移を実装する。

**根拠**:
- ブラウザがスナップショットのキャプチャと合成を自動で行い、実装コストが低い
- CSS の `view-transition-name` で要素ごとにヒーローアニメーションが実現できる
- Same-Document は Chrome 111+、Safari 18+、Firefox 144+ でサポート済み（Baseline Newly available: 2025-10）。Cross-Document は Chrome 126+、Safari 18.2+（Firefox は未対応）
- React の状態更新で DOM を変える場合は、コールバックの中で `flushSync` を使う。`startViewTransition` は新しい状態のスナップショットをコールバック完了直後に取るが、`setState` だけでは React がその時点までに DOM を更新する保証がない

**コード例**:
```tsx
// Good: document.startViewTransition() を直接使う状態遷移の例
// （Next.js の experimental.viewTransition は React の <ViewTransition> を App Router の遷移と連携させる設定で、
//   ネイティブ API を直接呼ぶだけなら不要）
'use client';
import { useState } from 'react';
import { flushSync } from 'react-dom';

type Tab = { id: string; label: string; content: React.ReactNode };

function TabPanel({ tabs }: { tabs: Tab[] }) {
  const [active, setActive] = useState(0);

  const switchTab = (index: number) => {
    if (typeof document.startViewTransition !== 'function') {
      setActive(index);  // 非対応ブラウザは即時切り替え
      return;
    }
    document.startViewTransition(() => {
      // flushSync で、新しい状態のスナップショットを取る前に DOM へ反映させる
      flushSync(() => setActive(index));
    });
  };

  // ARIA の tab / tablist / tabpanel ロールを付けるなら、APG の Tabs パターン（矢印キー操作・
  // roving tabindex・aria-controls / aria-labelledby）をすべて実装する。この例ではロールを付けず、
  // 選択状態を aria-pressed で伝えるボタン群にしている
  return (
    <div>
      <div>
        {tabs.map((tab, i) => (
          <button
            key={tab.id}
            type="button"
            aria-pressed={active === i}
            onClick={() => switchTab(i)}
          >
            {tab.label}
          </button>
        ))}
      </div>
      {/* view-transition-name でヒーロー要素を指定 */}
      <div style={{ viewTransitionName: 'tab-content' }}>
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

**React での利用（React 19.3+ で stable）**:
React は `<ViewTransition>` コンポーネントで `document.startViewTransition()` をラップし、`Suspense` の `fallback`（スケルトン画面）から実コンテンツへの切り替わりにもアニメーションを付けられる。React 19.3.0（2026-09-09リリース）で `<ViewTransition />` と `addTransitionType` が正式機能として安定版に入り、canary 限定ではなくなった。ただしアニメーション自体が数百ミリ秒の遅延を追加するため、体感待ち時間を減らすはずのスケルトン画面の目的と部分的に矛盾しうる点は変わらず注意が必要。

```tsx
// React 19.3+ (stable): Suspense + ViewTransition でスケルトン→実コンテンツを滑らかに切り替え
<ViewTransition update="thumbnail-reveal">
  <Suspense fallback={<SkeletonThumbnail />}>
    <ThumbnailContent promise={promise} />
  </Suspense>
</ViewTransition>
```
```css
::view-transition-new(.thumbnail-reveal) {
  animation: thumbnail-circle-reveal 680ms cubic-bezier(0.16, 1, 0.3, 1) both;
}
```

**出典**:
- [View Transitions API - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/API/View_Transitions_API) (MDN Web Docs)
- [Smooth transitions with the View Transition API](https://developer.chrome.com/docs/web-platform/view-transitions/) (Chrome for Developers)
- [web-features: view-transitions](https://github.com/web-platform-dx/web-features/blob/main/features/view-transitions.yml) / [cross-document-view-transitions](https://github.com/web-platform-dx/web-features/blob/main/features/cross-document-view-transitions.yml) (W3C WebDX CG、Same-Document は 2025-10 に Baseline Newly available、Cross-Document は Firefox 未対応) ※2026-10-25 確認
- [スケルトンスクリーンからコンテンツへの切り替わりをアニメーションする（ViewTransition & Suspense）](https://zenn.dev/dentyu/articles/animate-skeleton-with-view-transition) (Zenn、React `<ViewTransition>` + `Suspense` の実装例、canary版の制約とレイテンシのトレードオフ) ※2026-08-06に実際にfetch成功
- [CHANGELOG.md (React 19.3.0)](https://github.com/facebook/react/blob/main/CHANGELOG.md) (React 公式リポジトリ、`<ViewTransition />` が "New React Features" として stable リリースに掲載されていることを確認、canary限定の記述を修正する根拠) ※2026-09-25に実際にfetch成功

> "Reactの`<ViewTransition>`は、ReactによるUIの更新とブラウザのView Transition APIを連携させるコンポーネントです。"
> ([スケルトンスクリーンからコンテンツへの切り替わりをアニメーションする（ViewTransition & Suspense）](https://zenn.dev/dentyu/articles/animate-skeleton-with-view-transition), セクション "ViewTransition") ※2026-08-06に実際にfetch成功

> "<ViewTransition />: Adds <ViewTransition /> and addTransitionType APIs to power View Transition animations in React"
> ([CHANGELOG.md (React 19.3.0, 2026-09-09)](https://github.com/facebook/react/blob/main/CHANGELOG.md), セクション "19.3.0 (September 9, 2026)") ※2026-09-25に実際にfetch成功

**バージョン**: Same-Document は Chrome 111+, Safari 18+, Firefox 144+ / Cross-Document は Chrome 126+, Safari 18.2+；React `<ViewTransition>` は React 19.3+ で stable（2026-09-09リリース、canary限定ではない）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 5. `AbortController` で fetch リクエストをキャンセルする

コンポーネントのアンマウントやユーザーの操作によって不要になった fetch は
`AbortController` でキャンセルし、不要なネットワーク処理と、古いレスポンスが新しい状態を上書きする競合を防ぐ。

**根拠**:
- キャンセルしないと、`userId` が切り替わった後に古いリクエストの結果が届いて新しい表示を上書きする（競合状態）。アンマウント後の `setState` に対する警告は React 18 で削除されたが、不要な通信と処理は残る
- `AbortController` の signal は `fetch`・`axios`・`addEventListener` など signal を受け取る API で使える。`EventSource` は signal を受け取らないので `close()` で終了する
- React では `useEffect` のクリーンアップ関数でキャンセルするパターンが標準的
- 単純な時間経過での打ち切りだけなら `AbortSignal.timeout(ms)` が `AbortController` を手動生成せずにタイムアウト付き signal を返す。内部タイムアウトと呼び出し元からのキャンセルを同時に扱いたい場合は `AbortSignal.any([...])` で複数 signal を1つに合成できる
- 一度 abort した `AbortSignal` は abort 済み状態のまま再利用できない。同じ signal を使い回すと後続の `fetch` が即座に reject されるため、リクエストごとに signal を作り直す

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
    // userId が変わったら前回の結果とエラーを消してから取り直す
    setUser(null);
    setError(null);

    async function fetchUser() {
      try {
        const res = await fetch(`/api/users/${userId}`, { signal });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const data: User = await res.json();
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

// Bad: クリーンアップなし（競合状態の原因）
useEffect(() => {
  fetch(`/api/users/${userId}`)
    .then(r => r.json())
    .then(data => setUser(data));  // 古い userId の結果が後から届くと新しい表示を上書きする
}, [userId]);

// 複数リクエストを同時キャンセルする場合も同一 signal を渡すだけ。
// abort() は完了を待つ前に、cleanup や別のイベント（キャンセルボタン等）から呼ぶ
useEffect(() => {
  const controller = new AbortController();
  Promise.all([
    fetch('/api/a', { signal: controller.signal }),
    fetch('/api/b', { signal: controller.signal }),
  ]).catch((err) => {
    if ((err as Error).name !== 'AbortError') console.error(err);
  });
  return () => controller.abort();  // 両方を一度でキャンセル
}, []);

// Good: 単純なタイムアウトだけなら AbortController を手動生成しなくてよい
await fetch('/api/slow', { signal: AbortSignal.timeout(5_000) });

// Good: 内部タイムアウトと呼び出し元キャンセルを両方効かせたい場合
async function fetchWithTimeout(url: string, callerSignal?: AbortSignal) {
  const signal = callerSignal
    ? AbortSignal.any([AbortSignal.timeout(5_000), callerSignal])
    : AbortSignal.timeout(5_000);
  return fetch(url, { signal });
}
```

**出典**:
- [MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) (MDN Web Docs)
- [MDN: Fetch API - Canceling a request](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch#canceling_a_request) (MDN Web Docs)
- [MDN: AbortSignal.timeout()](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/timeout_static) (MDN Web Docs、ブラウザ対応版。MDN の browser-compat-data で Chrome 124 / Firefox 100 / Safari 16 を確認) ※2026-10-25 確認
- [MDN: EventSource.close()](https://developer.mozilla.org/en-US/docs/Web/API/EventSource/close) (MDN Web Docs)

**出典引用**:
> "Once aborted, an AbortSignal remains in aborted state, causing subsequent fetch calls using the same signal to immediately reject."
> ([Node.jsのfetchにタイムアウトを入れる：AbortSignal.timeoutとanyの実務的な使い分け](https://zenn.dev/sponge841841/articles/2026-09-21-node-fetch-timeout-abortsignal), セクション "Signal Reuseの落とし穴") ※2026-09-24に実際にfetch成功

**バージョン**: Chrome 66+, Firefox 57+, Safari 12.1+ / `AbortSignal.timeout` は Chrome 124+（103〜123 はタイムアウト時に `TimeoutError` ではなく `AbortError` になる）, Firefox 100+, Safari 16+ / `AbortSignal.any` は Chrome 116+, Firefox 124+, Safari 17.4+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 6. Baseline Widely Available 外の API 利用を TypeScript + CI で機械的に拒否する

`tsconfig` の `target` / `lib` はビルド出力の構文レベルを決めるだけで、実行時に使える Web API の可否は保証しない。
`noLib: true` の専用 `tsconfig` に Baseline Widely Available のみを含む型定義を読み込ませ、
対象外 API の使用を `tsc` のコンパイルエラーとして CI で機械的に検出する。

**根拠**:
- Vite/webpack の `target` はトランスパイル対象の構文（アロー関数・オプショナルチェイニング等）を下げるだけで、`Promise.try` や `document.startViewTransition` のような「構文的には新しくないが実装が新しい」API の呼び出しは素通りしてビルドが通ってしまう
- 通常の `tsconfig.json`（標準の `lib`）は Baseline に関わらず「その API が型定義に存在するかどうか」でしか判定できないため、ブラウザ互換性チェックとしては機能しない
- `typescript-baseline-lib` と `@baseline-types/dom-widely-available` を組み合わせ、`noLib: true` にした専用 `tsconfig.baseline.json` で型チェック専用に読み込ませることで、「型としては存在するが Baseline Widely Available 外」の API を型エラーとして検出できる
- CI で `tsc --noEmit -p tsconfig.baseline.json` を通すだけで PR 単位でのブラウザ互換性ゲートになり、レビュアーの目視確認に依存しない
- 限界: `typescript-baseline-lib` が絞り込むのは JavaScript の組み込み API だけで、DOM / Worker の API や構文は対象外。DOM 側の `@baseline-types/dom-widely-available` は型の参照関係を保つために Widely available の厳密な集合より広い（上位集合）ので、Widely available 外の API をすべて拒否できるわけではない。構文と Web API のチェックは `eslint-plugin-baseline-js` などと併用する

**コード例**:
```jsonc
// tsconfig.baseline.json
// 通常の tsconfig.json を extends して jsx / moduleResolution / paths などを引き継ぎ、lib と types だけを差し替える
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "lib": null,          // 継承した lib を消す（noLib と併用すると TS5053 になる）
    "noLib": true,
    "types": ["typescript-baseline-lib", "@baseline-types/dom-widely-available"],
    "noEmit": true,
    "skipLibCheck": false
  },
  "include": ["src/**/*.ts", "src/**/*.tsx"]
}
```

```yaml
# .github/workflows/baseline-check.yml
- name: Check Baseline Widely Available compatibility
  run: npx tsc --noEmit -p tsconfig.baseline.json
```

```ts
// Bad: Baseline Widely Available 外の API を無警告で使ってしまう
// （Promise.try は 2025-01 に Newly available になったばかりで、Widely available ではない）
function parseLater(text: string) {
  return Promise.try(() => JSON.parse(text)); // tsconfig.baseline.json では型エラーになる
}

// Good: Widely available の API だけで同じことを書く、または feature detection でフォールバック
function parseLater(text: string) {
  return new Promise((resolve) => resolve(JSON.parse(text)));
}
```

**出典**:
- [そのAPI、Baselineですか？ TypeScriptの型とCIでブラウザ互換性を守る](https://zenn.dev/ru/articles/17941500fa3391) (Zenn) ※2026-07-25に実際にfetch成功
- [Your Build Target Is Not an API Contract: Enforcing Baseline with TypeScript](https://dev.to/ryuya/your-build-target-is-not-an-api-contract-enforcing-baseline-with-typescript-epn) (dev.to、同一手法の別実装解説) ※2026-07-25に実際にfetch成功
- [typescript-baseline-lib](https://www.npmjs.com/package/typescript-baseline-lib) / [Usage Guide](https://github.com/3ru/TypeScript-Baseline-lib-generator/blob/main/docs/USAGE.md) (パッケージ公式。対応 TypeScript は `>=6 <8`、2026-10 のスナップショットで `Promise.try()` を拒否すること、`extends` した CI 用 tsconfig では `lib: null` が必要なこと、DOM は対象外であること) ※2026-10-25 確認
- [@baseline-types/dom-widely-available](https://www.npmjs.com/package/@baseline-types/dom-widely-available) (パッケージ公式。型の参照関係を保つため Widely available の上位集合になること) ※2026-10-25 確認

**出典引用**:
> "ViteとTypeScriptは、どちらも別の役割を正しく果たしています。足りないのは『このAPIはWidely Availableか』という検査です。"
> ([そのAPI、Baselineですか？](https://zenn.dev/ru/articles/17941500fa3391), セクション "2つの型パッケージを入れて、実際に止める") ※2026-07-25に実際にfetch成功

> "Its build target tells the transformer which JavaScript syntax needs to be lowered. It does not automatically reject every API that falls outside the corresponding Baseline target."
> ([Your Build Target Is Not an API Contract](https://dev.to/ryuya/your-build-target-is-not-an-api-contract-enforcing-baseline-with-typescript-epn), セクション "Why the build passes") ※2026-07-25に実際にfetch成功

**バージョン**: TypeScript 6+（`typescript-baseline-lib` 0.1.x の peerDependencies は `typescript >=6 <8`）, `@baseline-types/dom-widely-available`（日付版。2026-10 時点の最新は 1.0.20261019）
**確信度**: 中（2記事とも同一の2パッケージ・同一のtsconfigパターンを紹介しており、独立した手法検証というより同一ツールの再解説に近いため、パターン2の形式は満たすものの「高」への格上げは見送る）
**最終更新**: 2026-10-25

---
