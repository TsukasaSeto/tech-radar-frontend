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

App Router では Provider（Context と client インスタンス）を `'use client'` の Providers コンポーネントに分け、`app/layout.tsx` は Server Component のまま包む。

**コード例**:
```tsx
// app/providers.tsx（Client Component。App Router では @urql/next を使う）
'use client';

import { useMemo } from 'react';
import { UrqlProvider, ssrExchange, cacheExchange, fetchExchange, createClient } from '@urql/next';
import { getAuthToken } from '@/lib/auth-client'; // アプリ側で用意するトークン取得関数

export function Providers({ children }: { children: React.ReactNode }) {
  // client はコンポーネント内で 1 回だけ作る（リクエスト間で共有しない）
  const [client, ssr] = useMemo(() => {
    const ssr = ssrExchange({ isClient: typeof window !== 'undefined' });
    const client = createClient({
      url: 'https://api.example.com/graphql', // SSR でも解決できる絶対 URL
      exchanges: [cacheExchange, ssr, fetchExchange],
      suspense: true,
      fetchOptions: () => {
        const token = getAuthToken();
        return {
          headers: { authorization: token ? `Bearer ${token}` : '' },
        };
      },
    });
    return [client, ssr] as const;
  }, []);

  return (
    <UrqlProvider client={client} ssr={ssr}>
      {children}
    </UrqlProvider>
  );
}
```

```tsx
// app/layout.tsx（Server Component のまま）
import { Providers } from './providers';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

```tsx
// components/user-profile.tsx（使う側も Client Component）
'use client';

import { useQuery, gql } from '@urql/next';

const UserQuery = gql`
  query GetUser($id: ID!) {
    user(id: $id) { id name email }
  }
`;

// suspense: true なので、読み込み中は親の <Suspense> の fallback が表示される
export function UserProfile({ id }: { id: string }) {
  const [{ data, error }] = useQuery({ query: UserQuery, variables: { id } });

  if (error) return <p role="alert">ユーザーを読み込めませんでした</p>;
  return <p>{data?.user?.name}</p>;
}
```

**出典**:
- [urql Docs](https://commerce.nearform.com/open-source/urql/docs/) (nearform / urql公式)
- [urql Docs: Server-side Rendering - Next.js](https://github.com/urql-graphql/urql/blob/main/docs/advanced/server-side-rendering.md#nextjs) (urql公式ドキュメントのソース、`@urql/next` の `UrqlProvider` + `ssrExchange` を `'use client'` の layout で使う構成) ※2026-10-25 確認
- [Apollo Client Docs](https://www.apollographql.com/docs/react) (Apollo公式)

**バージョン**: urql 4+（App Router では @urql/next を併用）, Apollo Client 3+
**確信度**: 中
**最終更新**: 2026-10-25

---

### 2. Fragment Colocation でコンポーネントが必要なフィールドを宣言する

GraphQL フラグメントを、そのデータを使うコンポーネントと同一ファイルに定義する。
上位コンポーネントがフィールドを推測する必要がなくなる。

**根拠**:
- コンポーネントが必要なデータをコード上で宣言的に表現できる
- 上位コンポーネントがフィールドを追加・削除する際のリグレッションを防げる
- GraphQL Code Generator が Fragment から型を生成するため、型安全性が向上する
- Relay の思想を基盤としており、大規模アプリでのデータ管理に有効
- GraphQL Code Generator の `client-preset` では Fragment Masking が既定で有効。子コンポーネントは展開済みの型ではなく `FragmentType<typeof ...>` で受け取り、`useFragment()` で展開する。親が自分で使うフィールド（`key` に使う `id` など）は親のクエリでも選択する

**コード例**:
```tsx
// components/UserCard/UserCard.tsx
import { graphql } from '@/gql';
import { useFragment, type FragmentType } from '@/gql/fragment-masking';

// このコンポーネントが必要なフィールドをフラグメントで宣言
// 変数名は生成される型名（UserCard_UserFragment）と衝突しない名前にする
export const UserCardFragment = graphql(`
  fragment UserCard_User on User {
    id
    name
    avatarUrl
    role
  }
`);

interface Props {
  user: FragmentType<typeof UserCardFragment>;
}

export function UserCard(props: Props) {
  // useFragment は React Hook ではなく、マスクされた型を展開する関数
  const user = useFragment(UserCardFragment, props.user);
  return (
    <div>
      <img src={user.avatarUrl} alt={user.name} />
      <p>{user.name}</p>
      <span>{user.role}</span>
    </div>
  );
}
```

```tsx
// components/TeamPage/TeamPage.tsx
'use client';

import { useQuery } from '@urql/next';
import { graphql } from '@/gql';
import { UserCard } from '@/components/UserCard/UserCard';

const TeamPageQuery = graphql(`
  query TeamPage {
    team {
      members {
        id              # 親が key に使うので親のクエリでも選択する
        ...UserCard_User  # UserCard が必要なフィールドだけを取得
      }
    }
  }
`);

