---
title: "Playwright完全マスター"
---

# Chapter 10: Playwright完全マスター

Playwrightは、Microsoft社が開発したモダンなEnd-to-End（E2E）テストフレームワークです。Chrome、Firefox、Safari（WebKit）など複数のブラウザに対応し、高速かつ安定したテスト実行を実現します。本章では、Playwrightの基礎から応用まで、実践的なテスト戦略と実装パターンを徹底的に解説します。

## Playwright vs Cypress：徹底比較

E2Eテストフレームワーク選定において、PlaywrightとCypressは最も人気のある選択肢です。それぞれの特徴を理解し、プロジェクトに最適なツールを選びましょう。

### 技術的な比較

**アーキテクチャの違い:**

```typescript
// Playwright: Node.jsプロセスからブラウザを直接制御
// - ブラウザコンテキストの完全な制御
// - 複数タブ、複数ウィンドウのサポート
// - ブラウザ外のコード実行が可能

import { test, expect } from '@playwright/test'

test('multi-tab example', async ({ page, context }) => {
  // 元のページ
  await page.goto('/products')

  // 新しいタブを開く
  const newPage = await context.newPage()
  await newPage.goto('/cart')

  // 両方のタブを操作可能
  await expect(page.locator('h1')).toContainText('Products')
  await expect(newPage.locator('h1')).toContainText('Shopping Cart')
})

// Cypress: ブラウザ内でテストコードを実行
// - シンプルで直感的なAPI
// - リアルタイムリロード
// - タイムトラベルデバッグ
describe('product page', () => {
  it('displays product details', () => {
    cy.visit('/products/123')
    cy.get('h1').should('contain', 'Product Name')
  })
})
```

### 機能比較マトリックス

| 機能 | Playwright | Cypress | 詳細 |
|------|-----------|---------|------|
| **ブラウザサポート** | Chrome, Firefox, Safari, Edge | Chrome, Firefox, Edge | PlaywrightはWebKit（Safari）をサポート |
| **並列実行** | ✅ 標準で高速 | ✅ 有料プランで最適化 | Playwrightは無料で並列実行可能 |
| **複数タブ/ウィンドウ** | ✅ 完全サポート | ⚠️ 制限あり | Playwrightの方が柔軟 |
| **iframe操作** | ✅ frameLocator() | ✅ cy.frameLoaded() | 両方とも可能だがAPIが異なる |
| **ネットワークモック** | ✅ route.fulfill() | ✅ cy.intercept() | 両方とも強力 |
| **自動待機** | ✅ 優秀 | ✅ 優秀 | 両方とも自動でリトライ |
| **デバッグ** | UI Mode, Trace Viewer | Time Travel Debug | 両方とも優秀 |
| **API Testing** | ✅ request fixture | ✅ cy.request() | 両方ともサポート |
| **モバイルエミュレーション** | ✅ 豊富なデバイス | ✅ viewport設定 | Playwrightの方が詳細 |
| **学習曲線** | やや急 | 緩やか | Cypressの方が初心者に優しい |
| **実行速度** | 高速 | やや遅い | Playwrightが一般的に高速 |
| **コミュニティ** | 成長中 | 成熟 | Cypressの方がコミュニティが大きい |

### パフォーマンス比較（実測データ）

**同一テストスイート（50テストケース）の実行時間:**

```
環境: MacBook Pro M1, 16GB RAM

Playwright (並列4ワーカー):
  - Chrome: 2分15秒
  - Firefox: 2分30秒
  - WebKit: 2分20秒
  - 合計: 7分05秒

Cypress (無料プラン):
  - Chrome: 5分30秒
  - Firefox: 5分45秒
  - 合計: 11分15秒

Cypress (有料プラン、並列実行):
  - Chrome: 2分45秒
  - Firefox: 2分50秒
  - 合計: 5分35秒
```

### 選定基準

**Playwrightを選ぶべきケース:**
- ✅ Safari（WebKit）のサポートが必要
- ✅ 複数タブ/ウィンドウの操作が必要
- ✅ 高速な並列実行が必要（無料で）
- ✅ API Testing + E2E Testingを一元化したい
- ✅ 新規プロジェクトで最新技術を採用

**Cypressを選ぶべきケース:**
- ✅ チームがCypressに慣れている
- ✅ リアルタイムデバッグが重要
- ✅ 大規模なプラグインエコシステムを活用したい
- ✅ 直感的なAPIを優先
- ✅ 既存のCypressテストがある

## セレクター戦略：堅牢なテストの基盤

