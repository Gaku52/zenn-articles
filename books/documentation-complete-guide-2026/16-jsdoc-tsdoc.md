---
title: "JSDoc/TSDoc/SwiftDoc"
---

# 第16章: JSDoc/TSDoc/SwiftDoc

## この章で学ぶこと

コードを書く際、関数やクラスの使い方を説明するコメントを書くことがあります。しかし、単なる自由形式のコメントではなく、構造化された「ドキュメントコメント」を使うことで、以下のメリットが得られます：

- **エディタが自動補完・ヒント表示できる**（IntelliSense、コード補完）
- **APIリファレンスを自動生成できる**（TypeDoc、SwiftDoc等）
- **型情報を明示的に記述できる**（JavaScriptでも型安全性向上）
- **サンプルコードを関数定義と一緒に管理できる**

この章では以下を学びます：

- ドキュメントコメントとは何か、普通のコメントとの違い
- JSDoc/TSDocの基本的な書き方（タグ、型表記、例）
- SwiftDocの基本的な書き方（Markdownベース、パラメータ記法）
- APIリファレンス自動生成ツールの活用方法
- ドキュメントコメントを書くべき場所、書かなくてよい場所
- 実践的なドキュメントコメントのパターン集

**前提知識:** 第15章「コメントのベストプラクティス」の理解（Whyを説明する原則）

---

## ドキュメントコメントとは

### 普通のコメントとの違い

**普通のコメント**は、コードを読む人への補足説明です：

```typescript
// ユーザーIDを取得（普通のコメント）
const userId = req.params.id;

/*
 * 複数行のコメント
 * これも普通のコメント
 */
```

**ドキュメントコメント**は、特定のフォーマットに従って書かれ、ツールが自動的に処理できるコメントです：

```typescript
/**
 * ユーザーを作成します
 *
 * @param name - ユーザー名（1-50文字）
 * @param email - メールアドレス
 * @returns 作成されたユーザー
 */
function createUser(name: string, email: string): User {
  // 実装
}
```

### ドキュメントコメントの目的

ドキュメントコメントは以下の目的で使われます：

1. **API利用者への情報提供**
   - 関数・クラスの使い方
   - パラメータの意味と制約
   - 戻り値の説明
   - 例外・エラーの情報

2. **開発ツールとの統合**
   - VS Code、IntelliJ等のエディタがヒント表示
   - TypeScript型推論の補助
   - リファクタリング時の情報源

3. **自動ドキュメント生成**
   - APIリファレンスの自動生成
   - Webサイト形式でのドキュメント公開
   - 常に最新の情報を保持

### いつドキュメントコメントを書くか

以下の場合にドキュメントコメントを書きます：

- **公開API**（ライブラリ、フレームワーク、SDKの関数・クラス）
- **チーム内で共有される関数・クラス**
- **複雑な引数・戻り値を持つ関数**
- **エラー処理が重要な関数**

以下の場合は不要です：

- **privateな内部実装**（実装の詳細は普通のコメントで十分）
- **自明な関数**（`getUsername()` のような説明不要なもの）
- **型だけで十分に説明できる場合**（TypeScriptの型が完璧な場合）

---

## JSDoc/TSDocの基本

### JSDocとTSDocの違い

**JSDoc**は、JavaScript向けのドキュメントコメント標準です（2011年頃から）。

**TSDoc**は、TypeScript向けにMicrosoftが策定した標準です（2019年～）。JSDocをベースにしていますが、TypeScript特有の型情報との整合性を重視しています。

実務上、以下のように使い分けます：

- **TypeScriptプロジェクト**: TSDocを推奨（型情報はTypeScriptで書き、説明のみコメントに）
- **JavaScriptプロジェクト**: JSDocを使用（型情報もコメントで明示）

### 基本的な書き方

ドキュメントコメントは `/**` で始め、`*/` で終わります：

```typescript
/**
 * 関数の概要を一行で書きます
 *
 * 詳細説明が必要なら、空行の後に複数行で書けます。
 * この部分では、関数の目的や背景を説明します。
 *
 * @param name - パラメータの説明
 * @returns 戻り値の説明
 */
function greet(name: string): string {
  return `Hello, ${name}!`;
}
```

**構造:**

1. **概要**（Summary）: 最初の1行で関数の目的を簡潔に
2. **詳細**（Description）: 必要に応じて空行後に複数行で
3. **タグ**（Tags）: `@param`, `@returns` 等で構造化情報

### 主要なタグ一覧

#### @param - パラメータの説明

```typescript
/**
 * ユーザーを検索します
 *
 * @param query - 検索クエリ（部分一致）
 * @param limit - 取得件数（デフォルト: 10）
 * @param offset - オフセット（デフォルト: 0）
 */
function searchUsers(
  query: string,
  limit: number = 10,
  offset: number = 0
): User[] {
  // 実装
}
```

**書き方:**

- `@param パラメータ名 - 説明`
- 説明では、制約・デフォルト値・単位などを明記
- オプショナルパラメータは `@param [name] - 説明` とも書ける（JSDocスタイル）

#### @returns - 戻り値の説明

