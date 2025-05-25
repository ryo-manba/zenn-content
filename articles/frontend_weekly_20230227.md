---
title: "Remix 2.7.0 リリースなど : Cybozu Frontend Weekly (2024-02-27号)"
emoji: "👋"
type: "tech"
topics: [cybozufrontendweek, frontend]
published: true
publication_name: "cybozu_frontend"
---

こんにちは！サイボウズ株式会社フロントエンドエンジニアの[まっつー](https://twitter.com/ryo_manba)です。

## はじめに

サイボウズでは毎週火曜日にFrontend Weeklyという「一週間にあったフロントエンドニュースを共有する会」を社内で開催しています。

今回は、2024年2月27日のFrontend Weeklyで取り上げた記事や話題を紹介します。

## 取り上げた記事・話題

## input type=“date” の沼から、ライブラリを導入する意義を考える

https://tech.mirrativ.stream/entry/2024/02/20/100000

ブラウザ固有の不具合を回避するためにライブラリを選択することの意義について紹介されている記事です。不具合が解消される以前のブラウザを使用しているユーザーへの体験を考慮するという視点は大事ですね。

## You Don't Need Next.js

https://www.docswell.com/s/ashphy/KM1NQ6-you-dont-need-nextjs

Next.js 以外のフレームワークを使うべきか Next.js を使うべきかについて紹介されている記事です。新しいフレームワークを導入する上で、事前調査や何かあった時に自分達で対応していくという覚悟が必要ですね。

## Announcing web.dev for China

https://web.dev/blog/web-dev-for-china

web.dev と Chrome for Developers が中国の開発者向けにサイトを公開した告知記事です。
既にあるサイトのミラー版を .cn に公開することで、中国の開発者にもアクセスしやすくなるようです。

## probably meeting for some days during TPAC 2025 (2025-11-10..2025-11-14), Kobe, Japan

https://wiki.csswg.org/planning#section2025

W3C が年に一度開催している TPAC(Technical Plenary/Advisory Committee Meetings Week)が 2025年は神戸で開催されるようです。

## Remix Vite is Now Stable

https://remix.run/blog/remix-vite-stable

Remix v2.7.0 がリリースされ、Vite サポートや SPA モードが Stable になりました。
Basename や Cloudflare Pages のサポートなどが追加されています。

## 安全にミュータブル更新するためのJavaScriptライブラリ

https://github.com/unadlib/mutability

ミュータブルなオブジェクトを transactional に更新できるライブラリ。
オブジェクトの更新時に error が throw されると更新前の値にロールバックされるようです。

## JSDoc as an alternative TypeScript syntax

https://alexharri.com/blog/jsdoc-as-an-alternative-typescript-syntax

TypeScript の型定義を JSDoc で定義する方法を紹介しています。
TypeScript コンパイラーは JSDoc を解釈できるので、JavaScriptで型定義を行いたい場合はJSDocを使えます。

## Support for align-content in block and table layouts  |  Blog  |  Chrome for Developers

https://developer.chrome.com/blog/align-content?hl=en

Chrome 123 以降でブロックコンテナーとテーブルセルに対して align-content が使用できるようになりました。flex や grid を使用せずに中央揃えができて便利そうです。ただし仕様のステータスは [Editor's Draft](https://drafts.csswg.org/css-align/#align-justify-content) なので、主要ブラウザでの実装が揃うのはもう少し先の話かもしれません。

## #62439 Docs: Add CORS examples

https://github.com/vercel/next.js/pull/62439

Next.js のドキュメントに middleware と next.config で CORS の設定をする方法が追加されました。

## <input type=password >にはroleが無い！

https://zenn.dev/zozotech/articles/6bfc9cee222348

セキュリティの観点から role がないのではないかということを紹介している記事です。
input 属性一つとっても奥が深いですね。

## 米ホワイトハウス「将来のソフトウェアはメモリ安全になるべき」と声明発表。ソフトウェアコミュニティに呼びかけ

https://www.publickey1.jp/blog/24/post_294.html

テクノロジーコミュニティ全体としてメモリ安全なプログラミングへ取り組むべきという方向性が示されたようです。今後はメモリ安全とされるプログラミング言語が注目されるかもしれませんね。

## あとがき

サイボウズでは毎月、最終火曜日の 17 時から Frontend Monthly というイベントを YouTube Live で開催しています。その月のフロントエンド注目ニュースや、ゲストを呼んでの対談など、フロントエンドに関する発信をしていますので是非どうぞ！

https://cybozu.github.io/frontend-monthly/
