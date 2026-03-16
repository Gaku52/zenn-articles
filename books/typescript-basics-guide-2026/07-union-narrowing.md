---
title: "ユニオン型と型の絞り込み"
---

# ユニオン型と型の絞り込み

> 「この値は文字列か数値のどちらかです」を型で表現し、安全に使い分ける

## ユニオン型とは

ユニオン型（Union Types）は、ある値が「複数の型のいずれか」であることを表す仕組みです。`|`（パイプ）演算子で型をつなげて定義します。

```typescript
// string または number を受け取る関数
function formatId(id: string | number): string {
  if (typeof id === "string") {
    return id.toUpperCase();
  }
  return id.toString().padStart(6, "0");
}

formatId("abc"); // "ABC"
formatId(42);    // "000042"
```

変数にもユニオン型を使えます。

```typescript
let value: string | number | boolean;
value = "hello"; // OK
value = 42;      // OK
value = true;    // OK
// value = [];   // Error: 型 'never[]' を型 'string | number | boolean' に割り当てることはできません
```

## ユニオン型のメンバーアクセス

ユニオン型の変数に対しては、**すべての構成型に共通するメンバー**にのみアクセスできます。これが型安全性を保つための重要なルールです。

```typescript
function describe(value: string | number): string {
  // toString() は string にも number にもあるので OK
  return value.toString();

  // value.toUpperCase() → Error（number には toUpperCase がない）
  // value.toFixed(2)    → Error（string には toFixed がない）
}
```

どちらか一方にしかないメソッドを使いたい場合は、**型の絞り込み（narrowing）** が必要です。これが本章の主題です。

## リテラル型のユニオン

リテラル型をユニオンで組み合わせると、「決まった値のいずれか」を表現できます。`enum` の代替としてよく使われるパターンです。

```typescript
type Direction = "north" | "south" | "east" | "west";
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";

function move(direction: Direction): void {
  console.log(`Moving ${direction}`);
}

move("north"); // OK
// move("up"); // Error: 型 '"up"' を型 'Direction' に割り当てることはできません

// 数値リテラルのユニオンも定義できる
type DiceValue = 1 | 2 | 3 | 4 | 5 | 6;
```

`enum` でも同様のことができますが、リテラル型ユニオンや `as const` パターンの方がTree-shakingに有利で、生成されるJavaScriptもシンプルです。

```typescript
// as const パターン: enum の代替として推奨されることが多い
const Direction = { North: "north", South: "south", East: "east", West: "west" } as const;
type Direction = (typeof Direction)[keyof typeof Direction];
```

## typeof による型ガード

`typeof` 演算子を `if` 文で使うと、TypeScript はブロック内で変数の型を自動的に絞り込みます。この仕組みを**型ガード（type guard）** と呼びます。

```typescript
function padValue(value: string | number): string {
  if (typeof value === "string") {
    // このブロックでは value は string 型
    return value.padStart(10, " ");
  }
  // ここでは value は number 型
  return value.toFixed(2);
}
```

`typeof` で判定できるのは `"string"`, `"number"`, `"boolean"`, `"bigint"`, `"symbol"`, `"undefined"`, `"function"`, `"object"` の8種類です。

注意点として、`typeof null` は `"object"` を返します。これは JavaScript の歴史的な仕様です。`null` の判定には `=== null` を使いましょう。

## instanceof による型ガード

`instanceof` は、値がクラスのインスタンスかどうかを判定します。`Date` や `Error` などの組み込みクラスや、自作のクラスに対して使えます。

```typescript
function formatError(error: Error | string): string {
  if (error instanceof Error) {
    // Error 型に絞り込まれる
    return `${error.name}: ${error.message}`;
  }
  // string 型に絞り込まれる
  return error;
}

formatError(new TypeError("型が違います")); // "TypeError: 型が違います"
formatError("何かのエラー");                 // "何かのエラー"
```

## in 演算子による型ガード

`in` 演算子は、オブジェクトに特定のプロパティが存在するかを判定します。クラスを使わないオブジェクト型のユニオンに便利です。

```typescript
interface Dog {
  bark(): void;
  breed: string;
}

interface Cat {
  meow(): void;
  indoor: boolean;
}

function speak(pet: Dog | Cat): void {
  if ("bark" in pet) {
    pet.bark();  // Dog 型に絞り込まれる
  } else {
    pet.meow();  // Cat 型に絞り込まれる
  }
}
```

## truthy チェックによる絞り込み

TypeScript は truthy / falsy チェックでも型を絞り込みます。`null` や `undefined` を除外する場面で便利です。

```typescript
function greet(name: string | null | undefined): string {
  if (!name) {
    return "Hello, Guest!";
  }
  // name は string 型に絞り込まれる
  return `Hello, ${name.toUpperCase()}!`;
}
```