```typescript
/**
 * ユーザーIDからユーザー情報を取得します
 *
 * @param userId - ユーザーID
 * @returns ユーザー情報。見つからない場合はnull
 */
function getUserById(userId: string): User | null {
  // 実装
}
```

**書き方:**

- `@returns 説明`
- 戻り値の意味、nullの場合の条件、特殊なケースを説明
- TypeScriptでは型情報は関数定義で明示されるため、説明は「意味」に集中

#### @throws - 例外・エラーの説明

```typescript
/**
 * ユーザーを作成します
 *
 * @param name - ユーザー名
 * @param email - メールアドレス
 * @returns 作成されたユーザー
 * @throws {ValidationError} バリデーションエラー（名前が空、メールが不正）
 * @throws {DuplicateError} 同じメールアドレスが既に存在
 */
async function createUser(name: string, email: string): Promise<User> {
  // 実装
}
```

**書き方:**

- `@throws {エラー型} 説明`
- どういう条件で例外が発生するか明記
- 複数の例外がある場合は複数の `@throws` を書く

#### @example - サンプルコード

```typescript
/**
 * 配列から重複を除去します
 *
 * @param items - 元の配列
 * @returns 重複のない配列
 *
 * @example
 * ```typescript
 * const numbers = [1, 2, 2, 3, 3, 3];
 * const unique = removeDuplicates(numbers);
 * console.log(unique); // [1, 2, 3]
 * ```
 *
 * @example
 * オブジェクトの配列にも使えます:
 * ```typescript
 * const users = [
 *   { id: 1, name: 'Alice' },
 *   { id: 1, name: 'Alice' },
 *   { id: 2, name: 'Bob' }
 * ];
 * const uniqueUsers = removeDuplicates(users);
 * // [{ id: 1, name: 'Alice' }, { id: 2, name: 'Bob' }]
 * ```
 */
function removeDuplicates<T>(items: T[]): T[] {
  return Array.from(new Set(items));
}
```

**書き方:**

- `@example` の後に自由形式でコード・説明を書く
- コードブロック（` ```typescript ` ～ ` ``` `）を使うと見やすい
- 複数の `@example` を書いて、様々なユースケースを示せる

#### @deprecated - 非推奨の表示

```typescript
/**
 * ユーザーを取得します
 *
 * @deprecated バージョン2.0で削除予定。代わりに `getUserById()` を使ってください
 * @param id - ユーザーID
 * @returns ユーザー情報
 */
function getUser(id: string): User {
  // 実装
}
```

**書き方:**

- `@deprecated 理由と代替方法`
- エディタが警告を表示してくれる
- いつ削除されるか、何を使うべきかを明記

#### @see - 関連情報へのリンク

```typescript
/**
 * ユーザーを更新します
 *
 * @param userId - ユーザーID
 * @param updates - 更新内容
 * @see {@link createUser} - ユーザー作成
 * @see {@link deleteUser} - ユーザー削除
 * @see https://docs.example.com/api/users - API仕様書
 */
function updateUser(userId: string, updates: Partial<User>): Promise<User> {
  // 実装
}
```

**書き方:**

- `@see {@link 関数名}` で他の関数へのリンク
- `@see URL` で外部ドキュメントへのリンク

#### その他の便利なタグ

```typescript
/**
 * @internal - 内部実装用（公開APIでない）
 */

/**
 * @alpha - アルファ版（APIが変わる可能性大）
 */

/**
 * @beta - ベータ版（APIがほぼ安定）
 */

/**
 * @public - 公開API
 */

/**
 * @private - プライベート（使用禁止）
 */
```

### 型の書き方（JSDocの場合）

TypeScriptを使わない場合、JSDocで型を明示できます：

```javascript
/**
 * ユーザーを作成します
 *
 * @param {string} name - ユーザー名
 * @param {string} email - メールアドレス
 * @param {Object} options - オプション設定
 * @param {number} [options.age] - 年齢（オプショナル）
 * @param {string[]} [options.tags] - タグ配列（オプショナル）
 * @returns {Promise<User>} 作成されたユーザー
 */
async function createUser(name, email, options = {}) {
  // 実装
}
```

**型の記法:**

- `{型名}` で型を指定
- `{string[]}` で配列
- `{Object}` でオブジェクト
- `{Type1|Type2}` でユニオン型
- `{?Type}` でnull許容型
- `{!Type}` でnull非許容型

**TypeScriptでは型情報を二重に書かない:**

```typescript
// ❌ 悪い例: 型情報の重複
/**
 * @param {string} name - ユーザー名
 * @returns {User} ユーザー
 */
function createUser(name: string): User {
  // TypeScriptで型が明示されているのに、コメントでも書いている
}

// ✅ 良い例: 説明だけ書く
/**
 * 新しいユーザーを作成します
 *
 * @param name - ユーザー名（1-50文字）
 * @returns 作成されたユーザー
 */
function createUser(name: string): User {
  // TypeScriptの型情報は関数定義で十分
  // コメントは「意味」「制約」「背景」を説明
}
```

---

## TSDocの実践

### TypeScriptでの推奨スタイル

TypeScriptでは以下の原則に従います：

1. **型情報は関数定義で表現**（コメントに書かない）
2. **説明は意味・制約・背景に集中**
3. **TSDoc標準タグを使う**

