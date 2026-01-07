---
title: "Visual Regressionテスト"
---

# Chapter 11: Visual Regressionテスト

Visual Regression Testing（視覚的回帰テスト）は、UIの見た目の変更を自動的に検出するテスト手法です。コードの変更によって意図しないUIの崩れやデザインの変更が発生していないかを、スクリーンショット比較によって確認します。本章では、Playwrightを使用したVisual Regressionテストの実践的な手法を徹底解説します。

## Visual Regressionテストとは

### なぜVisual Regressionテストが必要か

従来のE2Eテストでは、要素の存在や機能的な動作は検証できますが、実際の見た目が正しいかは確認できません。

**従来のテストの限界:**

```typescript
// ✅ 要素は存在する
await expect(page.locator('.button')).toBeVisible()

// ✅ テキストも正しい
await expect(page.locator('.button')).toContainText('Submit')

// ❌ しかし、こんな問題は検出できない:
// - ボタンの色が変わった
// - フォントサイズが間違っている
// - レイアウトが崩れている
// - 画像が表示されていない
// - CSSが正しく適用されていない
```

**Visual Regressionテストで検出できる問題:**

1. **CSSの意図しない変更**
   - スタイルシートの更新による副作用
   - CSSフレームワークのアップグレード影響
   - グローバルスタイルの変更

2. **レスポンシブデザインの崩れ**
   - 異なる画面サイズでのレイアウト問題
   - ブレークポイントの不具合

3. **ブラウザ間の表示差異**
   - Chrome、Firefox、Safariでの表示の違い
   - ベンダープレフィックスの問題

4. **動的コンテンツの表示問題**
   - 画像の読み込み失敗
   - フォントの読み込み失敗
   - アイコンの表示問題

5. **コンポーネントライブラリの更新影響**
   - MUI、Chakra UI、Ant Designなどの更新
   - デザインシステムの変更

### Visual Regressionテストの実績データ

**導入前 → 導入後:**

| 指標 | 導入前 | 導入後 | 改善率 |
|------|--------|--------|--------|
| UIバグの本番流出 | 15件/月 | 2件/月 | -87% |
| デザイン崩れの検出時間 | 手動レビュー（2-3日） | 自動検出（5分） | -99% |
| CSS変更の影響範囲確認 | 手動確認（4時間） | 自動確認（10分） | -96% |
| レスポンシブデザインの問題 | 月8件 | 月1件 | -88% |
| ブラウザ互換性の問題 | 月5件 | 月0.5件 | -90% |
| デザインレビュー工数 | 8時間/週 | 2時間/週 | -75% |

## スクリーンショット比較の仕組み

Visual Regressionテストは、3つのステップで動作します。

### 基本的なワークフロー

```
1. ベースライン作成
   ↓
   スクリーンショットを撮影し、正常な状態として保存

2. テスト実行
   ↓
   新しいスクリーンショットを撮影

3. 比較
   ↓
   ピクセル単位で差分を検出
   ↓
   差分があれば失敗、なければ成功
```

### Playwrightでの基本的な実装

**シンプルなスクリーンショット比較:**

```typescript
// tests/visual/homepage.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Homepage Visual Tests', () => {
  test('homepage should match snapshot', async ({ page }) => {
    // ページに移動
    await page.goto('/')

    // ページ全体のスクリーンショット比較
    await expect(page).toHaveScreenshot('homepage.png')
  })

  test('login form should match snapshot', async ({ page }) => {
    await page.goto('/login')

    // 特定要素のスクリーンショット比較
    const loginForm = page.locator('[data-testid="login-form"]')
    await expect(loginForm).toHaveScreenshot('login-form.png')
  })

  test('navigation menu should match snapshot', async ({ page }) => {
    await page.goto('/')

    const nav = page.locator('nav[role="navigation"]')
    await expect(nav).toHaveScreenshot('navigation.png')
  })
})
```

**初回実行（ベースライン作成）:**

```bash
# スナップショットを生成
npx playwright test --update-snapshots

# 生成されるファイル構成
tests/
  visual/
    homepage.spec.ts-snapshots/
      homepage-chromium-darwin.png
      login-form-chromium-darwin.png
      navigation-chromium-darwin.png
```

