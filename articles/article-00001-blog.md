---
title: 'ブログテンプレート学習'
excerpt: 'Nextブログテンプレートを利用することにより、Reactコンポーネントの理解やES6についての理解を深める。'
coverImage: '/assets/blog/study-photo/study-01.jpg'
date: '2022-10-16'
ogImage:
  url: '/assets/blog/study-photo/study-01.jpg'
tags:
  - 'Next.js'
  - 'React'
  - 'ES6'
  - 'Markdown記法'
  - '学習法'
---

https://stitches.dev/

## ブログテンプレート学習

このブログはNext.jsのブログテンプレートを頂いたものです。今後、オリジナリティのあるものに改変しながらも、未経験独学なりのアウトプットを行うために使用いたします。

[参考サイト様（Next.js+GitHub Pagesでブログを作った）](https://zenn.dev/subt/articles/957bd5d01485e1)

主に現在、学びたい項目は以下になります。

言語・フレームワーク...HTML/CSS/JavaScript/Typescript/React/Next.js  
CSSフレームワーク...Tailwindcss/Bootstrap  
Deployの学習...AWS/Vercel/GithubPages  
ほか...Git/Docker/DB  

※以下、制作中

### React Slot

https://www.radix-ui.com/docs/primitives/utilities/slot

`Slot`は子コンポーネントに props を渡す役割を持ちます。

これが

```tsx
<Slot color="red">
  <AnyComponent />
</Slot>
```

実質的にこうなります。

```tsx
<AnyComponent color="red" />
```

`Slot`本体は消えるものの、props を介して任意のコンポーネントに機能を与えられるという点が重要です。余分な`div`を生成することはありません。

詳しくは[こちら](https://zenn.dev/subt/articles/b6aa48ccb0c884)を参照ください。

### Framer Motion

https://www.framer.com/motion

[Framer](https://www.framer.com/)が提供しているアニメーションライブラリです。

どんなアニメーションでも、基本的に**props だけ**で完結してしまうのが特徴です。

```tsx
<motion.div
  drag="x"
  dragConstraints={{ left: -100, right: 100 }}
  whileHover={{ scale: 1.1 }}
  whileTap={{ scale: 0.9 }}
/>
```

カスタムコンポーネントは`motion`関数に渡してアニメーションをつけます。

```tsx
const MotionComponent = motion(Component);
...
<MotionComponent animate={{ scale: 0.5 }} />
```

## `motion`+`Slot`で何ができる？

`motion`と`Slot`を組み合わせて`motion(Slot)`を作ります。すると、

`motion`が提供してくれる手軽でリッチなアニメーション機能をそのまま、提供元を完全に隠蔽して提供する

ことができるようになります。

何を言っているのか、私もよくわからないので具体例に移りましょう。

## 具体例

`motion(Slot)`を使って、みんな大好き「[ふわっ](https://qiita.com/yuneco/items/24a209cb14661b8a7a20)」が手軽に実装できるようにしましょう。

### 実装

```tsx
import React from 'react';
import { Slot } from '@radix-ui/react-slot';
import { motion } from 'framer-motion';

const ContentLayout = motion(Slot);

type Custom = {
  y?: number;
  once?: boolean;
  amount?: number;
  duration?: number;
};

const defaultCustom: Custom = {
  y: 20,
  once: true,
  amount: 0.3,
  duration: 0.6,
};

const config = (
  custom?: Custom,
): React.ComponentProps<typeof ContentLayout> => {
  const { y, once, amount, duration } = { ...defaultCustom, ...custom };

  return {
    initial: {
      opacity: 0,
      y,
    },
    whileInView: {
      opacity: 1,
      y: 0,
    },
    viewport: {
      once,
      amount,
    },
    transition: {
      duration,
    },
  };
};

type Props = {
  children: React.ReactNode;
  custom?: Custom;
};

export const Enter = React.forwardRef<
  React.ElementRef<typeof ContentLayout>,
  Props
>(({ children, custom }, forwardedRef) => (
  <ContentLayout {...config(custom)} ref={forwardedRef}>
    {children}
  </ContentLayout>
));

Enter.displayName = 'Enter';
```

### 使い方

たったこれだけで、`h1`がふわっと入場します。

```tsx
<Enter>
  <h1>Hello CodeSandbox</h1>
</Enter>
```

CodeSandbox に使用例を上げました。

@[codesandbox](https://codesandbox.io/embed/nifty-fast-gs8cjy?fontsize=14&hidenavigation=1&theme=dark)

ぜひ、別窓で開いて余計な`div`が生成されていないことをお確かめください。

### 注意点

`motion`は内部的に ref を使用します。なので、対象のコンポーネントがカスタムコンポーネントである場合、正しく ref をフォワーディングしている必要があります。

ref や`forwardRef`をご存知ない方は、調べてみてください。きっと、`Slot`や`motion`に比べてはるかに多くの記事がヒットするでしょう。

## まとめ

`motion(Slot)`で再利用性バツグンのアニメーションコンポーネントを作ることができました。

実装自体`motion`の軽い延長に過ぎないので、簡単にオリジナルのアニメーションコンポーネントが作れると思います。是非お試しください。

私も色々実装してみて、またの機会に紹介したいと思います。

`motion(Slot)`で再利用性バツグンのアニメーションコンポーネントを作ることができました。

実装自体`motion`の軽い延長に過ぎないので、簡単にオリジナルのアニメーションコンポーネントが作れると思います。是非お試しください。

私も色々実装してみて、またの機会に紹介したいと思います。最下部の段落は15行書くこと。
