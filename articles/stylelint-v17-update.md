---
title: "Stylelint v17 の主な変更点：ESM 移行と CSS Nesting 対応など"
emoji: "👔"
type: "tech"
topics: ["stylelint", "css", "frontend"]
published: true
published_at: "2026-01-21 09:00"
---

## はじめに

2026年1月15日に [Stylelint v17](https://github.com/stylelint/stylelint/releases/tag/17.0.0) がリリースされました。前回のメジャーアップデートは [v16（2023年12月）](https://github.com/stylelint/stylelint/releases/tag/16.0.0)のため、およそ2年ぶりのメジャーアップデートとなります。

v17 は breaking changes を含むリリースで、公式の[マイグレーションガイド](https://stylelint.io/migration-guide/to-17/)でも **ESM への移行**と、**CSS Nesting / 詳細度 / ベンダープレフィックス周りのルール近代化**が大きなテーマとして挙げられています。

この記事では、Stylelint v17 の主な変更点を解説します。

## 1. ESM への完全移行

Stylelint v16 では CommonJS (CJS) と ES Modules (ESM) の両方を提供していましたが、v17 では **ESM のみ**を提供するようになりました。この変更は v16 でCommonJS APIが非推奨化され、[次のメジャーバージョンで削除する](https://github.com/stylelint/stylelint/blob/da7ce21272d34928f40bbc68c1e01be1c076bb8a/docs/migration-guide/to-16.md?plain=1#L5-L7) と予告されていた流れを受けたものです。

あわせて、[Node.js 20.19.0](https://nodejs.org/en/blog/release/v20.19.0) 以上が必須になりました。Node.js 20.19.0 以降では `require(esm)` がデフォルトで有効になっているため、CJS のコードから `require('stylelint')` の形式で読み込んでいた場合でも、多くのケースではそのまま動作する見込みです。

## 2. CSS Nesting 対応の改善（ルールの近代化）

v17 では CSS Nesting と “modern” な functional pseudo-class（`:is()` / `:where()` / `:has()` など）を前提に、セレクター関連ルールが広く見直されました。

### 2.1 `selector-max-*` が "書いた見た目" に近い判定に

`selector-max-*` はセレクター内の特定要素（クラス、ID、属性など）の個数を制限するルールです。v17 では、その「数え方」が変わりました。

#### なぜ変更が必要だったのか

v16 以前の動作には直感に反する問題がありました。

```css
#foo {
  & a {}
}
```

`selector-max-id: 0`（ID セレクターを禁止）を設定した場合、v16 では以下のようになります。

- `#foo` に対してエラー（ID セレクター 1 個）→ これは期待どおり
- `& a` に対して**もエラー**（内部的に ID を含むと判定）→ **これが問題**

`& a` はソースコード上に ID セレクターを含まないのに、エラーが報告されます。これは開発者にとって分かりにくい動作です。

#### v16 と v17 の違い

v16 では以下の 2 つの処理を行っていました。

1. **desugar（展開）**: `#foo { & a {} }` を内部的に `#foo a {}` に変換してからカウント
2. **functional pseudo-class の分離評価**: `:has()` や `:not()` 内のセレクターを別コンテキストとして扱う

この実装は `:not()` が唯一の functional pseudo-class だった時代には妥当でしたが、`:is()`, `:has()`, `:where()` が標準化された現代では不適切です。v17 では **desugar と分離評価を廃止**し、ソースコードをそのまま検査するようになりました（[PR #8913](https://github.com/stylelint/stylelint/pull/8913)）。

#### 具体例

**例1: `.foo:has(.bar) {}`** に `selector-max-class: 1` を適用

- **v16**: パス（`:has(.bar)` を別コンテキストとして評価、外側は `.foo` の 1 個のみ）
- **v17**: エラー（セレクター全体で `.foo` + `.bar` = 2 個とカウント）

**例2: ネストセレクター** に `selector-max-id: 0` を適用

```css
#foo {
  & a {}
}
```

- **v16**: `#foo` と `& a` の両方でエラー（desugar して `#foo a` として評価するため）
- **v17**: `#foo` のみエラー（desugar しないため、`& a` には ID がない）

v17 の動作のほうが「ソースコードの見た目」と一致しており、直感的となっています。

### 2.2 詳細度（specificity）計算の変更

2.1 がセレクターの「数え方」の変更だったのに対し、こちらは**詳細度の計算方法**の変更です。次のルールに関連しています。

- `no-descending-specificity`
- `selector-max-specificity`

#### なぜ変更が必要だったのか

v16 以前では、ネストセレクターの詳細度計算が CSS 仕様と異なる方法で行われていました。これにより、結果が不安定で開発者が混乱する原因になっていました。

v17 では [CSS Nesting 仕様](https://drafts.csswg.org/css-nesting/#nest-selector)に準拠した計算方法に変更されました。具体的には、ネスティングセレクター（`&`）の詳細度は、親セレクターの中で**最も高い詳細度**を採用します。これは `:is()` と同じ挙動です。

#### 具体例

```css
#foo,
.bar {
  & a {}
}
```

このネストは `:is(#foo, .bar) a` と同等に解釈されます。`#foo` の詳細度は `1,0,0`、`.bar` は `0,1,0` なので、`&` は最大値の `1,0,0` を採用し、`& a` 全体の詳細度は `1,0,1` になります。

### 2.3 `fix` オプションのデフォルトが `strict` に

v17 から [`fix`](https://stylelint.io/user-guide/options/#fix) のデフォルトが `strict` になりました。`strict` モードでは、ソースに構文エラーがある場合は自動修正しません。v16 以前のデフォルトだった `lax` モードでは、ネストの閉じカッコが欠けているなど構文エラーがある状態で fix すると、壊れた修正結果になる問題が報告されていました（[issue #8881](https://github.com/stylelint/stylelint/issues/8881)）。

v16 以前の挙動に戻したい場合は `--fix=lax` を指定しましょう。

## 3. Node.js API / プラグイン開発者向けの変更

Node.js API やプラグイン開発に関わる変更点です。

### 3.1 `result.output` が削除

`stylelint.lint()` の戻り値から `output` が削除されました。`fix: true` で修正後のコードを取得する場合は `result.code` を使う必要があります。

### 3.2 `report` の扱いがより厳密に（位置指定が曖昧だとエラー）

プラグイン側で `utils.report` を使っている場合、位置指定が曖昧な呼び出しは v17 でエラーになります。`node` を渡したうえで、`index/endIndex` や `start/end` を整合する形で渡す必要があります。

### 3.3 GitHub formatter の削除

CLI の `--formatter github` や、Node.js API で `formatter: "github"` を指定していた場合、v17 で GitHub formatter が削除されました。代替として[コミュニティ製 formatter](https://www.npmjs.com/package/@csstools/stylelint-formatter-github) の利用が案内されています。

## まとめ

Stylelint v17 の要点をまとめます。

| 変更点 | 内容 |
|------|-----|
| **ESM 移行** | CommonJS API が削除。|
| **Node.js 要件** | 20.19.0 以上が必須 |
| **CSS Nesting 周り** | `selector-max-*` / `*-specificity` が仕様に合わせて近代化 |
| **`fix` オプション** | デフォルトが `strict` に（構文エラー時は fix しない） |

その他にもベンダープレフィックス関連ルールの変更や、一部オプションの削除などがあります。詳しくは[マイグレーションガイド](https://stylelint.io/migration-guide/to-17/)を参照してください。

## 参考

- [Migrating to 17.0.0 | Stylelint](https://stylelint.io/migration-guide/to-17/)
- [Release 17.0.0 · stylelint/stylelint](https://github.com/stylelint/stylelint/releases/tag/17.0.0)
