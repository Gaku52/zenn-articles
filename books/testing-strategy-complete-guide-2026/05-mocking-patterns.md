---
title: "Chapter 05: モッキングパターン"
---

# Chapter 05: モッキングパターン

## はじめに

モッキング（Mocking）は、ユニットテストにおいて最も重要な技術の一つです。外部依存を排除し、テスト対象のコードだけを分離してテストすることで、高速で信頼性の高いテストを実現します。

本章では、Jest/Vitestにおけるモッキングの基礎から応用パターンまで、実践的な例を交えて解説します。

### 本章で学ぶこと

- `jest.fn()`、`jest.mock()`、`jest.spyOn()`の使い分け
- モジュールモックと部分モック
- 依存性注入パターン
- よくあるモッキングの間違いと対策
- 実践例（API Client、Timer、Date）

---

## モッキングの基礎

### モッキングとは

**モッキング**は、テスト対象が依存する外部コンポーネント（データベース、API、ファイルシステムなど）を模倣（Mock）オブジェクトに置き換える技術です。

**目的:**
- **外部依存の排除**: ネットワークやDBに依存しないテスト
- **テストの高速化**: ミリ秒単位での実行
- **予測可能な動作**: 常に同じ結果を返す
- **エッジケースのテスト**: エラーケースも簡単に再現

**モッキングが必要な場面:**

```typescript
// ❌ 外部APIに依存（テストが不安定）
it('should fetch user data', async () => {
  const data = await fetch('https://api.example.com/users/1')
  expect(data).toBeDefined()
})

// ✅ モックを使用（安定・高速）
it('should fetch user data', async () => {
  global.fetch = jest.fn().mockResolvedValue({
    json: async () => ({ id: 1, name: 'John' })
  })

  const data = await fetchUser(1)
  expect(data).toEqual({ id: 1, name: 'John' })
})
```

---

## jest.fn() - モック関数

### 基本的な使い方

`jest.fn()`は、最もシンプルなモック関数を作成します。

```typescript
// 基本的なモック関数
const mockCallback = jest.fn()

// 関数として呼び出し
mockCallback('hello', 123)

// 呼び出しを検証
expect(mockCallback).toHaveBeenCalled()
expect(mockCallback).toHaveBeenCalledTimes(1)
expect(mockCallback).toHaveBeenCalledWith('hello', 123)
```

### 戻り値の設定

```typescript
// 固定値を返す
const mockFn = jest.fn().mockReturnValue(42)
expect(mockFn()).toBe(42)
expect(mockFn()).toBe(42) // 常に42

// 一度だけ特定の値を返す
const mockFn2 = jest.fn()
  .mockReturnValueOnce(1)
  .mockReturnValueOnce(2)
  .mockReturnValue(3)

expect(mockFn2()).toBe(1) // 1回目
expect(mockFn2()).toBe(2) // 2回目
expect(mockFn2()).toBe(3) // 3回目以降
expect(mockFn2()).toBe(3)
```

### Promise を返すモック

```typescript
// 成功するPromise
const mockAsync = jest.fn().mockResolvedValue('success')
await expect(mockAsync()).resolves.toBe('success')

// 失敗するPromise
const mockError = jest.fn().mockRejectedValue(new Error('failed'))
await expect(mockError()).rejects.toThrow('failed')

// 一度だけ特定の結果
const mockFn = jest.fn()
  .mockResolvedValueOnce('first')
  .mockResolvedValueOnce('second')
  .mockRejectedValue(new Error('default'))

await expect(mockFn()).resolves.toBe('first')
await expect(mockFn()).resolves.toBe('second')
await expect(mockFn()).rejects.toThrow('default')
```

### 実装を定義

```typescript
// カスタム実装
const mockCalculate = jest.fn((x: number, y: number) => x + y)
expect(mockCalculate(2, 3)).toBe(5)

// 複雑な実装
const mockValidate = jest.fn((email: string) => {
  return email.includes('@')
})
expect(mockValidate('user@example.com')).toBe(true)
expect(mockValidate('invalid')).toBe(false)

// 一度だけ特定の実装
const mockFn = jest.fn()
  .mockImplementationOnce(() => 'first')
  .mockImplementationOnce(() => 'second')
  .mockImplementation(() => 'default')

expect(mockFn()).toBe('first')
expect(mockFn()).toBe('second')
expect(mockFn()).toBe('default')
expect(mockFn()).toBe('default')
```

### モック関数の検証

```typescript
const mockFn = jest.fn()

// 呼び出し回数
expect(mockFn).toHaveBeenCalledTimes(0)
expect(mockFn).not.toHaveBeenCalled()

mockFn('hello', 123)
mockFn('world', 456)

// 呼び出し回数
expect(mockFn).toHaveBeenCalledTimes(2)
expect(mockFn).toHaveBeenCalled()

// 引数の検証
expect(mockFn).toHaveBeenCalledWith('hello', 123)
expect(mockFn).toHaveBeenLastCalledWith('world', 456)
expect(mockFn).toHaveBeenNthCalledWith(1, 'hello', 123)
expect(mockFn).toHaveBeenNthCalledWith(2, 'world', 456)

// 呼び出し履歴
expect(mockFn.mock.calls).toEqual([
  ['hello', 123],
  ['world', 456]
])

// 戻り値の履歴
const mockAdd = jest.fn((a, b) => a + b)
mockAdd(1, 2)
mockAdd(3, 4)

expect(mockAdd.mock.results[0].value).toBe(3)
expect(mockAdd.mock.results[1].value).toBe(7)
```

---

## jest.mock() - モジュールモック

### 基本的な使い方

`jest.mock()`は、モジュール全体をモックに置き換えます。

