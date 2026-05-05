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
