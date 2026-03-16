---
title: "型安全なコーディング実践"
---

# 型安全なコーディング実践

> `any` に頼らず、TypeScriptの型システムを最大限に活かすための実践パターン

ここまでの章で、TypeScriptの型システムの基礎から応用までを学んできました。この章では、実務で型安全なコードを書くための具体的なパターンとアンチパターンを紹介します。

## `any` を避ける

`any` 型はTypeScriptの型チェックを完全に無効化します。使った瞬間から型安全性が失われ、実行時エラーの原因になります。

```typescript
function processData(data: any) {
  console.log(data.name.toUpperCase()); // コンパイルは通るが...
}

processData(null); // 実行時エラー！
```

型が不明な値を扱う場合は、`unknown` 型を使います。`unknown` は型を確認するまで操作ができません。

```typescript
function processData(data: unknown) {
  // data.name; // Error: 'data' is of type 'unknown'

  if (typeof data === "object" && data !== null && "name" in data) {
    const name = data.name;
    if (typeof name === "string") {
      console.log(name.toUpperCase()); // OK
    }
  }
}
```

ユーザー定義型ガード（Chapter 07の `is` 構文）と組み合わせると、より読みやすくなります。

```typescript
type User = { name: string; age: number };

function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    "name" in value &&
    typeof (value as Record<string, unknown>).name === "string" &&
    "age" in value &&
    typeof (value as Record<string, unknown>).age === "number"
  );
}

function greetUser(data: unknown) {
  if (isUser(data)) {
    console.log(`Hello, ${data.name}!`); // data は User 型
  }
}
```

## 型アサーション（`as`）を最小限にする

型アサーションは、コンパイラの型チェックを上書きします。誤用すると実行時エラーにつながります。

```typescript
const user = {} as { name: string; email: string };
console.log(user.email.toUpperCase()); // 実行時エラー！
```

型アサーションが適切なケースは限られています。DOM要素の取得はその一つですが、型ガードの方がより安全です。

```typescript
// 型アサーション（HTMLの構造を把握している場合）
const input = document.getElementById("username") as HTMLInputElement;

// より安全: 型ガードを使う
const el = document.getElementById("username");
if (el instanceof HTMLInputElement) {
  el.value = "TypeScript"; // 型ガードで安全
}
```

`as const` は型アサーションとは異なり、リテラル型への絞り込みとして安全に使えます。

```typescript
const config = { mode: "development" as const, port: 3000 as const };
// config.mode の型は "development"（string ではなく）
```

## `strict` モードの各オプション

`tsconfig.json` で `"strict": true` を設定すると、複数の厳格な型チェックが有効になります。

**`strictNullChecks`** -- `null` と `undefined` を他の型と区別します。

```typescript
function getLength(str: string | null): number {
  // str.length; // Error: 'str' is possibly 'null'
  return str === null ? 0 : str.length; // OK
}
```

**`noImplicitAny`** -- 型が推論できない場合に暗黙の `any` を禁止します。

```typescript
// function parse(data) { } // Error: implicitly has an 'any' type
function parse(data: string) { return JSON.parse(data); } // OK
```

**`noUncheckedIndexedAccess`** -- インデックスアクセスの結果に `undefined` を含めます。

```typescript
const colors = ["red", "green", "blue"];
const first = colors[0]; // 型は string | undefined
if (first !== undefined) {
  console.log(first.toUpperCase()); // OK
}
```

推奨設定は以下の通りです。

```jsonc
{
  "compilerOptions": {
    "strict": true,                   // 全strictオプションを有効化
    "noUncheckedIndexedAccess": true, // strictに含まれないが推奨
    "noUnusedLocals": true,           // 未使用変数を警告
    "noUnusedParameters": true        // 未使用引数を警告
  }
}
```

## 型エラーの読み方と対処法

TypeScriptの型エラーメッセージが複数行にわたる場合、**最後の行が最も具体的な原因**を示しています。

