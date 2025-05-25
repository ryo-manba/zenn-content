---
title: "Next.js の Server Actions で引数を渡すときは bind を使おう"
emoji: "🐏"
type: "tech"
topics: ["nextjs", "react", "フロントエンド"]
published: true
publication_name: "cybozu_frontend"
---

この記事では、Next.js の Server Actions にフォームの入力値以外の情報を渡す方法について紹介します。

## Server Actions とは

Next.js の [Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations#binding-arguments) は、クライアントサイドのイベント（例えばフォームの送信やボタンのクリック）をトリガーとして、サーバーサイドの関数を呼び出す機能です。この機能はプログレッシブエンハンスメントをサポートしており、JavaScript が読み込まれていない状況や無効になっている場合でもフォームを送信することができます。

### 使用例

`'use server'` をファイルの先頭に追加して、サーバー側で実行したい関数を定義します。

```tsx:actions.ts
"use server";

export async function myAction(formData) {
  // データベースへのアクセスなどの処理
  // ...
}
```

Server Actions をフォームで使用する際には、`action` 属性にその関数を指定します。

```tsx
"use client";
import { myAction } from "./actions";

export default function ClientComponent() {
  return (
    <form action={myAction}>
      <button type="submit">Add to Cart</button>
    </form>
  );
}
```

このように Server Actions を使用したフォームを実装することができます。

## フォームの入力値以外の情報を送信する方法

フォームからサーバーに入力値以外の情報を送信する場合、どのように実装すればよいでしょうか。例えば、ユーザー情報を更新する際には、ログインしているユーザーの ID のような追加データが必要になる場合があります。

### 従来のアプローチ：input type hidden

一般的な方法として、[`<input type="hidden">`](https://developer.mozilla.org/ja/docs/Web/HTML/Element/input/hidden) を使用するアプローチがあります。この方法では、画面に表示されない隠しフィールドを通じて、データをサーバーに送信します。
Server Actions を使用する際も、以下の例のようにフォーム内に隠しフィールドを設定してデータを送信することができます。

```tsx
"use client";
import { myAction } from "./actions";

export default function ClientComponent() {
  return (
    <form action={myAction}>
      {/* 画面には表示されない隠しフィールドを設定する */}
      <input type="hidden" name="user-id" value="userId-1" />
      <button type="submit">Update User</button>
    </form>
  );
}
```

隠しフィールドで送信した情報は、`formData` オブジェクトを介して、サーバー側の関数で受け取ることができます。

```tsx:actions.ts
const myAction = async (formData) => {
  // hidden に設定した id を取得する
  const id = formData.get("user-id");
  console.log(id); // userId-1
}
```

### 問題点

この方法には、いくつかの問題があります。

#### hidden パラメータは書き換えできる

Chrome の DevTools 上や OWASP ZAP のようなプロキシツールを用いることで、`hidden` パラメータを書き換えることができます。これにより、意図しないデータの送信が行われる可能性があります。

#### 機密データの露出

DevTools などでソースコードを確認すると、`input type="hidden"` に設定された値も表示されてしまいます。そのため、機密性の高い情報には適していません。

![DevToolsで表示したinput type="hidden"属性を持つ要素"](/images/server-actions-bind/hidden-devtools.png)

## bind を使用する

Next.js ではこの問題を解決するために [`bind`](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations#passing-additional-arguments) というメソッドを使用することができます。

`bind` は Server Component と Client Component のどちらでも使用することが可能です。
また、プログレッシブエンハンスメントと互換性があるため、JavaScript が無効になっている環境でも機能します。

2024/2/26 追記: `bind` を使用すると `name` は `$ACTION_2:1` のように変数名とは異なる形式で表示される一方で、`value` 属性の内容はブラウザ上で直接確認できてしまいます。
また、`bind` で設定された値は JavaScript が無効な環境では、ブラウザ上で値を書き換えることが可能になってしまうため、セキュリティ上のリスクを生じることに注意が必要です。

![DevToolsで表示したbind属性を使用した要素](/images/server-actions-bind/bind-devtools.png)

```tsx
"use client";

import { updateUser } from "./actions";

export function UserProfile({ userId }) {
  // bind で userId を渡す
  const updateUserWithId = updateUser.bind(null, userId);

  return (
    // フォームの action 属性に bind した関数を渡す
    <form action={updateUserWithId}>
      <input type="text" name="name" />
      <button type="submit">Update User Name</button>
    </form>
  );
}
```

サーバー側の関数には、`formData` オブジェクトとは別にバインドした引数を受け取るように定義する必要があります。

```tsx
"use server";

// bind で渡した引数を受け取る
export const updateUser = async (userId: string, formData: FormData) => {
  console.log("userId", userId); // userId-1
  // ...
};
```

### Controlled コンポーネントとの組み合わせ

State を使用して入力値を管理する [Controlled コンポーネント（制御されたコンポーネント）](https://ja.react.dev/reference/react-dom/components/input#controlling-an-input-with-a-state-variable)と Server Actions を組み合わせて使用する場合にも `bind` を使用することができます。
例えば、 `input` 要素を拡張したカスタムコンポーネントをフォームに組み込む場合は次のように実装できます。

```tsx:my-input.tsx
type Props = {
  value: string;
  onChange: (e: React.ChangeEvent<HTMLInputElement>) => void;
};

// 本来はスタイルを設定するなどの処理を行う
export const MyInput = ({ value, onChange }: Props) => {
  return <input type="text" value={value} onChange={onChange} />;
};
```

```tsx:client-component.tsx
"use client";

export function ClientComponent() {
  const [value, setValue] = useState("");
  // bind を使用して state を渡す
  const actionWithState = updateUser.bind(null, value);

  return (
    <form action={actionWithState}>
      <MyInput value={value} onChange={(e) => setValue(e.target.value)} />
      <button type="submit">Submit</button>
    </form>
  );
}
```

この構成では、`MyInput` コンポーネントの状態を呼び出し元が `useState` で管理し、`bind` を用いてそのステート値をサーバー側で実行される関数に渡しています。これにより、Controlled コンポーネントと Server Actions の機能を組み合わせることが可能です。

## まとめ

2024/2/26 追記:
`bind` を使用することで、`<input type="hidden">` に比べると多少はセキュリティが向上しますが、JavaScript を無効にした場合はブラウザ上で書き換えることが可能になってしまうため、セキュリティ上の問題があります。詳しくは [Server Actions にユーザ操作されたくないデータは渡さない](https://zenn.dev/moozaru/articles/c3bfd1a7e3c004) をご参照ください。

## 参考

https://nextjs.org/docs/app/api-reference/functions/server-actions#binding-arguments
https://nextjs.org/learn/dashboard-app/mutating-data#4-pass-the-id-to-the-server-action
https://zenn.dev/moozaru/articles/c3bfd1a7e3c004
