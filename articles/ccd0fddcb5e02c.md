---
title: "React Ariaを用いてアクセシブルなReactコンポーネントを作成する"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React", "a11y"]
published: true
---

## React Ariaとは？

[React Aria](https://react-spectrum.adobe.com/react-aria/)は、Adobeから提供されるオープンソースのライブラリです。Reactでアクセシブルなコンポーネントを作成するためのフックとユーティリティを提供しています。

キーボードとマウスのインタラクション、スクリーンリーダーのサポート、ARIA属性などに関する動作を抽象化することで、開発者は実装の詳細を気に留めることなく、ユーザー体験にフォーカスを当てることができます。

## パッケージのインストール

https://www.npmjs.com/package/react-aria

React Ariaをインストールするには以下のコマンドを使用します。

```bash
yarn add react-aria
```

特定のフックだけをインストールすることも可能です。
例えばbuttonコンポーネントが必要な場合は以下のコマンドでインストールできます。

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

## React Ariaの基本的な使い方

React Ariaは、React Hooksを通してアクセシビリティ対応とユーザーインタラクションの実装をサポートします。

その中でも特筆すべきは、React Ariaが**直接的なレンダリングを提供しない**という点です。つまり、コンポーネントのDOM構造は開発者自身で定義し、React Ariaの各フックから提供されるDOMのプロパティを適切な要素に適用する必要があります。

この設計により、DOMの構造を完全に制御することができます。その結果、自分でスタイリングを制御したり、必要なDOM要素を自由に追加することが可能になります。

React Ariaの使用法について、Buttonコンポーネントを例に見てみましょう。

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

コンポーネント内で、`useRef`を用いてDOMノードへの参照（ref）を作成します。
ここでは、HTMLのボタン要素を参照するために`HTMLButtonElement`を指定します。

```tsx
const { buttonProps } = useButton(props, ref);
```

次に、`useButton`を呼び出し、コンポーネントからのpropsと作成したrefを渡します。フックからは、適切な要素に適用すべきDOMプロパティのセットを含む`buttonProps`オブジェクトが返されます。

```tsx
return (
  <button {...buttonProps} ref={ref}>
    {props.children}
  </button>
);
```

最後に、ボタン要素をレンダリングします。ここで`buttonProps`をスプレッド演算子を用いてボタン要素に適用し、refを設定します。そして、コンポーネントが受け取る子要素をボタンの中にレンダリングします。

一見するととてもシンプルに見えますが、この一連の処理により、アクセシビリティ対応が実現されています。

`buttonProps`の中身を見てみるとその理由が明らかになります。

```tsx
// https://github.com/adobe/react-spectrum/blob/main/packages/%40react-aria/button/src/useButton.ts
return {
    isPressed, // Used to indicate press state for visual
    buttonProps: mergeProps(additionalProps, buttonProps, {
      'aria-haspopup': props['aria-haspopup'],
      'aria-expanded': props['aria-expanded'],
      'aria-controls': props['aria-controls'],
      'aria-pressed': props['aria-pressed'],
      onClick: (e) => {
        if (deprecatedOnClick) {
          deprecatedOnClick(e);
          console.warn('onClick is deprecated, please use onPress');
        }
      }
    })
  }
```

上記のプロパティの中には以下のようなものが含まれます：

- `'aria-haspopup'`: ボタンがポップアップメニューやサブメニューを制御していることを示す。
- `'aria-expanded'`: ボタンが制御するコンテンツが展開または収縮している状態を示す。
- `'aria-controls'`: ボタンが制御する要素のIDを指定する。
- `'aria-pressed'`: ボタンが押されている（選択されている）状態を示す。
- `onClick`: ボタンがクリックされた時に実行される処理。

これらのプロパティはReact Ariaが提供する`useButton`を使用することで自動的に設定され、スクリーンリーダーやキーボード操作などによるアクセシビリティも実現しています。

このように、React Ariaを用いることで、アクセシビリティのベストプラクティスに従いながら、ユーザー体験を最大化することが可能となります。

## React Ariaを用いた高度なコンポーネント作成

React Ariaを活用してより複雑なコンポーネントの作成方法も見ていきましょう。
具体的な例として、タブコンポーネントの作成を取り上げます。

タブコンポーネントでは、現在選択されているタブを示す状態管理が必要になります。
そこで、Adobeが提供している状態管理ライブラリ、[React Stately](https://react-spectrum.adobe.com/react-stately/index.html)を導入します。

タブコンポーネントに関しては、`useTabListState`というフックが提供されています。
このフックを使用することで状態管理を意識することなく実装できます。

使用例を以下に示します。

```tsx
import { useRef } from 'react';
import { useTabList } from 'react-aria';
import { Item, useTabListState } from 'react-stately';
import type { TabListStateOptions } from '@react-stately/tabs';

export const Tabs = <T extends object>(props: TabListStateOptions<T>) => {
  const state = useTabListState(props);
  const ref = useRef(null);
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

// 使用例
<Tabs aria-label="User Menu">
  <Item key="profile" title="Profile">
    プロフィール
  </Item>
  <Item key="settings" title="Settings">
    設定
  </Item>
</Tabs>
```

上記のTabコンポーネントは、十字キーでのタブ間の移動や、デフォルトで選択するタブの設定、Tabキーでのフォーカスなどの便利な機能が自動的に提供されます。
これらの機能の状態管理を`useState`や`useContext`を使用して、自分で行うのは結構大変そうですが、`useTabListState`を活用することでとてもシンプルに実装できています。

ただし、今の実装では呼び出し元で直接React StatelyのItemを呼び出す必要があるため、実装の詳細を理解する必要が生じてしまいます。それは避けたいです。
そこで、Itemをラップしたコンポーネントを用意したいところですが、残念なことに現状ではサポートされていないようです。

https://github.com/adobe/react-spectrum/discussions/4270

しかしながら、2023年2月に作成されたReact AriaのRFCによれば、Itemコンポーネントをラップできるような実装が進行中とのことなので、将来的にはより使いやすくなるかもしれません。

https://github.com/adobe/react-spectrum/blob/main/rfcs/2023-react-aria-components.md#collections

なお、現時点でも`getCollectionNode`を継承することで、Itemコンポーネントをラップすることは可能ですが、公式が提供している手法ではないため、利用する場合は影響がないか確認する必要があります。

```tsx
type TabItemProps = React.ComponentProps<typeof Item>;

// Itemの代わりに使用できるようになる
export const TabItem = (props: TabItemProps) => {
  return <Item {...props} />;
};
// @ts-ignore
TabItem.getCollectionNode = Item.getCollectionNode;
```

2023-07-01 追記：React StatelyとReact Server Componentを組み合わせる場合の注意点

React Server Componentの特性として、クライアントコンポーネントの子要素としてサーバーコンポーネントを使用することが可能です。しかしReact AriaのItemコンポーネントを使用する場合は子要素もクライアントコンポーネントにしないといけません。

これは、Itemコンポーネントの内部で`getCollectionNode`ジェネレータ関数を使用して子要素を探索しているからだと思われます。

## まとめ

React Ariaを活用してわかったことは以下の３点です。

- 手軽にアクセシブルなコンポーネントが実現できる。
- コンポーネント単位で導入できるためコストが低い。
- 状態管理が必要な場合は、React Statelyと組み合わせて使うと良い。

ただし、React AriaやReact StatelyのAPIを理解するためにも一定の学習コストが伴います。
そのようなコストも考慮した上で、導入を検討してみてください。

React Ariaを活用して、手軽にアクセシブルなWebアプリケーション開発を行いましょう！

## 参考

https://react-spectrum.adobe.com/react-aria/getting-started.html
https://react-spectrum.adobe.com/react-aria/useTabList.html
https://react-spectrum.adobe.com/react-stately/useTabListState.html