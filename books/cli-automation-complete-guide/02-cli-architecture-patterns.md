# CLIアーキテクチャパターン

優れたCLIツールには、明確なアーキテクチャが必要です。本章では、保守性と拡張性を兼ね備えたCLIアーキテクチャパターンを学びます。

## 基本アーキテクチャ

### レイヤー構造

CLIツールを以下の3層に分割することで、関心の分離が実現できます。

```
┌─────────────────────────┐
│   Presentation Layer    │  コマンド定義、引数パース
├─────────────────────────┤
│   Application Layer     │  ビジネスロジック
├─────────────────────────┤
│   Infrastructure Layer  │  ファイルIO、API呼び出し
└─────────────────────────┘
```

### ディレクトリ構造

推奨されるプロジェクト構造:

```
my-cli-tool/
├── src/
│   ├── commands/          # コマンド定義
│   │   ├── create.ts
│   │   ├── list.ts
│   │   └── delete.ts
│   ├── core/             # ビジネスロジック
│   │   ├── project.ts
│   │   ├── template.ts
│   │   └── validator.ts
│   ├── infrastructure/   # 外部依存
│   │   ├── filesystem.ts
│   │   ├── api-client.ts
│   │   └── database.ts
│   ├── utils/           # ユーティリティ
│   │   ├── logger.ts
│   │   ├── config.ts
│   │   └── error.ts
│   └── index.ts         # エントリーポイント
├── tests/
├── templates/
└── package.json
```

## コマンドパターン

### コマンドクラス

各コマンドをクラスとして実装します。

```typescript
// src/commands/base.ts
export abstract class BaseCommand {
  abstract name: string
  abstract description: string

  abstract execute(args: any): Promise<void>
}

// src/commands/create.ts
import { BaseCommand } from './base'

export class CreateCommand extends BaseCommand {
  name = 'create'
  description = 'Create a new project'

  async execute(args: { name: string; template: string }) {
    const project = new Project(args.name, args.template)
    await project.create()
  }
}
```

### コマンドレジストリ

コマンドを一元管理します。

```typescript
// src/commands/registry.ts
import { BaseCommand } from './base'
import { CreateCommand } from './create'
import { ListCommand } from './list'
import { DeleteCommand } from './delete'

export class CommandRegistry {
  private commands = new Map<string, BaseCommand>()

  register(command: BaseCommand): void {
    this.commands.set(command.name, command)
  }

  get(name: string): BaseCommand | undefined {
    return this.commands.get(name)
  }

  all(): BaseCommand[] {
    return Array.from(this.commands.values())
  }
}

// 登録
const registry = new CommandRegistry()
registry.register(new CreateCommand())
registry.register(new ListCommand())
registry.register(new DeleteCommand())
```

## 依存性注入（DI）

### DIコンテナ

依存関係を外部から注入することで、テスト容易性が向上します。

```typescript
// src/di/container.ts
export class Container {
  private services = new Map<string, any>()

  register<T>(key: string, instance: T): void {
    this.services.set(key, instance)
  }

  resolve<T>(key: string): T {
    const service = this.services.get(key)
    if (!service) {
      throw new Error(`Service not found: ${key}`)
    }
    return service
  }
}

// src/index.ts
import { Container } from './di/container'
import { FileSystem } from './infrastructure/filesystem'
import { ApiClient } from './infrastructure/api-client'
import { ProjectService } from './core/project'

const container = new Container()

// インフラストラクチャの登録
container.register('filesystem', new FileSystem())
container.register('apiClient', new ApiClient())

// サービスの登録
container.register('projectService', new ProjectService(
  container.resolve('filesystem'),
  container.resolve('apiClient')
))
```

### インターフェースの活用

抽象に依存することで、実装の切り替えが容易になります。

