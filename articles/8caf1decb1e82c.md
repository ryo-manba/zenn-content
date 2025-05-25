---
title: "Next.js 13 Template と Layout の使い分け"
emoji: "📑"
type: "tech"
topics: ["nextjs", "react", "フロントエンド"]
published: true
publication_name: "cybozu_frontend"
---

Next.js 13には、LayoutとTemplateというよく似た機能が存在します。
この記事では、それぞれの特徴と使い分け方についてまとめてみました。

## Layoutとは？

Layoutは複数のページに渡って共有されるUIのことを指します。
特徴としては画面遷移が行われた際に、その状態を保持し、再レンダリングは行われません。またLayoutはネスト（入れ子）にして使用することも可能です。

### Layoutの定義方法

appディレクトリ配下で `layout.tsx` ファイルを定義するとLayoutとして定義できます。
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

`posts/1` と `posts/2` にアクセスした場合はそれぞれ以下のように表示されます。

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
しかし、Layoutがページ遷移間でステートを維持するのに対して、Templateは各ページ遷移ごとに新しいステートを生成します。
これは、ユーザがTemplateを適用した複数のページ間を移動する際に、新しいDOM要素が生成され、ステートやエフェクトは再同期されるということです。

### Templateの定義方法

`template.tsx`ファイルでReactコンポーネントとして定義できます。
例えば、次のようにTemplateを定義することが可能です。

```tsx
// app/posts/template.tsx
export default function Template({ children }: { children: React.ReactNode }) {
  return (
    <>
      <h2>Template Header</h2>
      {children}
      <h2>Template Footer</h2>
    </>
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
      <Link href={`/posts/${Number(params.slug) + 1}`}>
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

export default function Layout({ children }: { children: React.ReactNode }) {
  useEffect(() => {
    console.log('Layout rendered');
  }, []);
  return (
    <>
      <h2>Layout Header</h2>
      {children}
      <h2>Layout Footer</h2>
    </>
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
    <>
      <h2>Template Header</h2>
      {children}
      <h2>Template Footer</h2>
    </>
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

- 各ページで独立したロギングを行う
- ページ遷移時にアニメーションを行う
- ページごとにフィードバックフォームを表示する

以上のようなケースでは、Templateの使用が適しています。

ロギングについては、LayoutとTemplateの紹介の時点で `useEffect` を用いてログ出力の検証を行なったため挙動が予想できそうですね。
残りの2つについて実際にどのような違いが現れるのかを確認してみましょう。

### ページ遷移時にアニメーションを行う

一般的によく用いられるものとしてページ遷移時に画面がフェードインするアニメーションがあります。
以下は、Framer Motionを用いた実装例です。

```tsx
// app/posts/template.tsx
"use client";

import { motion } from "framer-motion";

const fadeInVariants = {
  hidden: { opacity: 0, y: -100 },
  visible: { opacity: 1, y: 0 },
};

export default function Template({ children }: { children: React.ReactNode }) {
  return (
    <motion.div
      initial="hidden"
      animate="visible"
      exit="hidden"
      variants={fadeInVariants}
      transition={{ duration: 0.5 }}
    >
      <h2>Template Header</h2>
      {children}
      <h2>Template Footer</h2>
    </motion.div>
  );
}
```

![ページ遷移時のフェードインアニメーション](/images/8caf1decb1e82c/fadein-template.gif)

この例でわかる通り、ページ遷移が行われる度にTemplateコンポーネントが再レンダリングされ、コンテンツがフェードインします。
同じアニメーションをLayoutで実装しても、ページ遷移後にはアニメーションは発生しません。

### ページごとにフィードバックフォームを表示する

Next.jsのドキュメントを始め、各ページごとにフィードバックを送信できるサイトはよく見かけると思います。TemplateとLayoutを使用してそれぞれでフィードバックフォームの違いを比較してみます。

以下は簡単なフィードバックフォームの実装例です。

```tsx
// app/posts/feedback-form.tsx
export const FeedbackForm = ({ label }: { label: string }) => {
  return (
    <form>
      <label>
        <span style={{ display: "block" }}>{label}</span>
        <input type="text" placeholder="Your feedback" />
      </label>
      <button type="submit">Submit</button>
    </form>
  );
};
```

このフィードバックフォームコンポーネントを、TemplateとLayoutで使い分けます。

```tsx
// app/posts/layout.tsx
import { FeedbackForm } from "./feedback-form";
export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <>
      {children}
      <FeedbackForm label="Layout Feedback" />
    </>
  );
}

// app/posts/template.tsx
import { FeedbackForm } from "./feedback-form";
export default function Template({ children }: { children: React.ReactNode }) {
  return (
    <>
      {children}
      <FeedbackForm label="Template Feedback" />
    </>
  );
}
```

![ページ遷移時のフィードバックフォーム](/images/8caf1decb1e82c/feedback-template.gif)

Templateでは、ページ遷移するたびに新しいフォームが表示されています。一方で、Layoutで設置したフォームは、ページ遷移後も前の入力状態を保持しています。
これでは途中までフィードバックを入力した状態でページ遷移を行った場合に誤ったフィードバックを送信しかねません。

このことからページごとに特定のフィードバックを送信する場合は、Templateを使用するのが適していることが分かりますね。

## まとめ

LayoutとTemplateの主な違いは、ページの再レンダリングに関わる挙動にあります。パフォーマンスを最適化する観点から、再レンダリングが不要な場合ではLayoutの使用が推奨されます。

具体的な使用ケースとしては、独立したロギングやページ遷移時のアニメーション、ページごとのフィードバックフォームなどがありそうでした。

使い分けの参考になれば幸いです。

## 参考

https://nextjs.org/docs/app/building-your-application/routing/pages-and-layouts