#### 基本パターン

```typescript
/**
 * 指定された条件でユーザーを検索します
 *
 * 検索はメールアドレスと名前の部分一致で行われます。
 * 検索結果は作成日時の降順でソートされます。
 *
 * @param query - 検索クエリ（空文字列の場合は全件取得）
 * @param options - 検索オプション
 * @returns 検索結果のユーザー配列
 * @throws {ValidationError} クエリが不正な形式
 *
 * @example
 * ```typescript
 * // 名前で検索
 * const users = await searchUsers('Alice', { limit: 10 });
 *
 * // メールアドレスで検索
 * const users = await searchUsers('example.com', { limit: 5 });
 * ```
 */
export async function searchUsers(
  query: string,
  options: SearchOptions = {}
): Promise<User[]> {
  // 実装
}

interface SearchOptions {
  /** 取得件数の上限（デフォルト: 20） */
  limit?: number;
  /** オフセット（デフォルト: 0） */
  offset?: number;
  /** ソート順（デフォルト: 'desc'） */
  sortOrder?: 'asc' | 'desc';
}
```

**ポイント:**

- 概要は具体的に（「検索します」だけでなく、どう検索するか）
- デフォルト値・動作を明記
- サンプルコードで実際の使い方を示す

#### クラスのドキュメント

```typescript
/**
 * ユーザー管理サービス
 *
 * ユーザーのCRUD操作と検索機能を提供します。
 * すべての操作はトランザクション内で実行され、
 * エラー時は自動的にロールバックされます。
 *
 * @example
 * ```typescript
 * const service = new UserService(database);
 *
 * // ユーザー作成
 * const user = await service.create({
 *   name: 'Alice',
 *   email: 'alice@example.com'
 * });
 *
 * // ユーザー検索
 * const users = await service.search('alice');
 * ```
 */
export class UserService {
  /**
   * UserServiceを初期化します
   *
   * @param db - データベース接続
   * @param logger - ロガー（オプショナル）
   */
  constructor(
    private db: Database,
    private logger?: Logger
  ) {}

  /**
   * 新しいユーザーを作成します
   *
   * メールアドレスの重複チェックが自動的に行われます。
   *
   * @param data - ユーザーデータ
   * @returns 作成されたユーザー
   * @throws {DuplicateError} メールアドレスが既に存在
   * @throws {ValidationError} データが不正
   */
  async create(data: CreateUserData): Promise<User> {
    // 実装
  }

  /**
   * ユーザーを検索します
   *
   * @param query - 検索クエリ
   * @returns 検索結果
   */
  async search(query: string): Promise<User[]> {
    // 実装
  }
}
```

**ポイント:**

- クラス全体の目的・責務を説明
- コンストラクタの引数を説明
- 各メソッドは独立して理解できるように

#### 型エイリアスのドキュメント

```typescript
/**
 * ユーザーのロール
 *
 * - `admin`: すべての操作が可能
 * - `editor`: コンテンツの編集が可能
 * - `viewer`: 閲覧のみ可能
 */
export type UserRole = 'admin' | 'editor' | 'viewer';

/**
 * ページネーション情報
 */
export interface Pagination {
  /** 現在のページ（1始まり） */
  page: number;
  /** 1ページあたりの件数 */
  perPage: number;
  /** 総件数 */
  total: number;
  /** 総ページ数 */
  totalPages: number;
}

/**
 * API レスポンスの共通構造
 *
 * @typeParam T - データの型
 */
export interface ApiResponse<T> {
  /** レスポンスデータ */
  data: T;
  /** ページネーション情報（リスト取得時のみ） */
  pagination?: Pagination;
  /** エラー情報（エラー時のみ） */
  error?: {
    /** エラーコード */
    code: string;
    /** エラーメッセージ */
    message: string;
  };
}
```

**ポイント:**

- 型の目的・用途を説明
- 各プロパティの意味を明記
- 選択肢がある場合はそれぞれの意味を説明

---

## SwiftDocの基本

### SwiftDocの特徴

SwiftDocは、Swiftのドキュメントコメント形式です。主な特徴：

- **Markdownベース**（見出し、リスト、リンクが使える）
- **簡潔な記法**（JSDocより記号が少ない）
- **Xcode統合**（Option+クリックでドキュメント表示）

### 基本的な書き方

```swift
/**
 ユーザーを作成します

 新しいユーザーをデータベースに登録します。
 メールアドレスの重複チェックが自動的に行われます。

 - Parameters:
   - name: ユーザー名（1-50文字）
   - email: メールアドレス
 - Returns: 作成されたユーザー
 - Throws: `ValidationError` バリデーションエラー
 - Throws: `DuplicateError` メールアドレスが既に存在
 */
func createUser(name: String, email: String) throws -> User {
    // 実装
}
```

**構造:**

- `/**` ～ `*/` で囲む
- 最初の段落が概要
- 空行の後に詳細説明
- ハイフン（`-`）で始まるキーワード（Parameters, Returns, Throws等）

### 主要なキーワード

#### Parameters - パラメータの説明

