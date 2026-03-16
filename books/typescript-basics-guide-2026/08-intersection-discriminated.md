---
title: "交差型と判別可能なユニオン"
---

# 交差型と判別可能なユニオン

> 型を「かつ」で合成し、「タグ」で安全に分岐する

## 交差型（Intersection Types）とは

交差型は、複数の型を `&` 演算子で組み合わせ、**すべてのプロパティを持つ型**を作る仕組みです。前章のユニオン型が「A または B」だったのに対し、交差型は「A かつ B」を意味します。

```typescript
type HasId = { id: number };
type HasName = { name: string };
type HasEmail = { email: string };

type User = HasId & HasName & HasEmail;
// { id: number; name: string; email: string } と同じ

const user: User = {
  id: 1,
  name: "Alice",
  email: "alice@example.com",
};
```

## 交差型の活用: 型の合成

交差型の最大の利点は、小さな型を組み合わせて大きな型を作れることです。関心事ごとに型を分離し、必要に応じて合成するパターンは実務で頻繁に使われます。

```typescript
type Timestamped = {
  createdAt: Date;
  updatedAt: Date;
};

type SoftDeletable = {
  deletedAt: Date | null;
};

type BaseEntity = HasId & Timestamped;
type DeletableEntity = BaseEntity & SoftDeletable;
```

API レスポンスの型にも便利です。

```typescript
type WithPagination = { page: number; pageSize: number; totalItems: number };
type UserListResponse = { items: User[] } & WithPagination;
```

`interface` の `extends` でも同様の拡張ができますが、交差型はその場で柔軟に型を組み合わせたいときに向いています。

## ユニオン型と交差型の比較

| 特性 | ユニオン (`A \| B`) | 交差型 (`A & B`) |
|------|---------------------|------------------|
| 意味 | A **または** B | A **かつ** B |
| アクセスできるプロパティ | 共通のもののみ | すべて |
| 値の制約 | どちらかを満たせばOK | すべてを満たす必要あり |
| 主な用途 | 複数の可能性を表す | 型の合成・拡張 |

## 交差型で起きる型の矛盾

プリミティブ型同士の交差型は `never` になります。「文字列であり、かつ数値である」値は存在しないためです。

```typescript
type Impossible = string & number; // never
```

オブジェクト型で同名プロパティの型が矛盾する場合も、そのプロパティが `never` になります。

```typescript
type A = { value: string };
type B = { value: number };
type C = A & B;
// C は { value: never } → C 型の値を作ることは実質不可能

// 矛盾を避けるには Omit で除外してから合成する
type SafeMerge = Omit<A, "value"> & B;
// { value: number }
```

## 判別可能なユニオン（Discriminated Unions）

判別可能なユニオンは、ユニオン型の中で最も実践的なパターンです。各メンバーに共通の**リテラル型プロパティ（判別子）** を持たせることで、`switch` 文で安全に型を絞り込めます。

```typescript
interface Circle {
  kind: "circle";
  radius: number;
}

interface Rectangle {
  kind: "rectangle";
  width: number;
  height: number;
}

interface Triangle {
  kind: "triangle";
  base: number;
  height: number;
}

type Shape = Circle | Rectangle | Triangle;

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "rectangle":
      return shape.width * shape.height;
    case "triangle":
      return (shape.base * shape.height) / 2;
  }
}
```

`kind` が判別子です。各 `case` ブロック内で型が自動的に絞り込まれ、そのメンバー固有のプロパティに安全にアクセスできます。

## 判別子の設計ルール

判別可能なユニオンを正しく機能させるためのルールは3つです。

1. **判別子はリテラル型であること** -- `kind: "circle"` は OK、`kind: string` は NG
2. **プロパティ名はユニオン全体で統一すること** -- すべてのメンバーで同じ名前を使う
3. **判別子の値はユニオン内で一意であること** -- 同じ値を持つメンバーが複数あると判別できない

## 実践例: API レスポンス

```typescript
type ApiResponse<T> =
  | { status: "success"; data: T }
  | { status: "error"; error: { code: string; message: string } }
  | { status: "loading" };

function handleResponse(response: ApiResponse<User[]>): void {
  switch (response.status) {
    case "success":
      console.log(`${response.data.length} 件取得`);
      break;
    case "error":
      console.error(`${response.error.code}: ${response.error.message}`);
      break;
    case "loading":
      console.log("読み込み中...");
      break;
  }
}
```

boolean の判別子を使うパターンもあります。

```typescript
type ValidationResult =
  | { valid: true }
  | { valid: false; errors: string[] };

const result = validateEmail("test");
if (!result.valid) {
  result.errors.forEach((err) => console.log(err)); // 型安全
}
```

## 網羅性チェック（Exhaustiveness Checking）

`switch` 文ですべてのケースを処理したことをコンパイラに保証させるパターンです。`default` ブロックで変数を `never` 型に代入することで実現します。