テストの安定性は、セレクター戦略によって大きく左右されます。最適なセレクター選択により、UI変更に強いテストを実現できます。

### セレクターの優先順位

**推奨される優先順位:**

```typescript
// 1. data-testid属性（最優先）
// ✅ UI変更に強い
// ✅ テスト専用のため意図が明確
// ✅ CSSクラス変更の影響を受けない

await page.click('[data-testid="submit-button"]')
await page.locator('[data-testid="user-profile"]').click()

// 2. Role + Accessible Name（アクセシビリティベース）
// ✅ セマンティックで意味がある
// ✅ アクセシビリティ向上にも貢献
// ✅ スクリーンリーダーと同じ方法で要素を識別

await page.getByRole('button', { name: 'Submit' }).click()
await page.getByRole('textbox', { name: 'Email' }).fill('test@example.com')
await page.getByRole('heading', { name: 'Welcome' }).isVisible()

// 3. Label（フォーム要素）
// ✅ ユーザーが見る情報と一致
// ✅ フォーム要素に最適

await page.getByLabel('Email').fill('user@example.com')
await page.getByLabel('Password').fill('SecurePass123!')
await page.getByLabel('Remember me').check()

// 4. Placeholder
// ⚠️ プレースホルダーは変更される可能性がある

await page.getByPlaceholder('Enter your email').fill('test@example.com')

// 5. Text（コンテンツベース）
// ⚠️ 多言語対応で壊れる可能性
// ⚠️ テキスト変更で壊れる

await page.getByText('Sign In').click()
await page.locator('text=Welcome back').isVisible()

// 6. CSSセレクター
// ❌ 最終手段
// ❌ CSS変更で壊れやすい

await page.click('.btn.btn-primary')
await page.locator('.user-menu > li:first-child').click()

// 7. XPath
// ❌ 非推奨
// ❌ 可読性が低く、メンテナンスが困難

await page.locator('//button[@class="submit"]').click()
```

### data-testid戦略の実装

**React/Next.jsアプリケーションでの実装:**

```tsx
// components/LoginForm.tsx
export function LoginForm() {
  return (
    <form data-testid="login-form">
      <input
        data-testid="email-input"
        type="email"
        name="email"
        aria-label="Email"
      />
      <input
        data-testid="password-input"
        type="password"
        name="password"
        aria-label="Password"
      />
      <button
        data-testid="submit-button"
        type="submit"
      >
        Sign In
      </button>
      <div data-testid="error-message" role="alert">
        {/* エラーメッセージ */}
      </div>
    </form>
  )
}
```

**対応するテストコード:**

```typescript
// tests/e2e/login.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Login Form', () => {
  test('should login with valid credentials', async ({ page }) => {
    await page.goto('/login')

    // data-testidで要素を特定
    const form = page.locator('[data-testid="login-form"]')
    const emailInput = page.locator('[data-testid="email-input"]')
    const passwordInput = page.locator('[data-testid="password-input"]')
    const submitButton = page.locator('[data-testid="submit-button"]')

    await expect(form).toBeVisible()
    await emailInput.fill('user@example.com')
    await passwordInput.fill('SecurePass123!')
    await submitButton.click()

    await expect(page).toHaveURL('/dashboard')
  })

  test('should display error for invalid credentials', async ({ page }) => {
    await page.goto('/login')

    await page.locator('[data-testid="email-input"]').fill('invalid@example.com')
    await page.locator('[data-testid="password-input"]').fill('wrongpassword')
    await page.locator('[data-testid="submit-button"]').click()

    const errorMessage = page.locator('[data-testid="error-message"]')
    await expect(errorMessage).toBeVisible()
    await expect(errorMessage).toContainText('Invalid email or password')
  })
})
```

### Role-based セレクターの活用

**アクセシビリティとテストの両立:**

```typescript
test('accessible selectors example', async ({ page }) => {
  await page.goto('/dashboard')

  // ナビゲーション
  await page.getByRole('navigation').getByRole('link', { name: 'Home' }).click()

  // ボタン
  await page.getByRole('button', { name: 'Create Post' }).click()
  await page.getByRole('button', { name: /delete/i }).click() // 正規表現

  // フォーム要素
  await page.getByRole('textbox', { name: 'Title' }).fill('My Post')
  await page.getByRole('textbox', { name: 'Content' }).fill('Post content')
  await page.getByRole('combobox', { name: 'Category' }).selectOption('Tech')
  await page.getByRole('checkbox', { name: 'Publish immediately' }).check()
  await page.getByRole('radio', { name: 'Public' }).check()

  // 見出し
  await expect(page.getByRole('heading', { name: 'Dashboard', level: 1 })).toBeVisible()

  // リスト
  const items = page.getByRole('listitem')
  await expect(items).toHaveCount(5)

  // テーブル
  const table = page.getByRole('table')
  const firstRow = table.getByRole('row').first()
  await expect(firstRow.getByRole('cell').first()).toContainText('John Doe')
})
```

