# Commander.js 基礎

Commander.jsは、Node.jsで最も人気のあるCLIフレームワークです。本章では、Commander.jsを使った引数パース、コマンド定義、オプション設定の基本を学びます。

## インストール

```bash
npm install commander
```

## 基本的な使い方

### シンプルなCLI

```typescript
#!/usr/bin/env node

import { Command } from 'commander'

const program = new Command()

program
  .name('mycli')
  .description('A simple CLI tool')
  .version('1.0.0')

program
  .argument('<name>', 'Project name')
  .option('-t, --template <type>', 'Template to use', 'default')
  .option('-v, --verbose', 'Verbose output')
  .action((name, options) => {
    console.log(`Creating project: ${name}`)
    console.log(`Template: ${options.template}`)
    if (options.verbose) {
      console.log('Verbose mode enabled')
    }
  })

program.parse()
```

実行例:

```bash
$ mycli myapp --template react --verbose
Creating project: myapp
Template: react
Verbose mode enabled
```

## コマンドの定義

### サブコマンド

```typescript
import { Command } from 'commander'

const program = new Command()

program
  .name('mycli')
  .description('A powerful CLI tool')
  .version('1.0.0')

// create コマンド
program
  .command('create <name>')
  .description('Create a new project')
  .option('-t, --template <type>', 'Template to use', 'default')
  .option('--typescript', 'Enable TypeScript', true)
  .option('--no-typescript', 'Disable TypeScript')
  .action((name, options) => {
    console.log(`Creating project: ${name}`)
    console.log(`Template: ${options.template}`)
    console.log(`TypeScript: ${options.typescript}`)
  })

// list コマンド
program
  .command('list')
  .description('List all projects')
  .option('-a, --all', 'Show all projects')
  .action((options) => {
    console.log('Listing projects...')
    if (options.all) {
      console.log('Showing all projects')
    }
  })

// delete コマンド
program
  .command('delete <name>')
  .description('Delete a project')
  .option('-f, --force', 'Force delete without confirmation')
  .action((name, options) => {
    if (!options.force) {
      // 確認プロンプト（Inquirerと組み合わせる）
      console.log(`Are you sure you want to delete ${name}?`)
    } else {
      console.log(`Deleting ${name}...`)
    }
  })

program.parse()
```

### 別ファイルでのコマンド定義

```typescript
// src/commands/create.ts
import { Command } from 'commander'

export const createCommand = new Command('create')
  .description('Create a new project')
  .argument('<name>', 'Project name')
  .option('-t, --template <type>', 'Template to use', 'default')
  .option('--typescript', 'Enable TypeScript', true)
  .action((name, options) => {
    console.log(`Creating project: ${name}`)
    console.log(`Template: ${options.template}`)
    console.log(`TypeScript: ${options.typescript}`)
  })

// src/index.ts
import { Command } from 'commander'
import { createCommand } from './commands/create'
import { listCommand } from './commands/list'
import { deleteCommand } from './commands/delete'

const program = new Command()

program
  .name('mycli')
  .description('A powerful CLI tool')
  .version('1.0.0')

program.addCommand(createCommand)
program.addCommand(listCommand)
program.addCommand(deleteCommand)

program.parse()
```

## 引数の種類

### 必須引数

```typescript
program
  .command('create <name>')
  .action((name) => {
    console.log(`Name: ${name}`)
  })

// $ mycli create myapp
// Name: myapp
```

### 任意引数

```typescript
program
  .command('list [filter]')
  .action((filter) => {
    if (filter) {
      console.log(`Filtering by: ${filter}`)
    } else {
      console.log('Showing all')
    }
  })

// $ mycli list
// Showing all

// $ mycli list active
// Filtering by: active
```

### 可変長引数

```typescript
program
  .command('delete <files...>')
  .action((files) => {
    console.log(`Deleting files: ${files.join(', ')}`)
  })

// $ mycli delete file1.txt file2.txt file3.txt
// Deleting files: file1.txt, file2.txt, file3.txt
```

## オプションの種類

### 真偽値フラグ

```typescript
program
  .option('-v, --verbose', 'Verbose output')
  .option('-q, --quiet', 'Quiet mode')
  .action((options) => {
    console.log(`Verbose: ${options.verbose}`)  // undefined or true
    console.log(`Quiet: ${options.quiet}`)      // undefined or true
  })

// $ mycli --verbose
// Verbose: true
// Quiet: undefined
```

### デフォルト値付きオプション

```typescript
program
  .option('-p, --port <number>', 'Port number', '3000')
  .option('-h, --host <address>', 'Host address', 'localhost')
  .action((options) => {
    console.log(`Server: ${options.host}:${options.port}`)
  })

// $ mycli
// Server: localhost:3000

// $ mycli --port 4000 --host 0.0.0.0
// Server: 0.0.0.0:4000
```

### 型変換

```typescript
program
  .option('-p, --port <number>', 'Port number', parseInt)
  .option('-t, --timeout <seconds>', 'Timeout in seconds', parseFloat)
  .action((options) => {
    console.log(typeof options.port)      // number
    console.log(typeof options.timeout)   // number
  })
```

