---
title: "Next.js14で導入されたReact Taint APIsを試してみた"
emoji: "🔏"
type: "tech"
topics: ["nextjs", "react", "フロントエンド"]
published: false
publication_name: "cybozu_frontend"
---

Next.js の公式ブログの [How to Think About Security in Next.js](https://nextjs.org/blog/security-nextjs-server-components-actions) という記事で Next.js 14 で導入される [React Taint APIs](https://react.dev/reference/react/experimental_taintObjectReference) について紹介されていました。

https://nextjs.org/blog/security-nextjs-server-components-actions

この記事では、Next.js 14 で React Taint APIs 実際に試してみて、どのような機能なのかを確認してみたいと思います。

## React Taint APIs とは？

React Taint APIs とは、React が experimental バージョンで提供する新しいセキュリティ保護機能の一つです。このAPIを使用することで、誤って Client Component にセキュリティ上の重要なデータが渡されることを防げるようになります。

具体的には、以下の2つの API が提供されています。

- [experimental_taintObjectReference](https://react.dev/reference/react/experimental_taintObjectReference)
- [experimental_taintUniqueValue](https://react.dev/reference/react/experimental_taintUniqueValue)

`taintObjectReference` は、特定のオブジェクト単位で、`taintUniqueValue` は、パスワードやトークンなどの一意な値単位で管理できます。

これらの APIは、データベースから直接生のユーザーデータやセンシティブな情報を取得し、それを安全に React Server Components で表示するための新たなユースケースをサポートするために導入されました。従来、このようなデータの加工やフィルタリングはバックエンドで行われていましたが、Server Components の導入により、この責任がフロントエンドにも及ぶようになったことが背景にあります。

React Taint APIs の目的は、このようなセンシティブなデータの取り扱いミスを防ぐことです。
例えば、アクセストークンやユーザーのパスワードなどを、うっかりクライアントサイドで表示してしまうリスクを減少させることができます。

## React Taint APIs を試してみる

### 前準備

Next.js の最新の canary バージョンでセットアップします。(canaryで試しましたが、2023/10/27 時点では Next.js 14 でも利用可能です)

```bash
npx create-next-app@canary --ts
```

next.config の experimental セクションに taint オプションを true に設定します。

```ts
const nextConfig = {
  experimental: {
    taint: true,
  },
};
```

次に API を使ってユーザー情報を取得し、その情報をクライアントサイドにそのまま渡すシチュエーションを想定します。Prisma を使用して、特定のユーザー情報を取得する関数を作成します。

```ts
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();

const getUser = async (userId: number) => {
  const user = await prisma.user.findUnique({
    where: {
      id: userId,
    },
  });
  return user;
};
```

上記で作成した関数を利用して、取得したユーザー情報を Client Component に渡すコードを作成します。

```tsx
import { getUser } from "./get-user";
import { InfoCard } from "./info-card";

type Props = {
  userId: number;
};

export const Profile = async ({ userId }: Props) => {
  const user = await getUser(userId);
  if (!user) {
    return <div>User not found.</div>;
  }
  return <InfoCard user={user} />;
};
```

ユーザー情報を表示する InfoCard コンポーネントは Client Component にしています。

```tsx
"use client";

import type { User } from "./types";

type Props = {
  user: User;
};

export const InfoCard = ({ user }: Props) => {
  return (
    <div>
      <h2 className="text-lg">User info</h2>
      <ul className="text-base list-disc list-inside">
        <li>Name: {user.name}</li>
        <li>Email: {user.email}</li>
        <li>Password: {user.password}</li>
      </ul>
    </div>
  );
};
```

上記のコードを実行すると、ユーザーの情報が表示されることが確認できます。

![ユーザーの情報が表示される](/images/react-taint-apis/user-info.png)

しかし、現状だとユーザーのパスワードも表示されるため、セキュリティ上のリスクが生じます。
そこで、 React Taint APIs を使用して、パスワードがクライアントに渡らないようにしてみましょう。

### taintObjectReference

`taintObjectReference` を使用して、特定のオブジェクトを Client Component でアクセスできないようにします。`getUser` で取得した生のユーザーデータに `taintObjectReference` を使用することで、そのオブジェクトが Client Component に渡った際にエラーが発生するようになります。

`taintObjectReference` の引数には以下を渡します。

- 第一引数：Client Component に渡した場合に表示するカスタムのエラーメッセージ
- 第二引数：対象のオブジェクト

```ts
export const getUserWithTaintObjectReference = async (userId: number) => {
  const user = await getUser(userId);
  if (!user) {
    return null;
  }

  // Client Component からのアクセスを禁止する
  taintObjectReference(
    "Do not pass user data to the client", 
    user
  );
  return user;
};
```

`user` オブジェクトに対して関数を実行しただけなので、これで本当に保護されているのか想像がつきませんね。そこで、実際にエラーが表示されることを確認してみます。

```tsx
import { getUserWithTaintObjectReference } from "./get-user";
import { InfoCard } from "./info-card";

type Props = {
  userId: number;
};

export const Profile = async ({ userId }: Props) => {
  const user = await getUserWithTaintObjectReference(userId);
  if (!user) {
    return <div>User not found.</div>;
  }
  return <InfoCard user={user} />;
};
```

すると以下のような結果になりました。エラーメッセージが適切に表示されています。

![taintObjectReference のエラーメッセージ](/images/react-taint-apis/taint-object-reference-error.png)

処理の流れを整理すると、`Profile`（Server Component）でユーザー情報を取得し、その後、`InfoCard`（Client Component）にその情報を渡しています。
この時、`getUserWithTaintObjectReference` で `taintObjectReference` を使用しているため、`InfoCard` に渡した際にエラーが発生します。

また以下のように、コンポーネントの階層が深くなっても、保護されたオブジェクトがClient Component に渡ったタイミングでエラーが発生します。

```tsx
import { getUserWithTaintObjectReference } from "./get-user";
import { InfoCard } from "./info-card";
import type { User } from "./types";

type Props = {
  userId: number;
};

const NestedComponent = ({ user }: { user: User }) => {
  return <InfoCard user={user} />;
}

export const Profile = async ({ userId }: Props) => {
  const user = await getUserWithTaintObjectReference(userId);
  if (!user) {
    return <div>User not found.</div>;
  }
  // 階層を深くしてもエラーが発生する
  return <NestedComponent user={user} />
};
```

![taintObjectReference のエラーメッセージ](/images/react-taint-apis/taint-object-reference-error.png)

このように、 `taintObjectReference` を使用することで、オブジェクト全体を保護してクライアントへ不正な引数渡しを防ぐことができます。

しかし、注意点として、オブジェクトのスプレッド演算子や分割代入を使用した場合などには、保護の効果が失われてしまいます。
例えば、以下の実装では、スプレッド演算子を用いてオブジェクトをコピーしているため、`taintObjectReference` の保護機能は無効となります。

```tsx
export const Profile = async ({ userId }: Props) => {
  const user = await getUserWithTaintObjectReference(userId);
  if (!user) {
    return <div>User not found.</div>;
  }
  return <InfoCard user={{...user}} />
};
```

![ユーザーの情報が表示される](/images/react-taint-apis/user-info.png)

これにより `taintObjectReference` を使用する際には、上述のような状況に注意して使用することが重要だと分かります。

### taintUniqueValue

`taintUniqueValue` を使用すると特定の値だけを Client Component からのアクセスを防ぐことができます。例えば、アクセストークンやユーザーのパスワードなどをクライアントに渡したくない場合に役立ちます。

引数には、以下の値を渡します。

第一引数：Client Component に渡した場合に表示するカスタムのエラーメッセージ
第二引数：対象の値を保護する期間。プロパティの場合は、オブジェクト自体を渡す。
第三引数：対象の値

以下の例では、ユーザー情報を取得し、その中の `password` プロパティだけを Client Component からのアクセスを禁止にしています。

```ts
export const getUserWithTaintUniqueValue = async (userId: number) => {
  const user = await getUser(userId);
  if (!user) {
    return null;
  }

  // user.password のみを client component からアクセスできないようにする
  taintUniqueValue(
    "Do not pass password to the client",
    user,
    user.password
  );
  return user;
};
```

上記の関数を用いてユーザー情報を取得した際、ユーザー全体をクライアント側に渡すとエラーが生じます。

```tsx
import {　getUserWithTaintUniqueValue } from "./get-user";
import { InfoCard } from "./info-card";

type Props = {
  userId: number;
};

export const Profile = async ({ userId }: Props) => {
  const user = await getUserWithTaintUniqueValue(userId);
  if (!user) {
    return <div>User not found.</div>;
  }
  return <InfoCard user={user} />;
};
```

![taintUniqueValue のエラーメッセージ](/images/react-taint-apis/taint-unique-value-error.png)

エラーメッセージを見ると、`taintUniqueValue` で設定したカスタムエラーメッセージが表示されていることが分かります。

ただし、Client Component からのアクセスを防ぎたいのは、パスワードだけなのでその他のプロパティにはアクセスしたいです。そこで、以下のように `password` プロパティを除いて他の情報を渡す場合、エラーは発生しません。

```tsx
import { getUserWithTaintUniqueValue } from "./get-user";
import { InfoCard } from "./info-card";

type Props = {
  userId: number;
};

export const Profile = async ({ userId }: Props) => {
  const user = await getUserWithTaintUniqueValue(userId);
  if (!user) {
    return <div>User not found.</div>;
  }
  // password を除いて渡す
  const { password, ...safeUser } = user;
  return <InfoCard user={safeUser} />;
};
```

![パスワード以外の情報が表示される](/images/react-taint-apis/user-info-no-password.png)

`taintUniqueValue` を実行したパスワード以外の情報は、Client Component に渡せることが確認できました。このように一部のプロパティだけを保護する時に役立つのが `taintUniqueValue` です。

ただし、これにも注意点があります。以下のように `toUpperCase` などのメソッドを使用して、値に変更を加えてしまうとその効果を失ってしまいます。

```tsx
export const Profile = async ({ userId }: Props) => {
  const user = await getUserWithTaintUniqueValue(userId);
  if (!user) {
    return <div>User not found.</div>;
  }
  const dirty = {
    ...user,
    password: user.password.toUpperCase(),
  };
  return <InfoCard user={dirty} />;
};
```

![パスワードが全て大文字で表示される](/images/react-taint-apis/user-info-password-uppercase.png)

`toUpperCase` が適用されて全て大文字でパスワードが表示されてしまいました。このように `taintUniqueValue` を使用する際には、値の変更に注意を払う必要があるようです。

## まとめ

React Taint APIs は、React が experimental バージョンで提供する新しいセキュリティ機能の一つです。ドキュメントにも記載されていますが、あくまでもセキュリティの補助機能であり、使う際には注意が必要です。それは実際に試してみても分かる通り、API の使用方法によっては保護の効果が失われてしまうことがあるからです。

今回実際に試してみて、セキュアなデータの取り扱いには、データアクセス層を分離し、補助的に React Taint APIs 活用するというやり方が良いように感じました。Next.js は現在この機能に関するドキュメントを公開していないため、今後のアップデートにも期待したいです。

今回使用したソースコードは公開しているので、興味ある方は参考までにご覧ください。

https://github.com/ryo-manba/next-taint-apis