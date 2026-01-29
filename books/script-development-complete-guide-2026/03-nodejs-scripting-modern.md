---
title: "Node.js による現代的なスクリプト開発"
---

# Node.js による現代的なスクリプト開発

Node.jsとTypeScriptを使用することで、型安全で保守性の高い現代的なスクリプトを開発できます。本章では、CLI ツール作成、非同期処理、ファイル操作などの実践的なテクニックを学びます。

## TypeScript によるスクリプト開発

### プロジェクトのセットアップ

```bash
# プロジェクト初期化
mkdir my-script && cd my-script
npm init -y

# TypeScript と必要なパッケージのインストール
npm install -D typescript @types/node ts-node
npm install commander chalk inquirer ora
npm install -D @types/inquirer

# tsconfig.json の作成
npx tsc --init
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

```json
// package.json
{
  "name": "my-script",
  "version": "1.0.0",
  "bin": {
    "my-script": "./dist/index.js"
  },
  "scripts": {
    "build": "tsc",
    "dev": "ts-node src/index.ts",
    "start": "node dist/index.js",
    "watch": "tsc --watch"
  }
}
```

### 基本的なスクリプト構造

```typescript
#!/usr/bin/env node
// src/index.ts

import { program } from 'commander'
import chalk from 'chalk'

const VERSION = '1.0.0'

program
  .name('my-script')
  .description('スクリプトの説明')
  .version(VERSION)

program
  .command('process')
  .description('ファイルを処理する')
  .argument('<input>', '入力ファイル')
  .option('-o, --output <path>', '出力ファイル')
  .option('-v, --verbose', '詳細出力')
  .action(async (input, options) => {
    try {
      console.log(chalk.blue('Processing...'))
      await processFile(input, options)
      console.log(chalk.green('✓ Complete'))
    } catch (error) {
      console.error(chalk.red('✗ Error:'), error)
      process.exit(1)
    }
  })

program.parse(process.argv)

async function processFile(input: string, options: any): Promise<void> {
  // 処理内容
  if (options.verbose) {
    console.log(`Input: ${input}`)
    console.log(`Output: ${options.output || 'stdout'}`)
  }
}
```

## Commander.js による CLI ツール開発

### 高度なコマンド定義

```typescript
import { program, Command, Option } from 'commander'
import { readFileSync } from 'fs'
import { resolve } from 'path'

interface DeployOptions {
  environment: string
  version?: string
  dryRun: boolean
  verbose: boolean
}

// グローバルオプション
program
  .option('-c, --config <path>', '設定ファイルのパス', './config.json')
  .option('--no-color', '色付き出力を無効化')

// deploy コマンド
const deployCommand = new Command('deploy')
  .description('アプリケーションをデプロイする')
  .addOption(
    new Option('-e, --environment <env>', '環境')
      .choices(['development', 'staging', 'production'])
      .makeOptionMandatory()
  )
  .option('-v, --version <version>', 'デプロイするバージョン', 'latest')
  .option('--dry-run', 'ドライランモード', false)
  .option('--verbose', '詳細出力', false)
  .action(async (options: DeployOptions) => {
    await deploy(options)
  })

program.addCommand(deployCommand)

async function deploy(options: DeployOptions): Promise<void> {
  console.log('Deploying...')
  console.log(`Environment: ${options.environment}`)
  console.log(`Version: ${options.version}`)
  console.log(`Dry run: ${options.dryRun}`)

  if (options.dryRun) {
    console.log('[DRY RUN] No actual changes will be made')
    return
  }

  // デプロイ処理
}

// サブコマンドのグループ化
const dbCommand = new Command('db').description('データベース操作')

dbCommand
  .command('migrate')
  .description('マイグレーションを実行')
  .option('--rollback', 'ロールバック')
  .action(async (options) => {
    if (options.rollback) {
      await rollbackMigration()
    } else {
      await runMigration()
    }
  })

dbCommand
  .command('seed')
  .description('シードデータを投入')
  .option('--truncate', '既存データを削除')
  .action(async (options) => {
    await seedDatabase(options)
  })

