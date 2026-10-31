# エラー追跡のベストプラクティス

> **層責任**: このファイルは **観測 / 集計層**（Sentry でのキャプチャ、サンプリング、フィルタ、PII スクラブ、Source Map、リリース管理）を扱う。
> - **UI 層**（Error Boundary、`error.tsx`、Result 型、回復 UI）→ [`architecture/error-handling.md`](../architecture/error-handling.md)
> - **HTTP / API 層**（4xx・5xx 分類、リトライ判定）→ [`api-client/error-handling.md`](../api-client/error-handling.md)
>
> 流れは `HTTP エラー検知 → 分類 → 回復 UI → ログ・Sentry 通知（ここ）` の方向。

## ルール

### 1. Sentry を Next.js に統合し、全環境・全ランタイムでエラーを自動キャプチャする

`@sentry/nextjs` の公式ウィザードで初期設定を行い、クライアント・サーバー・Edge の 3 ランタイムを個別に設定する。
App Router ではアプリ側が `createRoot` を呼ばないので、React 19 の root オプション（`onUncaughtError` など）は渡せない。捕捉の入口は `app/global-error.tsx`（クライアントのレンダリングエラー）、`instrumentation.ts` の `onRequestError`（Server Components・proxy などサーバー側のエラー）、Sentry ブラウザ SDK が既定で張るグローバルハンドラ（`onerror` / `unhandledrejection`。イベントハンドラや `setTimeout` 内の例外）の 3 つで揃える。

**根拠**:
- Sentry の Next.js SDK はクライアント（`instrumentation-client.ts`）・サーバー（Node.js、`sentry.server.config.ts`）・Edge Runtime（`sentry.edge.config.ts`）を個別に設定し、サーバーと Edge は `instrumentation.ts` の `register()` から読み込む
- ウィザード（`npx @sentry/wizard@latest -i nextjs`）が `instrumentation.ts` と `app/global-error.tsx` を自動生成する
- `onRequestError = Sentry.captureRequestError` で Server Components・middleware / proxy のエラーを捕捉できる（`@sentry/nextjs` 8.28.0+、Next.js 15+）
- Source Map が自動アップロードされ、本番でも可読なスタックトレースが得られる
- `output: "standalone"` の Docker イメージでは、`.next/standalone` に `instrumentation.js` が含まれず `register()` が走らないため、サーバー側の Sentry が一度も初期化されなかった事例が報告されている（Next.js 16.4.2 + Turbopack + `@sentry/nextjs` 10.76.2 での 1 件の観測で、上流の既知問題かは未確認）。`captureException` はイベント ID を返しても送信されず、`Sentry.flush()` が `false` を返す。導入後と大きなアップグレード後は、server / edge / browser の各ランタイムからテストエラーを送り、本番と同じ起動方法（standalone なら `.next/standalone/server.js`）で `flush()` の戻り値と Sentry 側の到着を確認する
- `withSentryConfig` が `next.config.ts` をラップし、ビルド時に設定が適用される（v11 では `@sentry/nextjs/config` から import する。v10 以前は `@sentry/nextjs`）
- `Sentry.setUser()` / `Sentry.setTag()` でユーザーコンテキストを付与すると、イベントを個人レベルでフィルタ・検索できる
- React 19 の `onUncaughtError` は「レンダリング中に投げられ Error Boundary で捕捉されなかったエラー」、`onCaughtError` は「Error Boundary が捕捉したエラー」、`onRecoverableError` は「React が自動回復したエラー（ハイドレーションミスマッチ等）」が対象。イベントハンドラや `setTimeout` などの非同期コードの例外は Error Boundary も root オプションも経由しないので、グローバルハンドラで拾う。root オプションで Sentry にブリッジできるのは、自前で `createRoot` を呼ぶ SPA（Vite など）の場合に限る
- Supabase など複数ランタイム構成でのプロジェクト分割と DB クエリのスパン計測は #8 を参照

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

// sentry.server.config.ts（サーバーサイド。sentry.edge.config.ts も同じ形で作る）
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1,
});

// instrumentation.ts（サーバー / Edge の設定を読み込み、サーバー側のエラーを捕捉する）
import * as Sentry from '@sentry/nextjs';

export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    await import('./sentry.server.config');
  }
  if (process.env.NEXT_RUNTIME === 'edge') {
    await import('./sentry.edge.config');
  }
}

