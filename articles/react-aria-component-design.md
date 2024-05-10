---
title: "React Aria のコンポーネント設計"
emoji: "😺"
type: "tech"
topics: ["react", "reactaria", "フロントエンド"]
published: false
publication_name: "cybozu_frontend"
---

[React Aria Components](https://react-spectrum.adobe.com/react-aria/components.html) は Adobe によって提供されている Headless UI コンポーネントライブラリです。振る舞いや国際化に, アクセシビリティに関する機能を備えており、Button や Input, TextField, Label などのシンプルな要素から、DatePicker や ComboBox などの様々なコンポーネントが提供されています。

今回は React Aria Components の設計について紹介します。

## React Aria Components のコンポーネントの設計

React Aria Components の API は[コンポジションを中心に設計](https://react-spectrum.adobe.com/react-aria/advanced.html#contexts)されています。これにより、パターン間で共通のコンポーネントを共有することも、個別に使用することも可能です。
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

他の UI ライブラリと比較した場合、コンポーネント単位でインポートして扱えた方が便利に感じるかもしれません。例えば [React Select](https://react-select.com/home) では次のように呼び出すことができます。

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

既にスタイルが適用されていて、カスタマイズの要件がある程度決まっているような場合ではこのようなアプローチは有効だと思います。しかし、Headless UI コンポーネントの場合は、各要素に対して独自のスタイルを適用することが前提となっているため、コンポーネントを分けていることで、スタイリングを柔軟に行うことができます。

仮に React Aria Components の Select コンポーネントが同様のスタイルで提供されていた場合、Button や Label, Popover に表示される選択項目のリストなど、それぞれの要素に独自のスタイルを当てることが困難になります。次のように各要素に対応する props を渡してるスタイリングすることになるでしょう。

```tsx
<Select buttonStyle={} labelStyle={} listBoxStyle={}>
```

これに対し、コンポジションを利用することで各要素に直接スタイルを当てることが可能になるCSS ライブラリに依存しない実装ができます。

```tsx
<Select>
  <Label className={style.label}>Permissions</Label>
  <Button className={style.button}>
    <SelectValue />
    <span>▼</span>
  </Button>
  ...
```

ではこれらの実装がどのような仕組みによってコンポジションが成り立っているかを見ていきます。

### コンポジションを実現するための仕組み

プレーンテキストを入力するためのシンプルな [TextField](https://react-spectrum.adobe.com/react-aria/TextField.html) コンポーネントを例に、コンポジションがどのように実現されているのかを見ていきます。

```tsx
import {TextField, Label, Input} from 'react-aria-components';

<TextField>
  <Label>First name</Label>
  <Input />
</TextField>
```

Label と Input を TextField の子要素として配置しています。

![TextField を画面に反映させた様子。](/images/react-aria-component-design/text-field.png)
`Label` の実装には、[`useContextProps`](https://react-spectrum.adobe.com/react-aria/advanced.html#usecontextprops) フックが使用されています。

```tsx
  [props, ref] = useContextProps(props, ref, LabelContext);
```

`Input` も同様に `useContextProps` が使用されています。

```tsx
  [props, ref] = useContextProps(props, ref, InputContext);
```

`useContextProps` は、ローカルの props と ref を、親コンポーネントからコンテキスト経由で提供されたものと [`mergeProps`](https://react-spectrum.adobe.com/react-aria/mergeProps.html) を使用してマージしてくれます。

```tsx
  // contextProps には、親コンポーネントから提供された props が含まれている
  let mergedProps = mergeProps(contextProps, props) as unknown as T;
```

この仕組みにより、コンポーネントは単体で使用される場合と他のコンポーネントと組み合わせて使用される場合のどちらともに対応可能です。

`Label` と `Input` に対して `TextField` がどのように props と ref を提供しているのかも確認してみましょう。TextField では Provider を介して、各子要素に props を提供しているようです。

```tsx
   <Provider
     values={[
       [LabelContext, {...labelProps, ref: labelRef}],
       [InputContext, {...inputProps, ref: inputOrTextAreaRef}],
     ...
     ]}/>
```

他にも ComboBox や Select などの複数のコンポーネントを組み合わせて構成することができるのは、このような設計によるものです。

TextField をもう一度見てみましょう。

```tsx
import {TextField, Label, Input} from 'react-aria-components';

<TextField>
  <Label>First name</Label>
  <Input />
</TextField>
```

この構成では、`<label>` に `for` 属性が自動で設定され、`<input>` の `id` が紐付けられます。さらに `<input>` には `<label>` の `id` が `aria-labelledby` 属性として設定されます。

![TextFieldのHTML。id と for 属性が反映されている。](/images/react-aria-component-design/text-field-html.png)

props を直接渡すことも可能なので `id` や `for`  を指定できます。

```tsx
<TextField>
  <Label id="test-label" htmlFor="test-input">
    First name
  </Label>
  <Input id="test-input" />
</TextField>
```

![TextFieldのHTML。id と for 属性が定義通りに反映されている。](/images/react-aria-component-design/text-field-html-props.png)

このようにタグをネストさせる構成は組み込みの HTML タグや [Jeremy が HTML Web Components](https://adactio.com/journal/20618) と呼んでいるものに類似しており、拡張性を意識したコンポーネント設計となっています。

### Context を活用したカスタマイズ

コンポジションの設計によって、Context を活用することが可能になりました。
これにより、コンポーネントの内部に独自の機能を組み込むことも容易に行うことができます。

例として、DatePickerコンポーネントに選択した日付をクリアするボタンを追加する場面を見てみましょう。

```tsx
// DatePicker の状態を Context から取得する
function DatePickerClearButton() {
  let state = useContext(DatePickerStateContext);

  return (
    <Button
      slot={null}
      aria-label="Clear"
      onPress={() => state.setValue(null)}>
      ✕
    </Button>
  );
}

<DatePicker>
  <Label>Date</Label>
  <Group>
    <DateInput>
      {segment => <DateSegment segment={segment} />}
    </DateInput>
    {/* DatePicker 配下に配置することでコンテキストへのアクセスが可能になる　*/}
    <DatePickerClearButton />
    <Button>▼</Button>
  </Group>
　　 ...
</DatePicker>
```

この実装では、DatePicker の内部に `DatePickerClearButton` を配置することで、DatePicker の `Context` にアクセスし、選択された日付をクリアする機能を提供しています。
`<input type='date'/>` のように単一のコンポーネントで完結する設計の場合、内部の実装をカスタマイズすることが困難ですが、このように `Context` を介して内部の状態にアクセスし、要素を配置できることによって、独自のカスタマイズが可能となっています。

## まとめ

- React Aria では、拡張性を意識して、コンポジションを中心に設計されています。
- コンポーネントは独立して使う場合と組み合わせて使う場合のどちらにも対応しています。
- コンテキストを活用することで、内部の状態にアクセスし、カスタマイズを行うことが可能です。

## 参考

- [Advanced Customization – React Aria](https://react-spectrum.adobe.com/react-aria/advanced.html)
- [Architecture – React Spectrum](https://react-spectrum.adobe.com/architecture.html)
- [Adactio: Journal—HTML web components](https://adactio.com/journal/20618)
- [React Select](https://react-select.com/home)