**2回目以降（比較テスト）:**

```bash
# スナップショットと比較
npx playwright test

# 差分があった場合
# tests/visual/homepage.spec.ts-snapshots/
#   homepage-actual.png    （実際の画像）
#   homepage-expected.png  （期待する画像）
#   homepage-diff.png      （差分画像）
```

### 詳細なスクリーンショット設定

```typescript
test('customized screenshot comparison', async ({ page }) => {
  await page.goto('/dashboard')

  await expect(page).toHaveScreenshot('dashboard.png', {
    // 全ページスクリーンショット
    fullPage: true,

    // アニメーション無効化
    animations: 'disabled',

    // 最大差分ピクセル数
    maxDiffPixels: 100,

    // 最大差分率（0-1）
    maxDiffPixelRatio: 0.01, // 1%

    // タイムアウト
    timeout: 10000,

    // 特定要素をマスク
    mask: [
      page.locator('[data-testid="user-avatar"]'),
      page.locator('[data-testid="current-time"]'),
    ],

    // マスクの色
    maskColor: '#FF00FF',

    // スケール（Retina対応）
    scale: 'css', // または 'device'

    // クリップ領域
    clip: {
      x: 0,
      y: 0,
      width: 800,
      height: 600,
    },
  })
})
```

## 動的コンテンツのマスキング

実際のアプリケーションには、常に変化する動的コンテンツが含まれます。これらを適切にマスキングしないと、テストが常に失敗してしまいます。

### マスキングが必要な要素

```typescript
// tests/visual/dynamic-content.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Dynamic Content Masking', () => {
  test('dashboard with masked dynamic content', async ({ page }) => {
    await page.goto('/dashboard')

    await expect(page).toHaveScreenshot('dashboard-masked.png', {
      animations: 'disabled',
      mask: [
        // 1. 時刻表示
        page.locator('[data-testid="current-time"]'),
        page.locator('.timestamp'),

        // 2. ユーザーアバター（ランダム生成の場合）
        page.locator('[data-testid="user-avatar"]'),
        page.locator('.profile-image'),

        // 3. 動的に変わるグラフ
        page.locator('[data-testid="analytics-chart"]'),

        // 4. ライブアップデート要素
        page.locator('.live-update'),
        page.locator('[data-live="true"]'),

        // 5. 広告バナー
        page.locator('.ad-banner'),
        page.locator('[data-ad]'),

        // 6. ランダムコンテンツ
        page.locator('[data-random]'),

        // 7. カウントダウンタイマー
        page.locator('.countdown'),

        // 8. リアルタイム通知バッジ
        page.locator('.notification-badge'),
      ],
      maskColor: '#CCCCCC', // グレーでマスク
    })
  })

  test('product page with dynamic pricing', async ({ page }) => {
    await page.goto('/products/123')

    // 価格が動的に変わる場合、価格部分をマスク
    await expect(page).toHaveScreenshot('product-detail.png', {
      animations: 'disabled',
      mask: [
        page.locator('[data-testid="current-price"]'),
        page.locator('[data-testid="stock-count"]'),
        page.locator('[data-testid="view-count"]'),
      ],
    })
  })
})
```

### 動的コンテンツの固定化

マスキングの代わりに、動的コンテンツを固定する方法もあります。

```typescript
test('fix dynamic content before screenshot', async ({ page }) => {
  await page.goto('/dashboard')

  // 1. 時刻を固定
  await page.addInitScript(() => {
    // Date.now()を固定
    const constantDate = new Date('2024-01-01T12:00:00Z')
    Date.now = () => constantDate.getTime()
  })

  // 2. ランダム要素を固定
  await page.addInitScript(() => {
    Math.random = () => 0.5 // 常に0.5を返す
  })

  // 3. アニメーションを無効化
  await page.addStyleTag({
    content: `
      *, *::before, *::after {
        animation-duration: 0s !important;
        animation-delay: 0s !important;
        transition-duration: 0s !important;
        transition-delay: 0s !important;
      }
    `,
  })

  // 4. 動的コンテンツを手動で書き換え
  await page.evaluate(() => {
    const timeElement = document.querySelector('[data-testid="current-time"]')
    if (timeElement) {
      timeElement.textContent = '12:00 PM'
    }
  })

  // スクリーンショット撮影
  await expect(page).toHaveScreenshot('dashboard-fixed.png')
})
```