```swift
/**
 ユーザーを検索します

 - Parameters:
   - query: 検索クエリ（部分一致）
   - limit: 取得件数（デフォルト: 10）
   - offset: オフセット（デフォルト: 0）
 - Returns: 検索結果のユーザー配列
 */
func searchUsers(
    query: String,
    limit: Int = 10,
    offset: Int = 0
) -> [User] {
    // 実装
}
```

**パラメータが1つだけの場合:**

```swift
/**
 ユーザーIDからユーザーを取得します

 - Parameter id: ユーザーID
 - Returns: ユーザー情報（見つからない場合はnil）
 */
func getUser(id: String) -> User? {
    // 実装
}
```

#### Returns - 戻り値の説明

```swift
/**
 配列から重複を除去します

 - Parameter items: 元の配列
 - Returns: 重複のない配列（順序は保持されない）
 */
func removeDuplicates<T: Hashable>(_ items: [T]) -> [T] {
    return Array(Set(items))
}
```

#### Throws - 例外の説明

```swift
/**
 JSONファイルを読み込んでパースします

 - Parameter path: ファイルパス
 - Returns: パースされたデータ
 - Throws: `FileNotFoundError` ファイルが見つからない
 - Throws: `ParseError` JSON形式が不正
 */
func loadJSON<T: Decodable>(from path: String) throws -> T {
    // 実装
}
```

### Markdown記法の活用

SwiftDocではMarkdownが使えます：

```swift
/**
 ユーザー管理サービス

 # 概要

 ユーザーのCRUD操作を提供します。

 # 使い方

 ```swift
 let service = UserService(database: db)
 let user = try await service.createUser(
     name: "Alice",
     email: "alice@example.com"
 )
 ```

 # 重要な注意点

 - すべての操作は非同期です
 - エラーハンドリングが必要です
 - トランザクションは自動的に管理されます

 - SeeAlso: `User`, `Database`
 */
class UserService {
    // 実装
}
```

**使える記法:**

- `# 見出し`, `## サブ見出し`
- `- リスト項目`
- `` `コード` ``
- ` ```swift ` ～ ` ``` ` コードブロック
- `**太字**`, `*斜体*`
- `[リンク](URL)`

### プロパティのドキュメント

```swift
/**
 ユーザー情報
 */
struct User {
    /// ユーザーID（UUID形式）
    let id: String

    /// ユーザー名（1-50文字）
    let name: String

    /// メールアドレス（一意制約）
    let email: String

    /// ユーザーのロール
    ///
    /// - `admin`: すべての操作が可能
    /// - `editor`: コンテンツの編集が可能
    /// - `viewer`: 閲覧のみ可能
    let role: UserRole

    /// 作成日時
    let createdAt: Date
}
```

**ポイント:**

- 簡単なプロパティは `///` で1行コメント
- 詳細説明が必要なら `/** */` で複数行
- 列挙型の場合は各ケースの意味を説明

### Xcodeでの表示

SwiftDocのコメントは、Xcodeで以下のように表示されます：

1. **Option + クリック**: ポップアップでドキュメント表示
2. **Quick Help**: サイドバーに常時表示
3. **自動補完**: 関数入力中にドキュメントがヒント表示

これにより、開発者はドキュメントを別途開かなくても、コード内で即座に情報を得られます。

---

## APIリファレンスの自動生成

### TypeDoc（TypeScript）

TypeDocは、TypeScriptコードからHTMLドキュメントを生成します。

#### インストール

```bash
npm install --save-dev typedoc
```

#### 設定ファイル（typedoc.json）

```json
{
  "entryPoints": ["src/index.ts"],
  "out": "docs",
  "exclude": ["**/*.test.ts", "**/*.spec.ts"],
  "excludePrivate": true,
  "excludeInternal": true,
  "includeVersion": true,
  "sort": ["source-order"],
  "plugin": ["typedoc-plugin-markdown"]
}
```

#### 生成コマンド

```bash
npx typedoc
```

生成されたドキュメントは `docs/` フォルダに出力されます。

#### package.jsonへの追加

```json
{
  "scripts": {
    "docs": "typedoc",
    "docs:watch": "typedoc --watch"
  }
}
```

#### 実例（想定される出力）

TypeDocは以下のような構造のHTMLを生成します：

- **トップページ**: プロジェクト概要、モジュール一覧
- **クラスページ**: コンストラクタ、メソッド一覧、プロパティ一覧
- **関数ページ**: シグネチャ、パラメータ、戻り値、例
- **型ページ**: 型定義、プロパティの説明

例えば、以下のコード：

```typescript
/**
 * ユーザー管理サービス
 *
 * @example
 * ```typescript
 * const service = new UserService(db);
 * const user = await service.createUser('Alice', 'alice@example.com');
 * ```
 */
export class UserService {
  /**
   * ユーザーを作成します
   *
   * @param name - ユーザー名
   * @param email - メールアドレス
   * @returns 作成されたユーザー
   */
  async createUser(name: string, email: string): Promise<User> {
    // 実装
  }
}
```

から、以下のようなHTMLドキュメントが生成されます：

