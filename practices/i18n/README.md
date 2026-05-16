# i18n Practices

国際化（Internationalization）と地域化（Localization）に関するベストプラクティス。
ルーティング・メッセージ管理・RTL 対応・日時／数値ローカライズを扱う。

## ファイル構成

- `app-router-i18n.md` — Next.js App Router でのロケール検出とルーティング
- `message-catalog.md` — メッセージ管理（ICU MessageFormat / 翻訳ファイル構成 / 型安全）
- `rtl.md` — RTL（右から左）言語対応（論理プロパティ・dir 属性）
- `localization.md` — `Intl.*` API による日時・数値・通貨フォーマット

## スコープ

- 関係する仕様: BCP 47, ECMAScript Internationalization API (ECMA-402), Unicode CLDR
- 対象フレームワーク: Next.js App Router / Pages Router / React 単体
- 翻訳サービス（Crowdin / Lokalise / Phrase）はリファレンスとして言及するが詳細は扱わない
