---
title: "Bundle Analyzer で Server Components と Client Components のバンドルサイズを可視化する"
emoji: "🔍"
type: "tech"
topics: ['react', 'nextjs', 'フロントエンド']
published: true
publication_name: "cybozu_frontend"
---

この記事では、Next.js で Bundle Analyzer を導入して、Server Components と Client Components のバンドルサイズを以下のパターンで比較します。

- Server Components と Client Components の両方でライブラリを利用する
- Server Components のみでライブラリを利用する
- Composition Pattern を使用して Server Components のみでライブラリを利用する

比較には Node.js とブラウザの両方で動作する [date-fns](https://date-fns.org/) を利用します。

:::message
Bundle Analyzer の導入手順は公式ドキュメントを参考にしてください。

https://nextjs.org/docs/app/building-your-application/optimizing/bundle-analyzer
:::

## Server Components と Client Components の両方でライブラリを利用する

以下のコードは、Server Components と Client Components の両方で `date-fns` を利用しています。

```tsx:app/both-use-date-fns/page.tsx
import { ServerComponent } from "./server-component";
import { ClientComponent } from "./client-component";

export default function App() {
  return (
    <>
      <ServerComponent />
      <ClientComponent />
    </>
  );
}
```

```tsx:app/both-use-date-fns/server-component.tsx
import { format } from "date-fns";

export const ServerComponent = () => {
  const date = format(new Date(), "yyyy-MM-dd");
  return <div>Server Component: ${date}</div>;
};
```

```tsx:app/both-use-date-fns/client-component.tsx
"use client";
import { format } from "date-fns";

export const ClientComponent = () => {
  const date = format(new Date(), "yyyy-MM-dd");
  return (
    <>
      <div>Client Component: ${date}</div>
      <button onClick={() => alert("Hello!")}>Click me!</button>
    </>
  );
};
```

画面上では次のように表示されます。

![Server Components と Client Components の両方で `date-fns` を利用した画面](/images/next-bundle-analyzer/both-use-date-fns-page.png)

Bundle Analyzer を使用した結果は次のように表示されます。

![Server Components と Client Components の両方で `date-fns` を利用した Bundle Analyzer の結果](/images/next-bundle-analyzer/both-use-date-fns-bundle-analyzer.png)

`date-fns` の `format` 関数がクライアントに配信されているみたいですね。

![Server Components と Client Components の両方で `date-fns` を利用した Bundle Analyzer の結果（client-component.tsx)](/images/next-bundle-analyzer/both-use-date-fns-client.png)

右側を拡大してみると `date-fns` と比べて、とても小さいですが `client-component.tsx` が配信されていることが分かります。

## Server Components のみでライブラリを利用する

次に `date-fns` を Server Components のみで使用している場合を確認します。

```tsx:app/server-only-use-date-fns/client-component.tsx
"use client";

export const ClientComponent = () => {
  return (
    <>
      <div>Client Component</div>
      <button onClick={() => alert("Hello!")}>Click me!</button>
    </>
  );
};
```

Client Components から `date-fns` を削除しました。

![Server Components のみで `date-fns` を利用した画面](/images/next-bundle-analyzer/server-only-use-date-fns-page.png)

表示から Server Components のみが `date-fns` を使用していることが確認できます。

![Server Components のみで `date-fns` を利用した Bundle Analyzerの結果](/images/next-bundle-analyzer/server-only-use-date-fns-bundle-analyzer.png)

Bundle Analyzer の結果から、Client Components で `date-fns` を使用していないため、クライアントに配信されていないことが確認できます。Server Components で使用したライブラリはクライアントのバンドルサイズに影響を与えないことが分かりますね。

## Composition Pattern を使用して Server Components のみでライブラリを利用する

