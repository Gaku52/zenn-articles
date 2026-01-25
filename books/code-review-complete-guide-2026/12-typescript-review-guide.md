---
title: "第12章 TypeScript/JavaScriptレビューガイド"
---

# 第12章 TypeScript/JavaScriptレビューガイド

本章ではTypeScript/JavaScriptレビューガイドについて、実践的な観点から詳しく解説します。TypeScriptは型安全性を提供することで、JavaScriptの弱点を補完し、大規模開発における品質向上に貢献します。適切な型定義とベストプラクティスに従うことで、バグの早期発見と保守性の高いコードを実現できます。

## 概要

TypeScript/JavaScriptレビューは、型定義の適切性、言語機能の効果的な使用、パフォーマンス、保守性の観点からコード品質を評価するプロセスです。本章では、型定義の適切性（any型の乱用回避）、ジェネリクスの活用、nullチェックの徹底、enum vs union type、readonly・constの活用など、具体的なチェックポイントとコード例を通じて、効果的なレビュー方法を学びます。

## 型定義の適切性

### any型の乱用回避

any型は型チェックを無効化するため、TypeScriptの利点を失います。可能な限り具体的な型を定義しましょう。

```typescript
// 悪い例: any型の乱用
function processData(data: any): any {
  return data.map((item: any) => item.value);
}

// 良い例: 具体的な型定義
interface DataItem {
  id: string;
  value: number;
  label: string;
}

function processDataTyped(data: DataItem[]): number[] {
  return data.map((item) => item.value);
}
```

どうしても型が不明な場合は、`unknown`型を使用し、型ガードで絞り込みます。

```typescript
// unknown型と型ガードの使用
function processUnknownData(data: unknown): string {
  if (typeof data === 'string') {
    return data.toUpperCase();
  }

  if (typeof data === 'number') {
    return data.toString();
  }

  if (isDataItem(data)) {
    return data.label;
  }

  throw new Error('Unsupported data type');
}

function isDataItem(value: unknown): value is DataItem {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'value' in value &&
    'label' in value
  );
}
```

レビュー時には、any型の使用箇所を確認し、具体的な型またはunknown型に置き換えられないか検討しましょう。

### 型推論の活用

TypeScriptの型推論を活用することで、冗長な型注釈を避けられます。

```typescript
// 悪い例: 不要な型注釈
const message: string = 'Hello, World!';
const count: number = 10;
const isActive: boolean = true;

// 良い例: 型推論を活用
const message = 'Hello, World!'; // string型と推論される
const count = 10; // number型と推論される
const isActive = true; // boolean型と推論される

// 型注釈が必要なケース
function greet(name: string): string {
  return `Hello, ${name}!`;
}

let result: string; // 初期化前に宣言する場合は型注釈が必要
result = greet('Alice');
```

## ジェネリクスの活用

### 型の再利用性向上

ジェネリクスを使用することで、型安全性を保ちながらコードの再利用性を高められます。

```typescript
// 悪い例: 型ごとに関数を定義
function getFirstString(arr: string[]): string | undefined {
  return arr[0];
}

function getFirstNumber(arr: number[]): number | undefined {
  return arr[0];
}

// 良い例: ジェネリクスで汎用化
function getFirst<T>(arr: T[]): T | undefined {
  return arr[0];
}

// 使用例
const firstString = getFirst(['a', 'b', 'c']); // string | undefined
const firstNumber = getFirst([1, 2, 3]); // number | undefined
```

### 制約付きジェネリクス

```typescript
// 特定のプロパティを持つ型のみ受け入れる
interface HasId {
  id: string;
}

function findById<T extends HasId>(items: T[], id: string): T | undefined {
  return items.find((item) => item.id === id);
}

// 使用例
interface User extends HasId {
  name: string;
  email: string;
}

const users: User[] = [
  { id: '1', name: 'Alice', email: 'alice@example.com' },
  { id: '2', name: 'Bob', email: 'bob@example.com' }
];

const user = findById(users, '1'); // User | undefined
```

### ジェネリクスを使った型安全なAPIレスポンス

```typescript
// API レスポンスの型定義
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
}

interface User {
  id: string;
  name: string;
  email: string;
}

interface Post {
  id: string;
  title: string;
  content: string;
  authorId: string;
}

// 汎用的なAPI呼び出し関数
async function fetchApi<T>(url: string): Promise<ApiResponse<T>> {
  const response = await fetch(url);
  return response.json();
}

// 使用例
const userResponse = await fetchApi<User>('/api/users/1');
const user = userResponse.data; // User型として扱える

const postsResponse = await fetchApi<Post[]>('/api/posts');
const posts = postsResponse.data; // Post[]型として扱える
```

## nullチェックの徹底

### strictNullChecksの有効化

