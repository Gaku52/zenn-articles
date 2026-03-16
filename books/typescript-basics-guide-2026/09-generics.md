---
title: "ジェネリクス入門"
---

# ジェネリクス入門

> 型をパラメータ化して、再利用可能かつ型安全なコードを書く

## ジェネリクスとは何か

ジェネリクスは、TypeScriptにおける「型の変数化」の仕組みです。関数や型を定義するとき、具体的な型を決めず**型パラメータ**として残しておき、使うときに実際の型が決まるようにします。

ジェネリクスがない世界では、次の2つの選択肢しかありません。

```typescript
// 選択肢1: 型ごとに関数を定義する（コードが重複する）
function identityString(value: string): string { return value; }
function identityNumber(value: number): number { return value; }

// 選択肢2: any を使う（型安全性が失われる）
function identityAny(value: any): any { return value; }
const result = identityAny("hello"); // result は any型 → 型情報がない
```

ジェネリクスを使えば、**再利用性と型安全性を両立**できます。

```typescript
function identity<T>(value: T): T {
  return value;
}

const str = identity("hello"); // str: string
const num = identity(42);      // num: number
```

`<T>` が型パラメータです。呼び出し時の引数から、TypeScriptが `T` の型を自動的に推論します。

## ジェネリック関数

### 型引数の明示と推論

型パラメータは明示的に指定できますが、多くの場合TypeScriptが自動推論してくれます。推論に任せ、うまくいかないときだけ明示するのがよいスタイルです。

```typescript
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

first<string>(["a", "b"]); // 明示的に指定 → string | undefined
first([1, 2, 3]);           // 推論される   → number | undefined
first(["a", "b"]);          //              → string | undefined
```

### 複数の型パラメータ

型パラメータは複数持てます。

```typescript
function pair<A, B>(first: A, second: B): [A, B] {
  return [first, second];
}

const p = pair("name", 42); // [string, number]
```

### アロー関数でのジェネリクス

```typescript
const toArray = <T>(value: T): T[] => [value];

toArray("hello"); // string[]
toArray(42);      // number[]
```

`.tsx`ファイルでは `<T>` がJSXタグと混同されることがあります。`<T,>` で回避できます。

## ジェネリック型

関数だけでなく、型エイリアスやインターフェース、クラスにも型パラメータを使えます。

### 型エイリアスとインターフェース

```typescript
// ジェネリック型エイリアス
type ApiResponse<T> = {
  status: number;
  data: T;
  timestamp: string;
};

type UserResponse = ApiResponse<{ id: string; name: string }>;

// ジェネリックインターフェース
interface PaginatedList<T> {
  items: T[];
  total: number;
  page: number;
  hasNext: boolean;
}

// 例で使用する User 型
type User = { id: string; name: string; email: string };

const userList: PaginatedList<User> = {
  items: [{ id: "1", name: "Alice", email: "alice@example.com" }],
  total: 100,
  page: 1,
  hasNext: true,
};
```

`ApiResponse<T>` は「どんなデータでも載せられるレスポンスの型」です。`T` を差し替えるだけで、さまざまなレスポンスの型を作れます。

### ジェネリッククラス

```typescript
class Stack<T> {
  private items: T[] = [];

  push(item: T): void { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); }
  peek(): T | undefined { return this.items[this.items.length - 1]; }
  get size(): number { return this.items.length; }
}

const nums = new Stack<number>();
nums.push(1);
nums.push(2);
nums.pop();  // number | undefined

const strs = new Stack<string>();
strs.push("hello");
strs.peek(); // string | undefined
```

## 制約（extends）

型パラメータに制約を付けると、「この型は少なくともこの構造を持つ」と保証できます。`extends` キーワードを使います。

### 基本的な制約

```typescript
function printLength<T extends { length: number }>(value: T): void {
  console.log(`Length: ${value.length}`);
}

printLength("hello");        // OK: string は length を持つ
printLength([1, 2, 3]);      // OK: 配列は length を持つ
printLength({ length: 10 }); // OK
// printLength(42);           // Error: number は length を持たない
```

制約がないと `value.length` にアクセスできません。`T extends { length: number }` と書くことで、`T` が `length` プロパティを持つことをTypeScriptに伝えています。

### keyof を使った制約

`keyof` と組み合わせると、オブジェクトのキーに安全にアクセスする関数を作れます。

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "Alice", age: 30 };

getProperty(user, "name"); // string
getProperty(user, "age");  // number
// getProperty(user, "foo"); // Error: "foo" は user のキーにない
```

`K extends keyof T` は「`K` は `T` のキーのいずれか」という制約です。存在しないキー名を渡すとコンパイルエラーになります。

## デフォルト型パラメータ

関数のデフォルト引数と同じように、型パラメータにもデフォルト値を設定できます。

```typescript
interface FetchOptions<T = unknown> {
  url: string;
  method?: "GET" | "POST";
  body?: T;
}

// 型引数なしで使える（T は unknown になる）
const getOpts: FetchOptions = { url: "/api/users" };

// 型引数を指定すると body の型が決まる
const postOpts: FetchOptions<{ name: string }> = {
  url: "/api/users",
  method: "POST",
  body: { name: "Alice" },
};
```

デフォルト型パラメータは末尾に配置する必要があります。

```typescript
type Result<T, E = Error> = { value: T } | { error: E }; // OK