```
Type '{ name: string; }' is not assignable to type 'User'.
  Property 'email' is missing in type '{ name: string; }' but required in type 'User'.
```

この例では、`email` プロパティの不足が原因です。

| エラーパターン | 原因 | 対処法 |
|:---|:---|:---|
| `is not assignable to` | 型の不一致 | 代入する値の型を確認する |
| `is possibly 'null'` | null チェック不足 | `if (value !== null)` で絞り込む |
| `is possibly 'undefined'` | undefined チェック不足 | オプショナルチェーン `?.` を使う |
| `Property ... does not exist` | 存在しないプロパティ | スペルミスを確認する |
| `implicitly has an 'any' type` | 型注釈の不足 | 明示的に型を指定する |

## 実務でよくあるパターン

### APIレスポンスの型定義

```typescript
type ApiResponse<T> = { data: T; status: number; message: string };
type UserResponse = { id: number; name: string; email: string };

async function fetchJson<T>(url: string): Promise<T> {
  const response = await fetch(url);
  if (!response.ok) throw new Error(`HTTP Error: ${response.status}`);
  const json: unknown = await response.json();
  return json as T; // Zodを導入するとランタイム検証も可能（Chapter 12参照）
}

async function getUser(id: number) {
  const result = await fetchJson<ApiResponse<UserResponse>>(`/api/users/${id}`);
  console.log(result.data.name);
}
```

### イベントハンドラの型

```typescript
// DOM の場合
const button = document.querySelector("button");
button?.addEventListener("click", (event: MouseEvent) => {
  console.log(event.clientX, event.clientY);
});

// React の場合
import { ChangeEvent, FormEvent } from "react";

function handleChange(event: ChangeEvent<HTMLInputElement>) {
  console.log(event.target.value);
}
```

### 環境変数の型安全な読み取り

Node.js の環境変数（`process.env`）はすべて `string | undefined` 型です。バリデーション関数で型安全に扱います。

```typescript
function getEnvVar(key: string): string {
  const value = process.env[key];
  if (value === undefined) {
    throw new Error(`Environment variable ${key} is not set`);
  }
  return value;
}

const config = {
  databaseUrl: getEnvVar("DATABASE_URL"),
  port: Number(getEnvVar("PORT")),
  nodeEnv: getEnvVar("NODE_ENV"),
};
```

## `satisfies` 演算子

TypeScript 4.9で導入された `satisfies` 演算子は、型チェックを行いつつ、推論された型（リテラル型など）を保持する仕組みです。型アノテーション（`: Type`）では推論がその型に固定されてしまいますが、`satisfies` なら元の推論結果を維持できます。

```typescript
type Config = { port: number; debug: boolean };

// 型アノテーション: 推論が Config に固定される
const config1: Config = { port: 3000, debug: true };
// config1.port → number

// satisfies: 型チェックしつつリテラル型を保持
const config2 = { port: 3000, debug: true } satisfies Config;
// config2.port → 3000（リテラル型が保持される）
```

`as const` はオブジェクト全体を `readonly` にしますが、`satisfies` は型の構造チェックだけを行い、変更可能性や推論型には影響しません。「型に適合するか確認したいが、推論を狭めたくない」場面で使いましょう。

## アンチパターン集

### `as any` で型エラーを消す

```typescript
// Bad: 型エラーの根本原因を隠している
const result = someFunction() as any;

// Good: 型を正しく定義する
const result: ExpectedType = someFunction();
```

### `@ts-ignore` の乱用

`@ts-ignore` はどんな型エラーも無視します。どうしても必要な場合は `@ts-expect-error` を使いましょう。`@ts-expect-error` はエラーが解消された場合に警告してくれるため、より安全です。

### 過剰な型定義

```typescript
// Bad: 推論で十分なのに手動で型を書いている
const name: string = "TypeScript";
const items: string[] = ["a", "b", "c"];

// Good: 型推論に任せる
const name = "TypeScript";
const items = ["a", "b", "c"];
```