export const onRequestError = Sentry.captureRequestError;

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
- [Sentry Docs: Next.js Manual Setup](https://docs.sentry.io/platforms/javascript/guides/nextjs/manual-setup/) (Sentry公式、`instrumentation.ts` の `register()` / `onRequestError`、`global-error.tsx`、`@sentry/nextjs/config`) ※2026-10-25 確認
- [Sentry Docs: GlobalHandlers](https://docs.sentry.io/platforms/javascript/guides/nextjs/configuration/integrations/globalhandlers/) (Sentry公式、既定で有効な `onerror` / `onunhandledrejection` の捕捉) ※2026-10-25 確認
- [React: createRoot](https://react.dev/reference/react-dom/client/createRoot) (React 公式、`onUncaughtError` / `onCaughtError` / `onRecoverableError` の対象) ※2026-10-25 確認
- [React: Component（Error Boundary が捕捉しないエラー）](https://react.dev/reference/react/Component) (React 公式、イベントハンドラ・非同期コードは対象外) ※2026-10-25 確認
- [getsentry/sentry-javascript packages/nextjs README](https://raw.githubusercontent.com/getsentry/sentry-javascript/develop/packages/nextjs/README.md) (Sentry 公式 develop ブランチ、React 19 の `createRoot` エラーハンドラ統合パターン) ※2026-05-06 fetch
- [React × Sentry でエラーを逃さない実践パターン](https://zenn.dev/levi/articles/4ace7342e2f77f) (Zenn levi) ※2026-05-06 fetch
- [Sentry never ran on my Next.js server, and the dashboard looked fine](https://dev.to/asermu/sentry-never-ran-on-my-nextjs-server-and-the-dashboard-looked-fine-1gpa) (dev.to asermu、standalone 出力で `instrumentation.js` が欠落し server Sentry が未初期化。単一の個人記事のため補足根拠) ※2026-10-31 fetch
  > "Check Sentry.flush()'s return value in a test route. An event id alone proves nothing."
  > (同記事、末尾 "What I'd tell past me")

**バージョン**: @sentry/nextjs 8.28+（v11 基準）, Next.js 15+（`onRequestError`。v11 の最小対応は Next.js 14）
**確信度**: 高
**最終更新**: 2026-10-31

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
- **`tunnelRoute` は `tracesSampleRate` と掛け合わさるとコストの上限を消す**: `tunnelRoute` は広告ブロッカー回避のため Sentry イベントを自社サーバー（Serverless Function）経由で送信する設定だが、`tracesSampleRate: 1.0` と組み合わせると全トレースが自社関数の呼び出しコストとして跳ね返る。トラフィックが 4 倍（80k→350k PV）になった際、月額 ¥10万→¥35万にコストが跳ね上がった実例がある。設定変更直後ではなくトラフィック増加時に遅れて顕在化するため、`tunnelRoute` を使う場合は特に本番の `tracesSampleRate` を低く保つ

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
- [Sentryの設定2行でVercel代が月10万円→35万円になった話](https://zenn.dev/slotana777/articles/sentry-tunnel-route-2400usd) (Zenn slotana777、`tunnelRoute` + `tracesSampleRate: 1.0` の組み合わせによるコスト事故の実例) ※2026-08-03に実際にfetch成功
  > "地雷は、埋めた時には爆発しない。"
  > (セクション "学んだこと")

**バージョン**: @sentry/nextjs 8+
**確信度**: 高
**最終更新**: 2026-08-03

---

### 3. `beforeSend` と `beforeBreadcrumb` で PII（個人情報）をスクラブする

エラーイベントをSentryへ送信する前に、パスワード・トークン・個人情報を
`beforeSend` / `beforeBreadcrumb` フックで除去する。
`sendDefaultPii: false` を前提にできるのは v10 以前だけで、v11 では `sendDefaultPii` が廃止され、`dataCollection` を書かないと Cookie・ユーザー情報・HTTP ボディなどを収集する側に倒れる。v11 では `dataCollection` で各カテゴリを明示的に宣言したうえで（詳細は #11）、同じ `beforeSend` を併用する。

**根拠**:
- Sentry はサードパーティサービスであり、個人情報の送信は GDPR・個人情報保護法に抵触する可能性がある
- フォームの入力値・URL クエリパラメータ・リクエストヘッダーに機密情報が含まれうる
- v10 以前は `sendDefaultPii: false`（デフォルト）でも一部の情報が含まれるため、`beforeSend` での追加制御が必要
- v11 の既定では `userInfo` / `cookies` が収集され、`httpBodies` もリクエスト・レスポンスのボディを収集する。`sendDefaultPii: false` を残したまま上げても効かない
- サーバー側（Node）のリクエストヘッダーのキーは小文字で届くため、`'Authorization'` のような大文字キーの完全一致では消えない。キーは小文字に正規化して比較する
- `event.request.data` はオブジェクトとは限らず文字列のこともある。型を判定せずに `in` 演算子を使うと `beforeSend` が例外で終わる。ネストしたオブジェクトも再帰的にスクラブする
- `beforeSend` はクライアントとサーバーの両方の設定に入れる（片方だけだと、もう片方のランタイムのイベントが素通りする）
- ブレッドクラム（操作ログ）にも機密情報が入り込む可能性がある

**コード例**:
```ts
// lib/sentry-scrub.ts（instrumentation-client.ts と sentry.server.config.ts の両方から使う）
const SENSITIVE_KEYS = new Set([
  'password', 'token', 'secret', 'creditcard', 'ssn', 'pin',
  'authorization', 'cookie', 'set-cookie', 'x-api-key',
]);
const SENSITIVE_QUERY_PARAMS = ['token', 'secret', 'key', 'auth'];

// ヘッダーは Node では小文字、フォーム値は camelCase など表記が揺れるので小文字で比較する
const isSensitiveKey = (key: string) => SENSITIVE_KEYS.has(key.toLowerCase());

function scrubValue(value: unknown): unknown {
  if (Array.isArray(value)) return value.map(scrubValue);
  if (value !== null && typeof value === 'object') {
    return Object.fromEntries(
      Object.entries(value).map(([k, v]) => [k, isSensitiveKey(k) ? '[REDACTED]' : scrubValue(v)]),
    );
  }
  return value;
}

function scrubBody(data: unknown): unknown {
  if (typeof data !== 'string') return scrubValue(data);
  try {
    return JSON.stringify(scrubValue(JSON.parse(data)));
  } catch {
    return '[REDACTED]'; // JSON でない文字列ボディは中身を判定できないので丸ごと伏せる
  }
}

export function scrubUrl(raw: string): string {
  try {
    const url = new URL(raw);
    SENSITIVE_QUERY_PARAMS.forEach((p) => url.searchParams.delete(p));
    return url.toString();
  } catch {
    return raw.replace(/\?.*$/, ''); // 相対 URL などパースできない値はクエリごと落とす
  }
}

type ScrubbableEvent = {
  request?: { headers?: Record<string, string>; data?: unknown; url?: string };
};

export function scrubEvent(event: ScrubbableEvent): void {
  const request = event.request;
  if (!request) return;
  if (request.headers) {
    request.headers = Object.fromEntries(
      Object.entries(request.headers).filter(([k]) => !isSensitiveKey(k)),
    );
  }
  if (request.data !== undefined) request.data = scrubBody(request.data);
  if (request.url) request.url = scrubUrl(request.url);
}
```

```ts
// instrumentation-client.ts（v8〜v10。sentry.server.config.ts にも同じ beforeSend を設定する）
import * as Sentry from '@sentry/nextjs';
import { scrubEvent, scrubUrl } from './lib/sentry-scrub';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  sendDefaultPii: false,  // v10 以前のデフォルト（IPアドレス等を送らない）。v11 では廃止

  beforeSend(event) {
    scrubEvent(event);
    return event;
  },

  beforeBreadcrumb(breadcrumb) {
    // URLブレッドクラムのクエリパラメータをクリーン
    if (typeof breadcrumb.data?.url === 'string') {
      breadcrumb.data.url = scrubUrl(breadcrumb.data.url);
    }

    // フォーム送信ブレッドクラムを除去（パスワード等が含まれる可能性）
    if (breadcrumb.category === 'ui.input' && breadcrumb.message?.includes('password')) {
      return null;  // このブレッドクラムを送信しない
    }

    return breadcrumb;
  },
});
```

```ts
// instrumentation-client.ts（v11。収集カテゴリを dataCollection で宣言し、beforeSend も併用する）
import * as Sentry from '@sentry/nextjs';
import { scrubEvent, scrubUrl } from './lib/sentry-scrub';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  dataCollection: {
    userInfo: false,
    cookies: false,
    httpHeaders: {
      request: { deny: ['authorization', 'cookie', 'x-api-key', 'forwarded', '-ip', 'remote-', 'via', '-user'] },
      response: { deny: ['set-cookie', 'forwarded', '-ip', 'remote-', 'via', '-user'] },
    },
    httpBodies: [],
    urlQueryParams: { deny: ['token', 'secret', 'key', 'auth'] },
  },

  beforeSend(event) {
    scrubEvent(event);  // deny list に載らないヘッダー・ボディ・URL のキーはここで落とす
    return event;
  },

  beforeBreadcrumb(breadcrumb) {
    if (typeof breadcrumb.data?.url === 'string') {
      breadcrumb.data.url = scrubUrl(breadcrumb.data.url);
    }
    return breadcrumb;
  },
});
```

**出典**:
- [Sentry Docs: Sensitive Data](https://docs.sentry.io/platforms/javascript/data-management/sensitive-data/) (Sentry公式)
- [Sentry Docs: Data Collected（Next.js）](https://docs.sentry.io/platforms/javascript/guides/nextjs/data-management/data-collected/) (Sentry公式、`dataCollection` の各カテゴリと allow / deny list、`setUser()` のデータは常に送信) ※2026-10-25 確認
- [Sentry Docs: Migrate from v10 to v11（Next.js）](https://docs.sentry.io/platforms/javascript/guides/nextjs/migration/v10-to-v11/) (Sentry公式、`sendDefaultPii` の廃止と v11 の収集側デフォルト) ※2026-10-25 確認

**バージョン**: @sentry/nextjs 8〜10（`sendDefaultPii` の例）/ 11+（`dataCollection` の例）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 4. Source Map をビルド時に自動アップロードし、本番のスタックトレースを可読にする

`withSentryConfig` の設定で Source Map のアップロードを自動化し、
本番環境でもソースコードにマッピングされたスタックトレースを得る。

**根拠**:
- 本番ビルドのコードは難読化・ミニファイされており、スタックトレースが読めない
- Source Map があれば元のファイル名・行番号・変数名がSentry上で確認できる
- v9 以降の SDK は、クライアントの Source Map を `hidden-source-map`（`sourceMappingURL` を埋め込まない）で出力し、アップロード後に既定で削除する（`sourcemaps.deleteSourcemapsAfterUpload: true`）。ただし Source Map を自分で明示的に有効化している場合、SDK は出力方法を変えず削除もしないので、`.map` が公開されたまま本番に出る。配信を制限する方法は #9 を参照
- `hideSourceMaps` は v9 で代替なしに削除された（v8 以前の設定。v9 以降は上の既定動作に置き換わった）
- SENTRY_AUTH_TOKEN を CI の環境変数に設定することでビルド時に自動アップロードされる
- `getsentry/action-release@v1` GitHub Action を使うとデプロイ後に自動でリリースを作成し Source Map をアップロードできる
- リリース名を `Application@1.0.0+202611100932`（名前 + バージョン + タイムスタンプ）形式にすると Sentry 上での時系列比較が容易。`dist` を `yyyyMMddHHmm` 形式にしておけば同一バージョンの複数デプロイも区別できる

**コード例**:
```ts
// next.config.ts（v11）
import { withSentryConfig } from '@sentry/nextjs/config';
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
  sourcemaps: {
    deleteSourcemapsAfterUpload: true, // アップロード後にクライアントの .map を削除（既定値。明示しておく）
  },

  // 以下の 2 つは webpack でビルドするときだけ効く
  webpack: {
    treeshake: { removeDebugLogging: true }, // Sentry のデバッグログを除去（バンドルサイズ削減）
    automaticVercelMonitors: true,           // vercel.json の cron からモニターを自動作成
  },
});
```

v10 以前は `withSentryConfig` を `@sentry/nextjs` から import し、`disableLogger: true` と `automaticVercelMonitors: true` をトップレベルに書く。v8 以前は `hideSourceMaps: true` でクライアントへの配信を防いでいた。

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
- [Sentry Docs: Build Options（Next.js）](https://docs.sentry.io/platforms/javascript/guides/nextjs/configuration/build/) (Sentry公式、`sourcemaps.deleteSourcemapsAfterUpload` / `widenClientFileUpload` / `webpack.automaticVercelMonitors`) ※2026-10-25 確認
- [Sentry Docs: Migrate from v8 to v9（Next.js）](https://docs.sentry.io/platforms/javascript/guides/nextjs/migration/v8-to-v9/) (Sentry公式、`hideSourceMaps` の削除と hidden source map・アップロード後削除の既定化) ※2026-10-25 確認
- [Sentry Docs: Migrate from v10 to v11（Next.js）](https://docs.sentry.io/platforms/javascript/guides/nextjs/migration/v10-to-v11/) (Sentry公式、`@sentry/nextjs/config` への移動、`disableLogger` → `webpack.treeshake.removeDebugLogging`) ※2026-10-25 確認
- [React × Sentry でエラーを逃さない実践パターン](https://zenn.dev/levi/articles/4ace7342e2f77f) (Zenn levi、`action-release` 連携 / リリース命名規則) ※2026-05-06 fetch

**バージョン**: @sentry/nextjs 11+（v10 以前の書き方は本文の注記）, Next.js 14+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 5. エラー境界と Sentry を連携させてコンポーネントツリーのエラーを追跡する

React の Error Boundary コンポーネントと Sentry を組み合わせ、
レンダリングエラーを自動でキャプチャする。

**根拠**:
- `global-error.tsx` はルートレベルのエラーのみキャッチするが、ネストした Error Boundary は手動設定が必要
- `Sentry.ErrorBoundary` コンポーネントを使うとエラーキャプチャとフォールバックUIが統合できる
- `beforeCapture` でエラーにタグを付与し、Sentry でのフィルタリングが容易になる
- `Sentry.ErrorBoundary` は class コンポーネントで、`beforeCapture` のような関数 props を受け取るため、ラッパーは `'use client'` を付けた Client Component にする。Server Component からは、関数を渡さずにラッパーを使う

**コード例**:
```tsx
// components/ErrorBoundary/SentryBoundary.tsx
'use client';

import * as Sentry from '@sentry/nextjs';
import type { ReactElement, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback?: ReactElement;
  feature?: string;
}

function DefaultErrorFallback() {
  return <p role="alert">表示中にエラーが発生しました</p>;
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
```

```tsx
// app/dashboard/page.tsx（Server Component から使う例）
import { SentryBoundary } from '@/components/ErrorBoundary/SentryBoundary';
import { StatsFallback, StatsPanel } from './stats';

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
**最終更新**: 2026-10-25

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
import * as Sentry from '@sentry/nextjs';
import { ApiError } from '@/lib/api-error'; // api-client/error-handling.md #1 の ApiError

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,

  beforeSend(event, hint) {
    const error = hint.originalException;

    // API エラーはステータスコードとエラーコードでグルーピング（URL は動的な ID を含むので入れない）
    if (error instanceof ApiError) {
      event.fingerprint = [
        'api-error',
        String(error.status),
        error.code,               // 例: 'ORDER_NOT_FOUND'
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
**最終更新**: 2026-10-25

---

### 7. `ignoreErrors` / `thirdPartyErrorFilterIntegration` / `beforeSend` の3層でノイズを削減する

Sentry に送信するイベントを3つの層で段階的にフィルタリングし、
アクション可能なエラーのみをアラートに繋げる。
`thirdPartyErrorFilterIntegration` は設定を誤ると自社のエラーまで「サードパーティ」と判定して捨てるので、導入は次の順で進める。

1. `withSentryConfig` のビルドオプション `applicationKey` と、`thirdPartyErrorFilterIntegration` の `filterKeys` に同じ値を設定する
2. 最初は `behaviour: 'apply-tag-if-exclusively-contains-third-party-frames'` でタグ付けだけ行い、Issue 検索の `third_party_code:true` に自社コード起因のエラーが混ざっていないことを確かめる
3. 問題がなければ `drop-error-if-exclusively-contains-third-party-frames` に切り替えて破棄する

**根拠**:
- ブラウザ拡張機能・サードパーティスクリプト・ResizeObserver 等のノイズが大量に入るとアラートが機能しない
- `ignoreErrors` は既知のノイズパターンを送信前に弾く（処理コストが最小）。文字列は部分一致、正規表現はパターン一致でエラーメッセージと照合される
- `ignoreErrors` で `/Non-Error promise rejection/` のような広いパターンを一律に除外すると、`beforeSend` の正規化に届く前に捨てられ、Error 以外の値で reject しているアプリ側の実際のバグも見えなくなる。正規化したあとで、調査して無害と判断した値だけを落とす
- `thirdPartyErrorFilterIntegration` は、ビルド時にバンドラープラグインが自社コードへ `applicationKey` を埋め込み、実行時に各スタックフレームのキーを `filterKeys` と照合する。キーのないフレームはサードパーティ扱いになるため、`filterKeys` にドメインを渡しても一致せず、自社のエラーも含めてほぼ全部が「サードパーティのみ」と判定される
- `behaviour` は 4 種類: `drop-error-if-contains-third-party-frames`（サードパーティのフレームが 1 つでもあれば破棄）/ `drop-error-if-exclusively-contains-third-party-frames`（全フレームがサードパーティなら破棄）/ `apply-tag-if-contains-third-party-frames` / `apply-tag-if-exclusively-contains-third-party-frames`（破棄せず `third_party_code: true` タグを付ける）。自社コードを経由するエラーまで消さないよう、通常は `exclusively` の方を使う
- `thirdPartyErrorFilterIntegration` はブラウザ向け SDK の機能（8.10.0+）で、Sentry の Loader Script / CDN バンドルとは併用できない（SDK 自体がサードパーティと判定され、ほぼ全イベントに影響する）
- `beforeSend` でルールに収まらない残余ノイズや、`unhandledRejection` の不正シリアライズを修正・フィルタする

**コード例**:
```ts
// next.config.ts（v11）
import { withSentryConfig } from '@sentry/nextjs/config';
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {};

export default withSentryConfig(nextConfig, {
  org: 'your-org',
  project: 'your-project',
  applicationKey: 'my-app',  // 自社のビルド成果物に埋め込むキー。filterKeys と同じ値にする
});
```

v10 以前は、webpack では `unstable_sentryWebpackPluginOptions: { applicationKey: 'my-app' }`、Turbopack では `_experimental: { turbopackApplicationKey: 'my-app' }` で指定する（v11 でトップレベルの `applicationKey` に統合された）。

```ts
// instrumentation-client.ts
import * as Sentry from '@sentry/nextjs';

// 調査して無害と判断した reject 値だけを列挙する（例: 自社の UI ライブラリがキャンセル時に reject する文字列）
const KNOWN_HARMLESS_REJECTIONS = new Set(['cancelled']);

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,

  // 層1: 既知のノイズパターンをパターンマッチで排除（広すぎるパターンは入れない）
  ignoreErrors: [
    /ResizeObserver loop/,               // ブラウザ内部ノイズ
    /Network request failed/,            // ネットワーク一時エラー
    /Loading chunk \d+ failed/,          // 再ロードで解消する ChunkLoadError
  ],

  integrations: [
    // 層2: 自社コードのフレームを持たないサードパーティエラーを判定する
    Sentry.thirdPartyErrorFilterIntegration({
      filterKeys: ['my-app'],            // next.config.ts の applicationKey と同じ値
      // 導入直後はタグ付けだけ。third_party_code:true の内訳を確かめてから
      // 'drop-error-if-exclusively-contains-third-party-frames' に切り替える
      behaviour: 'apply-tag-if-exclusively-contains-third-party-frames',
    }),
  ],

  // 層3: 残余ノイズと不正シリアライズの正規化
  beforeSend(event, hint) {
    const error = hint.originalException;

    // 無害と確認済みの非 Error リジェクションだけを落とす
    if (typeof error === 'string' && KNOWN_HARMLESS_REJECTIONS.has(error)) {
      return null;
    }

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
- [Sentry Docs: Filtering（Next.js）](https://docs.sentry.io/platforms/javascript/guides/nextjs/configuration/filtering/) (Sentry公式、`ignoreErrors` の照合方法、`thirdPartyErrorFilterIntegration` の `filterKeys` / `behaviour` 4 種 / `applicationKey`) ※2026-10-25 確認
- [Sentry Docs: Build Options（Next.js）](https://docs.sentry.io/platforms/javascript/guides/nextjs/configuration/build/) (Sentry公式、`applicationKey` は `filterKeys` と一致させる) ※2026-10-25 確認
- [Sentry Docs: Migrate from v10 to v11（Next.js）](https://docs.sentry.io/platforms/javascript/guides/nextjs/migration/v10-to-v11/) (Sentry公式、`_experimental.turbopackApplicationKey` → `applicationKey`、`unstable_sentryWebpackPluginOptions` の削除) ※2026-10-25 確認
- [React × Sentry でエラーを逃さない実践パターン](https://zenn.dev/levi/articles/4ace7342e2f77f) (Zenn levi) ※2026-05-06に実際にfetch成功

**バージョン**: @sentry/nextjs 8.10+（`thirdPartyErrorFilterIntegration`）。設定例は v11 基準
**確信度**: 高
**最終更新**: 2026-10-25

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
- インフラログ（DB、CDN）も別プロジェクトにし、Supabase や外部インフラのログは Log Drain で Sentry に引き込んで、アプリのトレースとインフラのイベント（コネクション断等）を相関させる
- 本番トレースでは N+1 クエリが自動でフラグ付けされる。dev 環境の小さなデータセットでは見えない問題が本番規模で顕在化する
- 引数はオブジェクト形式の `{ supabaseClient }` で、公式 docs に載っているオプションはこれだけ。`supabaseIntegration(client, Sentry, opts)` の 3 引数形式や `tracing` / `breadcrumbs` オプションは使わない（DB 操作のブレッドクラムとエラーのキャプチャは自動で付く）

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
    Sentry.supabaseIntegration({ supabaseClient }),  // DB クエリをスパンとブレッドクラムとして記録
    Sentry.browserTracingIntegration(),
  ],
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
});
```

**Sentry プロジェクトの推奨分離構成**:
- `frontend` — Next.js のクライアント + サーバーコンポーネントエラー
- `supabase-edge` — Supabase Edge Functions（Deno ランタイム）
- 2つを1プロジェクトに混在させない（エラーの分析精度が低下する）

**出典**:
- [Sentry Docs: Supabase（Next.js）](https://docs.sentry.io/platforms/javascript/guides/nextjs/configuration/integrations/supabase/) (Sentry公式、`supabaseIntegration({ supabaseClient })`、supabase-js >=2.0.0) ※2026-10-25 確認

**出典引用**:
> "Instruments Supabase queries as spans in your traces so you can see exactly which DB calls are slow"
> ([From vibe code to production-ready: observability for Next.js and Supabase apps](https://blog.sentry.io/nextjs-supabase-observability/), セクション "Instrumenting Next.js and Supabase Edge Functions") ※2026-05-11に実際にfetch成功

> "Mixing Next.js errors with Deno errors and Postgres logs in a single project makes that analysis noisier and less useful."
> ([From vibe code to production-ready: observability for Next.js and Supabase apps](https://blog.sentry.io/nextjs-supabase-observability/), セクション "Instrumenting Next.js and Supabase Edge Functions") ※2026-05-11に実際にfetch成功

**バージョン**: @sentry/nextjs 9+, @supabase/ssr 1+（@supabase/supabase-js 2+）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 9. Vercel Protected Source Maps でブラウザソースマップを認証保護する

本番環境に配置するブラウザソースマップ（`.map` ファイル）を Vercel の認証背後に置き、
チームメンバーのみがアクセスできる状態にする。外部ユーザーへは 404 を返すことで、
ソースマップ経由のコード構造露出を防ぎながら、内部のデバッグ品質を維持する。

**根拠**:
- ブラウザソースマップを公開したまま本番デプロイすると、難読化前のソースコード構造が誰でも参照できる
- Sentry へのアップロード（Rule #4）でデバッグは維持しつつ、`.map` ファイルへの外部アクセスを遮断できる
- Vercel の認証レイヤーを使うことで追加インフラなしに保護を実現できる
- 新規プロジェクトでは既定で有効。既存プロジェクトは Settings → Deployment Protection から有効化でき、再デプロイは不要
- 保護の対象はデプロイから配信される `.map` だけで、Sentry にアップロードした Source Map には影響しない。#4 の Sentry SDK の既定（アップロード後にクライアントの `.map` を削除）のままなら、そもそも配信される `.map` は残らない。ブラウザの DevTools でも本番のソースを読みたいときに、`.map` をデプロイに残す構成と組み合わせる
- チームメンバーがブラウザで見るには Vercel Toolbar でサインインして Debug Mode を有効にする。CI などブラウザ外から取得する場合は Protection Bypass for Automation のヘッダーを使う

**コード例**:
```bash
# REST API で Protected Source Maps を有効化する（ダッシュボードの Settings → Deployment Protection でも可）
curl -X PATCH "https://api.vercel.com/v9/projects/your_project_id_here?teamId=your_team_id_here" \
  -H "Authorization: Bearer $VERCEL_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"protectedSourcemaps": true}'

# 認証なしの取得が 404 になることを確かめる（URL は実際のチャンクの .map に置き換える）
curl -s -o /dev/null -w '%{http_code}\n' "https://your-app.vercel.app/_next/static/chunks/<chunk>.js.map"
```

**出典**:
- [Vercel Docs: Restrict access to production source maps](https://vercel.com/docs/deployment-protection/protected-source-maps) (Vercel 公式、全プランで利用可、`protectedSourcemaps`、未認証は 404、Vercel Toolbar での閲覧) ※2026-10-25 確認
- [Protected Source Maps: Ship browser source maps securely](https://vercel.com/changelog/protected-source-maps-ship-browser-source-maps-securely) (Vercel Changelog、2026-05-14) ※2026-10-25 確認

**出典引用**:
> "Your team can fetch them; everyone else gets a 404."
> ([Protected Source Maps: Ship browser source maps securely](https://vercel.com/changelog/protected-source-maps-ship-browser-source-maps-securely), Vercel Changelog) ※2026-05-16に実際にfetch成功

**バージョン**: Vercel（全プラン）
**確信度**: 高
**最終更新**: 2026-10-25

---

### 10. Sentry の本番コンテキスト（スタックトレース・リクエスト・ブレッドクラム）を AI エージェントに渡して修正精度を上げる

AI エージェントが本番バグを修正するには「実際に何が壊れているか」の情報が必要。
Sentry MCP（Claude Code・Cursor 等の MCP 対応ツール用）または Sentry CLI（CI/CD・スクリプト用）で
スタックトレース・リクエスト情報・ブレッドクラムと、秘密値を除いた実行環境情報（ランタイム・リリース・環境名など）をエージェントのコンテキストに注入する。
渡すのは #3 / #11 のスクラブを通ったあとの Sentry イベントに限り、環境変数の値や生のリクエストログを別途貼り付けない。
修正は Draft PR で開いてレビューを挟み、自動マージは避ける。

**根拠**:
- エージェントはコードベースだけでは本番の実際の壊れ方を把握できない。再現に必要なリクエストパラメータが欠落するため誤診が増える
- エージェントのコンテキストは外部のモデル提供者に送られ、会話ログにも残る。PII やシークレットの扱いは Sentry へ送るときと同じ基準（#3 / #11）にそろえる。スクラブ済みのイベントだけを渡せば、エージェントに渡るデータも Sentry に保存された範囲に収まる
- Sentry の分散トレーシングを使えばマイクロサービスをまたいだ根本原因を追跡できる
- Draft PR ＋ 人間レビューで誤ったマジックフィックスの本番流出を防ぐ

**コード例（Sentry CLI でイシューの文脈と原因分析をエージェントに渡す）**:
```bash
# Sentry CLI（getsentry の `sentry` コマンド）のインストール手順は https://cli.sentry.dev/ を参照
sentry auth login

# イシューの詳細（スタックトレース・最新イベント・トレース）を JSON で取得してエージェントに渡す
sentry issue view CHECKOUT-P1 --json

# Seer AI による根本原因の分析（新しいイシューは数分かかる）
sentry issue explain CHECKOUT-P1

# 自動化スクリプト（CI/CD から呼び出す場合）
# issue list は表形式で出力し、--status はないので、--query で絞って JSON から short ID だけを取り出す
sentry issue list --query "is:unresolved" --limit 5 --json --fields shortId \
  | jq -r '.data[].shortId' \
  | xargs -r sentry issue explain  # -r: 該当なしのときは実行しない（GNU xargs）
```

**MCP 設定例（Claude Code / Cursor 向け）**:
```bash
# Claude Code（Sentry 公式の手順。/mcp/<org>/<project> で対象を絞れる）
claude mcp add --transport http sentry https://mcp.sentry.dev/mcp

# 複数のエージェントにまとめて追加する場合（サードパーティの add-mcp を使う。Sentry ブログで紹介されている）
npx add-mcp https://mcp.sentry.dev/mcp
```

**出典引用**:
> "Your agents need someone/something telling them what's breaking in the wild and giving them the context they need to understand why."
> ([Your agent can't fix what it can't see](https://blog.sentry.io/agents-need-production-context/), セクション "What agents actually need") ※2026-05-26に実際にfetch成功

**出典**:
- [Your agent can't fix what it can't see](https://blog.sentry.io/agents-need-production-context/) (Sentry公式ブログ、`sentry issue view` / `explain` / `plan` と `npx add-mcp` の例) ※2026-10-25 確認
- [Sentry CLI: issue](https://cli.sentry.dev/commands/issue/) (Sentry CLI 公式、`issue list` のフラグ（`--status` はなく `--query` で絞る）、`issue explain` は Seer AI での分析で複数の ID を受け取る、全コマンド共通の `--json` / `--fields`) ※2026-10-25 確認
- [getsentry/toolkit: packages/cli](https://github.com/getsentry/toolkit/tree/main/packages/cli) (Sentry CLI のソース。`issue list --json` は `{ data: [...], hasMore, ... }` の形で出力する) ※2026-10-25 確認
- [Sentry MCP](https://mcp.sentry.dev/) (Sentry公式、`claude mcp add --transport http sentry https://mcp.sentry.dev/mcp/...`) ※2026-10-25 確認
- [add-mcp](https://github.com/neon-solutions/add-mcp) (サードパーティの MCP 設定ツール、`npx add-mcp <url>`) ※2026-10-25 確認

**バージョン**: Sentry SDK 全バージョン（MCP: 2026 年対応 SDK）
**確信度**: 中（方針は Sentry 公式ブログ 1 記事。CLI と MCP のコマンドは公式ドキュメントで確認）
**最終更新**: 2026-10-25

---

### 11. Sentry SDK v11 に上げる前に `dataCollection` へ移行し、収集カテゴリを個別に宣言する

Sentry JavaScript SDK は真偽値 1 個の `sendDefaultPii` を廃止し、カテゴリ単位で収集可否を宣言する `dataCollection` に置き換えた（v10.57.0 で利用可能、v11 でデフォルト）。
問題は移行漏れの向きで、**v11 のデフォルトは「収集する」側に倒れている**。`sendDefaultPii: false` に依存して PII を送っていなかったプロジェクトが v11 に上げると、Cookie・HTTP ボディ・DB クエリ・GenAI の入出力などが黙って送信され始める。
アップグレード前に `dataCollection` を明示的に書き、各カテゴリに `true` / `false` / allow list / deny list のいずれかを宣言すること。
`Sentry.setUser()` 等で手動添付したデータは `dataCollection` の対象外で常に送信される点も踏まえ、PII の入口は SDK 設定と手動添付の両方で塞ぐ。

**根拠**:
- v10 で `sendDefaultPii` を off にしていた場合、v11 のデフォルトでは `userInfo` / `cookies` / `httpBodies` / `databaseQueryData` / `genAI`（inputs・outputs）がいずれも収集側に変わる
- `dataCollection` の各カテゴリは真偽値だけでなく allow list / deny list を取れるため、「ヘッダーは送るが `-ip` を含むものだけ落とす」といった中間的な設定が `beforeSend` を書かずに宣言できる
- 手動で添付したデータは `dataCollection` を迂回して常に送信されるため、SDK 設定だけでは PII 対策は完結しない（本ファイル Rule #3 の `beforeSend` スクラブと併用する）
- スパン側のノイズ・機微情報は `beforeSendSpan` / `ignoreSpans` で別途落とす

**コード例**:
```javascript
// Good: v10 で sendDefaultPii: false だったプロジェクトの v11 移行（同等挙動を明示的に宣言）
Sentry.init({
  dataCollection: {
    userInfo: false,
    cookies: false,
    httpHeaders: {
      request: { deny: ['forwarded', '-ip', 'remote-', 'via', '-user'] },
      response: { deny: ['forwarded', '-ip', 'remote-', 'via', '-user'] },
    },
    httpBodies: [],
    urlQueryParams: { deny: ['forwarded', '-ip', 'remote-', 'via', '-user'] },
    genAI: { inputs: false, outputs: false },
    databaseQueryData: false,
    graphQL: { document: false, variables: false },
  },
});

// Good: v10 で sendDefaultPii: true だった場合は v11 のデフォルトと同じなので設定不要
Sentry.init({});

// Bad: sendDefaultPii: false のまま v11 に上げる（オプションが無くなり、収集側のデフォルトが適用される）
Sentry.init({ sendDefaultPii: false });
```
```javascript
// スパンに載る機微情報は beforeSendSpan / ignoreSpans で落とす
Sentry.init({
  dsn: 'example.com',
  beforeSendSpan: (span) => {
    if (span.attributes?.['sentry.op'] === 'db.query') {
      span.name = '[filtered]';
    }
    return span;
  },
  ignoreSpans: [
    'healthcheck',
    { name: /^GET \//, attributes: { 'http.route': '/api/status' } },
  ],
});
```

**アンチパターン**:
- `sendDefaultPii` の指定を残したまま v11 に上げ、「設定してあるから大丈夫」と判断する
- `dataCollection` を真偽値だけで運用し、`httpHeaders` / `urlQueryParams` の deny list を使わずに `beforeSend` で全部書き直す
- PII 対策を `dataCollection` だけに寄せ、`Sentry.setUser()` 等の手動添付を棚卸ししない

**出典引用**:
> "PII (or Personally Identifiable Information) is anything tied to a person: a user ID, email, username, name. Sensitive data is credentials and secrets"
> ([From one switch to a control panel: meet `dataCollection`](https://blog.sentry.io/datacollection-control-panel/), セクション "PII vs. sensitive data") ※2026-08-28に実際にfetch成功

> "You can pass true, false, an allow list, or a deny list"
> ([From one switch to a control panel: meet `dataCollection`](https://blog.sentry.io/datacollection-control-panel/), セクション "Granularity for what you really need") ※2026-08-28に実際にfetch成功

**バージョン**: Sentry JavaScript SDK 10.57.0+（v11 でデフォルト）
**確信度**: 高（Sentry 公式ブログ — パターン1）
**最終更新**: 2026-08-28

---

## 関連プラクティス

- [`architecture/error-handling.md`](../architecture/error-handling.md) - Next.js エラー境界の設計
- [`observability/logging.md`](./logging.md) - 構造化ロギング
- [`api-client/error-handling.md`](../api-client/error-handling.md) - API エラーの分類と処理
