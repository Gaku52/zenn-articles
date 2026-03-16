---
title: "TypeScriptとは"
---

# TypeScriptとは

> JavaScriptに静的型付けを追加し、コードの安全性と開発体験を向上させるスーパーセット言語

## この章で学ぶこと

- TypeScriptとJavaScriptの関係
- スーパーセットとは何か
- 型システムがもたらす具体的なメリット
- コンパイル（型消去）の仕組み

---

## TypeScriptの概要

TypeScriptはMicrosoftが2012年に公開したオープンソースの言語です。C#の設計者であるAnders Hejlsbergが主導し、JavaScriptに**静的型付け**を追加することを目的に開発されました。

現在、TypeScriptはJavaScriptエコシステムのデファクトスタンダードです。React、Next.js、Angular、Vue、Hono、Prismaなど、主要なフレームワークやライブラリがTypeScriptをサポートしています。

## JavaScriptのスーパーセット

TypeScriptを理解するうえで最も重要な概念が「スーパーセット」です。すべての正しいJavaScriptコードは、そのままTypeScriptとしても有効です。TypeScriptはそこに型の仕組みを追加します。

```
+------------------------------------------+
|            TypeScript                     |
|  +------------------------------------+  |
|  |          JavaScript                |  |
|  |  +------------------------------+  |  |
|  |  |       ECMAScript仕様         |  |  |
|  |  +------------------------------+  |  |
|  +------------------------------------+  |
|  + 型アノテーション                     |
|  + インターフェース                     |
|  + ジェネリクス                         |
|  + 列挙型                               |
|  + その他の型機能                       |
+------------------------------------------+
```

この関係は実践的に大きな利点をもたらします。

- **既存のJavaScriptファイルをそのまま `.ts` に変更**して、段階的に型を追加できます
- **JavaScriptライブラリをそのまま利用**でき、`@types/xxx` パッケージを追加すれば型補完も得られます
- 新しい言語をゼロから学ぶ必要がなく、**JavaScriptの知識がそのまま活きます**

### 型アノテーションを追加するとどうなるか

JavaScriptとして有効なコードと、そこにTypeScriptの型を追加したコードを見比べてみましょう。

```typescript
// これは有効なJavaScriptであり、同時に有効なTypeScriptでもある
const greet = (name) => `Hello, ${name}!`;
console.log(greet("World"));
```

```typescript
// 型アノテーションを追加した TypeScript コード
const greet = (name: string): string => `Hello, ${name}!`;

// greet(42);  // Error: Argument of type 'number' is not assignable to parameter of type 'string'
console.log(greet("World")); // OK
```

`name: string` と `: string` の部分が型アノテーションです。これにより、`greet(42)` のような誤った呼び出しをコード実行前に検出できます。

## 型システムがもたらすメリット

TypeScriptの型システムは、大きく分けて4つのメリットを提供します。

### バグの早期発見

JavaScriptでは実行するまで気づかないバグを、TypeScriptではコンパイル時に検出できます。

```typescript
// JavaScript の場合: 実行時まで気づかない
function calculateArea(width, height) {
  return width * height;
}
calculateArea("10", 20); // "1020" -- 文字列連結が起きる（意図しない動作）

// TypeScript の場合: コンパイル時に検出
function calculateArea(width: number, height: number): number {
  return width * height;
}
// calculateArea("10", 20); // Error: Argument of type 'string' is not assignable to parameter of type 'number'
calculateArea(10, 20); // 200 -- 正しい結果
```

型チェックが防止するバグのパターンは多岐にわたります。

```typescript
// null/undefined アクセスの防止
function getLength(str: string | null): number {
  // str.length;  // Error: Object is possibly 'null'
  if (str === null) return 0;
  return str.length; // OK: null チェック後は安全
}

// 存在しないプロパティへのアクセス防止
interface Config {
  host: string;
  port: number;
}

function createConnection(config: Config) {
  // config.hostname;  // Error: Property 'hostname' does not exist
  return `${config.host}:${config.port}`; // OK
}

// switch 文の網羅性チェック
type Shape = "circle" | "square" | "triangle";

function getArea(shape: Shape, size: number): number {
  switch (shape) {
    case "circle":
      return Math.PI * size * size;
    case "square":
      return size * size;
    case "triangle":
      return (Math.sqrt(3) / 4) * size * size;
    // 新しい Shape を追加したとき、case を書き忘れるとコンパイラが警告する
  }
}
```