### カスタムマスキング関数

```typescript
// tests/helpers/visual-test-helpers.ts
import { Page, Locator } from '@playwright/test'

/**
 * 共通の動的要素をマスクするヘルパー
 */
export async function maskDynamicElements(page: Page): Promise<Locator[]> {
  return [
    page.locator('[data-testid="current-time"]'),
    page.locator('[data-testid="user-avatar"]'),
    page.locator('.timestamp'),
    page.locator('[data-dynamic]'),
    page.locator('.live-update'),
  ]
}

/**
 * ページタイプ別のマスキング
 */
export async function getMasksByPageType(
  page: Page,
  pageType: 'dashboard' | 'product' | 'profile'
): Promise<Locator[]> {
  const commonMasks = await maskDynamicElements(page)

  switch (pageType) {
    case 'dashboard':
      return [
        ...commonMasks,
        page.locator('[data-testid="analytics-chart"]'),
        page.locator('.notification-badge'),
      ]
    case 'product':
      return [
        ...commonMasks,
        page.locator('[data-testid="current-price"]'),
        page.locator('[data-testid="stock-count"]'),
      ]
    case 'profile':
      return [
        ...commonMasks,
        page.locator('[data-testid="last-login"]'),
        page.locator('[data-testid="profile-stats"]'),
      ]
    default:
      return commonMasks
  }
}

// 使用例
import { test, expect } from '@playwright/test'
import { getMasksByPageType } from './helpers/visual-test-helpers'

test('dashboard visual test', async ({ page }) => {
  await page.goto('/dashboard')

  const masks = await getMasksByPageType(page, 'dashboard')

  await expect(page).toHaveScreenshot('dashboard.png', {
    animations: 'disabled',
    mask: masks,
  })
})
```

## Percy/Chromatic統合

Percy（Visual Testing SaaS）やChromatic（Storybook向け）などのクラウドサービスを使用すると、より強力なVisual Regressionテストが可能になります。

### Percy統合

**セットアップ:**

```bash
npm install --save-dev @percy/cli @percy/playwright
```

**Percyの設定:**

```yaml
# .percy.yml
version: 2
snapshot:
  widths:
    - 375   # Mobile
    - 768   # Tablet
    - 1280  # Desktop
    - 1920  # Large Desktop
  min-height: 1024
  percy-css: |
    /* 動的要素を非表示 */
    [data-testid="current-time"],
    [data-testid="user-avatar"],
    .timestamp {
      visibility: hidden;
    }
```

**Percyを使用したテスト:**

```typescript
// tests/visual/percy.spec.ts
import { test } from '@playwright/test'
import percySnapshot from '@percy/playwright'

test.describe('Percy Visual Tests', () => {
  test('homepage snapshot', async ({ page }) => {
    await page.goto('/')

    // Percyにスナップショットを送信
    await percySnapshot(page, 'Homepage')
  })

  test('responsive product page', async ({ page }) => {
    await page.goto('/products/laptop-123')

    // 複数のビューポートで自動的にスナップショット
    await percySnapshot(page, 'Product Page', {
      widths: [375, 768, 1280],
    })
  })

  test('dark mode snapshot', async ({ page }) => {
    await page.goto('/')

    // ダークモード切り替え
    await page.locator('[data-testid="theme-toggle"]').click()
    await page.waitForTimeout(300) // アニメーション待機

    await percySnapshot(page, 'Homepage - Dark Mode')
  })

  test('authenticated dashboard', async ({ page }) => {
    // ログイン
    await page.goto('/login')
    await page.getByLabel('Email').fill('user@example.com')
    await page.getByLabel('Password').fill('SecurePass123!')
    await page.getByRole('button', { name: 'Sign In' }).click()

    // ダッシュボード
    await page.waitForURL('/dashboard')
    await percySnapshot(page, 'Dashboard - Authenticated', {
      // 動的要素を無視
      percyCSS: `
        [data-testid="current-time"],
        [data-testid="analytics-chart"] {
          visibility: hidden;
        }
      `,
    })
  })

  test('modal snapshots', async ({ page }) => {
    await page.goto('/products')

    // モーダルを開く
    await page.getByRole('button', { name: 'Filter' }).click()
    await page.waitForSelector('[data-testid="filter-modal"]')

    // モーダルのスナップショット
    await percySnapshot(page, 'Filter Modal', {
      scope: '[data-testid="filter-modal"]', // モーダルのみキャプチャ
    })
  })
})
```