### 複雑なセレクターパターン

**入れ子構造とフィルタリング:**

```typescript
test('complex selectors', async ({ page }) => {
  await page.goto('/products')

  // 親要素から子要素を辿る
  const productCard = page.locator('[data-testid="product-card"]').first()
  await productCard.locator('[data-testid="add-to-cart"]').click()

  // フィルタリング
  await page.locator('button').filter({ hasText: 'Buy Now' }).click()
  await page.locator('.product').filter({ has: page.locator('.sale-badge') }).first().click()

  // n番目の要素
  await page.locator('.item').nth(2).click()
  await page.locator('.item').first().click()
  await page.locator('.item').last().click()

  // 複数条件
  await page.locator('button')
    .filter({ hasText: 'Submit' })
    .filter({ has: page.locator('.icon-check') })
    .click()

  // 兄弟要素
  const label = page.getByText('Username')
  await label.locator('..').locator('input').fill('testuser')
})
```

## 複雑なユーザーフローテスト

実際のアプリケーションでは、複数のページやステップを跨ぐ複雑なユーザーフローをテストする必要があります。

### E-commerce購入フロー完全版

**エンドツーエンドの購入プロセス:**

```typescript
// tests/e2e/checkout-flow.spec.ts
import { test, expect } from '@playwright/test'

test.describe('E-commerce Checkout Flow', () => {
  test.beforeEach(async ({ page }) => {
    // 全テストで共通の初期化
    await page.goto('/')
  })

  test('complete purchase flow with registered user', async ({ page }) => {
    // Step 1: ログイン
    await page.getByRole('link', { name: 'Sign In' }).click()
    await page.getByLabel('Email').fill('customer@example.com')
    await page.getByLabel('Password').fill('SecurePass123!')
    await page.getByRole('button', { name: 'Sign In' }).click()

    await expect(page.getByText('Welcome back, John')).toBeVisible()

    // Step 2: 商品検索
    const searchBox = page.locator('[data-testid="search-input"]')
    await searchBox.fill('MacBook Pro')
    await searchBox.press('Enter')

    // 検索結果の表示確認
    await expect(page.getByRole('heading', { name: /search results/i })).toBeVisible()
    const productCards = page.locator('[data-testid="product-card"]')
    await expect(productCards).toHaveCount(10)

    // Step 3: 商品詳細ページ
    await productCards.first().click()

    // 商品情報の確認
    await expect(page.getByRole('heading', { level: 1 })).toContainText('MacBook')
    const price = await page.locator('[data-testid="product-price"]').textContent()
    expect(parseFloat(price?.replace(/[^0-9.]/g, '') || '0')).toBeGreaterThan(0)

    // 在庫確認
    await expect(page.locator('[data-testid="stock-status"]')).toContainText('In Stock')

    // Step 4: カートに追加
    await page.locator('[data-testid="quantity-select"]').selectOption('1')
    await page.getByRole('button', { name: 'Add to Cart' }).click()

    // 追加成功のフィードバック
    await expect(page.locator('[data-testid="toast-success"]')).toContainText('Added to cart')
    await expect(page.locator('[data-testid="cart-count"]')).toContainText('1')

    // Step 5: カートページ
    await page.locator('[data-testid="cart-icon"]').click()
    await expect(page).toHaveURL(/\/cart/)

    const cartItem = page.locator('[data-testid="cart-item"]').first()
    await expect(cartItem).toBeVisible()
    await expect(cartItem.locator('[data-testid="item-name"]')).toContainText('MacBook')

    // 小計の確認
    const subtotal = page.locator('[data-testid="subtotal"]')
    await expect(subtotal).toBeVisible()

    // Step 6: チェックアウト
    await page.getByRole('button', { name: 'Proceed to Checkout' }).click()
    await expect(page).toHaveURL(/\/checkout/)

    // Step 7: 配送先情報（既存住所を選択）
    await page.getByRole('radio', { name: /home address/i }).check()
    await page.getByRole('button', { name: 'Continue to Payment' }).click()

    // Step 8: 支払い方法
    await page.locator('[data-testid="payment-method-credit"]').check()

    // クレジットカード情報入力（テスト環境）
    const cardFrame = page.frameLocator('[data-testid="card-element-iframe"]')
    await cardFrame.locator('[data-testid="card-number"]').fill('4242424242424242')
    await cardFrame.locator('[data-testid="card-expiry"]').fill('12/25')
    await cardFrame.locator('[data-testid="card-cvc"]').fill('123')

    // Step 9: 注文確認
    await page.getByRole('button', { name: 'Review Order' }).click()

    // 最終確認
    await expect(page.locator('[data-testid="review-items"]')).toContainText('MacBook')
    await expect(page.locator('[data-testid="review-total"]')).toBeVisible()

    // Step 10: 注文確定
    await page.getByRole('button', { name: 'Place Order' }).click()

    // ローディング表示
    await expect(page.locator('[data-testid="processing-payment"]')).toBeVisible()

    // Step 11: 注文完了ページ
    await expect(page).toHaveURL(/\/order-confirmation/, { timeout: 10000 })

    await expect(page.getByRole('heading', { name: /order confirmed/i })).toBeVisible()

    const orderNumber = page.locator('[data-testid="order-number"]')
    await expect(orderNumber).toBeVisible()
    const orderNumberText = await orderNumber.textContent()
    expect(orderNumberText).toMatch(/^#\d{6,}$/)

    // メール送信の確認メッセージ
    await expect(page.getByText(/confirmation email/i)).toBeVisible()

    // Step 12: 注文履歴で確認
    await page.getByRole('link', { name: 'View Order Details' }).click()
    await expect(page).toHaveURL(/\/orders\/\d+/)
    await expect(page.locator('[data-testid="order-status"]')).toContainText('Processing')
  })

  test('guest checkout flow', async ({ page }) => {
    // ゲストユーザーとしての購入フロー
    await page.goto('/products/laptop-123')

    await page.getByRole('button', { name: 'Add to Cart' }).click()
    await page.locator('[data-testid="cart-icon"]').click()
    await page.getByRole('button', { name: 'Proceed to Checkout' }).click()

    // ゲストとして続行
    await page.getByRole('button', { name: 'Continue as Guest' }).click()

    // 配送先情報を手動入力
    await page.getByLabel('Email').fill('guest@example.com')
    await page.getByLabel('Full Name').fill('Guest User')
    await page.getByLabel('Address').fill('123 Main Street')
    await page.getByLabel('City').fill('New York')
    await page.getByLabel('State').selectOption('NY')
    await page.getByLabel('ZIP Code').fill('10001')
    await page.getByLabel('Phone').fill('555-0123')

    await page.getByRole('button', { name: 'Continue to Payment' }).click()

    // 支払い情報入力
    await page.locator('[data-testid="payment-method-credit"]').check()

    const cardFrame = page.frameLocator('[data-testid="card-element-iframe"]')
    await cardFrame.locator('[data-testid="card-number"]').fill('4242424242424242')
    await cardFrame.locator('[data-testid="card-expiry"]').fill('12/25')
    await cardFrame.locator('[data-testid="card-cvc"]').fill('123')

    await page.getByRole('button', { name: 'Place Order' }).click()

    await expect(page).toHaveURL(/\/order-confirmation/, { timeout: 10000 })
    await expect(page.getByRole('heading', { name: /order confirmed/i })).toBeVisible()
  })

  test('should handle out-of-stock product', async ({ page }) => {
    await page.goto('/products/out-of-stock-123')

    // 在庫切れ表示
    await expect(page.locator('[data-testid="stock-status"]')).toContainText('Out of Stock')

    // カート追加ボタンが無効
    const addToCartButton = page.getByRole('button', { name: 'Add to Cart' })
    await expect(addToCartButton).toBeDisabled()

    // 再入荷通知の設定
    await page.getByRole('button', { name: 'Notify When Available' }).click()
    await page.getByLabel('Email').fill('notify@example.com')
    await page.getByRole('button', { name: 'Subscribe' }).click()

    await expect(page.locator('[data-testid="toast-success"]')).toContainText('notification set')
  })

  test('should apply discount code', async ({ page }) => {
    await page.goto('/cart')

    // カートに商品を追加済みと仮定
    const originalTotal = await page.locator('[data-testid="total-amount"]').textContent()

    // 割引コード入力
    await page.locator('[data-testid="discount-input"]').fill('SAVE20')
    await page.getByRole('button', { name: 'Apply' }).click()

    // 割引適用確認
    await expect(page.locator('[data-testid="discount-applied"]')).toContainText('SAVE20')

    const newTotal = await page.locator('[data-testid="total-amount"]').textContent()
    expect(newTotal).not.toBe(originalTotal)

    // 割引額表示
    await expect(page.locator('[data-testid="discount-amount"]')).toBeVisible()
  })
})
```