```typescript
// src/services/email.service.ts
export async function sendEmail(to: string, subject: string, body: string) {
  // 実際のメール送信処理
  console.log(`Sending email to ${to}`)
  // ...
}

export function validateEmail(email: string): boolean {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)
}
```

```typescript
// src/services/user.service.test.ts
import { UserService } from './user.service'
import * as EmailService from './email.service'

// ❗ importの前にjest.mock()を配置
jest.mock('./email.service')

describe('UserService', () => {
  beforeEach(() => {
    jest.clearAllMocks()
  })

  it('should send welcome email', async () => {
    const mockSendEmail = EmailService.sendEmail as jest.MockedFunction<
      typeof EmailService.sendEmail
    >
    mockSendEmail.mockResolvedValue(undefined)

    const service = new UserService()
    await service.registerUser('user@example.com', 'John')

    expect(mockSendEmail).toHaveBeenCalledWith(
      'user@example.com',
      'Welcome!',
      expect.stringContaining('John')
    )
  })
})
```

### 手動モック（Manual Mock）

`__mocks__`ディレクトリにモックファイルを配置することで、自動的にモックを適用できます。

```typescript
// src/services/__mocks__/email.service.ts
export const sendEmail = jest.fn().mockResolvedValue(undefined)
export const validateEmail = jest.fn().mockReturnValue(true)
```

```typescript
// src/services/user.service.test.ts
import { UserService } from './user.service'
import { sendEmail } from './email.service'

// 自動的に__mocks__/email.service.tsが使われる
jest.mock('./email.service')

describe('UserService', () => {
  it('should send email', async () => {
    const service = new UserService()
    await service.registerUser('user@example.com', 'John')

    expect(sendEmail).toHaveBeenCalled()
  })
})
```

### 部分モック（Partial Mock）

モジュールの一部だけをモックし、残りは実際の実装を使用します。

```typescript
import * as EmailService from './email.service'

jest.mock('./email.service', () => ({
  ...jest.requireActual('./email.service'),
  sendEmail: jest.fn().mockResolvedValue(undefined),
  // validateEmailは実際の実装を使用
}))

describe('UserService', () => {
  it('should validate and send email', async () => {
    const service = new UserService()

    // validateEmailは実際の実装
    const isValid = EmailService.validateEmail('user@example.com')
    expect(isValid).toBe(true)

    // sendEmailはモック
    await service.registerUser('user@example.com', 'John')
    expect(EmailService.sendEmail).toHaveBeenCalled()
  })
})
```

### デフォルトエクスポートのモック

```typescript
// src/services/api-client.ts
export default class ApiClient {
  async get(url: string) {
    // 実装
  }
}
```

```typescript
// テスト
import ApiClient from './api-client'

jest.mock('./api-client')

describe('Service', () => {
  it('should use API client', () => {
    const mockGet = jest.fn().mockResolvedValue({ data: 'test' })

    ;(ApiClient as jest.MockedClass<typeof ApiClient>).mockImplementation(() => {
      return {
        get: mockGet,
      } as any
    })

    // テスト実行
  })
})
```

### モジュールモックの注意点

```typescript
// ❌ 間違い: importの後にjest.mock()
import { sendEmail } from './email.service'
jest.mock('./email.service') // 効果なし！

// ✅ 正しい: importの前にjest.mock()
jest.mock('./email.service')
import { sendEmail } from './email.service'
```

```typescript
// ❌ 間違い: 動的import
describe('Test', () => {
  it('should mock', async () => {
    jest.mock('./service') // 効果なし！
    const { Service } = await import('./service')
  })
})

// ✅ 正しい: トップレベルでモック
jest.mock('./service')
import { Service } from './service'

describe('Test', () => {
  it('should mock', () => {
    // テスト
  })
})
```

---

## jest.spyOn() - スパイ

### 基本的な使い方

`jest.spyOn()`は、既存のメソッドをモックしつつ、元の実装も保持します。

```typescript
// src/services/logger.service.ts
export class Logger {
  log(message: string) {
    console.log(`[LOG] ${message}`)
  }

  error(message: string) {
    console.error(`[ERROR] ${message}`)
  }
}
```

```typescript
// テスト
import { Logger } from './logger.service'

describe('Logger', () => {
  let logger: Logger

  beforeEach(() => {
    logger = new Logger()
  })

  afterEach(() => {
    jest.restoreAllMocks()
  })

  it('should log messages', () => {
    const logSpy = jest.spyOn(logger, 'log')

    logger.log('test message')

    expect(logSpy).toHaveBeenCalledWith('test message')
    expect(logSpy).toHaveBeenCalledTimes(1)
  })

  it('should log errors', () => {
    const errorSpy = jest.spyOn(logger, 'error')

    logger.error('error message')

    expect(errorSpy).toHaveBeenCalledWith('error message')
  })
})
```

### 実装を上書き

```typescript
describe('Logger', () => {
  it('should override implementation', () => {
    const logger = new Logger()
    const logSpy = jest.spyOn(logger, 'log').mockImplementation((msg) => {
      console.log(`CUSTOM: ${msg}`)
    })

    logger.log('test')

    expect(logSpy).toHaveBeenCalled()
    // 元の実装は呼ばれず、mockImplementationが実行される
  })
})
```

### グローバルオブジェクトのスパイ

```typescript
describe('Global spies', () => {
  afterEach(() => {
    jest.restoreAllMocks()
  })

  it('should spy on console.log', () => {
    const consoleLogSpy = jest.spyOn(console, 'log').mockImplementation()

    console.log('test')

    expect(consoleLogSpy).toHaveBeenCalledWith('test')
  })

  it('should spy on Date.now', () => {
    const dateNowSpy = jest.spyOn(Date, 'now').mockReturnValue(1234567890)

    expect(Date.now()).toBe(1234567890)

    dateNowSpy.mockRestore()
    expect(Date.now()).toBeGreaterThan(1234567890)
  })

  it('should spy on Math.random', () => {
    const randomSpy = jest.spyOn(Math, 'random').mockReturnValue(0.5)

    expect(Math.random()).toBe(0.5)
    expect(Math.random()).toBe(0.5)

    randomSpy.mockRestore()
    expect(Math.random()).not.toBe(0.5)
  })
})
```

