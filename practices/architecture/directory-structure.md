# ディレクトリ構造のベストプラクティス

## ルール

### 1. 機能（feature）単位でコードをグループ化する

グローバルな `components/`・`hooks/`・`utils/` の代わりに、
機能ごとのディレクトリにコードをまとめる（Feature-Sliced Design）。

**根拠**:
- 機能に関連するすべてのファイルが一箇所にあり、削除・変更が容易
- 機能をまたぐ依存関係が明確になり、意図しない結合を防ぐ
- 機能のオーナーシップが明確になりチームのスケーラビリティが向上する

**コード例**:
```
src/
├── app/                    # Next.js App Router
│   ├── (auth)/             # 認証ルートグループ
│   │   ├── login/
│   │   └── register/
│   ├── dashboard/
│   └── api/
├── features/               # 機能単位のコード
│   ├── auth/               # 認証機能
│   │   ├── components/     # auth専用コンポーネント
│   │   │   ├── LoginForm.tsx
│   │   │   └── RegisterForm.tsx
│   │   ├── hooks/          # auth専用フック
│   │   │   └── useAuth.ts
│   │   ├── actions/        # auth専用 Server Actions
│   │   │   └── auth.ts
│   │   └── index.ts        # パブリックAPI（公開するものだけ export）
│   ├── users/              # ユーザー機能
│   │   ├── components/
│   │   ├── hooks/
│   │   └── index.ts
│   └── posts/              # 投稿機能
│       ├── components/
│       ├── hooks/
│       └── index.ts
├── shared/                 # 複数機能で共有
│   ├── components/         # Button, Modal, Input など汎用UI
│   ├── hooks/              # useDebounce, useLocalStorage など汎用フック
│   ├── lib/                # db, auth, email などのライブラリ設定
│   └── types/              # 共有型定義
└── test/                   # テスト設定
    ├── setup.ts
    └── mocks/
```

