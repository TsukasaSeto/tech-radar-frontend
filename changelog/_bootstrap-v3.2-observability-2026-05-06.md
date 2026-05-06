# Bootstrap v3.2: observability (2026-05-06)

## 結果サマリ
- 既存ルール強化: 3件
- 新規ルール追加: 1件
- 全段階失敗でスキップしたソース: 0件

## 公式ドキュメント取得統計

| リポジトリ | 候補1(raw) | 候補2(raw) | 候補3(raw) | Search API | 結果 |
|---|---|---|---|---|---|
| getsentry/sentry-javascript (README.md) | ✅ 200 (README.md) | - | - | - | 高レベル概要のみ、詳細ルール抽出に候補2も使用 |
| getsentry/sentry-javascript (packages/nextjs/README.md) | - | ✅ 200 (packages/nextjs/README.md) | - | - | 候補2で成功（setUser・setTag・ユーザーコンテキスト確認） |

## コミュニティ取得統計

| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 結果 |
|---|---|---|---|---|---|---|
| Zenn sentry | ✅ 200 | - | - | - | - | 段階1で成功 |

## 個別記事取得統計

| URL | 取得経路 | 結果 |
|---|---|---|
| https://zenn.dev/levi/articles/4ace7342e2f77f | 段階A 直接WebFetch ✅ 200 | React × Sentry実践パターン（3層ノイズフィルタ・動的サンプリング・React 19 createRootハンドラ・getsentry/action-release）抽出 |

## 抽出されたルール

### 既存ルール強化

1. `practices/observability/error-tracking.md` - ルール1「Sentry を Next.js に統合し、全環境でエラーを自動キャプチャする」
   - 追加根拠: React 19 の `createRoot` に追加された3カテゴリ（`onUncaughtError`・`onCaughtError`・`onRecoverableError`）が `global-error.tsx` だけでは取りこぼすエラーを補完。`Sentry.setUser()` / `Sentry.setTag()` によるユーザーコンテキスト付与パターン確認。
   - 出典: getsentry/sentry-javascript packages/nextjs README + Zenn levi記事 ※2026-05-06に実際にfetch成功

2. `practices/observability/error-tracking.md` - ルール2「本番では tracesSampleRate を 0.1 以下に設定し、コストを管理する」
   - 追加根拠: トレースサンプリングとは独立したエラーレベルサンプリング: `beforeSend` で通常エラー 25% / クリティカルエラー 100% の2段階制御パターン確認。
   - 出典: Zenn levi記事 ※2026-05-06に実際にfetch成功

3. `practices/observability/error-tracking.md` - ルール4「Source Map をビルド時に自動アップロードし、本番のスタックトレースを可読にする」
   - 追加根拠: `getsentry/action-release@v1` GitHub Action によるリリース自動作成。リリース名フォーマット "Application@1.0.0+202511100932"（名前+バージョン+タイムスタンプ）で時系列フィルタリング可能。
   - 出典: Zenn levi記事 ※2026-05-06に実際にfetch成功

### 新規ルール

1. `practices/observability/error-tracking.md` - ルール7「`ignoreErrors` / `thirdPartyErrorFilterIntegration` / `beforeSend` の3層でノイズを削減する」
   - 内容: 層1 `ignoreErrors`（既知ノイズパターンをregexpで排除）→ 層2 `thirdPartyErrorFilterIntegration`（自社コードフレームを持たないサードパーティエラーを自動排除）→ 層3 `beforeSend`（残余ノイズの除去 + `unhandledRejection`の`'[object Object]'`不正シリアライズ正規化）。
   - 出典: Zenn levi記事 ※2026-05-06に実際にfetch成功

## 参照した記事一覧

| URL | 取得経路 | 抽出形式 |
|---|---|---|
| https://raw.githubusercontent.com/getsentry/sentry-javascript/develop/README.md | raw.githubusercontent.com 直接 ✅ 200 | README（高レベル概要のみ） |
| https://raw.githubusercontent.com/getsentry/sentry-javascript/develop/packages/nextjs/README.md | raw.githubusercontent.com 直接 ✅ 200 | パッケージ固有README / 既存ルール追加根拠 |
| https://zenn.dev/topics/sentry/feed | Zenn RSS 直接 ✅ 200 | フィード一覧取得 |
| https://zenn.dev/levi/articles/4ace7342e2f77f | 直接WebFetch ✅ 200 | 個別記事 / React × Sentry実践パターン |

## アクセス経路統計サマリ

| ソース | 段階1 | 段階2 | 段階3 | 段階4 | 段階5 | 最終結果 |
|---|---|---|---|---|---|---|
| Zenn sentry | ✅ 200 | - | - | - | - | 段階1で成功 |

## 公式ドキュメント経路統計

| リポジトリ | 候補1(raw) | 候補2(raw) | 候補3(raw) | Search API | 結果 |
|---|---|---|---|---|---|
| getsentry/sentry-javascript (README.md) | ✅ 200 | - | - | - | 取得成功・高レベル概要のみのため候補2も使用 |
| getsentry/sentry-javascript (packages/nextjs/README.md) | - | ✅ 200 | - | - | 候補2で成功 |

合計成功: 4/4 (100%)
