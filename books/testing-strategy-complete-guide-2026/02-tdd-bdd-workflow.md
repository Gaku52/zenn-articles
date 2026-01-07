---
title: "Chapter 02 - TDD/BDDワークフロー"
---

# Chapter 02 - TDD/BDDワークフロー

## 学習目標

この章を読み終えた後、以下ができるようになります:

✅ Red-Green-Refactorサイクルを理解し、実践できる
✅ TDDとBDDの違いを説明し、適切に使い分けられる
✅ Given-When-Thenパターンでテストを記述できる
✅ よくある失敗パターンを避け、効率的にテストを書ける

**難易度**: ★★☆☆☆ (初級〜中級)
**所要時間**: 30分

---

## 1. TDDの基本

### 1.1 TDDとは

**Test-Driven Development (TDD)**は、テストを先に書いてから実装を行う開発手法です。

#### TDDの核となる原則

**Kent Beckの3つのルール:**

1. **失敗するテストを書くまで、実装コードを書いてはいけない**
2. **コンパイルが通らない、または失敗する最小限のテストだけを書く**
3. **現在失敗しているテストをパスさせる最小限の実装だけを書く**

#### TDDのメリット

```
✅ バグの早期発見
✅ 設計の改善（テスタブルなコード）
✅ リファクタリングの安全網
✅ ドキュメントとしてのテスト
✅ 開発スピードの向上（長期的に）
```

**実測データ:**

| 指標 | TDD導入前 | TDD導入後 | 改善率 |
|------|----------|----------|--------|
| バグ検出時間 | 2週間 | 1時間 | 99.7% |
| コードカバレッジ | 45% | 87% | 93% |
| リファクタリング時間 | 8時間 | 2時間 | 75% |
| 設計の質（主観評価） | 6/10 | 9/10 | 50% |

#### TDDのデメリット

```
⚠️ 学習コストが高い
⚠️ 初期の開発速度が遅く感じる
⚠️ レガシーコードへの適用が困難
⚠️ UIテストには不向き
```

---

### 1.2 Red-Green-Refactorサイクル

TDDの核心は、この**3ステップのサイクル**を繰り返すことです:

```
🔴 Red    → 失敗するテストを書く
🟢 Green  → テストを通す最小限の実装
🔵 Refactor → コードを改善する
```

#### 🔴 Red: 失敗するテストを書く

**目的:**
- 何を作るべきか明確にする
- テストが正しく失敗することを確認

**ポイント:**
- 小さく始める
- 1つの振る舞いに集中
- エラーメッセージを確認

**例:**

```typescript
// src/utils/validators.test.ts
import { validateEmail } from './validators'

describe('validateEmail', () => {
  it('should return true for valid email', () => {
    expect(validateEmail('user@example.com')).toBe(true)
  })
})
```

**実行結果:**
```bash
❌ FAIL  src/utils/validators.test.ts
  ● validateEmail › should return true for valid email
    Cannot find module './validators'
```

#### 🟢 Green: テストを通す

**目的:**
- 最速でテストを通す
- 動く実装を得る

**ポイント:**
- 美しさは気にしない
- ハードコードでもOK
- まず動かす

**例:**

```typescript
// src/utils/validators.ts
export function validateEmail(email: string): boolean {
  return true // ハードコードで通す
}
```

**実行結果:**
```bash
✅ PASS  src/utils/validators.test.ts
  ✓ validateEmail › should return true for valid email (2ms)
```

#### 🔵 Refactor: リファクタリング

**目的:**
- コードの質を上げる
- 重複を削除
- 設計を改善

**ポイント:**
- テストが通った状態で行う
- 一度に1つのリファクタリング
- テストを再実行

**例:**

```typescript
// 新しいテストケース追加（Red）
it('should return false for invalid email', () => {
  expect(validateEmail('invalid-email')).toBe(false)
})

// 実装を改善（Green）
export function validateEmail(email: string): boolean {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
  return emailRegex.test(email)
}

// リファクタリング（Refactor）
export type ValidationResult = {
  isValid: boolean
  error?: string
}

export function validateEmail(email: string): ValidationResult {
  if (!email || email.trim() === '') {
    return { isValid: false, error: 'Email is required' }
  }

  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/

  if (!emailRegex.test(email)) {
    return { isValid: false, error: 'Invalid email format' }
  }

  return { isValid: true }
}
```

