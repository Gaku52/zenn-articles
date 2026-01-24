# Node.js CLIベストプラクティス

本章では、プロフェッショナルなCLIツールを開発するためのベストプラクティスを学びます。パフォーマンス、セキュリティ、ユーザビリティの観点から、実践的な手法を紹介します。

## パフォーマンス最適化

### 起動時間の短縮

```typescript
// 推奨されないパターン: すべてを事前にインポート
import inquirer from 'inquirer'
import ora from 'ora'
import chalk from 'chalk'
import axios from 'axios'
// ... たくさんのインポート

// 推奨されるパターン: 遅延ロード
const program = new Command()

program
  .command('create')
  .action(async () => {
    // 必要な時だけインポート
    const inquirer = (await import('inquirer')).default
    const ora = (await import('ora')).default

    // コマンド実行
  })
```

### キャッシュの活用

```typescript
// src/utils/cache.ts
import { join } from 'path'
import { homedir } from 'os'
import fs from 'fs/promises'

export class Cache {
  private cacheDir: string

  constructor(toolName: string) {
    this.cacheDir = join(homedir(), `.${toolName}`, 'cache')
  }

  async get<T>(key: string): Promise<T | null> {
    const cachePath = join(this.cacheDir, `${key}.json`)

    try {
      const data = await fs.readFile(cachePath, 'utf-8')
      const cached = JSON.parse(data)

      // 有効期限チェック
      if (cached.expiresAt < Date.now()) {
        return null
      }

      return cached.value
    } catch {
      return null
    }
  }

  async set<T>(key: string, value: T, ttl: number = 3600000): Promise<void> {
    const cachePath = join(this.cacheDir, `${key}.json`)

    await fs.mkdir(this.cacheDir, { recursive: true })
    await fs.writeFile(
      cachePath,
      JSON.stringify({
        value,
        expiresAt: Date.now() + ttl
      }),
      'utf-8'
    )
  }
}

// 使用例
const cache = new Cache('mycli')
const templates = await cache.get('templates') || await fetchTemplates()
await cache.set('templates', templates, 24 * 60 * 60 * 1000) // 24時間
```

## セキュリティ

### 入力のサニタイズ

```typescript
// src/utils/sanitize.ts
export function sanitizePath(input: string): string {
  // パストラバーサル攻撃を防ぐ
  return input.replace(/\.\./g, '').replace(/^\/+/, '')
}

export function sanitizeCommand(input: string): string {
  // コマンドインジェクションを防ぐ
  return input.replace(/[;&|`$()]/g, '')
}

// 使用例
const userInput = '../../../etc/passwd'
const safePath = sanitizePath(userInput)  // 'etc/passwd'
```

### 機密情報の管理

```typescript
// 推奨されないパターン
console.log(`API Key: ${apiKey}`)

// 推奨されるパターン
console.log(`API Key: ${apiKey.substring(0, 4)}...`)

// 環境変数で管理
const apiKey = process.env.API_KEY
if (!apiKey) {
  console.error('Error: API_KEY environment variable is required')
  process.exit(1)
}
```

### 安全なファイル操作

```typescript
import { access, constants } from 'fs/promises'
import { resolve } from 'path'

async function safeWrite(filePath: string, content: string) {
  const absolutePath = resolve(filePath)

  // ディレクトリトラバーサルチェック
  if (!absolutePath.startsWith(process.cwd())) {
    throw new Error('Path outside current directory')
  }

  // 上書き前の確認
  try {
    await access(absolutePath, constants.F_OK)
    const { overwrite } = await inquirer.prompt([{
      type: 'confirm',
      name: 'overwrite',
      message: `File ${filePath} already exists. Overwrite?`,
      default: false
    }])

    if (!overwrite) {
      return
    }
  } catch {
    // ファイルが存在しない場合は続行
  }

  await writeFile(absolutePath, content)
}
```

## エラーハンドリング

### グローバルエラーハンドラー

```typescript
// src/utils/error-handler.ts
import chalk from 'chalk'

export class CLIError extends Error {
  constructor(
    message: string,
    public exitCode: number = 1,
    public suggestions?: string[]
  ) {
    super(message)
    this.name = 'CLIError'
  }
}

export function setupErrorHandler() {
  process.on('uncaughtException', (error) => {
    if (error instanceof CLIError) {
      console.error(chalk.red(`Error: ${error.message}`))

      if (error.suggestions) {
        console.error(chalk.yellow('\nSuggestions:'))
        error.suggestions.forEach(s => console.error(chalk.yellow(`  - ${s}`)))
      }

      process.exit(error.exitCode)
    } else {
      console.error(chalk.red('Unexpected error:'), error.message)
      console.error(error.stack)
      process.exit(1)
    }
  })

  process.on('unhandledRejection', (reason) => {
    console.error(chalk.red('Unhandled promise rejection:'), reason)
    process.exit(1)
  })

  // Ctrl+C のハンドリング
  process.on('SIGINT', () => {
    console.log(chalk.yellow('\n\nOperation cancelled'))
    process.exit(130)
  })
}

