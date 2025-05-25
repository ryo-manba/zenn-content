---
title: "React Ariaを用いてアクセシブルなReactコンポーネントを作成する"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React", "a11y", "アクセシビリティ", "reactaria"]
published: true
---

## React Ariaとは？

[React Aria](https://react-spectrum.adobe.com/react-aria/) は、Adobe から提供されるオープンソースのライブラリです。React でアクセシブルなコンポーネントを作成するためのフックとユーティリティを提供しています。

キーボードとマウスのインタラクション、スクリーンリーダーのサポート、WAI-ARIA の設定などに関する動作を抽象化することで、開発者は実装の詳細を気にすることなく、ユーザー体験にフォーカスを当てることができます。

## パッケージのインストール

https://www.npmjs.com/package/react-aria

React Aria をインストールするには以下のコマンドを使用します。

```bash
yarn add react-aria
```

特定のフックだけをインストールすることも可能です。
例えば button コンポーネントが必要な場合は以下のコマンドでインストールできます。

```bash
yarn add @react-aria/button
```

インストールしたフックは、以下のように使用できます。

```tsx
// Monopackage
import { useButton } from 'react-aria';
// Individual packages
import { useButton } from '@react-aria/button';
```

## React Aria の基本的な使い方

React Aria は、React Hooks を通してアクセシビリティ対応とユーザーインタラクションの実装をサポートします。

その中でも特筆すべきは、React Aria が**直接的なレンダリングを提供しない**という点です。つまり、コンポーネントの DOM 構造は開発者自身で定義し、React Aria の各フックから提供される DOM のプロパティを適切な要素に適用する必要があります。

この設計により、DOM の構造を完全に制御することができます。その結果、自分でスタイリングを制御したり、必要な DOM 要素を自由に追加することが可能になります。

React Aria の使用法について、Button コンポーネントを例に見てみましょう。

```tsx
import { useRef, ReactNode } from 'react';
import { useButton, AriaButtonProps } from '@react-aria/button';

interface ButtonProps extends AriaButtonProps {
  children: ReactNode;
}

const Button = (props: ButtonProps) => {
  const ref = useRef<HTMLButtonElement>(null);
  const { buttonProps } = useButton(props, ref);

  return (
    <button {...buttonProps} ref={ref}>
      {props.children}
    </button>
  );
};

// 使用例
<Button onPress={() => alert('Button pressed!')}>
  Press me
</Button>
```

以下に詳細の解説を続けます。

```tsx
const ref = useRef<HTMLButtonElement>(null);
```

コンポーネント内で、`useRef` を用いて DOM ノードへの参照（ref）を作成します。
ここでは、HTML のボタン要素を参照するために `HTMLButtonElement` を指定します。

```tsx
const { buttonProps } = useButton(props, ref);
```

次に `useButton` を呼び出し、コンポーネントからの props と作成した ref を渡します。フックからは、適切な要素に適用すべき DOM プロパティのセットを含む `buttonProps` オブジェクトが返されます。

```tsx
return (
  <button {...buttonProps} ref={ref}>
    {props.children}
  </button>
);
```

最後にボタン要素をレンダリングします。ここで `buttonProps` をスプレッド演算子を用いてボタン要素に適用し、ref を設定します。そして、コンポーネントが受け取る子要素をボタンの中にレンダリングします。

一見するととてもシンプルに見えますが、この一連の処理によりアクセシビリティ対応が実現されています。

## `useButton` の内部実装

`useButton` がどのようにしてアクセシビリティ対応を実現しているかも少しだけ見てみましょう。

https://github.com/adobe/react-spectrum/blob/%40adobe/react-spectrum%403.34.0/packages/%40react-aria/button/src/useButton.ts#L69-L86

この処理では、`<a>` 要素や `<input>` 要素など `button` 要素以外でボタンを作成する場合設定を行っています。例えば、`tabindex` を付与することで、タブキーによるフォーカス移動を可能にしたり、`role="button"` の設定をしていることがわかります。他にも各要素ごとにプロパティが設定されていることが確認できますね。

https://github.com/adobe/react-spectrum/blob/%40adobe/react-spectrum%403.34.0/packages/%40react-aria/button/src/useButton.ts#L88-L97

この処理では、プレスイベントに対応する処理を設定しています。`usePress` フックを使用することで、異なるデバイスやブラウザでのユーザーインタラクションに対応するための処理が設定されます。詳細は [usePress のドキュメント](https://react-spectrum.adobe.com/react-aria/usePress.html)を参照してください。

このように React Aria を用いることでユーザー体験の向上が期待できます。

## React Aria を用いた高度なコンポーネント作成

React Aria を活用してより複雑なコンポーネントの作成方法も見ていきましょう。
具体的な例として、タブコンポーネントの作成を取り上げます。

タブコンポーネントでは、現在選択されているタブを示す状態管理が必要になります。
そこでAdobe が提供している状態管理ライブラリの [React Stately](https://react-spectrum.adobe.com/react-stately/index.html) を導入します。

タブコンポーネントを実装する際 `useTabListState` という便利なフックが提供されています。このフックを使うことで、状態管理を気にすることなくシンプルにタブの実装が可能になります。

以下は、`useTabListState` フックと React Aria の `useTabList` を組み合わせて、アクセシビリティ対応のタブコンポーネントを実装する例です。

```tsx:tab.tsx
import { useRef } from 'react';
import { useTabList } from 'react-aria';
import { Item, useTabListState } from 'react-stately';
import type { TabListStateOptions } from '@react-stately/tabs';

export const Tabs = <T extends object>(props: TabListStateOptions<T>) => {
  // React Statelyでタブリストの状態を管理
  const state = useTabListState(props);
  // DOM要素への参照
  const ref = useRef(null);
  // React Ariaでタブリストのプロパティを取得
  const { tabListProps } = useTabList(props, state, ref);
  return (
    <>
      <div {...tabListProps} ref={ref}>
        {[...state.collection].map((item) => (
          <Tab key={item.key} item={item} state={state} />
        ))}
      </div>
      <TabPanel key={state.selectedItem?.key} state={state} />
    </>
  );
};
```

このコンポーネントは、`useTabListState` で生成された `state` を基に `useTabList` によって得られた `tabListProps` と共に、タブリストの各アイテムをレンダリングします。`state.collection` を展開して各タブをレンダリングすることで、動的なタブリストの構築が可能です。

React Aria と React Stately を組み合わせて作成したこの Tabs コンポーネントは、以下のようにして使用することができます。

```tsx
export const MyTab = () => {
  return (
    <Tabs aria-label="User Menu">
      <Item key="profile" title="Profile">
        プロフィール
      </Item>
      <Item key="settings" title="Settings">
        設定
      </Item>
    </Tabs>
  );
};
```

上記の Tab コンポーネントは、十字キーでのタブ間の移動や、デフォルトで選択するタブの設定、タブキーでのフォーカスなどの便利な機能が提供されます。
これらの機能を `useState` や `useContext` などの状態管理で自力で実装するのは、苦戦しそうですが `useTabListState` を活用することでとてもシンプルに実装できていますね。

一つ問題点を挙げると、`Item` コンポーネントを直接使用することで、利用者側が実装の内部を意識しなければならない点が挙げられます。利用者側にとっては、ラップしたコンポーネントを用意することが望ましいと思われます。しかしながら、現状では `Item` コンポーネントのラップする機能はサポートされていないようです。以下のディスカッションでやり取りが行われています。

https://github.com/adobe/react-spectrum/discussions/4270

ただし、2023年2月に作成された React Aria の RFC によると `Item` コンポーネントをラップできるような実装が進行中とのことなので、将来的にはより使いやすくなるかもしれません。

https://github.com/adobe/react-spectrum/blob/main/rfcs/2023-react-aria-components.md#collections

なお、現時点でも`getCollectionNode` を継承することで、`Item` コンポーネントをラップすることは可能ですが、公式が提供している手法ではないため、利用する場合は影響がないか確認する必要があります。

```tsx
type TabItemProps = React.ComponentProps<typeof Item>;

// Itemの代わりに使用できるようになる
export const TabItem = (props: TabItemProps) => {
  return <Item {...props} />;
};
// @ts-ignore
TabItem.getCollectionNode = Item.getCollectionNode;
```

2023-07-01 追記：React Stately とReact Server Components を組み合わせる場合の注意点

React Server Components の特性として、クライアントコンポーネントの子要素としてサーバーコンポーネントを使用することが可能です。しかし React Aria の `Item` コンポーネントを使用する場合は子要素もクライアントコンポーネントにしないといけません。これはあくまでも予想ですが、`Item` コンポーネントの内部で `getCollectionNode` ジェネレータ関数を使用して子要素を探索しているからではないかと思われます。

## まとめ

React Aria を活用してわかったことは以下の３点です。

- 手軽にアクセシブルなコンポーネントが実現できる。
- コンポーネント単位で導入できるためコストが低い。
- 状態管理が必要な場合は、React Statelyと組み合わせて使うと良い。

ただし、React Aria や React Stately の API を理解するためにも一定の学習コストが伴います。
そのようなコストも考慮した上で、導入を検討してみてください。

React Aria を活用して、手軽にアクセシブルなWebアプリケーション開発を行いましょう！

## 参考

https://react-spectrum.adobe.com/react-aria/getting-started.html
https://react-spectrum.adobe.com/react-aria/useTabList.html
https://react-spectrum.adobe.com/react-stately/useTabListState.html
