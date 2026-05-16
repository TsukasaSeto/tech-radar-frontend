# API エラー処理のベストプラクティス

> **層責任**: このファイルは **HTTP / API 層**（4xx・5xx 分類、リトライ判定、認証エラー処理、トランスポートエラー）を扱う。
> - **UI 層**（Error Boundary、`error.tsx`、Result 型、回復 UI）→ [`architecture/error-handling.md`](../architecture/error-handling.md)
> - **観測 / 集計層**（Sentry キャプチャ、ノイズ削減、PII スクラブ）→ [`observability/error-tracking.md`](../observability/error-tracking.md)
>
> 流れは `HTTP エラー検知（ここ）→ 分類 → 回復 UI → ログ・Sentry 通知` の方向。

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

**HTTP ステータスコード別リトライ判定表**:

クライアント側で「リトライしてよいか」を一義的に判断するための表。
リトライの可否は **冪等性** と **エラーの一時性** の 2 軸で決まる。

| ステータス | 意味 | 冪等メソッド (GET/HEAD/PUT/DELETE) | 非冪等メソッド (POST/PATCH) |
|---|---|---|---|
| **408** Request Timeout | クライアント送信が間に合わなかった | ✅ 再試行 | ⚠️ Idempotency-Key 必須 |
| **425** Too Early | early data 拒否 | ✅ 再試行 | ✅ 再試行 |
| **429** Too Many Requests | レート制限 | ✅ `Retry-After` ヘッダー尊重 | ✅ `Retry-After` ヘッダー尊重 |
| **500** Internal Server Error | サーバー側エラー | ✅ 再試行（1〜3 回） | ❌ サーバー側で処理済の可能性あり、冪等性キー必須 |
| **502** Bad Gateway | 上流サーバー応答異常 | ✅ 再試行 | ⚠️ Idempotency-Key 必須 |
| **503** Service Unavailable | 一時的障害 | ✅ `Retry-After` 尊重 | ✅ `Retry-After` 尊重 |
| **504** Gateway Timeout | 上流タイムアウト | ✅ 再試行 | ⚠️ サーバー側到達確認後に再試行 |
| **400** Bad Request | リクエスト不正 | ❌ リトライ不可 | ❌ リトライ不可 |
| **401** Unauthorized | 認証失敗 | ⚠️ トークンリフレッシュ後 1 回のみ | ⚠️ トークンリフレッシュ後 1 回のみ |
| **403** Forbidden | 認可失敗 | ❌ リトライ不可 | ❌ リトライ不可 |
| **404** Not Found | リソース不在 | ❌ リトライ不可 | ❌ リトライ不可 |
| **409** Conflict | 楽観ロック競合等 | ⚠️ 最新版を取得して再試行（ロジック側で判断） | ⚠️ 最新版を取得して再試行 |
| **410** Gone | 削除済み | ❌ リトライ不可 | ❌ リトライ不可 |
| **422** Unprocessable Entity | バリデーション失敗 | ❌ リトライ不可 | ❌ リトライ不可 |
| Network error | DNS/TCP/TLS 失敗 | ✅ 再試行 | ⚠️ Idempotency-Key 必須 |

凡例: ✅ リトライしてよい / ⚠️ 条件付き / ❌ リトライ不可

**冪等性チェックリスト（非冪等メソッドをリトライする前に確認）**:

1. **`Idempotency-Key` ヘッダーをサーバーが受け付けるか？**
   - Stripe / Square / 主要決済 API は対応。重複リクエストを検出してくれる
   - 自社 API なら同等の仕組みを実装する
2. **クライアント側で `crypto.randomUUID()` を生成しているか？**
   - リクエスト ID を Mutation 単位で固定（UI 上の 1 操作 = 1 ID）
   - リトライしても同じ ID を再送する
3. **副作用が外部に伝播する操作か？**
   - メール送信、課金、外部 webhook → 冪等性キー必須
   - 内部 DB 更新のみ → サーバー側で重複排除しているなら緩めて良い
4. **タイムアウト後の状態が判明しているか？**
   - 504 後に「リソース作成済みか」を GET で確認してから再試行する設計が安全

**コード例（Idempotency-Key）**:
```ts
import ky from 'ky';

async function purchase(item: Item, quantity: number) {
  const idempotencyKey = crypto.randomUUID(); // 1 操作 = 1 ID

  return ky.post('/api/purchase', {
    json: { itemId: item.id, quantity },
    headers: { 'Idempotency-Key': idempotencyKey },
    retry: {
      limit: 3,
      methods: ['post'], // 通常 ky は POST をリトライしないが、Idempotency-Key 付きなら可
      statusCodes: [408, 425, 429, 500, 502, 503, 504],
    },
  }).json();
}
```

**`Retry-After` ヘッダーの尊重**:

サーバーが `Retry-After: 30`（秒）または HTTP-date 形式で返したら、それを尊重する。
固定指数バックオフよりサーバーが指定した値を優先することで、レート制限解除を最短で待てる。

```ts
function parseRetryAfter(header: string | null): number | null {
  if (!header) return null;
  const seconds = Number(header);
  if (!Number.isNaN(seconds)) return seconds * 1000;
  const date = Date.parse(header);
  if (!Number.isNaN(date)) return Math.max(0, date - Date.now());
  return null;
}

// retry callback で Retry-After を尊重
ky.create({
  retry: {
    limit: 5,
    statusCodes: [429, 503],
    delay: (attempt) => Math.min(0.3 * 2 ** (attempt - 1) * 1000, 3000),
  },
  hooks: {
    afterResponse: [
      async (req, opts, res) => {
        const retryAfter = parseRetryAfter(res.headers.get('Retry-After'));
        if (retryAfter && (res.status === 429 || res.status === 503)) {
          await new Promise(r => setTimeout(r, retryAfter));
        }
        return res;
      },
    ],
  },
});
```