program.addCommand(dbCommand)
```

### 対話的なプロンプト

```typescript
import inquirer from 'inquirer'
import ora from 'ora'

interface SetupAnswers {
  projectName: string
  framework: string
  features: string[]
  installDeps: boolean
}

async function interactiveSetup(): Promise<void> {
  console.log('プロジェクトのセットアップ')
  console.log()

  const answers = await inquirer.prompt<SetupAnswers>([
    {
      type: 'input',
      name: 'projectName',
      message: 'プロジェクト名:',
      default: 'my-project',
      validate: (input) => {
        if (!/^[a-z0-9-]+$/.test(input)) {
          return 'プロジェクト名は小文字、数字、ハイフンのみ使用できます'
        }
        return true
      }
    },
    {
      type: 'list',
      name: 'framework',
      message: 'フレームワークを選択:',
      choices: [
        { name: 'React', value: 'react' },
        { name: 'Vue', value: 'vue' },
        { name: 'Next.js', value: 'nextjs' }
      ]
    },
    {
      type: 'checkbox',
      name: 'features',
      message: '追加機能を選択:',
      choices: [
        { name: 'TypeScript', value: 'typescript', checked: true },
        { name: 'ESLint', value: 'eslint', checked: true },
        { name: 'Prettier', value: 'prettier', checked: true },
        { name: 'Testing (Jest)', value: 'jest' }
      ]
    },
    {
      type: 'confirm',
      name: 'installDeps',
      message: '依存関係を自動インストールしますか?',
      default: true
    }
  ])

  // セットアップ処理
  const spinner = ora('プロジェクトを作成中...').start()

  try {
    await createProject(answers)
    spinner.succeed('プロジェクトが作成されました')

    if (answers.installDeps) {
      spinner.start('依存関係をインストール中...')
      await installDependencies(answers.projectName)
      spinner.succeed('依存関係のインストールが完了しました')
    }

    console.log()
    console.log(chalk.green('✓ セットアップ完了!'))
    console.log()
    console.log('次のコマンドでプロジェクトを開始:')
    console.log(chalk.cyan(`  cd ${answers.projectName}`))
    console.log(chalk.cyan('  npm start'))

  } catch (error) {
    spinner.fail('エラーが発生しました')
    throw error
  }
}
```

## ファイル操作

### 非同期ファイル操作

```typescript
import { promises as fs } from 'fs'
import { join, basename, extname } from 'path'
import { glob } from 'glob'

/**
 * ファイルを読み込む
 */
async function readFile(filePath: string): Promise<string> {
  try {
    return await fs.readFile(filePath, 'utf-8')
  } catch (error) {
    throw new Error(`Failed to read file: ${filePath}`)
  }
}

/**
 * ファイルに書き込む（アトミック）
 */
async function writeFileAtomic(
  filePath: string,
  content: string
): Promise<void> {
  const tempPath = `${filePath}.tmp`

  try {
    await fs.writeFile(tempPath, content, 'utf-8')
    await fs.rename(tempPath, filePath)
  } catch (error) {
    // クリーンアップ
    await fs.unlink(tempPath).catch(() => {})
    throw error
  }
}

/**
 * ディレクトリ内のファイルを再帰的に取得
 */
async function getFilesRecursive(
  dirPath: string,
  pattern: string = '**/*'
): Promise<string[]> {
  return glob(pattern, {
    cwd: dirPath,
    absolute: true,
    nodir: true
  })
}

/**
 * ファイルをコピー
 */
async function copyFile(src: string, dest: string): Promise<void> {
  // 親ディレクトリを作成
  const destDir = join(dest, '..')
  await fs.mkdir(destDir, { recursive: true })

  await fs.copyFile(src, dest)
}

/**
 * ディレクトリを再帰的にコピー
 */
