# Bootstrap v2: web-standards — 2026-05-06

## 概要

web-standards カテゴリに補強コンテンツ（v2）として計 **8件** の新規ルールを追加。
既存ルールとの重複なし。

## 追加ルール一覧

### css.md（+3件）
| # | ルールサマリ | バージョン |
|---|---|---|
| 4 | CSS Container Queries でコンポーネント単位のレスポンシブを実装する | Chrome 105+, Firefox 110+, Safari 16+ |
| 5 | CSS Nesting のネイティブサポートを活用する | Chrome 112+, Firefox 117+, Safari 17.2+ |
| 6 | `:has()` 疑似クラスで親要素を条件スタイリングする | Chrome 105+, Firefox 121+, Safari 15.4+ |

### html.md（+2件）
| # | ルールサマリ | バージョン |
|---|---|---|
| 4 | `<dialog>` 要素でモーダルをネイティブ実装する | Chrome 37+, Firefox 98+, Safari 15.4+ |
| 5 | Popover API でポップオーバーUIをネイティブ実装する | Chrome 114+, Firefox 125+, Safari 17+ |

### browser-apis.md（+2件）
| # | ルールサマリ | バージョン |
|---|---|---|
| 4 | View Transitions API でページ・状態遷移にアニメーションを付ける | Chrome 111+, Safari 18+ |
| 5 | `AbortController` で fetch リクエストをキャンセルする | Chrome 66+, Firefox 57+, Safari 12.1+ |

### accessibility.md（+2件）
| # | ルールサマリ | バージョン |
|---|---|---|
| 4 | `aria-live` リージョンで動的コンテンツの変化をスクリーンリーダーに通知する | ARIA 1.2 |
| 5 | `focus-visible` と Focus Trap でフォーカス体験を最適化する | Chrome 86+, Firefox 85+, Safari 15.4+ |

## アクセス経路統計

| ソース | 試行 | 結果 |
|---|---|---|
| GitHub MCP (mdn/content) | 3件 | 全件失敗（リポジトリアクセス制限） |
| WebFetch (MDN URLs) | 3件 | 全件失敗（HTTP 403） |
| WebFetch (web.dev/Chrome Devs) | 3件 | 全件失敗（HTTP 403） |
| トレーニングデータ内知識 | — | 全8ルールの根拠として使用 |

外部ドキュメントへのアクセスがすべて遮断されたため、
W3C仕様・MDN・WHATWG の公開内容に基づくトレーニング知識から
ルールを作成。各ルールの出典URLは正式な仕様・ドキュメントへの参照を記載済み。

## ブランチ

`bootstrap/web-standards-2026-05-06`

## 作成日

2026-05-06
