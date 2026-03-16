---
title: "型エイリアスとインターフェース"
---

# 型エイリアスとインターフェース

> `type` と `interface` の違いを理解し、場面に応じて使い分ける

## この章で学ぶこと

- `type`（型エイリアス）の定義方法と特徴
- `interface` の定義方法と特徴
- 構造的型付け（Structural Typing）の仕組み
- `extends` と Intersection（`&`）の違い
- Declaration Merging（宣言マージ）
- `type` と `interface` の使い分けの指針

## type（型エイリアス）

`type` キーワードを使うと、型に名前を付けることができます。

```typescript
// オブジェクト型
type User = {
  id: number;
  name: string;
  email: string;
};

// ユニオン型
type Status = "active" | "inactive" | "suspended";

// 関数型
type Formatter = (input: string) => string;

// タプル型
type Coordinate = [number, number];
```

型エイリアスはオブジェクト型だけでなく、ユニオン型、関数型、タプル型など、あらゆる型に名前を付けられます。

ジェネリクスと組み合わせて、再利用性の高い型も作れます（ジェネリクスについてはChapter 09で詳しく扱います）。

```typescript
type Result<T> =
  | { success: true; data: T }
  | { success: false; error: string };
```

### Intersection（`&`）による型の合成

`&`（Intersection型）を使って、複数の型を合成できます。

```typescript
type HasName = { name: string };
type HasEmail = { email: string };

type Person = HasName & HasEmail & { age: number };
// { name: string; email: string; age: number }
```

Intersection型は「すべての型のプロパティを持つ」型を作ります。

## interface

`interface` キーワードを使うと、オブジェクトの構造を定義できます。

```typescript
interface User {
  readonly id: number;   // 読み取り専用
  name: string;          // 必須プロパティ
  email: string;
  age?: number;          // オプショナルプロパティ
}

const user: User = {
  id: 1,
  name: "Alice",
  email: "alice@example.com",
};

// user.id = 2; // Error: readonly プロパティは変更できない
```

### `extends` による継承

`interface` は `extends` で他のインターフェースを継承できます。

```typescript
interface Employee extends User {
  department: string;
  salary: number;
}
```

複数のインターフェースを同時に継承することも可能です。

```typescript
interface Serializable { serialize(): string; }
interface Printable { print(): void; }

interface Document extends Serializable, Printable {
  title: string;
}
```

## 構造的型付け（Structural Typing）

TypeScriptでは **構造（プロパティの名前と型）が一致していれば、その型として扱えます**。JavaやC#のように `implements` を宣言する必要はありません。

```typescript
interface HasName {
  name: string;
}

function greet(obj: HasName): string {
  return `Hello, ${obj.name}!`;
}

// HasName を implements していないが、name を持っているのでOK
const user = { name: "Alice", email: "alice@example.com" };
const dog = { name: "Buddy", breed: "Labrador" };

greet(user); // OK
greet(dog);  // OK
```

「必要なプロパティを全て持っていれば、その型として扱える」 -- これが構造的型付けの基本原則です。

### 余剰プロパティチェック

ただし、オブジェクトリテラルを直接渡す場合は例外的に厳しいチェックが入ります。

```typescript
interface Point {
  x: number;
  y: number;
}

// Error: オブジェクトリテラルに余分な z がある
// const p: Point = { x: 10, y: 20, z: 30 };

// OK: 変数経由なら余分なプロパティは無視される
const obj = { x: 10, y: 20, z: 30 };
const p: Point = obj;
```

この「余剰プロパティチェック」は、タイプミスを検出するための仕組みです。オブジェクトリテラルを直接代入するときだけ発動します。

## `extends` と Intersection の違い

`interface` の `extends` と `type` の `&` は似ていますが、型の衝突時の挙動が異なります。

```typescript
// extends: 衝突するとコンパイルエラー（安全）
interface A { value: number; }
// interface B extends A {
//   value: string; // Error: number と string は互換性がない
// }

// Intersection(&): 衝突すると never になる（エラーにはならない）
type X = { value: number };
type Y = { value: string };
type Z = X & Y;
// Z の value は number & string = never（実質使えない）
```

`extends` は矛盾を早期に検出できるため、オブジェクトの継承には `extends` の方が安全です。

## Declaration Merging（宣言マージ）

`interface` は同じ名前で複数回宣言すると自動的にマージされます。`type` にはこの機能はありません。

```typescript
interface Config {
  host: string;
  port: number;
}

interface Config {
  debug: boolean; // マージされる
}

// 結果: { host: string; port: number; debug: boolean }
const config: Config = {
  host: "localhost",
  port: 3000,
  debug: true,
};
```

Declaration Merging は、既存ライブラリの型を拡張するときに特に役立ちます。

```typescript
// Express の Request にカスタムプロパティを追加
declare global {
  namespace Express {
    interface Request {
      user?: { id: string; name: string };
    }
  }
}
```

なお、`type` は同じ名前で再宣言するとエラーになります。

```typescript
type Status = "active" | "inactive";
// type Status = "suspended"; // Error: 重複した識別子
```

## typeof 型演算子

JavaScriptの `typeof` は実行時に値の型を文字列で返す演算子ですが、TypeScriptではもう一つ、**型レベルの `typeof`** が使えます。変数から型を取り出したいときに便利です。

```typescript
// 値レベルの typeof（型ガード、Chapter 07 で詳しく解説）
if (typeof value === "string") { /* ... */ }

// 型レベルの typeof（変数の型を取得して型定義に使う）
const user = { name: "Alice", age: 30 };
type User = typeof user; // { name: string; age: number }
```