tsconfig.jsonで`strictNullChecks`を有効にすることで、null/undefinedの安全性が向上します。

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "strictNullChecks": true
  }
}
```

### Optional ChainingとNullish Coalescing

```typescript
// 悪い例: 冗長なnullチェック
function getUserEmail(user: User | null | undefined): string {
  if (user && user.profile && user.profile.email) {
    return user.profile.email;
  }
  return 'No email';
}

// 良い例: Optional ChainingとNullish Coalescingを使用
function getUserEmailSafe(user: User | null | undefined): string {
  return user?.profile?.email ?? 'No email';
}
```

### 型ガードによるnullチェック

```typescript
interface User {
  id: string;
  name: string;
  email?: string;
}

function isValidUser(user: User | null | undefined): user is User {
  return user !== null && user !== undefined && user.id !== '';
}

function processUser(user: User | null | undefined): void {
  if (!isValidUser(user)) {
    console.error('Invalid user');
    return;
  }

  // この時点でuserはUser型として扱える
  console.log(`Processing user: ${user.name}`);
}
```

## enum vs union type

### union typeの推奨

enumよりもunion typeを使用することが、モダンなTypeScriptでは推奨される傾向にあります。

```typescript
// enum の使用
enum UserRole {
  Admin = 'ADMIN',
  Editor = 'EDITOR',
  Viewer = 'VIEWER'
}

function checkPermission(role: UserRole): boolean {
  return role === UserRole.Admin;
}

// union type の使用（推奨）
type UserRoleUnion = 'ADMIN' | 'EDITOR' | 'VIEWER';

function checkPermissionUnion(role: UserRoleUnion): boolean {
  return role === 'ADMIN';
}

// const assertionでオブジェクトとして定義
const USER_ROLES = {
  ADMIN: 'ADMIN',
  EDITOR: 'EDITOR',
  VIEWER: 'VIEWER'
} as const;

type UserRoleFromObject = typeof USER_ROLES[keyof typeof USER_ROLES];

function checkPermissionObject(role: UserRoleFromObject): boolean {
  return role === USER_ROLES.ADMIN;
}
```

union typeの利点:
- JavaScriptへのトランスパイル時にコードが追加されない
- より軽量で直感的
- 型推論がより効果的に働く

### 数値enumの注意点

```typescript
// 悪い例: 数値enumは予期しない動作を起こす可能性
enum Status {
  Pending,
  Active,
  Inactive
}

const status: Status = 999; // エラーにならない（バグの温床）

// 良い例: 文字列リテラル型を使用
type StatusType = 'PENDING' | 'ACTIVE' | 'INACTIVE';

const statusSafe: StatusType = 'PENDING'; // 型安全
// const statusError: StatusType = 'INVALID'; // エラー
```

## readonly と const の活用

### イミュータビリティの確保

```typescript
// 悪い例: ミュータブルなデータ構造
interface User {
  id: string;
  name: string;
  roles: string[];
}

function addRole(user: User, role: string): void {
  user.roles.push(role); // 元のオブジェクトを変更してしまう
}

// 良い例: readonlyで不変性を保証
interface UserImmutable {
  readonly id: string;
  readonly name: string;
  readonly roles: readonly string[];
}

function addRoleImmutable(user: UserImmutable, role: string): UserImmutable {
  return {
    ...user,
    roles: [...user.roles, role] // 新しいオブジェクトを返す
  };
}
```

### constアサーション

```typescript
// 悪い例: 型が広く推論される
const config = {
  apiUrl: 'https://api.example.com',
  timeout: 5000
};
// config.apiUrl は string型と推論される

// 良い例: const assertionで厳密な型に
const configStrict = {
  apiUrl: 'https://api.example.com',
  timeout: 5000
} as const;
// configStrict.apiUrl は 'https://api.example.com' 型と推論される
// configStrict.timeout = 3000; // エラー: readonlyプロパティに代入できない
```

### 配列のイミュータビリティ

```typescript
// 悪い例: ミュータブルな配列
const numbers = [1, 2, 3, 4, 5];
numbers.push(6); // 変更可能

// 良い例: readonly配列
const numbersImmutable: readonly number[] = [1, 2, 3, 4, 5];
// numbersImmutable.push(6); // エラー: pushメソッドは存在しない

// 配列の操作は新しい配列を返すメソッドを使用
const newNumbers = [...numbersImmutable, 6];
```

## 型安全なエラーハンドリング

### Result型パターン

```typescript
// 成功または失敗を表す型
type Result<T, E = Error> =
  | { success: true; value: T }
  | { success: false; error: E };

