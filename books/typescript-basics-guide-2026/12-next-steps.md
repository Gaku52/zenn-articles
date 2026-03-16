---
title: "次のステップ"
---

# 次のステップ

> 本書で学んだ型の基礎を土台に、TypeScriptの世界をさらに広げる

本書では、TypeScriptの型システムの基礎から実践パターンまでを学んできました。この章では、本書ではカバーしきれなかった高度な型機能と、TypeScriptを活かすためのエコシステムを紹介します。

## 条件付き型（Conditional Types）

条件付き型は、型レベルで条件分岐を表現します。

```typescript
// 基本構文: T extends U ? X : Y
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;  // true
type B = IsString<number>;  // false
```

Chapter 10で学んだ `Exclude` も、内部では条件付き型で実装されています。

```typescript
type MyExclude<T, U> = T extends U ? never : T;

type Result = MyExclude<"a" | "b" | "c", "a">; // "b" | "c"
```

ユニオン型に対して条件付き型を使うと、各メンバーに対して個別に評価される**分配的条件付き型**（Distributive Conditional Types）として動作します。

### `infer` キーワード

条件付き型の中で `infer` を使うと、型をパターンマッチで抽出できます。

```typescript
// 関数の戻り値の型を抽出する
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type Fn = (x: number) => string;
type Result = MyReturnType<Fn>; // string
```

Chapter 10で学んだ `ReturnType` は、まさにこの仕組みで実装されています。

## テンプレートリテラル型

テンプレートリテラル型は、文字列リテラル型を組み合わせて新しい型を生成します。

```typescript
type Color = "red" | "blue" | "green";
type Size = "small" | "large";

// すべての組み合わせが自動生成される
type ClassName = `${Size}-${Color}`;
// "small-red" | "small-blue" | "small-green" | "large-red" | "large-blue" | "large-green"
```

TypeScript組み込みのユーティリティとして、`Capitalize`, `Uppercase`, `Lowercase`, `Uncapitalize` も用意されています。

```typescript
type EventName = `on${Capitalize<"click" | "focus" | "blur">}`;
// "onClick" | "onFocus" | "onBlur"
```

## 型レベルプログラミングの世界

TypeScriptの型システムでは、型レベルで高度な変換を行えます。

### Mapped Types のキーリネーム

Chapter 10で触れたMapped Typesに `as` 句を加えると、キーを変換できます。

```typescript
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type User = { name: string; age: number };
type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number }
```

### 再帰的な型

型定義の中で自分自身を参照する再帰的な型も定義できます。

```typescript
type JsonValue =
  | string
  | number
  | boolean
  | null
  | JsonValue[]
  | { [key: string]: JsonValue };
```

型レベルプログラミングは奥深い分野です。日常の開発で必要になることは少ないですが、ライブラリの型定義を読み解く際に役立ちます。

## TypeScript と React / Next.js

React は TypeScript との統合が進んでおり、型安全なコンポーネント開発ができます。

```typescript
import { ReactNode, useState, ChangeEvent } from "react";

// Props の型定義
type ButtonProps = {
  label: string;
  onClick: () => void;
  variant?: "primary" | "secondary";
};

function Button({ label, onClick, variant = "primary" }: ButtonProps) {
  return (
    <button onClick={onClick} className={variant}>{label}</button>
  );
}

// children の型
type LayoutProps = { children: ReactNode; title: string };

// Hooks の型（ジェネリクスで state の型を指定）
type User = { id: number; name: string };

function useUser(id: number) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true); // boolean と推論される
  // ...
  return { user, loading };
}

// イベントの型
function handleChange(event: ChangeEvent<HTMLInputElement>) {
  console.log(event.target.value);
}
```

Next.js（App Router）では、サーバーコンポーネントとクライアントコンポーネントのそれぞれで型安全なデータフェッチが可能です。

## TypeScript と Node.js / Express

Node.js でのサーバーサイド開発でも TypeScript は広く使われています。型定義は `@types/node` と `@types/express` で提供されます。

```typescript
import express, { Request, Response } from "express";

const app = express();
app.use(express.json());

type CreateUserBody = { name: string; email: string };

app.post("/users", (req: Request<{}, {}, CreateUserBody>, res: Response) => {
  const { name, email } = req.body; // 型安全にアクセスできる
  res.json({ id: 1, name, email });
});
```

Node.js 22 LTS の組み込みモジュールも `@types/node` で型が提供されています。

```typescript
import { readFile } from "node:fs/promises";
import { join } from "node:path";

async function loadConfig(filename: string): Promise<string> {
  const filepath = join(process.cwd(), filename);
  return readFile(filepath, "utf-8");
}
```

## Zod（ランタイムバリデーション）

Chapter 11で見たように、TypeScriptの型チェックはコンパイル時のみです。外部データには、ランタイムでの検証も必要です。