// src/index.ts
setupErrorHandler()
```

### ユーザーフレンドリーなエラー

```typescript
// 推奨されないパターン
throw new Error('ENOENT: no such file or directory')

// 推奨されるパターン
throw new CLIError(
  'Configuration file not found',
  1,
  [
    'Run "mycli init" to create a configuration file',
    'Specify a config file with --config option'
  ]
)
```

## ユーザビリティ向上

### プログレス表示

```typescript
import ora from 'ora'

async function installDependencies() {
  const spinner = ora('Installing dependencies...').start()

  try {
    await execa('npm', ['install'])
    spinner.succeed('Dependencies installed')
  } catch (error) {
    spinner.fail('Failed to install dependencies')
    throw error
  }
}
```

### 適切なフィードバック

```typescript
import chalk from 'chalk'

// 成功
console.log(chalk.green('✓ Project created successfully!'))

// エラー
console.error(chalk.red('✗ Failed to create project'))

// 警告
console.warn(chalk.yellow('⚠ Warning: No .gitignore found'))

// 情報
console.log(chalk.blue('ℹ Installing dependencies...'))
```

### 詳細度の制御

```typescript
// src/utils/logger.ts
export enum LogLevel {
  Silent = 0,
  Error = 1,
  Warn = 2,
  Info = 3,
  Debug = 4
}

export class Logger {
  constructor(private level: LogLevel = LogLevel.Info) {}

  error(message: string): void {
    if (this.level >= LogLevel.Error) {
      console.error(chalk.red(message))
    }
  }

  warn(message: string): void {
    if (this.level >= LogLevel.Warn) {
      console.warn(chalk.yellow(message))
    }
  }

  info(message: string): void {
    if (this.level >= LogLevel.Info) {
      console.log(message)
    }
  }

  debug(message: string): void {
    if (this.level >= LogLevel.Debug) {
      console.log(chalk.gray(`[DEBUG] ${message}`))
    }
  }
}

// 使用例
const logger = new Logger(
  options.verbose ? LogLevel.Debug : LogLevel.Info
)
```

## ドキュメント

### インラインヘルプ

```typescript
program
  .command('create <name>')
  .description('Create a new project')
  .option('-t, --template <type>', 'Template to use', 'default')
  .addHelpText('after', `
Examples:
  $ mycli create myapp
  $ mycli create myapp --template react
  $ mycli create myapp --template nextjs --typescript

Templates:
  default  - Basic template
  react    - React with TypeScript
  vue      - Vue 3 with TypeScript
  nextjs   - Next.js with App Router
`)
```

### README.md

```markdown
# My CLI Tool

A powerful CLI tool for project generation.

## Installation

```bash
npm install -g my-cli-tool
```

## Usage

```bash
# Create a new project
mycli create myapp

# With options
mycli create myapp --template react --typescript

# List templates
mycli list

# Show help
mycli --help
```

## Examples

### Create a React project

```bash
mycli create myapp --template react
cd myapp
npm install
npm run dev
```

### Create a Next.js project

```bash
mycli create myapp --template nextjs
cd myapp
npm run dev
```

## Configuration

Create a `.myclirc` file in your project or home directory:

```json
{
  "template": "react",
  "typescript": true,
  "features": ["eslint", "prettier"]
}
```

## License

MIT
```

## テレメトリー

### 匿名使用統計（オプトイン）

```typescript
// src/utils/telemetry.ts
import { randomUUID } from 'crypto'

export class Telemetry {
  private userId: string
  private enabled: boolean

  constructor() {
    this.userId = this.getUserId()
    this.enabled = this.isEnabled()
  }

  private getUserId(): string {
    // ユーザー固有のIDを生成・取得
    const configPath = join(homedir(), '.mycli', 'config.json')
    // ... ID取得/生成処理
    return randomUUID()
  }

  private isEnabled(): boolean {
    // ユーザーが明示的にオプトインした場合のみ有効
    return process.env.MYCLI_TELEMETRY === 'true'
  }

  async track(event: string, properties?: Record<string, any>) {
    if (!this.enabled) {
      return
    }

    try {
      await axios.post('https://api.example.com/telemetry', {
        userId: this.userId,
        event,
        properties,
        timestamp: new Date().toISOString()
      })
    } catch {
      // テレメトリー送信失敗は無視
    }
  }
}
```

## まとめ

本章では、Node.js CLIのベストプラクティスを学びました。

**重要ポイント**:
- 遅延ロードで起動時間を短縮
- キャッシュでパフォーマンス向上
- 入力サニタイズでセキュリティ確保
- 適切なエラーハンドリングでユーザビリティ向上
- プログレス表示でフィードバック提供
- 充実したドキュメントでユーザーをサポート

これらのベストプラクティスを実践することで、プロフェッショナルなCLIツールを開発できます。次章からは、Python CLIの開発を学んでいきます。
