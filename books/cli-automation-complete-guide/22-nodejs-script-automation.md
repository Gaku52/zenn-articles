# Node.jsスクリプト自動化

本章では、Node.js/TypeScriptを使った自動化スクリプトの開発を学びます。Node.jsは、JavaScriptエコシステムの豊富なパッケージを活用でき、型安全性を確保しながら強力な自動化ツールを構築できます。

## Node.js自動化の基礎

### 基本スクリプト構成

```typescript
#!/usr/bin/env ts-node

import { execSync } from 'child_process'
import * as fs from 'fs'
import * as path from 'path'

// 引数解析
const args = process.argv.slice(2)
const env = args[0] || 'development'

console.log(`Deploying to ${env}...`)

// 環境変数
process.env.NODE_ENV = env

// ビルド実行
try {
    execSync('npm run build', { stdio: 'inherit' })
    console.log('Build completed successfully')
} catch (error) {
    console.error('Build failed:', error)
    process.exit(1)
}
```

### TypeScriptの型安全性を活用

```typescript
#!/usr/bin/env ts-node

import { execSync, ExecSyncOptions } from 'child_process'

// 設定型定義
interface DeployConfig {
    environment: string
    branch: string
    buildCommand: string
    deployPath: string
}

// 環境別設定
const configs: Record<string, DeployConfig> = {
    development: {
        environment: 'development',
        branch: 'develop',
        buildCommand: 'npm run build:dev',
        deployPath: '/var/www/dev',
    },
    production: {
        environment: 'production',
        branch: 'main',
        buildCommand: 'npm run build:prod',
        deployPath: '/var/www/prod',
    },
}

// デプロイ関数
function deploy(config: DeployConfig): void {
    console.log(`Deploying to ${config.environment}`)

    const execOptions: ExecSyncOptions = {
        stdio: 'inherit',
        encoding: 'utf-8',
    }

    // ビルド
    execSync(config.buildCommand, execOptions)

    // デプロイ
    execSync(`rsync -avz dist/ ${config.deployPath}/`, execOptions)

    console.log('Deployment completed')
}

// 実行
const env = process.argv[2] || 'development'
const config = configs[env]

if (!config) {
    console.error(`Unknown environment: ${env}`)
    process.exit(1)
}

deploy(config)
```

## ファイル操作

### fsモジュールによるファイル処理

```typescript
import * as fs from 'fs/promises'
import * as path from 'path'

// ディレクトリ内のファイルを再帰的に取得
async function getFiles(dir: string): Promise<string[]> {
    const entries = await fs.readdir(dir, { withFileTypes: true })
    const files = await Promise.all(
        entries.map(async (entry) => {
            const fullPath = path.join(dir, entry.name)
            return entry.isDirectory() ? getFiles(fullPath) : [fullPath]
        })
    )
    return files.flat()
}

// ファイル一括処理
async function processFiles(dirPath: string): Promise<void> {
    try {
        const files = await getFiles(dirPath)

        for (const filePath of files) {
            if (!filePath.endsWith('.txt')) continue

            console.log(`Processing: ${filePath}`)

            // ファイル読み込み
            const content = await fs.readFile(filePath, 'utf-8')

            // 処理（例: 大文字変換）
            const processed = content.toUpperCase()

            // ファイル書き込み
            await fs.writeFile(filePath, processed, 'utf-8')
        }

        console.log(`Processed ${files.length} files`)
    } catch (error) {
        console.error('Error processing files:', error)
        throw error
    }
}

// 実行
processFiles('./data').catch((error) => {
    console.error('Fatal error:', error)
    process.exit(1)
})
```

### ファイルコピー・移動

