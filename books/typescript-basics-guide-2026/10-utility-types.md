---
title: "ユーティリティ型"
---

# ユーティリティ型

TypeScriptには、既存の型を変換して新しい型を作る**ユーティリティ型**が組み込まれています。同じような型定義を繰り返し書く必要がなくなり、型の再利用性が大きく向上します。

この章では、実務で頻繁に使われる11個のユーティリティ型を「何をするか」「いつ使うか」の視点で解説します。以降の例では、次の `User` 型をベースに説明します。

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}
```

---

## Partial\<T\> -- 全プロパティをオプショナルにする

`Partial<T>` は、すべてのプロパティを省略可能（`?` 付き）にします。オブジェクトの**部分更新**を受け付ける関数で使います。

```typescript
function updateUser(id: number, updates: Partial<User>): void {
  // name だけ、email だけなど、一部だけ渡せる
}

updateUser(1, { name: "Alice" });    // OK
updateUser(1, { email: "a@b.com" }); // OK
```

## Required\<T\> -- 全プロパティを必須にする

`Required<T>` は `Partial` の逆で、オプショナルプロパティをすべて必須にします。デフォルト値でマージした後の「確定済みの設定」を表現するときに便利です。

```typescript
interface Config {
  host?: string;
  port?: number;
  debug?: boolean;
}

const defaults: Required<Config> = {
  host: "localhost",
  port: 3000,
  debug: false,
};
```

## Readonly\<T\> -- 全プロパティを読み取り専用にする

`Readonly<T>` は、すべてのプロパティに `readonly` を付けます。関数の引数に使うと、オブジェクトを変更しないことを型レベルで保証できます。

```typescript
function displayUser(user: Readonly<User>): void {
  console.log(user.name);
  // user.name = "Bob";  // Error: readonly プロパティに代入できない
}
```

---

## Pick\<T, K\> -- 特定のプロパティだけを取り出す

`Pick<T, K>` は、指定したプロパティだけを抽出した新しい型を作ります。

```typescript
// 表示に必要な情報だけ抽出
type ProfileCard = Pick<User, "name" | "email">;
// { name: string; email: string }
```

## Omit\<T, K\> -- 特定のプロパティを除外する

`Omit<T, K>` は `Pick` の逆で、指定したプロパティを除外した型を作ります。

```typescript
// password を除外した公開用の型
type PublicUser = Omit<User, "password">;
// { id: number; name: string; email: string; createdAt: Date }
```

**Pick と Omit の使い分け**: 少数のプロパティが欲しいときは `Pick`、少数を除きたいときは `Omit` を使います。プロパティが増えても安全なのは `Omit`（新しいプロパティが自動的に含まれる）です。

---

## Record\<K, V\> -- キーと値の型を指定した辞書型を作る

`Record<K, V>` は、キーの型が `K`、値の型が `V` であるオブジェクト型を作ります。キーの候補が決まっている辞書やルックアップテーブルの定義に使います。

```typescript
type ErrorMessages = Record<"notFound" | "unauthorized" | "serverError", string>;

const messages: ErrorMessages = {
  notFound: "ページが見つかりません",
  unauthorized: "ログインが必要です",
  serverError: "サーバーエラーが発生しました",
};
```

---

## ReturnType\<T\> -- 関数の戻り値の型を取得する

`ReturnType<T>` は、関数型の戻り値の型を抽出します。外部ライブラリの関数から型を取得したいときに便利です。

```typescript
function getUser() {
  return { id: 1, name: "Alice", active: true };
}

type UserInfo = ReturnType<typeof getUser>;
// { id: number; name: string; active: boolean }
```

:::message
`typeof` を付けるのを忘れがちです。`ReturnType` は**関数の型**を受け取るため、関数の値ではなく `typeof 関数名` を渡します。
:::

## Parameters\<T\> -- 関数の引数の型をタプルで取得する

`Parameters<T>` は、関数の引数の型をタプル型として返します。既存の関数と同じ引数を受け取るラッパー関数を作るときに使います。

```typescript
function sendMessage(to: string, body: string, priority: number): void {
  // ...
}

// 同じ引数を受け取るラッパー
function logAndSend(...args: Parameters<typeof sendMessage>): void {
  console.log("Sending:", args);
  sendMessage(...args);
}
```

---

## NonNullable\<T\> -- null と undefined を除外する

`NonNullable<T>` は、型から `null` と `undefined` を取り除きます。

```typescript
type MaybeString = string | null | undefined;
type DefiniteString = NonNullable<MaybeString>;  // string
```

## Exclude\<T, U\> -- ユニオン型から特定の型を除外する

`Exclude<T, U>` は、ユニオン型から指定した型を取り除きます。

```typescript
type Status = "active" | "inactive" | "pending" | "deleted";
type VisibleStatus = Exclude<Status, "deleted">;
// "active" | "inactive" | "pending"
```

## Extract\<T, U\> -- ユニオン型から特定の型だけ抽出する

`Extract<T, U>` は `Exclude` の逆で、ユニオン型から指定した型だけを残します。

```typescript
type AllEvents = "click" | "focus" | "blur" | "submit" | "reset";
type FormEvents = Extract<AllEvents, "submit" | "reset">;
// "submit" | "reset"
```

:::message
`Exclude` と `Omit` は名前が似ていますが操作対象が異なります。`Exclude` は**ユニオン型**に対して、`Omit` は**オブジェクト型**に対して使います。
:::

---

## ユーティリティ型の組み合わせ

ユーティリティ型の真価は、組み合わせて使うことで発揮されます。

### CRUD型の定義パターン

```typescript
// 作成時は id とタイムスタンプが不要
type CreateUser = Omit<User, "id" | "createdAt">;

