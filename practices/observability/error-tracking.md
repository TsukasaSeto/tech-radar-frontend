# エラー追跡のベストプラクティス

## ルール

### 1. Sentry を Next.js に統合し、全環境でエラーを自動キャプチャする

`@sentry/nextjs` の公式ウィザードで初期設定を行い、
クライアント・サーバー・Edge の3ランタイムを個別に設定する。

**根拠**:
- Sentry の Next.js SDK はクライアント・サーバー（Node.js）・Edge Runtime を個別に設定する必要がある
- ウィザード（`npx @sentry/wizard@latest -i nextjs`）が `instrumentation.ts` と `app/global-error.tsx` を自動生成する
- Source Map が自動アップロードされ、本番でも可読なスタックトレースが得られる
- `withSentryConfig` が `next.config.ts` をラップし、ビルド時に設定が適用される

**コード例**:
```bash
# ウィザードで初期設定（推奨）
npx @sentry/wizard@latest -i nextjs
```

```ts
// instrumentation-client.ts（クライアントサイド）
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NODE_ENV,

  // 本番では 10%、開発では 100% のトレースをサンプリング
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,

  // エラー発生時のセッションリプレイ
  replaysOnErrorSampleRate: 1.0,
  replaysSessionSampleRate: 0.01,  // 通常時は 1% のみ

  integrations: [
    Sentry.replayIntegration({
      maskAllText: true,    // テキストをマスク（PII保護）
      blockAllMedia: false,
    }),
    Sentry.browserTracingIntegration(),
  ],
});

// sentry.server.config.ts（サーバーサイド）
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1,
});

// app/global-error.tsx（ウィザードが生成）
'use client';
import * as Sentry from '@sentry/nextjs';
import { useEffect } from 'react';

export default function GlobalError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    Sentry.captureException(error);
  }, [error]);

  return (
    <html>
      <body>
        <div role="alert">
          <h2>エラーが発生しました</h2>
          <button onClick={reset}>再試行</button>
        </div>
      </body>
    </html>
  );
}
```

