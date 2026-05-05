# API エラー処理のベストプラクティス

## ルール

### 1. HTTP ステータスコードを分類してエラー種別を判定する

HTTP レスポンスのステータスコードを 4xx（クライアントエラー）と 5xx（サーバーエラー）に
分類し、それぞれに適した処理を行う。

**根拠**:
- 4xx はクライアント起因（バリデーションエラー、認証エラー等）でユーザーへの表示が必要
- 5xx はサーバー起因でリトライやフォールバックが有効なケースが多い
- 429 Too Many Requests はレートリミットのため `Retry-After` ヘッダーを尊重すべき
- エラー種別を統一した型で表現することでエラー処理が一貫する

**コード例**:
```ts
// lib/api-error.ts
export class ApiError extends Error {
  constructor(
    public readonly status: number,
    public readonly code: string,
    message: string,
    public readonly retryAfter?: number,
  ) {
    super(message);
    this.name = 'ApiError';
  }

  get isClientError() { return this.status >= 400 && this.status < 500; }
  get isServerError() { return this.status >= 500; }
  get isUnauthorized() { return this.status === 401; }
  get isForbidden() { return this.status === 403; }
  get isNotFound() { return this.status === 404; }
  get isRateLimited() { return this.status === 429; }
}

// lib/http.ts - エラーの分類と変換
async function handleResponse<T>(response: Response): Promise<T> {
  if (response.ok) return response.json() as Promise<T>;

  const body = await response.json().catch(() => ({}));
  const retryAfter = response.headers.get('Retry-After');

  throw new ApiError(
    response.status,
    (body as { code?: string }).code ?? 'UNKNOWN',
    (body as { message?: string }).message ?? `HTTP ${response.status}`,
    retryAfter ? Number(retryAfter) : undefined,
  );
}

// コンポーネントでのエラー処理
async function handleSubmit(data: FormData) {
  try {
    await createUser(data);
  } catch (error) {
    if (error instanceof ApiError) {
      if (error.isUnauthorized) {
        redirectToLogin();
      } else if (error.status === 422) {
        showValidationErrors(error);
      } else if (error.isRateLimited) {
        showMessage(`${error.retryAfter}秒後に再試行してください`);
      } else if (error.isServerError) {
        showMessage('サーバーエラーが発生しました。しばらく経ってから再試行してください。');
      }
    }
  }
}
```

**出典**:
- [MDN: HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status) (MDN Web Docs)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110) (IETF)

**バージョン**: 全バージョン（HTTP仕様）
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. ネットワークエラーとサーバーエラーを区別して処理する

`fetch` が reject されるネットワークエラー（オフライン、DNS失敗等）と、
レスポンスが返る HTTP エラーを別々に扱う。

**根拠**:
- ネットワークエラーはユーザーへ「接続を確認してください」のメッセージが適切
- サーバーエラーは「しばらくしてから再試行」が適切
- それぞれリトライ可能かどうかの判断が異なる
- `navigator.onLine` と組み合わせてオフライン検知を改善できる

**コード例**:
```ts
// lib/fetch-with-error-handling.ts
type NetworkError = { type: 'network'; message: string };
type HttpError = { type: 'http'; status: number; message: string };
type FetchError = NetworkError | HttpError;

async function safeFetch<T>(
  url: string,
  options?: RequestInit,
): Promise<{ data: T; error: null } | { data: null; error: FetchError }> {
  try {
    const res = await fetch(url, options);
    if (!res.ok) {
      const body = await res.json().catch(() => ({}));
      return {
        data: null,
        error: {
          type: 'http',
          status: res.status,
          message: (body as { message?: string }).message ?? `HTTP ${res.status}`,
        },
      };
    }
    return { data: await res.json() as T, error: null };
  } catch (err) {
    // TypeError: Failed to fetch → ネットワークエラー
    return {
      data: null,
      error: {
        type: 'network',
        message: navigator.onLine
          ? 'リクエストがタイムアウトしました'
          : 'インターネット接続を確認してください',
      },
    };
  }
}

// 使用例
const { data, error } = await safeFetch<User>('/api/users/123');
if (error) {
  if (error.type === 'network') {
    showOfflineMessage(error.message);
  } else {
    showHttpError(error.status, error.message);
  }
}
```

