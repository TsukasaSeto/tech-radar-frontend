# gRPC / Connect クライアントのベストプラクティス

## ルール

### 1. gRPC-Web より Connect-Web を優先する

ブラウザからの RPC 通信には、従来の gRPC-Web プロキシを必要とする gRPC-Web より
Connect-Web を優先する。

**根拠**:
- Connect-Web は標準 HTTP/1.1 と HTTP/2 の両方で動作し、Envoy 等のプロキシが不要
- Connect プロトコルは gRPC および gRPC-Web と相互運用可能で既存のサーバーとも接続できる
- `Content-Type: application/connect+proto` または `application/connect+json` でシンプルに通信
- バンドルサイズが小さく、Fetch API ベースで動作するためポリフィル不要

**コード例**:
```bash
# Buf CLI で proto から TypeScript コードを生成
npm install --save-dev @bufbuild/buf @bufbuild/protoc-gen-es @connectrpc/protoc-gen-connect-es

# buf.gen.yaml
version: v2
plugins:
  - local: protoc-gen-es
    out: src/gen
    opt: target=ts
  - local: protoc-gen-connect-es
    out: src/gen
    opt: target=ts
```

```ts
// src/gen/user/v1/user_connect.ts (生成済みファイル)
// src/lib/connect-client.ts
import { createClient } from '@connectrpc/connect';
import { createConnectTransport } from '@connectrpc/connect-web';
import { UserService } from '@/gen/user/v1/user_connect';

const transport = createConnectTransport({
  baseUrl: process.env.NEXT_PUBLIC_API_BASE_URL!,
});

export const userClient = createClient(UserService, transport);

// コンポーネントでの使用
async function fetchUser(id: string): Promise<User> {
  const response = await userClient.getUser({ id });
  return response.user!;
}

// Bad: gRPC-Web（プロキシが必要）
import { grpc } from '@improbable-eng/grpc-web';
// Envoy/Nginx プロキシの設定が別途必要
```

**出典**:
- [Connect-Web Docs](https://connectrpc.com/docs/web/getting-started) (connectrpc.com公式)
- [Buf Docs](https://buf.build/docs/generate/overview) (buf.build公式)

**バージョン**: @connectrpc/connect-web 1.0+, @bufbuild/protobuf 1.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. Buf で proto ファイルを管理し型生成を自動化する

Protocol Buffers の管理と型生成には Buf CLI を使う。
proto ファイルの lint・フォーマット・生成を統一されたツールチェーンで管理する。

**根拠**:
- `buf lint` でプロトコル定義のスタイル違反を CI で検出できる
- `buf generate` でクライアントコードの生成が1コマンドで完結する
- Buf Schema Registry (BSR) でプロトファイルをチームで共有できる
- `buf breaking` でAPIの後方互換性を CI で検証できる

**コード例**:
```yaml
# buf.yaml
version: v2
modules:
  - path: proto
lint:
  use:
    - STANDARD
breaking:
  use:
    - FILE
```

```bash
# proto ファイルの lint
npx buf lint

# 後方互換性チェック（main ブランチとの比較）
npx buf breaking --against '.git#branch=main'

# TypeScript コード生成
npx buf generate

# package.json scripts
{
  "scripts": {
    "proto:generate": "buf generate",
    "proto:lint": "buf lint",
    "proto:breaking": "buf breaking --against '.git#branch=main'"
  }
}
```

```ts
// 生成されたクライアントは完全に型安全
const client = createClient(UserService, transport);

// コンパイル時に proto の定義と一致しないと型エラー
const res = await client.createUser({
  name: 'Alice',
  email: 'alice@example.com',
  // age: 'invalid'  // エラー: number が期待される
});
```

**出典**:
- [Buf CLI Docs](https://buf.build/docs/cli/installation) (buf.build公式)

**バージョン**: Buf CLI 1.0+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. ストリーミング RPC はサーバーサイドで処理し、クライアントには SSE か WebSocket を使う

ブラウザで gRPC サーバーストリーミングを使う場合、Connect の `transport` を正しく設定するか、
Next.js の Route Handler で変換して SSE として提供する。

**根拠**:
- HTTP/1.1 ではストリーミング gRPC が動作しないためフォールバックが必要
- Next.js Route Handler で ReadableStream を返すことで SSE として提供できる
- Connect の `createGrpcWebTransport` は HTTP/1.1 でのストリーミングをサポート

**コード例**:
```ts
// app/api/stream/route.ts - Connect サーバーストリーミングを SSE に変換
import { createClient } from '@connectrpc/connect';
import { createGrpcTransport } from '@connectrpc/connect-node';
import { NotificationService } from '@/gen/notification/v1/notification_connect';

const transport = createGrpcTransport({
  baseUrl: process.env.INTERNAL_API_URL!,
  httpVersion: '2',
});

export async function GET(request: Request) {
  const client = createClient(NotificationService, transport);
  const encoder = new TextEncoder();

  const stream = new ReadableStream({
    async start(controller) {
      for await (const notification of client.streamNotifications({})) {
        const data = `data: ${JSON.stringify(notification)}\n\n`;
        controller.enqueue(encoder.encode(data));
      }
      controller.close();
    },
    cancel() {
      // クライアントが接続を切った場合の後処理
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      Connection: 'keep-alive',
    },
  });
}

// クライアント側での SSE 購読
function NotificationBell() {
  useEffect(() => {
    const eventSource = new EventSource('/api/stream');
    eventSource.onmessage = (e) => {
      const notification = JSON.parse(e.data);
      showNotification(notification);
    };
    return () => eventSource.close();
  }, []);
}
```

**出典**:
- [Connect Docs: Streaming](https://connectrpc.com/docs/web/streaming) (connectrpc.com公式)

**バージョン**: @connectrpc/connect-web 1.0+, Next.js 13+
**確信度**: 中
**最終更新**: 2026-05-05

---

## 関連プラクティス

- [`api-client/type-safety.md`](./type-safety.md) - スキーマ駆動型安全性
- [`api-client/error-handling.md`](./error-handling.md) - RPC エラーの処理
