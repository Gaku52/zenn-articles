# Node.js CLIテスト

CLIツールの品質を保つには、適切なテストが不可欠です。本章では、Jest を使ったユニットテスト、統合テスト、E2Eテストの実践的な手法を学びます。

## テスト環境セットアップ

### Jest インストール

```bash
npm install -D jest ts-jest @types/jest
npx ts-jest config:init
```

### Jest 設定

```javascript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/tests'],
  testMatch: ['**/__tests__/**/*.ts', '**/?(*.)+(spec|test).ts'],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/index.ts'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  }
}
```

### package.json スクリプト

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:verbose": "jest --verbose"
  }
}
```

## ユニットテスト

### コマンドロジックのテスト

```typescript
// src/core/project.ts
export class Project {
  constructor(private name: string, private template: string) {}

  validate(): boolean {
    return /^[a-z0-9-]+$/.test(this.name)
  }

  getOutputPath(): string {
    return `./${this.name}`
  }
}

// tests/core/project.test.ts
import { Project } from '../../src/core/project'

describe('Project', () => {
  describe('validate', () => {
    it('should return true for valid project names', () => {
      const project = new Project('my-project', 'default')
      expect(project.validate()).toBe(true)
    })

    it('should return false for names with uppercase letters', () => {
      const project = new Project('MyProject', 'default')
      expect(project.validate()).toBe(false)
    })

    it('should return false for names with spaces', () => {
      const project = new Project('my project', 'default')
      expect(project.validate()).toBe(false)
    })

    it('should return false for names with special characters', () => {
      const project = new Project('my_project!', 'default')
      expect(project.validate()).toBe(false)
    })
  })

  describe('getOutputPath', () => {
    it('should return correct output path', () => {
      const project = new Project('myapp', 'default')
      expect(project.getOutputPath()).toBe('./myapp')
    })
  })
})
```

### ユーティリティ関数のテスト

```typescript
// src/utils/validator.ts
export function validateEmail(email: string): boolean {
  const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
  return regex.test(email)
}

export function validatePort(port: number): boolean {
  return Number.isInteger(port) && port >= 1 && port <= 65535
}

// tests/utils/validator.test.ts
import { validateEmail, validatePort } from '../../src/utils/validator'

describe('validateEmail', () => {
  it('should return true for valid emails', () => {
    expect(validateEmail('user@example.com')).toBe(true)
    expect(validateEmail('test.user@domain.co.jp')).toBe(true)
  })

  it('should return false for invalid emails', () => {
    expect(validateEmail('invalid')).toBe(false)
    expect(validateEmail('@example.com')).toBe(false)
    expect(validateEmail('user@')).toBe(false)
  })
})

describe('validatePort', () => {
  it('should return true for valid ports', () => {
    expect(validatePort(3000)).toBe(true)
    expect(validatePort(1)).toBe(true)
    expect(validatePort(65535)).toBe(true)
  })

  it('should return false for invalid ports', () => {
    expect(validatePort(0)).toBe(false)
    expect(validatePort(65536)).toBe(false)
    expect(validatePort(-1)).toBe(false)
    expect(validatePort(3.14)).toBe(false)
  })
})
```

## モックとスパイ

### ファイルシステムのモック

```typescript
// src/infrastructure/filesystem.ts
import fs from 'fs/promises'

export class FileSystem {
  async exists(path: string): Promise<boolean> {
    try {
      await fs.access(path)
      return true
    } catch {
      return false
    }
  }

  async writeFile(path: string, content: string): Promise<void> {
    await fs.writeFile(path, content, 'utf-8')
  }
}

// tests/infrastructure/filesystem.test.ts
import { FileSystem } from '../../src/infrastructure/filesystem'
import fs from 'fs/promises'

jest.mock('fs/promises')

describe('FileSystem', () => {
  let fileSystem: FileSystem

  beforeEach(() => {
    fileSystem = new FileSystem()
    jest.clearAllMocks()
  })

  describe('exists', () => {
    it('should return true if file exists', async () => {
      ;(fs.access as jest.Mock).mockResolvedValue(undefined)

      const result = await fileSystem.exists('/path/to/file')

      expect(result).toBe(true)
      expect(fs.access).toHaveBeenCalledWith('/path/to/file')
    })

    it('should return false if file does not exist', async () => {
      ;(fs.access as jest.Mock).mockRejectedValue(new Error('Not found'))

      const result = await fileSystem.exists('/path/to/file')

      expect(result).toBe(false)
    })
  })

  describe('writeFile', () => {
    it('should write file with correct content', async () => {
      ;(fs.writeFile as jest.Mock).mockResolvedValue(undefined)

      await fileSystem.writeFile('/path/to/file', 'content')

      expect(fs.writeFile).toHaveBeenCalledWith(
        '/path/to/file',
        'content',
        'utf-8'
      )
    })
  })
})
```

### APIクライアントのモック

```typescript
// src/infrastructure/api-client.ts
import axios from 'axios'

export class ApiClient {
  async fetchUser(id: string) {
    const response = await axios.get(`https://api.example.com/users/${id}`)
    return response.data
  }
}

// tests/infrastructure/api-client.test.ts
import { ApiClient } from '../../src/infrastructure/api-client'
import axios from 'axios'

jest.mock('axios')

