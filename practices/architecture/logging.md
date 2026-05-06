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

### 4. ログレベルを環境別に制御し、開発時は詳細・本番は必要最小限にする

環境変数 `LOG_LEVEL` でログレベルを動的に制御する。
開発環境では `debug` 以上の全ログを出力し、本番環境では `warn` または `error` のみを出力する。
ステージング環境は `info` 以上とし、パフォーマンスへの影響を最小化する。

**根拠**:
- `debug` ログを本番で出し続けるとログストレージコストが増大し、ノイズで重要なログが埋もれる
- 環境変数でレベルを制御することでデプロイなしにログ粒度を変更できる
- 本番での過剰なログはパフォーマンスに影響し（I/O増加）、PII 漏洩リスクも高まる

**コード例**:
```ts
// src/lib/logger.ts
import pino from 'pino';

type LogLevel = 'trace' | 'debug' | 'info' | 'warn' | 'error' | 'fatal';

function resolveLogLevel(): LogLevel {
  // 環境変数で明示指定があればそれを優先
  if (process.env.LOG_LEVEL) {
    return process.env.LOG_LEVEL as LogLevel;
  }
  // 環境別デフォルト
  switch (process.env.NODE_ENV) {
    case 'production':  return 'warn';   // warn 以上のみ（error/fatal含む）
    case 'test':        return 'error';  // テストでは error のみ（ログノイズ抑制）
    default:            return 'debug';  // 開発環境は全レベル
  }
}

export const logger = pino({
  level: resolveLogLevel(),
  // 本番はJSONそのまま（Datadogなどに流す）、開発は見やすく整形
  transport: process.env.NODE_ENV === 'development'
    ? { target: 'pino-pretty', options: { colorize: true, translateTime: 'SYS:HH:MM:ss' } }
    : undefined,
});

// 使用例（本番では debug は出力されない）
logger.debug({ userId }, 'Cache hit for user');  // 開発のみ
logger.info({ userId, action }, 'User action performed');  // 開発・staging
logger.warn({ userId, reason }, 'Rate limit approaching');  // 全環境
logger.error({ error, userId }, 'Payment failed');  // 全環境

// Bad: 環境を問わず常に詳細ログを出力
const logger = pino({ level: 'debug' }); // 本番でも debug が流れる
```

**出典**:
- [Pino Docs: Log Level](https://getpino.io/#/docs/api?id=level-string) (Pino公式)
- [OWASP: Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) (OWASP)

**バージョン**: Pino 8+, Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 5. 構造化ログにリクエストコンテキスト（traceId・userId）を付与する

サーバーサイドのログには `traceId`（リクエスト追跡ID）・`userId`・`requestPath` を
必ずフィールドとして含め、JSON 構造化ログとして出力する。
これにより複数サービスにまたがるリクエストのトレーシングが可能になる。

**根拠**:
- リクエストIDがないと、複数ユーザーの同時リクエストのログが混在して追跡できない
- 構造化フィールドにより Datadog・CloudWatch Logs Insights での集計・フィルタリングが容易
- `AsyncLocalStorage` を使えば明示的な引数渡しなしにコンテキストをスコープできる

**コード例**:
```ts
// src/lib/request-context.ts
import { AsyncLocalStorage } from 'async_hooks';
import { randomUUID } from 'crypto';

type RequestContext = {
  traceId: string;
  userId?: string;
  requestPath?: string;
};

export const requestContext = new AsyncLocalStorage<RequestContext>();

// src/lib/logger.ts
import pino from 'pino';
import { requestContext } from './request-context';

const baseLogger = pino({ level: resolveLogLevel() });

// コンテキストを自動付与するプロキシロガー
export const logger = new Proxy(baseLogger, {
  get(target, prop) {
    const ctx = requestContext.getStore();
    if (ctx && typeof target[prop as keyof typeof target] === 'function') {
      return (obj: object, msg: string) =>
        (target[prop as 'info'])({
          traceId: ctx.traceId,
          userId: ctx.userId,
          path: ctx.requestPath,
          ...obj,
        }, msg);
    }
    return target[prop as keyof typeof target];
  },
});

// src/middleware.ts (Next.js Middleware でコンテキストを設定)
import { NextRequest, NextResponse } from 'next/server';
import { requestContext } from '@/lib/request-context';
import { randomUUID } from 'crypto';

export function middleware(req: NextRequest) {
  const traceId = req.headers.get('x-trace-id') ?? randomUUID();
  const res = NextResponse.next();
  res.headers.set('x-trace-id', traceId);
  return res;
}

// ログ出力例（JSON）:
// { "level": 30, "traceId": "abc-123", "userId": "user-456", "path": "/api/posts", "msg": "Post created" }
```

**出典**:
- [Node.js Docs: AsyncLocalStorage](https://nodejs.org/api/async_context.html#class-asynclocalstorage) (Node.js公式)
- [Pino Docs: Child Loggers](https://getpino.io/#/docs/child-loggers) (Pino公式)

**バージョン**: Pino 8+, Next.js 13+, Node.js 16+
**確信度**: 中
**最終更新**: 2026-05-06

---

## 関連プラクティス

- [`observability/logging.md`](../observability/logging.md) - ロギングの詳細ガイド（Pino 設定・ログレベル設計・PII マスキング）
- [`observability/error-tracking.md`](../observability/error-tracking.md) - Sentry の詳細設定とリリーストラッキング