最後に [Composition Pattern](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns#supported-pattern-passing-server-components-to-client-components-as-props) を使用して Server Components のみで `date-fns` を使用したパターンを確認します。

```tsx:app/composition-pattern/page.tsx
import { ServerComponent } from "./server-component";
import { ClientComponent } from "./client-component";

export default function App() {
  return (
    <ClientComponent>
      <ServerComponent />
    </ClientComponent>
  );
}
```

Server Components をClient Components の `children` として渡します。

```tsx:app/composition-pattern/client-component.tsx
"use client";

export const ClientComponent = ({
  children,
}: {
  children: React.ReactNode;
}) => {
  return (
    <>
      {children}
      <div>Client Component</div>
      <button onClick={() => alert("Hello!")}>Click me!</button>
    </>
  );
};
```

Client Component は、`children` を受け取り、そのままレンダリングします。
Server Component のコードは以前と同じです。

![Composition Pattern を使用して Server Components のみで `date-fns` を利用した画面](/images/next-bundle-analyzer/composition-pattern-page.png)

表示から Server Components のみが `date-fns` を使用していることが確認できます。

![Composition Pattern を使用して Server Components のみで `date-fns` を利用した Bundle Analyzer の結果](/images/next-bundle-analyzer/composition-pattern-bundle-analyzer.png)

Bundle Analyzer を使用した結果を見ると、`format` 関数がクライアントに配信されていないことが確認できます。これにより、Composition Pattern を使用することで、Server Components でのみ使用されるライブラリがクライアントのバンドルサイズに影響を与えないことが分かりますね。

## バンドルサイズの比較

`date-fns` と各ページのバンドルサイズを比較してみましょう。

![バンドルサイズの比較](/images/next-bundle-analyzer/bundle-size-comparison.png)

以下の表は、ファイルサイズを [parsed サイズ](https://github.com/webpack-contrib/webpack-bundle-analyzer?tab=readme-ov-file#parsed)と [gzip サイズ](https://github.com/webpack-contrib/webpack-bundle-analyzer?tab=readme-ov-file#gzip)で示しています。サイズはバイト単位（B）で統一しています。

| 使用パターン | parsed サイズ (B) | gzip サイズ (B) |
| --------------------------------- | ----------------- | --------------- |
| **Server + Client でライブラリを利用** | 21,266 (535 + 20,797: `date-fns`) | 6,044 (361 + 5,683: `date-fns`) |
| **Server のみでライブラリを利用** | 469 | 312 |
| **Composition Pattern で Server のみでライブラリを利用** | 495 | 322 |

この表から、特に Server Components と Client Components の両方で `date-fns` を利用した場合のバンドルサイズが大きくなることが分かります。`date-fns` のサイズを加味すると、parsed サイズでは約20.31KB、gzip サイズでは約5.55KB の増加が見られます。

静的ファイルを配信する際は圧縮を利用するため、バンドルサイズの差は gzip された単位で比較することが重要になります。最近では gzip よりも圧縮率のいい、[brotil を利用することも増えてきているようです](https://blog.cloudflare.com/this-is-brotli-from-origin-ja-jp)。

さらに、圧縮前のサイズはブラウザが JavaScript を解析、コンパイル、実行するのにかかる時間に影響します。これらはコードサイズに比例する傾向があります。詳しくは [極限環境で最終ビルドを絞るためのフロントエンド設計 - Speaker Deck](https://speakerdeck.com/mizchi/ji-xian-huan-jing-dezui-zhong-birudowojiao-rutamenohurontoendoshe-ji?slide=7) や [JavaScript Performance Beyond Bundle Size](https://nolanlawson.com/2021/02/23/javascript-performance-beyond-bundle-size/) を参照してください。

:::message
この記事では当初、圧縮前のサイズについての記載がありませんでしたが、フィードバックを受けて追記しました。コメントでのご指摘ありがとうございました。
:::

## まとめ

Server と Client の境界を適切にすることで、バンドルサイズの削減に繋がることが視覚的に確認できました。特に Composition Pattern は上手く活用できると良さそうですね。

試したコードは以下のリポジトリにあるので、興味があれば参考までにご覧ください。

https://github.com/ryo-manba/next-bandle-analyzer
