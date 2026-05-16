# Next.js Practices

Next.js の App Router、Server Components、Server Actions、ルーティング、キャッシング等に関するベストプラクティス。

## トピック

- `server-components.md` - Server Components の活用
- `data-fetching.md` - データ取得戦略
- `routing.md` - App Router 設計
- `caching.md` - キャッシュ戦略
- `server-actions.md` - Server Actions の活用
- `middleware.md` - Middleware の運用
- `../architecture/data-access-layer.md` - DAL（データアクセスの単一出入り口）

## 設計判断の優先順位

ルール同士が衝突した時、または採否に迷った時は **A → B → C** の順で適用する。
**A は破ると脆弱性、B は破るとアーキテクチャ手戻り、C は破るとパフォーマンス劣化**という影響度の差を意識する。

### A. セキュリティ（最優先 / 破ると脆弱性）

- データの読み書きは必ず DAL を経由する（コンポーネント・Server Action から直接 fetch/DB アクセスしない）
- `"use server"` は HTTP エンドポイント公開と等価。export 関数は最小化し、関数内で必ず認可・バリデーション
- レイアウトでの認可チェックは禁止（並行レンダリングで漏洩リスク）。認可は page.tsx 冒頭 + DAL の二重で実施
- `server-only` / `client-only` でモジュール境界を保護

### B. アーキテクチャ（破ると手戻り大）

- データフェッチは Server Components で行う（Client Components で fetch しない）
- データ取得は末端コンポーネントへコロケーションする（Props バケツリレーを作らない）
- データ操作は Server Actions で行う（Route Handlers は外部 webhook 用が基本）
- `"use client"` はバンドル境界の宣言。Server を子に渡したいときは **Composition (children)** を使う
- Server Components は純粋に保つ（`Math.random()` / `Date.now()` / Cookie `set/delete` などの副作用は禁止）

### C. パフォーマンス・キャッシュ

- v16 の Cache Components 環境では **キャッシュはデフォルトで効かない**。`"use cache"` で明示的にオプトイン
- `<Suspense>` 境界 = Dynamic Content の境界。動的 I/O は `<Suspense>` 内側 **または** `"use cache"` 配下のいずれかでしか扱えない

**出典**:
- [Next.jsの考え方 / 大原則](https://zenn.dev/akfm/books/nextjs-basic-principle)

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**バージョン**: Next.js 16+
**最終更新**: 2026-05-16