```
UserService
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ユーザー管理サービス

Example:
  const service = new UserService(db);
  const user = await service.createUser('Alice', 'alice@example.com');

Constructor
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

new UserService(db: Database)

Methods
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

createUser
  ユーザーを作成します

  Parameters:
    name: string - ユーザー名
    email: string - メールアドレス

  Returns: Promise<User> - 作成されたユーザー
```

### JSDoc（JavaScript）

JavaScriptの場合も同様にJSDocツールを使えます。

#### インストール

```bash
npm install --save-dev jsdoc
```

#### 設定ファイル（jsdoc.json）

```json
{
  "source": {
    "include": ["src"],
    "includePattern": ".+\\.js$",
    "excludePattern": "(node_modules|docs)"
  },
  "opts": {
    "destination": "./docs",
    "recurse": true
  },
  "plugins": ["plugins/markdown"],
  "templates": {
    "default": {
      "staticFiles": {
        "include": ["./static"]
      }
    }
  }
}
```

#### 生成コマンド

```bash
npx jsdoc -c jsdoc.json
```

### Swift DocC（Swift）

SwiftのドキュメントはXcodeの「Product > Build Documentation」で生成されます。

#### DocCカタログの作成

1. Xcodeでプロジェクトを開く
2. File > New > File > Documentation Catalog
3. 名前を設定（例: `MyLibrary.docc`）

#### カタログ内の構造

```
MyLibrary.docc/
├── MyLibrary.md           # トップページ
├── GettingStarted.md      # 入門ガイド
├── UserService.md         # 詳細ガイド
└── Resources/             # 画像等
    └── hero.png
```

#### トップページの例（MyLibrary.md）

```markdown
# ``MyLibrary``

ユーザー管理のための包括的なライブラリ

## Overview

MyLibraryは、ユーザーのCRUD操作、検索、認証を提供します。

## Topics

### Essentials

- <doc:GettingStarted>
- ``UserService``
- ``User``

### User Management

- ``UserService/createUser(name:email:)``
- ``UserService/getUser(id:)``
- ``UserService/searchUsers(query:limit:)``

### Authentication

- ``AuthService``
- ``Token``
```

#### 生成とプレビュー

```bash
# コマンドラインで生成
xcodebuild docbuild \
  -scheme MyLibrary \
  -derivedDataPath ./build

# プレビュー（ブラウザで開く）
open ./build/Build/Products/Debug/MyLibrary.doccarchive
```

生成されたドキュメントは `.doccarchive` ファイルとして保存され、Xcodeで閲覧できます。また、ホスティングサービス（GitHub Pages等）で公開も可能です。

---

## 実践的なパターン集

### パターン1: 複雑な引数を持つ関数

```typescript
/**
 * データベースクエリを実行します
 *
 * 複雑な検索条件、ソート、ページネーションをサポートします。
 * SQLインジェクション対策として、すべての値はプレースホルダーで渡されます。
 *
 * @param table - テーブル名
 * @param options - クエリオプション
 * @returns クエリ結果
 *
 * @example
 * 基本的な検索:
 * ```typescript
 * const users = await query('users', {
 *   where: { status: 'active' },
 *   limit: 10
 * });
 * ```
 *
 * @example
 * 複雑な検索条件:
 * ```typescript
 * const users = await query('users', {
 *   where: {
 *     age: { gte: 18, lte: 65 },
 *     status: { in: ['active', 'pending'] }
 *   },
 *   orderBy: { createdAt: 'desc' },
 *   skip: 20,
 *   take: 10
 * });
 * ```
 */
export async function query<T>(
  table: string,
  options: QueryOptions
): Promise<T[]> {
  // 実装
}

interface QueryOptions {
  /**
   * WHERE条件
   *
   * @example
   * ```typescript
   * { status: 'active', age: { gte: 18 } }
   * ```
   */
  where?: Record<string, any>;

  /**
   * ソート順
   *
   * @example
   * ```typescript
   * { createdAt: 'desc', name: 'asc' }
   * ```
   */
  orderBy?: Record<string, 'asc' | 'desc'>;

  /** スキップ件数（ページネーション用） */
  skip?: number;

  /** 取得件数（デフォルト: 20、最大: 100） */
  take?: number;
}
```

**ポイント:**

- オプションオブジェクトの各プロパティを詳細に説明
- 複数の `@example` で様々なユースケースを提示
- 制約（最大値等）を明記

### パターン2: エラーハンドリングが重要な関数

```typescript
/**
 * ファイルをアップロードします
 *
 * ファイルサイズ・形式のバリデーション、ウイルススキャン、
 * S3へのアップロードを順次実行します。
 *
 * @param file - アップロードするファイル
 * @param options - アップロードオプション
 * @returns アップロードされたファイルの情報
 *
 * @throws {FileSizeError} ファイルサイズが上限（10MB）を超えている
 * @throws {FileTypeError} 許可されていないファイル形式
 * @throws {VirusScanError} ウイルスが検出された
 * @throws {StorageError} ストレージへのアップロードに失敗
 *
 * @example
 * ```typescript
 * try {
 *   const fileInfo = await uploadFile(file, {
 *     folder: 'avatars',
 *     allowedTypes: ['image/jpeg', 'image/png']
 *   });
 *   console.log('Uploaded:', fileInfo.url);
 * } catch (error) {
 *   if (error instanceof FileSizeError) {
 *     alert('ファイルサイズが大きすぎます');
 *   } else if (error instanceof FileTypeError) {
 *     alert('この形式のファイルはアップロードできません');
 *   } else {
 *     alert('アップロードに失敗しました');
 *   }
 * }
 * ```
 */
export async function uploadFile(
  file: File,
  options: UploadOptions
): Promise<FileInfo> {
  // 実装
}
```