ただし、`""` や `0` も falsy です。空文字列や 0 を有効な値として扱いたい場合は `!= null` を使いましょう。

```typescript
function showCount(count: number | null): string {
  // null だけを除外する安全な方法
  if (count != null) {
    return `Count: ${count}`; // 0 も正しく表示される
  }
  return "default";
}
```

## 型の絞り込み（narrowing）の仕組み

ここまで見てきた型ガードは、すべて TypeScript の**コントロールフロー解析**によって動作しています。コンパイラが条件分岐を追跡し、各ブロック内で変数の型を自動的に狭めます。

```typescript
function handle(x: string | number | null) {
  // x の型: string | number | null

  if (x === null) return;
  // x の型: string | number（null が除外された）

  if (typeof x === "string") {
    // x の型: string
    console.log(x.toUpperCase());
  } else {
    // x の型: number
    console.log(x.toFixed(2));
  }
}
```

早期リターン（early return）で分岐から抜けると、以降のコードではその型が除外されます。`return` だけでなく `throw` や `break` でも同様です。

## ユーザー定義型ガード

組み込みの型ガードでは対応できない複雑な条件には、**ユーザー定義型ガード**を使います。戻り値の型を `value is Type`（型述語）の形式で書きます。

```typescript
interface Fish { swim(): void; }
interface Bird { fly(): void; }

function isFish(pet: Fish | Bird): pet is Fish {
  return "swim" in pet;
}

function move(pet: Fish | Bird): void {
  if (isFish(pet)) {
    pet.swim(); // Fish として安全に使える
  } else {
    pet.fly();  // Bird として安全に使える
  }
}
```

配列の `filter` と組み合わせると、`null` を型安全に除去できます。

```typescript
const items: (string | null)[] = ["a", null, "b", null, "c"];

function isNotNull<T>(value: T | null): value is T {
  return value !== null;
}

const filtered: string[] = items.filter(isNotNull);
// ["a", "b", "c"]（型も string[] に絞り込まれる）
```

## まとめ

| 概念 | 要点 |
|------|------|
| ユニオン型 | `A \| B` で「A または B」を表す型 |
| メンバーアクセス | ユニオン型では全構成型に共通するメンバーのみ使える |
| リテラル型ユニオン | `"a" \| "b" \| "c"` で決まった値のいずれかを表す |
| typeof ガード | `typeof x === "string"` でプリミティブ型を判定 |
| instanceof ガード | `x instanceof Error` でクラスインスタンスを判定 |
| in ガード | `"key" in obj` でプロパティの有無を判定 |
| truthy チェック | `if (x)` で null / undefined / falsy を除外 |
| narrowing | コントロールフロー解析による型の自動絞り込み |
| ユーザー定義型ガード | `value is Type` で独自の判定関数を定義 |

## やってみよう！

### 練習1: typeof で型を絞り込む

`string | number | boolean` を受け取り、型に応じたメッセージを返す関数 `describeValue` を書いてください。

- `string` → `"文字列: ○○"`（大文字に変換）
- `number` → `"数値: ○○"`（小数点2桁に整形）
- `boolean` → `"真偽値: true"` または `"真偽値: false"`

```typescript
function describeValue(value: string | number | boolean): string {
  // ここに実装してください
}

// テスト
console.log(describeValue("hello"));  // "文字列: HELLO"
console.log(describeValue(3.14159)); // "数値: 3.14"
console.log(describeValue(true));    // "真偽値: true"
```

### 練習2: in ガードで分岐する

以下の型を使い、`getArea` 関数を実装してください。`in` 演算子で型を絞り込みます。

```typescript
interface Circle { radius: number; }
interface Rectangle { width: number; height: number; }

function getArea(shape: Circle | Rectangle): number {
  // ここに実装してください
  // ヒント: "radius" in shape で Circle かどうかを判定
}

// テスト
console.log(getArea({ radius: 5 }));            // 78.53981633974483
console.log(getArea({ width: 4, height: 6 }));  // 24
```

### 練習3: ユーザー定義型ガードを書く

`unknown` 型の値が `{ name: string; age: number }` を満たすかを判定する型ガード `isPerson` を実装してください。

```typescript
interface Person { name: string; age: number; }

function isPerson(value: unknown): value is Person {
  // ここに実装してください
  // ヒント: typeof でオブジェクトかを確認し、各プロパティの型もチェック
}

// テスト
console.log(isPerson({ name: "Alice", age: 30 })); // true
console.log(isPerson({ name: "Bob" }));             // false
console.log(isPerson("hello"));                     // false
console.log(isPerson(null));                         // false
```