```typescript
import * as fs from 'fs/promises'
import * as path from 'path'

interface CopyOptions {
    overwrite?: boolean
    filter?: (src: string) => boolean
}

// ディレクトリコピー
async function copyDirectory(
    src: string,
    dest: string,
    options: CopyOptions = {}
): Promise<void> {
    const { overwrite = false, filter = () => true } = options

    // コピー先ディレクトリ作成
    await fs.mkdir(dest, { recursive: true })

    const entries = await fs.readdir(src, { withFileTypes: true })

    for (const entry of entries) {
        const srcPath = path.join(src, entry.name)
        const destPath = path.join(dest, entry.name)

        if (!filter(srcPath)) continue

        if (entry.isDirectory()) {
            await copyDirectory(srcPath, destPath, options)
        } else {
            // ファイル存在チェック
            if (!overwrite) {
                try {
                    await fs.access(destPath)
                    console.log(`Skipping existing file: ${destPath}`)
                    continue
                } catch {
                    // ファイルが存在しない場合は継続
                }
            }

            await fs.copyFile(srcPath, destPath)
            console.log(`Copied: ${srcPath} -> ${destPath}`)
        }
    }
}

// 使用例
copyDirectory('./src', './backup', {
    overwrite: false,
    filter: (src) => !src.includes('node_modules'),
}).catch(console.error)
```

## child_processによるコマンド実行

### exec vs spawn

```typescript
import { exec, spawn } from 'child_process'
import { promisify } from 'util'

const execAsync = promisify(exec)

// exec: 短時間のコマンド、出力が小さい場合
async function runExec(): Promise<void> {
    try {
        const { stdout, stderr } = await execAsync('ls -la')
        console.log('Output:', stdout)
        if (stderr) {
            console.error('Errors:', stderr)
        }
    } catch (error) {
        console.error('Command failed:', error)
        throw error
    }
}

// spawn: 長時間のコマンド、出力が大きい場合
function runSpawn(): Promise<void> {
    return new Promise((resolve, reject) => {
        const child = spawn('npm', ['install'], {
            stdio: 'inherit', // 親プロセスの stdio を継承
        })

        child.on('close', (code) => {
            if (code === 0) {
                resolve()
            } else {
                reject(new Error(`Process exited with code ${code}`))
            }
        })

        child.on('error', (error) => {
            reject(error)
        })
    })
}
```

### パイプライン処理

```typescript
import { spawn } from 'child_process'
import * as fs from 'fs'

// コマンドパイプライン
function runPipeline(): Promise<void> {
    return new Promise((resolve, reject) => {
        // 最初のコマンド
        const grep = spawn('grep', ['-r', 'TODO', './src'])

        // 2番目のコマンド
        const wc = spawn('wc', ['-l'])

        // ファイル出力
        const output = fs.createWriteStream('todo-count.txt')

        // パイプ接続
        grep.stdout.pipe(wc.stdin)
        wc.stdout.pipe(output)

        // エラーハンドリング
        grep.stderr.on('data', (data) => {
            console.error(`grep error: ${data}`)
        })

        wc.stderr.on('data', (data) => {
            console.error(`wc error: ${data}`)
        })

        output.on('finish', () => {
            console.log('Pipeline completed')
            resolve()
        })

        grep.on('error', reject)
        wc.on('error', reject)
    })
}
```

## 非同期処理

### async/awaitによる並行処理

```typescript
import * as fs from 'fs/promises'
import fetch from 'node-fetch'

interface TaskResult {
    id: number
    success: boolean
    data?: any
    error?: string
}

// 並行処理
async function processParallel(items: number[]): Promise<TaskResult[]> {
    const promises = items.map(async (id) => {
        try {
            const response = await fetch(`https://api.example.com/items/${id}`)
            const data = await response.json()

            return {
                id,
                success: true,
                data,
            }
        } catch (error) {
            return {
                id,
                success: false,
                error: error instanceof Error ? error.message : 'Unknown error',
            }
        }
    })

    return Promise.all(promises)
}

