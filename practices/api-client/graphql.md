# GraphQL クライアントのベストプラクティス

## ルール

### 1. Apollo Client より urql を優先的に検討する

新規プロジェクトでは Apollo Client より urql を優先的に評価する。
機能が十分な場合は urql の軽量・柔軟なアーキテクチャが有利。

**根拠**:
- urql はバンドルサイズが Apollo Client より大幅に小さい（~25KB vs ~90KB gzip）
- Exchange ベースのアーキテクチャで、キャッシュ・認証・エラー処理が差し替え可能
- Apollo Client は大規模な機能セット（Apollo Studio 連携、詳細なキャッシュ制御）が必要な場合に有利
- どちらも React 18 の Suspense に対応している

**コード例**:
```tsx
// urql のセットアップ（シンプル）
import { createClient, cacheExchange, fetchExchange, Provider } from 'urql';

const client = createClient({
  url: '/api/graphql',
  exchanges: [cacheExchange, fetchExchange],
  fetchOptions: () => {
    const token = getAuthToken();
    return {
      headers: { authorization: token ? `Bearer ${token}` : '' },
    };
  },
});

// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <Provider value={client}>
      {children}
    </Provider>
  );
}

// コンポーネントでの使用
import { useQuery } from 'urql';

const UserQuery = `
  query GetUser($id: ID!) {
    user(id: $id) { id name email }
  }
`;

function UserProfile({ id }: { id: string }) {
  const [result] = useQuery({ query: UserQuery, variables: { id } });
  const { data, fetching, error } = result;

  if (fetching) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <ProfileCard user={data.user} />;
}
```