```typescript
// src/interfaces/storage.ts
export interface IStorage {
  read(path: string): Promise<string>
  write(path: string, content: string): Promise<void>
  exists(path: string): Promise<boolean>
}

// src/infrastructure/filesystem.ts
export class FileSystemStorage implements IStorage {
  async read(path: string): Promise<string> {
    return fs.promises.readFile(path, 'utf-8')
  }

  async write(path: string, content: string): Promise<void> {
    await fs.promises.writeFile(path, content, 'utf-8')
  }

  async exists(path: string): Promise<boolean> {
    try {
      await fs.promises.access(path)
      return true
    } catch {
      return false
    }
  }
}

// src/infrastructure/s3-storage.ts
export class S3Storage implements IStorage {
  constructor(private s3Client: S3Client) {}

  async read(path: string): Promise<string> {
    const result = await this.s3Client.getObject({ Key: path })
    return result.Body.toString()
  }

  // ...その他の実装
}
```

## プラグインアーキテクチャ

### プラグインインターフェース

拡張可能な設計にします。

```typescript
// src/plugins/base.ts
export interface Plugin {
  name: string
  version: string
  initialize(context: PluginContext): Promise<void>
  beforeCommand?(command: string, args: any): Promise<void>
  afterCommand?(command: string, result: any): Promise<void>
}

export interface PluginContext {
  config: Config
  logger: Logger
  storage: IStorage
}

// src/plugins/analytics.ts
export class AnalyticsPlugin implements Plugin {
  name = 'analytics'
  version = '1.0.0'

  async initialize(context: PluginContext): Promise<void> {
    context.logger.info('Analytics plugin initialized')
  }

  async beforeCommand(command: string, args: any): Promise<void> {
    // コマンド実行前の処理
    console.log(`Executing command: ${command}`)
  }

  async afterCommand(command: string, result: any): Promise<void> {
    // コマンド実行後の処理
    console.log(`Command completed: ${command}`)
  }
}
```

### プラグインマネージャー

プラグインのライフサイクルを管理します。

```typescript
// src/plugins/manager.ts
export class PluginManager {
  private plugins: Plugin[] = []

  async register(plugin: Plugin, context: PluginContext): Promise<void> {
    await plugin.initialize(context)
    this.plugins.push(plugin)
  }

  async executeBeforeHooks(command: string, args: any): Promise<void> {
    for (const plugin of this.plugins) {
      if (plugin.beforeCommand) {
        await plugin.beforeCommand(command, args)
      }
    }
  }

  async executeAfterHooks(command: string, result: any): Promise<void> {
    for (const plugin of this.plugins) {
      if (plugin.afterCommand) {
        await plugin.afterCommand(command, result)
      }
    }
  }
}
```

## 設定管理パターン

### Cosmiconfig

複数の設定ファイル形式をサポートします。

```typescript
// src/config/loader.ts
import { cosmiconfig } from 'cosmiconfig'

export interface Config {
  template: string
  port: number
  features: string[]
}

export class ConfigLoader {
  private moduleName: string

  constructor(moduleName: string) {
    this.moduleName = moduleName
  }

  async load(): Promise<Config> {
    const explorer = cosmiconfig(this.moduleName)
    const result = await explorer.search()

    if (!result) {
      return this.getDefaults()
    }

    return this.mergeWithDefaults(result.config)
  }

  private getDefaults(): Config {
    return {
      template: 'default',
      port: 3000,
      features: []
    }
  }

  private mergeWithDefaults(config: Partial<Config>): Config {
    return {
      ...this.getDefaults(),
      ...config
    }
  }
}
```

対応する設定ファイル:

```javascript
// mytool.config.js
module.exports = {
  template: 'react',
  port: 4000,
  features: ['eslint', 'prettier']
}
```

```json
// .mytoolrc
{
  "template": "react",
  "port": 4000,
  "features": ["eslint", "prettier"]
}
```

## ミドルウェアパターン

### ミドルウェアチェーン

コマンド実行前後の処理を追加します。