**ポイント:**

- すべての例外を `@throws` で明示
- 例外が発生する条件を具体的に説明
- サンプルコードでエラーハンドリングのパターンを示す

### パターン3: ジェネリック関数

```typescript
/**
 * APIレスポンスをラップします
 *
 * 成功時はデータを、エラー時はエラー情報を含む
 * 統一的なレスポンス形式を生成します。
 *
 * @typeParam T - レスポンスデータの型
 * @param data - レスポンスデータ
 * @param options - レスポンスオプション
 * @returns API レスポンス
 *
 * @example
 * 成功レスポンス:
 * ```typescript
 * const response = wrapResponse<User[]>(users, {
 *   pagination: {
 *     page: 1,
 *     perPage: 10,
 *     total: 50
 *   }
 * });
 * // { data: [...], pagination: {...} }
 * ```
 *
 * @example
 * エラーレスポンス:
 * ```typescript
 * const response = wrapResponse<null>(null, {
 *   error: {
 *     code: 'NOT_FOUND',
 *     message: 'User not found'
 *   }
 * });
 * // { data: null, error: {...} }
 * ```
 */
export function wrapResponse<T>(
  data: T,
  options: ResponseOptions = {}
): ApiResponse<T> {
  // 実装
}
```

**ポイント:**

- `@typeParam` でジェネリック型パラメータを説明
- 型パラメータの使い方を例で示す

### パターン4: 非同期・Promise

```typescript
/**
 * 複数のユーザーを並列で取得します
 *
 * Promise.allを使用して、すべてのユーザーを同時に取得します。
 * 一つでも取得に失敗した場合、全体が失敗します。
 *
 * @param userIds - ユーザーIDの配列
 * @returns ユーザー配列のPromise
 * @throws {NotFoundError} いずれかのユーザーが見つからない
 *
 * @example
 * ```typescript
 * const users = await fetchUsers(['user1', 'user2', 'user3']);
 * console.log(`Fetched ${users.length} users`);
 * ```
 *
 * @see {@link fetchUsersAllSettled} - 失敗を許容するバージョン
 */
export async function fetchUsers(userIds: string[]): Promise<User[]> {
  return Promise.all(userIds.map(id => getUserById(id)));
}

/**
 * 複数のユーザーを並列で取得します（失敗を許容）
 *
 * Promise.allSettledを使用して、失敗したユーザーがあっても
 * 取得できたユーザーは返します。
 *
 * @param userIds - ユーザーIDの配列
 * @returns 取得結果の配列
 *
 * @example
 * ```typescript
 * const results = await fetchUsersAllSettled(['user1', 'invalid', 'user3']);
 * const users = results
 *   .filter(r => r.status === 'fulfilled')
 *   .map(r => r.value);
 * const errors = results
 *   .filter(r => r.status === 'rejected')
 *   .map(r => r.reason);
 * ```
 *
 * @see {@link fetchUsers} - 失敗を許容しないバージョン
 */
export async function fetchUsersAllSettled(
  userIds: string[]
): Promise<PromiseSettledResult<User>[]> {
  return Promise.allSettled(userIds.map(id => getUserById(id)));
}
```

**ポイント:**

- 非同期処理の振る舞いを明記（並列実行、失敗時の挙動）
- 関連する関数との違いを `@see` で明示

### パターン5: Swiftのクロージャ

```swift
/**
 ユーザーを非同期で取得します

 ネットワークリクエストを実行し、結果をコールバックで返します。

 - Parameters:
   - userId: ユーザーID
   - completion: 完了ハンドラー
     - user: 取得されたユーザー（失敗時はnil）
     - error: エラー情報（成功時はnil）

 - Important: completionはメインスレッドで呼ばれます

 - Note: この関数は非推奨です。async/await版の `getUser(userId:)` を使ってください
 */
func getUser(
    userId: String,
    completion: @escaping (User?, Error?) -> Void
) {
    // 実装
}

/**
 ユーザーを非同期で取得します（async/await版）

 - Parameter userId: ユーザーID
 - Returns: 取得されたユーザー
 - Throws: ネットワークエラーまたはパースエラー

 - Example:
 ```swift
 do {
     let user = try await getUser(userId: "user123")
     print("User: \(user.name)")
 } catch {
     print("Error: \(error)")
 }
 ```
 */
func getUser(userId: String) async throws -> User {
    // 実装
}
```

**ポイント:**

- `- Important:` でスレッドや重要な注意事項を強調
- `- Note:` で補足情報や非推奨の案内
- 新旧API両方を文書化し、移行を促す

---

## ドキュメントコメントのメンテナンス

### コードとドキュメントの同期

ドキュメントコメントの最大の課題は、**コードが変わったときにコメントも更新すること**です。