**出典**:
- [urql Docs](https://commerce.nearform.com/open-source/urql/docs/) (nearform / urql公式)
- [Apollo Client Docs](https://www.apollographql.com/docs/react) (Apollo公式)

**バージョン**: urql 4+, Apollo Client 3+
**確信度**: 中
**最終更新**: 2026-05-05

---

### 2. Fragment Colocation でコンポーネントが必要なフィールドを宣言する

GraphQL フラグメントを、そのデータを使うコンポーネントと同一ファイルに定義する。
上位コンポーネントがフィールドを推測する必要がなくなる。

**根拠**:
- コンポーネントが必要なデータをコード上で宣言的に表現できる
- 上位コンポーネントがフィールドを追加・削除する際のリグレッションを防げる
- GraphQL Code Generator が Fragment から型を生成するため、型安全性が向上する
- Relay の思想を基盤としており、大規模アプリでのデータ管理に有効

**コード例**:
```tsx
// components/UserCard/UserCard.tsx
import { graphql } from '@/gql';

// このコンポーネントが必要なフィールドをフラグメントで宣言
export const UserCard_UserFragment = graphql(`
  fragment UserCard_User on User {
    id
    name
    avatarUrl
    role
  }
`);

// graphql-codegen が生成した型を使用
import type { UserCard_UserFragment } from '@/gql/graphql';

interface Props {
  user: UserCard_UserFragment;
}

export function UserCard({ user }: Props) {
  return (
    <div>
      <img src={user.avatarUrl} alt={user.name} />
      <p>{user.name}</p>
      <span>{user.role}</span>
    </div>
  );
}

// pages/TeamPage/TeamPage.tsx
const TeamPageQuery = graphql(`
  query TeamPage {
    team {
      members {
        ...UserCard_User  # UserCard が必要なフィールドだけを取得
      }
    }
  }
`);

function TeamPage() {
  const [{ data }] = useQuery({ query: TeamPageQuery });
  return (
    <div>
      {data?.team.members.map(member => (
        <UserCard key={member.id} user={member} />
      ))}
    </div>
  );
}
```

**出典**:
- [Apollo Docs: Fragments](https://www.apollographql.com/docs/react/data/fragments/) (Apollo公式)
- [urql Docs: Fragments](https://commerce.nearform.com/open-source/urql/docs/basics/core-logic/) (urql公式)

**バージョン**: Apollo Client 3+, urql 4+, GraphQL 16+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 3. GraphQL Code Generator で型とフック（またはドキュメント）を自動生成する

GraphQL スキーマとクエリファイルから TypeScript の型を自動生成する。
手書きの型定義は GraphQL スキーマとの乖離が生じるため使わない。

**根拠**:
- スキーマ変更時に型エラーが即座に検出される
- `client-preset` プラグインで Fragment の型も自動生成される
- `graphql-tag` の代わりに Codegen の `graphql()` 関数を使うことで型推論が働く
- CI でスキーマとのチェックを自動化できる

**コード例**:
```yaml
# codegen.ts (設定ファイル)
import type { CodegenConfig } from '@graphql-codegen/cli';

const config: CodegenConfig = {
  schema: 'https://api.example.com/graphql',
  documents: ['src/**/*.tsx', 'src/**/*.ts'],
  generates: {
    './src/gql/': {
      preset: 'client',
      plugins: [],
      presetConfig: {
        gqlTagName: 'graphql',
      },
    },
  },
};
export default config;
```

```bash
# 型生成コマンド（開発時は watch モード）
npx graphql-codegen --config codegen.ts --watch

# package.json
{
  "scripts": {
    "codegen": "graphql-codegen --config codegen.ts",
    "codegen:watch": "graphql-codegen --config codegen.ts --watch"
  }
}
```

```tsx
// 生成された graphql() 関数を使用（型推論が効く）
import { graphql } from '@/gql';

const GetUserQuery = graphql(`
  query GetUser($id: ID!) {
    user(id: $id) {
      id
      name
      email
    }
  }
`);

// GetUserQuery から自動的に型が推論される
const [{ data }] = useQuery({ query: GetUserQuery, variables: { id } });
//    ^^^^ data の型は { user: { id: string; name: string; email: string } | null } | undefined
```

**出典**:
- [GraphQL Code Generator Docs](https://the-guild.dev/graphql/codegen/docs/getting-started) (The Guild / GraphQL Code Generator公式)

**バージョン**: @graphql-codegen/cli 5+, @graphql-codegen/client-preset 4+
**確信度**: 高
**最終更新**: 2026-05-05

---

### 4. Mutation 後は適切なキャッシュ無効化または楽観的更新を行う

GraphQL Mutation 後、影響を受けたクエリを再フェッチまたは楽観的に更新する。
stale なデータをUIに表示し続けないよう必ず対処する。

**根拠**:
- Mutation 後にキャッシュが古い状態のままだとUIが不整合を起こす
- 楽観的更新はサーバーレスポンスを待たずにUIを更新し体感速度を向上させる
- urql の `cache.invalidate` / Apollo の `refetchQueries` で一貫した無効化が可能

**コード例**:
```tsx
// urql での Mutation 後キャッシュ無効化
import { useMutation, gql } from 'urql';

const CreatePostMutation = graphql(`
  mutation CreatePost($input: CreatePostInput!) {
    createPost(input: $input) {
      id
      title
    }
  }
`);

function CreatePostForm() {
  const [, createPost] = useMutation(CreatePostMutation);

  async function handleSubmit(input: CreatePostInput) {
    const result = await createPost({ input });
    if (result.error) {
      showErrorToast(result.error.message);
    }
    // urql の cache exchange が __typename を見て自動的に関連クエリを無効化
  }
}

// Apollo Client での楽観的更新
const [createPost] = useMutation(CREATE_POST_MUTATION, {
  optimisticResponse: {
    createPost: {
      __typename: 'Post',
      id: 'temp-id',
      title: input.title,
    },
  },
  update(cache, { data }) {
    cache.modify({
      fields: {
        posts(existingPosts = []) {
          const newPostRef = cache.writeFragment({
            data: data?.createPost,
            fragment: PostFragment,
          });
          return [...existingPosts, newPostRef];
        },
      },
    });
  },
});
```

**出典**:
- [Apollo Docs: Optimistic UI](https://www.apollographql.com/docs/react/performance/optimistic-ui/) (Apollo公式)
- [urql Docs: Cache Updates](https://commerce.nearform.com/open-source/urql/docs/graphcache/cache-updates/) (urql公式)

**バージョン**: Apollo Client 3+, urql 4+
**確信度**: 高
**最終更新**: 2026-05-05

---

## 関連プラクティス

- [`api-client/caching.md`](./caching.md) - キャッシュ戦略全般
- [`api-client/type-safety.md`](./type-safety.md) - 型生成ツール
- [`api-client/error-handling.md`](./error-handling.md) - GraphQL エラー処理