async function copyDirectory(src: string, dest: string): Promise<void> {
  await fs.mkdir(dest, { recursive: true })

  const entries = await fs.readdir(src, { withFileTypes: true })

  for (const entry of entries) {
    const srcPath = join(src, entry.name)
    const destPath = join(dest, entry.name)

    if (entry.isDirectory()) {
      await copyDirectory(srcPath, destPath)
    } else {
      await fs.copyFile(srcPath, destPath)
    }
  }
}
```

### ストリーム処理

```typescript
import { createReadStream, createWriteStream } from 'fs'
import { createInterface } from 'readline'
import { Transform } from 'stream'
import { pipeline } from 'stream/promises'

/**
 * 大きなファイルを行単位で処理
 */
async function processLargeFile(
  inputPath: string,
  outputPath: string,
  transform: (line: string) => string
): Promise<void> {
  const input = createReadStream(inputPath)
  const output = createWriteStream(outputPath)

  const rl = createInterface({
    input,
    crlfDelay: Infinity
  })

  for await (const line of rl) {
    const processed = transform(line)
    output.write(processed + '\n')
  }

  output.end()
}

/**
 * カスタムTransformストリーム
 */
class UpperCaseTransform extends Transform {
  _transform(
    chunk: Buffer,
    encoding: string,
    callback: Function
  ): void {
    const transformed = chunk.toString().toUpperCase()
    callback(null, transformed)
  }
}

/**
 * ストリームパイプライン
 */
async function transformFile(
  inputPath: string,
  outputPath: string
): Promise<void> {
  await pipeline(
    createReadStream(inputPath),
    new UpperCaseTransform(),
    createWriteStream(outputPath)
  )
}
```

## API クライアント

### 型安全な API クライアント

```typescript
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios'

interface User {
  id: number
  name: string
  email: string
}

interface ApiResponse<T> {
  data: T
  meta: {
    total: number
    page: number
  }
}

class APIClient {
  private client: AxiosInstance

  constructor(baseURL: string, apiKey?: string) {
    this.client = axios.create({
      baseURL,
      timeout: 30000,
      headers: {
        'Content-Type': 'application/json'
      }
    })

    if (apiKey) {
      this.client.defaults.headers.common['Authorization'] = `Bearer ${apiKey}`
    }

    // リクエストインターセプター
    this.client.interceptors.request.use(
      (config) => {
        console.log(`→ ${config.method?.toUpperCase()} ${config.url}`)
        return config
      },
      (error) => {
        return Promise.reject(error)
      }
    )

    // レスポンスインターセプター
    this.client.interceptors.response.use(
      (response) => {
        console.log(`← ${response.status} ${response.config.url}`)
        return response
      },
      (error) => {
        console.error(`✗ ${error.response?.status} ${error.config?.url}`)
        return Promise.reject(error)
      }
    )
  }

  async get<T>(
    endpoint: string,
    params?: Record<string, any>
  ): Promise<T> {
    const response = await this.client.get<T>(endpoint, { params })
    return response.data
  }

  async post<T>(
    endpoint: string,
    data: any
  ): Promise<T> {
    const response = await this.client.post<T>(endpoint, data)
    return response.data
  }

  async put<T>(
    endpoint: string,
    data: any
  ): Promise<T> {
    const response = await this.client.put<T>(endpoint, data)
    return response.data
  }

  async delete<T>(endpoint: string): Promise<T> {
    const response = await this.client.delete<T>(endpoint)
    return response.data
  }
}

// 使用例
async function fetchUsers(): Promise<void> {
  const client = new APIClient(
    'https://api.example.com',
    process.env.API_KEY
  )

  try {
    const response = await client.get<ApiResponse<User[]>>('/users', {
      limit: 100
    })

    console.log(`Fetched ${response.data.length} users`)

    // ファイルに保存
    await fs.writeFile(
      'users.json',
      JSON.stringify(response.data, null, 2)
    )

  } catch (error) {
    if (axios.isAxiosError(error)) {
      console.error(`API Error: ${error.response?.status}`)
      console.error(error.response?.data)
    }
    throw error
  }
}
```

### リトライ機能付きHTTPクライアント

```typescript
import axios, { AxiosError } from 'axios'