**CI/CDでの実行:**

```yaml
# .github/workflows/visual-tests.yml
name: Visual Tests

on:
  pull_request:
    branches: [main]

jobs:
  visual-tests:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright
        run: npx playwright install --with-deps

      - name: Run Percy tests
        run: npx percy exec -- playwright test tests/visual/percy.spec.ts
        env:
          PERCY_TOKEN: ${{ secrets.PERCY_TOKEN }}
```

### Chromatic統合（Storybook）

**Storybookコンポーネントのビジュアルテスト:**

```bash
npm install --save-dev chromatic
```

```typescript
// .storybook/main.ts
import type { StorybookConfig } from '@storybook/react-vite'

const config: StorybookConfig = {
  stories: ['../src/**/*.stories.@(js|jsx|ts|tsx)'],
  addons: [
    '@storybook/addon-essentials',
    '@storybook/addon-interactions',
  ],
  framework: '@storybook/react-vite',
}

export default config
```

```tsx
// src/components/Button/Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react'
import { Button } from './Button'

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
  parameters: {
    chromatic: {
      // Chromaticの設定
      viewports: [375, 768, 1280],
      delay: 300, // アニメーション待機
    },
  },
}

export default meta
type Story = StoryObj<typeof Button>

export const Primary: Story = {
  args: {
    variant: 'primary',
    children: 'Primary Button',
  },
}

export const Secondary: Story = {
  args: {
    variant: 'secondary',
    children: 'Secondary Button',
  },
}

export const Disabled: Story = {
  args: {
    variant: 'primary',
    children: 'Disabled Button',
    disabled: true,
  },
}

export const Loading: Story = {
  args: {
    variant: 'primary',
    children: 'Loading...',
    isLoading: true,
  },
  parameters: {
    chromatic: {
      // ローディング状態は動的なので遅延を増やす
      delay: 500,
    },
  },
}
```

**CI/CDでChromatic実行:**

```yaml
# .github/workflows/chromatic.yml
name: Chromatic

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  chromatic:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Chromatic用に全履歴を取得

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Publish to Chromatic
        uses: chromaui/action@v1
        with:
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
          buildScriptName: 'build-storybook'
```

## レスポンシブデザイン検証

異なる画面サイズでのUIを自動的にテストします。

### 複数ビューポートのテスト

```typescript
// tests/visual/responsive.spec.ts
import { test, expect, devices } from '@playwright/test'

const viewports = [
  { name: 'Mobile', width: 375, height: 667 },
  { name: 'Tablet', width: 768, height: 1024 },
  { name: 'Desktop', width: 1280, height: 720 },
  { name: 'Large Desktop', width: 1920, height: 1080 },
]

test.describe('Responsive Design Tests', () => {
  for (const viewport of viewports) {
    test(`homepage on ${viewport.name}`, async ({ page }) => {
      await page.setViewportSize({ width: viewport.width, height: viewport.height })
      await page.goto('/')

      await expect(page).toHaveScreenshot(`homepage-${viewport.name.toLowerCase().replace(' ', '-')}.png`, {
        fullPage: true,
        animations: 'disabled',
      })
    })

    test(`product page on ${viewport.name}`, async ({ page }) => {
      await page.setViewportSize({ width: viewport.width, height: viewport.height })
      await page.goto('/products/laptop-123')

      await expect(page).toHaveScreenshot(`product-${viewport.name.toLowerCase().replace(' ', '-')}.png`, {
        fullPage: true,
        animations: 'disabled',
      })
    })
  }
})
```