[Zod](https://zod.dev/) は、スキーマ定義からTypeScriptの型を自動生成できるバリデーションライブラリです。

```typescript
import { z } from "zod";

// スキーマ定義（バリデーションルール）
const UserSchema = z.object({
  id: z.number(),
  name: z.string().min(1),
  email: z.string().email(),
  age: z.number().int().min(0).optional(),
});

// スキーマから型を生成
type User = z.infer<typeof UserSchema>;
// { id: number; name: string; email: string; age?: number }

// ランタイムバリデーション
function parseUser(data: unknown): User {
  return UserSchema.parse(data); // 不正データなら ZodError をスロー
}
```

Zodの最大のメリットは、**型定義とバリデーションルールを一箇所で管理できる**点です。

## tRPC（型安全なAPI）

[tRPC](https://trpc.io/) は、フロントエンドとバックエンド間のAPI通信を型安全にするフレームワークです。

```typescript
// サーバー側
import { initTRPC } from "@trpc/server";
import { z } from "zod";

const t = initTRPC.create();

const appRouter = t.router({
  getUser: t.procedure
    .input(z.object({ id: z.number() }))
    .query(async ({ input }) => {
      return { id: input.id, name: "Alice" }; // input.id は number 型
    }),
});

export type AppRouter = typeof appRouter;
```

```typescript
// クライアント側
import { createTRPCClient } from "@trpc/client";
import type { AppRouter } from "./server";

const client = createTRPCClient<AppRouter>({ /* 設定 */ });

const user = await client.getUser.query({ id: 1 });
console.log(user.name); // string として推論される
// client.getUser.query({ id: "abc" }); // Error: 型エラー！
```

tRPCはZodと組み合わせることで、入力バリデーションと型安全性の両方を実現します。

## おすすめの学習リソース

### 公式ドキュメント・演習

- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/) -- TypeScript公式のハンドブック
- [type-challenges](https://github.com/type-challenges/type-challenges) -- 型パズル集（Easy〜Extreme）
- [TypeScript Playground](https://www.typescriptlang.org/play) -- ブラウザ上で試せる公式ツール

### フレームワーク・ライブラリ

- [React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app/) -- React + TypeScript のベストプラクティス集
- [Zod ドキュメント](https://zod.dev/) -- ランタイムバリデーション
- [tRPC ドキュメント](https://trpc.io/docs) -- 型安全なAPI

## この本から先の学習ロードマップ

本書で学んだ内容を土台に、以下のステップで学習を進められます。

### ステップ1: 型システムを深める

- **条件付き型**と `infer` -- ライブラリの型定義を読み解く力が付きます
- **テンプレートリテラル型** -- 文字列パターンの型安全性を実現します
- **type-challenges** への挑戦 -- Easyレベルから始めて型の応用力を鍛えます

### ステップ2: フレームワークと組み合わせる

- **React + TypeScript** -- コンポーネント、Hooks、Context の型定義
- **Next.js + TypeScript** -- App Router、Server Actions の型安全なデータフェッチ
- **Node.js + TypeScript** -- サーバーサイドAPI、CLIツールの開発

### ステップ3: エコシステムを活用する

- **Zod** -- 外部データのランタイムバリデーション
- **tRPC** -- フロントエンドとバックエンド間の型安全な通信
- **Prisma** -- データベース操作の型安全なORM
- **ESLint + typescript-eslint** -- 型情報を活用したリントルール

### ステップ4: 型の設計力を磨く

- **Branded Types** -- プリミティブ型に意味を持たせるパターン
- **型安全なエラーハンドリング** -- Result型パターン
- **Declaration Files** (`.d.ts`) -- ライブラリの型定義の読み書き

---

本書を通じてTypeScriptの型システムの基礎を学んできました。TypeScriptは「コードを書く段階でバグを防ぐ」という考え方に基づいた言語です。型を正しく使うことで、エディタの補完、リファクタリングの安全性、チーム開発でのコミュニケーション、すべてが改善されます。

ここで学んだ知識は、どのフレームワークやライブラリを使う場合にも活きる基礎です。ぜひ実際のプロジェクトで型を書き、エラーメッセージと向き合いながら、TypeScriptの力を体感してください。

## まとめ

| 概念 | 要点 |
|:---|:---|
| 条件付き型 | `T extends U ? X : Y` で型レベルの条件分岐を表現する。`infer` で型のパターンマッチも可能 |
| テンプレートリテラル型 | 文字列リテラル型を組み合わせて新しい文字列パターンの型を生成する |
| 型レベルプログラミング | Mapped Types のキーリネーム、再帰型など、型を動的に構築する高度な手法 |
| React / Next.js | Props、Hooks、イベントハンドラなど、コンポーネント開発での型の活用 |
| Node.js / Express | リクエスト・レスポンスの型定義、`@types/node` による組み込みモジュールの型サポート |
| Zod | スキーマ定義から型を生成し、ランタイムバリデーションも行うライブラリ |
| tRPC | フロントエンドとバックエンド間でTypeScriptの型をそのまま共有するAPIフレームワーク |
| 学習ロードマップ | 型システムの深化 → フレームワーク活用 → エコシステム導入 → 型設計力の向上 |
