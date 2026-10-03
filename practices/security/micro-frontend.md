# マイクロフロントエンドのセキュリティ

> **役割分担**: Module Federation / iframe / Web Components / 遅延ロードするリモート等で複数チームの UI を1画面に合成する構成の、**実行時の信頼境界**を扱う。単一アプリ内の CSP・XSS・トークン保存は `csp.md` / `xss.md` / `auth-token-storage.md`、モジュール依存の方向は `architecture/module-boundary.md` を参照。

## ルール

### 1. マイクロフロントエンドの合成方式ごとに信頼境界を判断し、リポジトリ分割やスコープ付きスタイルを「境界」とみなさない

リポジトリやデプロイチームが分かれていても、ブラウザ上のセキュリティ境界にはならない。同一オリジン制約が分離するのは機能単位ではなくオリジン単位である。
合成方式を選ぶ前に、各マイクロフロントエンドの**オリジン・デプロイ担当・必要なデータアクセス**を文書化し、方式ごとの信頼の前提を決める。

| 合成方式 | セキュリティ上の判断 |
|---|---|
| ホストに読み込むリモート JavaScript（Module Federation を含む） | リモートにホストページと同じ権限を与えたものとして扱う。配信元オリジンが違っても、実行中のコードはサンドボックス化されない |
| ホストページ内の Web Components | 同一アプリケーションの一部として扱う。Shadow DOM やスコープ付きスタイルはホストからスクリプトを隔離しない |
| クロスオリジン iframe | 機能をホストの DOM とオリジンのストレージから分離したい場合に使う。能力を制限し、境界を越えるメッセージを明示的に制御する |
| サーバー／エッジで組み立てた HTML フラグメント | フラグメントとそのスクリプトをホストのコンテンツとしてレビューする。配信前の組み立ては、ブラウザ上の隔離境界を作らない |

**根拠**:
- Module Federation 等で読み込んだリモートは、ホストと同じオリジンで同じ権限を持って実行される。「別チームのコード」であることは実行時の隔離を意味しない
- 信頼レベルの異なる機能（サードパーティ提供・外部委託など）だけを、専用オリジンの iframe に分離する設計が、ブラウザが実際に強制できる唯一の境界になる
- オリジン分離はホストの DOM とストレージへの直接アクセスを制限するだけで、バックエンドへのリクエストを認可したり、意図的に iframe へ渡したデータを秘匿したりはしない

**コード例**:
```html
<!-- Good: 信頼度の低い機能は専用オリジン + 必要最小限の sandbox -->
<iframe
  src="https://widget.partner.example.com/embed"
  sandbox="allow-scripts allow-forms"
  referrerpolicy="strict-origin-when-cross-origin"
></iframe>

<!-- Bad: ホストと同一オリジンのコンテンツで allow-scripts と allow-same-origin を併用 -->
<iframe src="/embedded/widget" sandbox="allow-scripts allow-same-origin"></iframe>
```

**sandbox 設定の注意点**:
- トップレベルナビゲーションとポップアップの許可は、必要になるまで無効のままにする
- ホスト自身のオリジンのコンテンツで `allow-scripts` と `allow-same-origin` を併用しない。埋め込まれたアプリが自分の sandbox を外せてしまう
- `allow-same-origin` を付けない sandbox 文書のオリジンは opaque になり、メッセージ上は `"null"` と表示される。`"null"` を送信者の識別に使わない

**出典引用**:
> "Separate repositories and deployment teams do not create browser security boundaries: the same-origin policy separates origins, not individual features within a page."
> ([Micro-Frontend Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Micro_Frontend_Security_Cheat_Sheet.md), セクション "Introduction") ※2026-10-03に実際にfetch成功

> "Do not combine `allow-scripts` and `allow-same-origin` for content on the host's own origin; that combination can let the embedded application remove its sandbox."
> ([Micro-Frontend Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Micro_Frontend_Security_Cheat_Sheet.md), セクション "Isolate Features with Different Trust Levels") ※2026-10-03に実際にfetch成功

