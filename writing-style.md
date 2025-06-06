# Writing Style Guide

## 1. 文体・トーン

### ルール

- 日本語の「です・ます調」を用いる。
- 読者に敬意を払い、丁寧な語り口で書く。
- 主観的な意見や体験談を述べる場合は「私」を使う。
- 断定・推測・提案・意見表明を文脈に応じて使い分ける。

### 具体例

- 「この記事では、ReactのuseStateフックについて解説します。」
- 「私が実際に試したところ、次のような結果になりました。」
- 「この方法を使うことで、より簡単に状態管理ができるようになります。」

---

## 2. 構成

### ルール

- 記事は「導入」「本文（複数セクション可）」「まとめ」「参考文献」の順で構成する。
- 各セクションはMarkdownの見出し（`#`, `##`, `###`）で明確に区切る。
- 導入では記事の目的・背景を簡潔に述べる。
- まとめでは要点や今後の展望を記載する。

### 具体例

```markdown
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

...
```

---

## 3. コードブロック・差分

### ルール

- コード例は必ず言語指定を行う（例: `js`, `ts`, `sh`）。
- 差分を示す場合は `diff` 構文を使う。
- コード内の重要な部分はコメントで補足する。

### 具体例

```js
// カウンターの初期値を0に設定
const [count, setCount] = useState(0);
```

```diff
- const [count, setCount] = useState();
+ const [count, setCount] = useState(0);
```

---

## 4. 視覚要素・リスト・表

### ルール

- 手順や特徴は箇条書き（`-`）や番号付きリスト（`1.`）で整理する。
- 比較や設定項目はMarkdownテーブルで示す。
- 図やスクリーンショットは適切な位置に挿入し、キャプションを付ける。

### 具体例

```markdown
### 手順

1. プロジェクトを作成する
2. 必要なパッケージをインストールする
3. コードを編集する

### 設定比較

| 設定項目 | デフォルト | 推奨値 |
|----------|------------|--------|
| timeout  | 30         | 60     |
```

---

## 5. 強調・リンク・注意喚起・引用

### ルール

- 重要な語句は太字（`**text**`）やインラインコード（`` `code` ``）で強調する。
- 公式ドキュメントや参考記事にはMarkdownリンクを使う。
- 注意点や補足は `!>` 構文で明示する。
- 他資料からの引用は引用ブロック（`>`）を使う。

### 具体例

```markdown
**注意:** `useState` の初期値は必ず指定してください。

!> この機能は実験的です。実運用では注意してください。

> React公式ドキュメントより：「useStateは関数コンポーネントでのみ使用できます。」

[React公式ドキュメント](https://react.dev/)
```

---

## 6. 語彙・文法

### ルール

- 専門用語は正確に使い、初出時は簡単に説明する。
- 接続詞（「しかし」「また」「例えば」など）を使い、論理的な流れを明確にする。
- 導入表現（「この記事では」「〜について説明します」など）を使う。

### 具体例

- 「しかし、この方法にはいくつかの注意点があります。」
- 「例えば、useStateを使う場合は次のように記述します。」
- 「この記事では、Next.jsのISRについて説明します。」

---

## 7. フロントマター

### ルール

- 記事冒頭にYAML形式でメタデータを記載する。
- 必須項目：title, slug, published, tags, created, updated

### 具体例

```yaml
---
title: "ReactのuseStateフックを理解する"
slug: "react-usestate-basic"
published: true
tags: ["React", "フロントエンド"]
created: "2024-06-01"
updated: "2024-06-10"
---
```

---

## 8. 品質管理

### ルール

- `.textlintrc.json` のルールに従い、textlintで日本語校正を行う。
- 校正エラーは必ず修正する。

### 具体例

- 記事執筆後、`pnpm lint:txt` を実行し、指摘事項を修正する。
