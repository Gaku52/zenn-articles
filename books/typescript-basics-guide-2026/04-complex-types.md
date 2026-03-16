---
title: "配列・タプル・オブジェクト"
---

# Chapter 04: 配列・タプル・オブジェクト

> 複数の値をまとめる型 — 配列、タプル、オブジェクト、そして特殊な型たち

## この章で学べること

- 配列の型付けと readonly 配列
- タプル型で「位置ごとに型が異なる配列」を表現する方法
- オブジェクト型リテラル、readonly / オプショナルプロパティ
- any, unknown, never の使い分け

---

## 配列の型

配列の型を書く方法は2つあります。どちらも同じ意味です。

```typescript
const numbers: number[] = [1, 2, 3];          // 記法1（主流）
const strings: Array<string> = ["a", "b"];     // 記法2（ジェネリック記法）
```

初期値があれば型アノテーションは省略できます。

```typescript
const fruits = ["apple", "banana"];  // string[] と推論
const mixed = [1, "hello", true];   // (string | number | boolean)[]
```

### 配列メソッドの型推論

```typescript
const numbers = [1, 2, 3, 4, 5];

const doubled = numbers.map(n => n * 2);        // number[]
const found = numbers.find(n => n === 3);       // number | undefined
const total = numbers.reduce((sum, n) => sum + n, 0); // number
```

`find` の戻り値が `number | undefined` になる点に注目してください。見つからない可能性を型に反映しています。

複数の型を含む配列は `()` で囲みます。

```typescript
const values: (string | number)[] = [1, "hello", 2, "world"];
```

### readonly 配列

`readonly` を付けると変更メソッド（`push`, `sort` など）が使えなくなります。

```typescript
const colors: readonly string[] = ["red", "green", "blue"];
// colors.push("yellow"); // Error: readonly 配列に push は存在しない

// 変更が必要ならスプレッド構文でコピーしてから操作
function getSorted(items: readonly string[]): string[] {
  return [...items].sort();
}
```

---

## タプル型

タプルは「要素数と各位置の型が決まった配列」です。

```typescript
type NameAge = [string, number];

const alice: NameAge = ["Alice", 30]; // OK
// const bad: NameAge = [30, "Alice"]; // Error: 順序が違う
// const short: NameAge = ["Alice"];   // Error: 要素が足りない
```

### 分割代入と関数の戻り値

タプルと分割代入を組み合わせると、各変数に正しい型が付きます。React の `useState` もこの形式です。

```typescript
function divide(a: number, b: number): [number, number] {
  return [Math.floor(a / b), a % b];
}

const [quotient, remainder] = divide(17, 5); // [3, 2]
// quotient: number, remainder: number
```

### ラベル付きタプルとオプショナル要素

```typescript
// ラベルで可読性を向上
type UserEntry = [id: number, name: string, active: boolean];

// オプショナル要素
type Point = [number, number, number?];
const point2d: Point = [1, 2];     // OK
const point3d: Point = [1, 2, 3];  // OK
```

### 配列 vs タプルの比較

| 特性 | 配列 (`number[]`) | タプル (`[string, number]`) |
|---|---|---|
| 要素数 | 可変 | 固定 |
| 要素の型 | 全要素が同じ型 | 位置ごとに異なる型が可能 |
| 用途 | 同種データのコレクション | 異種データの組み合わせ |

---

## オブジェクト型リテラル

オブジェクトの構造を型として表現できます。

```typescript
const user: { name: string; age: number } = {
  name: "Alice",
  age: 30,
};
```

通常は `type` で名前を付けます（型エイリアスの詳細は Chapter 06 で解説します）。

```typescript
type User = {
  name: string;
  age: number;
  email: string;
};

const alice: User = { name: "Alice", age: 30, email: "alice@example.com" };
```

### オプショナルプロパティと readonly プロパティ

`?` を付けると省略可能に、`readonly` を付けると再代入禁止になります。

```typescript
type Config = {
  readonly host: string;    // 再代入禁止
  readonly port: number;    // 再代入禁止
  ssl?: boolean;            // 省略可能（boolean | undefined）
  timeout?: number;         // 省略可能（number | undefined）
};

const config: Config = { host: "localhost", port: 3000 }; // OK
// config.host = "other";  // Error: readonly
config.ssl;                // boolean | undefined

function getTimeout(c: Config): number {
  return c.timeout ?? 3000; // undefined ならデフォルト値
}
```

`Readonly<T>` で全プロパティを一括 readonly にすることもできます。なお `readonly` はコンパイル時のチェックのみです。

---

## any 型

`any` は型チェックを完全に無効化する特殊な型です。

```typescript
let value: any = "hello";
value.foo.bar.baz(); // コンパイルエラーなし → 実行時エラー
```

