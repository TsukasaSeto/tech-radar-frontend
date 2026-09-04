# コンポーネントテストのベストプラクティス

## ルール

### 1. Testing Library の `getByRole` を優先してクエリを選択する

要素の取得は `getByRole`、`getByLabelText`、`getByText` の順で優先する。
`getByTestId` は最後の手段とする。

**根拠**::
- `getByRole` はアクセシビリティツリーを使うため、アクセシブルなUIを自然と促進する
- ロールで取得するとリファクタリング（クラス名変更・マークアップ変更）に強い
- `data-testid` は実装詳細への依存であり、本番コードを汚染する

**コード例**:
```tsx
import { render, screen } from '@testing-library/react';

// Bad: data-testid や CSS クラスに依存
render(<LoginForm />);
const button = screen.getByTestId('submit-button');   // 実装詳細
const input = document.querySelector('.email-input'); // DOMに直接依存

// Good: ロールとラベルで取得
render(<LoginForm />);

// ロールで取得（最優先）
const submitButton = screen.getByRole('button', { name: 'ログイン' });

// ラベルで取得（フォーム要素）
const emailInput = screen.getByLabelText('メールアドレス');

// テキストで取得（見出しなど）
const heading = screen.getByRole('heading', { name: 'ログインフォーム' });

// aria-label が付いた要素
const closeButton = screen.getByRole('button', { name: '閉じる' });

// クエリの優先順位:
// getByRole > getByLabelText > getByPlaceholderText > getByText
// > getByDisplayValue > getByAltText > getByTitle > getByTestId
```