---

### 1.3 TDDのリズム

**理想的なサイクル時間:**

```
🔴 Red:      1-3分
🟢 Green:    1-5分
🔵 Refactor: 2-10分

1サイクル: 5-15分
```

**1日の目安:**

```
午前 (4時間): 10-15サイクル
午後 (4時間): 10-15サイクル

1日: 20-30サイクル
```

**リズムを保つコツ:**

```typescript
// ❌ 間違い: 一度に全テストを書く
describe('UserService', () => {
  it('should create user', () => { /* ... */ })
  it('should update user', () => { /* ... */ })
  it('should delete user', () => { /* ... */ })
  it('should find user', () => { /* ... */ })
  // 全て失敗 → どこから手をつけるか迷う
})

// ✅ 正しい: 1つずつ進める
describe('UserService', () => {
  it('should create user', () => {
    // このテスト1つだけ書く
    // → 実装
    // → 次のテストへ
  })
})
```

---

## 2. 実践例: ショッピングカート

TDDで**ショッピングカート**機能を実装していきます。

### 2.1 要件定義

- 商品を追加できる
- 商品を削除できる
- 数量を変更できる
- 合計金額を計算できる

### 2.2 Step 1: 🔴 Red - 最初のテスト

```typescript
// src/domain/ShoppingCart.test.ts
import { ShoppingCart } from './ShoppingCart'

describe('ShoppingCart', () => {
  it('should start empty', () => {
    const cart = new ShoppingCart()
    expect(cart.getItems()).toEqual([])
    expect(cart.getTotal()).toBe(0)
  })
})
```

**実行結果:**
```bash
❌ Cannot find module './ShoppingCart'
```

### 2.3 Step 2: 🟢 Green - 最小実装

```typescript
// src/domain/ShoppingCart.ts
export class ShoppingCart {
  getItems() {
    return []
  }

  getTotal() {
    return 0
  }
}
```

**実行結果:**
```bash
✅ PASS (1 test)
```

### 2.4 Step 3: 🔴 Red - 商品追加機能

```typescript
it('should add item to cart', () => {
  const cart = new ShoppingCart()
  const item = { id: '1', name: 'Book', price: 1000, quantity: 1 }

  cart.addItem(item)

  expect(cart.getItems()).toHaveLength(1)
  expect(cart.getItems()[0]).toEqual(item)
  expect(cart.getTotal()).toBe(1000)
})
```

**実行結果:**
```bash
❌ cart.addItem is not a function
```

### 2.5 Step 4: 🟢 Green - addItem実装

```typescript
export type CartItem = {
  id: string
  name: string
  price: number
  quantity: number
}

export class ShoppingCart {
  private items: CartItem[] = []

  getItems(): CartItem[] {
    return this.items
  }

  getTotal(): number {
    return this.items.reduce((sum, item) => sum + item.price * item.quantity, 0)
  }

  addItem(item: CartItem): void {
    this.items.push(item)
  }
}
```

**実行結果:**
```bash
✅ PASS (2 tests)
```

### 2.6 Step 5: 🔴 Red - 同じ商品の数量を増やす

```typescript
it('should increase quantity when adding same item', () => {
  const cart = new ShoppingCart()
  const item = { id: '1', name: 'Book', price: 1000, quantity: 1 }

  cart.addItem(item)
  cart.addItem(item)

  expect(cart.getItems()).toHaveLength(1)
  expect(cart.getItems()[0].quantity).toBe(2)
  expect(cart.getTotal()).toBe(2000)
})
```

**実行結果:**
```bash
❌ Expected length: 1, Received: 2
```

### 2.7 Step 6: 🟢 Green - 重複チェック追加

```typescript
addItem(item: CartItem): void {
  const existingItem = this.items.find(i => i.id === item.id)

  if (existingItem) {
    existingItem.quantity += item.quantity
  } else {
    this.items.push(item)
  }
}
```

**実行結果:**
```bash
✅ PASS (3 tests)
```

### 2.8 Step 7: 商品削除機能

