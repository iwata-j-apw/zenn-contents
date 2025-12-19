---
title: "キーボードだけで操作できるWEBアプリを作りたい with アクセシビリティ"
emoji: "🫶"
type: "idea"
topics:
  - "a11y"
  - "waiaria"
  - "keyboard"
  - "React"
  - "nextjs"
published: false
---

※この記事は [アップルワールド Advent Calendar 2025](https://qiita.com/advent-calendar/2025/appleworld) 20 日目の記事です。

## 1. 概要

### 1.1 経緯

新卒 1 年目として働き始めた今年度の始め、別の会社で長期インターンをしていたため 2 社目となる環境で PG として働き始めました。この新しい環境で、早々からあるツールにお世話になっております。そう、それは coderabbit です。GitHub 上でレビューしてくれるツールですが、そこで度々ある指摘を受けました。

```diff:coderabbitより
+ aria-label="タイトル" // ラベル追加
```

そう、WAI-ARIA です。
何がしたいのかは初心者の私でも分かりますが、必要な理由が実感できず、モヤモヤしたままでいました。「Name プロパティを付与することで支援技術が〜」という説明をもらっても、実感が全く湧かないので「あー、そうですか」で終わってしまうという状態です。

そんな状態を脱するべく、腰を据えて勉強してみようと思い、こちらの書籍を手に取りました。

https://gihyo.jp/book/2023/978-4-297-13366-5

体系的に勉強することができ、アクセシビリティ（＋ユーザビリティ）についての知見と感覚を広く身につけることができたように感じました。その中で、ネイティブセマンティクスを最大限活用することが最優先であることは承知のもと、div タグだけでアクセシブルな UI を作ることができるのではないかと思ってしまいました。その好奇心をこの記事では発散しようと思います。

### 1.2 記事の内容

以下のように使用しても良い HTML タグに制約を設けてログイン画面を作成していきます。
※ input などの form 系のタグは自前の実装が大変なので使わせてもらいます。

- div タグ
- form 系のタグ（input, select, label など）

完成品を先にお見せしておきます。一般的によく見る、ただのログインフォームです。キーボードからのみの操作でログイン操作をすることができるようになっています。
※ 乗っ取られた時に出てきそうなデザインの完了画面ですが、気にしないでください。

![ログインフォームの完成画面](/images/a11y-div-challenge/comp1.gif)



左上の「ログインダイアログを開く」を押下すると、ダイアログからもログインすることができるという、二段構えになっています。ログインUIも冗長化させる時代を勝手に到来させました。

![ダイアログからログインする画面](/images/a11y-div-challenge/comp2.gif)

## 2. 準備

本記事では、Next.js を使用して実装を行います。
設定はデフォルトのものを使用して進めていきます。（TypeScript, AppRouter, Tailwind, ESLint, etc.）

https://nextjs.org/docs/app/getting-started/installation

ログインフォームを作成するということで、実装を単純にするために react-hook-form と zod を入れてフォーム管理をしやすい状態で進めることができるようにしておきます。

https://zenn.dev/b13o/articles/about-react-hook-form

## 3. 汎用コンポーネントの作成

まずは汎用的に使えるコンポーネントを先に実装し、それらを活用してログインフォームを作成する方針で進めていきます。

※ 本記事全体についてですが、ロジックを見やすくするために、添付しているコードは Tailwind でのスタイリングに関する部分をほぼカットして載せています。

### 3-1. ボタン

まず始めに Button コンポーネントから作成していきます。実装したい内容に関しては単純で、button タグが持っているセマンティクスを div タグで再現するという、全く疑いようのない車輪の再開発です。
クリック時の処理とテキスト部分だけを指定できる、最低限 Button として使用可能な状態を初期状態として実装を始めます。分かりやすいように、本記事で作成したコンポーネントの命名の頭には「A11y」という接頭辞を付けています。

:::details Button コンポーネントの土台

```js
"use client";

import { forwardRef } from "react";

interface A11yButtonProps
  extends Omit<React.HTMLAttributes<HTMLDivElement>, "onClick"> {
  onClick: () => void;
  children: React.ReactNode;
}

export const A11yButton = forwardRef<HTMLDivElement, A11yButtonProps>(
  ({ onClick, children, ...rest }, ref) => {

    return (
      <div
        {...rest}
        ref={ref}
        onClick={onClick}
      >
        {children}
      </div>
    );
  }
);

A11yButton.displayName = "A11yButton";
```

:::

#### 支援技術にボタンであることを伝える

まずは、ただの div タグである以上、このコンポーネントはブラウザにも支援技術にもボタンであることが伝わっていないので、WAI-ARIA の role 属性を付与することにします。
これを入れることで、スクリーンリーダは「ボタンテキスト、ボタン」のようにボタンであることを伝えてくれるようになります。加えて私の視覚的に、`<div` という文字列が若干霞みがかり `<button` に見えるような気がしてきます。

:::details role 属性を付与

```diff
"use client";

import { forwardRef } from "react";

interface A11yButtonProps
  extends Omit<React.HTMLAttributes<HTMLDivElement>, "onClick"> {
  onClick: () => void;
  children: React.ReactNode;
}

export const A11yButton = forwardRef<HTMLDivElement, A11yButtonProps>(
  ({ onClick, children, ...rest }, ref) => {

    return (
      <div
        {...rest}
        ref={ref}
+       role='button'
        onClick={onClick}
      >
        {children}
      </div>
    );
  }
);

A11yButton.displayName = "A11yButton";
```

:::

#### キーボードからクリックできるようにする

次は、キーボードのみでボタンを操作できるようにします。

実装する内容としては、以下の 3 つです。

- Tab キーでフォーカスを当てることができるようになる
- フォーカスが当たった状態で Enter キーを押下するとクリック判定になる
- フォーカスが当たった状態で Space キーを押下して離すとクリック判定になる

onKeyDown と onKeyUp については、このコンポーネントを使用する側でもカスタムできるように、rest プロパティとして受け取って実行できるようにしています。

※ div タグはクリック時のデフォルトの挙動を持っていないので、`e.preventDefault()` は不要です。私は安心するというだけの理由で入れています。

:::details キーボード操作を可能にする

```diff
"use client";

import { forwardRef } from "react";

interface A11yButtonProps
  extends Omit<React.HTMLAttributes<HTMLDivElement>, "onClick"> {
  onClick: () => void;
  children: React.ReactNode;
}

export const A11yButton = forwardRef<HTMLDivElement, A11yButtonProps>(
  ({ onClick, children, ...rest }, ref) => {
+   const handleClick = (e: React.MouseEvent<HTMLDivElement>) => {
+     e.preventDefault();
+     onClick();
+   };

+   const handleKeyDown = (e: React.KeyboardEvent<HTMLDivElement>) => {
+     if (e.key === "Enter") {
+       e.preventDefault();
+       onClick();
+     } else if (e.key === " ") {
+       // スペースキーの場合はKeyUpでクリックイベントを発火させるので、ここではデフォルトの動作を防ぐだけ
+       e.preventDefault();
+     }
+     rest.onKeyDown?.(e);
+   };

+   const handleKeyUp = (e: React.KeyboardEvent<HTMLDivElement>) => {
+     if (e.key === " ") {
+       onClick();
+     }
+     rest.onKeyUp?.(e);
+   };

    return (
      <div
        {...rest}
        ref={ref}
        role='button'
+       tabIndex={0} // 0以上に設定することにより、 tabキーでフォーカスを当てることができるようになる
+       onClick={handleClick}
+       onKeyUp={handleKeyUp}
+       onKeyDown={handleKeyDown}
      >
        {children}
      </div>
    );
  }
);

A11yButton.displayName = "A11yButton";
```

:::

#### disabled 状態の制御ができるようにする

button タグには、ニーズが比較的高めな disabled 属性が存在するので、こちらも実装します。
実装内容はシンプルで、disabled プロパティを受け取ることができるように変更し、その内容をもとにクリックアクションを起こしても良いのかの判定を行うようにするだけです。
加えて、disabled 時には tab キーでフォーカスを当てることができないようにし、aria-disabled を設定することで disabled 時に「ボタンテキスト、ボタン、使用不可」のようにスクリーンリーダに読んでもらえるようにします。

:::details disabled 状態を設定できるようにする

```diff
"use client";

import { forwardRef } from "react";

interface A11yButtonProps
  extends Omit<React.HTMLAttributes<HTMLDivElement>, "onClick"> {
  onClick: () => void;
+ disabled?: boolean;
  children: React.ReactNode;
}

export const A11yButton = forwardRef<HTMLDivElement, A11yButtonProps>(
- ({ onClick, children, ...rest }, ref) => {
+ ({ onClick, disabled = false, children, ...rest }, ref) => {
    const handleClick = (e: React.MouseEvent<HTMLDivElement>) => {
+     if (disabled) return;
      e.preventDefault();
      onClick();
    };

    const handleKeyDown = (e: React.KeyboardEvent<HTMLDivElement>) => {
+     if (disabled) return;
      if (e.key === "Enter") {
        e.preventDefault();
        onClick();
      } else if (e.key === " ") {
        // スペースキーの場合はKeyUpでクリックイベントを発火させるので、ここではデフォルトの動作を防ぐだけ
        e.preventDefault();
      }
      rest.onKeyDown?.(e);
    };

    const handleKeyUp = (e: React.KeyboardEvent<HTMLDivElement>) => {
+     if (disabled) return;
      if (e.key === " ") {
        onClick();
      }
      rest.onKeyUp?.(e);
    };

    return (
      <div
        {...rest}
        ref={ref}
        role='button'
+       aria-disabled={disabled}
-       tabIndex={0}
+       tabIndex={disabled ? -1 : 0}
        onClick={handleClick}
        onKeyUp={handleKeyUp}
        onKeyDown={handleKeyDown}
      >
        {children}
      </div>
    );
  }
);

A11yButton.displayName = "A11yButton";

```

:::

#### 完成した Button コンポーネント

ここまでで、button タグがデフォルトで持っている機能のうち、ニーズが高そうなものをひと通り実装することができました。

完成形の画面での実装確認にはなりますが、「ログインダイアログを開く」ボタンを

- 1回目は、フォーカスを当てて Enter キーを押下
- 2回目は、フォーカスを当てて Space キーを押下

で開くことできることを確認できました。

![Button コンポーネントの動作確認](/images/a11y-div-challenge/button.gif)

完成形のコードを置いておきます。

:::details 最終的な Button コンポーネント（スタイルなし）

```js
"use client";

import { forwardRef } from "react";

interface A11yButtonProps
  extends Omit<React.HTMLAttributes<HTMLDivElement>, "onClick"> {
  onClick: () => void;
  disabled?: boolean;
  children: React.ReactNode;
}

export const A11yButton = forwardRef<HTMLDivElement, A11yButtonProps>(
  ({ onClick, disabled = false, children, ...rest }, ref) => {
    const handleClick = (e: React.MouseEvent<HTMLDivElement>) => {
      if (disabled) return;
      e.preventDefault();
      onClick();
    };

    const handleKeyDown = (e: React.KeyboardEvent<HTMLDivElement>) => {
      if (disabled) return;
      if (e.key === "Enter") {
        e.preventDefault();
        onClick();
      } else if (e.key === " ") {
        // スペースキーの場合はKeyUpでクリックイベントを発火させるので、ここではデフォルトの動作を防ぐだけ
        e.preventDefault();
      }
      rest.onKeyDown?.(e);
    };

    const handleKeyUp = (e: React.KeyboardEvent<HTMLDivElement>) => {
      if (disabled) return;
      if (e.key === " ") {
        onClick();
      }
      rest.onKeyUp?.(e);
    };

    return (
      <div
        {...rest}
        ref={ref}
        role='button'
        aria-disabled={disabled}
        tabIndex={disabled ? -1 : 0}
        onClick={handleClick}
        onKeyUp={handleKeyUp}
        onKeyDown={handleKeyDown}
      >
        {children}
      </div>
    );
  }
);

A11yButton.displayName = "A11yButton";

```

:::

### 3-2. ダイアログ

次は Dialog コンポーネントを作成します。shadcn や MUI などの著名な UI コンポーネントライブラリの Dialog でも、 div や span を多用して実装している UI ではあるので、それと似通った実装をすることになります。
初期状態を以下のように作成しました。機能としては、ダイアログ外をクリックした際に閉じることができるように画面全体を覆うオーバーレイの要素と、ダイアログを閉じるボタン、タイトル、中身を流し込む children だけです。機能面だけ考えれば最低限使えそうな状態にはなっています。

:::details ダイアログのマークアップ

見渡す限り一面の div タグは、やはり美しいですね。

```js
"use client";

import { A11yButton } from "./button";

interface A11yDialogProps {
  title: string;
  open: boolean;
  onClose: () => void;
  children: React.ReactNode;
}

export const A11yDialog = ({
  title,
  open,
  onClose,
  children,
}: A11yDialogProps) => {
  if (!open) {
    return null;
  }

  return (
    <div>
      {/* オーバーレイ */}
      <div onClick={onClose}></div>

      <div>
        <A11yButton onClick={onClose}>×</A11yButton>
        <div>{title}</div>
        {children}
      </div>
    </div>
  );
};
```

:::

#### 支援技術にダイアログであることを伝える

まずは、ブラウザと支援技術に対してダイアログであることを伝えるために、WAI-ARIA を存分に活用します。Button コンポーネントの時と同じように、どんな UI なのかというのを role 属性で指定します。今回は、`role='dialog'` を指定します。
useId で一意のキーを作成していますが、こちらはダイアログとその見出しを紐づけることを目的としています。 `role='dialog'` が付与された div タグに指定されている `aria-labelledby` プロパティに指定された値を id に持つ要素の中身がダイアログの見出しとなり、ダイアログを開いたタイミングでスクリーンリーダに読んでもらえます。
見出しには `<h*>` タグを使用したいところですが、div タグのみという制約があるので、苦し紛れに最大限の努力をして `<h2>` に近づけます。
ここまで設定することにより、スクリーンリーダは「見出し、ダイアログ」と読んでくれるようになります。`aria-labelledby` の紐付けと同じように、`aria-describedby` を設定することで、「見出し、ダイアログ、説明」を読んでもらえるように設定することも可能です。

閉じるボタンに関しては、今回のようにテキストではない記号のような文字列を使用している場合、スクリーンリーダなどの支援技術は、「×、ボタン」と読んでしまうので、分かりやすくするために `aria-label` を設定することで、今回のケースでは「ダイアログを閉じる、ボタン」と読んでもらえるようにしています。

:::details role 属性と aria-labelledby を付与

```diff
"use client";

+ import { useId } from "react";
import { A11yButton } from "./button";

interface A11yDialogProps {
  title: string;
  open: boolean;
  onClose: () => void;
  children: React.ReactNode;
}

export const A11yDialog = ({
  title,
  open,
  onClose,
  children,
}: A11yDialogProps) => {
+ const titleId = useId();

  if (!open) {
    return null;
  }

  return (
    <div>
      {/* オーバーレイ */}
      <div onClick={onClose}></div>

-     <div>
+     <div role='dialog' aria-modal aria-labelledby={titleId}>
        <A11yButton
          onClick={onClose}
+         aria-label='ダイアログを閉じる'
        >
          ×
        </A11yButton>
        <div
+         id={titleId}
+         role='heading'
+         aria-level={2}
        >
          {title}
        </div>
        {children}
      </div>
    </div>
  );
};

```

:::

#### ダイアログ開閉時にフォーカスを移動する

次は、ダイアログを開いた際に、ダイアログ内の要素にフォーカスを自動で移動するように変更します。
現状だと、ダイアログを開いたとしても、ダイアログを開く際にクリックしたボタンにフォーカスが当たったままになってしまいます。
それに加え、ダイアログを閉じた際、開く前にフォーカスが当たっていた場所にフォーカスを戻す必要があることを踏まえ、以下のフォーカス移動のフローを考えます。

1. ダイアログを開く直前、フォーカスが当たっていた要素の参照を保持する（コード上では `triggerRef`）
2. ダイアログが開いた直後、ダイアログの見出しにフォーカスを当てる（このタイミング以外では見出しにフォーカスを当てることができないようにする）
3. ダイアログを閉じる直前、`triggerRef` にフォーカスを移動させる

見出しに `tabIndex={-1}` を指定することで、tab キーでフォーカスを当てることはできないが、JS での操作によりフォーカスを当てることができるようになります。

:::details フォーカス管理を実装

```diff
"use client";

- import { useId } from "react";
+ import { useEffect, useId, useRef } from "react";
import { A11yButton } from "./button";

interface A11yDialogProps {
  title: string;
  open: boolean;
  onClose: () => void;
  children: React.ReactNode;
}

export const A11yDialog = ({
  title,
  open,
  onClose,
  children,
}: A11yDialogProps) => {
  const titleId = useId();

+ // フォーカス管理のためのRef
+ const triggerRef = useRef<HTMLElement | null>(null);
+ const titleRef = useRef<HTMLDivElement | null>(null);

+ useEffect(() => {
+   if (open) {
+     // 開く直前にフォーカスがあった要素を保存
+     const active = document.activeElement;
+     if (active instanceof HTMLElement) triggerRef.current = active;
+
+     // タイトルにフォーカス
+     titleRef.current?.focus();
+   } else {
+     triggerRef.current?.focus();
+   }
+ }, [open]);

  if (!open) {
    return null;
  }

  return (
    <div>
      {/* オーバーレイ */}
      <div onClick={onClose}></div>

      <div role='dialog' aria-modal aria-labelledby={titleId}>
        <A11yButton
          onClick={onClose}
          aria-label='ダイアログを閉じる'
        >
          ×
        </A11yButton>
        <div
          id={titleId}
+         ref={titleRef}
          role='heading'
          aria-level={2}
+         tabIndex={-1}
+         // ダイアログ初期表示時に、フォーカスインジケーターが表示されないようにするためのリセットCSSクラス
+         className='outline-none focus:outline-none focus:ring-0 focus-visible:outline-none focus-visible:ring-0'
        >
          {title}
        </div>
        {children}
      </div>
    </div>
  );
};

```

:::

#### フォーカストラップを実装する

ここからはフォーカストラップと呼ばれるものを実装します。
現状の状態では、ダイアログを開いた際にダイアログの見出しにフォーカスを合わせることはできましたが、このフォーカスを Tab キーで移動し続けてみると、ダイアログ外に出てしまい、ダイアログ下の要素にフォーカスを当てることができてしまいます。そこで、ダイアログコンポーネントの DOM の境界に、フォーカス移動の壁となるよう要素を作成することでこれを回避します。その壁のことをフォーカストラップと言います。（ちなみにですが、shadcn と MUI の Dialog コンポーネントにもフォーカストラップらしき、それぞれ span タグと div タグが入っていました）
実装内容としては直感的で、そのフォーカストラップにフォーカスが当たると、ダイアログ要素の真反対のフォーカス可能要素にフォーカスを渡すようにして、ダイアログ内でフォーカス移動が無限ループするようにします。

:::details フォーカストラップの実装

```diff
"use client";

import { useEffect, useId, useRef } from "react";
import { A11yButton } from "./button";

interface A11yDialogProps {
  title: string;
  open: boolean;
  onClose: () => void;
  children: React.ReactNode;
}

+ // フォーカス可能要素
+ const focusableSelector = [
+   "[tabindex]:not([tabindex='-1'])",
+   "input",
+   "select",
+   "textarea",
+ ].join(",");

export const A11yDialog = ({
  title,
  open,
  onClose,
  children,
}: A11yDialogProps) => {
  const titleId = useId();

  // フォーカス管理のためのRef
+ const dialogRef = useRef<HTMLDivElement | null>(null);
  const triggerRef = useRef<HTMLElement | null>(null);
  const titleRef = useRef<HTMLDivElement | null>(null);
+ const closeBtnRef = useRef<HTMLDivElement | null>(null);
+ const lastFocusableElementRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (open) {
      // 開く直前にフォーカスがあった要素を保存
      const active = document.activeElement;
      if (active instanceof HTMLElement) triggerRef.current = active;

      // タイトルにフォーカス
      titleRef.current?.focus();

+     // フォーカストラップ用：最後のフォーカス可能要素を保存
+     const root = dialogRef.current;
+     if (root) {
+       const focusableElements = Array.from(
+         root.querySelectorAll<HTMLElement>(focusableSelector)
+       ).filter(
+         (el) =>
+           !el.hasAttribute("disabled") &&
+           el.getAttribute("aria-hidden") !== "true"
+       );

+       lastFocusableElementRef.current =
+         focusableElements[focusableElements.length - 1] ?? null;
+     } else {
+       lastFocusableElementRef.current = null;
+     }
    } else {
      triggerRef.current?.focus();
    }
  }, [open]);

  if (!open) {
    return null;
  }

  return (
    <div>
      {/* オーバーレイ */}
      <div onClick={onClose}></div>

+     {/* フォーカストラップ（ダイアログ最下部フォーカス可能要素にフォーカスを渡す） */}
+     <div
+       onFocus={() => {
+         lastFocusableElementRef.current?.focus();
+       }}
+       role="presentation" // AOMに登録しないようにするため
+       tabIndex={0}
+     ></div>

-     <div role='dialog' aria-modal aria-labelledby={titleId}>
+     <div ref={dialogRef} role='dialog' aria-modal aria-labelledby={titleId}>
        <A11yButton
+         ref={closeBtnRef}
          onClick={onClose}
          aria-label='ダイアログを閉じる'
        >
          ×
        </A11yButton>
        <div
          id={titleId}
          ref={titleRef}
          role='heading'
          aria-level={2}
          tabIndex={-1}
          // ダイアログ初期表示時に、フォーカスインジケーターが表示されないようにするためのリセットCSSクラス
          className='outline-none focus:outline-none focus:ring-0 focus-visible:outline-none focus-visible:ring-0'
        >
          {title}
        </div>
        {children}
      </div>

+     {/* フォーカストラップ（ダイアログ最上部フォーカス可能要素にフォーカスを渡す） */}
+     <div
+       onFocus={() => {
+         closeBtnRef.current?.focus();
+       }}
+       role="presentation"
+       tabIndex={0}
+     ></div>
    </div>
  );
};

```

:::

#### キーボードからダイアログを閉じることができるようにする

ダイアログは、ESC キーをクリックされた際に閉じることが一般的なので、そちらを可能にするための実装を行います。
内容としては、ESC キー押下時にダイアログを閉じるという処理をイベントリスナーに登録するだけです。

:::details キーボードからダイアログを閉じることを可能にする

```diff
"use client";

import { useCallback, useEffect, useId, useRef } from "react";
import { A11yButton } from "./button";

interface A11yDialogProps {
  title: string;
  open: boolean;
  onClose: () => void;
  children: React.ReactNode;
}

// フォーカス可能要素
const focusableSelector = [
  "[tabindex]:not([tabindex='-1'])",
  "input",
  "select",
  "textarea",
].join(",");

export const A11yDialog = ({
  title,
  open,
  onClose,
  children,
}: A11yDialogProps) => {
  const titleId = useId();
  // フォーカス管理のためのRef
  const dialogRef = useRef<HTMLDivElement | null>(null);
  const triggerRef = useRef<HTMLElement | null>(null);
  const titleRef = useRef<HTMLDivElement | null>(null);
  const closeBtnRef = useRef<HTMLDivElement | null>(null);
  const lastFocusableElementRef = useRef<HTMLElement | null>(null);

+ const handleEscKeyDown = useCallback(
+   (e: KeyboardEvent) => {
+     if (e.key === "Escape") {
+       e.preventDefault();
+       onClose();
+     }
+   },
+   [onClose]
+ );

  useEffect(() => {
    if (open) {
      // 開く直前にフォーカスがあった要素を保存
      const active = document.activeElement;
      if (active instanceof HTMLElement) triggerRef.current = active;

      // タイトルにフォーカス
      titleRef.current?.focus();

      // フォーカストラップ用：最後のフォーカス可能要素を保存
      const root = dialogRef.current;
      if (root) {
        const focusableElements = Array.from(
          root.querySelectorAll<HTMLElement>(focusableSelector)
        ).filter(
          (el) =>
            !el.hasAttribute("disabled") &&
            el.getAttribute("aria-hidden") !== "true"
        );

        lastFocusableElementRef.current =
          focusableElements[focusableElements.length - 1] ?? null;
      } else {
        lastFocusableElementRef.current = null;
      }

+     // Esc でダイアログを閉じることができるようにイベントリスナーを追加
+     window.addEventListener("keydown", handleEscKeyDown);
+     // クリーンアップ関数でイベントリスナーを削除
+     return () => {
+       window.removeEventListener("keydown", handleEscKeyDown);
+     };
    } else {
      triggerRef.current?.focus();
+     return;
    }
  }, [open, handleEscKeyDown]);

  if (!open) {
    return null;
  }

  return (
    <div>
      {/* オーバーレイ */}
      <div onClick={onClose}></div>

      {/* フォーカストラップ（ダイアログ最下部フォーカス可能要素にフォーカスを渡す） */}
      <div
        onFocus={() => {
          lastFocusableElementRef.current?.focus();
        }}
        role="presentation"
        tabIndex={0}
      ></div>

      <div ref={dialogRef} role='dialog' aria-modal aria-labelledby={titleId}>
        <A11yButton
          ref={closeBtnRef}
          onClick={onClose}
          aria-label='ダイアログを閉じる'
        >
          ×
        </A11yButton>
        <div
          id={titleId}
          ref={titleRef}
          role='heading'
          aria-level={2}
          tabIndex={-1}
          // ダイアログ初期表示時に、フォーカスインジケーターが表示されないようにするためのリセットCSSクラス
          className='outline-none focus:outline-none focus:ring-0 focus-visible:outline-none focus-visible:ring-0'
        >
          {title}
        </div>
        {children}
      </div>

      {/* フォーカストラップ（ダイアログ最上部フォーカス可能要素にフォーカスを渡す） */}
      <div
        onFocus={() => {
          closeBtnRef.current?.focus();
        }}
        role="presentation"
        tabIndex={0}
      ></div>
    </div>
  );
};

```

:::

#### 完成した Dialog コンポーネント

想定以上にコードの分量が多くなってしまいましたが、ここまで実装することで最低限キーボードから開閉の一連の操作をすることが可能になりました。

完成形の状態で確認すると、ダイアログを開いているうちはダイアログ内でのみフォーカスが移動するようになっていて、閉じた際にはダイアログを開く直前にフォーカスが当たっていた場所に戻ることが確認できます。閉じる操作に関しても、右上の閉じるボタンと Esc キーで閉じることができることを確認できました。

![Dialog コンポーネントの動作確認](/images/a11y-div-challenge/dialog.gif)

:::details 最終的な Dialog コンポーネント（スタイルなし）

```js
"use client";

import { useCallback, useEffect, useId, useRef } from "react";
import { A11yButton } from "./button";

interface A11yDialogProps {
  title: string;
  open: boolean;
  onClose: () => void;
  children: React.ReactNode;
}

// フォーカス可能要素
const focusableSelector = [
  "[tabindex]:not([tabindex='-1'])",
  "input",
  "select",
  "textarea",
].join(",");

export const A11yDialog = ({
  title,
  open,
  onClose,
  children,
}: A11yDialogProps) => {
  const titleId = useId();
  // フォーカス管理のためのRef
  const dialogRef = useRef<HTMLDivElement | null>(null);
  const triggerRef = useRef<HTMLElement | null>(null);
  const titleRef = useRef<HTMLDivElement | null>(null);
  const closeBtnRef = useRef<HTMLDivElement | null>(null);
  const lastFocusableElementRef = useRef<HTMLElement | null>(null);

  const handleEscKeyDown = useCallback(
    (e: KeyboardEvent) => {
      if (e.key === "Escape") {
        e.preventDefault();
        onClose();
      }
    },
    [onClose]
  );

  useEffect(() => {
    if (open) {
      // 開く直前にフォーカスがあった要素を保存
      const active = document.activeElement;
      if (active instanceof HTMLElement) triggerRef.current = active;

      // タイトルにフォーカス
      titleRef.current?.focus();

      // フォーカストラップ用：最後のフォーカス可能要素を保存
      const root = dialogRef.current;
      if (root) {
        const focusableElements = Array.from(
          root.querySelectorAll<HTMLElement>(focusableSelector)
        ).filter(
          (el) =>
            !el.hasAttribute("disabled") &&
            el.getAttribute("aria-hidden") !== "true"
        );

        lastFocusableElementRef.current =
          focusableElements[focusableElements.length - 1] ?? null;
      } else {
        lastFocusableElementRef.current = null;
      }

      // Esc でダイアログを閉じることができるようにイベントリスナーを追加
      window.addEventListener("keydown", handleEscKeyDown);
      // クリーンアップ関数でイベントリスナーを削除
      return () => {
        window.removeEventListener("keydown", handleEscKeyDown);
      };
    } else {
      triggerRef.current?.focus();
      return;
    }
  }, [open, handleEscKeyDown]);

  if (!open) {
    return null;
  }
  return (
    <div>
      {/* オーバーレイ */}
      <div onClick={onClose}></div>

      {/* フォーカストラップ（ダイアログ最下部フォーカス可能要素にフォーカスを渡す） */}
      <div
        onFocus={() => {
          lastFocusableElementRef.current?.focus();
        }}
        role="presentation"
        tabIndex={0}
      ></div>
      <div ref={dialogRef} role='dialog' aria-modal aria-labelledby={titleId}>
        <A11yButton
          ref={closeBtnRef}
          onClick={onClose}
          aria-label='ダイアログを閉じる'
        >
          ×
        </A11yButton>
        <div
          id={titleId}
          ref={titleRef}
          role='heading'
          aria-level={2}
          tabIndex={-1}
          // ダイアログ初期表示時に、フォーカスインジケーターが表示されないようにするためのリセットCSSクラス
          className='outline-none focus:outline-none focus:ring-0 focus-visible:outline-none focus-visible:ring-0'
        >
          {title}
        </div>
        {children}
      </div>
      {/* フォーカストラップ（ダイアログ最上部フォーカス可能要素にフォーカスを渡す） */}
      <div
        onFocus={() => {
          closeBtnRef.current?.focus();
        }}
        role="presentation"
        tabIndex={0}
      ></div>
    </div>
  );
};
```

:::

### 3-3. エラー

次はエラー文を表示させることに特化したコンポーネントを作成します。用途としてはバリデーションエラーの表示です。

```react
{error.message && <span>{error.message}</span>}
```

Claude Code 含め、AI に生成させるエラー文の表示ロジックは一般的に上記のような形式になるかと思いますが、これだと表示非表示が切り替わる度に AOM に登録されたり登録が外れたりするので、エラー文の読み上げを本当の意味でコントロールすることができません。他のコンテンツの読み上げに被せるような形でエラーを読み上げてもらうのでも問題ないということであれば上記の実装でも問題ありませんが。（AOM については以下参考）

https://qiita.com/fsd-tetsu/items/d28e9f910ad9c62b6ef4

例に漏れず、最低限の実装だけしたコンポーネントを準備します。
渡ってきたテキストの先頭にエラーっぽい飾りを付けて表示させます。

:::details エラー表示の土台

```js
interface A11yErrorProps extends React.HTMLAttributes<HTMLDivElement> {
  errorText: string;
}

/** idプロパティを渡して、紐づけることを推奨する */
export const A11yError = ({ errorText, ...rest }: A11yErrorProps) => {
  return (
    <div {...rest}>
      {errorText && 
        <>
          ❗️
          エラー：{errorText}
        </>
      }
    </div>
  );
};
```

:::

#### 支援技術が適切にエラーを扱えるようにする

エラーを支援技術が適切に管理することを目的として、WAI-ARIA を活用します。
`aria-live` 属性を設定することで、子要素の変更を検知して、スクリーンリーダなどの支援技術が読み上げるようにします。今回は他の要素を読み上げている場合は、そちらの終了を待ってから読み上げができるように、`polite` という値を設定します。`aria-atomic` をつけるのは、変更差分しか読み上げないところを、その直前の「エラー：」の部分もセットで読んでもらうことを目的としています。
「❗️」は読み上げなどは全く不要（寧ろノイズ）なので念の為 `aria-hidden` を付与しておき、AOM に登録されないようにしておきます。

:::details aria-live と aria-atomic を付与

```diff
interface A11yErrorProps extends React.HTMLAttributes<HTMLDivElement> {
  errorText: string;
}

/** idプロパティを渡して、紐づけることを推奨する */
export const A11yError = ({ errorText, ...rest }: A11yErrorProps) => {
  return (
    <div
      {...rest}
+     aria-live='polite'
+     aria-atomic
    >
      {errorText && 
        <>
-         ❗️
+         <div aria-hidden className="inline">❗️</div>
          エラー：{errorText}
        </>
      }
    </div>
  );
};
```

:::

#### 完成した Error コンポーネント

スクリーンリーダへの表示の仕方を最大限コントロールしたエラー表示を実装できました。
動作確認は、後続の Form 系コンポーネントを作成してから行います。

:::details 最終的な Error コンポーネント（スタイルなし）

```js
interface A11yErrorProps extends React.HTMLAttributes<HTMLDivElement> {
  errorText: string;
}

/** idプロパティを渡して、紐づけることを推奨する */
export const A11yError = ({ errorText, ...rest }: A11yErrorProps) => {
  return (
    <div
      {...rest}
      aria-live='polite'
      aria-atomic
    >
      {errorText &&
        <>
          <div aria-hidden className="inline">❗️</div>
          エラー：{errorText}
        </>
      }
    </div>
  );
};

```

:::

### 3-4. Input フォーム

ここからフォーム関連のコンポーネントを幾つか作ります。
先に Input コンポーネントから作成します。
例の如く、ベースとなる部分だけ作成しておきます。
id を用いて、label と input の紐付けはしてあります。

:::details Input のマークアップ

```js
"use client";

import { forwardRef, useId } from "react";
import { A11yError } from "./error";

interface A11yInputProps extends React.ComponentPropsWithoutRef<"input"> {
  label?: string;
  errorText?: string;
}

export const A11yInput = forwardRef<HTMLInputElement, A11yInputProps>(
  ({ label, errorText, ...rest }, ref) => {
    const autoId = useId();
    const formId = rest.id ?? autoId;

    return (
      <div>
        {label && <label htmlFor={formId}>{label}</label>}
        <input
          {...rest}
          ref={ref}
          id={formId}
        />
        <A11yError errorText={errorText ?? ""} />
      </div>
    );
  }
);

A11yInput.displayName = "A11yInput";

```

:::

#### エラー文とフォームを紐づける

エラー文については、`aria-live` が設定されているので、エラーが発生した場合に順次エラー内容がスクリーンリーダに読まれていきます。
フォームにはそれだけでなく、スクリーンリーダが選択的にエラーが起こっているフォームにジャンプすることができるものも多く、そのフォームにフォーカスを当てた際に、再度エラー内容を読んでもらえるように設定することができます。
エラーが起こっていることをスクリーンリーダに伝えるためには `aria-invalid` を設定します。こちらが `true` である場合に、エラーが発生しているフォームであると検知され、そのエラー内容は `aria-describedby` に紐付いている要素の中身が読まれます。なので下記の設定では、エラー処理の手順として、以下のようになっています。

1. エラーが発生した際に、順次エラー内容がスクリーンリーダに読まれる
2. スクリーンリーダの機能を用いて、エラーが発生しているフォームにジャンプする
3. フォーカスを当てると、再度エラー内容が読まれる

:::details エラー内容を支援技術に適切に伝える

```diff
"use client";

import { forwardRef, useId } from "react";
import { A11yError } from "./error";

interface A11yInputProps extends React.ComponentPropsWithoutRef<"input"> {
  label?: string;
  errorText?: string;
}

export const A11yInput = forwardRef<HTMLInputElement, A11yInputProps>(
  ({ label, errorText, ...rest }, ref) => {
    const autoId = useId();
    const formId = rest.id ?? autoId;
+   const errorId = `${formId}-error`;
+   const hasError = Boolean(errorText);

    return (
      <div>
        {label && <label htmlFor={formId}>{label}</label>}
        <input
          {...rest}
          ref={ref}
          id={formId}
+         aria-invalid={hasError || undefined}
+         aria-describedby={errorText ? errorId : undefined}
        />
-       <A11yError errorText={errorText ?? ""} />
+       <A11yError id={errorId} errorText={errorText ?? ""} />
      </div>
    );
  }
);

A11yInput.displayName = "A11yInput";

```

:::

#### 完成した Input コンポーネント

エラーが発生した際に、スクリーンリーダが適切にエラーを表示させることができるように実装することができました。

:::details 最終的な Input コンポーネント（スタイルなし）

```js
"use client";

import { forwardRef, useId } from "react";
import { A11yError } from "./error";

interface A11yInputProps extends React.ComponentPropsWithoutRef<"input"> {
  label?: string;
  errorText?: string;
}

export const A11yInput = forwardRef<HTMLInputElement, A11yInputProps>(
  ({ label, errorText, ...rest }, ref) => {
    const autoId = useId();
    const formId = rest.id ?? autoId;
    const errorId = `${formId}-error`;
    const hasError = Boolean(errorText);

    return (
      <div>
        {label && <label htmlFor={formId}>{label}</label>}
        <input
          {...rest}
          ref={ref}
          id={formId}
          aria-invalid={hasError || undefined}
          aria-describedby={errorText ? errorId : undefined}
        />
        <A11yError id={errorId} errorText={errorText ?? ""} />
      </div>
    );
  }
);

A11yInput.displayName = "A11yInput";

```

:::

### 3-5. その他のフォーム

これ以外に Select コンポーネントと Checkbox コンポーネントを作成しました。
基本的な構造は Input と同様なので、ここでは完成形だけ載せておきます。

:::details 最終的な Select コンポーネント（スタイルなし）

```js
"use client";

import { forwardRef, useId } from "react";
import { A11yError } from "./error";

interface Option {
  value: string | number;
  label: string;
}

interface A11ySelectProps extends React.ComponentPropsWithoutRef<"select"> {
  options: Option[];
  label?: string;
  errorText?: string;
}

export const A11ySelect = forwardRef<HTMLSelectElement, A11ySelectProps>(
  ({ options, label, errorText, ...rest }, ref) => {
    const autoId = useId();
    const formId = rest.id ?? autoId;
    const errorId = `${formId}-error`;
    const hasError = Boolean(errorText);

    return (
      <div>
        {label && <label htmlFor={formId}>{label}</label>}
        <select
          {...rest}
          ref={ref}
          id={formId}
          aria-invalid={hasError || undefined}
          aria-describedby={errorText ? errorId : undefined}
        >
          <option value="">選択してください</option>
          {options.map((option) => (
            <option key={option.value} value={option.value}>
              {option.label}
            </option>
          ))}
        </select>
        <A11yError id={errorId} errorText={errorText ?? ""} />
      </div>
    );
  }
);

A11ySelect.displayName = "A11ySelect";
```

:::

:::details 最終的な Checkbox コンポーネント（スタイルなし）

```js
"use client";

import { forwardRef, useId } from "react";
import { A11yError } from "./error";

interface A11yCheckboxProps extends React.ComponentPropsWithoutRef<"input"> {
  label?: string;
  errorText?: string;
}

export const A11yCheckbox = forwardRef<HTMLInputElement, A11yCheckboxProps>(
  ({ label, errorText, ...rest }, ref) => {
    const autoId = useId();
    const formId = rest.id ?? autoId;
    const errorId = `${formId}-error`;
    const hasError = Boolean(errorText);

    return (
      <div>
        <input
          type="checkbox"
          {...rest}
          ref={ref}
          id={formId}
          aria-invalid={hasError || undefined}
          aria-describedby={errorText ? errorId : undefined}
        />
        {label && <label htmlFor={formId}>{label}</label>}
        <A11yError id={errorId} errorText={errorText ?? ""} />
      </div>
    );
  }
);

A11yCheckbox.displayName = "A11yCheckbox";

```

:::

### 3-6. Form コンポーネント

ここでは、form タグの機能を表現するようなコンポーネントを作成します。
例の如く、土台だけ作成しています。現状では、ただ div タグでラップするだけのコンポーネントになっています。無限に包みたくなりますね。

:::details Form コンポーネントの土台

```js
"use client";

import { forwardRef } from "react";

interface A11yFormProps extends React.HTMLAttributes<HTMLDivElement> {
  children: React.ReactNode;
}

export const A11yForm = forwardRef<HTMLDivElement, A11yFormProps>(
  ({ children, ...rest }, ref) => {
    return (
      <div
        {...rest}
        ref={ref}
      >
        {children}
      </div>
    );
  }
);

A11yForm.displayName = "A11yForm";

```

:::

#### 支援技術にフォームであることを伝える

Button コンポーネントや Dialog コンポーネントと同様に、`role='form'` を付与することで、支援技術にフォームであることを伝えます。
加えて、`aria-label` を設定することで、スクリーンリーダは「ラベル名、フォーム」のように読み上げてくれるようになります。スクリーンリーダにはページ内のフォームを一覧表示してジャンプする機能があるものも多く、この設定により目的のフォームを見つけやすくなります。

:::details role 属性と aria-label を付与

```diff
"use client";

import { forwardRef } from "react";

interface A11yFormProps extends React.HTMLAttributes<HTMLDivElement> {
  onSubmit: () => void;
+ ariaLabel?: string;
  children: React.ReactNode;
}

export const A11yForm = forwardRef<HTMLDivElement, A11yFormProps>(
- ({ children, ...rest }, ref) => {
+ ({ children, ariaLabel, ...rest }, ref) => {
    return (
      <div
        {...rest}
        ref={ref}
+       role="form"
+       aria-label={ariaLabel}
      >
        {children}
      </div>
    );
  }
);

A11yForm.displayName = "A11yForm";

```

:::

#### フォーム内フォーカス時に、Enter キー押下で Submit できるようにする

ネイティブの form タグでは、フォーム内の input 要素などにフォーカスがある状態で Enter キーを押下すると、フォームが送信されます。この挙動を div タグで再現するために、`onKeyDown` イベントを設定します。
実装のポイントとして、`e.nativeEvent.isComposing` のチェックを行っています。これは、日本語入力などの変換確定時に Enter キーが押されることを考慮したもので、変換中の Enter キー押下では送信されないようにしています。

:::details Enter キー押下で Submit できるようにする

```diff
"use client";

import { forwardRef } from "react";

interface A11yFormProps extends React.HTMLAttributes<HTMLDivElement> {
+ onSubmit: () => void;
  ariaLabel?: string;
  children: React.ReactNode;
}

export const A11yForm = forwardRef<HTMLDivElement, A11yFormProps>(
- ({ children, ariaLabel, ...rest }, ref) => {
+ ({ onSubmit, children, ariaLabel, ...rest }, ref) => {
    return (
      <div
        {...rest}
        ref={ref}
        role="form"
        aria-label={ariaLabel}
+       onKeyDown={(e) => {
+         if (
+           e.key === "Enter" &&
+           !e.nativeEvent.isComposing &&
+           (e.target instanceof HTMLInputElement ||
+             e.target instanceof HTMLSelectElement)
+         ) {
+           onSubmit();
+         }
+         rest.onKeyDown?.(e);
+       }}
      >
        {children}
      </div>
    );
  }
);

A11yForm.displayName = "A11yForm";

```

:::

#### 完成した Form コンポーネント

ここまでの実装で、div タグを使ってフォームとしての基本的な機能を持たせることができました。スクリーンリーダにフォームとして認識され、キーボードからの Enter キー送信にも対応しています。

フォーム関連の実装がひと通り終わったので、動作を確認します。
今回はスクリーンリーダでの表示も確認できます。

![Form コンポーネントの動作確認](/images/a11y-div-challenge/form.gif)

:::details 最終的な Form コンポーネント

```js
"use client";

import { forwardRef } from "react";

interface A11yFormProps extends React.HTMLAttributes<HTMLDivElement> {
  onSubmit: () => void;
  ariaLabel?: string;
  children: React.ReactNode;
}

export const A11yForm = forwardRef<HTMLDivElement, A11yFormProps>(
  ({ onSubmit, children, ariaLabel, ...rest }, ref) => {
    return (
      <div
        {...rest}
        ref={ref}
        role="form"
        aria-label={ariaLabel}
        onKeyDown={(e) => {
          if (
            e.key === "Enter" &&
            !e.nativeEvent.isComposing &&
            (e.target instanceof HTMLInputElement ||
              e.target instanceof HTMLSelectElement)
          ) {
            onSubmit();
          }
          rest.onKeyDown?.(e);
        }}
      >
        {children}
      </div>
    );
  }
);

A11yForm.displayName = "A11yForm";

```

:::

## 4. 完成品

ここまで汎用的に使用できるコンポーネントを作成してきましたが、最後にこれらを組み合わせて記事の始めにお見せしたログインフォームを実装します。

本記事の冒頭でお見せした完成品デモを再掲しておきます。

![完成品の動作確認（メイン画面からログイン）](/images/a11y-div-challenge/comp1.gif)

![完成品の動作確認（ダイアログからログイン）](/images/a11y-div-challenge/comp2.gif)

実装内容は以下の通りです。

:::details 完成したログインフォーム

```js
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { A11yDialog } from "./_components/dialog";
import { A11yButton } from "./_components/button";
import { A11yInput } from "./_components/input";
import { A11ySelect } from "./_components/select";
import { A11yCheckbox } from "./_components/checkbox";
import { A11yForm } from "./_components/form";
import { loginSchema, type LoginFormValues } from "./schema";

const languageOptions = [
  { value: "ja", label: "日本語" },
  { value: "en", label: "English" },
  { value: "zh", label: "中文" },
];

export default function Sample() {
  const [isOpen, setIsOpen] = useState(false);
  const router = useRouter();

  const handleLogin = () => {
    router.push("/sample-complete");
  };

  return (
    <div>
      <div role="banner">
        <A11yButton onClick={() => setIsOpen(true)}>
          ログインダイアログを開く
        </A11yButton>
      </div>
      <div role="main">
        <LoginForm onSubmit={handleLogin} />
      </div>
      {/* <div role="contentinfo">フッターはここ</div> */}
      <A11yDialog
        open={isOpen}
        onClose={() => setIsOpen(false)}
        title="ログイン"
      >
        <LoginForm onSubmit={handleLogin} />
      </A11yDialog>
    </div>
  );
}

const LoginForm = ({ onSubmit }: { onSubmit: () => void }) => {
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<LoginFormValues>({
    resolver: zodResolver(loginSchema),
  });

  return (
    <A11yForm onSubmit={handleSubmit(onSubmit)} ariaLabel="ログインフォーム">
      <A11yInput
        label="メールアドレス"
        type="email"
        errorText={errors.email?.message}
        {...register("email")}
      />
      <A11yInput
        label="パスワード"
        type="password"
        errorText={errors.password?.message}
        {...register("password")}
      />
      <A11ySelect
        label="言語"
        options={languageOptions}
        errorText={errors.language?.message}
        {...register("language")}
      />
      <A11yCheckbox
        label="ログイン状態を保持する"
        {...register("rememberMe")}
      />
      <A11yButton onClick={handleSubmit(onSubmit)}>
        ログイン
      </A11yButton>
    </A11yForm>
  );
}
```

:::

### 4.1 検証

フォームを作成するセクションでスクリーンリーダを用いた動作検証を行い、想定通りの出力をしていることを確認できました。

それに加えて、Chromeに搭載されている Lighthouse というツールを用いてアクセシビリティの検証を行いました。今回作成したアプリのスコアを出力させてみたところ、満点をいただくことができました。

![Lighthouseでの効果検証1](/images/a11y-div-challenge/a11y-score1.png)

規模感が小さいために高得点になっている可能性もあるなと感じ、アクセシビリティ対策関連の記述だけを全て除いたページを新しく作成して、再度スコアを出力させてみたところ、85点と15点も下がってしまいました。フォーム周りの設定と、見出しがページに設定されていないことが問題のようですね。少なからず、今回の取り組みによってアクセシブルなUIを作成することができたということにしておきます。

![Lighthouseでの効果検証2](/images/a11y-div-challenge/a11y-score2.png)

## まとめ

これまで何となく使っていたWAI-ARIAを、明確に意図を持って使用できるようになってきたような気がしました。あくまで実験的な実装ではあるので、まだまだ磨ける部分や修正しないといけない部分すらあるかと思いますが、とにかくWAI-ARIAを肌身で感じることができたような気がしました。
それに加えて、支援技術を使用するのは初めての経験だったので、エンジニアとして働いていながらも、まだまだ知らないことが多いなと改めて感じました。それと同時に、様々なユーザにできる限り不便なく使用してもらえるWEBアプリを作ることができるようなエンジニアを目指さなければならないなとも感じました。（巷でよく耳にする、プログレッシブエンハンスメント等のユーザビリティ領域も含め）


↓ 今回作成したアプリのリポジトリ

https://github.com/IJproject/a11y-challenge-app
