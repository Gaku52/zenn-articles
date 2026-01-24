# 引数パースの基礎

CLIツールの使いやすさは、引数パースの設計に大きく依存します。本章では、引数とオプションの種類、パース戦略、検証方法を学びます。

## 引数の種類

### 位置引数（Positional Arguments）

コマンドラインでの位置によって意味が決まる引数です。

```bash
# 基本形
mytool create <name>
mytool copy <source> <destination>

# 実行例
mytool create myapp
mytool copy file.txt backup.txt
```

**特徴**:
- 必須の引数に適している
- 順序が重要
- 分かりやすい

### オプション引数（Options/Flags）

`--`または`-`で始まる名前付き引数です。

```bash
# 長形式
mytool create myapp --template react

# 短縮形
mytool create myapp -t react

# 両方をサポート
mytool create myapp --template react
mytool create myapp -t react
```

**特徴**:
- 省略可能
- 順序は任意
- デフォルト値を設定できる

### フラグ（Boolean Flags）

値を取らず、存在するかどうかだけを判定する引数です。

```bash
# フラグの例
mytool build --watch       # true
mytool build              # false (デフォルト)

# 否定形もサポート
mytool create myapp --typescript      # true
mytool create myapp --no-typescript   # false
```

## 引数のパターン

### 必須 vs 任意

```bash
# 必須引数
mytool create <name>          # name は必須

# 任意引数
mytool list [filter]          # filter は任意

# オプションで必須指定
mytool deploy --env <environment>  # env は必須オプション
```

### 単一値 vs 複数値

```bash
# 単一値
mytool create myapp --template react

# 複数値（配列）
mytool install package1 package2 package3
mytool build --include src --include lib --include tests

# 可変長引数
mytool delete <files...>
```

### カウント

フラグの繰り返し回数をカウントします。

```bash
# 詳細度レベル
mytool run           # 通常
mytool run -v        # verbose (1)
mytool run -vv       # more verbose (2)
mytool run -vvv      # most verbose (3)
```

## パース戦略

### 標準的なパースフロー

```
1. コマンドライン文字列を受け取る
2. トークンに分割
3. オプションと引数を識別
4. 型変換
5. バリデーション
6. デフォルト値の適用
```

### GNU形式 vs BSD形式

```bash
# GNU形式（--で始まる長いオプション）
mytool --output file.txt --verbose

# BSD形式（-1文字）
mytool -o file.txt -v

# 組み合わせ（推奨）
mytool --output file.txt --verbose
mytool -o file.txt -v
```

### オプションの終端

`--`以降はすべて引数として扱います。

```bash
# --以降はファイル名として扱われる
mytool delete -- --weird-filename.txt

# grepなどでよく使われる
grep -- "-pattern" file.txt
```

## 型変換とバリデーション

### 基本的な型

```typescript
// 文字列（デフォルト）
--name myapp              → "myapp"

// 数値
--port 3000               → 3000
--timeout 5.5             → 5.5

// 真偽値
--watch                   → true
--no-typescript           → false

// 配列
--include src --include lib → ["src", "lib"]
```

### カスタム型

```typescript
// 列挙型
type Environment = 'development' | 'staging' | 'production'
--env production

// パス型
--config /path/to/config.json  → Path オブジェクト

// 日付型
--since 2025-01-01            → Date オブジェクト
```

### バリデーション

```typescript
// 範囲チェック
--port 3000                    // 1-65535の範囲
--timeout 30                   // 0以上

// パターンマッチ
--name my-project              // 正規表現: ^[a-z0-9-]+$

// 列挙値
--env production               // development | staging | production

// ファイル存在チェック
--config ./config.json         // ファイルが存在するか

// カスタムバリデーション
--email user@example.com       // メールアドレス形式
```

## ヘルプの自動生成

### 基本的なヘルプ

```bash
$ mytool --help

Usage: mytool <command> [options]

Commands:
  create <name>    Create a new project
  list [filter]    List all projects
  delete <name>    Delete a project

Options:
  -h, --help       Show help
  -v, --version    Show version
  --verbose        Verbose output

Examples:
  mytool create myapp --template react
  mytool list --all
  mytool delete myapp --force
```

### コマンド別ヘルプ