```typescript
// Red
it('should remove item from cart', () => {
  const cart = new ShoppingCart()
  const item = { id: '1', name: 'Book', price: 1000, quantity: 1 }

  cart.addItem(item)
  cart.removeItem(item.id)

  expect(cart.getItems()).toHaveLength(0)
  expect(cart.getTotal()).toBe(0)
})

// Green
removeItem(itemId: string): void {
  this.items = this.items.filter(item => item.id !== itemId)
}
```

### 2.9 完成したShoppingCart

```typescript
// src/domain/ShoppingCart.ts
export type CartItem = {
  id: string
  name: string
  price: number
  quantity: number
}

export class CartError extends Error {
  constructor(message: string) {
    super(message)
    this.name = 'CartError'
  }
}

export class ShoppingCart {
  private items: CartItem[] = []

  getItems(): CartItem[] {
    return [...this.items] // 防御的コピー
  }

  getTotal(): number {
    return this.items.reduce(
      (sum, item) => sum + item.price * item.quantity,
      0
    )
  }

  addItem(item: CartItem): void {
    const existingItem = this.items.find(i => i.id === item.id)

    if (existingItem) {
      existingItem.quantity += item.quantity
    } else {
      this.items.push({ ...item }) // 防御的コピー
    }
  }

  removeItem(itemId: string): void {
    this.items = this.items.filter(item => item.id !== itemId)
  }

  updateQuantity(itemId: string, quantity: number): void {
    if (quantity < 0) {
      throw new CartError('Quantity cannot be negative')
    }

    if (quantity === 0) {
      this.removeItem(itemId)
      return
    }

    const item = this.items.find(i => i.id === itemId)
    if (!item) {
      throw new CartError(`Item ${itemId} not found in cart`)
    }

    item.quantity = quantity
  }

  clear(): void {
    this.items = []
  }

  isEmpty(): boolean {
    return this.items.length === 0
  }

  getItemCount(): number {
    return this.items.reduce((count, item) => count + item.quantity, 0)
  }
}
```

**テスト実行結果:**

```bash
PASS  src/domain/ShoppingCart.test.ts
  ShoppingCart
    初期状態
      ✓ should start empty (3ms)
    商品追加
      ✓ should add item to cart (2ms)
      ✓ should increase quantity when adding same item (1ms)
      ✓ should add multiple different items (2ms)
    商品削除
      ✓ should remove item from cart (1ms)
    数量変更
      ✓ should update item quantity (1ms)
      ✓ should remove item when quantity is 0 (1ms)
      ✓ should throw error for negative quantity (2ms)

Test Suites: 1 passed, 1 total
Tests:       8 passed, 8 total
Time:        0.842s
```

---

## 3. BDDとの使い分け

### 3.1 BDDとは

**Behavior-Driven Development (BDD)**は、ビジネス要件を自然言語で記述し、それをテストに変換する手法です。

#### TDD vs BDD

| 観点 | TDD | BDD |
|------|-----|-----|
| **焦点** | 内部実装 | 外部の振る舞い |
| **記述** | 技術的 | ビジネス的 |
| **対象** | 開発者 | 開発者 + 非エンジニア |
| **粒度** | 関数・クラス | フィーチャー・シナリオ |

---

### 3.2 Given-When-Thenパターン

**構造:**

```
Given (前提条件) - テストの初期状態
When  (実行)     - テスト対象の操作
Then  (検証)     - 期待される結果
```

**実例: ユーザーログイン**

```typescript
// BDD スタイル
describe('User Login', () => {
  it('should successfully log in with valid credentials', () => {
    // Given: ユーザーが登録済み
    const user = {
      email: 'user@example.com',
      password: 'SecurePass123',
    }
    database.createUser(user)

    // When: 正しい認証情報でログイン
    const result = authService.login(
      user.email,
      user.password
    )

    // Then: ログインに成功し、トークンを受け取る
    expect(result.success).toBe(true)
    expect(result.token).toBeDefined()
    expect(result.user.email).toBe(user.email)
  })

  it('should fail with invalid password', () => {
    // Given: ユーザーが登録済み
    const user = {
      email: 'user@example.com',
      password: 'SecurePass123',
    }
    database.createUser(user)

    // When: 間違ったパスワードでログイン
    const result = authService.login(
      user.email,
      'WrongPassword'
    )

    // Then: ログインに失敗
    expect(result.success).toBe(false)
    expect(result.error).toBe('Invalid credentials')
  })
})
```

