# ロギングのベストプラクティス

## ルール

### 1. 構造化ログ（JSON）を使い、`console.log` を本番環境から排除する

本番環境では構造化されたログライブラリ（Pino など）を使い、
`console.log` を直接使わない。

**根拠**:
- 構造化ログ（JSON）は Datadog・CloudWatch などのログサービスで検索・フィルタリングが容易
- ログレベル（info/warn/error/debug）で重要度を区別でき、アラート設定が可能
- `console.log` はクライアントバンドルに混入し、機密情報漏洩のリスクがある

**コード例**:
```ts
// src/lib/logger.ts
import pino from 'pino';

export const logger = pino({
  level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
  transport: process.env.NODE_ENV !== 'production'
    ? { target: 'pino-pretty', options: { colorize: true } }
    : undefined,
  base: {
    service: 'tech-radar-frontend',
    env: process.env.NODE_ENV,
  },
  redact: {
    paths: ['*.password', '*.token', '*.secret', '*.authorization'],
    censor: '[REDACTED]',
  },
});

// app/actions/user.ts
'use server';
import { logger } from '@/lib/logger';

export async function createUser(formData: FormData) {
  const email = formData.get('email') as string;

  logger.info({ email }, 'Creating user');

  try {
    const user = await db.user.create({ data: { email } });
    logger.info({ userId: user.id }, 'User created successfully');
    return { success: true };
  } catch (error) {
    logger.error({ error, email }, 'Failed to create user');
    throw error;
  }
}

// Bad: console.log を使う
console.log('Creating user:', email);
console.error('Error:', error.message);
```

**出典**:
- [Pino Docs: Best Practices](https://getpino.io/#/docs/web) (Pino公式)

**バージョン**: Pino 8+, Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. エラー監視サービス（Sentry など）を Next.js に統合する

未捕捉エラーとパフォーマンス問題を Sentry などのエラー監視サービスで追跡する。
ユーザー影響のあるエラーをリアルタイムで検知できる体制を整える。

**根拠**:
- ユーザーが遷遇しているエラーを開発者がリアルタイムで把握できる
- スタックトレース・ユーザー情報・リクエスト情報が自動的に収集される
- Next.js との公式統合（`@sentry/nextjs`）があり設定が容易

**コード例**:
```ts
// sentry.client.config.ts
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  replaysOnErrorSampleRate: 1.0,
  integrations: [
    Sentry.replayIntegration({
      maskAllText: true,
      blockAllMedia: false,
    }),
  ],
});

// sentry.server.config.ts
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1,
});

// 手動でエラーを報告
import * as Sentry from '@sentry/nextjs';

export async function processPayment(amount: number) {
  try {
    return await paymentGateway.charge(amount);
  } catch (error) {
    Sentry.captureException(error, {
      tags: { feature: 'payment' },
      extra: { amount },
    });
    throw error;
  }
}
```

**出典**:
- [Sentry Docs: Next.js SDK](https://docs.sentry.io/platforms/javascript/guides/nextjs/) (Sentry公式)

**バージョン**: @sentry/nextjs 8+, Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. クライアントサイドのエラーログは機密情報を含まないようにする

クライアントサイドのログや Sentry レポートに、パスワード・トークン・個人情報が
含まれないよう、ログ送信前にデータをサニタイズする。

**根拠**:
- クライアントから送信されたエラーログは第三者サービス（Sentry など）に保存される
- フォームの値・URL クエリパラメータにトークンや機密情報が含まれる可能性がある
- GDPR / 個人情報保護法への対応として最低限の情報のみを送信すべき

**コード例**:
```ts
Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,

  beforeSend(event) {
    if (event.request?.data) {
      const data = event.request.data as Record<string, unknown>;
      const sensitiveFields = ['password', 'token', 'secret', 'creditCard'];
      sensitiveFields.forEach(field => {
        if (field in data) data[field] = '[REDACTED]';
      });
    }

    if (event.request?.headers) {
      delete event.request.headers['Authorization'];
      delete event.request.headers['Cookie'];
    }

    return event;
  },

  beforeBreadcrumb(breadcrumb) {
    if (breadcrumb.data?.url) {
      const url = new URL(breadcrumb.data.url);
      ['token', 'secret', 'key'].forEach(param => url.searchParams.delete(param));
      breadcrumb.data.url = url.toString();
    }
    return breadcrumb;
  },
});
```

**出典**:
- [Sentry Docs: Sensitive Data](https://docs.sentry.io/platforms/javascript/data-management/sensitive-data/) (Sentry公式)
- [OWASP: Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) (OWASP)

**バージョン**: @sentry/nextjs 8+
**確信度**: 高
**最終更新**: 2026-05-05

---

## 関連プラクティス

- [`observability/logging.md`](../observability/logging.md) - ロギングの詳細ガイド（Pino 設定・ログレベル設計・PII マスキング）
- [`observability/error-tracking.md`](../observability/error-tracking.md) - Sentry の詳細設定とリリーストラッキング