**出典**:
- [Sentry Docs: Next.js SDK](https://docs.sentry.io/platforms/javascript/guides/nextjs/) (Sentry公式)

**バージョン**: @sentry/nextjs 8+, Next.js 14+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. 本番では tracesSampleRate を 0.1 以下に設定し、コストを管理する

開発・ステージング環境では 1.0、本番では 0.05〜0.1 を基本とし、
`tracesSampler` で重要なトランザクションのみ高サンプルする。

**根拠**:
- `tracesSampleRate: 1.0` の本番運用は高トラフィックで Sentry の容量コストが急増する
- 全トレースが必要なのは開発・ステージング時のみ（デバッグ目的）
- `tracesSampler` 関数で「チェックアウト」や「ログイン」などクリティカルフローのみ 100% サンプルできる
- エラーは `tracesSampleRate` と独立してキャプチャされるため、エラー取りこぼしは起きない

**コード例**:
```ts
// instrumentation-client.ts
Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,

  // 環境別サンプリング設定
  tracesSampler: (samplingContext) => {
    const { name } = samplingContext;

    // クリティカルなフローは常にサンプル
    if (name.includes('/checkout') || name.includes('/login')) {
      return 1.0;
    }

    // ヘルスチェックはサンプルしない
    if (name.includes('/api/health')) {
      return 0;
    }

    // その他は環境で判定
    return process.env.NODE_ENV === 'production' ? 0.05 : 1.0;
  },
});
```

**出典**:
- [Sentry Docs: Performance Sampling](https://docs.sentry.io/platforms/javascript/performance/) (Sentry公式)

**バージョン**: @sentry/nextjs 8+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. `beforeSend` と `beforeBreadcrumb` で PII（個人情報）をスクラブする

エラーイベントをSentryへ送信する前に、パスワード・トークン・個人情報を
`beforeSend` / `beforeBreadcrumb` フックで除去する。

**根拠**:
- Sentry はサードパーティサービスであり、個人情報の送信は GDPR・個人情報保護法に抵触する可能性がある
- フォームの入力値・URL クエリパラメータ・リクエストヘッダーに機密情報が含まれうる
- `sendDefaultPii: false`（デフォルト）でも一部の情報が含まれるため、`beforeSend` での追加制御が必要
- ブレッドクラム（操作ログ）にも機密情報が入り込む可能性がある

**コード例**:
```ts
// instrumentation-client.ts
Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  sendDefaultPii: false,  // デフォルトは false（IPアドレス等を送らない）

  beforeSend(event) {
    // リクエストボディの機密フィールドをマスク
    if (event.request?.data) {
      const data = event.request.data as Record<string, unknown>;
      const sensitiveFields = ['password', 'token', 'secret', 'creditCard', 'ssn', 'pin'];
      for (const field of sensitiveFields) {
        if (field in data) data[field] = '[REDACTED]';
      }
    }

    // Authorization ヘッダーを除去
    if (event.request?.headers) {
      delete event.request.headers['Authorization'];
      delete event.request.headers['Cookie'];
      delete event.request.headers['X-API-Key'];
    }

    // URL のクエリパラメータからトークンを除去
    if (event.request?.url) {
      try {
        const url = new URL(event.request.url);
        ['token', 'secret', 'key', 'auth'].forEach(p => url.searchParams.delete(p));
        event.request.url = url.toString();
      } catch {
        // URLのパースに失敗した場合はそのまま送信
      }
    }

    return event;
  },

  beforeBreadcrumb(breadcrumb) {
    // URLブレッドクラムのクエリパラメータをクリーン
    if (breadcrumb.data?.url) {
      try {
        const url = new URL(breadcrumb.data.url as string);
        ['token', 'secret', 'key'].forEach(p => url.searchParams.delete(p));
        breadcrumb.data.url = url.toString();
      } catch {}
    }

    // フォーム送信ブレッドクラムを除去（パスワード等が含まれる可能性）
    if (breadcrumb.category === 'ui.input' && breadcrumb.message?.includes('password')) {
      return null;  // このブレッドクラムを送信しない
    }

    return breadcrumb;
  },
});
```

**出典**:
- [Sentry Docs: Sensitive Data](https://docs.sentry.io/platforms/javascript/data-management/sensitive-data/) (Sentry公式)

**バージョン**: @sentry/nextjs 8+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. Source Map をビルド時に自動アップロードし、本番のスタックトレースを可読にする

`withSentryConfig` の設定で Source Map のアップロードを自動化し、
本番環境でもソースコードにマッピングされたスタックトレースを得る。

**根拠**:
- 本番ビルドのコードは難読化・ミニファイされており、スタックトレースが読めない
- Source Map があれば元のファイル名・行番号・変数名がSentry上で確認できる
- Source Map ファイル自体はクライアントに配信されないため、ソースコード漏洩のリスクはない
- SENTRY_AUTH_TOKEN を CI の環境変数に設定することでビルド時に自動アップロードされる

**コード例**:
```ts
// next.config.ts
import { withSentryConfig } from '@sentry/nextjs';
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  // ... その他の設定
};

export default withSentryConfig(nextConfig, {
  org: 'your-org',
  project: 'your-project',

  // Source Map のアップロード設定
  silent: !process.env.CI,          // CI 以外ではログを抑制
  widenClientFileUpload: true,       // クライアントバンドルを広くアップロード
  hideSourceMaps: true,              // クライアントへの Source Map 配信を防ぐ
  disableLogger: true,               // 本番での Sentry ロガーを無効化（バンドルサイズ削減）
  automaticVercelMonitors: true,     // Vercel 環境でのモニタリング自動設定
});
```

```bash
# .env.sentry-build-plugin（ウィザードが生成）
SENTRY_AUTH_TOKEN=sntrys_...
```

```yaml
# .github/workflows/deploy.yml（CI での環境変数設定）
env:
  SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
  SENTRY_ORG: your-org
  SENTRY_PROJECT: your-project
```

**出典**:
- [Sentry Docs: Source Maps for Next.js](https://docs.sentry.io/platforms/javascript/guides/nextjs/manual-setup/#configure-source-maps) (Sentry公式)

**バージョン**: @sentry/nextjs 8+, Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 5. エラー境界と Sentry を連携させてコンポーネントツリーのエラーを追跡する

React の Error Boundary コンポーネントと Sentry を組み合わせ、
レンダリングエラーを自動でキャプチャする。

**根拠**:
- `global-error.tsx` はルートレベルのエラーのみキャッチするが、ネストした Error Boundary は手動設定が必要
- `Sentry.ErrorBoundary` コンポーネントを使うとエラーキャプチャとフォールバックUIが統合できる
- `beforeCapture` でエラーにタグを付与し、Sentry でのフィルタリングが容易になる

**コード例**:
```tsx
// components/ErrorBoundary/SentryBoundary.tsx
import * as Sentry from '@sentry/nextjs';

interface Props {
  children: React.ReactNode;
  fallback?: React.ReactNode;
  feature?: string;
}

export function SentryBoundary({ children, fallback, feature }: Props) {
  return (
    <Sentry.ErrorBoundary
      fallback={fallback ?? <DefaultErrorFallback />}
      beforeCapture={(scope) => {
        if (feature) scope.setTag('feature', feature);
      }}
      showDialog={false}
    >
      {children}
    </Sentry.ErrorBoundary>
  );
}

// 使用例
export default function DashboardPage() {
  return (
    <SentryBoundary feature="dashboard-stats" fallback={<StatsFallback />}>
      <StatsPanel />
    </SentryBoundary>
  );
}
```

**出典**:
- [Sentry Docs: React Error Boundary](https://docs.sentry.io/platforms/javascript/guides/react/features/error-boundary/) (Sentry公式)

**バージョン**: @sentry/nextjs 8+, React 18+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 6. `fingerprint` でエラーをグルーピングし、アラートノイズを削減する

Sentry の `fingerprint` フィールドを使ってエラーのグルーピングルールを明示的に定義し、
同種エラーが別々の Issue に分散することを防ぐ。

**根拠**:
- Sentry はデフォルトでスタックトレースベースにエラーをグルーピングするが、動的なメッセージや外部ライブラリのエラーは同種でも別 Issue になりやすい
- `fingerprint` を明示することで「同じ原因」のエラーを1つの Issue にまとめ、影響範囲の正確な把握とアラート制御が可能になる
- `beforeSend` 内で `event.fingerprint` を設定することで、エラーの種類・機能ドメイン・HTTPステータスコードを軸にグルーピングを制御できる
- Issue 数が適切にまとまることで、アラートのしきい値設定とエスカレーション設計が機能するようになる

**コード例**:
```ts
// Good
// instrumentation-client.ts
Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,

  beforeSend(event, hint) {
    const error = hint.originalException;

    // API エラーはステータスコードとエンドポイントでグルーピング
    if (error instanceof ApiError) {
      event.fingerprint = [
        'api-error',
        String(error.statusCode),
        error.endpoint,           // 例: '/api/orders'
      ];
    }

    // ネットワークエラーは種別でグルーピング（URLは含めない）
    if (error instanceof TypeError && error.message.includes('fetch')) {
      event.fingerprint = ['network-error', 'fetch-failed'];
    }

    // 機能ドメインタグを付与してダッシュボードでフィルタしやすくする
    if (event.tags?.feature) {
      event.fingerprint = [
        ...(event.fingerprint ?? ['{{ default }}']),
        event.tags.feature as string,
      ];
    }

    return event;
  },
});

// Bad
// fingerprintを設定しないと、同一エラーが複数の Issue に分散する
// 例: fetch エラーのメッセージに URL が含まれ、エンドポイントごとに別 Issue になる
Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  // fingerprint 未設定 → 同種エラーが Issue に散乱する
});
```

**出典**:
- [Sentry Docs: Fingerprinting](https://docs.sentry.io/concepts/data-management/event-grouping/fingerprint-rules/) (Sentry公式)

**バージョン**: @sentry/nextjs 8+
**確信度**: 高
**最終更新**: 2026-05-06

---

## 関連プラクティス

- [`architecture/error-handling.md`](../architecture/error-handling.md) - Next.js エラー境界の設計
- [`observability/logging.md`](./logging.md) - 構造化ロギング
- [`api-client/error-handling.md`](../api-client/error-handling.md) - API エラーの分類と処理