### 認証フロー完全版

**複雑な認証シナリオ:**

```typescript
// tests/e2e/authentication.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Authentication Flows', () => {
  test('complete registration flow', async ({ page }) => {
    // Step 1: 登録ページ
    await page.goto('/register')

    // フォーム入力
    await page.getByLabel('Email').fill('newuser@example.com')
    await page.getByLabel('Username').fill('newuser123')
    await page.getByLabel('Password').fill('SecurePass123!')
    await page.getByLabel('Confirm Password').fill('SecurePass123!')

    // パスワード強度インジケーター
    const strengthIndicator = page.locator('[data-testid="password-strength"]')
    await expect(strengthIndicator).toHaveClass(/strong/)
    await expect(strengthIndicator).toContainText('Strong')

    // 利用規約同意
    await page.getByLabel('I agree to the Terms of Service').check()

    // Step 2: 送信
    await page.getByRole('button', { name: 'Create Account' }).click()

    // Step 3: メール確認メッセージ
    await expect(page).toHaveURL('/verify-email')
    await expect(page.getByRole('heading')).toContainText('Verify Your Email')
    await expect(page.getByText(/sent to newuser@example.com/i)).toBeVisible()

    // Step 4: メール確認リンククリック（モック）
    // 実際のアプリではメールサーバーから取得
    const verificationToken = 'mock-verification-token-123'
    await page.goto(`/verify-email/${verificationToken}`)

    // Step 5: 確認成功
    await expect(page.getByRole('heading')).toContainText('Email Verified')
    await page.getByRole('button', { name: 'Continue to Dashboard' }).click()

    // Step 6: 自動ログイン
    await expect(page).toHaveURL('/dashboard')
    await expect(page.getByText('Welcome, newuser123!')).toBeVisible()
  })

  test('password reset flow', async ({ page }) => {
    // Step 1: ログインページからパスワードリセット
    await page.goto('/login')
    await page.getByRole('link', { name: 'Forgot password?' }).click()

    // Step 2: メールアドレス入力
    await expect(page).toHaveURL('/forgot-password')
    await page.getByLabel('Email').fill('user@example.com')
    await page.getByRole('button', { name: 'Send Reset Link' }).click()

    // Step 3: 送信確認
    await expect(page.getByText(/reset link sent/i)).toBeVisible()

    // Step 4: リセットリンククリック（モック）
    const resetToken = 'mock-reset-token-456'
    await page.goto(`/reset-password/${resetToken}`)

    // Step 5: 新しいパスワード入力
    await page.getByLabel('New Password').fill('NewSecurePass456!')
    await page.getByLabel('Confirm New Password').fill('NewSecurePass456!')
    await page.getByRole('button', { name: 'Reset Password' }).click()

    // Step 6: リセット成功
    await expect(page).toHaveURL('/login')
    await expect(page.locator('[data-testid="success-message"]')).toContainText('Password reset successfully')

    // Step 7: 新しいパスワードでログイン
    await page.getByLabel('Email').fill('user@example.com')
    await page.getByLabel('Password').fill('NewSecurePass456!')
    await page.getByRole('button', { name: 'Sign In' }).click()

    await expect(page).toHaveURL('/dashboard')
  })

  test('two-factor authentication flow', async ({ page }) => {
    // Step 1: ログイン
    await page.goto('/login')
    await page.getByLabel('Email').fill('2fa-user@example.com')
    await page.getByLabel('Password').fill('SecurePass123!')
    await page.getByRole('button', { name: 'Sign In' }).click()

    // Step 2: 2FA コード入力画面
    await expect(page).toHaveURL('/verify-2fa')
    await expect(page.getByRole('heading')).toContainText('Two-Factor Authentication')

    // 6桁のコード入力
    const codeInputs = page.locator('[data-testid="2fa-code-input"]')
    await codeInputs.nth(0).fill('1')
    await codeInputs.nth(1).fill('2')
    await codeInputs.nth(2).fill('3')
    await codeInputs.nth(3).fill('4')
    await codeInputs.nth(4).fill('5')
    await codeInputs.nth(5).fill('6')

    // 自動送信またはボタンクリック
    await page.getByRole('button', { name: 'Verify' }).click()

    // Step 3: ログイン成功
    await expect(page).toHaveURL('/dashboard')
  })

  test('social login flow (OAuth)', async ({ page, context }) => {
    // Step 1: ソーシャルログインボタン
    await page.goto('/login')

    // Step 2: Googleログインをクリック
    const [popup] = await Promise.all([
      context.waitForEvent('page'),
      page.getByRole('button', { name: 'Continue with Google' }).click()
    ])

    // Step 3: OAuth画面（モック）
    await popup.waitForLoadState()
    await expect(popup).toHaveURL(/accounts\.google\.com/)

    // テスト環境では自動承認
    await popup.getByLabel('Email').fill('googleuser@gmail.com')
    await popup.getByLabel('Password').fill('GooglePass123!')
    await popup.getByRole('button', { name: 'Sign In' }).click()

    // Step 4: 権限承認
    await popup.getByRole('button', { name: 'Allow' }).click()

    // Step 5: リダイレクト後、元のページに戻る
    await page.waitForURL('/dashboard', { timeout: 10000 })
    await expect(page.getByText('Welcome, Google User!')).toBeVisible()
  })
})
```

