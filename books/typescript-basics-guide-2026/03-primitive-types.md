---
title: "基本の型"
---

# Chapter 03: 基本の型

> 型アノテーション、型推論、プリミティブ型 — TypeScriptの型システムの土台

## この章で学べること

- string, number, boolean の3大プリミティブ型
- null と undefined の違いと使い分け
- リテラル型で値を限定するテクニック
- 型推論の仕組みと型アノテーションの使いどころ

---

## 型アノテーションと型推論

TypeScriptでは、変数名の後に `: 型` を書いて型を明示できます。これが**型アノテーション**です。

```typescript
const message: string = "こんにちは";
const count: number = 42;
const isReady: boolean = true;
```

ただし、初期値がある場合はTypeScriptが自動で型を推論するため、省略できます。

```typescript
const message = "こんにちは";  // string と推論
const count = 42;              // number と推論
const isReady = true;          // boolean と推論
```

**型アノテーションが必要な場面**は限られています。

```typescript
// 初期値がない変数 → 型アノテーションが必要
let userName: string;
userName = "Alice";

// 関数の引数 → 型アノテーションが必要
function greet(name: string): string {
  return `Hello, ${name}!`;
}

// 初期値がある変数 → 省略可能
const age = 30; // : number は不要
```

推論できる場所では省略し、推論できない場所で明示するのが一般的なスタイルです。

---

## string 型

文字列を表す型です。常に小文字の `string` を使います（大文字の `String` はラッパーオブジェクト型で、非推奨です）。

```typescript
const name = "Alice";
const greeting = 'Hello';
const message = `Welcome, ${name}!`;  // テンプレートリテラル

// 文字列メソッドの戻り値も型推論される
const upper = "hello".toUpperCase();    // string
const found = "hello".includes("ell");  // boolean
const parts = "a,b,c".split(",");       // string[]
```

## number 型

整数も小数も、すべて `number` 型です。

```typescript
const integer = 42;
const float = 3.14;
const hex = 0xff;      // 16進数（255）
const binary = 0b1010; // 2進数（10）
```

`Infinity` や `NaN` も `number` 型に含まれる点に注意してください。NaN のチェックには `Number.isNaN()` を使います。

`number` の安全な範囲（約 9000 兆）を超える場合は `bigint` を使いますが、入門段階では `number` だけ覚えておけば十分です。

## boolean 型

`true` か `false` の2値のみを取る型です。

```typescript
const isActive = true;   // boolean と推論
const isAdult = 20 >= 18; // boolean と推論（比較演算の結果）
```

JavaScriptの falsy 値（`0`, `""`, `null` など）は TypeScript では `boolean` 型ではありません。明示的に `Boolean(value)` や `!!value` で変換します。

---

## null と undefined

TypeScriptでは `null` と `undefined` を区別して扱います。

| 値 | 意味 | 使用場面 |
|---|---|---|
| `null` | 「値が存在しない」ことを意図的に示す | APIレスポンス、DB検索結果 |
| `undefined` | 「値がまだ設定されていない」 | オプショナルな引数やプロパティ |

```typescript
// APIレスポンスでは null が多い（JSONに undefined は存在しないため）
interface ApiResponse {
  user: User | null;
}

// オプショナルプロパティは undefined を含む
interface Config {
  timeout?: number;   // number | undefined と同義
}
```

### strictNullChecks の重要性

`tsconfig.json` で `strict: true` を有効にすると、`null` / `undefined` を他の型に代入できなくなります。

```typescript
let name: string = "Alice";
// name = null;      // Error
// name = undefined; // Error

// null を許容するなら Union 型で明示
let nickname: string | null = "Bob";
nickname = null; // OK
```

この設定は必ず有効にしてください。null 起因のバグを型レベルで防げます。

---

## リテラル型

特定の値そのものを型として扱えます。これが**リテラル型**です。

```typescript
type Direction = "north" | "south" | "east" | "west";
let dir: Direction = "north"; // OK
// dir = "up";                // Error

type DiceRoll = 1 | 2 | 3 | 4 | 5 | 6;
let roll: DiceRoll = 3;  // OK
// roll = 7;              // Error
```

