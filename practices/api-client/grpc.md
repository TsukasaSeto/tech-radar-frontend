# gRPC / Connect クライアントのベストプラクティス

## ルール

### 1. gRPC-Web より Connect-Web を優先する

ブラウザからの RPC 通信には、従来の gRPC-Web プロキシを必要とする gRPC-Web より
Connect-Web を優先する。

**根拠**:
- サーバーが Connect プロトコルか gRPC-Web に対応していれば、Envoy 等のプロキシなしでブラウザから直接呼べる（Connect for Node / connect-go は両方に対応）。素の gRPC サーバーには、Envoy などの gRPC-Web への変換層が別途必要
- `@connectrpc/connect-web` は `createConnectTransport()`（Connect プロトコル）と `createGrpcWebTransport()`（gRPC-Web）を持つので、サーバーの対応プロトコルに合わせて transport を選ぶ
- `Content-Type: application/connect+proto` または `application/connect+json` でシンプルに通信
- バンドルサイズが小さく、Fetch API ベースで動作するためポリフィル不要
- v2 では `protoc-gen-connect-es` が廃止され、サービス定義も `protoc-gen-es` が `*_pb.ts` に生成する。v1 用プラグインを混ぜると生成できないので、パッケージは v2 にそろえる

**コード例**:
```bash
# 生成ツール（v2 は protoc-gen-es だけでよい）
npm install --save-dev @bufbuild/buf @bufbuild/protoc-gen-es
# ランタイム
npm install @connectrpc/connect @connectrpc/connect-web @bufbuild/protobuf
```

```yaml
# buf.gen.yaml
version: v2
plugins:
  - local: protoc-gen-es
    out: src/gen
    opt: target=ts
    include_imports: true
```

```ts
// src/lib/connect-client.ts
import { createClient } from '@connectrpc/connect';
import { createConnectTransport } from '@connectrpc/connect-web';
import { UserService, type User } from '@/gen/user/v1/user_pb'; // v2 はサービスも *_pb.ts に生成される

// サーバーが Connect プロトコルに対応している場合
const transport = createConnectTransport({
  baseUrl: 'https://api.example.com',
});
// サーバーが gRPC-Web にだけ対応している場合は createGrpcWebTransport({ baseUrl }) を使う

export const userClient = createClient(UserService, transport);

export async function fetchUser(id: string): Promise<User> {
  const response = await userClient.getUser({ id });
  if (!response.user) throw new Error('user がレスポンスに含まれていません');
  return response.user;
}
```

```ts
// Bad: 素の gRPC サーバーに旧来の gRPC-Web クライアントでつなぐ
import { grpc } from '@improbable-eng/grpc-web';
// Envoy/Nginx プロキシの設定が別途必要
```