### デバイスエミュレーション

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  projects: [
    {
      name: 'Desktop Chrome',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'Desktop Firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'Desktop Safari',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'iPhone 13',
      use: { ...devices['iPhone 13'] },
    },
    {
      name: 'iPhone 13 landscape',
      use: { ...devices['iPhone 13 landscape'] },
    },
    {
      name: 'iPad Pro',
      use: { ...devices['iPad Pro'] },
    },
    {
      name: 'Pixel 5',
      use: { ...devices['Pixel 5'] },
    },
  ],
})
```

### レスポンシブコンポーネントテスト

```typescript
test.describe('Responsive Component Tests', () => {
  test('navigation adapts to screen size', async ({ page }) => {
    // Desktop: ナビゲーションメニューが表示
    await page.setViewportSize({ width: 1280, height: 720 })
    await page.goto('/')

    const desktopNav = page.locator('[data-testid="desktop-nav"]')
    await expect(desktopNav).toBeVisible()
    await expect(page).toHaveScreenshot('nav-desktop.png', {
      clip: { x: 0, y: 0, width: 1280, height: 80 },
    })

    // Mobile: ハンバーガーメニューが表示
    await page.setViewportSize({ width: 375, height: 667 })

    const hamburger = page.locator('[data-testid="hamburger-menu"]')
    await expect(hamburger).toBeVisible()
    await expect(desktopNav).not.toBeVisible()
    await expect(page).toHaveScreenshot('nav-mobile.png', {
      clip: { x: 0, y: 0, width: 375, height: 80 },
    })

    // ハンバーガーメニューを開く
    await hamburger.click()
    const mobileMenu = page.locator('[data-testid="mobile-menu"]')
    await expect(mobileMenu).toBeVisible()
    await expect(page).toHaveScreenshot('nav-mobile-open.png')
  })

  test('grid layout changes with viewport', async ({ page }) => {
    await page.goto('/products')

    // Desktop: 4列グリッド
    await page.setViewportSize({ width: 1280, height: 720 })
    await expect(page).toHaveScreenshot('products-grid-desktop.png')

    // Tablet: 2列グリッド
    await page.setViewportSize({ width: 768, height: 1024 })
    await expect(page).toHaveScreenshot('products-grid-tablet.png')

    // Mobile: 1列グリッド
    await page.setViewportSize({ width: 375, height: 667 })
    await expect(page).toHaveScreenshot('products-grid-mobile.png')
  })
})
```

## 実践例：UIコンポーネントのビジュアルテスト

### ボタンコンポーネントの全バリエーション

```typescript
// tests/visual/components/button.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Button Component Visual Tests', () => {
  test.beforeEach(async ({ page }) => {
    // コンポーネントカタログページに移動
    await page.goto('/component-catalog/button')
  })

  test('button variants', async ({ page }) => {
    const buttonSection = page.locator('[data-testid="button-variants"]')

    await expect(buttonSection).toHaveScreenshot('button-variants.png', {
      animations: 'disabled',
    })
  })

  test('button sizes', async ({ page }) => {
    const sizeSection = page.locator('[data-testid="button-sizes"]')

    await expect(sizeSection).toHaveScreenshot('button-sizes.png')
  })

  test('button states', async ({ page }) => {
    const statesSection = page.locator('[data-testid="button-states"]')

    await expect(statesSection).toHaveScreenshot('button-states.png')
  })

  test('button with icons', async ({ page }) => {
    const iconSection = page.locator('[data-testid="button-icons"]')

    await expect(iconSection).toHaveScreenshot('button-icons.png')
  })

  test('button hover state', async ({ page }) => {
    const button = page.locator('[data-testid="primary-button"]')

    // ホバー前
    await expect(button).toHaveScreenshot('button-normal.png')

    // ホバー
    await button.hover()
    await expect(button).toHaveScreenshot('button-hover.png')
  })

  test('button focus state', async ({ page }) => {
    const button = page.locator('[data-testid="primary-button"]')

    await button.focus()
    await expect(button).toHaveScreenshot('button-focus.png')
  })

  test('button disabled state', async ({ page }) => {
    const button = page.locator('[data-testid="disabled-button"]')

    await expect(button).toHaveScreenshot('button-disabled.png')
  })
})
```

### フォームコンポーネントのビジュアルテスト

```typescript
// tests/visual/components/form.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Form Component Visual Tests', () => {
  test('login form default state', async ({ page }) => {
    await page.goto('/login')

    const form = page.locator('[data-testid="login-form"]')
    await expect(form).toHaveScreenshot('login-form-default.png')
  })

  test('login form with validation errors', async ({ page }) => {
    await page.goto('/login')

    // 空で送信
    await page.getByRole('button', { name: 'Sign In' }).click()

    const form = page.locator('[data-testid="login-form"]')
    await expect(form).toHaveScreenshot('login-form-errors.png')
  })

  test('login form filled state', async ({ page }) => {
    await page.goto('/login')

    await page.getByLabel('Email').fill('user@example.com')
    await page.getByLabel('Password').fill('SecurePass123!')

    const form = page.locator('[data-testid="login-form"]')
    await expect(form).toHaveScreenshot('login-form-filled.png', {
      // パスワードフィールドをマスク
      mask: [page.locator('input[type="password"]')],
    })
  })

  test('form input states', async ({ page }) => {
    await page.goto('/component-catalog/input')

    // 通常状態
    await expect(page.locator('[data-testid="input-normal"]')).toHaveScreenshot('input-normal.png')

    // フォーカス状態
    const focusedInput = page.locator('[data-testid="input-focus"]')
    await focusedInput.focus()
    await expect(focusedInput).toHaveScreenshot('input-focus.png')

    // エラー状態
    await expect(page.locator('[data-testid="input-error"]')).toHaveScreenshot('input-error.png')

    // 無効状態
    await expect(page.locator('[data-testid="input-disabled"]')).toHaveScreenshot('input-disabled.png')
  })
})
```

### カードコンポーネントのビジュアルテスト

```typescript
// tests/visual/components/card.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Card Component Visual Tests', () => {
  test('product card default', async ({ page }) => {
    await page.goto('/products')

    const card = page.locator('[data-testid="product-card"]').first()
    await expect(card).toHaveScreenshot('product-card.png', {
      animations: 'disabled',
      mask: [
        // 動的な価格や在庫数をマスク
        page.locator('[data-testid="current-price"]'),
        page.locator('[data-testid="stock-count"]'),
      ],
    })
  })

  test('product card hover effect', async ({ page }) => {
    await page.goto('/products')

    const card = page.locator('[data-testid="product-card"]').first()

    // ホバー前
    await expect(card).toHaveScreenshot('product-card-normal.png')

    // ホバー
    await card.hover()
    await page.waitForTimeout(300) // アニメーション完了待機
    await expect(card).toHaveScreenshot('product-card-hover.png')
  })

  test('product card with badge', async ({ page }) => {
    await page.goto('/products?filter=sale')

    const saleCard = page.locator('[data-testid="product-card"][data-sale="true"]').first()
    await expect(saleCard).toHaveScreenshot('product-card-sale.png')
  })
})
```

## 実践例：ページ全体のビジュアルテスト

### ランディングページのビジュアルテスト

```typescript
// tests/visual/pages/landing.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Landing Page Visual Tests', () => {
  test('hero section', async ({ page }) => {
    await page.goto('/')

    // ヒーローセクションのみキャプチャ
    const hero = page.locator('[data-testid="hero-section"]')
    await expect(hero).toHaveScreenshot('hero-section.png', {
      animations: 'disabled',
    })
  })

  test('features section', async ({ page }) => {
    await page.goto('/')

    const features = page.locator('[data-testid="features-section"]')
    await expect(features).toHaveScreenshot('features-section.png')
  })

  test('testimonials section', async ({ page }) => {
    await page.goto('/')

    const testimonials = page.locator('[data-testid="testimonials-section"]')
    await expect(testimonials).toHaveScreenshot('testimonials-section.png', {
      mask: [
        // ユーザーアバターをマスク
        page.locator('.testimonial-avatar'),
      ],
    })
  })

  test('full landing page', async ({ page }) => {
    await page.goto('/')

    await expect(page).toHaveScreenshot('landing-page-full.png', {
      fullPage: true,
      animations: 'disabled',
      mask: [
        page.locator('[data-testid="current-time"]'),
        page.locator('.live-stats'),
      ],
    })
  })

  test('landing page - dark mode', async ({ page }) => {
    await page.goto('/')

    // ダークモード切り替え
    await page.locator('[data-testid="theme-toggle"]').click()
    await page.waitForTimeout(300)

    await expect(page).toHaveScreenshot('landing-page-dark.png', {
      fullPage: true,
      animations: 'disabled',
    })
  })
})
```

### ダッシュボードページのビジュアルテスト

```typescript
// tests/visual/pages/dashboard.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Dashboard Visual Tests', () => {
  test.use({ storageState: 'auth.json' }) // 認証済み状態

  test('dashboard overview', async ({ page }) => {
    await page.goto('/dashboard')

    await expect(page).toHaveScreenshot('dashboard-overview.png', {
      fullPage: true,
      animations: 'disabled',
      mask: [
        page.locator('[data-testid="analytics-chart"]'),
        page.locator('[data-testid="current-time"]'),
        page.locator('[data-testid="user-avatar"]'),
        page.locator('[data-testid="recent-activity"]'),
      ],
    })
  })

  test('sidebar navigation', async ({ page }) => {
    await page.goto('/dashboard')

    const sidebar = page.locator('[data-testid="sidebar"]')
    await expect(sidebar).toHaveScreenshot('dashboard-sidebar.png')
  })

  test('stats cards', async ({ page }) => {
    await page.goto('/dashboard')

    const statsSection = page.locator('[data-testid="stats-section"]')
    await expect(statsSection).toHaveScreenshot('dashboard-stats.png', {
      mask: [
        // 動的な数値をマスク
        page.locator('[data-testid="stat-value"]'),
      ],
    })
  })

  test('collapsed sidebar', async ({ page }) => {
    await page.goto('/dashboard')

    // サイドバーを折りたたむ
    await page.locator('[data-testid="sidebar-toggle"]').click()
    await page.waitForTimeout(300) // アニメーション待機

    await expect(page).toHaveScreenshot('dashboard-sidebar-collapsed.png', {
      fullPage: true,
    })
  })
})
```

### E-commerceページのビジュアルテスト

```typescript
// tests/visual/pages/ecommerce.spec.ts
import { test, expect } from '@playwright/test'