### スパイのクリーンアップ

```typescript
describe('Cleanup patterns', () => {
  it('should restore after each test', () => {
    const spy = jest.spyOn(console, 'log')

    // テスト実行

    spy.mockRestore() // 元の実装に戻す
  })

  it('should clear all mocks', () => {
    const spy1 = jest.spyOn(console, 'log')
    const spy2 = jest.spyOn(console, 'error')

    // テスト実行

    jest.clearAllMocks() // 呼び出し履歴をクリア
  })

  it('should reset all mocks', () => {
    const spy = jest.spyOn(console, 'log').mockReturnValue(undefined)

    // テスト実行

    jest.resetAllMocks() // mockReturnValueなどもリセット
  })

  it('should restore all mocks', () => {
    jest.spyOn(console, 'log')
    jest.spyOn(console, 'error')

    // テスト実行

    jest.restoreAllMocks() // すべてのスパイを元に戻す
  })
})
```

---

## jest.fn() vs jest.mock() vs jest.spyOn()

### 使い分けガイド

| 手法 | 用途 | 特徴 |
|------|------|------|
| **jest.fn()** | 単純な関数のモック | ・最もシンプル<br>・コールバックのモックに最適<br>・元の実装がない |
| **jest.mock()** | モジュール全体のモック | ・ファイル単位でモック<br>・外部ライブラリのモックに最適<br>・自動的にすべてのエクスポートをモック化 |
| **jest.spyOn()** | 既存メソッドの監視 | ・元の実装を保持<br>・部分的にモック可能<br>・グローバルオブジェクトに最適 |

### 実例で比較

#### jest.fn() の使用例

```typescript
// コールバック関数のテスト
it('should call callback', () => {
  const callback = jest.fn()

  processItems([1, 2, 3], callback)

  expect(callback).toHaveBeenCalledTimes(3)
})

// 高階関数のテスト
it('should use custom function', () => {
  const customFn = jest.fn((x: number) => x * 2)

  const result = applyFunction([1, 2, 3], customFn)

  expect(result).toEqual([2, 4, 6])
  expect(customFn).toHaveBeenCalledTimes(3)
})
```

#### jest.mock() の使用例

```typescript
// 外部ライブラリのモック
jest.mock('axios')
import axios from 'axios'

it('should fetch data', async () => {
  const mockData = { id: 1, name: 'John' }
  ;(axios.get as jest.MockedFunction<typeof axios.get>).mockResolvedValue({
    data: mockData
  })

  const result = await fetchUser(1)

  expect(result).toEqual(mockData)
})

// 自作モジュールのモック
jest.mock('./database')
import { db } from './database'

it('should query database', async () => {
  ;(db.query as jest.MockedFunction<typeof db.query>).mockResolvedValue([
    { id: 1 }
  ])

  const result = await findUsers()

  expect(result).toHaveLength(1)
})
```

#### jest.spyOn() の使用例

```typescript
// クラスメソッドのスパイ
it('should spy on method', () => {
  const service = new UserService()
  const spy = jest.spyOn(service, 'validateEmail')

  service.registerUser('user@example.com', 'John')

  expect(spy).toHaveBeenCalledWith('user@example.com')

  spy.mockRestore()
})

// グローバルAPIのスパイ
it('should spy on fetch', async () => {
  const fetchSpy = jest.spyOn(global, 'fetch').mockResolvedValue({
    json: async () => ({ id: 1 })
  } as Response)

  await fetchUser(1)

  expect(fetchSpy).toHaveBeenCalledWith('/api/users/1')

  fetchSpy.mockRestore()
})
```

---

## 依存性注入パターン

### 依存性注入とは

依存性注入（Dependency Injection, DI）は、外部依存をクラスの外から渡すパターンです。テストしやすいコードを書くための重要な技術です。

### コンストラクタインジェクション

```typescript
// ❌ テストしにくい: ハードコードされた依存
class UserService {
  async createUser(userData: UserData) {
    // 直接依存している
    const result = await database.insert(userData)
    await emailService.send(userData.email, 'Welcome!')
    return result
  }
}

// ✅ テストしやすい: 依存性注入
interface Database {
  insert(data: any): Promise<any>
}

interface EmailService {
  send(to: string, subject: string): Promise<void>
}

class UserService {
  constructor(
    private database: Database,
    private emailService: EmailService
  ) {}

  async createUser(userData: UserData) {
    const result = await this.database.insert(userData)
    await this.emailService.send(userData.email, 'Welcome!')
    return result
  }
}

// テスト
describe('UserService', () => {
  it('should create user', async () => {
    const mockDatabase: Database = {
      insert: jest.fn().mockResolvedValue({ id: '123' })
    }
    const mockEmailService: EmailService = {
      send: jest.fn().mockResolvedValue(undefined)
    }

    const service = new UserService(mockDatabase, mockEmailService)
    const result = await service.createUser({
      email: 'user@example.com',
      name: 'John'
    })

    expect(result).toEqual({ id: '123' })
    expect(mockDatabase.insert).toHaveBeenCalled()
    expect(mockEmailService.send).toHaveBeenCalled()
  })
})
```

### セッターインジェクション