リテラル型を使うと、タイポが即座にコンパイルエラーになります。

```typescript
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";

function request(method: HttpMethod, url: string): void {
  // ...
}

request("GET", "/api/users");     // OK
// request("GETT", "/api/users"); // Error: タイポを即座に検出
```

---

## 型推論の仕組み

### const と let の推論の違い

```typescript
// const → リテラル型（値が変わらないため）
const x = "hello";  // 型: "hello"
const n = 42;       // 型: 42

// let → ワイドな型（再代入の可能性があるため）
let y = "hello";    // 型: string
let m = 42;         // 型: number
```

### オブジェクトの型推論と as const

`const` で宣言してもオブジェクトのプロパティはワイドな型に推論されます。`as const` を付けるとリテラル型 + `readonly` になります。

```typescript
// as const なし
const config = { host: "localhost", port: 3000 };
// 型: { host: string; port: number }

// as const あり
const config2 = { host: "localhost", port: 3000 } as const;
// 型: { readonly host: "localhost"; readonly port: 3000 }
```

`as const` から Union 型を生成するテクニックは実務でよく使います。

```typescript
const STATUS = {
  Active: "active",
  Inactive: "inactive",
  Pending: "pending",
} as const;

type Status = (typeof STATUS)[keyof typeof STATUS];
// "active" | "inactive" | "pending"
```

### 文脈的型推論

TypeScriptは「この場所で期待される型」からも推論します。

```typescript
const numbers = [1, 2, 3];
const doubled = numbers.map(n => n * 2);
// n は number と推論（numbers が number[] なので）

document.addEventListener("click", event => {
  console.log(event.clientX);
  // event は MouseEvent と推論（"click" の定義から）
});
```

### 型アノテーションを書く/書かないの判断基準

| 場面 | 推奨 |
|---|---|
| 初期値のある変数 | 推論に任せる |
| 関数の引数 | 型アノテーションを書く |
| 関数の戻り値 | 推論に任せる（公開APIでは明示も可） |
| 初期値のない変数 | 型アノテーションを書く |

推論で十分な場所にまで型を書くと冗長になり、リファクタリング時の変更箇所も増えてしまいます。

---

## まとめ

| 概念 | 要点 |
|---|---|
| string | 文字列型。常に小文字の `string` を使う |
| number | 数値型。整数と小数の区別なし |
| boolean | `true` / `false` の2値型 |
| null | 「値が存在しない」ことを意図的に示す |
| undefined | 「値が未設定」であることを示す |
| リテラル型 | 特定の値そのものを型にする（`"GET"`, `42`, `true`） |
| 型推論 | 初期値や文脈から型を自動判定する仕組み |
| 型アノテーション | `: 型名` の形式で型を明示する記法 |
| as const | オブジェクトや配列をリテラル型 + readonly にする |

---

## やってみよう！

### 練習1: 型推論を確認する

以下のコードで、各変数がどの型に推論されるか予想してから、エディタで確認してみてください。

```typescript
const appName = "MyApp";
let version = 1;
const isProduction = false;
const settings = { debug: true, port: 8080 };
const modes = ["development", "production"] as const;
```

### 練習2: リテラル型でログレベルを定義する

ログレベルを表す型 `LogLevel` を定義し、それを引数に取る `log` 関数を作ってみてください。

```typescript
// LogLevel型を定義（"debug" | "info" | "warn" | "error"）
// log関数を定義（level: LogLevel, message: string）
// console.log でレベルとメッセージを出力

// 想定出力例:
// log("info", "サーバー起動");   → [INFO] サーバー起動
// log("error", "接続失敗");     → [ERROR] 接続失敗
// log("trace", "詳細ログ");     → コンパイルエラー
```

### 練習3: as const で定数オブジェクトを作る

`as const` を使った定数オブジェクトを定義し、値の Union 型を取り出してみてください。

```typescript
// 要件:
// - オブジェクト名: PRIORITY
// - キー: Low, Medium, High / 値: "low", "medium", "high"
// - 値のUnion型 Priority を typeof と keyof で生成する
```