#### よくある問題

```typescript
// ❌ コードとコメントが合っていない
/**
 * ユーザーを作成します
 *
 * @param name - ユーザー名
 * @param email - メールアドレス
 * @returns 作成されたユーザー
 */
async function createUser(
  name: string,
  email: string,
  role: UserRole  // roleパラメータが追加されたが、コメントが更新されていない
): Promise<User> {
  // 実装
}
```

#### 対策

1. **コード変更時にコメントも変更する習慣**
   - PRレビューでコメントの更新を確認
   - リファクタリング時は必ずドキュメントも見直す

2. **ESLintルールで検証**（TypeScript）

```json
{
  "plugins": ["jsdoc"],
  "rules": {
    "jsdoc/check-param-names": "error",
    "jsdoc/require-param": "error",
    "jsdoc/require-returns": "error"
  }
}
```

3. **テストで暗黙的に検証**
   - サンプルコードをそのままテストとして実行
   - ドキュメントが古いとテストが失敗する

```typescript
// サンプルコードをそのままテストに
describe('createUser example', () => {
  it('should work as documented', async () => {
    // ドキュメントの @example をコピペ
    const user = await createUser('John', 'john@example.com', 'viewer');
    expect(user.name).toBe('John');
  });
});
```

### ドキュメントレビューのチェックリスト

PRレビュー時に以下を確認します：

- [ ] 公開APIにドキュメントコメントがある
- [ ] すべてのパラメータが `@param` で説明されている
- [ ] 戻り値が `@returns` で説明されている
- [ ] 例外が `@throws` で説明されている
- [ ] サンプルコードが動作する（実行可能か確認）
- [ ] コードの変更に合わせてコメントも更新されている
- [ ] タイポや文法ミスがない

---

## よくある間違いと改善例

### 間違い1: What（何をするか）だけ書く

```typescript
// ❌ 悪い例: コードを見れば分かる
/**
 * ユーザーを取得します
 *
 * @param id - ユーザーID
 * @returns ユーザー
 */
function getUser(id: string): User {
  // 実装
}
```

**問題点:**

- コードを見れば分かることしか書いていない
- 「取得」の詳細が不明（キャッシュ？DB？API？）
- エラー時の挙動が不明

```typescript
// ✅ 良い例: 詳細・制約・背景を説明
/**
 * ユーザーをキャッシュから取得します
 *
 * キャッシュにない場合は、データベースから取得し、
 * 5分間キャッシュに保存します。
 *
 * @param id - ユーザーID（UUID形式）
 * @returns ユーザー情報。見つからない場合はnull
 *
 * @example
 * ```typescript
 * const user = getUser('550e8400-e29b-41d4-a716-446655440000');
 * if (user) {
 *   console.log(user.name);
 * } else {
 *   console.log('User not found');
 * }
 * ```
 */
function getUser(id: string): User | null {
  // 実装
}
```

### 間違い2: 型情報の重複（TypeScript）

```typescript
// ❌ 悪い例: TypeScriptで型が明示されているのに、コメントにも書く
/**
 * ユーザーを検索します
 *
 * @param {string} query - 検索クエリ
 * @param {number} limit - 取得件数
 * @returns {Promise<User[]>} ユーザー配列のPromise
 */
async function searchUsers(query: string, limit: number): Promise<User[]> {
  // 実装
}
```

**問題点:**

- 型情報が関数定義とコメントで二重管理
- コードを変更したときにコメントの更新を忘れやすい

```typescript
// ✅ 良い例: 説明だけ書く
/**
 * ユーザーを名前・メールアドレスで検索します
 *
 * @param query - 検索クエリ（部分一致）
 * @param limit - 取得件数（最大100）
 * @returns 検索結果の配列（該当なしの場合は空配列）
 */
async function searchUsers(query: string, limit: number): Promise<User[]> {
  // 実装
}
```

### 間違い3: サンプルコードが動かない

```typescript
// ❌ 悪い例: 実際には動かないコード
/**
 * ユーザーを作成します
 *
 * @example
 * ```typescript
 * const user = createUser('John', 'john@example.com');
 * ```
 */
async function createUser(name: string, email: string): Promise<User> {
  // 実装
}
```

**問題点:**

- 非同期関数なのに `await` がない
- このコードを実行するとPromiseが返ってくるだけで、実際のUserは得られない

```typescript
// ✅ 良い例: 実際に動くコード
/**
 * ユーザーを作成します
 *
 * @example
 * ```typescript
 * const user = await createUser('John', 'john@example.com');
 * console.log(user.id); // user_xxxxx
 * ```
 */
async function createUser(name: string, email: string): Promise<User> {
  // 実装
}
```

### 間違い4: 過剰なドキュメント

```typescript
// ❌ 悪い例: 自明なことまで説明
/**
 * 数値を2倍にします
 *
 * この関数は、引数として受け取った数値を2倍にして返します。
 * 内部的には、引数に2を掛け算しています。
 *
 * @param value - 2倍にする数値
 * @returns 2倍された数値
 *
 * @example
 * ```typescript
 * const result = double(5);
 * console.log(result); // 10
 * ```
 */
function double(value: number): number {
  return value * 2;
}
```