interface RetryConfig {
  maxRetries: number
  retryDelay: number
  retryableStatuses: number[]
}

async function fetchWithRetry<T>(
  url: string,
  config: RetryConfig = {
    maxRetries: 3,
    retryDelay: 1000,
    retryableStatuses: [429, 500, 502, 503, 504]
  }
): Promise<T> {
  let lastError: Error | undefined

  for (let attempt = 1; attempt <= config.maxRetries; attempt++) {
    try {
      const response = await axios.get<T>(url)
      return response.data
    } catch (error) {
      lastError = error as Error

      if (axios.isAxiosError(error)) {
        const status = error.response?.status

        // リトライ可能なステータスコードでない場合は即座に失敗
        if (status && !config.retryableStatuses.includes(status)) {
          throw error
        }

        if (attempt < config.maxRetries) {
          const delay = config.retryDelay * attempt
          console.log(`Retry ${attempt}/${config.maxRetries} after ${delay}ms`)
          await new Promise(resolve => setTimeout(resolve, delay))
          continue
        }
      }

      throw error
    }
  }

  throw lastError
}
```

## 並行処理

### Promise による並行処理

```typescript
/**
 * 複数のタスクを並行実行（制限付き）
 */
async function parallelLimit<T, R>(
  items: T[],
  limit: number,
  fn: (item: T) => Promise<R>
): Promise<R[]> {
  const results: R[] = []
  const executing: Promise<void>[] = []

  for (const item of items) {
    const promise = fn(item).then(result => {
      results.push(result)
    })

    executing.push(promise)

    if (executing.length >= limit) {
      await Promise.race(executing)
      // 完了したプロミスを削除
      for (let i = executing.length - 1; i >= 0; i--) {
        if (await Promise.race([executing[i], Promise.resolve('done')]) === 'done') {
          executing.splice(i, 1)
        }
      }
    }
  }

  await Promise.all(executing)
  return results
}

// 使用例
async function downloadFiles(urls: string[]): Promise<void> {
  const downloadFile = async (url: string): Promise<string> => {
    const response = await axios.get(url, { responseType: 'arraybuffer' })
    const filename = basename(url)
    await fs.writeFile(filename, response.data)
    return filename
  }

  const downloaded = await parallelLimit(urls, 5, downloadFile)
  console.log(`Downloaded ${downloaded.length} files`)
}
```

### Worker Threads の活用

```typescript
import { Worker } from 'worker_threads'
import { cpus } from 'os'

interface WorkerTask<T, R> {
  data: T
  resolve: (value: R) => void
  reject: (error: Error) => void
}

class WorkerPool<T, R> {
  private workers: Worker[] = []
  private queue: WorkerTask<T, R>[] = []
  private activeWorkers = 0

  constructor(
    private workerScript: string,
    private poolSize: number = cpus().length
  ) {
    this.initializeWorkers()
  }

  private initializeWorkers(): void {
    for (let i = 0; i < this.poolSize; i++) {
      const worker = new Worker(this.workerScript)

      worker.on('message', (result: R) => {
        this.activeWorkers--
        this.processQueue()
      })

      worker.on('error', (error) => {
        console.error('Worker error:', error)
        this.activeWorkers--
        this.processQueue()
      })

      this.workers.push(worker)
    }
  }

  async execute(data: T): Promise<R> {
    return new Promise<R>((resolve, reject) => {
      this.queue.push({ data, resolve, reject })
      this.processQueue()
    })
  }

  private processQueue(): void {
    if (this.queue.length === 0 || this.activeWorkers >= this.poolSize) {
      return
    }

    const task = this.queue.shift()
    if (!task) return

    this.activeWorkers++

    const worker = this.workers[this.activeWorkers - 1]
    worker.postMessage(task.data)

    worker.once('message', (result: R) => {
      task.resolve(result)
    })

    worker.once('error', (error: Error) => {
      task.reject(error)
    })
  }

