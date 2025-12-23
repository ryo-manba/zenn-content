---
title: "Markdown を見た目のままレビューできるツールを作った"
emoji: "📝"
type: "tech"
topics: ["markdown", "review", "cli"]
published: true
---

## はじめに

Spec Driven Development の普及により、コーディングエージェントを使って仕様書や実装計画などの Markdown を作成する機会が増えてきました。

通常のコードであれば、[difit](https://zenn.dev/yoshiko/articles/reviewit-for-local-code-review) を使うことで、ローカルでも差分を確認しながら簡単にレビューできます。一方で Markdown の場合、平文の diff では表やリストの構造が把握しづらく、レビューしにくいという課題があります。

さらに、修正してほしい箇所を AI に伝えようとすると、行番号を調べたり、該当部分を引用したりと、細かな手作業が発生しがちです。

## 何を作ったか

こうした課題を解決するために、Markdown のレビューに特化したツール [md-review](https://www.npmjs.com/package/md-review) を作りました。

![md-review のスクリーンショット](/images/md-review-for-markdown-review/screenshot.png)

md-review では、Markdown をレンダリング結果のまま確認しながら、特定の範囲にコメントを付け、その内容を AI に渡しやすい形式でコピーできます。

## 使い方

```sh
npx md-review
```

カレントディレクトリ以下の Markdown ファイルを対象にレビュー画面が起動します。

```sh
# docs ディレクトリ配下を対象にする
npx md-review docs/
```

特定のディレクトリを対象にしたい場合は、引数で指定できます。

### コメント機能

![フィードバック機能のデモ](/images/md-review-for-markdown-review/demo.gif)

プレビュー画面でテキストを選択してコメントを追加すると、サイドバーにコメントが表示されます。コピーボタンを押すと、次のような形式でコピーされます。

```text
specification.md:L42-48
バリデーションエラー時の挙動を追記して
```

ファイルパスと行番号付きでコピーされるため、Claude Code や Cursor にそのまま貼り付けるだけで、修正箇所を具体的に伝えられます。

なお、コメントは `localStorage` に保存されるため、外部に送信されることはありません。

## おわりに

AI が生成したドキュメントをレビューする機会は、今後さらに増えていくと考えています。md-review が、その際の手間を減らすツールとして役立てば幸いです。

改善提案やフィードバックお待ちしています。Star もいただけると嬉しいです！

https://github.com/ryo-manba/md-review
