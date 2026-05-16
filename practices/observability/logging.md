# ロギングの実装ガイド

> **役割分担**: 「どの層が何をログするか」という責任分担は [`architecture/logging.md`](../architecture/logging.md) が定義する。このファイルは**実装パターン**（Pino 設定 / redact / ログレベル / Correlation ID 伝播）を集約する。
> エラー特化の経路は [`observability/error-tracking.md`](./error-tracking.md)、分散トレーシングは [`observability/tracing.md`](./tracing.md) を参照。

## ルール

### 1. Pino でサーバーサイドの構造化ログを設定し、`redact` で PII を自動マスクする

Next.js の Route Handler・Server Actions・サーバーコンポーネントのログには
Pino を使い、`redact` オプションで機密フィールドを自動マスクする。

**根拠**:
- Pino は Node.js 向け最速のロガーの1つで、ゼロコスト JSON ログを実現する
- `redact` オプションはログオブジェクトの特定パスを自動的に `[REDACTED]` に置換する
- Datadog・CloudWatch・Loki などのログサービスで構造化 JSON を検索・集計できる
- `pino-pretty` は開発時のみ使用し、本番では JSON のまま出力する

**コード例**:
```ts
// lib/logger.ts
import pino from 'pino';

const isDev = process.env.NODE_ENV !== 'production';

export const logger = pino({
  level: isDev ? 'debug' : 'info',

  // 本番: JSON 出力、開発: 人間が読みやすい形式
  transport: isDev
    ? { target: 'pino-pretty', options: { colorize: true, translateTime: 'SYS:standard' } }
    : undefined,

  // サービス情報を全ログに付与
  base: {
    service: process.env.SERVICE_NAME ?? 'frontend',
    env: process.env.NODE_ENV,
    version: process.env.NEXT_PUBLIC_APP_VERSION,
  },

  // 機密フィールドの自動マスク
  redact: {
    paths: [
      'password',
      'token',
      'secret',
      'authorization',
      '*.password',
      '*.token',
      '*.secret',
      '*.authorization',
      'req.headers.authorization',
      'req.headers.cookie',
      'user.email',   // PII: メールアドレスをマスク
      'user.phone',   // PII: 電話番号をマスク
    ],
    censor: '[REDACTED]',
  },
});

// 子ロガーでコンテキストを付与（リクエスト単位）
export function createRequestLogger(requestId: string, userId?: string) {
  return logger.child({
    requestId,
    userId: userId ? hashUserId(userId) : undefined,  // ハッシュ化してPII回避
  });
}

// lib/hash.ts
import { createHash } from 'crypto';
function hashUserId(userId: string): string {
  return createHash('sha256').update(userId).digest('hex').slice(0, 16);
}
```

```ts
// app/api/users/route.ts
import { logger } from '@/lib/logger';
import { nanoid } from 'nanoid';

export async function GET(request: Request) {
  const requestId = nanoid();
  const log = logger.child({ requestId, path: '/api/users' });

  log.info('Handling request');

  try {
    const users = await db.user.findMany();
    log.info({ count: users.length }, 'Users fetched');
    return Response.json(users);
  } catch (error) {
    log.error({ error }, 'Failed to fetch users');
    return Response.json({ error: 'Internal Server Error' }, { status: 500 });
  }
}
```