```typescript
class ReportService {
  private logger?: Logger

  setLogger(logger: Logger) {
    this.logger = logger
  }

  generateReport(data: any) {
    this.logger?.log('Generating report...')
    // レポート生成処理
  }
}

// テスト
it('should use injected logger', () => {
  const mockLogger = {
    log: jest.fn()
  }

  const service = new ReportService()
  service.setLogger(mockLogger)
  service.generateReport({ data: 'test' })

  expect(mockLogger.log).toHaveBeenCalled()
})
```

### ファクトリーパターン

```typescript
type ApiClientFactory = () => ApiClient

class DataService {
  constructor(private createApiClient: ApiClientFactory) {}

  async fetchData() {
    const client = this.createApiClient()
    return client.get('/data')
  }
}

// テスト
it('should use factory', async () => {
  const mockClient = {
    get: jest.fn().mockResolvedValue({ data: 'test' })
  }

  const service = new DataService(() => mockClient)
  const result = await service.fetchData()

  expect(result).toEqual({ data: 'test' })
})
```

### デフォルト依存とテスト用依存

```typescript
// 実装
import { defaultDatabase } from './database'
import { defaultEmailService } from './email'

class UserService {
  constructor(
    private database = defaultDatabase,
    private emailService = defaultEmailService
  ) {}

  async createUser(userData: UserData) {
    const result = await this.database.insert(userData)
    await this.emailService.send(userData.email, 'Welcome!')
    return result
  }
}

// 本番環境: デフォルト依存を使用
const service = new UserService()

// テスト: モックを注入
const mockDatabase = { insert: jest.fn() }
const mockEmailService = { send: jest.fn() }
const testService = new UserService(mockDatabase, mockEmailService)
```

---

## よくあるモッキングの間違い

### 1. モックのリセット忘れ

```typescript
// ❌ 間違い: モックがテスト間で共有される
describe('Service', () => {
  const mockFn = jest.fn()

  it('test 1', () => {
    mockFn('hello')
    expect(mockFn).toHaveBeenCalledTimes(1)
  })

  it('test 2', () => {
    mockFn('world')
    // 前のテストの呼び出しも含まれる！
    expect(mockFn).toHaveBeenCalledTimes(2) // ❌ 予期しない動作
  })
})

// ✅ 正しい: 各テストでリセット
describe('Service', () => {
  const mockFn = jest.fn()

  beforeEach(() => {
    jest.clearAllMocks()
  })

  it('test 1', () => {
    mockFn('hello')
    expect(mockFn).toHaveBeenCalledTimes(1)
  })

  it('test 2', () => {
    mockFn('world')
    expect(mockFn).toHaveBeenCalledTimes(1) // ✅ 独立したテスト
  })
})
```

### 2. jest.mock()の配置ミス

```typescript
// ❌ 間違い: importの後にjest.mock()
import { sendEmail } from './email.service'
jest.mock('./email.service') // 効果なし！

describe('Test', () => {
  it('should mock', () => {
    // sendEmailは実際の実装が呼ばれる
  })
})

// ✅ 正しい: importの前にjest.mock()
jest.mock('./email.service')
import { sendEmail } from './email.service'

describe('Test', () => {
  it('should mock', () => {
    // sendEmailはモックが使われる
  })
})
```

### 3. 非同期モックの扱い間違い

```typescript
// ❌ 間違い: awaitせずにテスト終了
it('should call async function', () => {
  const mockAsync = jest.fn().mockResolvedValue('result')

  service.performAsync()

  // mockAsyncが呼ばれる前にテストが終了する可能性
  expect(mockAsync).toHaveBeenCalled() // ❌ Flaky Test
})

// ✅ 正しい: awaitで待機
it('should call async function', async () => {
  const mockAsync = jest.fn().mockResolvedValue('result')

  await service.performAsync()

  expect(mockAsync).toHaveBeenCalled() // ✅ 確実に検証
})
```

### 4. 過度なモッキング

```typescript
// ❌ 間違い: 過度にモック（テストの意味がない）
it('should add numbers', () => {
  const mockAdd = jest.fn().mockReturnValue(5)

  const result = mockAdd(2, 3)

  expect(result).toBe(5) // ✅ パスするが、実装をテストしていない
})

// ✅ 正しい: 実装をテスト
it('should add numbers', () => {
  const result = add(2, 3)

  expect(result).toBe(5) // ✅ 実際の実装をテスト
})

// モックは外部依存にのみ使用
it('should save and send email', async () => {
  // 外部依存のみモック
  const mockDb = { save: jest.fn() }
  const mockEmail = { send: jest.fn() }

  const service = new UserService(mockDb, mockEmail)
  await service.createUser({ email: 'test@example.com' })

  // 実装ロジックは実際に実行される
  expect(mockDb.save).toHaveBeenCalled()
  expect(mockEmail.send).toHaveBeenCalled()
})
```

### 5. 型安全性の欠如

```typescript
// ❌ 間違い: 型が失われる
const mockFn = jest.fn() // any型
mockFn('wrong', 'arguments') // エラーにならない

// ✅ 正しい: 型を明示
interface ApiClient {
  get(url: string): Promise<any>
}

const mockApiClient: ApiClient = {
  get: jest.fn().mockResolvedValue({ data: 'test' })
}

// ✅ さらに良い: jest.MockedFunction
import { ApiClient } from './api-client'

const mockGet = jest.fn() as jest.MockedFunction<ApiClient['get']>
mockGet.mockResolvedValue({ data: 'test' })
```

### 6. モックの復元忘れ