```bash
$ mytool create --help

Usage: mytool create <name> [options]

Create a new project from a template

Arguments:
  name                Project name (required)

Options:
  -t, --template <name>    Template to use (default: "default")
      --typescript         Enable TypeScript (default: true)
      --no-typescript      Disable TypeScript
  -p, --port <number>      Dev server port (default: 3000)
  -f, --force              Overwrite if exists

Examples:
  mytool create myapp
  mytool create myapp --template react --no-typescript
  mytool create myapp --port 4000
```

## エラーメッセージ

### 引数エラー

```bash
# 必須引数が不足
$ mytool create
Error: Missing required argument: name

Usage: mytool create <name> [options]

# 不正な値
$ mytool create myapp --port abc
Error: Invalid value for --port: expected a number, got "abc"

# 不明なオプション
$ mytool create myapp --unknown
Error: Unknown option: --unknown

Did you mean: --typescript?
```

### サジェスト機能

```bash
# typoの検出
$ mytool crate myapp
Error: Unknown command: crate

Did you mean: create?

# オプションのtypo
$ mytool create myapp --templte react
Error: Unknown option: --templte

Did you mean: --template?
```

## 実装例（TypeScript）

### シンプルなパーサー

```typescript
interface ParsedArgs {
  command: string
  args: string[]
  options: Record<string, any>
}

function parseArgs(argv: string[]): ParsedArgs {
  const command = argv[2] || ''
  const args: string[] = []
  const options: Record<string, any> = {}

  for (let i = 3; i < argv.length; i++) {
    const arg = argv[i]

    if (arg.startsWith('--')) {
      // 長形式オプション
      const key = arg.slice(2)
      const nextArg = argv[i + 1]

      if (nextArg && !nextArg.startsWith('-')) {
        options[key] = nextArg
        i++
      } else {
        options[key] = true
      }
    } else if (arg.startsWith('-')) {
      // 短縮形オプション
      const key = arg.slice(1)
      options[key] = true
    } else {
      // 位置引数
      args.push(arg)
    }
  }

  return { command, args, options }
}

// 使用例
const parsed = parseArgs(process.argv)
console.log(parsed)
// {
//   command: 'create',
//   args: ['myapp'],
//   options: { template: 'react', typescript: true }
// }
```

### 型安全なパーサー

```typescript
interface CreateOptions {
  template: string
  typescript: boolean
  port: number
}

function parseCreateOptions(options: Record<string, any>): CreateOptions {
  return {
    template: options.template || 'default',
    typescript: options.typescript !== false,
    port: parseInt(options.port || '3000', 10)
  }
}

// バリデーション付き
function validateCreateOptions(options: CreateOptions): void {
  if (options.port < 1 || options.port > 65535) {
    throw new Error('Port must be between 1 and 65535')
  }

  const validTemplates = ['default', 'react', 'vue', 'nextjs']
  if (!validTemplates.includes(options.template)) {
    throw new Error(`Invalid template: ${options.template}`)
  }
}
```

## ベストプラクティス

### 1. 一貫性を保つ

```bash
# 推奨されないパターン
mytool create --name myapp      # --name を使う
mytool delete myapp             # 位置引数を使う

# 推奨されるパターン
mytool create myapp             # 常に位置引数
mytool delete myapp             # 常に位置引数
```

### 2. デフォルト値を提供

```bash
# 省略時のデフォルト値
mytool create myapp                # template: default, port: 3000
mytool create myapp --port 4000    # template: default, port: 4000
```

### 3. 短縮形を提供

```bash
# よく使うオプションに短縮形を
mytool create myapp -t react -p 4000 -f
mytool create myapp --template react --port 4000 --force
```

### 4. 環境変数フォールバック

```bash
# 環境変数からデフォルト値を取得
export MYTOOL_TEMPLATE=react
mytool create myapp              # template: react (環境変数から)
```

### 5. 対話的プロンプト

```bash
# オプションが省略された場合はプロンプト
$ mytool create
? Project name: myapp
? Template: react
? Enable TypeScript? Yes
? Port: 3000
```

## まとめ

本章では、引数パースの基礎を学びました。

**重要ポイント**:
- 位置引数とオプション引数を適切に使い分ける
- 型変換とバリデーションを実装する
- 分かりやすいヘルプとエラーメッセージを提供する
- デフォルト値と環境変数をサポートする
- 一貫性のある引数設計を心がける

次章では、設定管理の実践的な手法を学びます。
