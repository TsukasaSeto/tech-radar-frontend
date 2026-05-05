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
