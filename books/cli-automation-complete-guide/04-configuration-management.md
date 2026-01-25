---
title: "設定管理"
---

# 設定管理

CLIツールの柔軟性を高めるには、適切な設定管理が不可欠です。本章では、設定ファイル、環境変数、優先順位の制御など、実践的な設定管理手法を学びます。

## 設定の優先順位

設定値は以下の優先順位で適用されます（上が優先）:

```
1. コマンドライン引数       (最優先)
2. 環境変数
3. プロジェクト設定ファイル  (.mytoolrc, mytool.config.js)
4. ユーザー設定ファイル     (~/.mytoolrc)
5. グローバル設定ファイル   (/etc/mytool/config)
6. デフォルト値            (最低優先)
```

### 実装例

```typescript
// src/config/loader.ts
import { join } from 'path'
import { homedir } from 'os'
import { existsSync, readFileSync } from 'fs'

interface Config {
  template: string
  port: number
  verbose: boolean
}

export class ConfigLoader {
  load(options: Partial<Config> = {}): Config {
    const defaults = this.getDefaults()
    const globalConfig = this.loadGlobalConfig()
    const userConfig = this.loadUserConfig()
    const projectConfig = this.loadProjectConfig()
    const envConfig = this.loadEnvConfig()

    // 優先順位に従ってマージ
    return {
      ...defaults,
      ...globalConfig,
      ...userConfig,
      ...projectConfig,
      ...envConfig,
      ...options  // コマンドライン引数が最優先
    }
  }

  private getDefaults(): Config {
    return {
      template: 'default',
      port: 3000,
      verbose: false
    }
  }

  private loadGlobalConfig(): Partial<Config> {
    const path = '/etc/mytool/config.json'
    return this.loadJsonConfig(path)
  }

  private loadUserConfig(): Partial<Config> {
    const path = join(homedir(), '.mytoolrc')
    return this.loadJsonConfig(path)
  }

  private loadProjectConfig(): Partial<Config> {
    // 複数の形式をサポート
    const paths = [
      '.mytoolrc',
      '.mytoolrc.json',
      'mytool.config.js',
      'mytool.config.json'
    ]

    for (const path of paths) {
      if (existsSync(path)) {
        if (path.endsWith('.js')) {
          return require(join(process.cwd(), path))
        }
        return this.loadJsonConfig(path)
      }
    }

    return {}
  }

  private loadEnvConfig(): Partial<Config> {
    return {
      template: process.env.MYTOOL_TEMPLATE,
      port: process.env.MYTOOL_PORT
        ? parseInt(process.env.MYTOOL_PORT, 10)
        : undefined,
      verbose: process.env.MYTOOL_VERBOSE === 'true'
    }
  }

  private loadJsonConfig(path: string): Partial<Config> {
    if (!existsSync(path)) {
      return {}
    }

    try {
      const content = readFileSync(path, 'utf-8')
      return JSON.parse(content)
    } catch (error) {
      console.error(`Failed to load config from ${path}:`, error)
      return {}
    }
  }
}
```

## 設定ファイル形式

### JSON形式

シンプルで扱いやすい形式です。

```json
// .mytoolrc
{
  "template": "react",
  "port": 3000,
  "features": ["eslint", "prettier", "husky"],
  "typescript": true
}
```

### JavaScript形式

動的な設定が可能です。

```javascript
// mytool.config.js
module.exports = {
  template: 'react',
  port: process.env.PORT || 3000,
  features: ['eslint', 'prettier'],

  // 条件分岐
  typescript: process.env.NODE_ENV === 'production',

  // 関数も使用可能
  getOutputDir() {
    return `./dist/${this.template}`
  }
}
```

### YAML形式

階層構造が見やすい形式です。

```yaml
# .mytoolrc.yml
template: react
port: 3000

features:
  - eslint
  - prettier
  - husky

typescript: true

build:
  output: ./dist
  minify: true
```

### TOML形式

設定ファイルに特化した形式です。

```toml
# mytool.config.toml
template = "react"
port = 3000
typescript = true

[build]
output = "./dist"
minify = true

[[features]]
name = "eslint"
enabled = true

[[features]]
name = "prettier"
enabled = true
```

## Cosmiconfigの活用

複数の設定ファイル形式を自動的にサポートします。

```bash
npm install cosmiconfig
```

```typescript
// src/config/cosmiconfig-loader.ts
import { cosmiconfig } from 'cosmiconfig'

export class ConfigLoader {
  private moduleName: string

  constructor(moduleName: string) {
    this.moduleName = moduleName
  }

  async load(): Promise<Config> {
    const explorer = cosmiconfig(this.moduleName, {
      searchPlaces: [
        'package.json',           // package.json の mytool フィールド
        `.${this.moduleName}rc`,
        `.${this.moduleName}rc.json`,
        `.${this.moduleName}rc.yaml`,
        `.${this.moduleName}rc.yml`,
        `.${this.moduleName}rc.js`,
        `.${this.moduleName}rc.cjs`,
        `${this.moduleName}.config.js`,
        `${this.moduleName}.config.cjs`
      ]
    })

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
      verbose: false
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

サポートされる設定ファイル:

```json
// package.json
{
  "name": "my-project",
  "mytool": {
    "template": "react",
    "port": 3000
  }
}
```

```javascript
// .mytoolrc.js
module.exports = {
  template: 'react',
  port: 3000
}
```

## 環境変数

### 命名規則

一貫した命名規則を使用します。

```bash
# プレフィックス + アンダースコア + 大文字
MYTOOL_TEMPLATE=react
MYTOOL_PORT=3000
MYTOOL_VERBOSE=true