**出典**:
- [Pino Docs: Redaction](https://getpino.io/#/docs/redaction) (Pino公式)
- [Pino Docs: Child Logger](https://getpino.io/#/docs/child-loggers) (Pino公式)

**バージョン**: Pino 9+, Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. ログレベルをデータの重要度と緊急度で設計する

`debug` / `info` / `warn` / `error` の使い分けを明確にし、
本番ではアラートが必要なレベルのみを出力する。

**根拠**:
- ログレベルを間違えるとアラートが誤作動（warn の乱用）したり、問題の発見が遅れる（error の使い不足）
- 本番環境で `debug` ログを出力するとコストと I/O が無駄になる
- ログ集計ツールでレベルフィルタリングを使うためにレベルの一貫性が重要

**コード例**:
```ts
// ログレベルの使い分け指針

// debug: 開発時のみ。内部状態・リクエスト詳細
logger.debug({ queryParams, headers }, 'Incoming request details');
logger.debug({ cacheKey, hit: false }, 'Cache miss');

// info: 正常な業務フロー。主要なビジネスイベント
logger.info({ userId, orderId }, 'Order created successfully');
logger.info({ userId }, 'User logged in');
logger.info({ count: items.length }, 'Items loaded');

// warn: 異常だが回復可能。注意が必要だがアラート不要
logger.warn({ retryCount: 2, url }, 'API request retried');
logger.warn({ userId, planId }, 'User on deprecated plan');
logger.warn({ cacheHitRate: 0.3 }, 'Low cache hit rate');

// error: 回復不能なエラー。アラート必要
logger.error({ error, userId, operation: 'payment' }, 'Payment processing failed');
logger.error({ error, requestId }, 'Unhandled exception in route handler');

// Bad: error レベルを乱用する
logger.error('User not found');  // これは warn か info が適切
logger.error('Validation failed');  // 予期されるエラーは warn
```

**出典**:
- [Pino Docs: Levels](https://getpino.io/#/docs/api?id=levels) (Pino公式)

**バージョン**: Pino 9+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. クライアントサイドのエラーログは console ではなくエラー追跡サービスに集約する

ブラウザ側のエラーは `console.error` で出力するだけでなく、
Sentry などのサービスに送信して集計・アラートを設定する。

**根拠**:
- `console.error` はクライアントのブラウザコンソールに出力されるだけで、開発者は見ることができない
- Sentry はエラーの発生頻度・影響ユーザー数・スタックトレースを集計・アラートできる
- クライアントサイドのログは PII（個人情報）を含まない設計にする必要がある
- アーキテクチャのロギングルールと矛盾しないよう、詳細ガイドはこちらに集約する

**コード例**:
```ts
// lib/client-logger.ts
import * as Sentry from '@sentry/nextjs';

type LogLevel = 'info' | 'warn' | 'error';

interface LogOptions {
  extra?: Record<string, unknown>;
  tags?: Record<string, string>;
}

export const clientLogger = {
  info(message: string, options?: LogOptions) {
    if (process.env.NODE_ENV === 'development') {
      console.info(`[INFO] ${message}`, options?.extra);
    }
    // Sentry にブレッドクラムとして記録（エラー発生時のコンテキストになる）
    Sentry.addBreadcrumb({ level: 'info', message, data: options?.extra });
  },

  warn(message: string, options?: LogOptions) {
    console.warn(`[WARN] ${message}`, options?.extra);
    Sentry.addBreadcrumb({ level: 'warning', message, data: options?.extra });
  },

  error(message: string, error?: unknown, options?: LogOptions) {
    console.error(`[ERROR] ${message}`, error);

    // エラーオブジェクトがあれば例外として送信
    if (error instanceof Error) {
      Sentry.captureException(error, {
        extra: { message, ...options?.extra },
        tags: options?.tags,
      });
    } else {
      Sentry.captureMessage(message, {
        level: 'error',
        extra: { error, ...options?.extra },
        tags: options?.tags,
      });
    }
  },
};

// 使用例
try {
  await checkout();
} catch (error) {
  clientLogger.error('Checkout failed', error, {
    tags: { feature: 'checkout' },
    extra: { cartItemCount: cart.items.length },
  });
}
```

**出典**:
- [Sentry Docs: Capture Exception](https://docs.sentry.io/platforms/javascript/usage/) (Sentry公式)

**バージョン**: @sentry/nextjs 8+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. リクエスト ID（Correlation ID）を全ログに付与してトレースと相関させる

全てのリクエストに一意の ID を付与し、フロントエンド・BFF・バックエンドのログを横断して追跡できるようにする。

**根拠**:
- 分散システムでは単一リクエストが複数サービスをまたぐため、ID なしではログの紐付けが困難
- リクエスト ID を OpenTelemetry の Trace ID と同期させることで、トレーシングツールとの連携も可能
- Next.js の `headers()` API でリクエスト ID をレスポンスヘッダーに付与し、クライアントも受け取れる

**コード例**:
```ts
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { nanoid } from 'nanoid';

export function middleware(request: NextRequest) {
  // リクエスト ID がなければ生成（ロードバランサーが付与する場合はそちらを優先）
  const requestId = request.headers.get('X-Request-ID') ?? nanoid();

  const response = NextResponse.next({
    request: {
      headers: new Headers({
        ...Object.fromEntries(request.headers),
        'X-Request-ID': requestId,
      }),
    },
  });

  // レスポンスヘッダーにも付与（クライアントがサポートチケットと紐付けできる）
  response.headers.set('X-Request-ID', requestId);
  return response;
}

// app/api/orders/route.ts
import { headers } from 'next/headers';
import { logger } from '@/lib/logger';

export async function POST(request: Request) {
  const headersList = await headers();
  const requestId = headersList.get('X-Request-ID') ?? 'unknown';
  const log = logger.child({ requestId });

  log.info('Creating order');
  // バックエンド API 呼び出し時にリクエスト ID を伝播
  const res = await fetch('https://api.example.com/orders', {
    method: 'POST',
    headers: { 'X-Request-ID': requestId },
    body: await request.text(),
  });

  log.info({ status: res.status }, 'Order creation response');
  return Response.json(await res.json(), { status: res.status });
}
```

**出典**:
- [Next.js Docs: Middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware) (Next.js公式)

**バージョン**: Next.js 13+, Pino 9+
**確信度**: 高
**最終更新**: 2026-05-05

---

## 関連プラクティス

- [`architecture/logging.md`](../architecture/logging.md) - ロギングの基本ルール（構造化ログ採用・console.log 排除・PII 保護）
- [`observability/error-tracking.md`](./error-tracking.md) - Sentry によるエラー追跡
- [`observability/tracing.md`](./tracing.md) - リクエスト ID と OpenTelemetry Trace ID の連携