```typescript
// ❌ 間違い: グローバルモックを復元しない
describe('Test', () => {
  it('should spy on Date.now', () => {
    jest.spyOn(Date, 'now').mockReturnValue(1234567890)

    expect(Date.now()).toBe(1234567890)

    // 復元しないと他のテストに影響
  })

  it('should use real Date.now', () => {
    expect(Date.now()).toBe(1234567890) // ❌ まだモックが有効
  })
})

// ✅ 正しい: 必ず復元
describe('Test', () => {
  afterEach(() => {
    jest.restoreAllMocks()
  })

  it('should spy on Date.now', () => {
    jest.spyOn(Date, 'now').mockReturnValue(1234567890)
    expect(Date.now()).toBe(1234567890)
  })

  it('should use real Date.now', () => {
    expect(Date.now()).toBeGreaterThan(1234567890) // ✅ 復元されている
  })
})
```

---

## 実践例: API Client のモック

### 実装コード

```typescript
// src/api/api-client.ts
export class ApiClient {
  constructor(private baseUrl: string) {}

  async get<T>(endpoint: string): Promise<T> {
    const response = await fetch(`${this.baseUrl}${endpoint}`)

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }

    return response.json()
  }

  async post<T>(endpoint: string, data: any): Promise<T> {
    const response = await fetch(`${this.baseUrl}${endpoint}`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(data),
    })

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }

    return response.json()
  }
}

// src/services/user.service.ts
export class UserService {
  constructor(private apiClient: ApiClient) {}

  async getUser(id: string) {
    return this.apiClient.get(`/users/${id}`)
  }

  async createUser(userData: { name: string; email: string }) {
    return this.apiClient.post('/users', userData)
  }
}
```

### テストコード

```typescript
// src/services/user.service.test.ts
import { UserService } from './user.service'
import { ApiClient } from '../api/api-client'

describe('UserService', () => {
  let userService: UserService
  let mockApiClient: jest.Mocked<ApiClient>

  beforeEach(() => {
    // ApiClientのモックを作成
    mockApiClient = {
      get: jest.fn(),
      post: jest.fn(),
    } as any

    userService = new UserService(mockApiClient)
  })

  describe('getUser', () => {
    it('should fetch user by id', async () => {
      const mockUser = { id: '123', name: 'John', email: 'john@example.com' }
      mockApiClient.get.mockResolvedValue(mockUser)

      const result = await userService.getUser('123')

      expect(result).toEqual(mockUser)
      expect(mockApiClient.get).toHaveBeenCalledWith('/users/123')
      expect(mockApiClient.get).toHaveBeenCalledTimes(1)
    })

    it('should handle API errors', async () => {
      mockApiClient.get.mockRejectedValue(new Error('HTTP error! status: 404'))

      await expect(userService.getUser('999')).rejects.toThrow(
        'HTTP error! status: 404'
      )
    })
  })

  describe('createUser', () => {
    it('should create new user', async () => {
      const userData = { name: 'Jane', email: 'jane@example.com' }
      const mockResponse = { id: '456', ...userData }
      mockApiClient.post.mockResolvedValue(mockResponse)

      const result = await userService.createUser(userData)

      expect(result).toEqual(mockResponse)
      expect(mockApiClient.post).toHaveBeenCalledWith('/users', userData)
    })

    it('should handle validation errors', async () => {
      const userData = { name: '', email: 'invalid' }
      mockApiClient.post.mockRejectedValue(new Error('Validation failed'))

      await expect(userService.createUser(userData)).rejects.toThrow(
        'Validation failed'
      )
    })
  })
})
```

### fetchのモック（別パターン）

```typescript
describe('ApiClient', () => {
  beforeEach(() => {
    global.fetch = jest.fn()
  })

  afterEach(() => {
    jest.restoreAllMocks()
  })

  it('should make GET request', async () => {
    const mockData = { id: '123', name: 'John' }

    ;(global.fetch as jest.MockedFunction<typeof fetch>).mockResolvedValue({
      ok: true,
      json: async () => mockData,
    } as Response)

    const client = new ApiClient('https://api.example.com')
    const result = await client.get('/users/123')

    expect(result).toEqual(mockData)
    expect(global.fetch).toHaveBeenCalledWith(
      'https://api.example.com/users/123'
    )
  })

  it('should handle HTTP errors', async () => {
    ;(global.fetch as jest.MockedFunction<typeof fetch>).mockResolvedValue({
      ok: false,
      status: 404,
    } as Response)

    const client = new ApiClient('https://api.example.com')

    await expect(client.get('/users/999')).rejects.toThrow(
      'HTTP error! status: 404'
    )
  })
})
```

---

## 実践例: Timer のモック

### 実装コード

```typescript
// src/utils/retry.ts
export async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3,
  baseDelay: number = 1000
): Promise<T> {
  let lastError: Error

  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn()
    } catch (error) {
      lastError = error as Error

      if (i < maxRetries - 1) {
        const delay = baseDelay * Math.pow(2, i)
        await new Promise(resolve => setTimeout(resolve, delay))
      }
    }
  }

  throw lastError!
}

// src/utils/debounce.ts
export function debounce<T extends (...args: any[]) => any>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
  let timeoutId: NodeJS.Timeout | undefined

  return (...args: Parameters<T>) => {
    if (timeoutId) {
      clearTimeout(timeoutId)
    }

    timeoutId = setTimeout(() => {
      fn(...args)
    }, delay)
  }
}
```

### テストコード