// 並行数制限
async function processWithConcurrencyLimit(
    items: number[],
    limit: number
): Promise<TaskResult[]> {
    const results: TaskResult[] = []

    for (let i = 0; i < items.length; i += limit) {
        const chunk = items.slice(i, i + limit)
        const chunkResults = await processParallel(chunk)
        results.push(...chunkResults)

        console.log(`Processed ${i + chunk.length}/${items.length} items`)
    }

    return results
}

// 使用例
const items = Array.from({ length: 100 }, (_, i) => i + 1)
processWithConcurrencyLimit(items, 10)
    .then((results) => {
        const successful = results.filter((r) => r.success).length
        console.log(`Completed: ${successful}/${results.length} successful`)
    })
    .catch(console.error)
```

## エラーハンドリング

### エラークラスと詳細情報

```typescript
// カスタムエラークラス
class ScriptError extends Error {
    constructor(
        message: string,
        public readonly code: string,
        public readonly details?: any
    ) {
        super(message)
        this.name = 'ScriptError'
    }
}

class ValidationError extends ScriptError {
    constructor(field: string, message: string) {
        super(message, 'VALIDATION_ERROR', { field })
        this.name = 'ValidationError'
    }
}

// エラーハンドリング
async function safeExecute<T>(
    fn: () => Promise<T>,
    errorMessage: string
): Promise<T> {
    try {
        return await fn()
    } catch (error) {
        if (error instanceof ScriptError) {
            console.error(`${errorMessage}: ${error.message}`)
            console.error(`Error code: ${error.code}`)
            if (error.details) {
                console.error('Details:', error.details)
            }
        } else if (error instanceof Error) {
            console.error(`${errorMessage}: ${error.message}`)
        } else {
            console.error(`${errorMessage}: Unknown error`)
        }
        throw error
    }
}

// 使用例
async function validateAndProcess(data: any): Promise<void> {
    await safeExecute(async () => {
        if (!data.name) {
            throw new ValidationError('name', 'Name is required')
        }

        // 処理実行
        console.log(`Processing: ${data.name}`)
    }, 'Validation failed')
}
```

## 実用例

### ファイル一括処理スクリプト

```typescript
#!/usr/bin/env ts-node

import * as fs from 'fs/promises'
import * as path from 'path'
import { createHash } from 'crypto'

interface FileInfo {
    path: string
    size: number
    hash: string
}

// ファイルハッシュ計算
async function calculateHash(filePath: string): Promise<string> {
    const content = await fs.readFile(filePath)
    return createHash('sha256').update(content).digest('hex')
}

// ファイル情報収集
async function collectFileInfo(dir: string): Promise<FileInfo[]> {
    const files: FileInfo[] = []
    const entries = await fs.readdir(dir, { withFileTypes: true })

    for (const entry of entries) {
        const fullPath = path.join(dir, entry.name)

        if (entry.isDirectory()) {
            const subFiles = await collectFileInfo(fullPath)
            files.push(...subFiles)
        } else {
            const stats = await fs.stat(fullPath)
            const hash = await calculateHash(fullPath)

            files.push({
                path: fullPath,
                size: stats.size,
                hash,
            })
        }
    }

    return files
}

// レポート生成
async function generateReport(dir: string): Promise<void> {
    console.log('Collecting file information...')
    const files = await collectFileInfo(dir)

    // 統計計算
    const totalSize = files.reduce((sum, file) => sum + file.size, 0)
    const duplicates = findDuplicates(files)

    // レポート出力
    const report = {
        timestamp: new Date().toISOString(),
        directory: dir,
        fileCount: files.length,
        totalSize,
        duplicates: duplicates.length,
        files: files.map((f) => ({
            path: f.path,
            size: f.size,
            hash: f.hash,
        })),
    }

    await fs.writeFile(
        'file-report.json',
        JSON.stringify(report, null, 2),
        'utf-8'
    )

    console.log(`Report generated: ${files.length} files, ${totalSize} bytes`)
    if (duplicates.length > 0) {
        console.log(`Found ${duplicates.length} duplicate files`)
    }
}