// 更新時は全プロパティオプショナル（id は指定不可）
type UpdateUser = Partial<Omit<User, "id" | "createdAt">>;

// 公開用はパスワードを除外して読み取り専用
type PublicUser = Readonly<Omit<User, "password">>;
```

### 一部だけオプショナルにする

```typescript
type PartialBy<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;

interface Article {
  title: string;
  body: string;
  tags: string[];
  publishedAt: Date;
}

// publishedAt だけオプショナル（下書きの場合は未設定）
type DraftArticle = PartialBy<Article, "publishedAt">;
```

### Record と Readonly の組み合わせ

```typescript
type Role = "admin" | "editor" | "viewer";

interface Permission {
  read: boolean;
  write: boolean;
  delete: boolean;
}

// ロールごとの変更不可な権限マップ
type RolePermissions = Record<Role, Readonly<Permission>>;

const permissions: RolePermissions = {
  admin:  { read: true,  write: true,  delete: true },
  editor: { read: true,  write: true,  delete: false },
  viewer: { read: true,  write: false, delete: false },
};
```

---

## ユーティリティ型の仕組み（Mapped Types）

ここまで紹介したユーティリティ型の多くは、内部的に **Mapped Types（マップ型）** という仕組みで実装されています。深く理解する必要はありませんが、仕組みを知っておくと型エラーのメッセージが読みやすくなります。

Mapped Types の基本構文は `{ [K in keyof T]: ... }` です。`T` のプロパティを1つずつ取り出して、新しい型を組み立てます。

```typescript
// Partial<T> の内部実装（イメージ）
type MyPartial<T> = {
  [K in keyof T]?: T[K];  // 各プロパティに ? を付ける
};

// Readonly<T> の内部実装（イメージ）
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];  // 各プロパティに readonly を付ける
};
```

`[K in keyof T]` は「`T` のキーを1つずつ `K` に代入しながら繰り返す」という意味です。`T[K]` はそのキーに対応する値の型を取り出します。

```typescript
type User = { name: string; age: number };
type ReadonlyUser = MyReadonly<User>;
// { readonly name: string; readonly age: number }
```

型エラーで `{ [K in keyof T]?: T[K] }` のような表記を見かけたら、「各プロパティを変換しているんだな」と理解できれば十分です。

---

## まとめ

| 概念 | 要点 |
|------|------|
| `Partial<T>` | 全プロパティをオプショナルにする。部分更新に使う |
| `Required<T>` | 全プロパティを必須にする。デフォルト値マージ後に使う |
| `Readonly<T>` | 全プロパティを読み取り専用にする。変更禁止の保証に使う |
| `Pick<T, K>` | 指定プロパティだけ抽出する。少数を取り出すときに使う |
| `Omit<T, K>` | 指定プロパティを除外する。少数を除くときに使う |
| `Record<K, V>` | キーと値の型を指定した辞書型を作る |
| `ReturnType<T>` | 関数の戻り値の型を取得する。`typeof` と併用する |
| `Parameters<T>` | 関数の引数の型をタプルで取得する |
| `NonNullable<T>` | `null` と `undefined` を型から除外する |
| `Exclude<T, U>` | ユニオン型から特定の型を除外する |
| `Extract<T, U>` | ユニオン型から特定の型だけを抽出する |
| 組み合わせ | `Partial<Omit<T, K>>` のように組み合わせて実務の型を作る |
| Mapped Types | `{ [K in keyof T]: T[K] }` でユーティリティ型が内部的に動作する仕組み |

---

## やってみよう！

### 練習1: CRUD用の型を定義する

以下の `Product` 型から、作成用・更新用・一覧表示用の型をユーティリティ型で作成してください。

```typescript
interface Product {
  id: number;
  name: string;
  price: number;
  description: string;
  categoryId: number;
  createdAt: Date;
  updatedAt: Date;
}

// 1. 作成用の型（id, createdAt, updatedAt は自動生成されるので不要）
type CreateProduct = /* ??? */;

// 2. 更新用の型（id は変更不可、それ以外はオプショナル）
type UpdateProduct = /* ??? */;

// 3. 一覧表示用の型（id, name, price だけあればよい）
type ProductListItem = /* ??? */;
```

### 練習2: ステータス管理の型を作る

`Record` と `Exclude` を使って、以下の要件を満たす型を定義してください。

```typescript
type OrderStatus = "cart" | "pending" | "paid" | "shipped" | "delivered" | "cancelled";

// 1. 各ステータスにラベルを持たせる型を Record で作る
type StatusLabels = /* ??? */;

// 2. 「完了済み」のステータスだけを抽出する（delivered と cancelled）
type FinishedStatus = /* ??? */;

// 3. 「アクティブ」なステータスだけを残す（cart, cancelled を除外）
type ActiveStatus = /* ??? */;
```

### 練習3: 関数の型情報を活用する

`ReturnType` と `Parameters` を使って、以下の関数の型情報を取得してください。

```typescript
function fetchUserById(id: number, options?: { includeProfile: boolean }) {
  return { id, name: "Alice", email: "alice@example.com" };
}

// 1. fetchUserById の戻り値の型を取得する
type UserResult = /* ??? */;

// 2. fetchUserById と同じ引数を受け取るラッパー関数の型を定義する
type FetchUserFn = (...args: /* ??? */) => Promise</* ??? */>;
```

---

次の章では、型安全なコーディングの実践パターンを学びます。