```typescript
// src/middleware/types.ts
export type Middleware = (
  context: Context,
  next: () => Promise<void>
) => Promise<void>

export interface Context {
  command: string
  args: any
  config: Config
  logger: Logger
}

// src/middleware/logging.ts
export const loggingMiddleware: Middleware = async (context, next) => {
  context.logger.info(`Starting command: ${context.command}`)
  const start = Date.now()

  await next()

  const duration = Date.now() - start
  context.logger.info(`Completed in ${duration}ms`)
}

// src/middleware/validation.ts
export const validationMiddleware: Middleware = async (context, next) => {
  // バリデーション処理
  if (!context.args.name) {
    throw new Error('Name is required')
  }

  await next()
}

// src/middleware/runner.ts
export class MiddlewareRunner {
  private middlewares: Middleware[] = []

  use(middleware: Middleware): void {
    this.middlewares.push(middleware)
  }

  async run(context: Context, handler: () => Promise<void>): Promise<void> {
    let index = 0

    const next = async (): Promise<void> => {
      if (index < this.middlewares.length) {
        const middleware = this.middlewares[index++]
        await middleware(context, next)
      } else {
        await handler()
      }
    }

    await next()
  }
}

// 使用例
const runner = new MiddlewareRunner()
runner.use(loggingMiddleware)
runner.use(validationMiddleware)

await runner.run(context, async () => {
  // コマンド実行
})
```

## エラーハンドリングパターン

### カスタムエラークラス

エラーの種類を明確に区別します。

```typescript
// src/errors/base.ts
export abstract class CLIError extends Error {
  abstract exitCode: number

  constructor(message: string) {
    super(message)
    this.name = this.constructor.name
    Error.captureStackTrace(this, this.constructor)
  }
}

// src/errors/user-error.ts
export class UserError extends CLIError {
  exitCode = 1

  constructor(message: string, public suggestions?: string[]) {
    super(message)
  }
}

// src/errors/system-error.ts
export class SystemError extends CLIError {
  exitCode = 2

  constructor(message: string, public cause?: Error) {
    super(message)
  }
}

// 使用例
throw new UserError(
  'Invalid project name',
  ['Use only lowercase letters and hyphens', 'Example: my-project']
)
```

### グローバルエラーハンドラー

すべてのエラーを一箇所で処理します。

```typescript
// src/utils/error-handler.ts
export class ErrorHandler {
  handle(error: unknown): void {
    if (error instanceof UserError) {
      console.error(`Error: ${error.message}`)
      if (error.suggestions) {
        console.error('\nSuggestions:')
        error.suggestions.forEach(s => console.error(`  - ${s}`))
      }
      process.exit(error.exitCode)
    } else if (error instanceof SystemError) {
      console.error(`System Error: ${error.message}`)
      if (error.cause) {
        console.error(`Cause: ${error.cause.message}`)
      }
      process.exit(error.exitCode)
    } else if (error instanceof Error) {
      console.error(`Unexpected Error: ${error.message}`)
      console.error(error.stack)
      process.exit(1)
    } else {
      console.error('Unknown error occurred')
      process.exit(1)
    }
  }
}

// src/index.ts
const errorHandler = new ErrorHandler()

process.on('uncaughtException', (error) => {
  errorHandler.handle(error)
})

process.on('unhandledRejection', (error) => {
  errorHandler.handle(error)
})
```

## まとめ

本章では、CLIアーキテクチャパターンを学びました。

**重要ポイント**:
- レイヤー構造で関心を分離
- コマンドパターンで拡張性を確保
- 依存性注入でテスト容易性を向上
- プラグインアーキテクチャで柔軟性を実現
- ミドルウェアパターンで横断的関心事を処理
- カスタムエラーで適切なエラーハンドリング

これらのパターンを活用することで、保守性と拡張性の高いCLIツールを構築できます。次章では、引数パースの基礎を学びます。
