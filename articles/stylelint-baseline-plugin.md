---
title: "Stylelint で Baseline をチェックするプラグインを作った"
emoji: "👔"
type: "tech"
topics: ["css", "stylelint", "baseline"]
published: true
---

2025年2月に [ESLint が CSS のサポート](https://eslint.org/blog/2025/02/eslint-css-support/)を開始し、その中に Baseline をチェックするルールが追加されました。これを機に Stylelint でも同様の機能を提供するプラグインを作成しました。この記事ではその概要と使い方について解説します。

## 何を作ったか

Stylelint のプラグイン [stylelint-plugin-use-baseline - npm](https://www.npmjs.com/package/stylelint-plugin-use-baseline) を作成しました。

https://github.com/ryo-manba/stylelint-plugin-use-baseline

このプラグインは CSS が [Baseline](https://developer.mozilla.org/docs/Glossary/Baseline/Compatibility) の基準を満たしていない場合に警告を出します。

```css
a {
  accent-color: red;
  width: abs(20% - 10px);
}
```

上記のコードに対してリントを実行すると、次のようなメッセージが出力されます。

```shell
test.css
  2:3   ✖  Property "accent-color" is not a widely available baseline feature. (plugin/use-baseline)
  3:10  ✖  Type "abs" is not a widely available baseline feature. (plugin/use-baseline)

✖ 2 problems (2 errors, 0 warnings)
```

これは [accent-color](https://developer.mozilla.org/en-US/docs/Web/CSS/accent-color) と [abs()](https://developer.mozilla.org/en-US/docs/Web/CSS/abs) が Widely available の基準を満たしていないためです。

VS Code で [Stylelint 拡張](https://marketplace.visualstudio.com/items?itemName=stylelint.vscode-stylelint)を有効にしている場合は、エディター上でも警告がハイライトされます。

![VS Code 上でBaseline警告が表示されている様子](/images/stylelint-baseline-plugin/vscode-stylelint-baseline-warning.png)

## なぜ作ったか

ESLint が [CSS サポートを発表したブログ](https://eslint.org/blog/2025/02/eslint-css-support/)に次の一文があります。

> We feel that validation and enforcing baseline features are the minimum that a linter needs to support in 2025, and so we spent a lot of time making the no-invalid-at-rules, no-invalid-properties, and require-baseline rules as comprehensive as possible.

Baseline のチェックは2025年時点でリンターがサポートすべき**最低限の機能**と位置づけられています。そこで Stylelint でも同様の機能を提供すべく [issue](https://github.com/stylelint/stylelint/issues/8391) を起票しましたが、まずはユーザー需要の検証や組み込み機能の安全性を考慮し、プラグインとして実装することになりました。

## 使い方

導入手順と使い方についても簡単に紹介します。

### インストールする

```shell
npm install stylelint-plugin-use-baseline --save-dev
```

:::message
[stylelint](https://www.npmjs.com/package/stylelint) を事前にインストールしている必要があります
:::

### 設定を追加する

`.stylelintrc.js`にプラグインの設定を追加します。

```js
/** @type {import("stylelint").Config} */
export default {
  plugins: ["stylelint-plugin-use-baseline"],
  rules: {
    "plugin/use-baseline": [
      true,
      {
        // "widely" or "newly" or 年 (例: 2023)
        available: "widely",
      },
    ],
  },
};
```

`available` オプションには以下の3種類が設定できます。

- `widely` (default): 主要ブラウザで2.5年以上サポートされている機能のみ許可
- `newly`: 主要ブラウザで2.5年未満サポートされている機能も許可
- 年 (例：2023): その年までに Widely に加わった機能を許可

#### 年指定が便利なケース

年を指定すると、自サイトのユーザー環境に合わせて Baseline を細かく調整できます。
たとえば Google Analytics のブラウザレポートを [Google Analytics Baseline Checker](https://chrome.dev/google-analytics-baseline-checker/) にかけると、「Baseline 2021 なら 98 % のユーザーが対応」などのデータが取れます。この結果をもとに `available: 2021` のように設定すれば、自サイトに最適化されたブラウザ対応ポリシーを lint で強制できます。詳しい議論については ESLint の [eslint/css#78](https://github.com/eslint/css/issues/78) を参照ください。

### 実行する

```shell
> npx stylelint "src/**/*.css"

test.css
   2:3   ✖  Property "accent-color" is not a widely available baseline feature. (plugin/use-baseline)
   3:14  ✖  Value "stroke-box" of property "clip-path" is not a widely available baseline feature. (plugin/use-baseline)
   4:10  ✖  Type "abs" is not a widely available baseline feature. (plugin/use-baseline)
   6:1   ✖  At-rule "@view-transition" is not a widely available baseline feature. (plugin/use-baseline)
  10:2   ✖  Selector "has" is not a widely available baseline feature. (plugin/use-baseline)
  14:3   ✖  Selector "nesting" is not a widely available baseline feature. (plugin/use-baseline)
  18:9   ✖  Media condition "color-gamut" is not a widely available baseline feature. (plugin/use-baseline)

✖ 7 problems (7 errors, 0 warnings)
```

Property や Selector, At-rule など機能ごとに警告が表示されます。

---

より詳細な使用例は GoogleChromeLabs が作成した以下のリポジトリで確認できます。

https://github.com/GoogleChromeLabs/baseline-demos/tree/main/tooling/stylelint

## 振る舞いについて

このプラグインは、ESLintの [use-baseline](https://github.com/captainbrosset/eslint-css/blob/main/docs/rules/use-baseline.md) と同じインターフェースと挙動になるように設計されています。

[年指定できる機能](https://github.com/ryo-manba/stylelint-plugin-use-baseline/pull/7)や [Baseline データの不具合の修正](https://github.com/ryo-manba/stylelint-plugin-use-baseline/pull/15)などは、Baseline の DevRel である [rviscomi 氏](https://github.com/rviscomi) のご協力によるものです。お力添えに感謝します。

## まとめ

`stylelint-plugin-use-baseline` を導入すると、Stylelint でも Baseline のチェックが簡単に行えます。ESLint の `use-baseline` と同じ使い勝手なので移行もスムーズに行えます。ぜひお試しください！

フィードバックや改善提案は、[GitHub Issues](https://github.com/ryo-manba/stylelint-plugin-use-baseline/issues) までお気軽にお寄せください。

---

余談ですが、このプラグインは [web.dev](http://web.dev) の以下の記事で紹介されました🎉

https://web.dev/blog/whats-new-in-web-io2025?hl=en#baseline_in_linters%E2%80%94eslint_and_stylelint

https://web.dev/blog/baseline-digest-apr-2025#stylelint_adds_support_for_baseline