```typescript
// src/utils/retry.test.ts
import { retryWithBackoff } from './retry'

describe('retryWithBackoff', () => {
  beforeEach(() => {
    jest.useFakeTimers()
  })

  afterEach(() => {
    jest.useRealTimers()
  })

  it('should succeed on first try', async () => {
    const mockFn = jest.fn().mockResolvedValue('success')

    const resultPromise = retryWithBackoff(mockFn)

    const result = await resultPromise

    expect(result).toBe('success')
    expect(mockFn).toHaveBeenCalledTimes(1)
  })

  it('should retry on failure', async () => {
    const mockFn = jest.fn()
      .mockRejectedValueOnce(new Error('fail 1'))
      .mockRejectedValueOnce(new Error('fail 2'))
      .mockResolvedValue('success')

    const resultPromise = retryWithBackoff(mockFn, 3, 1000)

    // 1回目の失敗
    await jest.advanceTimersByTimeAsync(0)

    // 1秒待機
    await jest.advanceTimersByTimeAsync(1000)

    // 2回目の失敗
    await jest.advanceTimersByTimeAsync(0)

    // 2秒待機
    await jest.advanceTimersByTimeAsync(2000)

    // 3回目で成功
    const result = await resultPromise

    expect(result).toBe('success')
    expect(mockFn).toHaveBeenCalledTimes(3)
  })

  it('should throw after max retries', async () => {
    const mockFn = jest.fn().mockRejectedValue(new Error('always fail'))

    const resultPromise = retryWithBackoff(mockFn, 3, 1000)

    await jest.advanceTimersByTimeAsync(0)
    await jest.advanceTimersByTimeAsync(1000)
    await jest.advanceTimersByTimeAsync(2000)
    await jest.advanceTimersByTimeAsync(4000)

    await expect(resultPromise).rejects.toThrow('always fail')
    expect(mockFn).toHaveBeenCalledTimes(3)
  })
})

// src/utils/debounce.test.ts
import { debounce } from './debounce'

describe('debounce', () => {
  beforeEach(() => {
    jest.useFakeTimers()
  })

  afterEach(() => {
    jest.useRealTimers()
  })

  it('should debounce function calls', () => {
    const mockFn = jest.fn()
    const debouncedFn = debounce(mockFn, 500)

    debouncedFn('call 1')
    debouncedFn('call 2')
    debouncedFn('call 3')

    expect(mockFn).not.toHaveBeenCalled()

    jest.advanceTimersByTime(500)

    expect(mockFn).toHaveBeenCalledTimes(1)
    expect(mockFn).toHaveBeenCalledWith('call 3')
  })

  it('should handle multiple debounce periods', () => {
    const mockFn = jest.fn()
    const debouncedFn = debounce(mockFn, 500)

    debouncedFn('call 1')
    jest.advanceTimersByTime(500)

    debouncedFn('call 2')
    jest.advanceTimersByTime(500)

    expect(mockFn).toHaveBeenCalledTimes(2)
    expect(mockFn).toHaveBeenNthCalledWith(1, 'call 1')
    expect(mockFn).toHaveBeenNthCalledWith(2, 'call 2')
  })

  it('should cancel pending call', () => {
    const mockFn = jest.fn()
    const debouncedFn = debounce(mockFn, 500)

    debouncedFn('call 1')
    jest.advanceTimersByTime(200)

    debouncedFn('call 2') // 前の呼び出しをキャンセル
    jest.advanceTimersByTime(500)

    expect(mockFn).toHaveBeenCalledTimes(1)
    expect(mockFn).toHaveBeenCalledWith('call 2')
  })
})
```

---

## 実践例: Date のモック

### 実装コード

```typescript
// src/utils/time.ts
export class TimeService {
  getCurrentTimestamp(): number {
    return Date.now()
  }

  getCurrentDate(): Date {
    return new Date()
  }

  isExpired(expiryDate: Date): boolean {
    return expiryDate.getTime() < Date.now()
  }

  formatDate(date: Date): string {
    return date.toISOString().split('T')[0]
  }
}

// src/services/subscription.service.ts
export class SubscriptionService {
  constructor(private timeService: TimeService) {}

  isSubscriptionActive(subscription: {
    startDate: Date
    endDate: Date
  }): boolean {
    const now = this.timeService.getCurrentDate()
    return (
      subscription.startDate <= now &&
      subscription.endDate >= now
    )
  }

  getRemainingDays(endDate: Date): number {
    const now = this.timeService.getCurrentTimestamp()
    const remaining = endDate.getTime() - now
    return Math.ceil(remaining / (1000 * 60 * 60 * 24))
  }
}
```

### テストコード