### IDE による開発支援

型情報があることで、VSCodeなどのエディタが正確な自動補完、ホバー時の型情報表示、定義ジャンプなどを提供できます。

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  createdAt: Date;
}

function displayUser(user: User) {
  // user. と入力した時点で id, name, email, createdAt が候補に表示される
  console.log(`${user.name} <${user.email}>`);
}

const users = [
  { id: 1, name: "Alice", age: 30 },
  { id: 2, name: "Bob", age: 25 },
];
// users にホバーすると { id: number; name: string; age: number; }[] と表示される
```

### リファクタリングの安全性

プロパティ名の変更やインターフェースの修正を行うと、コンパイラが影響を受ける全箇所をエラーとして報告します。たとえば `Product` インターフェースの `price` を `priceInCents` にリネームすると、`price` を参照しているすべての関数でコンパイルエラーが発生します。修正漏れが起きません。

### ドキュメントとしての型

型定義はコードの意図を伝える「生きたドキュメント」として機能します。コメントと違い、コードの変更に追従しないと型エラーになるため、陳腐化しません。

### メリットの比較表

| 観点 | JavaScript | TypeScript |
|------|-----------|-----------|
| バグ検出 | 実行時（本番含む） | コンパイル時（開発中） |
| リファクタリング | 手動で全箇所確認 | コンパイラが影響範囲を自動検出 |
| IDE補完 | 推測ベース（不正確） | 型情報ベース（正確） |
| ドキュメント | コメント（陳腐化しやすい） | 型が生きたドキュメントになる |
| チーム開発 | 口頭・ドキュメント依存 | 型がコントラクトとして機能 |

## コンパイルの仕組み

TypeScriptのコードは、そのままではブラウザやNode.jsで実行できません。TypeScriptコンパイラ（`tsc`）によってJavaScriptに変換されます。

```
  TypeScript ソースコード (.ts)
         |
         v
  +-------------------+
  | TypeScript         |
  | コンパイラ (tsc)    |
  +-------------------+
         |
    +----+----+
    |         |
    v         v
 JavaScript  型エラー
 (.js)       レポート
```

### 型消去（Type Erasure）

コンパイルの最も重要な特徴は「型消去」です。型の情報はコンパイル時にすべて取り除かれ、実行時には存在しません。

```typescript
// TypeScript ソース
interface User {
  id: number;
  name: string;
}

function getUser(id: number): User {
  return { id, name: "Alice" };
}

const user: User = getUser(1);
console.log(user.name);
```

```javascript
// コンパイル後の JavaScript（型情報が完全に消去される）
"use strict";
function getUser(id) {
  return { id, name: "Alice" };
}
const user = getUser(1);
console.log(user.name);