型レベルの `typeof` は、型注釈の中（`type` の右辺や引数の型など）でのみ使えます。既存の値から型を自動生成できるため、型定義の重複を減らすのに役立ちます。

```typescript
const config = { host: "localhost", port: 3000 };
type Config = typeof config; // { host: string; port: number }

function startServer(c: Config) { /* ... */ }
```

## keyof 演算子

`keyof` は、オブジェクト型のプロパティ名をユニオン型として取り出す型演算子です。

```typescript
type User = { name: string; age: number; email: string };
type UserKey = keyof User; // "name" | "age" | "email"
```

`keyof` を使うと、オブジェクトのキーだけを受け取る関数を型安全に書けます。

```typescript
function getValue<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "Alice", age: 30 };
getValue(user, "name");  // OK: string
// getValue(user, "foo"); // Error: "foo" は keyof typeof user に含まれない
```

`typeof` と `keyof` を組み合わせると、**値として定義したオブジェクトからキーのユニオン型を取り出す**こともできます。

```typescript
const palette = { primary: "#8b5cf6", secondary: "#06b6d4", danger: "#ef4444" } as const;
type ColorName = keyof typeof palette; // "primary" | "secondary" | "danger"
```

## type と interface の使い分け

オブジェクト型の定義に関しては、`type` と `interface` は互換性が高く、どちらを使っても問題ないケースが多いです。

**interface が適しているケース:**

- オブジェクトの構造を定義するとき
- `extends` で継承したいとき
- Declaration Merging が必要なとき

```typescript
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
}
```

**type が適しているケース:**

- ユニオン型を定義するとき
- 関数型やタプル型を定義するとき
- ユーティリティ型を組み合わせるとき

```typescript
type Role = "admin" | "editor" | "viewer";
type Handler = (req: Request, res: Response) => void;
type CreateInput<T> = Omit<T, "id" | "createdAt">;
```

### チーム内の統一が大切

どちらのスタイルも広く使われています。迷ったら「オブジェクトの形状には `interface`、それ以外には `type`」から始めてみてください。どちらを選ぶかよりも、チーム内で統一することの方が重要です。

```typescript
// スタイルA: オブジェクトは interface、それ以外は type
interface User { id: number; name: string; }
type Role = "admin" | "editor" | "viewer";

// スタイルB: すべて type で統一
type User = { id: number; name: string; };
type Role = "admin" | "editor" | "viewer";
```

### 機能比較表

| 機能 | `type` | `interface` |
|------|--------|-------------|
| オブジェクト型の定義 | OK | OK |
| ユニオン型の定義 | OK | 不可 |
| 拡張 | `&`（Intersection） | `extends`（継承） |
| Declaration Merging | 不可 | OK（自動マージ） |
| 条件型・Mapped Types | OK | 不可 |

## まとめ

| 概念 | 要点 |
|------|------|
| `type`（型エイリアス） | あらゆる型に名前を付けられる。ユニオン型や関数型に必須 |
| `interface` | オブジェクトの構造を定義する。`extends` で継承できる |
| 構造的型付け | 名前ではなく構造（プロパティ）で型の互換性を判定する |
| 余剰プロパティチェック | オブジェクトリテラルを直接渡すときだけ発動する |
| `extends` vs `&` | `extends` は衝突時にエラー、`&` は衝突時に `never` |
| Declaration Merging | `interface` は同名で再宣言すると自動マージされる |
| `typeof`（型レベル） | 変数から型を取り出す。型定義の重複を減らせる |
| `keyof` | オブジェクト型のプロパティ名をユニオン型として取得する |
| 使い分け | オブジェクト形状には `interface`、それ以外には `type` が一般的 |

## やってみよう！

### 練習1: type と interface で同じ型を定義する

以下の仕様を満たす `Product` 型を `type` と `interface` の両方で定義してみましょう。

- `id`: `number`（読み取り専用）
- `name`: `string`
- `price`: `number`
- `category`: `"food" | "drink" | "other"`
- `description`: `string`（オプショナル）

```typescript
// type で定義
type ProductType = /* ここに書いてみましょう */;

// interface で定義
interface ProductInterface {
  /* ここに書いてみましょう */
}
```

### 練習2: interface の継承を使う

`Animal` インターフェースを定義し、それを継承した `Dog` インターフェースを作ってみましょう。

- `Animal`: `name`（string）, `age`（number）
- `Dog`: `Animal` を継承 + `breed`（string）, `bark()` メソッド（戻り値 `void`）

```typescript
interface Animal {
  /* ここに書いてみましょう */
}

interface Dog extends Animal {
  /* ここに書いてみましょう */
}

const myDog: Dog = {
  name: "Buddy",
  age: 3,
  breed: "Labrador",
  bark() { console.log("Woof!"); },
};
```

### 練習3: 構造的型付けを体験する

以下のコードで、`printLength` に渡せるものと渡せないものを考えてみましょう。

```typescript
interface HasLength {
  length: number;
}

function printLength(obj: HasLength): void {
  console.log(`Length: ${obj.length}`);
}

const str = "hello";
const arr = [1, 2, 3];
const num = 42;
const obj = { length: 10, width: 20 };

// printLength(str);  // ?
// printLength(arr);  // ?
// printLength(num);  // ?
// printLength(obj);  // ?
```

ヒント: `length` プロパティを持っているかどうかがポイントです。
