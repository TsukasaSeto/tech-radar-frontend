# テスト戦略のベストプラクティス

## ルール

### 1. テストピラミッドを意識してテストを配置する

ユニットテストを最も多く、E2E テストを最も少なく配置する。
コストと信頼性のバランスをテストレイヤーで最適化する。

**根拠**:
- ユニットテストは高速・低コスト・安定しているため大量に書ける
- E2E テストは遅く・高コスト・不安定なため重要フローのみに絞る
- テストピラミッドの逆転（アイスクリームコーン型）はCI時間の肥大化を招く

**コード例**:
```
テストピラミッド:

       /\
      /E2E\     少: 重要なユーザージャーニーのみ (5〜10%)
     /------\
    /Integ.  \  中: APIとの統合、コンポーネント統合 (20〜30%)
   /----------\
  / Unit Tests \  多: 関数・フック・ユーティリティ (60〜70%)
 /--------------\

アンチパターン（アイスクリームコーン型）:
       /---\
      / E2E \    多: E2Eばかり (遅い・不安定・高コスト)
     /-------\
    /Integ.   \  少
   /-----------\
  / Unit Tests  \ 少: ユニットテストが少ない
```

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    // ユニット・コンポーネントテスト
    include: ['src/**/*.test.{ts,tsx}'],
    environment: 'jsdom',
    globals: true,
    coverage: {
      provider: 'v8',
      thresholds: {
        lines: 80,   // カバレッジ目標: 行
        functions: 80,
        branches: 70,
      },
    },
  },
});
```

**出典**:
- [The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html) (Martin Fowler)
- [Write tests. Not too many. Mostly integration.](https://kentcdodds.com/blog/write-tests) (Kent C. Dodds)

**バージョン**: Vitest 1+, Playwright 1.40+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. カバレッジは指標として使い、100% を目標にしない

コードカバレッジは品質の指標の1つであり、最終目標ではない。
重要なビジネスロジックに対して高カバレッジを維持し、自動生成コードや設定ファイルは除外する。

**根拠**:
- 100% カバレッジを目標にすると意味のない（アサーションのない）テストが増える
- カバレッジが高くても、重要なエッジケースがテストされていない可能性がある
- 価値のある 80% カバレッジのほうが、意味のない 100% より安全なコードを生む

**コード例**:
```ts
// vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      // 自動生成コードや設定ファイルを除外
      exclude: [
        'src/**/*.config.{ts,js}',
        'src/**/*.stories.{ts,tsx}',
        'src/mocks/**',
        'src/types/**',  // 型定義のみのファイル
      ],
      thresholds: {
        // カテゴリ別のしきい値設定
        lines: 80,
        functions: 80,
        branches: 70,
        statements: 80,
      },
    },
  },
});

// Bad: カバレッジのためだけのアサーションなしテスト
it('should call formatDate', () => {
  formatDate(new Date());  // アサーションなし = 無意味なカバレッジ
});

// Good: 意味のあるアサーション
it('should format date as YYYY-MM-DD', () => {
  expect(formatDate(new Date('2026-05-05'))).toBe('2026-05-05');
});
```

**出典**:
- [Measuring Code Coverage](https://martinfowler.com/bliki/TestCoverage.html) (Martin Fowler)

**バージョン**: Vitest 1+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. Vitest を Next.js プロジェクトのデフォルトテストランナーとして使う

Next.js プロジェクトでは Jest の代わりに Vitest を使う。
ESM ネイティブで高速なウォッチモード、TypeScript のネイティブサポートが特長。

**根拠**:
- Vite のトランスフォームパイプラインを共有するため TypeScript・JSX の設定が不要
- Jest より大幅に高速（コールドスタートで2〜10倍）
- Jest と互換性のある API（`describe`, `it`, `expect`, `vi`）で移行コストが低い
- ブラウザモードで実際のブラウザ環境でのテストも可能（Vitest Browser Mode）

**コード例**:
```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [react(), tsconfigPaths()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
  },
});

// src/test/setup.ts
import '@testing-library/jest-dom';
import { server } from '@/mocks/server';

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

// package.json scripts
// "test": "vitest",
// "test:ui": "vitest --ui",
// "test:coverage": "vitest run --coverage"
```

**出典**:
- [Vitest Docs](https://vitest.dev/guide/) (Vitest公式)
- [Next.js Docs: Setting up Vitest](https://nextjs.org/docs/app/building-your-application/testing/vitest) (Next.js公式)

**バージョン**: Vitest 1+, Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. スナップショットテストは慎重に使う

スナップショットテストはUIの意図しない変更を検出するには有効だが、
大量に使うとメンテコストが高くなる。インラインスナップショットを優先する。

**根拠**:
- スナップショットファイルは大きくなりがちで diff が読みにくい
- スナップショットを安易に更新（`--updateSnapshot`）するとバグを見逃す
- インラインスナップショットはテストコード内に収まり変更の意図が明確になる

**コード例**:
```tsx
// Bad: 大きなコンポーネントのスナップショット（変更の意図が不明確）
it('should match snapshot', () => {
  const { asFragment } = render(<LargeComplexComponent data={mockData} />);
  expect(asFragment()).toMatchSnapshot();  // 何百行もの snapshot ファイルが生成される
});

// Good: インラインスナップショット（重要な部分のみ）
it('should render user greeting', () => {
  render(<Greeting name="Alice" />);
  expect(screen.getByRole('heading')).toMatchInlineSnapshot(`
    <h1>
      こんにちは、Alice さん！
    </h1>
  `);
});

