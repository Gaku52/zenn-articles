---
title: "関数の型付け"
---

# 関数の型付け

> 関数の引数と戻り値に型を付けて、呼び出し時のミスをコンパイル時に検出する

## この章で学ぶこと

- 引数と戻り値に型注釈を付ける方法
- オプション引数とデフォルト引数の違い
- レストパラメータの型付け
- コールバック関数の型定義
- `void` 型の意味と使いどころ
- 関数オーバーロードの書き方

## 引数と戻り値の型

TypeScriptでは、関数の引数と戻り値に型注釈を付けることができます。

```typescript
// 関数宣言
function add(a: number, b: number): number {
  return a + b;
}

// アロー関数
const multiply = (a: number, b: number): number => a * b;

add(1, 2);       // OK: 3
// add("1", 2);  // Error: string は number に代入できない
```

引数に型がないJavaScriptでは、`add("1", 2)` が `"12"` を返すようなバグが起きえます。TypeScriptでは、こうしたミスをコンパイル時に検出できます。

### 戻り値の型推論

戻り値の型は省略しても、TypeScriptが `return` 文から推論してくれます。

```typescript
// 戻り値型は number と推論される
function add(a: number, b: number) {
  return a + b;
}

// 戻り値型は string と推論される
const greet = (name: string) => `Hello, ${name}!`;
```

推論に任せても多くの場合は問題ありません。ただし、公開APIや複雑な関数では、戻り値型を明示した方が意図が伝わりやすくなります。

### 関数型の変数

関数そのものの型を変数に付けることもできます。

```typescript
// 変数に関数型を注釈
const divide: (a: number, b: number) => number = (a, b) => a / b;

// 型エイリアスで関数型を定義すると再利用しやすい
type MathOp = (a: number, b: number) => number;
const subtract: MathOp = (a, b) => a - b;
```

## オプション引数

引数名の後に `?` を付けると、その引数は省略可能になります。省略された場合の値は `undefined` です。

```typescript
function greet(name: string, greeting?: string): string {
  return `${greeting ?? "Hello"}, ${name}!`;
}

greet("Alice");         // "Hello, Alice!"
greet("Alice", "Hi");   // "Hi, Alice!"
```

オプション引数の型は自動的に `string | undefined` のユニオン型になります。そのため、関数本体では `undefined` を考慮した処理が必要です。

**注意点**: オプション引数は、必須引数よりも後ろに配置する必要があります。

```typescript
// Error: 必須引数をオプション引数の後に配置できない
// function bad(x?: number, y: number) {}

// OK: オプション引数は後ろに
function good(y: number, x?: number) {}
```

## デフォルト引数

デフォルト値を設定すると、引数が省略された場合にその値が使われます。

```typescript
function createUser(name: string, role: string = "viewer") {
  return { name, role };
}

createUser("Bob");            // { name: "Bob", role: "viewer" }
createUser("Bob", "admin");   // { name: "Bob", role: "admin" }
```

### オプション引数との違い

オプション引数とデフォルト引数は似ていますが、型の扱いが異なります。

```typescript
function withOptional(x: number, y?: number): number {
  // y の型は number | undefined
  return x + (y ?? 0);
}

function withDefault(x: number, y: number = 0): number {
  // y の型は number（undefined が来てもデフォルト値が使われる）
  return x + y;
}

withDefault(1, undefined); // 1（y は 0 として扱われる）
```

デフォルト引数の場合、`undefined` を明示的に渡してもデフォルト値が適用されます。

## レストパラメータ

`...` を使うと、可変長の引数をまとめて配列として受け取れます。

```typescript
function sum(...numbers: number[]): number {
  return numbers.reduce((total, n) => total + n, 0);
}

sum(1, 2, 3);       // 6
sum(1, 2, 3, 4, 5); // 15
```

通常の引数と組み合わせることもできます。

```typescript
function log(prefix: string, ...messages: string[]): void {
  console.log(`[${prefix}]`, ...messages);
}

log("APP", "Server", "started"); // [APP] Server started
```

### スプレッド引数と `as const`

配列をスプレッドして関数に渡す場合、`as const` を付けないとエラーになることがあります。

```typescript
function add3(a: number, b: number, c: number): number {
  return a + b + c;
}

// const args = [1, 2, 3];     // 型は number[]（長さが不定）
// add3(...args);               // Error: number[] を3引数に展開できない

const args = [1, 2, 3] as const; // 型は readonly [1, 2, 3]
add3(...args);                    // OK
```

`as const` によって配列がタプル型（固定長）として扱われ、引数の数が一致することをコンパイラが確認できるようになります。

## コールバック関数の型

コールバック関数にも型を付けられます。型エイリアスで定義しておくと見通しが良くなります。

```typescript
type Callback = (error: Error | null, data: string) => void;

function readFile(path: string, callback: Callback): void {
  try {
    const content = "file content";
    callback(null, content);
  } catch (err) {
    callback(err instanceof Error ? err : new Error(String(err)), "");
  }
}

readFile("/tmp/test.txt", (err, data) => {
  if (err) {
    console.error(err.message);
    return;
  }
  console.log(data);
});
```