**型推論で正しく推論される箇所には、型注釈を書かない**のが基本方針です。型注釈を書くべき場面は、関数の引数・戻り値、`unknown` を扱う場面、リテラル型が必要な場合などに限られます。

## やってみよう！

### 練習1: `unknown` で安全にデータを処理する

以下の関数を、`any` を使わずに `unknown` と型ガードで書き直してください。

```typescript
function formatValue(value: any): string {
  if (value.toFixed) {
    return value.toFixed(2);
  }
  return value.toString();
}
```

:::details 解答例

```typescript
function formatValue(value: unknown): string {
  if (typeof value === "number") {
    return value.toFixed(2);
  }
  if (typeof value === "string") {
    return value;
  }
  return String(value);
}
```

`typeof` による型ガードで、値の型を安全に判定してから操作しています。

:::

### 練習2: APIレスポンスに型を付ける

以下のAPIレスポンスに対して、型定義と型ガード関数を作成してください。

```typescript
// レスポンス: { "id": 1, "title": "Learn TypeScript", "completed": false }

// 1. Todo 型を定義
// 2. isTodo 型ガード関数を作成
// 3. fetchTodo 関数で型ガードを使う
async function fetchTodo(id: number) {
  const response = await fetch(`/api/todos/${id}`);
  const data: unknown = await response.json();
  // ここで型ガードを使って安全に返す
}
```

:::details 解答例

```typescript
type Todo = { id: number; title: string; completed: boolean };

function isTodo(value: unknown): value is Todo {
  return (
    typeof value === "object" && value !== null &&
    "id" in value && typeof (value as Record<string, unknown>).id === "number" &&
    "title" in value && typeof (value as Record<string, unknown>).title === "string" &&
    "completed" in value && typeof (value as Record<string, unknown>).completed === "boolean"
  );
}

async function fetchTodo(id: number): Promise<Todo> {
  const response = await fetch(`/api/todos/${id}`);
  const data: unknown = await response.json();
  if (isTodo(data)) return data;
  throw new Error("Invalid todo data");
}
```

:::

### 練習3: 型エラーを修正する

以下のコードの型エラーを、`as any` や `@ts-ignore` を使わずに修正してください。

```typescript
type Config = { host: string; port: number; ssl: boolean };

function createConfig(overrides: Partial<Config>): Config {
  const defaults: Config = { host: "localhost", port: 3000, ssl: false };
  return { ...defaults, ...overrides };
}

const config = createConfig({ port: "8080", ssl: "true" });
//                                   ^^^^^^       ^^^^^^ 型エラー！
```

:::details 解答例

```typescript
const config = createConfig({ port: 8080, ssl: true });
```

`port` に文字列 `"8080"` を渡していたのを数値 `8080` に、`ssl` に文字列 `"true"` を渡していたのを真偽値 `true` に修正しました。型エラーは「間違った型の値を渡している」ことを教えてくれるサインです。

:::

## まとめ

| 概念 | 要点 |
|:---|:---|
| `any` と `unknown` | `any` は型チェックを無効化する。型が不明な場合は `unknown` を使い、型ガードで絞り込む |
| 型アサーション（`as`） | コンパイラの型チェックを上書きする。DOM要素の取得など限定的な場面でのみ使う |
| `strict` モード | `strictNullChecks`, `noImplicitAny` など複数の厳格チェックを有効化する。常に有効にすることを推奨 |
| 型エラーの読み方 | エラーメッセージの最後の行が最も具体的な原因。「期待される型」と「実際の型」の差分を確認する |
| APIレスポンスの型 | 外部データは型が保証されない。型定義＋型ガード、またはZodでランタイム検証を行う |
| アンチパターン | `as any`, `@ts-ignore` の乱用, 過剰な型注釈を避ける。型推論が十分な場面では型を書かない |