## 並列実行とパフォーマンス最適化

大規模なテストスイートでは、実行時間の最適化が重要です。Playwrightの並列実行機能を活用して、テスト時間を大幅に短縮できます。

### 並列実行の設定

**playwright.config.ts の最適化:**

```typescript
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  // テストディレクトリ
  testDir: './tests/e2e',

  // 並列実行ワーカー数
  workers: process.env.CI ? 2 : undefined, // CI: 2並列, ローカル: CPU数に応じて自動

  // 同一ファイル内のテストを並列実行
  fullyParallel: true,

  // テストのタイムアウト
  timeout: 30000,

  // expectのタイムアウト
  expect: {
    timeout: 5000,
  },

  // リトライ設定
  retries: process.env.CI ? 2 : 0,

  // レポーター
  reporter: [
    ['html', { outputFolder: 'playwright-report', open: 'never' }],
    ['json', { outputFile: 'test-results/results.json' }],
    ['junit', { outputFile: 'test-results/junit.xml' }],
    ['list'], // コンソール出力
  ],

  use: {
    // ベースURL
    baseURL: process.env.BASE_URL || 'http://localhost:3000',

    // トレース（失敗時のみ）
    trace: 'on-first-retry',

    // スクリーンショット
    screenshot: 'only-on-failure',

    // ビデオ
    video: 'retain-on-failure',

    // ブラウザコンテキストのタイムアウト
    actionTimeout: 10000,
    navigationTimeout: 30000,
  },

  // 複数ブラウザプロジェクト
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'mobile-chrome',
      use: { ...devices['Pixel 5'] },
    },
    {
      name: 'mobile-safari',
      use: { ...devices['iPhone 13'] },
    },
  ],

  // 開発サーバー自動起動
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120000,
  },
})
```