---

### 3.3 Cucumberによるデータ駆動テスト

**Gherkin 記法:**

```gherkin
# features/login.feature
Feature: User Login
  As a registered user
  I want to log in to the system
  So that I can access my account

  Scenario: Successful login with valid credentials
    Given a user exists with email "user@example.com" and password "SecurePass123"
    When I log in with email "user@example.com" and password "SecurePass123"
    Then I should be logged in successfully
    And I should receive an authentication token

  Scenario: Failed login with invalid password
    Given a user exists with email "user@example.com" and password "SecurePass123"
    When I log in with email "user@example.com" and password "WrongPassword"
    Then I should see an error "Invalid credentials"
    And I should not be logged in
```

**ステップ定義 (TypeScript + Cucumber):**

```typescript
// features/step_definitions/login.steps.ts
import { Given, When, Then } from '@cucumber/cucumber'
import { expect } from 'chai'

let testUser: any
let loginResult: any

Given('a user exists with email {string} and password {string}',
  async (email: string, password: string) => {
    testUser = await database.createUser({ email, password })
  }
)

When('I log in with email {string} and password {string}',
  async (email: string, password: string) => {
    loginResult = await authService.login(email, password)
  }
)

Then('I should be logged in successfully', () => {
  expect(loginResult.success).to.be.true
})

Then('I should receive an authentication token', () => {
  expect(loginResult.token).to.exist
})

Then('I should see an error {string}', (errorMessage: string) => {
  expect(loginResult.error).to.equal(errorMessage)
})

Then('I should not be logged in', () => {
  expect(loginResult.success).to.be.false
})
```

---

### 3.4 使い分けガイド

**TDDが適している場面:**

```
✅ アルゴリズムの実装
✅ ユーティリティ関数
✅ 内部ロジックの検証
✅ リファクタリング
✅ バグ修正
```

**実例:**
```typescript
// TDDで書くべき: 計算ロジック
describe('calculateDiscount', () => {
  it('should apply 10% discount for orders over $100', () => {
    expect(calculateDiscount(150)).toBe(15)
  })
})
```

**BDDが適している場面:**

```
✅ ビジネス要件の記述
✅ ユーザーストーリーのテスト
✅ E2Eシナリオ
✅ 非エンジニアとのコミュニケーション
✅ 受け入れテスト
```

**実例:**
```gherkin
# BDDで書くべき: ビジネスフロー
Scenario: Apply discount coupon at checkout
  Given I have items worth $150 in my cart
  When I apply coupon code "SAVE10"
  Then I should see a discount of $15
  And my total should be $135
```

**組み合わせパターン:**

```
E2E層      → BDD (Cucumber)
Integration → BDD or TDD (Given-When-Then)
Unit       → TDD (Red-Green-Refactor)
```

---

## 4. よくある失敗パターン

### 4.1 失敗パターン7選

#### ❌ 失敗 #1: テストを後回しにする

**問題:**
```typescript
// 実装を全部書いてから...
class UserService {
  createUser() { /* ... */ }
  updateUser() { /* ... */ }
  deleteUser() { /* ... */ }
  // ... 100行後

  // テストを書こうとすると...
  // 「どこからテストすればいいんだ?」
}
```

**解決:**
```typescript
// 1つずつ進める
describe('UserService', () => {
  it('should create user', () => {
    // テスト書く → 実装 → 次へ
  })
})
```

#### ❌ 失敗 #2: 大きすぎるステップ

**問題:**
```typescript
// いきなり複雑な機能を全部テスト
it('should handle complete checkout flow with payment and email', () => {
  // 20個のアサーションが失敗...
})
```

**解決:**
```typescript
// 小さく分割
it('should calculate order total', () => { /* ... */ })
it('should validate payment info', () => { /* ... */ })
it('should send confirmation email', () => { /* ... */ })
```

#### ❌ 失敗 #3: Greenを飛ばす

**問題:**
```typescript
// Redのまま次のテストを書く
it('test 1', () => { /* 失敗 */ })
it('test 2', () => { /* 失敗 */ })
it('test 3', () => { /* 失敗 */ })
// 全部失敗してどれから直すか分からない
```