**出典**:
- [MDN: TypeError: Failed to fetch](https://developer.mozilla.org/en-US/docs/Web/API/fetch#exceptions) (MDN Web Docs)

**バージョン**: 全モダンブラウザ
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. 指数バックオフ＋ジッターでリトライ戦略を実装する

サーバーエラー（5xx）やネットワークエラーのリトライには、
固定間隔ではなく指数バックオフ＋ランダムジッターを使う。

**根拠**:
- 固定間隔のリトライは多数のクライアントが同時にサーバーを叩く「Thundering Herd」問題を引き起こす
- ジッターを加えることでリトライのタイミングを分散させ、サーバーへの負荷集中を防ぐ
- ky の `retry` オプションは指数バックオフとジッターを組み込みでサポートしている
- べき等でない操作（POST等）はリトライしない

**コード例**:
```ts
// ky を使ったリトライ設定（推奨）
import ky from 'ky';

const http = ky.create({
  retry: {
    limit: 3,
    methods: ['get', 'head'],  // べき等なメソッドのみ
    statusCodes: [408, 429, 500, 502, 503, 504],
    backoffLimit: 3000,  // 最大待機時間 3秒
    delay: attemptCount => 0.3 * 2 ** (attemptCount - 1) * 1000,  // 300ms, 600ms, 1200ms
  },
});

// 手動実装（ky が使えない場合）
async function fetchWithRetry<T>(
  url: string,
  options: RequestInit = {},
  maxRetries = 3,
): Promise<T> {
  let lastError: Error;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const res = await fetch(url, options);
      if (res.ok) return res.json() as Promise<T>;

      // 4xx はリトライしない（クライアント起因）
      if (res.status >= 400 && res.status < 500 && res.status !== 429) {
        throw new Error(`HTTP ${res.status}`);
      }

      lastError = new Error(`HTTP ${res.status}`);
    } catch (err) {
      lastError = err instanceof Error ? err : new Error(String(err));
    }

    if (attempt < maxRetries) {
      // 指数バックオフ＋ジッター
      const baseDelay = Math.min(1000 * 2 ** attempt, 10000);
      const jitter = Math.random() * baseDelay * 0.1;
      await new Promise(resolve => setTimeout(resolve, baseDelay + jitter));
    }
  }

  throw lastError!;
}

// Bad: 固定間隔リトライ（Thundering Herd の原因）
for (let i = 0; i < 3; i++) {
  await new Promise(resolve => setTimeout(resolve, 1000));  // 全クライアントが同時にリトライ
  try { return await fetch(url); } catch {}
}
```

**出典**:
- [AWS: Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) (AWS Architecture Blog / 2015)
- [ky README: retry](https://github.com/sindresorhus/ky#retry) (sindresorhus/ky / GitHub)

**バージョン**: ky 1.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. GraphQL エラーは HTTP 200 でも発生することを考慮する

GraphQL API は HTTP 200 でレスポンスを返しつつ、
`errors` フィールドにエラーを含めることがある。HTTP ステータスだけで成否を判定しない。

**根拠**:
- GraphQL 仕様では部分成功（`data` と `errors` が共存）が許容される
- `response.ok` が true でも `data.errors` が存在する場合がある
- urql / Apollo は `errors` フィールドを自動的に処理するが、素の fetch では手動チェックが必要

**コード例**:
```ts
// GraphQL レスポンスの型
interface GraphQLResponse<T> {
  data?: T;
  errors?: Array<{ message: string; locations?: unknown[]; path?: unknown[] }>;
}

// 素の fetch で GraphQL を呼ぶ場合
async function graphqlFetch<T>(
  query: string,
  variables?: Record<string, unknown>,
): Promise<T> {
  const res = await fetch('/api/graphql', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query, variables }),
  });

  const json = await res.json() as GraphQLResponse<T>;

  // HTTP エラーチェック
  if (!res.ok) throw new Error(`HTTP ${res.status}`);

  // GraphQL エラーチェック（HTTP 200 でも発生する）
  if (json.errors?.length) {
    throw new Error(json.errors.map(e => e.message).join(', '));
  }

  if (!json.data) throw new Error('No data returned');
  return json.data;
}

// urql / Apollo を使えば自動的に処理される
const [{ data, error }] = useQuery({ query: MyQuery });
//                  ^^^^^ GraphQL errors も error に含まれる
```

**出典**:
- [GraphQL Spec: Errors](https://spec.graphql.org/October2021/#sec-Errors) (GraphQL仕様)

**バージョン**: GraphQL 16+
**確信度**: 高
**最終更新**: 2026-05-05

---

## 関連プラクティス

- [`architecture/error-handling.md`](../architecture/error-handling.md) - アプリ全体のエラー処理
- [`api-client/rest.md`](./rest.md) - REST クライアントのリトライ設定
- [`observability/error-tracking.md`](../observability/error-tracking.md) - エラーの監視・追跡