配列メソッドの `filter` や `map` でよく使う「関数を引数に取る関数」も同じ要領で型を付けられます。

```typescript
type Predicate<T> = (value: T) => boolean;

function filter<T>(arr: T[], predicate: Predicate<T>): T[] {
  return arr.filter(predicate);
}

const evens = filter([1, 2, 3, 4, 5], (n) => n % 2 === 0);
// [2, 4]
```

関数が関数を返すパターンも型を明示できます。

```typescript
function createGreeter(greeting: string): (name: string) => string {
  return (name) => `${greeting}, ${name}!`;
}

const hello = createGreeter("Hello");
hello("Alice"); // "Hello, Alice!"
```

## void 型

`void` は「戻り値がない」ことを表す型です。`return` 文を書かない関数や、`return;` だけの関数に使います。

```typescript
function logMessage(message: string): void {
  console.log(message);
}
```

### void と undefined の違い

`void` と `undefined` は似ていますが、コールバックの型で重要な違いがあります。

```typescript
// void: 「戻り値を使わない」という意図
type VoidCallback = () => void;

// void型のコールバックは、値を返す関数も代入できる
const cb: VoidCallback = () => 42; // OK（戻り値は無視される）

// undefined: 文字通り undefined を返す必要がある
type UndefinedCallback = () => undefined;
// const cb2: UndefinedCallback = () => 42; // Error
```

`void` は「戻り値を使わない」という契約です。`Array.prototype.forEach` のコールバックのように、戻り値が無視される場面で広く使われています。

## 関数オーバーロード

関数オーバーロードを使うと、引数の型に応じて異なる戻り値の型を返す関数を定義できます。

```typescript
// オーバーロードシグネチャ（呼び出し側に見える型）
function createElement(tag: "div"): HTMLDivElement;
function createElement(tag: "span"): HTMLSpanElement;
function createElement(tag: "input"): HTMLInputElement;
// 実装シグネチャ（実際の処理）
function createElement(tag: string): HTMLElement {
  return document.createElement(tag);
}

const div = createElement("div");     // 型: HTMLDivElement
const span = createElement("span");   // 型: HTMLSpanElement
```

オーバーロードシグネチャは上から順にチェックされます。実装シグネチャは外部から直接呼び出すことはできません。

### 引数の数によるオーバーロード

```typescript
function padding(all: number): string;
function padding(vertical: number, horizontal: number): string;
function padding(a: number, b?: number): string {
  if (b === undefined) {
    return `${a}px`;
  }
  return `${a}px ${b}px`;
}

padding(10);      // "10px"
padding(10, 20);  // "10px 20px"
```

### オーバーロード vs ユニオン型 vs ジェネリクス

オーバーロードは強力ですが、常に必要なわけではありません。

```typescript
// 戻り値が同じならユニオン型で十分
function len(x: string | unknown[]): number {
  return x.length;
}

// 入力型と出力型が対応するならジェネリクスで解決できることも
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}
```

オーバーロードは「入力の型によって戻り値の型が変わる」場合に使うのが適切です。それ以外のケースでは、ユニオン型やジェネリクスの方がシンプルになります。

## まとめ

| 概念 | 要点 |
|------|------|
| 引数・戻り値の型 | 関数シグネチャに型注釈を付けて呼び出し時のミスを防ぐ |
| 戻り値の型推論 | 省略しても推論されるが、公開APIでは明示が望ましい |
| オプション引数（`?`） | 省略可能。型は `T \| undefined` になる |
| デフォルト引数 | 省略時にデフォルト値が使われる。`undefined` でも適用 |
| レストパラメータ | `...args: T[]` で可変長引数を配列として受け取る |
| コールバック関数の型 | 型エイリアスで定義すると再利用しやすい |
| `void` 型 | 「戻り値を使わない」意図を表す。`undefined` とは異なる |
| 関数オーバーロード | 入力型に応じて戻り値型を変えたいときに使う |

## やってみよう！

### 練習1: 挨拶関数を作る

以下の仕様を満たす `greet` 関数を書いてみましょう。

- 第1引数 `name`（`string`、必須）
- 第2引数 `greeting`（`string`、デフォルト値 `"Hello"`）
- 戻り値は `"Hello, Alice!"` のような文字列

```typescript
// ここに greet 関数を書いてみましょう

console.log(greet("Alice"));          // "Hello, Alice!"
console.log(greet("Bob", "Hey"));     // "Hey, Bob!"
```

### 練習2: 合計値を計算する

任意の数の数値を受け取り、合計を返す `total` 関数を書いてみましょう。レストパラメータを使います。

```typescript
// ここに total 関数を書いてみましょう

console.log(total(1, 2, 3));       // 6
console.log(total(10, 20));        // 30
console.log(total());              // 0
```

### 練習3: コールバック付き処理

以下の `processItems` 関数の型を完成させてみましょう。配列の各要素に対してコールバックを実行し、結果の配列を返します。

```typescript
type Transform = /* ここに型を定義 */;

function processItems(items: string[], transform: Transform): string[] {
  return items.map(transform);
}

const upper = processItems(["hello", "world"], (s) => s.toUpperCase());
console.log(upper); // ["HELLO", "WORLD"]
```