```typescript
type Status = "pending" | "approved" | "rejected";

function handleStatus(status: Status): string {
  switch (status) {
    case "pending":
      return "審査中です";
    case "approved":
      return "承認されました";
    case "rejected":
      return "却下されました";
    default:
      const _exhaustive: never = status;
      throw new Error(`Unknown status: ${_exhaustive}`);
  }
}
```

このパターンの威力は、**新しいケースが追加されたとき**に発揮されます。`Status` に `"on_hold"` を追加して `case` を書き忘れると、`default` ブロックで `never` への代入がコンパイルエラーになり、処理漏れを検出できます。

再利用しやすいように `assertNever` ヘルパー関数にまとめるのが一般的です。`default` で `assertNever(value)` を呼ぶだけで網羅性チェックが入ります。

```typescript
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${value}`);
}
```

## never 型の活用

`never` は「ありえない値」を表す型です。網羅性チェック以外にも活用できます。

```typescript
// 関数が正常に終了しないことを示す
function throwError(message: string): never {
  throw new Error(message);
}

// ユニオン型から自動的に除去される
type Clean = string | number | never; // string | number

// 型のフィルタリングに使う
type ExtractStrings<T> = T extends string ? T : never;
type Result = ExtractStrings<"a" | 1 | "b" | true>;
// "a" | "b"
```

## Record による網羅性チェック

`switch` 文の代わりに `Record` 型を使う方法もあります。すべてのキーに値を設定しないとコンパイルエラーになります。

```typescript
type Fruit = "apple" | "banana" | "cherry";

const fruitColors: Record<Fruit, string> = {
  apple: "red",
  banana: "yellow",
  cherry: "dark red",
  // キーを1つでも省略するとコンパイルエラー
};
```

## 交差型と判別可能なユニオンの組み合わせ

交差型で共通部分をまとめ、判別可能なユニオンで分岐するパターンです。実務で頻繁に登場します。

```typescript
type EventMetadata = {
  timestamp: number;
  source: string;
};

type UserEvent = EventMetadata & {
  type: "user_created";
  data: { userId: string; email: string };
};

type OrderEvent = EventMetadata & {
  type: "order_placed";
  data: { orderId: string; amount: number };
};

type AppEvent = UserEvent | OrderEvent;

function handleEvent(event: AppEvent): void {
  // timestamp, source は共通でアクセスできる
  console.log(`[${event.source}] ${event.timestamp}`);

  switch (event.type) {
    case "user_created":
      console.log(`User: ${event.data.userId}`);
      break;
    case "order_placed":
      console.log(`Order: ${event.data.orderId}`);
      break;
    default:
      assertNever(event);
  }
}
```

## まとめ

| 概念 | 要点 |
|------|------|
| 交差型 (`&`) | 複数の型を合成し、すべてのプロパティを持つ型を作る |
| 型の合成パターン | 小さな型を `&` で組み合わせて大きな型を構築する |
| 型の矛盾 | 同名プロパティの型が矛盾すると `never` になる |
| 判別可能なユニオン | 共通のリテラル型プロパティ（判別子）で型を安全に分岐する |
| 判別子のルール | リテラル型 / 名前統一 / 値の一意性 の3点を守る |
| 網羅性チェック | `default` で `never` に代入し、処理漏れをコンパイル時に検出 |
| assertNever | 網羅性チェック用のヘルパー関数 |
| never 型 | 「ありえない値」を表す型。型のフィルタリングにも使う |
| Record による網羅性 | `Record<Union, T>` で全キーの網羅を強制する |

## やってみよう！

### 練習1: 交差型で型を合成する

以下の3つの型を交差型で合成して `Product` 型を作り、値を代入してください。

```typescript
type HasId = { id: number };
type HasName = { name: string };
type HasPrice = { price: number; currency: string };

// Product 型を定義してください
type Product = /* ここに実装 */;

// product に値を代入してください
const product: Product = {
  // ここに実装
};

console.log(`${product.name}: ${product.price} ${product.currency}`);
```

### 練習2: 判別可能なユニオンと網羅性チェック

通知の種類（email / sms / push）を判別可能なユニオンで定義し、メッセージを返す関数を `assertNever` 付きで実装してください。

```typescript
type Notification = /* ここに実装 */;

function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${value}`);
}

function formatNotification(notification: Notification): string {
  // switch 文で分岐し、default で assertNever を呼んでください
}

// テスト
formatNotification({ type: "email", to: "a@example.com", subject: "Hello" });
formatNotification({ type: "sms", phoneNumber: "090-1234-5678", body: "Hi" });
formatNotification({ type: "push", userId: "user-1", title: "New!" });
```

### 練習3: 交差型と判別可能なユニオンを組み合わせる

`BaseEvent` を交差型で共有しつつ、3種類のイベントを定義してください。`handleEvent` 関数で各イベントを処理し、網羅性チェックも入れてください。

```typescript
type BaseEvent = {
  timestamp: Date;
  userId: string;
};

// 3種類のイベントを定義してください
// - "login": { device: string }
// - "purchase": { itemId: string; amount: number }
// - "logout": { reason: string }
type AppEvent = /* ここに実装 */;

function handleEvent(event: AppEvent): string {
  // ここに実装してください
}
```