**問題点:**

- 関数名と実装で十分に説明できている
- 過剰な説明は逆に読みにくい

```typescript
// ✅ 良い例: ドキュメントコメント不要
function double(value: number): number {
  return value * 2;
}

// もし特殊な背景があるなら簡潔に
/**
 * 数値を2倍にします
 *
 * Note: この関数は旧APIとの互換性のために残されています。
 * 新しいコードでは `value * 2` を直接書いてください。
 *
 * @deprecated v3.0で削除予定
 */
function double(value: number): number {
  return value * 2;
}
```

---

## 実務での運用戦略

### いつドキュメントコメントを書くか

以下のタイミングでドキュメントコメントを書きます：

1. **関数を設計する時**
   - インターフェースを考える段階で、使い方を文書化
   - ドキュメントを書くことで、APIの使いやすさを検証

2. **公開APIを作る時**
   - ライブラリ・SDK・共有モジュールの関数
   - 最初から完璧に書く

3. **バグ修正・リファクタリング後**
   - 仕様が変わったらすぐに更新
   - 特にパラメータや戻り値の変更時

4. **レビュー指摘があった時**
   - 「この関数の使い方が分からない」と指摘されたら追加

### ドキュメントコメントを書かなくてよい場合

以下の場合はドキュメントコメント不要です：

- **privateな内部関数**（チーム内で使わない）
- **自明な関数**（名前と型で十分に説明できる）
- **テストコード**（テスト自体がドキュメント）

### チームでのルール策定

プロジェクトごとに以下を決めます：

1. **どのスコープでドキュメントコメントを書くか**
   - 公開APIのみ？
   - チーム内共有関数も？
   - すべての関数？

2. **必須タグの定義**
   - `@param` は必須
   - `@returns` は必須
   - `@example` はあると嬉しい

3. **レビュー基準**
   - PRチェックリストに追加
   - 自動チェック（ESLint）を導入

4. **自動生成の頻度**
   - CIで毎回生成？
   - リリース時のみ？
   - 手動？

### CIでの自動チェック

#### ESLint設定（TypeScript）

```json
{
  "plugins": ["jsdoc"],
  "rules": {
    "jsdoc/check-alignment": "warn",
    "jsdoc/check-param-names": "error",
    "jsdoc/check-tag-names": "error",
    "jsdoc/require-param": "warn",
    "jsdoc/require-param-description": "warn",
    "jsdoc/require-returns": "warn",
    "jsdoc/require-returns-description": "warn"
  }
}
```

#### GitHub Actionsでドキュメント生成

```yaml
name: Generate Docs

on:
  push:
    branches: [main]

jobs:
  docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm install
      - run: npm run docs
      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./docs
```

これにより、mainブランチにマージされるたびに、最新のAPIリファレンスが自動的に生成・公開されます。

---

## チェックリスト

この章で学んだことを確認しましょう：

### 基本理解

- [ ] ドキュメントコメントと普通のコメントの違いを理解している
- [ ] JSDoc/TSDoc/SwiftDocの基本的な書き方を知っている
- [ ] ドキュメントコメントを書くべき場所、書かなくてよい場所を判断できる

### タグの活用

- [ ] `@param`, `@returns`, `@throws` を適切に使える
- [ ] `@example` で実際に動くコードを書ける
- [ ] `@deprecated`, `@see` で関連情報を提供できる

### 実践

- [ ] TypeScriptで型情報を二重に書かない原則を理解している
- [ ] APIリファレンス生成ツール（TypeDoc等）を使える
- [ ] コード変更時にドキュメントも更新する習慣がある
- [ ] ドキュメントコメントをレビュー基準に含められる

### 誠実性（第2章の原則）

- [ ] サンプルコードが実際に動作することを確認している
- [ ] 架空の使い方を「実例」として提示していない
- [ ] 不確かな情報を断定的に書いていない

---

## 次のステップ

この章では、JSDoc/TSDoc/SwiftDocを使った構造化されたドキュメントコメントの書き方を学びました。次の第17章「ドキュメント管理戦略」では、以下を学びます：

- ドキュメントの配置場所（コードと同じリポジトリ？別リポジトリ？）
- バージョン管理の方法
- ドキュメントの種類別の管理戦略
- 更新責任の明確化

ドキュメントコメントで「コードレベル」のドキュメントを充実させた後は、プロジェクト全体のドキュメント管理戦略を学びましょう。

### 関連リソース

**公式ドキュメント:**

- JSDoc: https://jsdoc.app/
- TSDoc: https://tsdoc.org/
- TypeDoc: https://typedoc.org/
- Swift Documentation: https://www.swift.org/documentation/docc/

**ツール:**

- TypeDoc（TypeScript APIリファレンス生成）
- JSDoc（JavaScript APIリファレンス生成）
- ESLint JSDocプラグイン（自動チェック）
- Swift DocC（Swift ドキュメント生成）

**参考記事:**

- TypeScript公式「JSDocリファレンス」: https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html
- Apple「Writing Symbol Documentation」: https://developer.apple.com/documentation/xcode/writing-symbol-documentation-in-your-source-files