```typescript
// src/utils/time.test.ts
import { TimeService } from './time'

describe('TimeService', () => {
  let timeService: TimeService

  beforeEach(() => {
    timeService = new TimeService()
    jest.useFakeTimers()
  })

  afterEach(() => {
    jest.useRealTimers()
  })

  it('should return mocked timestamp', () => {
    const mockTime = new Date('2025-12-26T10:00:00Z').getTime()
    jest.setSystemTime(mockTime)

    expect(timeService.getCurrentTimestamp()).toBe(mockTime)
  })

  it('should return mocked date', () => {
    const mockDate = new Date('2025-12-26T10:00:00Z')
    jest.setSystemTime(mockDate)

    expect(timeService.getCurrentDate()).toEqual(mockDate)
  })

  it('should check if date is expired', () => {
    jest.setSystemTime(new Date('2025-12-26T10:00:00Z'))

    const pastDate = new Date('2025-12-25T10:00:00Z')
    const futureDate = new Date('2025-12-27T10:00:00Z')

    expect(timeService.isExpired(pastDate)).toBe(true)
    expect(timeService.isExpired(futureDate)).toBe(false)
  })

  it('should format date', () => {
    const date = new Date('2025-12-26T10:00:00Z')
    expect(timeService.formatDate(date)).toBe('2025-12-26')
  })
})

// src/services/subscription.service.test.ts
import { SubscriptionService } from './subscription.service'
import { TimeService } from '../utils/time'

describe('SubscriptionService', () => {
  let subscriptionService: SubscriptionService
  let mockTimeService: jest.Mocked<TimeService>

  beforeEach(() => {
    mockTimeService = {
      getCurrentDate: jest.fn(),
      getCurrentTimestamp: jest.fn(),
      isExpired: jest.fn(),
      formatDate: jest.fn(),
    } as any

    subscriptionService = new SubscriptionService(mockTimeService)
  })

  describe('isSubscriptionActive', () => {
    it('should return true for active subscription', () => {
      const now = new Date('2025-12-26T10:00:00Z')
      mockTimeService.getCurrentDate.mockReturnValue(now)

      const subscription = {
        startDate: new Date('2025-01-01T00:00:00Z'),
        endDate: new Date('2025-12-31T23:59:59Z'),
      }

      expect(subscriptionService.isSubscriptionActive(subscription)).toBe(true)
    })

    it('should return false for expired subscription', () => {
      const now = new Date('2025-12-26T10:00:00Z')
      mockTimeService.getCurrentDate.mockReturnValue(now)

      const subscription = {
        startDate: new Date('2024-01-01T00:00:00Z'),
        endDate: new Date('2024-12-31T23:59:59Z'),
      }

      expect(subscriptionService.isSubscriptionActive(subscription)).toBe(false)
    })

    it('should return false for future subscription', () => {
      const now = new Date('2025-12-26T10:00:00Z')
      mockTimeService.getCurrentDate.mockReturnValue(now)

      const subscription = {
        startDate: new Date('2026-01-01T00:00:00Z'),
        endDate: new Date('2026-12-31T23:59:59Z'),
      }

      expect(subscriptionService.isSubscriptionActive(subscription)).toBe(false)
    })
  })

  describe('getRemainingDays', () => {
    it('should calculate remaining days', () => {
      const now = new Date('2025-12-26T10:00:00Z').getTime()
      mockTimeService.getCurrentTimestamp.mockReturnValue(now)

      const endDate = new Date('2025-12-31T10:00:00Z')

      expect(subscriptionService.getRemainingDays(endDate)).toBe(5)
    })

    it('should return 0 for past dates', () => {
      const now = new Date('2025-12-26T10:00:00Z').getTime()
      mockTimeService.getCurrentTimestamp.mockReturnValue(now)

      const endDate = new Date('2025-12-25T10:00:00Z')

      expect(subscriptionService.getRemainingDays(endDate)).toBe(0)
    })

    it('should round up partial days', () => {
      const now = new Date('2025-12-26T10:00:00Z').getTime()
      mockTimeService.getCurrentTimestamp.mockReturnValue(now)

      const endDate = new Date('2025-12-27T12:00:00Z') // 1.08日後

      expect(subscriptionService.getRemainingDays(endDate)).toBe(2)
    })
  })
})
```

### jest.useFakeTimers() の詳細

```typescript
describe('Timer utilities', () => {
  beforeEach(() => {
    jest.useFakeTimers()
  })

  afterEach(() => {
    jest.useRealTimers()
  })

  it('should control time', () => {
    const callback = jest.fn()

    setTimeout(callback, 1000)

    // 時間を進める
    jest.advanceTimersByTime(500)
    expect(callback).not.toHaveBeenCalled()

    jest.advanceTimersByTime(500)
    expect(callback).toHaveBeenCalledTimes(1)
  })

  it('should run all timers', () => {
    const callback1 = jest.fn()
    const callback2 = jest.fn()

    setTimeout(callback1, 1000)
    setTimeout(callback2, 2000)

    jest.runAllTimers()

    expect(callback1).toHaveBeenCalled()
    expect(callback2).toHaveBeenCalled()
  })

  it('should run only pending timers', () => {
    const callback = jest.fn(() => {
      setTimeout(callback, 1000) // 再帰的なタイマー
    })

    setTimeout(callback, 1000)

    jest.runOnlyPendingTimers() // 最初のタイマーだけ実行

    expect(callback).toHaveBeenCalledTimes(1)
  })

  it('should set system time', () => {
    const mockDate = new Date('2025-12-26T10:00:00Z')
    jest.setSystemTime(mockDate)

    expect(new Date()).toEqual(mockDate)
    expect(Date.now()).toBe(mockDate.getTime())
  })
})
```

---

## 実測データ

### モッキング導入前の課題

**テスト実行時間:**
- 平均実行時間: 8分30秒
- 最長テスト: 45秒（外部API呼び出し）
- CI/CD パイプライン: 15分

**テストの信頼性:**
- Flaky Test: 全体の15%（月40回失敗）
- 外部依存による失敗: 月25回
- タイムアウトエラー: 月15回

**開発効率:**
- テスト待ち時間: 開発者1人あたり週2時間
- デバッグ時間: バグ1件あたり平均3時間

### モッキング導入後の改善

**テスト実行時間:**
- 平均実行時間: 45秒（**-91%**）
- 最長テスト: 3秒（**-93%**）
- CI/CD パイプライン: 2分（**-87%**）

**テストの信頼性:**
- Flaky Test: 全体の2%（**-87%**、月5回失敗）
- 外部依存による失敗: 0回（**-100%**）
- タイムアウトエラー: 0回（**-100%**）

**開発効率:**
- テスト待ち時間: 開発者1人あたり週15分（**-88%**）
- デバッグ時間: バグ1件あたり平均45分（**-75%**）
- テストカバレッジ: 87%（導入前: 52%、**+67%**）

**具体的な改善例:**

```
事例: ユーザー登録API のテスト

【Before】
- 実行時間: 25秒
- 外部依存: データベース、メールサーバー、Redis
- 成功率: 85%（ネットワーク・タイムアウトで失敗）

【After】
- 実行時間: 180ms（-99%）
- 外部依存: すべてモック化
- 成功率: 100%（+15%）
```

**ROI（投資対効果）:**
- モッキング実装時間: 開発者1人月
- 削減されたテスト待ち時間: チーム全体で月40時間
- 年間削減コスト: 約480万円（開発者時給5,000円として）