**出典**:
- [Connect-Web Docs](https://connectrpc.com/docs/web/getting-started) (connectrpc.com公式)
- [Connect for Web: Generating code](https://connectrpc.com/docs/web/generating-code) (connectrpc.com公式、v2 の依存パッケージと buf.gen.yaml、`*_pb.ts` からの import) ※2026-10-25 確認
- [Connect for Web: Choosing a protocol](https://connectrpc.com/docs/web/choosing-a-protocol) (connectrpc.com公式、transport の種類とサーバー側の対応要件) ※2026-10-25 確認
- [connect-es MIGRATING.md](https://github.com/connectrpc/connect-es/blob/main/MIGRATING.md) (Connect 公式リポジトリ、v1 → v2 で `protoc-gen-connect-es` 廃止・`*_connect` → `*_pb`) ※2026-10-25 確認
- [Buf Docs](https://buf.build/docs/generate/overview) (buf.build公式)

**バージョン**: @connectrpc/connect 2.x, @connectrpc/connect-web 2.x, @bufbuild/protobuf 2.x, @bufbuild/protoc-gen-es 2.x（v1 系は `protoc-gen-connect-es` と `*_connect` からの import を使う）
**確信度**: 高
**最終更新**: 2026-10-25

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

### 3. ストリーミング RPC は、ブラウザから直接受けるか Route Handler で SSE に変換するかを条件で選ぶ

ブラウザで gRPC サーバーストリーミングを受けるときは、次の条件で方式を選ぶ。

- サーバーが Connect プロトコルか gRPC-Web に対応していて、ブラウザから直接到達できる → `@connectrpc/connect-web` の transport でサーバーストリーミングを直接受ける
- サーバーが素の gRPC にしか対応していない、内部ネットワークにしかない、または認証情報をブラウザに渡したくない → Next.js の Route Handler（Node.js ランタイム）で gRPC を受け、SSE に変換して提供する

**根拠**:
- 素の gRPC サーバーにはブラウザから直接つなげず、gRPC-Web への変換層（Envoy など）かサーバー側での中継が要る
- Connect プロトコルはすべての種類のストリーミング RPC をサポートするが、ブラウザではクライアントストリーミングに制約がある。ブラウザから直接使うのはサーバーストリーミングまでにとどめる
- Next.js Route Handler で ReadableStream を返すことで SSE として提供できる
- SSE に変換する場合は、ブラウザが切断したら上流の RPC も止める。`request.signal` と `ReadableStream` の `cancel()` の両方から AbortController を abort し、その `signal` を RPC の呼び出しオプションに渡す（キャンセルの方針は `practices/api-client/rest.md` #1）。止めないと、上流のストリームが残り続ける

**コード例**:
```ts
// lib/notification-client.ts - ブラウザから直接受ける場合（サーバーが Connect か gRPC-Web に対応）
import { createClient } from '@connectrpc/connect';
import { createConnectTransport } from '@connectrpc/connect-web';
import { NotificationService, type Notification } from '@/gen/notification/v1/notification_pb';

const client = createClient(
  NotificationService,
  createConnectTransport({ baseUrl: 'https://api.example.com' }),
);

export async function subscribeNotifications(
  signal: AbortSignal,
  onNotification: (notification: Notification) => void,
) {
  for await (const notification of client.streamNotifications({}, { signal })) {
    onNotification(notification);
  }
}
```

```ts
// app/api/stream/route.ts - 素の gRPC サーバーストリーミングを SSE に変換
import { toJson } from '@bufbuild/protobuf';
import { Code, ConnectError, createClient } from '@connectrpc/connect';
import { createGrpcTransport } from '@connectrpc/connect-node';
import { NotificationService, NotificationSchema } from '@/gen/notification/v1/notification_pb';

// v2 の gRPC transport は HTTP/2 固定（httpVersion オプションはない）
const transport = createGrpcTransport({
  baseUrl: process.env.INTERNAL_API_URL!,
});

const client = createClient(NotificationService, transport);

export async function GET(request: Request) {
  const encoder = new TextEncoder();
  const abort = new AbortController();
  request.signal.addEventListener('abort', () => abort.abort());

  const stream = new ReadableStream({
    async start(controller) {
      try {
        for await (const notification of client.streamNotifications({}, { signal: abort.signal })) {
          // v2 のメッセージは bigint を含みうるので JSON.stringify ではなく toJson を使う
          const data = `data: ${JSON.stringify(toJson(NotificationSchema, notification))}\n\n`;
          controller.enqueue(encoder.encode(data));
        }
        controller.close();
      } catch (err) {
        // 切断による中断は正常終了として扱う
        if (err instanceof ConnectError && err.code === Code.Canceled) return;
        controller.error(err);
      }
    },
    cancel() {
      abort.abort();
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
```

```tsx
// components/notification-bell.tsx - クライアント側での SSE 購読
'use client';

import { useEffect } from 'react';
import { showNotification } from '@/lib/notifications'; // アプリ側で用意する表示関数

export function NotificationBell() {
  useEffect(() => {
    const eventSource = new EventSource('/api/stream');
    eventSource.onmessage = (e) => {
      const notification = JSON.parse(e.data);
      showNotification(notification);
    };
    return () => eventSource.close();
  }, []);

  return null;
}
```

**出典**:
- [Connect for Web: Using clients](https://connectrpc.com/docs/web/using-clients) (connectrpc.com公式、サーバーストリーミングを `for await` で受ける) ※2026-10-25 確認
- [Connect for Web: Cancellation & Timeouts](https://connectrpc.com/docs/web/cancellation-and-timeouts/) (connectrpc.com公式、`{ signal }` で中断すると `Code.Canceled` の `ConnectError`) ※2026-10-25 確認
- [Connect FAQ](https://connectrpc.com/docs/faq/) (connectrpc.com公式、ブラウザのクライアントストリーミングの制約) ※2026-10-25 確認
- [Connect for Web: Choosing a protocol](https://connectrpc.com/docs/web/choosing-a-protocol) (connectrpc.com公式、素の gRPC には Envoy などの変換層が必要) ※2026-10-25 確認
- [connect-es MIGRATING.md](https://github.com/connectrpc/connect-es/blob/main/MIGRATING.md) (Connect 公式リポジトリ、v2 の gRPC transport は HTTP/2 必須で `httpVersion` を削除) ※2026-10-25 確認

**バージョン**: @connectrpc/connect 2.x, @connectrpc/connect-web 2.x, @connectrpc/connect-node 2.x, Next.js 13+
**確信度**: 中
**最終更新**: 2026-10-25

---

## 関連プラクティス

- [`api-client/type-safety.md`](./type-safety.md) - スキーマ駆動型安全性
- [`api-client/error-handling.md`](./error-handling.md) - RPC エラーの処理