describe('ApiClient', () => {
  let apiClient: ApiClient

  beforeEach(() => {
    apiClient = new ApiClient()
    jest.clearAllMocks()
  })

  describe('fetchUser', () => {
    it('should fetch user data', async () => {
      const mockUser = { id: '123', name: 'John' }
      ;(axios.get as jest.Mock).mockResolvedValue({ data: mockUser })

      const user = await apiClient.fetchUser('123')

      expect(user).toEqual(mockUser)
      expect(axios.get).toHaveBeenCalledWith('https://api.example.com/users/123')
    })

    it('should throw error when API fails', async () => {
      ;(axios.get as jest.Mock).mockRejectedValue(new Error('Network error'))

      await expect(apiClient.fetchUser('123')).rejects.toThrow('Network error')
    })
  })
})
```

## CLIコマンドのテスト

### Commanderコマンドのテスト

```typescript
// src/commands/create.ts
import { Command } from 'commander'
import { Project } from '../core/project'

export const createCommand = new Command('create')
  .argument('<name>', 'Project name')
  .option('-t, --template <type>', 'Template', 'default')
  .action(async (name, options) => {
    const project = new Project(name, options.template)

    if (!project.validate()) {
      throw new Error('Invalid project name')
    }

    console.log(`Creating project: ${name}`)
    // ... プロジェクト作成処理
  })

// tests/commands/create.test.ts
import { Command } from 'commander'
import { createCommand } from '../../src/commands/create'

describe('create command', () => {
  let program: Command

  beforeEach(() => {
    program = new Command()
    program.addCommand(createCommand)

    // console.log をモック
    jest.spyOn(console, 'log').mockImplementation()
  })

  afterEach(() => {
    jest.restoreAllMocks()
  })

  it('should create project with default template', async () => {
    await program.parseAsync(['node', 'test', 'create', 'myapp'])

    expect(console.log).toHaveBeenCalledWith('Creating project: myapp')
  })

  it('should create project with specified template', async () => {
    await program.parseAsync(['node', 'test', 'create', 'myapp', '--template', 'react'])

    expect(console.log).toHaveBeenCalledWith('Creating project: myapp')
  })

  it('should throw error for invalid project name', async () => {
    await expect(
      program.parseAsync(['node', 'test', 'create', 'My App'])
    ).rejects.toThrow('Invalid project name')
  })
})
```

## 統合テスト

### コマンド全体のテスト

```typescript
// tests/integration/cli.test.ts
import { exec } from 'child_process'
import { promisify } from 'util'
import path from 'path'
import fs from 'fs/promises'

const execAsync = promisify(exec)
const CLI_PATH = path.join(__dirname, '../../dist/index.js')

describe('CLI Integration', () => {
  it('should show help message', async () => {
    const { stdout } = await execAsync(`node ${CLI_PATH} --help`)

    expect(stdout).toContain('Usage:')
    expect(stdout).toContain('Commands:')
  })

  it('should show version', async () => {
    const { stdout } = await execAsync(`node ${CLI_PATH} --version`)

    expect(stdout).toMatch(/\d+\.\d+\.\d+/)
  })

  it('should create project', async () => {
    const projectName = 'test-project'
    const projectPath = path.join(__dirname, projectName)

    try {
      await execAsync(`node ${CLI_PATH} create ${projectName}`)

      // プロジェクトディレクトリが作成されたか確認
      const stat = await fs.stat(projectPath)
      expect(stat.isDirectory()).toBe(true)

      // package.jsonが作成されたか確認
      const packageJson = await fs.readFile(
        path.join(projectPath, 'package.json'),
        'utf-8'
      )
      expect(JSON.parse(packageJson).name).toBe(projectName)
    } finally {
      // クリーンアップ
      await fs.rm(projectPath, { recursive: true, force: true })
    }
  })
})
```

## スナップショットテスト

### 出力のスナップショット

```typescript
// src/commands/info.ts
export function showProjectInfo(project: Project) {
  return `
Project: ${project.name}
Template: ${project.template}
TypeScript: ${project.typescript ? 'Yes' : 'No'}
Features: ${project.features.join(', ')}
`
}

// tests/commands/info.test.ts
import { showProjectInfo } from '../../src/commands/info'
import { Project } from '../../src/core/project'

describe('showProjectInfo', () => {
  it('should match snapshot', () => {
    const project = new Project('myapp', 'react')
    project.typescript = true
    project.features = ['eslint', 'prettier']

    const output = showProjectInfo(project)

    expect(output).toMatchSnapshot()
  })
})
```

## カバレッジ

### カバレッジレポート

```bash
# カバレッジ取得
npm run test:coverage

# 出力例
-----------------|---------|----------|---------|---------|
File             | % Stmts | % Branch | % Funcs | % Lines |
-----------------|---------|----------|---------|---------|
All files        |   95.83 |    91.67 |     100 |   95.65 |
 commands        |     100 |      100 |     100 |     100 |
  create.ts      |     100 |      100 |     100 |     100 |
  delete.ts      |     100 |      100 |     100 |     100 |
 core            |   92.31 |    85.71 |     100 |   91.67 |
  project.ts     |   92.31 |    85.71 |     100 |   91.67 |
-----------------|---------|----------|---------|---------|
```

## まとめ

本章では、Node.js CLIのテスト手法を学びました。

**重要ポイント**:
- ユニットテストでロジックを検証
- モックで外部依存を制御
- 統合テストでコマンド全体をテスト
- スナップショットテストで出力を保護
- カバレッジで品質を可視化

適切なテストにより、信頼性の高いCLIツールを開発できます。次章では、CLIツールの配布方法を学びます。