# ネストした設定
MYTOOL_BUILD_OUTPUT=./dist
MYTOOL_BUILD_MINIFY=true
```

### .envファイル

dotenvで環境変数を管理します。

```bash
npm install dotenv
```

```
# .env
MYTOOL_TEMPLATE=react
MYTOOL_PORT=3000
MYTOOL_API_KEY=your_api_key_here
MYTOOL_VERBOSE=false
```

```typescript
// src/index.ts
import 'dotenv/config'

// 環境変数が読み込まれる
console.log(process.env.MYTOOL_TEMPLATE)  // "react"
```

### 環境別設定

```bash
# .env.development
MYTOOL_API_URL=http://localhost:3000
MYTOOL_DEBUG=true

# .env.production
MYTOOL_API_URL=https://api.example.com
MYTOOL_DEBUG=false
```

```typescript
// src/config/env-loader.ts
import { config } from 'dotenv'
import { join } from 'path'

export class EnvLoader {
  load(): void {
    const env = process.env.NODE_ENV || 'development'

    // 環境別の.envファイル
    config({ path: join(process.cwd(), `.env.${env}`) })

    // 共通の.envファイル
    config()
  }
}
```

## 設定のバリデーション

### Zodによるバリデーション

```bash
npm install zod
```

```typescript
// src/config/schema.ts
import { z } from 'zod'

export const configSchema = z.object({
  template: z.enum(['default', 'react', 'vue', 'nextjs']),
  port: z.number().int().min(1).max(65535),
  typescript: z.boolean().default(true),
  features: z.array(z.string()).default([]),
  build: z.object({
    output: z.string().default('./dist'),
    minify: z.boolean().default(false)
  }).optional()
})

export type Config = z.infer<typeof configSchema>

// バリデーション
export function validateConfig(data: unknown): Config {
  return configSchema.parse(data)
}
```

使用例:

```typescript
// src/config/loader.ts
import { validateConfig } from './schema'

export class ConfigLoader {
  async load(): Promise<Config> {
    const rawConfig = await this.loadRawConfig()

    try {
      return validateConfig(rawConfig)
    } catch (error) {
      if (error instanceof z.ZodError) {
        console.error('Config validation failed:')
        error.errors.forEach(err => {
          console.error(`  - ${err.path.join('.')}: ${err.message}`)
        })
      }
      throw error
    }
  }
}
```

## 設定の上書き

### マージ戦略

```typescript
// src/config/merger.ts
import merge from 'deepmerge'

export class ConfigMerger {
  merge(...configs: Partial<Config>[]): Config {
    return configs.reduce((acc, config) => {
      return merge(acc, config, {
        arrayMerge: (target, source) => source  // 配列は上書き
      })
    }, {} as Config)
  }
}

// 使用例
const merged = merger.merge(
  defaults,
  userConfig,
  projectConfig,
  cliOptions
)
```

### 部分的な上書き

```typescript
// ドット記法での上書き
mytool build --config.build.output ./out

// 対応する設定マージ
{
  config: {
    build: {
      output: './out'
    }
  }
}
```

## 設定のエクスポート

### 現在の設定を表示

```bash
$ mytool config show

Current configuration:
  template: react
  port: 3000
  typescript: true
  features: [eslint, prettier]

Sources:
  template: .mytoolrc (project)
  port: environment variable (MYTOOL_PORT)
  typescript: default
  features: mytool.config.js (project)
```

### 設定の初期化

```bash
$ mytool config init

Creating configuration file...

? Template: (Use arrow keys)
❯ default
  react
  vue
  nextjs

? Port: (3000)

? Enable TypeScript? (Y/n)

✓ Created .mytoolrc
```

## セキュリティ

### 機密情報の管理

```typescript
// 推奨されないパターン: 設定ファイルに直接記載
{
  "apiKey": "sk_live_xxxxx"  // NG
}

// 推奨されるパターン: 環境変数を使用
{
  "apiKey": "${MYTOOL_API_KEY}"  // 環境変数参照
}
```

### .gitignore

```
# .gitignore
.env
.env.local
.env.*.local
.mytoolrc.local
```

### 変数展開

```typescript
// src/config/expander.ts
export class ConfigExpander {
  expand(config: any): any {
    if (typeof config === 'string') {
      return this.expandString(config)
    }

    if (Array.isArray(config)) {
      return config.map(item => this.expand(item))
    }

    if (typeof config === 'object' && config !== null) {
      const result: any = {}
      for (const [key, value] of Object.entries(config)) {
        result[key] = this.expand(value)
      }
      return result
    }

    return config
  }

  private expandString(str: string): string {
    return str.replace(/\$\{(\w+)\}/g, (_, varName) => {
      return process.env[varName] || ''
    })
  }
}

// 使用例
const config = {
  apiKey: '${MYTOOL_API_KEY}',
  apiUrl: '${MYTOOL_API_URL}'
}

const expanded = new ConfigExpander().expand(config)
// {
//   apiKey: 'actual_api_key',
//   apiUrl: 'https://api.example.com'
// }
```

## まとめ

本章では、設定管理の実践的な手法を学びました。

**重要ポイント**:
- 明確な優先順位を定義する
- 複数の設定ファイル形式をサポートする
- 環境変数で柔軟性を確保する
- Zodなどでバリデーションを行う
- 機密情報は環境変数で管理する
- Cosmiconfigで設定の読み込みを簡素化する

適切な設定管理により、柔軟で使いやすいCLIツールを実現できます。次章では、Node.js CLIのプロジェクトセットアップを学びます。