// API呼び出しの例
async function fetchUserSafe(id: string): Promise<Result<User>> {
  try {
    const response = await fetch(`/api/users/${id}`);

    if (!response.ok) {
      return {
        success: false,
        error: new Error(`HTTP ${response.status}: ${response.statusText}`)
      };
    }

    const user = await response.json();
    return { success: true, value: user };
  } catch (error) {
    return {
      success: false,
      error: error instanceof Error ? error : new Error('Unknown error')
    };
  }
}

// 使用例
const result = await fetchUserSafe('123');

if (result.success) {
  console.log(result.value.name); // User型として扱える
} else {
  console.error(result.error.message); // Error型として扱える
}
```

### カスタムエラークラス

```typescript
// カスタムエラークラスの定義
class ValidationError extends Error {
  constructor(
    message: string,
    public field: string,
    public value: unknown
  ) {
    super(message);
    this.name = 'ValidationError';
  }
}

class NotFoundError extends Error {
  constructor(message: string, public resourceId: string) {
    super(message);
    this.name = 'NotFoundError';
  }
}

// エラーの型ガード
function isValidationError(error: unknown): error is ValidationError {
  return error instanceof ValidationError;
}

function isNotFoundError(error: unknown): error is NotFoundError {
  return error instanceof NotFoundError;
}

// 使用例
try {
  await updateUser(userId, userData);
} catch (error) {
  if (isValidationError(error)) {
    console.error(`Validation failed for field: ${error.field}`);
  } else if (isNotFoundError(error)) {
    console.error(`User not found: ${error.resourceId}`);
  } else {
    console.error('Unknown error:', error);
  }
}
```

## Utility Typesの活用

TypeScriptが提供する組み込みのUtility Typesを活用することで、型定義を簡潔にできます。

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  age: number;
  createdAt: Date;
}

// Partial: すべてのプロパティをオプションに
type UserUpdate = Partial<User>;
// { id?: string; name?: string; email?: string; age?: number; createdAt?: Date }

// Pick: 特定のプロパティのみ選択
type UserSummary = Pick<User, 'id' | 'name'>;
// { id: string; name: string }

// Omit: 特定のプロパティを除外
type UserWithoutDates = Omit<User, 'createdAt'>;
// { id: string; name: string; email: string; age: number }

// Required: すべてのプロパティを必須に
type UserRequired = Required<UserUpdate>;
// { id: string; name: string; email: string; age: number; createdAt: Date }

// Readonly: すべてのプロパティをreadonlyに
type UserReadonly = Readonly<User>;
// { readonly id: string; readonly name: string; ... }

// Record: キーと値の型を指定
type UserRoles = Record<string, 'ADMIN' | 'USER' | 'GUEST'>;
// { [key: string]: 'ADMIN' | 'USER' | 'GUEST' }
```

## TypeScriptレビューチェックリスト

レビュー時に確認すべき項目をチェックリスト形式でまとめます。

- [ ] any型が使用されている場合、具体的な型またはunknown型に置き換えられないか
- [ ] 型推論が効果的に活用されているか（不要な型注釈がないか）
- [ ] ジェネリクスが適切に使用されているか
- [ ] strictNullChecksが有効で、null/undefinedのチェックが適切か
- [ ] Optional ChainingとNullish Coalescingが活用されているか
- [ ] enumよりもunion typeが使用されているか（モダンな実装）
- [ ] readonlyやconst assertionでイミュータビリティが確保されているか
- [ ] 型ガードが適切に実装されているか
- [ ] Utility Typesが効果的に活用されているか
- [ ] エラーハンドリングが型安全に実装されているか
- [ ] インターフェースと型エイリアスが適切に使い分けられているか
- [ ] tsconfig.jsonのstrictモードが有効になっているか

## まとめ

本章では、TypeScript/JavaScriptレビューにおける重要なポイントを解説しました。

主なポイント:
- any型を避け、具体的な型定義を心がける
- ジェネリクスを活用して型安全性と再利用性を両立
- strictNullChecksとOptional Chainingでnull安全性を確保
- enumよりもunion typeを使用するモダンな実装
- readonlyとconst assertionでイミュータビリティを保証
- Utility Typesを活用した型定義の簡潔化

TypeScriptの型システムを最大限に活用することで、バグの早期発見、リファクタリングの安全性向上、コードの自己文書化など、多くの恩恵を受けられます。レビューを通じて、チーム全体の型安全性への意識を高めましょう。

## 参考文献

- [TypeScript Official Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- [TypeScript Deep Dive](https://basarat.gitbook.io/typescript/)
- [Effective TypeScript](https://effectivetypescript.com/)
- [TypeScript Best Practices](https://typescript-eslint.io/rules/)
- [Microsoft TypeScript Coding Guidelines](https://github.com/microsoft/TypeScript/wiki/Coding-guidelines)
- [Google TypeScript Style Guide](https://google.github.io/styleguide/tsguide.html)
