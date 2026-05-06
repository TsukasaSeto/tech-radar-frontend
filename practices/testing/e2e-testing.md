# E2E テストのベストプラクティス

## ルール

### 1. E2E テストはユーザーの重要なジャーニーに絞る

E2E テストはコスト（実行時間・メンテコスト）が高いため、
ビジネスクリティカルなユーザーフローのみをカバーする。

**根拠**:
- E2E テストはユニット・コンポーネントテストより10〜100倍遅い
- すべての機能をE2EでカバーしようとするとCI時間が数時間になる
- テストピラミッドの最上位として、ユニット/コンポーネントテストでカバーできないフローのみ担う

**コード例**:
```ts
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  timeout: 30_000,
  retries: process.env.CI ? 2 : 0,  // CI では最大2回リトライ
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',  // 失敗時のみトレース保存
  },
});

// e2e/auth.spec.ts - 重要なジャーニー: 認証フロー
import { test, expect } from '@playwright/test';

test.describe('認証フロー', () => {
  test('ユーザーがログインしてダッシュボードを見られる', async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('メールアドレス').fill('user@example.com');
    await page.getByLabel('パスワード').fill('password123');
    await page.getByRole('button', { name: 'ログイン' }).click();

    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByRole('heading', { name: 'ダッシュボード' })).toBeVisible();
  });

  test('無効な認証情報でエラーメッセージが表示される', async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('メールアドレス').fill('wrong@example.com');
    await page.getByLabel('パスワード').fill('wrongpassword');
    await page.getByRole('button', { name: 'ログイン' }).click();

    await expect(page.getByRole('alert')).toContainText('メールアドレスまたはパスワードが間違っています');
  });
});
```