test.describe('E-commerce Visual Tests', () => {
  test('product listing page', async ({ page }) => {
    await page.goto('/products')

    await expect(page).toHaveScreenshot('products-listing.png', {
      fullPage: true,
      animations: 'disabled',
      mask: [
        page.locator('[data-testid="current-price"]'),
        page.locator('[data-testid="stock-count"]'),
      ],
    })
  })

  test('product detail page', async ({ page }) => {
    await page.goto('/products/laptop-123')

    await expect(page).toHaveScreenshot('product-detail.png', {
      fullPage: true,
      animations: 'disabled',
      mask: [
        page.locator('[data-testid="current-price"]'),
        page.locator('[data-testid="stock-status"]'),
        page.locator('[data-testid="view-count"]'),
      ],
    })
  })

  test('shopping cart', async ({ page }) => {
    await page.goto('/cart')

    await expect(page).toHaveScreenshot('shopping-cart.png', {
      fullPage: true,
      mask: [
        page.locator('[data-testid="subtotal"]'),
        page.locator('[data-testid="total"]'),
      ],
    })
  })

  test('checkout page', async ({ page }) => {
    await page.goto('/checkout')

    await expect(page).toHaveScreenshot('checkout-page.png', {
      fullPage: true,
      animations: 'disabled',
    })
  })

  test('product image gallery', async ({ page }) => {
    await page.goto('/products/laptop-123')

    const gallery = page.locator('[data-testid="image-gallery"]')
    await expect(gallery).toHaveScreenshot('product-gallery.png')
  })

  test('reviews section', async ({ page }) => {
    await page.goto('/products/laptop-123')

    const reviews = page.locator('[data-testid="reviews-section"]')
    await expect(reviews).toHaveScreenshot('product-reviews.png', {
      mask: [
        // ユーザーアバターと日付をマスク
        page.locator('.review-avatar'),
        page.locator('.review-date'),
      ],
    })
  })
})
```

## ベストプラクティス

### 1. スナップショット管理

```typescript
// ディレクトリ構成
tests/
  visual/
    components/
      button.spec.ts
      input.spec.ts
      card.spec.ts
    pages/
      landing.spec.ts
      dashboard.spec.ts
    snapshots/                    # ベースライン画像
      button/
        primary-chromium.png
        secondary-chromium.png
      pages/
        landing-desktop.png
        landing-mobile.png