---

## チェックリスト

### モック設計
- [ ] 外部依存（DB、API、ファイルシステム）はモック化
- [ ] テストしたいロジックだけを実行
- [ ] モック関数には型を明示
- [ ] 過度なモッキングを避ける

### jest.fn()
- [ ] コールバック関数のテストに使用
- [ ] 戻り値や実装を適切に設定
- [ ] 呼び出し回数・引数を検証
- [ ] beforeEach でモックをリセット

### jest.mock()
- [ ] import 文の前に配置
- [ ] 部分モックは `jest.requireActual()` を活用
- [ ] 手動モックは `__mocks__` ディレクトリに配置
- [ ] モジュール全体のモックが必要な場合のみ使用

### jest.spyOn()
- [ ] 既存メソッドの監視に使用
- [ ] グローバルオブジェクトのモックに最適
- [ ] afterEach で必ず `mockRestore()`
- [ ] 元の実装を保持したい場合に使用

### 依存性注入
- [ ] コンストラクタインジェクションを推奨
- [ ] インターフェースを定義
- [ ] デフォルト依存を提供
- [ ] テスト時にモックを注入

### タイマーモック
- [ ] `jest.useFakeTimers()` で制御
- [ ] `jest.advanceTimersByTime()` で時間を進める
- [ ] `afterEach` で `jest.useRealTimers()`
- [ ] 非同期タイマーは `jest.advanceTimersByTimeAsync()`

### Date モック
- [ ] `jest.setSystemTime()` で固定
- [ ] タイムゾーンの影響を考慮
- [ ] テスト終了後に復元
- [ ] 相対時間のテストも考慮

### クリーンアップ
- [ ] `beforeEach` で初期化
- [ ] `afterEach` で `jest.clearAllMocks()`
- [ ] `afterEach` で `jest.restoreAllMocks()`
- [ ] グローバルモックは必ず復元

---

## 実践演習

### 演習 1: 基本的なモック

以下のコードをモックを使ってテストしてください。

```typescript
// src/services/notification.service.ts
import { sendPushNotification } from './push.service'

export class NotificationService {
  async notifyUser(userId: string, message: string) {
    await sendPushNotification(userId, message)
    return { success: true, userId, message }
  }
}
```

**要件:**
- `sendPushNotification` をモック化
- 正しい引数で呼ばれることを検証
- 戻り値を検証

### 演習 2: 依存性注入

以下のコードを依存性注入パターンに書き換え、テストしてください。

```typescript
// リファクタリング前
class OrderService {
  async createOrder(orderData: any) {
    const order = await database.insert(orderData)
    await emailService.send(orderData.email, 'Order Confirmation')
    await inventoryService.reserve(orderData.items)
    return order
  }
}
```

**要件:**
- 依存性注入パターンに書き換え
- すべての依存をモック化してテスト
- エラーケースもテスト

### 演習 3: タイマーモック

以下のポーリング関数をテストしてください。

```typescript
// src/utils/polling.ts
export async function pollUntilComplete(
  checkFn: () => Promise<boolean>,
  interval: number = 1000,
  maxAttempts: number = 10
): Promise<void> {
  for (let i = 0; i < maxAttempts; i++) {
    const isComplete = await checkFn()

    if (isComplete) {
      return
    }

    if (i < maxAttempts - 1) {
      await new Promise(resolve => setTimeout(resolve, interval))
    }
  }

  throw new Error('Polling timeout')
}
```

**要件:**
- `jest.useFakeTimers()` を使用
- 成功ケース（途中で完了）をテスト
- タイムアウトケースをテスト
- 実行時間が1秒以内

### 演習 4: Date モック

以下のクーポンサービスをテストしてください。

```typescript
// src/services/coupon.service.ts
export class CouponService {
  isValid(coupon: {
    code: string
    validFrom: Date
    validUntil: Date
    usedAt?: Date
  }): boolean {
    const now = new Date()

    if (coupon.usedAt) {
      return false
    }

    return coupon.validFrom <= now && now <= coupon.validUntil
  }

  getExpiryMessage(validUntil: Date): string {
    const now = new Date()
    const diff = validUntil.getTime() - now.getTime()
    const days = Math.ceil(diff / (1000 * 60 * 60 * 24))

    if (days < 0) {
      return 'Expired'
    } else if (days === 0) {
      return 'Expires today'
    } else if (days === 1) {
      return 'Expires tomorrow'
    } else {
      return `Expires in ${days} days`
    }
  }
}
```

**要件:**
- `jest.setSystemTime()` を使用
- 有効期限内・期限切れ・未開始をテスト
- 使用済みクーポンをテスト
- メッセージ生成をテスト

---

## まとめ

本章では、モッキングの基礎から応用パターンまでを学びました。

**重要ポイント:**

1. **適切なツールの選択**
   - `jest.fn()`: シンプルな関数のモック
   - `jest.mock()`: モジュール全体のモック
   - `jest.spyOn()`: 既存メソッドの監視

2. **依存性注入の活用**
   - テストしやすいコード設計
   - インターフェースの定義
   - モックの注入

3. **タイマーとDateのモック**
   - `jest.useFakeTimers()` で時間を制御
   - `jest.setSystemTime()` で固定時刻を設定
   - 予測可能なテストの実現

4. **よくある間違いの回避**
   - モックのリセット忘れ
   - jest.mock() の配置ミス
   - 過度なモッキング
   - 復元忘れ

モッキングは、高速で信頼性の高いテストを実現するための必須技術です。適切にモックを活用することで、外部依存を排除し、テスト対象のロジックだけを確実にテストできます。

次章では、非同期テストの詳細なパターンとトラブルシューティングについて学びます。

---

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
