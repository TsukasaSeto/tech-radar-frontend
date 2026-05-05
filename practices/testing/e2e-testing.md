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