// Better: 具体的なアサーション（スナップショット不要）
it('should render user greeting', () => {
  render(<Greeting name="Alice" />);
  expect(screen.getByRole('heading')).toHaveTextContent('こんにちは、Alice さん！');
});
```

**出典**:
- [Vitest Docs: Snapshot Testing](https://vitest.dev/guide/snapshot) (Vitest公式)

**バージョン**: Vitest 1+, Jest 27+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 5. テストピラミッドとテストダイヤモンドをフロントエンド文脈で使い分ける

従来のテストピラミッド（ユニット > 統合 > E2E）に加え、フロントエンドではコンポーネント統合テストを厚くした
「テストダイヤモンド」モデルが有効な場合がある。プロジェクトの特性に応じて戦略を選択する。

**根拠**:
- フロントエンドのビジネスロジックは UIコンポーネントに分散しているため、コンポーネント統合テストが最もROIが高い
- Testing Library + MSW の組み合わせにより、ブラウザなしで統合テストを高速実行できるようになった
- 純粋な関数が少なく、副作用の多いUIコードはユニットテストの比率を下げ、統合レベルを厚くする方が現実的

**コード例**:
```
テストピラミッド（バックエンド寄りの伝統的モデル）:
       /\
      /E2E\
     /------\
    /  Integ. \
   /------------\
  /  Unit Tests  \
 /-----------------\
 比率: Unit 70% / Integ 20% / E2E 10%

テストダイヤモンド（フロントエンド推奨モデル）:
     /------\
    / Comp.   \   ← コンポーネント統合テストを最も厚く
   /  Integ.   \
  /-------------\
 /  Unit Tests   \  ← 純粋関数・hooks のユニットテスト
/-----------------\
      /E2E\        ← 最重要フローのみ
比率: Unit 30% / Component Integ 50% / E2E 20%

判断基準:
- SPAでビジネスロジックがUIに密結合 → ダイヤモンド型
- ライブラリ・ユーティリティ中心     → ピラミッド型
- BFF/フルスタック構成               → ハイブリッド
```

```ts
// コンポーネント統合テストの例（MSW + Testing Library）
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { CheckoutFlow } from '@/components/CheckoutFlow';
// MSWのハンドラが /api/cart, /api/order などをモック

it('should complete checkout flow end-to-end', async () => {
  const user = userEvent.setup();
  render(<CheckoutFlow />);

  // 住所入力
  await user.type(screen.getByLabelText('郵便番号'), '150-0001');
  await screen.findByDisplayValue('東京都渋谷区'); // 住所自動補完

  // 支払い
  await user.click(screen.getByRole('radio', { name: 'クレジットカード' }));
  await user.click(screen.getByRole('button', { name: '注文を確定する' }));

  await expect(screen.findByText('ご注文ありがとうございます')).resolves.toBeInTheDocument();
});
```

**出典**:
- [The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html) (Martin Fowler)
- [Static vs Unit vs Integration vs E2E Testing](https://kentcdodds.com/blog/unit-vs-integration-vs-e2e-tests) (Kent C. Dodds)

**バージョン**: Vitest 2+, @testing-library/react 14+
**確信度**: 中
**最終更新**: 2026-05-06

---

### 6. MSW でAPIモッキング戦略を統一し、テスト・開発・Storybook で共有する

MSW（Mock Service Worker）のハンドラをテスト・開発サーバー・Storybook で共有する単一モック戦略を採用する。
環境ごとにモックを重複定義するアンチパターンを排除し、モックの一貫性を保つ。

**根拠**:
- テスト・開発・Storybook でモックを共有することで、「テストは通るがブラウザで壊れる」問題を防ぐ
- MSW 2.x の `http` ハンドラはリクエスト/レスポンスの型安全なインターセプトをサポートする
- `server.use()` によるテストごとのハンドラ上書きで、エラーケース・ローディング状態を細かくテストできる

**コード例**:
```ts
// src/mocks/handlers/users.ts - 共通ハンドラ定義
import { http, HttpResponse, delay } from 'msw';

export const userHandlers = [
  // 正常系
  http.get('/api/users', async () => {
    await delay(100); // 現実的な遅延をシミュレート
    return HttpResponse.json([
      { id: '1', name: 'Alice', role: 'admin' },
      { id: '2', name: 'Bob', role: 'member' },
    ]);
  }),

  http.get('/api/users/:id', ({ params }) => {
    if (params.id === '999') {
      return new HttpResponse(null, { status: 404 });
    }
    return HttpResponse.json({ id: params.id, name: 'Alice' });
  }),
];

// src/mocks/browser.ts - ブラウザ用（開発サーバー・Storybook）
import { setupWorker } from 'msw/browser';
import { userHandlers } from './handlers/users';
export const worker = setupWorker(...userHandlers);

// src/mocks/server.ts - Node.js 用（Vitest）
import { setupServer } from 'msw/node';
import { userHandlers } from './handlers/users';
export const server = setupServer(...userHandlers);

// テストでのエラーケース上書き
it('should show error state when API fails', async () => {
  server.use(
    http.get('/api/users', () => {
      return HttpResponse.json(
        { message: 'Internal Server Error' },
        { status: 500 }
      );
    })
  );

  render(<UserList />);
  await expect(screen.findByRole('alert')).resolves.toHaveTextContent('エラーが発生しました');
});

// Storybook での利用（.storybook/preview.ts）
// import { initialize, mswLoader } from 'msw-storybook-addon';
// initialize();
// export const loaders = [mswLoader];
```

**出典**:
- [MSW Docs: Integrations](https://mswjs.io/docs/integrations) (MSW公式)
- [MSW Docs: Network behavior](https://mswjs.io/docs/network-behavior) (MSW公式)
- [Storybook: Mock Service Worker Addon](https://storybook.js.org/addons/msw-storybook-addon) (Storybook公式)

**バージョン**: msw 2+, Vitest 2+
**確信度**: 高
**最終更新**: 2026-05-06
