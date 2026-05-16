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

### 7. `ignoreErrors` / `thirdPartyErrorFilterIntegration` / `beforeSend` の3層でノイズを削減する

Sentry に送信するイベントを3つの層で段階的にフィルタリングし、
アクション可能なエラーのみをアラートに繋げる。

**根拠**:
- ブラウザ拡張機能・サードパーティスクリプト・ResizeObserver 等のノイズが大量に入るとアラートが機能しない
- `ignoreErrors` は既知のノイズパターンを送信前に弾く（処理コストが最小）
- `thirdPartyErrorFilterIntegration` は自社コードのスタックフレームが一切含まれないエラーを自動排除する
- `beforeSend` でルールに収まらない残余ノイズや、`unhandledRejection` の不正シリアライズを修正・フィルタする

**コード例**:
```ts
// instrumentation-client.ts
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,

  // 層1: 既知のノイズパターンをパターンマッチで排除
  ignoreErrors: [
    /ResizeObserver loop/,               // ブラウザ内部ノイズ
    /Network request failed/,            // ネットワーク一時エラー
    /Loading chunk \d+ failed/,          // 再ロードで解消する ChunkLoadError
    /Non-Error promise rejection/,       // Object 等の非 Error リジェクション
  ],

  integrations: [
    // 層2: 自社コードフレームを持たないサードパーティエラーを排除
    Sentry.thirdPartyErrorFilterIntegration({
      filterKeys: ['your-domain.com'],   // 自社スクリプトのドメインを指定
      behaviour: 'drop-error-if-contains-third-party-frames-only',
    }),
  ],

  // 層3: 残余ノイズと不正シリアライズの正規化
  beforeSend(event, hint) {
    const error = hint.originalException;

    // unhandledRejection で value が '[object Object]' になる不正シリアライズを修正
    const firstValue = event.exception?.values?.[0];
    if (firstValue?.value === '[object Object]' || firstValue?.value === 'undefined') {
      firstValue.value = error
        ? JSON.stringify(error, Object.getOwnPropertyNames(error))
        : 'Unknown error';
    }

    return event;
  },
});
```