// interface User は完全に消えている
// 引数の型・戻り値の型・変数の型アノテーションもすべて消えている
```

型消去があるため、TypeScriptは**実行時のパフォーマンスに一切影響を与えません**。コンパイル後はただのJavaScriptです。

:::message
**型消去されないTypeScript構文もある**

`enum`、`namespace`、クラスのパラメータプロパティなど、一部の構文はコンパイル後にJavaScriptのコードとして残ります。TypeScript 5.7で導入された `--erasableSyntaxOnly` オプションを使うと、型消去だけで処理できる構文に限定することもできます。なお、`enum` の代替として `as const` オブジェクトパターンが推奨されることがあります（Chapter 07参照）。
:::

## TypeScriptのバージョンと現在

TypeScriptは約3か月ごとにマイナーバージョンがリリースされます。本書はTypeScript 5.7に対応しています。

近年の注目すべき進化を紹介します。

| バージョン | 主な機能 |
|-----------|---------|
| 4.9 (2022) | `satisfies` 演算子 |
| 5.0 (2023) | デコレータ (Stage 3)、`const` 型パラメータ |
| 5.2 (2023) | `using` 宣言（Explicit Resource Management） |
| 5.5 (2024) | 型述語の推論、正規表現の構文チェック |
| 5.7 (2025) | `--erasableSyntaxOnly`、Node.jsネイティブ実行対応強化 |

Node.js 22.6.0以降では `--experimental-strip-types` フラグにより、TypeScriptファイルを直接実行できるようになっています。

## TypeScriptの設計思想

TypeScriptの公式Design Goalsから、重要なポイントを紹介します。

**目指すこと:**
- コンパイル時にコードの構造的な不整合を検出する
- 実行時にオーバーヘッドを課さない（型消去）
- 出力されるJavaScriptは明快で慣用的なものにする
- 現在と将来のECMAScript仕様に沿った言語であり続ける

**目指さないこと:**
- 100%安全（sound）な型システムの追求 -- 実用性を優先します
- JavaScriptプログラムの高速化 -- 型は実行時に消えます

TypeScriptは「実用性と安全性のバランス」を設計の軸としています。そのため、`any` 型による型システムからの脱出口も意図的に用意されています。

## まとめ

| 概念 | 要点 |
|------|------|
| TypeScript | JavaScriptに静的型付けを追加したスーパーセット言語 |
| スーパーセット | すべてのJavaScriptはそのままTypeScriptとして有効 |
| 型のメリット | バグ早期発見、IDE支援、リファクタリング安全性、ドキュメント効果 |
| コンパイル | `tsc` が `.ts` を `.js` に変換。型情報は消去される |
| 型消去 | 型は実行時に存在しない。パフォーマンスへの影響はゼロ |
| 設計思想 | 実用性と安全性のバランス。JavaScript互換性の維持 |
| 現在のバージョン | TypeScript 5.7（本書対応版） |

## やってみよう！

### 練習1: 型アノテーションを追加する

以下のJavaScriptコードにTypeScriptの型アノテーションを追加してください。

```typescript
// Before: 型なし
const add = (a, b) => a + b;
const result = add(10, 20);
console.log(result);
```

:::details 解答例
```typescript
const add = (a: number, b: number): number => a + b;
const result: number = add(10, 20);
console.log(result); // 想定出力例: 30
```
`result` の型アノテーションは省略しても、TypeScriptが `number` と推論してくれます。戻り値型も推論可能ですが、関数の意図を明確にするために書くことが多いです。
:::

### 練習2: 型エラーを見つける

以下のコードにはTypeScriptの型エラーがあります。どこがエラーになるか、理由とともに考えてみてください。

```typescript
interface Book {
  title: string;
  pages: number;
  published: boolean;
}

const myBook: Book = {
  title: "TypeScript入門",
  pages: "320",
  published: true,
};
```

:::details 解答例
`pages` プロパティに `"320"`（文字列）を代入しています。`Book` インターフェースでは `pages: number` と定義されているため、型エラーになります。正しくは `pages: 320`（数値）とします。
:::

### 練習3: 型消去を想像する

以下のTypeScriptコードがコンパイルされると、どのようなJavaScriptになるか考えてみてください。

```typescript
interface Point {
  x: number;
  y: number;
}

function distance(a: Point, b: Point): number {
  return Math.sqrt((a.x - b.x) ** 2 + (a.y - b.y) ** 2);
}

const p1: Point = { x: 0, y: 0 };
const p2: Point = { x: 3, y: 4 };
console.log(distance(p1, p2));
```

:::details 解答例
```javascript
"use strict";
function distance(a, b) {
  return Math.sqrt((a.x - b.x) ** 2 + (a.y - b.y) ** 2);
}
const p1 = { x: 0, y: 0 };
const p2 = { x: 3, y: 4 };
console.log(distance(p1, p2)); // 想定出力例: 5
```
`interface Point` は完全に消え、型アノテーションも消えます。ロジック部分はそのまま残ります。
:::
