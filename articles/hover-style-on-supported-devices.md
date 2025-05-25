---
title: "タッチデバイスでホバースタイルを回避する方法"
emoji: "🕊️"
type: "tech"
topics: ["CSS", "React"]
published: true
published_at: 2024-08-16 11:00
publication_name: "cybozu_frontend"
---

:::message
この記事は、[CYBOZU SUMMER BLOG FES '24](https://cybozu.github.io/summer-blog-fes-2024/) (Frontend Stage) DAY 16 の記事です。
:::

タッチデバイスではホバー状態が存在しないため、iOS などではタップ時にホバースタイルが適用されて、タップ後もスタイルが維持されることがあります。このような予期しない挙動は、ユーザー体験に悪影響を与える可能性があります。

この記事ではホバーが有効なデバイスを判定し、スタイルを適用する方法を紹介します。

## メディアクエリを使用する

ホバーが有効なデバイスかどうかを判定するために、[`hover`](https://developer.mozilla.org/docs/Web/CSS/@media/hover) や [`any-hover`](https://developer.mozilla.org/docs/Web/CSS/@media/any-hover) メディアクエリを使用できます。

- `@media (hover: hover)`: 主な入力デバイスがホバーに対応している場合に適用される。（例: デスクトップやノート PC）
- `@media (any-hover: hover)`: 入力デバイスのいずれかがホバーに対応している場合に適用される。（例: マウスが接続されているタッチデバイス）

@[stackblitz](https://stackblitz.com/edit/hover-media-query-demo?embed=1&file=styles.css)

この例では、`hover` や `any-hover` が無効なデバイスでは打ち消し線を表示しています。
Chrome を利用している場合は、[Devtool のデバイスモード](https://developer.chrome.com/docs/devtools/device-mode)を利用して、タッチデバイスをエミュレートし、ホバーの挙動を確認できます。

ただし、これらのメディアクエリを使用する際には注意が必要です。

例えば、iPad にトラックパッドやマウスを接続した場合、デバイスにはタッチとマウスカーソルの2つの入力デバイスが存在します。通常、主な入力モードはタッチであるため、 `hover` メディアクエリは適用されませんが、`any-hover` メディアクエリは有効になります。さらにブラウザや OS によって主となる入力デバイスの扱いが異なるため、さらに複雑になることがあります。

結果として、以下のような不整合が生じることがあります。

- `hover` を使用した場合
  - タッチデバイス: ホバースタイルが適用されない
  - マウスカーソル: ホバースタイルが適用されない（望ましくない）
- `any-hover` を使用した場合
  - タッチデバイス: ホバースタイルが適用される（望ましくない）
  - マウスカーソル: ホバースタイルが適用される

現状ではメディアクエリのみでこれらの不整合を避けることはできません[^1]。
ユーザーの操作環境に応じて、ホバースタイルをより厳密に制御したい場合は、JavaScript を使用する方法があります。

[^1]: CSSWG ではホバー可能なデバイスでのみ適用されるメディアクエリの[提案](https://github.com/w3c/csswg-drafts/issues/7544)がありますが、今のところ進展はないようです。

## JavaScript でホバーの状態を判定する

[Pointer Events API](https://developer.mozilla.org/docs/Web/API/Pointer_events) を利用することで、マウスやタッチデバイスの操作を検知し、適切なホバースタイルを適用できます。

Pointer Events は、マウス、タッチ、ペンなどのポインティングデバイスに対して発生する DOM イベントです。このイベントを基に、ホバーの状態を管理するカスタムフックを作成できます。

```tsx:useHover.ts
import { useState, DOMAttributes } from "react";

const useHover = () => {
  const [isHovered, setIsHovered] = useState(false);

  let hoverProps: DOMAttributes<Element> = {};

  // ポインターが要素内に移動したときの処理
  hoverProps.onPointerEnter = (e: React.PointerEvent<Element>) => {
    // ポインターの種類がタッチデバイスの場合は、ホバーを適用しない
    if (e.pointerType === "touch") {
      return;
    }
    setIsHovered(true);
  };

  // ポインターが領域外に移動したときの処理
  hoverProps.onPointerLeave = (e: React.PointerEvent<Element>) => {
    // ポインターの種類がタッチデバイスの場合は、ホバーを適用しない
    if (e.pointerType === "touch") {
      return;
    }
    setIsHovered(false);
  };

  return {
    hoverProps,
    isHovered,
  };
};
```

このコードでは、ホバー状態を管理しています。タッチデバイスが利用されている場合はホバー状態を変更させません。
これにより、タッチデバイスとマウスデバイスの切り替えが可能な場合でも、適切にホバースタイルを当てることができます。

このフックは次のように使用できます。

```tsx
export default function Page() {
  const { hoverProps, isHovered } = useHover();

  return (
    <button
      // ホバー状態を判定するためのイベントハンドラーを追加する
      {...hoverProps}
      style={{ backgroundColor: isHovered ? "pink" : "lightgray" }}
    >
      Hover me!
    </button>
  );
}
```

この例では、ホバー状態に応じてインラインスタイルを適用していますが、[`data` 属性](https://developer.mozilla.org/docs/Learn/HTML/Howto/Use_data_attributes)を利用してスタイルを分離することも可能です。

@[stackblitz](https://stackblitz.com/edit/react-hover-handler?embed=1&file=src%2FApp.tsx)

### おまけ: React Aria の `useHover` フック

さらに高度なホバーの制御が必要な場合は、React Aria の [`useHover`](https://react-spectrum.adobe.com/react-aria/useHover.html) が便利です。
このフックを利用すると、要素に `disabled` が付与されている場合にはホバーの判定を無効化したり、[iOS 13 以前で発生していたタッチイベント後にマウスイベントが発生する不具合](https://bugs.webkit.org/show_bug.cgi?id=214609)や、[Portal を介したイベント伝播の問題](https://github.com/facebook/react/issues/19637#issuecomment-850594683)などのより複雑なユースケースにも対応できます。

詳しくは [Building a Button Part 2: Hover Interactions](https://react-spectrum.adobe.com/blog/building-a-button-part-2.html) が参考になるので、興味がある方はご覧ください。

## まとめ

多用なデバイスが存在する現在では、メディアクエリだけではホバースタイルを適切に制御できない場合があります。そのようなケースでは、JavaScript や React Aria などのユーティリティを活用するのも一つの手段です。
ただし、多くの場合はメディアクエリで対応できるため、実装コストとのバランスを考慮して適切な方法を選択できると良さそうですね。

## 参考

- [hover - CSS: Cascading Style Sheets | MDN](https://developer.mozilla.org/docs/Web/CSS/@media/hover)
- [any-hover - CSS: Cascading Style Sheets | MDN](https://developer.mozilla.org/docs/Web/CSS/@media/any-hover)
- [Are Hover Events Extinct? | Design Shack](https://designshack.net/articles/css/are-hover-events-extinct/)
- [Finally, a CSS only solution to :hover on touchscreens | by Mezo Istvan | ITNEXT](https://itnext.io/finally-a-css-only-solution-to-hover-on-touchscreens-c498af39c31c)
- [Building a Button Part 2: Hover Interactions – React Spectrum Blog](https://react-spectrum.adobe.com/blog/building-a-button-part-2.html)
- [Pointing the way forward  |  Blog  |  Chrome for Developers](https://developer.chrome.com/blog/pointer-events)
- [Pointer events - Web APIs | MDN](https://developer.mozilla.org/docs/Web/API/Pointer_events)