**出典**:
- [Feature-Sliced Design](https://feature-sliced.design/) (Feature-Sliced Design公式)
- [Next.js Docs: Project Structure](https://nextjs.org/docs/app/getting-started/project-structure) (Next.js公式)

**バージョン**: Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 2. `index.ts` バレルファイルで機能のパブリック API を定義する

各機能ディレクトリに `index.ts` を置き、外部に公開するものだけを再エクスポートする。
機能内部の実装詳細を外部から直接インポートできなくする。

**根拠**:
- 機能の境界が明確になり、内部実装を変えても外部への影響が限定される
- インポートパスが短くなる（`features/auth` で済む）
- 「何が公開 API か」が `index.ts` を見るだけで分かる

**コード例**:
```ts
// features/auth/index.ts - パブリック API のみ公開
export { LoginForm } from './components/LoginForm';
export { RegisterForm } from './components/RegisterForm';
export { useAuth } from './hooks/useAuth';
export type { AuthUser, AuthError } from './types';
// 内部実装（AuthFormField, validatePassword など）は export しない

// features/posts/index.ts
export { PostList } from './components/PostList';
export { PostCard } from './components/PostCard';
export { usePost } from './hooks/usePost';
export type { Post, CreatePostInput } from './types';

// 利用側: 機能のパブリック API からインポート
import { LoginForm, useAuth } from '@/features/auth';
import { PostList } from '@/features/posts';

// Bad: 内部実装への直接インポート
import { AuthFormField } from '@/features/auth/components/AuthFormField';  // 内部詳細
```

**出典**:
- [Bulletproof React: Project Structure](https://github.com/alan2207/bulletproof-react) (Alan Alickovic)

**バージョン**: TypeScript 5+, Next.js 13+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. Feature-Sliced Design の7層レイヤーを意識して配置する

FSD（Feature-Sliced Design）では `app > pages > widgets > features > entities > shared` の階層を定義する。
Next.js App Router 環境では `app/` がルーティング層を担うため、残りの層を `src/` 以下に明示的に配置する。
各層は自分より下位の層にのみ依存でき、同一層・上位層への依存は禁止する。

**根拠**:
- 層の制約により依存の方向が一意に定まり、スパゲッティ依存が発生しない
- 新機能追加時にどの層に置くべきかの判断基準が明確になる
- チームが共通の語彙（"これは entity か feature か"）で議論できる

**コード例**:
```
src/
├── app/            # (Next.js App Router) ルーティング・レイアウト
├── pages/          # ページ単位の合成コンポーネント（widget/feature を組み合わせる）
│   └── dashboard/
│       └── ui/DashboardPage.tsx
├── widgets/        # 独立した大きなUIブロック（ヘッダー・サイドバーなど）
│   └── sidebar/
├── features/       # ユーザーシナリオ単位（認証・投稿作成など）
│   └── auth/
├── entities/       # ビジネスエンティティ（User・Post・Order など）
│   └── user/
│       ├── model/  # 型・ストア・セレクター
│       └── ui/     # エンティティUI（UserAvatar など）
└── shared/         # 再利用可能なインフラ（UI kit・lib・api）
    ├── ui/
    ├── lib/
    └── api/

// Good: features は entities に依存できる
// features/auth/hooks/useAuth.ts
import { type User } from '@/entities/user';

// Bad: entities が features に依存
// entities/user/model/user.ts
import { useAuthStore } from '@/features/auth'; // 上位層への依存 NG
```

**出典**:  
- [Feature-Sliced Design: Reference](https://feature-sliced.design/docs/reference/layers) (Feature-Sliced Design公式 / 2023)

**バージョン**: Next.js 13+, TypeScript 5+
**確信度**: 高
**最終更新**: 2026-05-06

---

### 4. コロケーション原則：関連ファイルはコンポーネントと同じディレクトリに置く

コンポーネントのテスト・スタイル・型・Storybook ファイルは、
そのコンポーネントと同じディレクトリに置く（コロケーション）。
グローバルな `__tests__/` や `styles/` ディレクトリへの分散を避ける。

**根拠**:
- コンポーネントを削除するときに関連ファイルが見つけやすく、削除漏れが防げる
- ファイルの関連性がディレクトリ構造から一目で分かる
- Kent C. Dodds の「Place code as close to where it's relevant as possible」原則に基づく

**コード例**:
```
// Good: コロケーション
features/auth/components/LoginForm/
├── LoginForm.tsx          # コンポーネント本体
├── LoginForm.test.tsx     # ユニットテスト
├── LoginForm.stories.tsx  # Storybook
├── LoginForm.module.css   # スコープ付きCSS (CSS Modules使用時)
└── index.ts               # 再エクスポート

// Bad: グローバルに分散
src/
├── components/LoginForm.tsx
├── __tests__/LoginForm.test.tsx   # 離れた場所にテスト
├── stories/LoginForm.stories.tsx  # 離れた場所に Story
└── styles/LoginForm.module.css    # 離れた場所にスタイル
```

**出典**:
- [Colocation](https://kentcdodds.com/blog/colocation) (Kent C. Dodds / 2019)
- [Testing Library: Colocation](https://testing-library.com/docs/guiding-principles) (Testing Library公式)

**バージョン**: React 18+, TypeScript 5+
**確信度**: 高
**最終更新**: 2026-05-06

---

#### 追加根拠 (2026-05-16) — ルール4「コロケーション原則：関連ファイルはコンポーネントと同じディレクトリに置く」

新たに以下の記事でコロケーション + バレルインポートのより具体的な実装パターンが示された:
- [Next.js App Router × コロケーション × バレルインポートで実現する、AI時代の人間との協働開発](https://qiita.com/masakinihirota/items/ffce9f02e066f52d4d97) (Qiita masakinihirota / 2026-05-16) ※2026-05-16に実際にfetch成功

**出典引用**:
> 「巨大なモノリシックアプリを構築するのではなく、小さくて自己完結したミニアプリを組み合わせて全体を作ります。」
> (セクション "主要な設計思想")

App Router 環境でのコロケーションを強化する具体的な追加パターン:

**ファイル命名規則（サフィックスによる役割分離）**:
```
features/home/
├── home.view.tsx        # 純粋なUI表示（propsのみ受け取る）
├── home.container.tsx   # データ取得・状態管理
├── home.logic.ts        # ビジネスロジック
├── home.hook.ts         # カスタムReact Hook
├── home.types.ts        # 型定義
├── home.logic.test.ts   # ユニットテスト
└── index.ts             # 公開API（バレル）
```

**`@barrel-type` アノテーションで Server/Client 境界を明示**:
```typescript
// src/components/home/index.ts
// @barrel-type: server    ← Server Component のみ
// @barrel-type: client    ← Client Component のみ（"use client"付き）
// @barrel-type: mixed     ← 両方が混在（AIへの警告を含む）
export { HomeHeader } from './home-header';
export { HomeTabs } from './home-tabs';
```

深いインポートパス（`@/components/home/home-tabs/home-tabs-skeleton`）を禁止し、
`index.ts` 経由のみに限定することで内部リファクタリング時の影響を最小化する。
AI協働開発時は「`src/components/home/` を修正して。公開APIは `home/index.ts`」という指示で
Claude の参照対象を絞り込め、関係ないファイルへの変更提案が減少する。

**確信度**: 既存（高）→ 高（App Router 固有のServer/Client境界管理パターンを追加）

---

### 5. バレルエクスポート（index.ts）の過剰使用を避ける

`index.ts` を機能の境界（features/・shared/）にのみ配置し、
すべてのサブディレクトリに再帰的に作成しない。
過剰なバレルエクスポートはビルドツールの Tree Shaking 効率を下げ、循環依存の温床になる。

**根拠**:
- すべてのディレクトリに `index.ts` を置くと、バンドラーが不要なモジュールをすべて解析する
- 循環インポートが `index.ts` を経由して検出しにくくなる
- Vercel の調査では、過剰なバレルファイルが Next.js のコールドスタート時間を増加させると報告されている

**コード例**:
```ts
// Good: 機能境界のみに index.ts を置く
// features/auth/index.ts (OK - 外部公開APIの境界)
export { LoginForm } from './components/LoginForm/LoginForm';
export { useAuth } from './hooks/useAuth';

// features/auth/components/index.ts (不要)
// features/auth/components/LoginForm/index.ts (不要 - 直接参照で十分)

// Bad: すべての階層に index.ts を作る
// features/auth/components/index.ts
export { LoginForm } from './LoginForm';
export { RegisterForm } from './RegisterForm';
export { ForgotPasswordForm } from './ForgotPasswordForm';
// → 内部コンポーネントの変更がすべてここを通過し、循環依存の発見が遅れる

// tsconfig.json でパスエイリアスを使い直接参照する
// {
//   "paths": { "@/features/*": ["./src/features/*"] }
// }
import { LoginForm } from '@/features/auth'; // 境界のみ経由
```

**出典**:
- [Barrel files and why you should avoid them](https://marvinh.dev/blog/speeding-up-javascript-ecosystem-part-7/) (Marvin Hagemeister / 2023)
- [Vercel: How we optimized Next.js cold starts](https://vercel.com/blog/how-we-optimized-package-imports-in-next-js) (Vercel公式 / 2023)

**バージョン**: Next.js 13+, TypeScript 5+
**確信度**: 高
**最終更新**: 2026-05-06

---

#### 追加根拠 (2026-05-06) — ルール5「バレルエクスポート（index.ts）の過剰使用を避ける」

新たに以下の記事/ドキュメントで同じプラクティスが推奨された:
- [なぜ大規模ではバレルエクスポートが使えないのか](https://zenn.dev/fujishu/articles/c7c7799140b488) (Zenn fujishu / 2026) ※2026-05-06に実際にfetch成功
- [Bulletproof React: Project Structure](https://raw.githubusercontent.com/alan2207/bulletproof-react/master/docs/project-structure.md) (alan2207 / masterブランチ) ※2026-05-06に実際にfetch成功

Zenn記事が数学的根拠と実測データを提示: 1ファイルが barrel から import するとき取り込まれる不要モジュールの割合は (N - k) / N（Nはモジュール総数、kは実際に必要なモジュール数）。プロジェクト規模が大きくなるほどNが増大し、オーバーヘッド率が1に収束する。実測でバレルexport 使用時のコンパイル 7.44s vs 直接インポート 5.98s（1.55秒 = 約21%改善）。Bulletproof React公式ドキュメントも「barrel filesを使わずにファイルを直接インポートすること」を明示推奨し、Viteのtree-shakingを有効に維持する目的で feature 内部は直接パス参照を原則にしている。

**確信度**: 既存（高）→ 高（数学的根拠 + 実測コンパイル時間 + 公式ガイド実証済み）

---

### 6. Server Component 配下では Container/Presentational パターンで「データ取得」と「表示」を分離する

React Server Components は React Testing Library / Storybook の対応が遅れているため、Server Component を直接ユニットテスト・ビジュアル確認するのが難しい。
そこで **DAL 呼び出しなどサーバ処理を担う Container** と **props を受け取って表示するだけの Presentational** に明示的に分離し、Presentational 側をテスト・Storybook の対象にする。
Container 配下は `_containers/<name>/` でコロケーションし、`index.tsx` から Container のみを再エクスポートして外部公開窓口とする。

**根拠**:
- RTL / Storybook が Server Component に未対応のため、純粋表示部だけを Client Component（または props 受け取りの関数コンポーネント）として切り出すとテスト・カタログ化が容易になる
- Container には DAL 呼び出し（§Data Access Layer）が集約され、Presentational は副作用フリーで「props → JSX」の純粋関数として扱える
- ディレクトリ名 `_containers/` のアンダースコア prefix によりルーティング対象から除外され、`page.tsx` / `layout.tsx` と並べて配置しても URL を汚染しない
- `index.tsx` 経由でのみ Container を公開することで、`presentational.tsx` への外部 import を機械的に禁止できる

**コード例**:
```
// Good: _containers/<name>/ にコロケーション
app/posts/[postId]/
├── page.tsx
├── layout.tsx
└── _containers/
    ├── post/
    │   ├── index.tsx          # Container を re-export（外部公開窓口）
    │   ├── container.tsx      # DAL 呼び出しなどサーバ処理
    │   └── presentational.tsx # props を受けて表示するだけ
    └── user-profile/
        ├── index.tsx
        ├── container.tsx
        └── presentational.tsx
```

```tsx
// Good: container.tsx — サーバ処理のみ
import { getPost } from '@/app/_lib/dal/post';
import { PostPresentational } from './presentational';

export async function PostContainer({ postId }: { postId: string }) {
  const post = await getPost(postId); // DAL 経由
  return <PostPresentational post={post} />;
}

// Good: presentational.tsx — props を受けて表示するだけ（テスト・Storybook 対象）
type Props = { post: { title: string; body: string } };
export function PostPresentational({ post }: Props) {
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
    </article>
  );
}

// Good: index.tsx — Container のみ公開
export { PostContainer } from './container';

// Bad: page.tsx で DAL 呼び出しと JSX を混在させ、Storybook も書けない
export default async function Page({ params }: { params: Promise<{ postId: string }> }) {
  const { postId } = await params;
  const post = await getPost(postId);
  return <article><h1>{post.title}</h1><p>{post.body}</p></article>;
}
```

**アンチパターン**:
- Presentational を外部から直接 import してしまい、Container 経由の責務分離が崩れる（`eslint-plugin-import-access` や Biome の `noPrivateImports` で防ぐ）
- Presentational 側に DAL 呼び出しや `cookies()` などのサーバ専用 API を持ち込み、テスト可能性が失われる
- `_containers/` を作らず、`page.tsx` 1 ファイル内でデータ取得と表示を混在させる（Server Component なのでテスト困難）

**出典**:
- [Next.jsの考え方 / Container/Presentational パターン](https://zenn.dev/akfm/books/nextjs-basic-principle) (akfm_sato)

**取り込み元**: 別プロジェクト sstf-5461-admin-app チームドキュメント (2026-05-16 手動取り込み、akfm_sato 氏の Zenn book を原典として参照)

**バージョン**: Next.js 16+
**確信度**: 高（v16 公式相当の知見）
**最終更新**: 2026-05-16
