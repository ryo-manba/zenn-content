---
title: "Next.js 13 Template と Layout の使い分け"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

Next.js 13には、LayoutとTemplateというよく似た機能が存在します。
この記事では、それぞれの特徴と使い分け方についてまとめてみました。

## Layoutとは？

Layoutは複数のページに渡って共有されるUIのことを指します。
特徴としてはナビゲーションが行われた際に、その状態を保持し、再レンダリングは行われません。またLayoutはネスト（入れ子）にして使用することも可能です。

### Layoutの定義方法

`layout.tsx` ファイルでReactコンポーネントとして定義できます。
例えば、以下のようなLayoutを定義することができます。

```tsx
// app/posts/layout.tsx
export default function Layout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div style={{ border: '1px solid #ccc', padding: '20px' }}>
      <h2>Layout Header</h2>
      {children}
      <h2>Layout Footer</h2>
    </div>
  );
}
```

`posts/` ディレクトリ以下のページでこのレイアウトが適用されることになります。

### Layoutの使用例

単純なページを用意してみます。

```tsx
export default function Page() {
  return <div>posts</div>;
}
```

このページは以下のように表示されます。

```bash
Layout Header
posts
Layout Footer
```

動的ルーティングを利用した場合にも全てのページで適用されることになります。

```tsx
// app/posts/[slug]/page.tsx
export default function Page({ params }: { params: { slug: string } }) {
  return <div>posts: {params.slug}</div>;
}
```

`posts/1` と `posts/2` にアクセスした場合はそれぞれいかのように表示されます。

```tsx
// posts/1
Layout Header
posts: 1
Layout Footer

// posts/2
Layout Header
posts: 2
Layout Footer
```

以上のように、Layoutを使用することでルーティングに関係なく共通のUIを定義することができます。

## Templateとは？

TemplateはLayoutによく似た機能で、子レイアウトやページをラップします。
しかし、Layoutがページ遷移間でステートを維持するのに対して、Templateは各ページ遷移ごとに新しいインスタンスを生成します。
これは、ユーザがTemplateを適用した複数のページ間を移動する際に、新しいDOM要素が生成され、ステートやエフェクトは再同期されるということです。

### Templateの定義方法

`template.tsx`ファイルでReactコンポーネントとして定義できます。
例えば、次のようにTemplateを定義することが可能です。

```tsx
// app/posts/template.tsx
"use client";

export default function Template({ children }: { children: React.ReactNode }) {
  return (
    <div>
      <h2>Template Header</h2>
      {children}
      <h2>Template Footer</h2>
    </div>
  );
}
```

すると、 先程のLayout同様に`posts/` 配下にあるページでTemplateが適用されることになります。

```tsx
Template Header
posts
Template Footer
```

### LayoutとTemplateの共存

LayoutとTemplateは共存すること可能です。
同じ階層に `layout.tsx` と `template.tsx` が存在する場合、TemplateはLayoutの子要素として適用されます。

```tsx
Layout Header
Template Header
posts
Template Footer
Layout Footer
```

これで、Next.js 13におけるLayoutとTemplateの基本的な使い方と特徴について説明しました。
続いては、これらの使い分けについて解説していきます。

## LayoutとTemplateの使い分け方

これまでの説明に基づいて、LayoutとTemplateの使い分けについて考察します。
一見、両者は画面表示において似ていますが、実際には大きな違いがあります。

それが、**ページの再レンダリング**です。
Layoutはページ遷移時に再レンダリングを行わず、状態を維持します。一方で、Templateは新しいページ遷移ごとに再レンダリングを行います。

以下は、URLパラメータを受け取って表示するシンプルなページの例です。

```tsx
// app/posts/[slug]/page.tsx
"use client";

import Link from "next/link";

export default function Page({ params }: { params: { slug: string } }) {
  return (
    <>
      <div>posts: {params.slug}</div>
      {/* 以下のリンクをクリックすると posts/1 -> posts/2 のように画面遷移を行う */}
      <Link href={`/posts/${Number(params.slug) + 1}`} className="outline">
        Go to next page
      </Link>
    </>
  )
}
```

このページをLayoutとTemplateでラップしてみると、それぞれがどのように挙動するのかを確認することができます。

```tsx
// app/posts/layout.tsx
"use client";

export default function Layout({
  children,
}: {
  children: React.ReactNode;
}) {
  useEffect(() => {
    console.log('Layout rendered');
  }, []);
  return (
    <div>
      <h2>Layout Header</h2>
      {children}
      <h2>Layout Footer</h2>
    </div>
  );
}
```

```tsx
// app/posts/template.tsx
"use client";

export default function Template({ children }: { children: React.ReactNode }) {
  useEffect(() => {
    console.log('Template rendered');
  }, []);
  return (
    <div>
      <h2>Template Header</h2>
      {children}
      <h2>Template Footer</h2>
    </div>
  );
}
```

初めてページを表示すると、LayoutとTemplateの両方でログが出力されます。

```bash
Layout rendered
Template rendered
```

Go to next page リンクをクリックしてページ遷移を行うと、Templateだけが再レンダリングされることが確認できます。

```bash
Template rendered
```

### 使い分けのポイント

この挙動の違いを踏まえると、ページの再レンダリングが不要な場合はLayoutを、必要な場合はTemplateを使用すると良いでしょう。

ページの再レンダリングが必要な場合とは以下のようなシチュエーションが考えられます。

- ページ遷移時にAPIを叩いてデータを取得する
- ページ遷移時にアニメーションを行う
- 各ページで独立したロギングを行う
- ページごとにフィードバックフォームを表示する

以上のようなケースでは、Templateを使用するのが適しています。

## まとめ

LayoutとTemplateの違いは、ページの再レンダリングの挙動です。
パフォーマンスの観点から、特に理由がない場合はLayoutを使用した方が良さそうですね。

## 参考

https://nextjs.org/docs/app/building-your-application/routing/pages-and-layouts