### カスタムパーサー

```typescript
function parseRange(value: string): [number, number] {
  const parts = value.split('-').map(Number)
  if (parts.length !== 2) {
    throw new Error('Invalid range format')
  }
  return [parts[0], parts[1]]
}

program
  .option('-r, --range <start-end>', 'Port range', parseRange)
  .action((options) => {
    const [start, end] = options.range
    console.log(`Range: ${start} to ${end}`)
  })

// $ mycli --range 3000-3100
// Range: 3000 to 3100
```

### 選択肢

```typescript
program
  .option(
    '-e, --env <type>',
    'Environment',
    'development'
  )
  .addOption(
    new Option('-e, --env <type>', 'Environment')
      .choices(['development', 'staging', 'production'])
      .default('development')
  )
  .action((options) => {
    console.log(`Environment: ${options.env}`)
  })

// $ mycli --env production
// Environment: production

// $ mycli --env invalid
// error: option '-e, --env <type>' argument 'invalid' is invalid.
// Allowed choices are development, staging, production.
```

### 複数値

```typescript
program
  .option('-i, --include <path>', 'Include path', (value, previous = []) => {
    return previous.concat([value])
  }, [])
  .action((options) => {
    console.log(`Included paths: ${options.include.join(', ')}`)
  })

// $ mycli --include src --include lib --include tests
// Included paths: src, lib, tests
```

### カウント

```typescript
program
  .option('-v, --verbose', 'Verbose output', (_, prev) => prev + 1, 0)
  .action((options) => {
    if (options.verbose >= 2) {
      console.log('Debug mode')
    } else if (options.verbose === 1) {
      console.log('Verbose mode')
    }
  })

// $ mycli -vvv
// Debug mode
```

## 高度な機能

### バリデーション

```typescript
import { InvalidArgumentError } from 'commander'

function validatePort(value: string): number {
  const port = parseInt(value, 10)
  if (isNaN(port) || port < 1 || port > 65535) {
    throw new InvalidArgumentError('Port must be between 1 and 65535')
  }
  return port
}

program
  .option('-p, --port <number>', 'Port number', validatePort)
  .action((options) => {
    console.log(`Port: ${options.port}`)
  })

// $ mycli --port 70000
// error: option '-p, --port <number>' argument '70000' is invalid.
// Port must be between 1 and 65535
```

### 環境変数フォールバック

```typescript
program
  .option(
    '-k, --api-key <key>',
    'API key',
    process.env.API_KEY
  )
  .action((options) => {
    if (!options.apiKey) {
      console.error('Error: API key is required')
      process.exit(1)
    }
    console.log(`API Key: ${options.apiKey.substring(0, 4)}...`)
  })

// $ export API_KEY=abc123
// $ mycli
// API Key: abc1...
```

### ヘルプのカスタマイズ

```typescript
program
  .name('mycli')
  .usage('[command] [options]')
  .description('A powerful CLI tool for project management')
  .addHelpText('before', `
Custom CLI Tool v1.0.0
━━━━━━━━━━━━━━━━━━━━━
`)
  .addHelpText('after', `
Examples:
  $ mycli create myapp --template react
  $ mycli list --all
  $ mycli delete myapp --force

Documentation:
  https://github.com/username/mycli#readme
`)
```

### エラーハンドリング

```typescript
program
  .exitOverride()  // エラー時に終了しない
  .configureOutput({
    writeErr: (str) => console.error(`Error: ${str}`),
    outputError: (str, write) => write(`Custom error: ${str}`)
  })

try {
  program.parse()
} catch (error) {
  console.error('Command failed:', error.message)
  process.exit(1)
}
```

## TypeScript型定義

### オプションの型

```typescript
interface CreateOptions {
  template: string
  typescript: boolean
  port: number
  features: string[]
}

program
  .command('create <name>')
  .option('-t, --template <type>', 'Template', 'default')
  .option('--typescript', 'Enable TypeScript', true)
  .option('-p, --port <number>', 'Port', parseInt, 3000)
  .option('-f, --features <items>', 'Features', (val, prev = []) => {
    return prev.concat([val])
  }, [])
  .action((name: string, options: CreateOptions) => {
    // options は型付けされている
    console.log(`Creating ${name}`)
    console.log(`Template: ${options.template}`)
    console.log(`TypeScript: ${options.typescript}`)
    console.log(`Port: ${options.port}`)
    console.log(`Features: ${options.features.join(', ')}`)
  })
```

## まとめ

本章では、Commander.jsの基本的な使い方を学びました。

**重要ポイント**:
- サブコマンドで機能を整理
- 適切なオプションで柔軟性を提供
- バリデーションでユーザー入力をチェック
- 型定義で開発体験を向上
- カスタマイズでユーザビリティを改善

Commander.jsを使うことで、プロフェッショナルなCLIツールを効率的に開発できます。次章では、Inquirerを使ったインタラクティブUIを学びます。
