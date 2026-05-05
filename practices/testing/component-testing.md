# コンポーネントテストのベストプラクティス

## ルール

### 1. Testing Library の `getByRole` を優先してクエリを選択する

要素の取得は `getByRole`、`getByLabelText`、`getByText` の順で優先する。
`getByTestId` は最後の手段とする。

**根拠**:
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

**出典**:
- [Testing Library: Async Utilities](https://testing-library.com/docs/dom-testing-library/api-async) (Testing Library公式)

**バージョン**: @testing-library/react 14+
**確信度**: 高
**最終更新**: 2026-05-05

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