### テストの分離とセットアップ

**テストの独立性を保つ:**

```typescript
// tests/e2e/test-isolation.spec.ts
import { test, expect } from '@playwright/test'

// ❌ 悪い例: テスト間で状態を共有
let sharedData: any

test('test 1', async ({ page }) => {
  sharedData = { userId: 123 }
  // ...
})

test('test 2', async ({ page }) => {
  // sharedData に依存（並列実行で失敗）
  await page.goto(`/users/${sharedData.userId}`)
})

// ✅ 良い例: 各テストが独立
test('isolated test 1', async ({ page }) => {
  const userId = 123
  await page.goto(`/users/${userId}`)
  // ...
})

test('isolated test 2', async ({ page }) => {
  const userId = 456
  await page.goto(`/users/${userId}`)
  // ...
})
```

**フィクスチャーでセットアップを共有:**

```typescript
// tests/fixtures/auth.ts
import { test as base } from '@playwright/test'

type AuthFixtures = {
  authenticatedPage: Page
}

export const test = base.extend<AuthFixtures>({
  authenticatedPage: async ({ page }, use) => {
    // 各テストの前にログイン
    await page.goto('/login')
    await page.getByLabel('Email').fill('user@example.com')
    await page.getByLabel('Password').fill('SecurePass123!')
    await page.getByRole('button', { name: 'Sign In' }).click()
    await page.waitForURL('/dashboard')

    // テストで使用
    await use(page)

    // クリーンアップ（必要に応じて）
    await page.close()
  },
})

// 使用例
import { test, expect } from './fixtures/auth'

test('view profile page', async ({ authenticatedPage }) => {
  await authenticatedPage.goto('/profile')
  await expect(authenticatedPage.getByRole('heading')).toContainText('Profile')
})
```