**出典**:
- [Playwright Docs: Getting Started](https://playwright.dev/docs/intro) (Playwright公式)
- [Google Testing Blog: Just Say No to More End-to-End Tests](https://testing.googleblog.com/2015/04/just-say-no-to-more-end-to-end-tests.html)

**バージョン**: Playwright 1.40+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. Page Object Model でテストコードを保守しやすくする

E2E テストの操作ロジックを Page Object クラスに抽出し、テストコードとUI実装を分離する。

**根拠**:
- UIの変更（ボタンのテキスト変更など）があっても Page Object だけ修正すればよい
- テストコードが「何をするか」に集中し、「どうやるか」が隠蔽される
- Page Object の再利用で複数テスト間の重複を削減できる

**コード例**:
```ts
// e2e/pages/login-page.ts
import { Page, Locator, expect } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel('メールアドレス');
    this.passwordInput = page.getByLabel('パスワード');
    this.submitButton = page.getByRole('button', { name: 'ログイン' });
    this.errorMessage = page.getByRole('alert');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}

// e2e/auth.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from './pages/login-page';

test('正しい認証情報でログインできる', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login('user@example.com', 'password123');

  await expect(page).toHaveURL('/dashboard');
});
```

**出典**:
- [Playwright Docs: Page Object Models](https://playwright.dev/docs/pom) (Playwright公式)

**バージョン**: Playwright 1.40+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. テストデータは各テストで独立させ、`beforeEach` でリセットする

テスト間でデータが共有・依存しないようにする。
各テストは独立して実行でき、順序に依存しないようにする。

**根拠**:
- テストの実行順序に依存すると並列実行ができなくなる
- 前のテストが失敗したとき、後続テストが誤った状態でスタートして偽陰性になる
- テストの独立性は再実行可能性（Idempotency）を保証する

**コード例**:
```ts
// e2e/checkout.spec.ts
import { test, expect } from '@playwright/test';

// テスト毎に独立したユーザーを使う（並列実行可能）
test.use({ storageState: 'playwright/.auth/user.json' });

test.beforeEach(async ({ request }) => {
  // テスト前にカートをリセット
  await request.delete('/api/cart');
});

test('商品をカートに追加して購入できる', async ({ page }) => {
  await page.goto('/products/1');
  await page.getByRole('button', { name: 'カートに追加' }).click();

  await page.goto('/cart');
  await expect(page.getByRole('listitem')).toHaveCount(1);

  await page.getByRole('button', { name: '購入する' }).click();
  await expect(page).toHaveURL('/order/confirmation');
});

// playwright.config.ts での並列実行設定
// workers: process.env.CI ? 1 : 4,  // CI では直列、ローカルは並列
```

**出典**:
- [Playwright Docs: Test Isolation](https://playwright.dev/docs/test-isolation) (Playwright公式)

**バージョン**: Playwright 1.40+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. Playwright の Fixture でテストの前提条件を宣言的に共有する

認証状態・DBシード・共通コンテキストなどの前提条件は `test.extend()` で Fixture として定義し、
テストファイルをまたいで再利用する。`beforeEach` のコピーペーストを排除できる。

**根拠**:
- Fixture は依存関係を宣言的に記述でき、必要なテストだけがセットアップコストを払う
- `scope: 'worker'` を使うと同じワーカー内で Fixture を1度だけ初期化でき、認証などの重い処理を最小化できる
- Page Object と組み合わせることでテストコードを最大限に簡潔にできる

**コード例**:
```ts
// e2e/fixtures.ts
import { test as base, expect } from '@playwright/test';
import { LoginPage } from './pages/login-page';
import { DashboardPage } from './pages/dashboard-page';

// Fixture の型定義
type TestFixtures = {
  loginPage: LoginPage;
  dashboardPage: DashboardPage;
  authenticatedPage: void; // 認証済み状態を保証するフィクスチャ
};

export const test = base.extend<TestFixtures>({
  // loginPage フィクスチャ: Page Object を自動的に用意
  loginPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page);
    await use(loginPage);
  },

  // authenticatedPage フィクスチャ: ログイン済み状態を保証
  authenticatedPage: async ({ page }, use) => {
    // ストレージから認証状態を復元（global setup で保存済み）
    await page.goto('/dashboard');
    await expect(page).toHaveURL('/dashboard'); // 認証済みであることを確認
    await use(); // テスト実行
  },

  dashboardPage: async ({ page, authenticatedPage }, use) => {
    // authenticatedPage フィクスチャに依存（自動的に認証処理が走る）
    const dashboardPage = new DashboardPage(page);
    await use(dashboardPage);
  },
});

export { expect };

// e2e/dashboard.spec.ts - フィクスチャを使ったテスト
import { test, expect } from './fixtures';

test('ダッシュボードに統計データが表示される', async ({ dashboardPage }) => {
  // authenticatedPage フィクスチャが自動的に認証処理を済ませてくれる
  await expect(dashboardPage.statsSection).toBeVisible();
  await expect(dashboardPage.totalSalesCard).toContainText('売上');
});

test('ログインページのテスト（認証不要）', async ({ loginPage }) => {
  await loginPage.goto();
  await expect(loginPage.submitButton).toBeEnabled();
});
```

**出典**:
- [Playwright Docs: Fixtures](https://playwright.dev/docs/test-fixtures) (Playwright公式)
- [Playwright Docs: Authentication](https://playwright.dev/docs/auth) (Playwright公式)

**バージョン**: Playwright 1.40+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 5. `toHaveScreenshot()` で視覚的回帰テストを自動化する

Playwright の `expect(page).toHaveScreenshot()` でスクリーンショットを撮影・比較し、
UIの意図しない見た目の変化を自動的に検出する。
ピクセル単位の差分検出でリグレッションを防ぐ。

**根拠**:
- CSSの変更やサードパーティライブラリのアップデートによる意図しないUI崩れを自動検出できる
- スクリーンショットは CI のアーティファクトとして保存でき、差分を視覚的にレビューできる
- `maxDiffPixelRatio` で許容誤差を設定でき、アンチエイリアスや描画差異によるフレーキーを軽減できる

**コード例**:
```ts
// e2e/visual.spec.ts
import { test, expect } from '@playwright/test';

test.describe('視覚的回帰テスト', () => {
  test('トップページが期待通りに表示される', async ({ page }) => {
    await page.goto('/');

    // アニメーションが完了するまで待機してから撮影
    await page.waitForLoadState('networkidle');

    // ページ全体のスクリーンショットと比較
    await expect(page).toHaveScreenshot('top-page.png', {
      maxDiffPixelRatio: 0.02, // 2%以内の差異は許容
    });
  });

  test('商品カードコンポーネントが正しく表示される', async ({ page }) => {
    await page.goto('/products');

    // 特定の要素だけをスクリーンショット
    const productCard = page.getByTestId('product-card').first();
    await expect(productCard).toHaveScreenshot('product-card.png');
  });

  test('ダークモードでも正しく表示される', async ({ page }) => {
    // カラースキームを設定
    await page.emulateMedia({ colorScheme: 'dark' });
    await page.goto('/');
    await page.waitForLoadState('networkidle');

    await expect(page).toHaveScreenshot('top-page-dark.png', {
      maxDiffPixelRatio: 0.02,
    });
  });
});

// playwright.config.ts - スクリーンショット設定
// expect: {
//   toHaveScreenshot: { threshold: 0.2 }, // ピクセル差異の許容閾値
// },
// スクリーンショット更新: npx playwright test --update-snapshots
```

**出典**:
- [Playwright Docs: Visual Comparisons](https://playwright.dev/docs/test-snapshots) (Playwright公式)
- [Playwright API: toHaveScreenshot](https://playwright.dev/docs/api/class-pageassertions#page-assertions-to-have-screenshot-1) (Playwright公式)

**バージョン**: Playwright 1.40+
**確信度**: 高
**最終更新**: 2026-05-06

---

#### 追加根拠 (2026-05-06) — ルール1「E2E テストはユーザーの重要なジャーニーに絞る」

新たに以下の記事/ドキュメントで同じプラクティスが推奨された:
- [Playwright README](https://raw.githubusercontent.com/microsoft/playwright/main/README.md) (microsoft/playwright / mainブランチ) ※2026-05-06に実際にfetch成功

Playwright の公式 README は3つの重要な実装詳細を強調: (1) **Auto-waiting** — "No artificial timeouts. Playwright waits for elements to be actionable, and assertions automatically retry until conditions are met." 人工的な `setTimeout` や `waitForTimeout` は不要で、`await expect(locator).toBeVisible()` のようなアサーションが自動リトライを行う。(2) **User-centric locators** — `page.getByRole()`, `page.getByLabel()`, `page.getByPlaceholder()`, `page.getByTestId()` を優先し、脆弱な CSS セレクタを避ける。(3) **トレース/デバッグ** — 失敗時に実行トレース・スクリーンショット・動画を自動保存し、Trace Viewer で「すべてのアクション・DOMスナップショット・ネットワークリクエスト・コンソールメッセージ」を詳細確認できる。`trace: 'on-first-retry'` 設定（rule 1 のコード例に記載）はこの機能を活用する推奨設定。

**確信度**: 既存（高）→ 高（公式 README で実証済み）

---

#### 追加根拠 (2026-05-06) — ルール2「Page Object Model でテストコードを保守しやすくする」

新たに以下の記事/ドキュメントで同じプラクティスが推奨された:
- [Playwright README](https://raw.githubusercontent.com/microsoft/playwright/main/README.md) (microsoft/playwright / mainブランチ) ※2026-05-06に実際にfetch成功
- [React Testing Library README](https://raw.githubusercontent.com/testing-library/react-testing-library/main/README.md) (testing-library / mainブランチ) ※2026-05-06に実際にfetch成功

Playwright README は「Page Objects Pattern for maintainable, scalable test suites」として POM を明示推奨。React Testing Library README はフレームワーク横断で「The more your tests resemble the way your software is used, the more confidence they can give you.」という哲学を提示。テスト実装の詳細（内部状態・コンポーネント構造）ではなくユーザーの操作（what）に着目することが、POM と RTL 両方で共通した原則として確認された。

**確信度**: 既存（高）→ 高（複数の公式 README で実証済み）

---

#### 追加根拠 (2026-05-06) — ルール5「`toHaveScreenshot()` で視覚的回帰テストを自動化する」

新たに以下の記事/ドキュメントで同じプラクティスが推奨された:
- [PlaywrightでVRT環境を構築し偽陽性を撲滅する](https://qiita.com/y_harada/items/b8a20797340a8ffcd75f) (Qiita y_harada / 2026-05) ※2026-05-06に実際にfetch成功
- [Visual Regression Testing: The Missing Piece in Your Testing Stack](https://dev.to/andresclua/visual-regression-testing-the-missing-piece-in-your-testing-stack-1c79) (dev.to andresclua / 2026-05) ※2026-05-06に実際にfetch成功

Dockerコンテナ内でPlaywrightを実行することで、CIとローカル環境のフォントレンダリング差異・サブピクセルアンチエイリアシングの違いを完全に排除し、スクリーンショット比較の偽陽性をゼロにできると両記事で実証された。共通した実装ポイント3点: (1) **CSSアニメーション無効化** — `page.addStyleTag({ content: '*, *::before, *::after { animation-duration: 0s !important; transition-duration: 0s !important; }' })` をVRTテスト前に実行し、アニメーション起因の差分を排除する。(2) **動的コンテンツのマスキング** — `toHaveScreenshot({ mask: [page.getByTestId('dynamic-date')] })` で日付・タイムスタンプなど毎回変わる要素をグレーボックスで隠す。(3) **Storybookとの連携** — `@storybook/test-runner` でStorybookのストーリーを全件VRTの対象にすることで、コンポーネント単位のリグレッションをCI段階で検出できる。

**確信度**: 既存（高）→ 高（Docker環境統一・Storybook連携で実証済み）

---

### 6. TypeScript インターフェースで定義した JSON テストデータでテストケースを動的生成する

テストデータを TypeScript インターフェースで型定義した JSON ファイルに外出しし、
`for` ループでテストケースを動的生成する。
テストコードと入力データを分離し、ケース追加コストをゼロにする。

**根拠**:
- テストデータをコード内にハードコードすると、ケース追加のたびにテストファイルを修正する必要がある
- JSON ファイルに分離することで、エンジニア以外（QA・企画）がテストデータを追加できる
- TypeScript インターフェースで JSON スキーマを定義することで、データの型安全性を保証できる
- 認証情報は JSON に含めず環境変数から取得し、リポジトリへの機密情報漏洩を防ぐ

**コード例**:
```ts
// e2e/data/types.ts - テストデータの型定義
interface ArticleData {
  id: string;
  title: string;
  body: string;
  tags: string[];
}

// e2e/data/test-articles.json
// [
//   { "id": "1", "title": "React入門", "body": "Reactの基礎...", "tags": ["react", "frontend"] },
//   { "id": "2", "title": "TypeScript基礎", "body": "TSの基礎...", "tags": ["typescript"] }
// ]

// e2e/article.spec.ts
import { test, expect } from '@playwright/test';
import testArticlesJson from './data/test-articles.json';

const testArticles = testArticlesJson as ArticleData[];

for (const article of testArticles) {
  test(`記事「${article.title}」を投稿できる`, async ({ page }) => {
    // 認証情報は環境変数から取得（JSON ファイルには含めない）
    await page.goto('/login');
    await page.getByLabel('メールアドレス').fill(process.env.TEST_USER_EMAIL!);
    await page.getByLabel('パスワード').fill(process.env.TEST_USER_PASSWORD!);
    await page.getByRole('button', { name: 'ログイン' }).click();

    await page.goto('/articles/new');
    await page.getByLabel('タイトル').fill(article.title);
    await page.getByLabel('本文').fill(article.body);

    for (const tag of article.tags) {
      await page.getByLabel('タグ').fill(tag);
      await page.keyboard.press('Enter');
    }

    await page.getByRole('button', { name: '投稿する' }).click();
    await expect(page.getByRole('heading', { name: article.title })).toBeVisible();
  });
}
```

**出典**:
- [PlaywrightのE2EテストでJSONデータを使ってテストケースを動的生成する](https://qiita.com/jptools_jp/items/7f5cdff3bf3fa77dffd3) (Qiita jptools_jp / 2026-05) ※2026-05-06に実際にfetch成功

**バージョン**: Playwright 1.40+, TypeScript 5+
**確信度**: 高
**最終更新**: 2026-05-06