**出典**:
- [React × Sentry でエラーを逃さない実践パターン](https://zenn.dev/levi/articles/4ace7342e2f77f) (Zenn levi) ※2026-05-06に実際にfetch成功

**バージョン**: @sentry/nextjs 8+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 8. Next.js + Supabase スタックでは `Sentry.supabaseIntegration()` を使い、DB クエリをスパンとして計測する

`@sentry/nextjs` の `supabaseIntegration` を使って Supabase クライアントを Sentry に接続し、
全 DB クエリを自動的にトレーススパンとしてキャプチャする。
Next.js + Supabase Edge Functions の混在スタックでは、Sentry プロジェクトをランタイム別に分ける。

**根拠**:
- `supabaseIntegration` は Supabase の全クエリをスパンとして記録し、N+1 問題やスロークエリを本番で可視化できる
- DB クエリのスパン計測と Core Web Vitals を同一 Sentry プロジェクト内で照合することで、原因がフロントかバックエンドか判断できる
- Next.js（Node.js）・Supabase Edge Functions（Deno）を1プロジェクトに混在させると分析ノイズが増す
- 分離により、ランタイムごとのアラートルールとサンプリングレートを独立して設定できる

**コード例**:
```ts
// instrumentation-client.ts
import * as Sentry from '@sentry/nextjs';
import { createBrowserClient } from '@supabase/ssr';

const supabaseClient = createBrowserClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
);

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  integrations: [
    Sentry.supabaseIntegration(supabaseClient, Sentry, {
      tracing: true,      // DB クエリをスパンとして記録
      breadcrumbs: true,  // クエリをブレッドクラムにも記録
    }),
    Sentry.browserTracingIntegration(),
  ],
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
});
```

**Sentry プロジェクトの推奨分離構成**:
- `frontend` — Next.js のクライアント + サーバーコンポーネントエラー
- `supabase-edge` — Supabase Edge Functions（Deno ランタイム）
- 2つを1プロジェクトに混在させない（エラーの分析精度が低下する）

**出典引用**:
> "Instruments Supabase queries as spans in your traces so you can see exactly which DB calls are slow"
> ([From vibe code to production-ready: observability for Next.js and Supabase apps](https://blog.sentry.io/nextjs-supabase-observability/), セクション "Instrumenting Next.js and Supabase Edge Functions") ※2026-05-11に実際にfetch成功

> "Mixing Next.js errors with Deno errors and Postgres logs in a single project makes that analysis noisier and less useful."
> ([From vibe code to production-ready: observability for Next.js and Supabase apps](https://blog.sentry.io/nextjs-supabase-observability/), セクション "Instrumenting Next.js and Supabase Edge Functions") ※2026-05-11に実際にfetch成功

**バージョン**: @sentry/nextjs 9+, @supabase/ssr 1+
**確信度**: 高（公式 Sentry Blog）
**最終更新**: 2026-05-11

---

## 関連プラクティス

- [`architecture/error-handling.md`](../architecture/error-handling.md) - Next.js エラー境界の設計
- [`observability/logging.md`](./logging.md) - 構造化ロギング
- [`api-client/error-handling.md`](../api-client/error-handling.md) - API エラーの分類と処理

---

#### 追加根拠 (2026-05-06) — ルール1「Sentry を Next.js に統合し、全環境でエラーを自動キャプチャする」

新たに以下のソースでエラーカバレッジの拡張パターンが確認された:
- [getsentry/sentry-javascript packages/nextjs README](https://raw.githubusercontent.com/getsentry/sentry-javascript/develop/packages/nextjs/README.md) (Sentry公式 / developブランチ) ※2026-05-06に実際にfetch成功
- [React × Sentry でエラーを逃さない実践パターン](https://zenn.dev/levi/articles/4ace7342e2f77f) (Zenn levi) ※2026-05-06に実際にfetch成功

`Sentry.setUser()` / `Sentry.setTag()` でユーザーコンテキストを付与し、イベントを個人レベルでフィルタ・検索できるようになる。React 19 では `createRoot` に3種のエラーハンドラコールバックが追加され、`global-error.tsx` だけでは捕捉できなかったカテゴリをカバーできる: `onUncaughtError`（イベントハンドラ・非同期・setTimeout 内のエラー）、`onCaughtError`（ErrorBoundary でキャッチされたエラー — `Sentry.ErrorBoundary` 未使用時でも捕捉可）、`onRecoverableError`（ハイドレーションミスマッチ等 React が自動回復するエラー、`level: 'warning'` で記録推奨）。

**確信度**: 既存（高）→ 高（React 19 の追加エラーカテゴリを実践記事で確認）

---

#### 追加根拠 (2026-05-06) — ルール2「本番では tracesSampleRate を 0.1 以下に設定し、コストを管理する」

新たに以下の記事でエラーレベルの動的サンプリングパターンが示された:
- [React × Sentry でエラーを逃さない実践パターン](https://zenn.dev/levi/articles/4ace7342e2f77f) (Zenn levi) ※2026-05-06に実際にfetch成功

トレースサンプリング（`tracesSampleRate`）とは独立して、エラーイベント自体もサンプリングできる: `beforeSend` 内で通常エラーを 25% のみ通過させ、クリティカルパス（checkout・auth・payment 等）のエラーは 100% 送信する。この2段階サンプリングにより Sentry の月間イベント消費量を大幅に削減しながら、重要エラーを取りこぼさない。

**確信度**: 既存（高）→ 高（エラーレベルサンプリングの2段階パターンを実践記事で確認）

---

#### 追加根拠 (2026-05-06) — ルール4「Source Map をビルド時に自動アップロードし、本番のスタックトレースを可読にする」

新たに以下の記事で Source Map のリリース管理パターンが示された:
- [React × Sentry でエラーを逃さない実践パターン](https://zenn.dev/levi/articles/4ace7342e2f77f) (Zenn levi) ※2026-05-06に実際にfetch成功

`getsentry/action-release@v1` GitHub Action を使うと、デプロイ後に自動でリリースを作成し Source Map をアップロードできる。リリース名に "Application@1.0.0+202511100932"（名前+バージョン+タイムスタンプ）フォーマットを採用すると、Sentry 上でのリリース間比較・時系列フィルタリングが可能になる。`dist` を "yyyyMMddHHmm" 形式のタイムスタンプにすることで、同一バージョン内での複数デプロイも区別できる。

**確信度**: 既存（高）→ 高（GitHub Actions リリース連携と命名規則を実践記事で確認）