// 重複ファイル検出
function findDuplicates(files: FileInfo[]): string[][] {
    const hashMap = new Map<string, string[]>()

    for (const file of files) {
        const paths = hashMap.get(file.hash) || []
        paths.push(file.path)
        hashMap.set(file.hash, paths)
    }

    return Array.from(hashMap.values()).filter((paths) => paths.length > 1)
}

// 実行
const targetDir = process.argv[2] || '.'
generateReport(targetDir).catch((error) => {
    console.error('Error:', error)
    process.exit(1)
})
```

### API呼び出し自動化

```typescript
#!/usr/bin/env ts-node

import fetch from 'node-fetch'
import * as fs from 'fs/promises'

interface ApiConfig {
    baseUrl: string
    apiKey: string
    timeout: number
}

class ApiClient {
    constructor(private config: ApiConfig) {}

    async request<T>(endpoint: string, options: RequestInit = {}): Promise<T> {
        const url = `${this.config.baseUrl}${endpoint}`
        const controller = new AbortController()
        const timeout = setTimeout(() => controller.abort(), this.config.timeout)

        try {
            const response = await fetch(url, {
                ...options,
                headers: {
                    'Authorization': `Bearer ${this.config.apiKey}`,
                    'Content-Type': 'application/json',
                    ...options.headers,
                },
                signal: controller.signal,
            })

            if (!response.ok) {
                throw new Error(`API error: ${response.statusText}`)
            }

            return await response.json() as T
        } finally {
            clearTimeout(timeout)
        }
    }
}

// データ同期スクリプト
async function syncData(): Promise<void> {
    const config: ApiConfig = {
        baseUrl: 'https://api.example.com',
        apiKey: process.env.API_KEY || '',
        timeout: 10000,
    }

    const client = new ApiClient(config)

    try {
        // データ取得
        console.log('Fetching data from API...')
        const data = await client.request<any[]>('/api/items')

        // ローカルファイルに保存
        await fs.writeFile(
            'data.json',
            JSON.stringify(data, null, 2),
            'utf-8'
        )

        console.log(`Synced ${data.length} items`)
    } catch (error) {
        console.error('Sync failed:', error)
        throw error
    }
}

syncData().catch((error) => {
    console.error('Fatal error:', error)
    process.exit(1)
})
```

## Node.jsスクリプトチェックリスト

### 基本
- [ ] TypeScriptで型安全性を確保している
- [ ] シバン（#!/usr/bin/env ts-node）を記述している
- [ ] 適切なエラーハンドリングを実装している

### ファイル操作
- [ ] fs/promisesで非同期処理を使用している
- [ ] パス結合にpath.joinを使用している
- [ ] ファイル存在チェックを実施している

### コマンド実行
- [ ] 用途に応じてexec/spawnを使い分けている
- [ ] エラーコードを適切にハンドリングしている
- [ ] stdioの設定を適切に行っている

### 非同期処理
- [ ] async/awaitで可読性を確保している
- [ ] Promise.allで並行処理を活用している
- [ ] 並行数制限を実装している

### エラーハンドリング
- [ ] カスタムエラークラスを定義している
- [ ] try-catchで適切にエラーを捕捉している
- [ ] エラー情報をログ出力している

## まとめ

本章では、Node.js/TypeScriptによるスクリプト自動化を学びました。

**重要ポイント**:
- TypeScriptの型安全性を活用した堅牢なスクリプト
- fs/promisesで非同期ファイル操作
- child_processで外部コマンド実行
- async/awaitによる並行処理
- 実用的なファイル処理・API連携の実装例

Node.jsの豊富なエコシステムを活用し、効率的な自動化スクリプトを開発してください。

## 参考文献

- [Node.js Documentation](https://nodejs.org/docs/latest/api/)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- [Node.js Best Practices](https://github.com/goldbergyoni/nodebestpractices)
- [ts-node - GitHub](https://github.com/TypeStrong/ts-node)