**出典**:
- [AWS: Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) (AWS Architecture Blog / 2015)
- [ky README: retry](https://github.com/sindresorhus/ky#retry) (sindresorhus/ky / GitHub)
- [RFC 9110: HTTP Semantics - Retry-After](https://www.rfc-editor.org/rfc/rfc9110.html#field.retry-after) (IETF)
- [RFC 9110: HTTP Semantics - Idempotent Methods](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.2.2) (IETF)
- [Stripe API: Idempotent Requests](https://docs.stripe.com/api/idempotent_requests) (Stripe Docs)

**バージョン**: ky 1.0+
**確信度**: 高
**最終更新**: 2026-05-05 / 補強 2026-05-16

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

### 5. HTTPステータスコード別の回復戦略を実装する

HTTP エラーを「種別ごとに異なる回復アクション」にマッピングし、
ユーザー体験を最大化する。401は認証リダイレクト、429は指数バックオフ、
5xx系はサーキットブレーカーで過負荷時の障害拡大を防ぐ。

**根拠**:
- エラーごとに適切な回復戦略が異なり、一律の処理ではUXが低下する
- 401は即座に認証フローへ誘導することでユーザーが迷わない
- 429のバックオフはサーバー過負荷を悪化させないために必須
- サーキットブレーカーは5xxが連続する障害時にリクエストを遮断し、システム全体の回復を助ける

**コード例**:
```ts
// lib/recovery-strategy.ts

// サーキットブレーカーの状態
type CircuitState = 'closed' | 'open' | 'half-open';

class CircuitBreaker {
  private state: CircuitState = 'closed';
  private failureCount = 0;
  private lastFailureTime?: number;

  constructor(
    private readonly threshold = 5,       // 失敗N回でopen
    private readonly resetTimeout = 30_000, // 30秒後にhalf-openへ
  ) {}

  isOpen(): boolean {
    if (this.state === 'open') {
      const elapsed = Date.now() - (this.lastFailureTime ?? 0);
      if (elapsed > this.resetTimeout) {
        this.state = 'half-open';  // 復旧試行へ
        return false;
      }
      return true;
    }
    return false;
  }

  recordSuccess() {
    this.failureCount = 0;
    this.state = 'closed';
  }

  recordFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    if (this.failureCount >= this.threshold) {
      this.state = 'open';
    }
  }
}

const breaker = new CircuitBreaker();

// ステータスコード別の回復戦略
async function fetchWithRecovery<T>(url: string, options?: RequestInit): Promise<T> {
  // サーキットブレーカーが開いている場合はリクエストを遮断
  if (breaker.isOpen()) {
    throw new Error('Service temporarily unavailable (circuit open)');
  }

  try {
    const res = await fetch(url, options);

    if (res.ok) {
      breaker.recordSuccess();
      return res.json() as Promise<T>;
    }

    switch (true) {
      // 401: 認証リダイレクト
      case res.status === 401: {
        await refreshTokenOrRedirect();  // トークン更新を試み、失敗時は /login へ
        throw new ApiError(401, 'UNAUTHORIZED', 'Session expired');
      }

      // 429: Retry-After を尊重した指数バックオフ
      case res.status === 429: {
        const retryAfter = res.headers.get('Retry-After');
        const waitMs = retryAfter
          ? Number(retryAfter) * 1000
          : Math.min(1000 * 2 ** retryCount, 32_000);  // 最大32秒
        await sleep(waitMs + Math.random() * 1000);      // ジッター加算
        return fetchWithRecovery(url, options);           // 再試行
      }

      // 5xx: サーキットブレーカーに記録
      case res.status >= 500: {
        breaker.recordFailure();
        throw new ApiError(res.status, 'SERVER_ERROR', `HTTP ${res.status}`);
      }

      default:
        throw new ApiError(res.status, 'CLIENT_ERROR', `HTTP ${res.status}`);
    }
  } catch (err) {
    if (err instanceof TypeError) {
      // ネットワークエラーもサーキットブレーカーに記録
      breaker.recordFailure();
    }
    throw err;
  }
}

// TanStack Query との統合例
const { data } = useQuery({
  queryKey: ['users'],
  queryFn: () => fetchWithRecovery<User[]>('/api/users'),
  retry: (failureCount, error) => {
    if (error instanceof ApiError && error.status === 401) return false;  // 401はリトライしない
    if (error instanceof ApiError && error.status === 429) return failureCount < 3;
    return failureCount < 2;
  },
});
```

**出典**:
- [Martin Fowler: Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html) (martinfowler.com)
- [AWS: Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) (AWS Architecture Blog)
- [RFC 6585: 429 Too Many Requests](https://www.rfc-editor.org/rfc/rfc6585) (IETF)

**バージョン**: 全バージョン（パターン）
**確信度**: 高
**最終更新**: 2026-05-06

---

## 関連プラクティス

- [`architecture/error-handling.md`](../architecture/error-handling.md) - アプリ全体のエラー処理
- [`api-client/rest.md`](./rest.md) - REST クライアントのリトライ設定
- [`observability/error-tracking.md`](../observability/error-tracking.md) - エラーの監視・追跡