export function TeamPage() {
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

**Data Masking（Apollo 3.12+ / Relay）**:

Fragment Colocation をさらに強制するのが Data Masking。有効化すると、**コンポーネントは自分のフラグメントで宣言したフィールドにしかアクセスできない**。ネットワークレスポンスに他のコンポーネントが要求した `avatarUrl` が含まれていても、そのフィールドを宣言していないコンポーネントには `undefined` が返る。これにより「フラグメントを書き忘れても動く」という穴を型安全で塞ぐ。

```tsx
// Apollo Client 3.12+: @unmask ディレクティブで親からフラグメントにアクセス
// 子コンポーネントのフラグメントを親クエリに展開
const TeamPageQuery = graphql(`
  query TeamPage {
    team {
      members {
        ...UserCard_User @unmask  # データマスキングを解除して型アクセス
      }
    }
  }
`);
```

**出典**:
- [Apollo Docs: Fragments](https://www.apollographql.com/docs/react/data/fragments/) (Apollo公式)
- [urql Docs: Fragments](https://commerce.nearform.com/open-source/urql/docs/basics/core-logic/) (urql公式)
- [GraphQL Code Generator: client-preset - Fragment Masking](https://the-guild.dev/graphql/codegen/plugins/presets/preset-client#fragment-masking) (The Guild公式、既定で有効・`FragmentType` / `useFragment()` での受け渡し) ※2026-10-25 確認
- [GraphQL Fragments: Let Each Component Own Its Data](https://dev.to/grimicorn/graphql-fragments-let-each-component-own-its-data-5359) (dev.to、Data Masking の解説と導入タイミングの判断基準) ※2026-06-25 fetch

**出典引用**（追加）:
> "a component can only access fields it explicitly requested in its own fragment. Even if the network response contains avatarUrl because another component requested it, your component gets undefined unless it declared the field itself."
> ([GraphQL Fragments: Let Each Component Own Its Data](https://dev.to/grimicorn/graphql-fragments-let-each-component-own-its-data-5359), セクション "Data Masking for Type Safety") ※2026-06-25に実際にfetch成功

**バージョン**: Apollo Client 3.12+（Data Masking）, urql 4+, GraphQL 16+
**確信度**: 高
**最終更新**: 2026-10-25

---

### 3. GraphQL Code Generator で型とフック（またはドキュメント）を自動生成する

GraphQL スキーマとクエリファイルから TypeScript の型を自動生成する。
手書きの型定義は GraphQL スキーマとの乖離が生じるため使わない。
TypeScript 7.1+ では Content Mappers で `*.graphql` を生成ファイル無しに直接 import する構成も選べる（beta 検証段階。下記）。

**根拠**:
- スキーマ変更時に型エラーが即座に検出される
- TypeScript 7.1 の Content Mappers（`tsconfig.json` の `contentMappers` + マッパー側 `package.json` の `typescript.contentMapper`）で `graphql-codegen` コアを外部プロセスから呼べば、codegen 出力ファイルを作らずに型解決できる。外部プロセスが任意コードを実行するため `tsc --runExternalCode` が必須で、信頼できるマッパーのみ使う（Zenn 単一記事の検証、7.1.0-beta 前提）
- `client-preset` プラグインで Fragment の型も自動生成される
- `graphql-tag` の代わりに Codegen の `graphql()` 関数を使うことで型推論が働く
- CI でスキーマとのチェックを自動化できる

**コード例**:
```ts
// codegen.ts (設定ファイル)
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
- [Next.js で TypeScript7.1 の Content Mappers を使い、*.graphql ファイルを直接使用する](https://zenn.dev/sora_kumo/articles/typescript7-content-mappers) (Zenn、codegen 出力無しの代替構成) ※2026-10-22 fetch
  > "外部プロセスが任意のコードを実行できるため、安全機能としてコンパイル時には `--runExternalCode` フラグの指定が必須となっています。"
  > (セクション "Content Mappers の仕組み")

**バージョン**: @graphql-codegen/cli 5+, @graphql-codegen/client-preset 4+（Content Mappers 構成は TypeScript 7.1+）
**確信度**: 高（codegen 本体）/ 中（Content Mappers 構成は単一記事・beta）
**最終更新**: 2026-10-25

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
import { useMutation } from 'urql';
import { graphql } from '@/gql';
import type { CreatePostInput } from '@/gql/graphql';

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
```

```tsx
// Apollo Client での楽観的更新（urql とは別のクライアントの例）
import { useMutation } from '@apollo/client/react';

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
**最終更新**: 2026-10-25

---

## 関連プラクティス

- [`api-client/caching.md`](./caching.md) - キャッシュ戦略全般
- [`api-client/type-safety.md`](./type-safety.md) - 型生成ツール
- [`api-client/error-handling.md`](./error-handling.md) - GraphQL エラー処理