**出典**:
- [OWASP Micro-Frontend Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Micro_Frontend_Security_Cheat_Sheet.md) (OWASP CheatSheetSeries 公式、PR #2498 で新規追加) ※2026-10-03 fetch

**バージョン**: 全モダンブラウザ（iframe `sandbox` / `postMessage`）
**確信度**: 高
**最終更新**: 2026-10-03

---

### 2. リモートの公開権限を「ホストの実行中アプリを変更する権限」として扱い、読み込み先・リリース・整合性を制御する

マイクロフロントエンドのリモートを公開できることは、ホストが実行するコードを書き換えられることと同義である。リモートの読み込み元はホスト側の設定で固定し、レビュー済みの不変リリースだけを選択する。

**根拠**:
- 読み込む HTTPS URL はホストが管理する設定の承認済みリストに限り、クエリパラメータ等の信頼できない入力で実行コードを選ばせない
- エントリスクリプトと、そこから動的に読み込まれる依存チャンクの両方を含めて不変のリリースを選び、既知の正常なリリースへ戻せる状態を保つ
- シェルと各リモートでデプロイ用資格情報を分離すると、他のデプロイへの直接変更は制限できる。ただし、すでにシェルで実行を許可された侵害済みリモートは封じ込められない
- Subresource Integrity はバイト列の変更を検出するだけで、承認済みリリース内の悪意ある挙動は検出しない。動的チャンクが検証対象に含まれているかも確認する
- CSP はリモートの読み込み元の制限に使えるが、許可されたスクリプトソースは信頼されたコードのままであり、許可された複数のマイクロフロントエンド同士の隔離にはならない

**コード例**:
```ts
// Good: 読み込み先はホストが管理する設定の固定リストから選ぶ（URL パラメータで選ばせない）
const REMOTES = {
  checkout: 'https://cdn.example.com/checkout/2026-10-02.4f3a1c/remoteEntry.js', // 不変リリース
} as const;

type RemoteName = keyof typeof REMOTES;

export function remoteUrl(name: string): string {
  if (!(name in REMOTES)) throw new Error(`unknown remote: ${name}`);
  return REMOTES[name as RemoteName];
}

// Bad: 信頼できない入力が実行コードの取得先を決める
const url = new URLSearchParams(location.search).get('remote')!;
await import(/* webpackIgnore: true */ url);
```

**出典引用**:
> "Load only approved HTTPS remote URLs from host-controlled configuration. Do not let query parameters or other untrusted input choose executable code."
> ([Micro-Frontend Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Micro_Frontend_Security_Cheat_Sheet.md), セクション "Control Remote Code and Deployments") ※2026-10-03に実際にfetch成功

> "Verify coverage of dynamically loaded chunks; checking an entry script alone does not verify everything it later loads. Integrity checks detect changed bytes, not malicious behavior in an approved release."
> ([Micro-Frontend Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Micro_Frontend_Security_Cheat_Sheet.md), セクション "Control Remote Code and Deployments") ※2026-10-03に実際にfetch成功

**出典**:
- [OWASP Micro-Frontend Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Micro_Frontend_Security_Cheat_Sheet.md) (OWASP CheatSheetSeries 公式、PR #2498 で新規追加) ※2026-10-03 fetch

**バージョン**: Module Federation / 動的 `import()` を使う構成全般
**確信度**: 高
**最終更新**: 2026-10-03

---

### 3. マイクロフロントエンド間の通信は `postMessage` の厳密な検証とサーバー側認可で固め、共有イベントバスや共有ストレージを信頼境界にしない

ホストと iframe の各ペアについて、小さなメッセージ契約を定義し、`targetOrigin`・`event.origin`・`event.source`・データを検証する。これらは「誰が送ったか」を識別するだけで、ユーザーの権限は証明しない。特権操作を要求するメッセージは、必ずサーバー側の認可に繋げる。

**根拠**:
- 同一オリジンの iframe は複数存在しうるため、`event.origin` だけでなく `event.source` を期待するフレームの `contentWindow`（または親ウィンドウ）と照合する
- ページ内の共有イベントバスにはブラウザが強制する識別境界がなく、イベント名やアプリケーション識別子を権限の証明に使えない
- ストレージキーのプレフィックスや状態ストア・コンポーネントの分割は、同一ページ内のスクリプト同士のセキュリティ隔離にならない。ページやオリジンのストレージに置いたデータは、そこで実行される全リモートから読めるものとして扱う
- シェルでもリモートでも、クライアント側のコードは認可を強制できない。ルートガード・非表示のコントロール・クライアント側のロール判定は表示にしか効かず、迂回できる
- `HttpOnly` Cookie は JavaScript からの読み取りを防ぐが、ホスト内の侵害コードは認証済みリクエストをそのまま送れる

**コード例**:
```ts
// Good: origin と source の両方を検証し、受け付けるメッセージ型を絞る
const FRAME_ORIGIN = 'https://widget.partner.example.com';

window.addEventListener('message', (event: MessageEvent) => {
  if (event.origin !== FRAME_ORIGIN) return;                 // sandbox 付き iframe は "null" になるため識別に使わない
  if (event.source !== widgetFrame.contentWindow) return;    // 同一オリジンの別フレームを排除
  const msg = event.data;
  if (msg?.type !== 'widget:resize' || typeof msg.height !== 'number') return;
  widgetFrame.style.height = `${Math.min(msg.height, 2000)}px`;
});

// 送信側: targetOrigin は '*' にせず厳密に指定する
widgetFrame.contentWindow?.postMessage({ type: 'host:init', locale: 'ja' }, FRAME_ORIGIN);

// Bad: 全オリジンに送信し、受信側も origin を見ない
window.parent.postMessage({ type: 'session', token }, '*');
```

**アンチパターン**:
- メッセージで受け取った `role` / `tenantId` / 権限フラグをバックエンドが信用する
- セッション識別子を `localStorage` / `sessionStorage` に置く（BFF 構成ならアップストリームのアクセストークンはサーバー側に保持し、セッション Cookie を使う）
- 認証情報や機微な状態を全機能にブロードキャストする

**出典引用**:
> "Check `event.source` against the expected frame's `contentWindow` or the expected parent window, not only the origin. Several frames can share one origin."
> ([Micro-Frontend Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Micro_Frontend_Security_Cheat_Sheet.md), セクション "Restrict Cross-Application Communication") ※2026-10-03に実際にfetch成功

> "A shared in-page event bus has no browser-enforced identity boundary between its participants. Do not use event names or application identifiers as proof of authority."
> ([Micro-Frontend Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Micro_Frontend_Security_Cheat_Sheet.md), セクション "Restrict Cross-Application Communication") ※2026-10-03に実際にfetch成功

> "Neither the shell nor a remote micro-frontend can enforce authorization in client-side code."
> ([Micro-Frontend Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Micro_Frontend_Security_Cheat_Sheet.md), セクション "Authorize Every Request on the Server") ※2026-10-03に実際にfetch成功

**出典**:
- [OWASP Micro-Frontend Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Micro_Frontend_Security_Cheat_Sheet.md) (OWASP CheatSheetSeries 公式、PR #2498 で新規追加) ※2026-10-03 fetch

**バージョン**: 全モダンブラウザ（`postMessage` / `MessageEvent.source`）
**確信度**: 高
**最終更新**: 2026-10-03