**出典**:
- [Testing Library: Which query should I use?](https://testing-library.com/docs/queries/about#priority) (Testing Library公式)

**バージョン**: @testing-library/react 14+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. ユーザーインタラクションには `userEvent` を使う

クリック・入力などのユーザー操作は `@testing-library/user-event` の `userEvent` を使う。
`fireEvent` は直接イベントをトリガーするが、実際のユーザー操作と異なる。

**根拠**:
- `userEvent.click()` はフォーカス・mousedown・mouseup・click を順番に発火し実際の動作を再現する
- `userEvent.type()` は1文字ずつ入力イベントを発火しバリデーションを正しくテストできる
- `fireEvent` はユニットテストレベルの低レベルな操作で、ユーザーの操作を完全には再現しない

**コード例**:
```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

describe('SearchInput', () => {
  it('should call onSearch when Enter key is pressed', async () => {
    const handleSearch = vi.fn();
    render(<SearchInput onSearch={handleSearch} />);

    const input = screen.getByRole('textbox', { name: '検索' });

    // Bad: fireEvent は Enter キーのみを発火（不完全）
    // fireEvent.keyDown(input, { key: 'Enter' });

    // Good: userEvent はテキスト入力の全イベントを正確に再現
    const user = userEvent.setup();
    await user.type(input, '検索キーワード{Enter}');

    expect(handleSearch).toHaveBeenCalledWith('検索キーワード');
  });

  it('should clear input when × button is clicked', async () => {
    const user = userEvent.setup();
    render(<SearchInput onSearch={vi.fn()} />);

    const input = screen.getByRole('textbox', { name: '検索' });
    await user.type(input, 'some text');

    const clearButton = screen.getByRole('button', { name: 'クリア' });
    await user.click(clearButton);

    expect(input).toHaveValue('');
  });
});
```

**出典**:
- [Testing Library: user-event](https://testing-library.com/docs/user-event/intro) (Testing Library公式)

**バージョン**: @testing-library/user-event 14+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. 非同期処理は `waitFor` / `findBy*` で正しく待機する

非同期でUIが更新される場合は `await waitFor()` や `findBy*` クエリを使う。
`getBy*` で取得する前に非同期処理の完了を待たないと不安定なテストになる。

**根拠**:
- 非同期処理完了前にアサートすると「まだ表示されていない要素」を探してテストが落ちる
- `findBy*` は内部で `waitFor` を使い要素が現れるまで待機する
- `waitFor` で複数のアサーションをまとめると再試行が最小限になる
- `act()` は React の state 更新・副作用が完了するまで待機するヘルパーで、`waitFor` とは目的が異なる：`act()` は同期的な React 更新を確定させる用途、`waitFor` は非同期でアサーションが通るまでリトライする用途
- `fetch` リクエストやタイマー等の非 React API は `act()` だけでは待機できない。これらは mock と組み合わせて使う

**コード例**:
```tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { server } from '@/mocks/server';  // MSW
import { rest } from 'msw';

describe('UserList', () => {
  it('should display users after loading', async () => {
    render(<UserList />);

    // ローディング中の表示を確認
    expect(screen.getByRole('status')).toBeInTheDocument();

    // findBy* は要素が現れるまで待機（デフォルト1000ms）
    const userItems = await screen.findAllByRole('listitem');
    expect(userItems).toHaveLength(3);

    // ローディングが消えたことを確認
    expect(screen.queryByRole('status')).not.toBeInTheDocument();
  });

  it('should show error message on fetch failure', async () => {
    server.use(
      rest.get('/api/users', (req, res, ctx) => {
        return res(ctx.status(500));
      })
    );

    render(<UserList />);

    // waitFor でアサーションが通るまで再試行
    await waitFor(() => {
      expect(screen.getByRole('alert')).toHaveTextContent('データの取得に失敗しました');
    });
  });
});
```

**コード例（act vs waitFor）**:
```tsx
// act(): React の state 更新を確定させる（カスタムフックのテストで多用）
it('should increment counter', async () => {
  const { result } = renderHook(() => useCounter());
  await act(async () => {
    result.current.increment();
  });
  expect(result.current.count).toBe(1);
});

// waitFor(): 非同期処理後にアサーションが通るまでリトライ
it('should display fetched message', async () => {
  render(<MessageDisplay />);
  await waitFor(() => {
    expect(screen.getByText('Hello World!')).toBeInTheDocument();
  }, { timeout: 3000 });
});

// NG: fetch などの非 React API は act() だけでは待機できない
// → MSW や vi.mock でネットワーク層をモックした上で waitFor() と組み合わせる
```

**出典**:
- [Testing Library: Async Utilities](https://testing-library.com/docs/dom-testing-library/api-async) (Testing Library公式)
- [act vs waitFor](https://dev.to/hmcodes/act-vs-waitfor-5713) (dev.to、`act()` と `waitFor()` の目的の違いと使い分けルール) ※2026-05-25に実際にfetch成功

**バージョン**: @testing-library/react 14+
**確信度**: 高
**最終更新**: 2026-05-25

---

### 4. MSW（Mock Service Worker）でAPIモックを行う

APIリクエストのモックは `msw` を使い、ネットワークレベルでインターセプトする。
`fetch` や `axios` のモジュールモックは避ける。

**根拠**:
- MSW はネットワーク層でモックするため、fetch・axios・ky などのHTTPクライアントを問わない
- テストとブラウザ（開発環境）で同じモックハンドラを再利用できる
- モジュールのモックより実際のHTTPリクエストに近い動作をテストできる

**コード例**:
```ts
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/users', () => {
    return HttpResponse.json([
      { id: '1', name: 'Alice' },
      { id: '2', name: 'Bob' },
    ]);
  }),

  http.post('/api/users', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: '3', ...body }, { status: 201 });
  }),
];

// src/mocks/server.ts（テスト用）
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);

// vitest.setup.ts
import { server } from '@/mocks/server';

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

// テスト内での上書き
import { http, HttpResponse } from 'msw';

it('should handle 404', async () => {
  server.use(
    http.get('/api/users/:id', () => {
      return new HttpResponse(null, { status: 404 });
    })
  );
  // ...
});
```

**出典**:
- [MSW Docs: Getting Started](https://mswjs.io/docs/getting-started) (MSW公式)

**バージョン**: msw 2+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 5. Accessibility Query の優先順位を守り、アクセシビリティを担保する

Testing Library のクエリ選択はアクセシビリティツリーを優先し、
`getByRole` → `getByLabelText` → `getByPlaceholderText` → `getByText` の順序を厳守する。
このルールに従うことで、テストがアクセシビリティ検証を兼ねる「二重の安全網」になる。

**根拠**:
- `getByRole` が通過することはスクリーンリーダーが認識できることを意味し、a11y を自動的に保証する
- ロールと名前（`name` オプション）の組み合わせは ARIA 仕様の正確な反映であり、HTML 構造の変更に対して安定している
- `getByTestId` に頼るコンポーネントは `aria-label` や `role` が欠落している可能性があり、アクセシビリティ負債を生む

**コード例**:
```tsx
import { render, screen } from '@testing-library/react';

// コンポーネント例
function ProductCard({ product }: { product: Product }) {
  return (
    <article aria-label={`商品: ${product.name}`}>
      <h2>{product.name}</h2>
      <img src={product.imageUrl} alt={product.name} />
      <p>{product.description}</p>
      <button aria-label={`${product.name}をカートに追加`}>
        カートに追加
      </button>
    </article>
  );
}

// Good: Accessibility Query を優先
it('should render product card with accessible elements', () => {
  render(<ProductCard product={mockProduct} />);

  // article ロールで取得（aria-label で特定）
  expect(screen.getByRole('article', { name: '商品: テスト商品' })).toBeInTheDocument();

  // 見出しロールで取得
  expect(screen.getByRole('heading', { level: 2, name: 'テスト商品' })).toBeInTheDocument();

  // img の alt テキストで取得
  expect(screen.getByRole('img', { name: 'テスト商品' })).toBeInTheDocument();

  // ボタンは aria-label で特定
  expect(screen.getByRole('button', { name: 'テスト商品をカートに追加' })).toBeEnabled();
});

// Bad: testid や class に頼る
it('bad example', () => {
  render(<ProductCard product={mockProduct} />);
  expect(screen.getByTestId('product-card')).toBeInTheDocument(); // a11y 担保なし
  expect(document.querySelector('.add-to-cart-btn')).toBeInTheDocument(); // 実装詳細
});
```

**出典**:
- [Testing Library: About Queries - Priority](https://testing-library.com/docs/queries/about#priority) (Testing Library公式)
- [WAI-ARIA Roles](https://www.w3.org/TR/wai-aria/#role_definitions) (W3C / WAI-ARIA仕様)

**バージョン**: @testing-library/react 14+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 6. async Server Component のテストツール選択フロー（Vitest / Storybook / E2E の振り分け）

Next.js App Router (v16 時点) では、`async function Page()` のように **データフェッチを内包する Server Component は Vitest で直接テストできない**（RSC レンダラーが必要なため）。テスト対象が「同期コンポーネント」か「async Server Component」かを最初に判別し、適切なツールへ振り分ける。Container/Presentational パターン（Container を薄く、Presentational を厚く）で分離しておくと、Presentational を Vitest + RTL でカバーでき、Container は E2E に任せる割り切りが効きやすい。

**根拠**:
- Vitest（jsdom / happy-dom）は React Server Components ランタイムを持たないため、`async` を含む Server Component を `render()` すると Promise が露出して assert できない
- 一方、**同期** Server Component や Client Component は通常の関数コンポーネントとして RTL でテスト可能。境界は「Server か Client か」ではなく「`async` か否か」
- async Server Component を `renderToStaticMarkup` などで無理やり同期化する迂回はアンチパターン（Suspense / streaming 挙動が再現できず、本番と乖離する）
- ロジックは DAL に追い出し、Server Component 自体は薄く保つことで Vitest でカバーできる範囲を最大化する設計指針が成立する

**コード例**:
```text
判断フロー:

  対象コンポーネントは async？
   ├─ No (同期 Server Component / Client Component)
   │    └─ Vitest + React Testing Library で単体テスト
   │       例: Presentational コンポーネント、Client Component の state / event
   │
   └─ Yes (async Server Component / Container)
        ├─ データ層をモジュールモックで差し替えて UI を確認したい
        │    └─ Storybook (experimentalRSC) + sb.mock() で server function をモック
        │       .storybook/main.ts で features.experimentalRSC: true
        │
        ├─ ページ全体の挙動 (RSC の出力が DOM に反映された後) を確認したい
        │    └─ E2E (Playwright)
        │
        └─ Container を薄く / Presentational に分離できる
             └─ Presentational だけ Vitest + RTL で厚くテスト
                Container 自体は E2E で end-to-end カバー (割り切り)
```

| 対象 | 推奨ツール | 補足 |
|---|---|---|
| 同期 Server Component / Presentational | Vitest + RTL | 通常の関数コンポーネントと同じ扱い |
| Client Component | Vitest + RTL + `userEvent` | state / event / form を直接検証 |
| async Server Component（UI 確認重視） | Storybook (experimentalRSC) | `sb.mock()` で server function をモック |
| async Server Component（end-to-end 確認） | Playwright (E2E) | RSC 出力がブラウザに反映された後を見る |
| Container (薄いオーケストレーション) | E2E に委譲 | Presentational を Vitest で厚くカバーした上で |

```ts
// Good: Presentational を Vitest + RTL でテスト（同期）
// _containers/post/presentational.tsx は普通の純粋コンポーネント
import { render, screen } from '@testing-library/react';
import { PostPresentational } from './presentational';

test('title と本文を表示する', () => {
  render(<PostPresentational post={{ title: 'Hello', body: '...' }} />);
  expect(screen.getByRole('heading', { name: 'Hello' })).toBeInTheDocument();
});

// Bad: async Server Component を Vitest で無理やり render
import { render } from '@testing-library/react';
import Page from './page'; // async function Page()

test('does not work', async () => {
  render(<Page />); // Promise が露出して assert できない / RSC ランタイム不在
});
```

**出典**:
- [Next.jsの考え方 / RSC のテスト戦略（async Server Component）](https://zenn.dev/akfm/books/nextjs-basic-principle)

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**バージョン**: Next.js 16+
**確信度**: 高（v16 公式相当の知見）
**最終更新**: 2026-05-16

---

### 7. Playwright 1.62 のコンポーネントテストは「stories & galleries」モデルに移行しており、`mount()` の返り値が変わったことを踏まえて構成する

Playwright のコンポーネントテスト（Component Testing）は 1.62 で「stories & galleries」モデルに再設計された。旧方式では Playwright 自身がバンドラを持ち JSX を直接マウントしていたが、新方式では JSX がアプリ側の `*.story.tsx` に移り、Playwright は「バンドラを持つ側」をやめて、アプリ自身の dev server が配信する1枚の HTML（gallery）を叩くだけになる。`playwright.config.ts` の `webServer` / `baseURL` を gallery の URL に向け、gallery 側で `window.mount()` / `window.unmount()` の契約を実装する必要がある。1.62.0 にはリグレッションが報告されているため、`1.62.1` 以降を使う。
運用面では、**story を「コンポーネントの 1 状態（props・モック・Provider・コールバックを固定したもの）」として先に定義し、テストは story id を `mount()` に渡すだけにする**。テストごとに JSX を書かなくなり、同じ状態定義を複数テストから再利用できる。

**根拠**:
- 新方式では Playwright がバンドラを持たなくなり、アプリ自身の dev server（Vite 等）が配信する gallery HTML を経由してコンポーネントをマウントするため、`playwright.config.ts` の `webServer` / `baseURL` を gallery URL に向ける設定が必須になる
- `fixtures.mount()` は gallery に遷移して story id でマウントし、story のルート要素にスコープした `Locator` を返す設計になっている
- 1.62.0 自体にリグレッションが報告されているため、導入時は `1.62.1` 以降を明示的に指定する
- story は「コンポーネントの 1 つの状態」を表す単位であり、ハードコードした props・モックデータ・Provider・必要なコールバックを 1 か所に束ねる。テストは story を指定して `mount()` するだけになるため、テストごとに JSX を書き直す必要がなくなり、同じ状態を複数テストで再利用できる
- Storybook が「人が目視で確認する」用途なのに対し、Playwright の CT は「実ブラウザで自動検証する」用途。同じ story 概念でも役割が異なるため、どちらかで代替しようとしない

**コード例**:
```typescript
// playwright.config.ts
const GALLERY_URL = 'http://localhost:5173/playwright/gallery/index.html';

export default defineConfig({
  // ...
  use: { baseURL: GALLERY_URL },
  webServer: {
    command: 'npm run dev',
    url: GALLERY_URL,
    reuseExistingServer: !process.env.CI,
  },
});
```

```typescript
// gallery 側（index.html から読み込まれるスクリプト）が実装する mount/unmount 契約
window.mount = async ({ story, props }) => {
  const Story = await resolveStory(story);
  if (!Story) throw new Error(`Unknown story: ${story}`);
  root ??= createRoot(rootEl);
  flushSync(() => {
    root!.render(<StrictMode><Story {...props} /></StrictMode>);
  });
};

window.unmount = async () => {
  root?.unmount();
  root = undefined;
};
```

**アンチパターン**:
- 旧方式（Playwright 自身がバンドラを持ちJSXを直接マウントする）のドキュメント・記事を参照して `playwright.config.ts` を構成し、`webServer` / `baseURL` を gallery URL に向け忘れる
- リグレッションが報告されている `1.62.0` をそのまま使う

**出典引用**:
> 「Component testing が stories and galleries モデルに移った。`fixtures.mount()` は gallery に遷移して story id で mount し、story の root 要素にスコープした `Locator` を返す」
> ([Playwright 1.62 の stories & galleries でコンポーネントテストを最小構成から作ってみた](https://zenn.dev/clopy/articles/playwright162-ct-stories-galleries), セクション "何が変わったのか（1.62.0 のリリースノート）") ※2026-08-23に実際にfetch成功

> "`mount()` は、その story を実ブラウザ上に載せる役割です。" / "テストごとに JSX を毎回書かなくてよい"
> ([Playwright の component testing と `mount()` を整理する](https://zenn.dev/pug/articles/playwright-component-testing-story-mount), セクション "mount とは" / "何が変わるか") ※2026-08-28に実際にfetch成功

**story を再利用するテストの書き方**:
```typescript
// story 側: コンポーネントの 1 状態を props ごと固定する
export const Disabled = {
  args: { label: 'Submit', disabled: true },
};

// テスト側: story id を渡すだけ。JSX をテストに書かない
test('disabled button is disabled', async ({ mount }) => {
  const component = await mount('components/Button/Disabled');
  await expect(component.getByRole('button')).toBeDisabled();
});

// 同じ story を状態遷移の検証にも再利用する
test('button becomes enabled after input', async ({ mount }) => {
  const component = await mount('components/Form/Primary');
  await component.getByLabel('Email').fill('user@example.com');
  await expect(component.getByRole('button', { name: 'Submit' })).toBeEnabled();
});
```

**出典**:
- [Playwright 1.62 の stories & galleries でコンポーネントテストを最小構成から作ってみた](https://zenn.dev/clopy/articles/playwright162-ct-stories-galleries) (Zenn clopy、`webServer` / `baseURL` と gallery 契約の最小構成) ※2026-08-23 fetch
- [Playwright の component testing と `mount()` を整理する](https://zenn.dev/pug/articles/playwright-component-testing-story-mount) (Zenn pug、story を状態定義として再利用する書き方・Storybook との役割分担という追加観点) ※2026-08-28 fetch

**バージョン**: Playwright 1.62.1+
**確信度**: 高（異なる著者による2記事＋コード例でパターン2を満たすため 2026-08-28 に「中」から格上げ）
**最終更新**: 2026-08-28