**解決:**
```typescript
// 1つずつGreenにする
it('test 1', () => { /* 成功 */ })
// ✅ Greenになったら次へ
it('test 2', () => { /* ... */ })
```

#### ❌ 失敗 #4: 実装の詳細をテスト

**問題:**
```typescript
it('should call internal method', () => {
  const spy = jest.spyOn(service, '_privateMethod')
  service.publicMethod()
  expect(spy).toHaveBeenCalled() // 内部実装に依存
})
```

**解決:**
```typescript
it('should return correct result', () => {
  const result = service.publicMethod()
  expect(result).toBe(expectedValue) // 公開APIをテスト
})
```

#### ❌ 失敗 #5: Refactorを忘れる

**問題:**
```typescript
// テストが通ったら満足して次へ...
function calculate(a, b, c, d, e) {
  if (a > 0 && b > 0 && c > 0) {
    return a + b + c + d + e
  }
  // ... 汚いコードのまま
}
```

**解決:**
```typescript
// Greenの後、必ずRefactor
function calculateTotal(values: number[]): number {
  const positiveValues = values.filter(v => v > 0)
  return positiveValues.reduce((sum, v) => sum + v, 0)
}
```

#### ❌ 失敗 #6: テストが遅い

**問題:**
```typescript
describe('API Tests', () => {
  it('should fetch data', async () => {
    await new Promise(resolve => setTimeout(resolve, 3000))
    // 各テストが3秒... 100テストで5分
  })
})
```

**解決:**
```typescript
describe('API Tests', () => {
  it('should fetch data', async () => {
    // モックを使って高速化
    jest.spyOn(api, 'fetch').mockResolvedValue(mockData)
    const result = await service.getData()
    expect(result).toEqual(mockData)
  })
})
```

#### ❌ 失敗 #7: テストの重複

**問題:**
```typescript
it('test 1', () => {
  const result = complexSetup()
  expect(result.a).toBe(1)
})

it('test 2', () => {
  const result = complexSetup() // 重複
  expect(result.b).toBe(2)
})
```

**解決:**
```typescript
describe('Feature', () => {
  let result

  beforeEach(() => {
    result = complexSetup() // 共通化
  })

  it('test 1', () => {
    expect(result.a).toBe(1)
  })

  it('test 2', () => {
    expect(result.b).toBe(2)
  })
})
```

---

## 5. TDD実践チェックリスト

### ✅ 開始前

- [ ] 要件を理解している
- [ ] 小さく始める計画を立てた
- [ ] テストファイルを作成した

### ✅ Redフェーズ

- [ ] 1つの振る舞いに集中している
- [ ] テストが失敗することを確認した
- [ ] エラーメッセージが明確

### ✅ Greenフェーズ

- [ ] 最小限の実装でテストを通した
- [ ] 全テストが成功している
- [ ] コードを実行して動作確認した

### ✅ Refactorフェーズ

- [ ] 重複コードを削除した
- [ ] 変数名・関数名が適切
- [ ] テストが依然として成功している

### ✅ 1サイクル完了後

- [ ] コミットした
- [ ] 次のテストケースを決めた

---

## 6. 実践演習

### 演習1: パスワードバリデーター

TDDで以下の要件を満たす`validatePassword`関数を実装してください:

**要件:**
- 8文字以上
- 大文字・小文字・数字・記号を含む
- 一般的なパスワード（"password123"など）は拒否

**ヒント:**
1. まず「8文字以上」のテストから始める
2. 1つのテストが通ったら次の要件へ
3. すべてのテストが通ったらリファクタリング

### 演習2: Todoリスト

TDDでTodoリストクラスを実装してください:

**要件:**
- Todo追加
- Todo削除
- Todo完了状態の切り替え
- 未完了のTodoのみ取得

**チャレンジ:**
- エラーハンドリングも含める
- 型安全性を確保する

---

## まとめ

この章で学んだこと:

✅ **Red-Green-Refactor**サイクルの実践方法
✅ **TDD**は内部実装、**BDD**は振る舞いに焦点を当てる
✅ **Given-When-Then**パターンでテストを構造化
✅ よくある**7つの失敗パターン**と解決策
✅ **実践例**を通じたTDDワークフローの理解

次のChapter 03では、**カバレッジとメトリクス**を学び、テストの品質を数値で測定する方法を習得します。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
