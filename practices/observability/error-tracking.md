# エラー追跡のベストプラクティス

> **層責任**: このファイルは **観測 / 集計層**（Sentry でのキャプチャ、サンプリング、フィルタ、PII スクラブ、Source Map、リリース管理）を扱う。
> - **UI 層**（Error Boundary、`error.tsx`、Result 型、回復 UI）→ [`architecture/error-handling.md`](../architecture/error-handling.md)
> - **HTTP / API 層**（4xx・5xx 分類、リトライ判定）→ [`api-client/error-handling.md`](../api-client/error-handling.md)
>
> 流れは `HTTP エラー検知 → 分類 → 回復 UI → ログ・Sentry 通知（ここ）` の方向。

## ルール

### 1. Sentry を Next.js に統合し、全環境・全ランタイムでエラーを自動キャプチャする

`@sentry/nextjs` の公式ウィザードで初期設定を行い、クライアント・サーバー・Edge の 3 ランタイムを個別に設定する。
さらに React 19 環境では `createRoot` のエラーハンドラ 3 種（`onUncaughtError` / `onCaughtError` / `onRecoverableError`）も Sentry にブリッジし、ErrorBoundary でも `global-error.tsx` でも捕捉できないカテゴリを塞ぐ。

**根拠**:
- Sentry の Next.js SDK はクライアント・サーバー（Node.js）・Edge Runtime を個別に設定する必要がある
- ウィザード（`npx @sentry/wizard@latest -i nextjs`）が `instrumentation.ts` と `app/global-error.tsx` を自動生成する
- Source Map が自動アップロードされ、本番でも可読なスタックトレースが得られる
- `withSentryConfig` が `next.config.ts` をラップし、ビルド時に設定が適用される
- `Sentry.setUser()` / `Sentry.setTag()` でユーザーコンテキストを付与すると、イベントを個人レベルでフィルタ・検索できる
- React 19 の `onUncaughtError`（イベントハンドラ・非同期・setTimeout 内）/ `onCaughtError`（ErrorBoundary キャッチ済み・`Sentry.ErrorBoundary` 未使用時でも捕捉可）/ `onRecoverableError`（ハイドレーションミスマッチ等の自動回復系、`level: 'warning'` で記録推奨）を併用しないと捕捉漏れカテゴリが残る
- 複数ランタイム構成（Next.js + Edge Functions + Supabase 等）では Sentry プロジェクトを**ランタイムごとに分離**するのが Sentry 公式推奨。混在させると AI 支援デバッグの精度が下がる
- ORM / SDK の integration（`Sentry.supabaseIntegration()` 等）を `instrumentation.ts` に追加すると DB クエリ単位でスパンが取れ、N+1 を本番トレースから自動検出できる

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

**運用上の補足**:
- **プロジェクト分割**: Next.js（Node.js）/ Edge Functions（Deno 等）/ インフラログ（DB、CDN）を別 Sentry プロジェクトにする
- **Log Drain**: Supabase や外部インフラのログを Sentry に引き込み、アプリトレースとインフライベント（コネクション断等）を相関させる
- **N+1 自動検出**: 本番トレースで N+1 クエリを自動フラグ立て。dev 環境の小データセットでは見えない問題が本番規模で顕在化する

**出典**:
- [Sentry Docs: Next.js SDK](https://docs.sentry.io/platforms/javascript/guides/nextjs/) (Sentry公式)
- [getsentry/sentry-javascript packages/nextjs README](https://raw.githubusercontent.com/getsentry/sentry-javascript/develop/packages/nextjs/README.md) (Sentry 公式 develop ブランチ、React 19 の `createRoot` エラーハンドラ統合パターン) ※2026-05-06 fetch
- [React × Sentry でエラーを逃さない実践パターン](https://zenn.dev/levi/articles/4ace7342e2f77f) (Zenn levi) ※2026-05-06 fetch
- [From vibe code to production-ready: observability for Next.js and Supabase apps](https://blog.sentry.io/nextjs-supabase-observability/) (Sentry Blog、複数ランタイム分離 / DB スパン / N+1 検出) ※2026-05-12 fetch

**バージョン**: @sentry/nextjs 8+, Next.js 14+
**確信度**: 高
**最終更新**: 2026-05-12

---

### 2. 本番では tracesSampleRate を 0.1 以下に設定し、コストを管理する

開発・ステージング環境では 1.0、本番では 0.05〜0.1 を基本とし、
`tracesSampler` で重要なトランザクションのみ高サンプルする。

**根拠**:
- `tracesSampleRate: 1.0` の本番運用は高トラフィックで Sentry の容量コストが急増する
- 全トレースが必要なのは開発・ステージング時のみ（デバッグ目的）
- `tracesSampler` 関数で「チェックアウト」や「ログイン」などクリティカルフローのみ 100% サンプルできる
- エラーは `tracesSampleRate` と独立してキャプチャされるため、エラー取りこぼしは起きない
- エラーイベント自体も 2 段階サンプリングできる: `beforeSend` 内で通常エラーを 25% のみ通過させ、checkout / auth / payment 等のクリティカルパスのみ 100% 送信。月間イベント消費量を抑えつつ重要エラーを取りこぼさない

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
- [React × Sentry でエラーを逃さない実践パターン](https://zenn.dev/levi/articles/4ace7342e2f77f) (Zenn levi、エラーレベル 2 段階サンプリング) ※2026-05-06 fetch

**バージョン**: @sentry/nextjs 8+
**確信度**: 高
**最終更新**: 2026-05-06

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
- `getsentry/action-release@v1` GitHub Action を使うとデプロイ後に自動でリリースを作成し Source Map をアップロードできる
- リリース名を `Application@1.0.0+202611100932`（名前 + バージョン + タイムスタンプ）形式にすると Sentry 上での時系列比較が容易。`dist` を `yyyyMMddHHmm` 形式にしておけば同一バージョンの複数デプロイも区別できる

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
- [React × Sentry でエラーを逃さない実践パターン](https://zenn.dev/levi/articles/4ace7342e2f77f) (Zenn levi、`action-release` 連携 / リリース命名規則) ※2026-05-06 fetch

**バージョン**: @sentry/nextjs 8+, Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-06

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

### 9. Vercel Protected Source Maps でブラウザソースマップを認証保護する

本番環境に配置するブラウザソースマップ（`.map` ファイル）を Vercel の認証背後に置き、
チームメンバーのみがアクセスできる状態にする。外部ユーザーへは 404 を返すことで、
ソースマップ経由のコード構造露出を防ぎながら、内部のデバッグ品質を維持する。

**根拠**:
- ブラウザソースマップを公開したまま本番デプロイすると、難読化前のソースコード構造が誰でも参照できる
- Sentry へのアップロード（Rule #4）でデバッグは維持しつつ、`.map` ファイルへの外部アクセスを遮断できる
- Vercel の認証レイヤーを使うことで追加インフラなしに保護を実現できる

**出典引用**:
> "Your team can fetch them; everyone else gets a 404."
> ([Protected Source Maps: Ship browser source maps securely](https://vercel.com/changelog/protected-source-maps-ship-browser-source-maps-securely), Vercel Changelog) ※2026-05-16に実際にfetch成功

**バージョン**: Vercel（全プラン）
**確信度**: 高（Vercel 公式 Changelog — Pattern 1）
**最終更新**: 2026-05-16

---

## 関連プラクティス

- [`architecture/error-handling.md`](../architecture/error-handling.md) - Next.js エラー境界の設計
- [`observability/logging.md`](./logging.md) - 構造化ロギング
- [`api-client/error-handling.md`](../api-client/error-handling.md) - API エラーの分類と処理