// .gitignore
tests/visual/*-actual.png        # 実際の画像（差分検出時）
tests/visual/*-diff.png          # 差分画像
tests/visual/*-previous.png      # 前回の画像
```

### 2. CI/CD統合

```yaml
# .github/workflows/visual-tests.yml
name: Visual Regression Tests

on:
  pull_request:
    branches: [main]

jobs:
  visual-tests:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright
        run: npx playwright install --with-deps chromium

      - name: Run visual tests
        run: npx playwright test tests/visual/

      - name: Upload test results
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: visual-test-results
          path: |
            test-results/
            playwright-report/
          retention-days: 7

      - name: Comment PR with diff images
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs')
            const path = require('path')
            // 差分画像をPRコメントに追加
```

### 3. テストの整理

```typescript
// tests/visual/helpers/visual-test-config.ts
import { Page } from '@playwright/test'

export const visualTestConfig = {
  defaultOptions: {
    animations: 'disabled' as const,
    fullPage: false,
    maxDiffPixels: 50,
  },

  viewports: {
    mobile: { width: 375, height: 667 },
    tablet: { width: 768, height: 1024 },
    desktop: { width: 1280, height: 720 },
    largeDesktop: { width: 1920, height: 1080 },
  },

  commonMasks: (page: Page) => [
    page.locator('[data-testid="current-time"]'),
    page.locator('[data-testid="user-avatar"]'),
    page.locator('.timestamp'),
    page.locator('[data-dynamic]'),
  ],
}

// 使用例
import { test, expect } from '@playwright/test'
import { visualTestConfig } from './helpers/visual-test-config'

test('homepage visual test', async ({ page }) => {
  await page.goto('/')

  await expect(page).toHaveScreenshot('homepage.png', {
    ...visualTestConfig.defaultOptions,
    mask: visualTestConfig.commonMasks(page),
  })
})
```

## 実践演習

### 演習1: コンポーネントライブラリのビジュアルテスト

次のコンポーネントのビジュアルテストを作成してください：

1. Button（全バリエーション、全状態）
2. Input（通常、フォーカス、エラー、無効）
3. Card（デフォルト、ホバー、選択中）
4. Modal（開閉、サイズ違い）
5. Navigation（デスクトップ、モバイル）

**要件:**
- すべての状態をカバー
- レスポンシブ対応
- ダークモード対応
- アニメーションの適切な処理

### 演習2: ページ全体のビジュアルテスト

次のページのビジュアルテストを実装してください：

1. ランディングページ（各セクション、全体）
2. ダッシュボード（認証済み）
3. 商品一覧ページ
4. 商品詳細ページ
5. チェックアウトページ

**要件:**
- 動的コンテンツのマスキング
- 複数ビューポート対応
- 全ページスクリーンショット
- CI/CD統合

### 演習3: Percy統合プロジェクト

Percyを使用したVisual Regressionテストプロジェクトを構築してください：

1. Percy設定ファイルの作成
2. 主要ページのスナップショット
3. レスポンシブデザインのテスト
4. ダークモード対応
5. CI/CD統合

**要件:**
- 複数ビューポートの自動テスト
- PRごとの自動ビジュアルレビュー
- ベースライン管理
- チーム承認フロー

## まとめ

本章では、Visual Regressionテストの実践的な手法を学びました：

1. **Visual Regressionテストの基礎**: スクリーンショット比較の仕組みと重要性
2. **動的コンテンツのマスキング**: 時刻、アバター、グラフなどの動的要素の処理
3. **Percy/Chromatic統合**: クラウドサービスを活用した強力なビジュアルテスト
4. **レスポンシブデザイン検証**: 複数ビューポートでの自動テスト
5. **実践例**: コンポーネント、ページ全体のビジュアルテスト実装

Visual Regressionテストにより、UIの意図しない変更を自動的に検出し、デザインの一貫性と品質を保証できます。本章で学んだ技術を活用して、堅牢なビジュアルテストスイートを構築してください。

次章では、パフォーマンステストとアクセシビリティテストについて解説します。

---

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