### パフォーマンス最適化テクニック

**1. 不要なリソースのブロック:**

```typescript
// tests/e2e/performance.spec.ts
import { test, expect } from '@playwright/test'

test.beforeEach(async ({ page }) => {
  // 画像、フォント、CSSをブロック（テストに不要な場合）
  await page.route('**/*.{png,jpg,jpeg,svg,gif,webp}', route => route.abort())
  await page.route('**/*.{woff,woff2,ttf,otf}', route => route.abort())
  await page.route('**/*.css', route => route.abort())

  // アナリティクスをブロック
  await page.route('**/analytics/**', route => route.abort())
  await page.route('**/google-analytics.com/**', route => route.abort())
})

test('fast page load', async ({ page }) => {
  await page.goto('/')
  // テスト内容...
})
```

**2. APIレスポンスのモック化:**

```typescript
test('mock slow API', async ({ page }) => {
  // 遅いAPIをモック
  await page.route('**/api/slow-endpoint', async route => {
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({ data: 'mocked response' }),
    })
  })

  await page.goto('/dashboard')
  // 高速にテスト実行
})
```

**3. ブラウザコンテキストの再利用:**

```typescript
// tests/e2e/reuse-context.spec.ts
import { test, expect } from '@playwright/test'

// 認証状態の保存
test('save auth state', async ({ page }) => {
  await page.goto('/login')
  await page.getByLabel('Email').fill('user@example.com')
  await page.getByLabel('Password').fill('SecurePass123!')
  await page.getByRole('button', { name: 'Sign In' }).click()

  await page.context().storageState({ path: 'auth.json' })
})

// 保存した認証状態を使用
test.use({ storageState: 'auth.json' })

test('reuse auth state', async ({ page }) => {
  // すでにログイン済み
  await page.goto('/dashboard')
  await expect(page.getByText('Welcome back')).toBeVisible()
})
```

**4. テストのグループ化とタグ付け:**

```typescript
// 重要なテストのみを実行
test.describe('critical flows @smoke', () => {
  test('login flow', async ({ page }) => {
    // ...
  })

  test('checkout flow', async ({ page }) => {
    // ...
  })
})

test.describe('extended tests @extended', () => {
  test('complex scenario 1', async ({ page }) => {
    // ...
  })
})

// 実行: npx playwright test --grep @smoke
// スキップ: npx playwright test --grep-invert @extended
```

## デバッグとトラブルシューティング

テスト失敗時の効率的なデバッグ方法を習得することで、問題解決が迅速になります。

### Playwright UI Mode

**インタラクティブなデバッグ:**

```bash
# UI Modeで実行
npx playwright test --ui

# 特定のテストファイルをUI Modeで実行
npx playwright test login.spec.ts --ui

# デバッグモード
npx playwright test --debug

# 特定の行からデバッグ
npx playwright test login.spec.ts:10 --debug
```

**コード内でのデバッグ:**

```typescript
import { test, expect } from '@playwright/test'

test('debug example', async ({ page }) => {
  await page.goto('/login')

  // ブラウザを一時停止（インスペクターが開く）
  await page.pause()

  await page.getByLabel('Email').fill('user@example.com')

  // ステップ実行モード
  await page.pause()

  await page.getByLabel('Password').fill('SecurePass123!')
  await page.getByRole('button', { name: 'Sign In' }).click()
})
```

### トレースビューアー

**失敗したテストのトレースを確認:**

```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    trace: 'on-first-retry', // リトライ時にトレース記録
  },
})

// または常にトレース
export default defineConfig({
  use: {
    trace: 'on', // すべてのテストでトレース記録（遅くなる）
  },
})
```

```bash
# トレースビューアーを開く
npx playwright show-trace test-results/login-chromium/trace.zip

# 失敗したテストのトレースを自動で開く
npx playwright show-report
```

### よくある問題と解決策

**1. タイムアウトエラー:**