TypeScriptの恩恵がなくなるため、基本的に使わないでください。`tsconfig.json` で `noImplicitAny: true` を設定して暗黙の `any` を禁止するのが推奨です。

## unknown 型

`unknown` は `any` と同じくあらゆる値を代入できますが、**使う前に型チェックが必要**です。

```typescript
let value: unknown = "hello";
// value.toUpperCase(); // Error: unknown 型には操作できない

if (typeof value === "string") {
  console.log(value.toUpperCase()); // OK: string と確認済み
}
```

外部からのデータを受け取る場面では `any` ではなく `unknown` を使いましょう。

```typescript
// catch ブロック
try {
  await fetchData();
} catch (error: unknown) {
  if (error instanceof Error) {
    console.error(error.message);
  }
}
```

### any vs unknown

| 特性 | `any` | `unknown` |
|---|---|---|
| 代入 | 何でも可 | 何でも可 |
| 操作 | 何でも可（危険） | 型チェック後のみ（安全） |
| 推奨度 | 非推奨 | 型がわからない場合に推奨 |

---

## never 型

`never` は「値が存在しない」ことを表す型です。

```typescript
// 必ず例外をスローする関数
function throwError(message: string): never {
  throw new Error(message);
}
```

### 網羅性チェック（exhaustive check）

`never` の最も実用的な使い方は、switch 文での網羅性チェックです。

```typescript
type Shape = "circle" | "square" | "triangle";

function getArea(shape: Shape): number {
  switch (shape) {
    case "circle":   return Math.PI * 10 * 10;
    case "square":   return 10 * 10;
    case "triangle": return (10 * 10) / 2;
    default:
      const _exhaustive: never = shape;
      return _exhaustive;
  }
}
```

`Shape` に新しい値（例: `"rectangle"`）を追加すると、`default` で**コンパイルエラー**になり、ケースの追加忘れを防げます。

```typescript
// ヘルパー関数として切り出すとさらに便利
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${value}`);
}
```

### any / unknown / never の関係

| 型 | 代入できる値 | 用途 |
|---|---|---|
| `any` | すべて | レガシーコードの一時的な回避（非推奨） |
| `unknown` | すべて | 外部データの型安全な受け取り |
| `never` | なし | 到達不能コード、網羅性チェック |

---

## void 型

`void` は「戻り値がない」ことを示す型です。`never`（正常終了しない）とは異なり、関数は正常に終了します。

```typescript
function log(message: string): void {
  console.log(message);
}
```

---

## まとめ

| 概念 | 要点 |
|---|---|
| 配列 (`T[]`) | 同じ型の要素を可変個数で持つ |
| readonly 配列 | `readonly T[]` で変更メソッドを禁止 |
| タプル (`[T, U]`) | 要素数と位置ごとの型が固定された配列 |
| オブジェクト型 | `{ key: Type }` でプロパティの構造を定義 |
| オプショナル (`?`) | プロパティを省略可能にする |
| readonly | プロパティの再代入を禁止（コンパイル時のみ） |
| `any` | 型チェック無効化。非推奨 |
| `unknown` | 型安全な any。使う前に型チェックが必要 |
| `never` | 値を持たない型。網羅性チェックに活用 |
| `void` | 戻り値なしを示す型 |

---

## やってみよう！

### 練習1: 配列とタプルの使い分け

以下の要件を型で表現してみてください。

```typescript
// 1. 商品名の一覧（文字列の配列）
// 2. 座標（x, y の2要素タプル）
// 3. ユーザー情報（id: number, name: string, isAdmin: boolean のタプル）
// 型を定義したら、それぞれに値を代入してみましょう
```

### 練習2: readonly とオプショナルを組み合わせる

以下の `AppConfig` 型を定義してみてください。

```typescript
// - appName: string（readonly、必須）
// - version: string（readonly、必須）
// - debug: boolean（省略可能）
// - apiUrl: string（省略可能）

// 想定出力例:
// const config: AppConfig = { appName: "MyApp", version: "1.0.0" }; → OK
// config.appName = "Other"; → コンパイルエラー
```

### 練習3: never で網羅性チェック

`getColorCode` 関数を完成させてください。`Color` に値を追加したとき、コンパイルエラーで漏れを検出できるようにします。

```typescript
type Color = "red" | "green" | "blue";

function getColorCode(color: Color): string {
  switch (color) {
    case "red":   return "#ff0000";
    case "green": return "#00ff00";
    case "blue":  return "#0000ff";
    // default ケースに never による網羅性チェックを追加
  }
}

// 想定出力例:
// getColorCode("red") → "#ff0000"
// Color に "yellow" を追加 → default でコンパイルエラー
```