  async close(): Promise<void> {
    await Promise.all(
      this.workers.map(worker => worker.terminate())
    )
  }
}
```

## 実践例: ファイル変換ツール

```typescript
#!/usr/bin/env node
// src/converter.ts

import { program } from 'commander'
import { promises as fs } from 'fs'
import { join, extname, basename } from 'path'
import chalk from 'chalk'
import ora from 'ora'
import { glob } from 'glob'

interface ConvertOptions {
  output?: string
  format: 'json' | 'csv' | 'yaml'
  verbose: boolean
}

program
  .name('converter')
  .description('ファイル形式変換ツール')
  .version('1.0.0')

program
  .command('convert')
  .description('ファイルを変換する')
  .argument('<input>', '入力ファイル（ワイルドカード対応）')
  .option('-o, --output <dir>', '出力ディレクトリ')
  .option('-f, --format <format>', '出力フォーマット', 'json')
  .option('-v, --verbose', '詳細出力')
  .action(async (input: string, options: ConvertOptions) => {
    const spinner = ora('ファイルを検索中...').start()

    try {
      // ファイルを取得
      const files = await glob(input)

      if (files.length === 0) {
        spinner.fail('ファイルが見つかりません')
        process.exit(1)
      }

      spinner.succeed(`${files.length} 個のファイルが見つかりました`)

      // 変換処理
      const convertSpinner = ora('変換中...').start()

      for (const file of files) {
        try {
          await convertFile(file, options)

          if (options.verbose) {
            console.log(chalk.gray(`  ✓ ${basename(file)}`))
          }
        } catch (error) {
          convertSpinner.warn(`Failed to convert ${file}`)
          console.error(chalk.red(`    ${error}`))
        }
      }

      convertSpinner.succeed('変換が完了しました')

    } catch (error) {
      spinner.fail('エラーが発生しました')
      console.error(chalk.red(error))
      process.exit(1)
    }
  })

async function convertFile(
  inputPath: string,
  options: ConvertOptions
): Promise<void> {
  // ファイルを読み込み
  const content = await fs.readFile(inputPath, 'utf-8')

  // JSONとしてパース
  const data = JSON.parse(content)

  // 変換
  let converted: string

  switch (options.format) {
    case 'json':
      converted = JSON.stringify(data, null, 2)
      break
    case 'csv':
      converted = convertToCSV(data)
      break
    case 'yaml':
      converted = convertToYAML(data)
      break
    default:
      throw new Error(`Unknown format: ${options.format}`)
  }

  // 出力パスを決定
  const outputDir = options.output || process.cwd()
  const filename = basename(inputPath, extname(inputPath))
  const outputPath = join(outputDir, `${filename}.${options.format}`)

  // 出力ディレクトリを作成
  await fs.mkdir(outputDir, { recursive: true })

  // ファイルに書き込み
  await fs.writeFile(outputPath, converted, 'utf-8')
}

function convertToCSV(data: any[]): string {
  if (!Array.isArray(data) || data.length === 0) {
    throw new Error('CSV conversion requires an array')
  }

  const headers = Object.keys(data[0])
  const rows = data.map(row =>
    headers.map(header => JSON.stringify(row[header] ?? '')).join(',')
  )

  return [headers.join(','), ...rows].join('\n')
}

function convertToYAML(data: any): string {
  // 簡易的なYAML変換
  return JSON.stringify(data, null, 2)
    .replace(/^\{/gm, '')
    .replace(/\}$/gm, '')
    .replace(/"/g, '')
}

program.parse(process.argv)
```

## まとめ

本章では、Node.jsとTypeScriptを使った現代的なスクリプト開発を学びました：

- **TypeScript**: 型安全なスクリプト開発
- **Commander.js**: 高機能なCLIツール作成
- **ファイル操作**: 非同期処理、ストリーム
- **API クライアント**: 型安全なHTTPクライアント、リトライ機能
- **並行処理**: Promise、Worker Threads

次章では、これらの知識を活用した自動化とデプロイメントについて学びます。