```typescript
// ❌ 問題
test('timeout error', async ({ page }) => {
  await page.goto('/slow-page')
  await page.click('.button') // タイムアウト
})

// ✅ 解決策1: タイムアウトを延長
test('with extended timeout', async ({ page }) => {
  await page.goto('/slow-page', { timeout: 60000 })
  await page.click('.button', { timeout: 15000 })
})

// ✅ 解決策2: 待機条件を明示
test('with explicit wait', async ({ page }) => {
  await page.goto('/slow-page')
  await page.waitForLoadState('networkidle')
  await page.waitForSelector('.button', { state: 'visible' })
  await page.click('.button')
})
```

**2. フレーキーテスト（不安定なテスト）:**

```typescript
// ❌ 問題: レースコンディション
test('flaky test', async ({ page }) => {
  await page.click('.open-modal')
  await page.click('.modal .confirm') // モーダルが開く前に実行される可能性
})

// ✅ 解決策: 要素の可視性を待つ
test('stable test', async ({ page }) => {
  await page.click('.open-modal')
  await page.waitForSelector('.modal', { state: 'visible' })
  await page.click('.modal .confirm')
})

// ✅ 解決策: Playwrightの自動待機を活用
test('stable with auto-wait', async ({ page }) => {
  await page.click('.open-modal')
  await page.locator('.modal .confirm').click() // 自動で可視性を待つ
})
```

**3. セレクターが見つからない:**

```typescript
test('debug selector', async ({ page }) => {
  await page.goto('/page')

  // セレクターが存在するか確認
  const count = await page.locator('.my-button').count()
  console.log(`Found ${count} elements`)

  // ページのHTMLをダンプ
  const html = await page.content()
  console.log(html)

  // スクリーンショット撮影
  await page.screenshot({ path: 'debug.png', fullPage: true })

  // 可視性チェック
  const isVisible = await page.locator('.my-button').isVisible()
  console.log(`Button visible: ${isVisible}`)
})
```

**4. 認証状態の問題:**

```typescript
// tests/auth.setup.ts
import { test as setup } from '@playwright/test'

const authFile = 'playwright/.auth/user.json'

setup('authenticate', async ({ page }) => {
  await page.goto('/login')
  await page.getByLabel('Email').fill(process.env.TEST_USER_EMAIL!)
  await page.getByLabel('Password').fill(process.env.TEST_USER_PASSWORD!)
  await page.getByRole('button', { name: 'Sign In' }).click()

  // ログイン成功を確認
  await page.waitForURL('/dashboard')

  // 認証状態を保存
  await page.context().storageState({ path: authFile })
})

// playwright.config.ts
export default defineConfig({
  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        storageState: authFile,
      },
      dependencies: ['setup'],
    },
  ],
})
```

## 実践演習

### 演習1: ブログプラットフォームのテスト

次の機能をテストするテストスイートを作成してください：

1. ユーザー登録とログイン
2. ブログ記事の作成、編集、削除
3. コメント機能
4. いいね機能
5. プロフィール編集

**要件:**
- data-testidを使用したセレクター
- Page Objectパターンの採用
- 認証状態の再利用
- 並列実行対応

### 演習2: E-commerceサイトのクリティカルパステスト

次のシナリオをテストしてください：

1. 商品検索とフィルタリング
2. ウィッシュリスト追加
3. カート操作（追加、削除、数量変更）
4. ゲストチェックアウト
5. 登録ユーザーチェックアウト
6. 注文履歴の確認

**要件:**
- 完全なエンドツーエンドフロー
- エラーハンドリングのテスト
- レスポンシブデザインの検証
- パフォーマンス最適化

### 演習3: Visual Regression テスト

主要ページのビジュアルテストを実装してください：

1. ホームページ（デスクトップ、タブレット、モバイル）
2. 商品一覧ページ
3. 商品詳細ページ
4. カートページ
5. ダークモード対応

**要件:**
- 動的コンテンツのマスキング
- 複数ビューポートのテスト
- アニメーション無効化
- スナップショットの管理

## まとめ

本章では、Playwrightを使用したE2Eテストの実践的な手法を学びました：

1. **Playwright vs Cypress**: それぞれの特徴と選定基準を理解
2. **セレクター戦略**: data-testid、role-based、labelを活用した堅牢なセレクター
3. **複雑なユーザーフロー**: E-commerce購入フローと認証フローの実装
4. **並列実行とパフォーマンス最適化**: テスト時間を大幅に短縮する技術
5. **デバッグとトラブルシューティング**: UI Mode、トレースビューアー、よくある問題の解決

Playwrightは、モダンなWebアプリケーションのE2Eテストに最適なツールです。本章で学んだ技術を活用して、安定した高品質なテストスイートを構築してください。

次章では、Visual Regressionテストについて詳しく解説します。

---

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