// type Bad<T = string, U> = { a: T; b: U }; // Error: デフォルトの後に必須は置けない
```

## 実践的な使用例

### APIレスポンスの型定義

成功と失敗を型で表現するパターンは、実務でよく使われます。

```typescript
type ApiResult<T> =
  | { ok: true; data: T }
  | { ok: false; error: { code: number; message: string } };

function handleResult(result: ApiResult<User>) {
  if (result.ok) {
    console.log(result.data.name); // data が User として絞り込まれる
  } else {
    console.log(result.error.message); // error として絞り込まれる
  }
}
```

### 非同期データの状態管理

Reactなどのフレームワークでよく使われるデータ取得状態の型です。

```typescript
type AsyncState<T> = {
  data: T | null;
  loading: boolean;
  error: Error | null;
};

type UserListState = AsyncState<User[]>;     // ユーザー一覧の状態
type UserDetailState = AsyncState<User>;     // 単一ユーザーの状態
```

### 型安全なイベントリスナー

`keyof` 制約とジェネリクスを組み合わせると、イベント名とペイロードの型を関連付けられます。

```typescript
type EventMap = {
  click: { x: number; y: number };
  error: { message: string };
};

function on<K extends keyof EventMap>(
  event: K, handler: (payload: EventMap[K]) => void
): void { /* 登録処理 */ }

on("click", (payload) => {
  console.log(payload.x, payload.y); // { x: number; y: number }
});
// on("unknown", () => {}); // Error: "unknown" は EventMap のキーにない
```

## ジェネリクスが必要なとき・不要なとき

ジェネリクスは強力ですが、すべてに使う必要はありません。

```typescript
// NG: T を戻り値に使っていない → ジェネリクス不要
function badLength<T extends { length: number }>(value: T): number {
  return value.length;
}

// OK: ジェネリクスなしで十分
function goodLength(value: { length: number }): number {
  return value.length;
}

// OK: 入力と出力の型が連動 → ジェネリクスが有効
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}
```

**使うべきとき:** 入力の型と出力の型を関連付けたい、複数の引数の型を関連付けたい、同じロジックを型安全に複数の型で再利用したいとき。

**不要なとき:** 型パラメータを戻り値で使わない場合や、具体的な型で十分な場合。

## まとめ

| 概念 | 要点 |
|------|------|
| ジェネリクス | 型をパラメータ化して、再利用性と型安全性を両立する仕組み |
| 型パラメータ `<T>` | 関数・型・クラスに付ける「型の変数」 |
| 型推論 | 呼び出し時の引数から型パラメータを自動的に決定する |
| 複数の型パラメータ | `<A, B>` のように複数の型を同時にパラメータ化できる |
| 制約 `extends` | 型パラメータが満たすべき条件を指定する |
| `keyof` 制約 | オブジェクトのキーに型パラメータを制限する |
| デフォルト型 | `<T = DefaultType>` で型引数省略時の既定値を設定する |
| ジェネリック型 | 型エイリアス・インターフェース・クラスを汎用化する |
| 使い分けの基準 | 入力と出力の型が連動するときにジェネリクスを使う |

## やってみよう！

### 練習1: ジェネリック関数を作る

配列から条件に合う最初の要素を返す `findFirst` 関数を作ってみましょう。

```typescript
function findFirst<T>(arr: T[], predicate: (item: T) => boolean): T | undefined {
  // ここを実装してください
}

// 動作確認
const nums = [1, 5, 12, 8, 3];
const found = findFirst(nums, (n) => n > 10);
console.log(found); // 12

const users = [{ name: "Alice", age: 25 }, { name: "Bob", age: 35 }];
const senior = findFirst(users, (u) => u.age >= 30);
console.log(senior); // { name: "Bob", age: 35 }
```

### 練習2: ジェネリック型エイリアスを作る

成功・失敗を表すAPIレスポンス型を定義してみましょう。以下のコードが型エラーなく動くように `ApiResponse<T>` を定義してください。

```typescript
type ApiResponse<T> = // ???

const success: ApiResponse<{ id: string }> = { status: "success", data: { id: "1" } };
const failure: ApiResponse<{ id: string }> = { status: "error", message: "Not found" };
```

ヒント: Chapter 07で学んだユニオン型を使って、成功パターンと失敗パターンを定義します。

### 練習3: 制約付きジェネリック関数を作る

`id` プロパティを持つオブジェクトの配列から、指定IDのオブジェクトを探す `findById` 関数を作ってみましょう。

```typescript
function findById<T extends { id: string }>(items: T[], id: string): T | undefined {
  // ここを実装してください
}

// 動作確認
const products = [
  { id: "p1", name: "TypeScript本", price: 3000 },
  { id: "p2", name: "JavaScript本", price: 2500 },
];

const product = findById(products, "p1"); // Product型として推論される
console.log(product); // { id: "p1", name: "TypeScript本", price: 3000 }
```

`T extends { id: string }` の制約により、`id` プロパティを持たないオブジェクトの配列を渡すとコンパイルエラーになります。この制約の仕組みを確認してみてください。
