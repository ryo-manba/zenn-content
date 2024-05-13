---
title: "拡張性に優れた React Aria のコンポーネント設計"
emoji: "😺"
type: "tech"
topics: ["react", "reactaria", "フロントエンド"]
published: false
publication_name: "cybozu_frontend"
---

[React Aria Components](https://react-spectrum.adobe.com/react-aria/components.html) は Adobe によって提供されている Headless UI コンポーネントライブラリです。振る舞いや国際化に, アクセシビリティに関する機能を備えており、Button や Input, TextField, Label などのシンプルな要素から、DatePicker や ComboBox などの様々なコンポーネントが提供されています。

今回は React Aria Components の設計について紹介します。

## React Aria Components のコンポーネントの設計

React Aria Components の API は[コンポジションを中心に設計](https://react-spectrum.adobe.com/react-aria/advanced.html#contexts)されています。これにより、パターン間で共通のコンポーネントを共有することも、個別に使用することも可能です。なお、コンポジションについては [React Component Composition Explained](https://felixgerschau.com/react-component-composition/) や [Next.js の Composition Patterns](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns) が参考になります。

例として [Select コンポーネント](https://react-spectrum.adobe.com/react-aria/Select.html)を見てみましょう。

```tsx
<Select>
  <Label>Permissions</Label>
  <Button>
    <SelectValue />
    <span>▼</span>
  </Button>
  <Popover>
    <ListBox>
      <ListBoxItem>Read Only</ListBoxItem>
      <ListBoxItem>Edit</ListBoxItem>
      <ListBoxItem>Admin</ListBoxItem>
    </ListBox>
  </Popover>
</Select>
```

Select コンポーネントは Label, Button, Popover, ListBox, ListBoxItem などのコンポーネントを組み合せて構成されています。これらは、それぞれ独立して利用可能ですが、組み合わせることでより複雑な UI を構築することが可能となっています。

ただし、この例だけを見るとコンポーネント単位でインポートして扱えた方が便利に感じるかもしれません。例えば [React Select](https://react-select.com/home) では次のように呼び出すことができます。

```tsx
const options = [
  { value: 'chocolate', label: 'Chocolate' },
  { value: 'strawberry', label: 'Strawberry' },
  { value: 'vanilla', label: 'Vanilla' }
]

const MyComponent = () => (
  <Select options={options} />
)
```

既にスタイルが適用されておりカスタマイズの要件がある程度決まっているような場合は、ライブラリの学習コストも抑えられるため、このようなアプローチは有効だと思います。しかし、Headless UI コンポーネントでは、各要素に独自のスタイルを適用することが前提とされてます。そのため、コンポジションを利用してコンポーネントを個別に配置することで、スタイリングを柔軟に行えることは利点になります。

仮に React Aria Components の Select コンポーネントが同様のインターフェースで提供されていた場合、Button や Label, Popover に表示される選択項目のリストなど、それぞれの要素に独自のスタイルを当てることが困難になります。次のように各要素に対応する props を渡してスタイリングすることになるでしょう。

```tsx
<Select buttonStyle={} labelStyle={} listBoxStyle={}>
```

コンポジションを利用することで各要素に直接スタイルを当てることが可能であり、CSS ライブラリに依存しない実装が可能です。

```tsx
<Select>
  <Label className={style.label}>Permissions</Label>
  <Button className={style.button}>
    <SelectValue />
    <span>▼</span>
  </Button>
  ...
```

## コンポジションを実現する仕組み

プレーンテキストを入力するためのシンプルな [TextField](https://react-spectrum.adobe.com/react-aria/TextField.html) コンポーネントを例に、コンポジションがどのように実現されているのかを見ていきます。

```tsx
import {TextField, Label, Input} from 'react-aria-components';

<TextField>
  <Label>First name</Label>
  <Input />
</TextField>
```

この構成では、Label と Input が TextField の子要素として配置されています。

![TextField を画面に反映させた様子。](/images/react-aria-component-design/text-field.png)

表面上は単なる `<input>` と `<label>` が配置されているように見えますが、実際には TextField によって関連付けが行われています。
具体的には `<label>` の `for` 属性に `<input>` の `id` が紐付けられ、`<input>` には `<label>` の `id` が `aria-labelledby` 属性として設定されます。

![TextFieldのHTML。id と for 属性が反映されている。](/images/react-aria-component-design/text-field-html.png)

このような連携には、Label と Input が TextField 内でどのように扱われているかが関係しています。

### プロパティの管理とコンテキストの活用

Label や Input の実装には、[`useContextProps`](https://react-spectrum.adobe.com/react-aria/advanced.html#usecontextprops) が使用されており、次のようにコンテキストから props と ref を取得しています。

```tsx
// Label の場合は LabelContext から props と ref を取得する
[props, ref] = useContextProps(props, ref, LabelContext);

// Input の場合は InputContext から props と ref を取得する
[props, ref] = useContextProps(props, ref, InputContext);
```

`useContextProps` は、[`mergeProps`](https://react-spectrum.adobe.com/react-aria/mergeProps.html) を使用してローカルの props と ref を親コンポーネントからコンテキスト経由で提供されたものとマージします。

```tsx
// contextProps には、親コンポーネントから提供された props が含まれている
const mergedProps = mergeProps(contextProps, props);
```

この仕組みにより、コンポーネントが単体で使用される場合は直接 props を使用し、親要素からコンテキストが提供されている場合は、そのコンテキストから props を取得します。これによって、Label や Input などのコンポーネントは単体で使用される場合と、他のコンポーネントと組み合わせて使用される場合のどちらにも適応可能です。

props を直接渡すこともでき、次のように `id` や `for` 属性を明示的に指定することが可能です。

```tsx
<TextField>
  <Label id="test-label" htmlFor="test-input">
    First name
  </Label>
  <Input id="test-input" />
</TextField>
```

props で指定した値が `id` や `for` 属性として反映されていることが確認できます。

![TextFieldのHTML。id と for 属性が定義通りに反映されている。](/images/react-aria-component-design/text-field-html-props.png)

### Provider を介したコンテキストの提供

TextField 内でどのようにして Label と Input に対して props と ref が提供されているかも見てみましょう。

```tsx
// useTextField を使用して、Label や Input などに対応する props を生成する
let {labelProps, inputProps} = useTextField({...props}, inputRef);

// hooks で生成した props を Provider に渡す
<Provider
  values={[
    [LabelContext, {...labelProps, ref: labelRef}],
    [InputContext, {...inputProps, ref: inputOrTextAreaRef}],
  ...
  ]}>
   {children}
 </Provider>
```

[useTextField](https://react-spectrum.adobe.com/react-aria/useTextField.html) を利用して props を生成し、Provider を介して、各子要素にコンテキストが提供されています。この仕組みにより、TextField 配下に配置された Label や Input は、TextField から提供されたコンテキストを利用して props を取得することが可能となっています。

このタグをネストさせる構成は、組み込みの HTML タグや Jeremy が [HTML Web Components](https://adactio.com/journal/20618) と呼んでいるものに類似しており、拡張性を重視したコンポーネント設計が実現されているようです。

## Context を活用したカスタマイズ

コンポジションの設計により Context が活用できるため、コンポーネントの内部に独自の機能を組み込むことも可能になります。

次のコードでは、DatePicker コンポーネントに選択した日付をクリアするボタンを独自に追加しています。

```tsx
import {DatePickerContext} from 'react-aria-components';

// 選択した日付をクリアするカスタムコンポーネント
function DatePickerClearButton() {
  // DatePicker の Context にアクセスする
  const state = useContext(DatePickerStateContext);
  return (
    <Button onPress={() => state.setValue(null)}>
      Clear
    </Button>
  );
}

<DatePicker>
  <Label>Date</Label>
  <Group>
    ...
    {/* DatePicker 配下に配置することでコンテキストへのアクセスが可能になる　*/}
    <DatePickerClearButton />
  </Group>
  ...
</DatePicker>
```

Context には、コンポーネントの内部の状態が提供されており、`children` から `useContext` を使用してアクセスすることができます。DatePicker の `children` に配置された `DatePickerClearButton` は、この Context にアクセスすることで、選択された日付をクリアする機能を提供しています。

`<input type='date'/>` のように単一のコンポーネントで完結する設計の場合、内部の実装をカスタマイズすることが困難ですが、このように Context を介して内部の状態にアクセスし、要素を追加できることによって、独自のカスタマイズが可能となっています。

## まとめ

React Aria Components のコンポーネント設計に関するポイントをまとめます。

- React Aria では、拡張性を意識して、コンポジションを中心に設計されている
- コンポーネントは独立して使う場合と組み合わせて使う場合のどちらにも対応している
- コンテキストを活用することで、内部の状態にアクセスし、カスタマイズすることが可能

これらの特徴により、React Aria Components は Headless UI コンポーネントとして、柔軟なカスタマイズが可能なライブラリとなっています。興味ある方はぜひ一度試してみてください。

https://react-spectrum.adobe.com/react-aria/components.html

## 参考

- [React Aria Components – React Aria](https://react-spectrum.adobe.com/react-aria/components.html)
- [Architecture – React Spectrum](https://react-spectrum.adobe.com/architecture.html)
- [Advanced Customization – React Aria](https://react-spectrum.adobe.com/react-aria/advanced.html)
- [React Component Composition Explained | Felix Gerschau](https://felixgerschau.com/react-component-composition/)
- [Rendering: Composition Patterns | Next.js](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns)
- [Adactio: Journal—HTML web components](https://adactio.com/journal/20618)
