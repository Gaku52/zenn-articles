---
title: "本番運用とデプロイメント"
---

# Chapter 9: 本番運用とデプロイメント

## この章で学べること

この章では、Reactアプリケーションの本番運用とデプロイメントを学びます。

- ✅ Git運用のベストプラクティス
- ✅ CI/CDパイプライン構築
- ✅ テスト戦略
- ✅ 本番環境での監視とエラー対応

**前提知識**: Chapter 01-08の内容、Gitの基礎

**所要時間**: 50-70分

**難易度**: ★★★★☆（中上級）

---

## 目次

1. [Git Workflow とブランチ戦略](#1-git-workflow-とブランチ戦略)
   - Git Flow vs GitHub Flow vs Trunk-Based Development
   - コミットメッセージ規約
   - Pull Request のベストプラクティス
2. [CI/CD パイプライン構築](#2-cicd-パイプライン構築)
   - GitHub Actions の基本
   - 自動テストとリント
   - 自動デプロイ
3. [テスト戦略](#3-テスト戦略)
   - Unit Testing with Vitest
   - Integration Testing with Testing Library
   - E2E Testing with Playwright
   - テストピラミッド
4. [本番環境へのデプロイメント](#4-本番環境へのデプロイメント)
   - Vercel へのデプロイ
   - 環境変数の管理
   - プレビュー環境の活用
5. [監視とエラー対応](#5-監視とエラー対応)
   - Sentry によるエラー監視
   - パフォーマンスモニタリング
   - ログ収集と分析

---

## 1. Git Workflow とブランチ戦略

### 1.1 ブランチ戦略の選択

**チームサイズ別の推奨戦略**:

```typescript
// Small Team (1-5 people) → GitHub Flow
// 特徴: シンプル、mainブランチから直接feature作成
main
  └── feature/user-login
  └── feature/dashboard

// Medium Team (5-15 people) → Git Flow
// 特徴: develop, release ブランチを活用
main
  └── develop
      ├── feature/user-login
      ├── feature/dashboard
      └── release/v1.2.0

// Large Team (15+ people) → Trunk-Based Development
// 特徴: 短命なfeatureブランチ、頻繁なmainマージ
main (常にデプロイ可能)
  └── feature/user-login (1-2日で完了)
```

**GitHub Flow の実践例**:

```bash
# 1. main から feature ブランチ作成
git checkout main
git pull origin main
git checkout -b feature/add-user-profile

# 2. 開発とコミット
git add .
git commit -m "feat: add user profile page"

# 3. リモートにプッシュ
git push origin feature/add-user-profile

# 4. Pull Request 作成（GitHubのUI上で）
# 5. レビュー後、mainにマージ
# 6. ブランチ削除
git checkout main
git pull origin main
git branch -d feature/add-user-profile
```

### 1.2 コミットメッセージ規約

**Conventional Commits の採用**:

```bash
# フォーマット
<type>(<scope>): <subject>

<body>

<footer>

# 例
feat(auth): add login with Google OAuth

- Implement Google OAuth 2.0 flow
- Add user session management
- Create auth middleware for protected routes

Closes #123
```

**Type の種類**:

```typescript
type CommitType =
  | 'feat'     // 新機能
  | 'fix'      // バグ修正
  | 'docs'     // ドキュメント変更
  | 'style'    // コードスタイル（フォーマット、セミコロン等）
  | 'refactor' // リファクタリング
  | 'perf'     // パフォーマンス改善
  | 'test'     // テスト追加・修正
  | 'chore'    // ビルド、ツール変更
  | 'ci'       // CI設定変更
  | 'revert'   // コミット取り消し

// 実践例
const examples = [
  'feat(dashboard): add analytics chart component',
  'fix(api): resolve race condition in data fetching',
  'perf(list): implement virtualization for 10k+ items',
  'docs(readme): update installation instructions',
  'test(auth): add integration tests for login flow'
]
```

**Commitlint の導入**:

```bash
# インストール
npm install --save-dev @commitlint/cli @commitlint/config-conventional

# .commitlintrc.json
{
  "extends": ["@commitlint/config-conventional"],
  "rules": {
    "type-enum": [
      2,
      "always",
      ["feat", "fix", "docs", "style", "refactor", "perf", "test", "chore", "ci", "revert"]
    ],
    "subject-case": [2, "always", "lower-case"],
    "subject-max-length": [2, "always", 72]
  }
}

# Husky で自動チェック
npx husky add .husky/commit-msg 'npx --no -- commitlint --edit "$1"'
```

### 1.3 Pull Request のベストプラクティス

**PR テンプレート**:

```markdown
<!-- .github/pull_request_template.md -->
## 概要
<!-- このPRで何を変更したか -->

## 変更内容
- [ ] 新機能追加
- [ ] バグ修正
- [ ] リファクタリング
- [ ] ドキュメント更新

## スクリーンショット（UIの変更がある場合）
<!-- Before/After の画像を貼る -->

## テスト
- [ ] Unit Tests 追加済み
- [ ] Integration Tests 追加済み
- [ ] E2E Tests 追加済み
- [ ] 手動テスト完了

## チェックリスト
- [ ] コードレビューを依頼した
- [ ] CI が全てパスしている
- [ ] 関連するドキュメントを更新した
- [ ] breaking change がある場合、CHANGELOG を更新した

## 関連 Issue
Closes #123
```

**小さなPRを心がける**:

```typescript
// ❌ 悪い例: 1つのPRで複数の機能を変更
// PR #1: "Add user profile, dashboard, and settings page" (500 lines changed)

// ✅ 良い例: 機能ごとに分割
// PR #1: "Add user profile page" (100 lines changed)
// PR #2: "Add dashboard page" (120 lines changed)
// PR #3: "Add settings page" (80 lines changed)

// 目安: 200-400行以内に収める
// - レビューしやすい
// - バグ混入リスク低減
// - 早くマージできる
```

---

## 2. CI/CD パイプライン構築

### 2.1 GitHub Actions の基本

**基本的なワークフロー**:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [18.x, 20.x]

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run type check
        run: npm run type-check

      - name: Run tests
        run: npm run test:ci

      - name: Build
        run: npm run build
```

### 2.2 自動テストとリント

**包括的なテストワークフロー**:

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  unit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run test:unit -- --coverage
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json

  integration-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run test:integration

  e2e-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npm run build
      - run: npm run test:e2e
      - uses: actions/upload-artifact@v3
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
```

**ESLint と Prettier の自動チェック**:

```yaml
# .github/workflows/lint.yml
name: Lint

on: [push, pull_request]

jobs:
  eslint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run lint

  prettier:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run format:check

  type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run type-check
```

### 2.3 自動デプロイ

**Vercel への自動デプロイ**:

```yaml
# .github/workflows/deploy.yml
name: Deploy to Vercel

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Vercel CLI
        run: npm install --global vercel@latest

      - name: Pull Vercel Environment Information
        run: vercel pull --yes --environment=production --token=${{ secrets.VERCEL_TOKEN }}

      - name: Build Project Artifacts
        run: vercel build --prod --token=${{ secrets.VERCEL_TOKEN }}

      - name: Deploy Project Artifacts to Vercel
        run: vercel deploy --prebuilt --prod --token=${{ secrets.VERCEL_TOKEN }}
```

**環境別デプロイ**:

```yaml
# .github/workflows/deploy-preview.yml
name: Deploy Preview

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Vercel Preview
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          scope: ${{ secrets.VERCEL_ORG_ID }}

      - name: Comment PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '✅ Preview deployed to: ${{ steps.deploy.outputs.preview-url }}'
            })
```

---

## 3. テスト戦略

### 3.1 Unit Testing with Vitest

**Vitest のセットアップ**:

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/tests/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: [
        'node_modules/',
        'src/tests/setup.ts',
        '**/*.d.ts',
        '**/*.config.ts',
        '**/mockData.ts'
      ]
    }
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src')
    }
  }
})
```

**hooks のテスト**:

```typescript
// src/hooks/useCounter.test.ts
import { renderHook, act } from '@testing-library/react'
import { describe, it, expect } from 'vitest'
import { useCounter } from './useCounter'

describe('useCounter', () => {
  it('初期値が0であること', () => {
    const { result } = renderHook(() => useCounter())
    expect(result.current.count).toBe(0)
  })

  it('incrementで値が1増えること', () => {
    const { result } = renderHook(() => useCounter())

    act(() => {
      result.current.increment()
    })

    expect(result.current.count).toBe(1)
  })

  it('decrementで値が1減ること', () => {
    const { result } = renderHook(() => useCounter(5))

    act(() => {
      result.current.decrement()
    })

    expect(result.current.count).toBe(4)
  })

  it('resetで初期値に戻ること', () => {
    const { result } = renderHook(() => useCounter(10))

    act(() => {
      result.current.increment()
      result.current.increment()
      result.current.reset()
    })

    expect(result.current.count).toBe(10)
  })
})
```

**ユーティリティ関数のテスト**:

```typescript
// src/lib/formatters.test.ts
import { describe, it, expect } from 'vitest'
import { formatCurrency, formatDate, formatRelativeTime } from './formatters'

describe('formatCurrency', () => {
  it('日本円を正しくフォーマット', () => {
    expect(formatCurrency(1000, 'JPY')).toBe('¥1,000')
    expect(formatCurrency(1234567, 'JPY')).toBe('¥1,234,567')
  })

  it('米ドルを正しくフォーマット', () => {
    expect(formatCurrency(1000, 'USD')).toBe('$1,000.00')
    expect(formatCurrency(1234.5, 'USD')).toBe('$1,234.50')
  })
})

describe('formatDate', () => {
  it('日付を正しくフォーマット', () => {
    const date = new Date('2026-01-05T12:00:00Z')
    expect(formatDate(date, 'ja')).toBe('2026年1月5日')
    expect(formatDate(date, 'en')).toBe('January 5, 2026')
  })
})

describe('formatRelativeTime', () => {
  it('相対時間を正しく表示', () => {
    const now = new Date()
    const oneHourAgo = new Date(now.getTime() - 60 * 60 * 1000)
    const oneDayAgo = new Date(now.getTime() - 24 * 60 * 60 * 1000)

    expect(formatRelativeTime(oneHourAgo, 'ja')).toBe('1時間前')
    expect(formatRelativeTime(oneDayAgo, 'ja')).toBe('1日前')
  })
})
```

### 3.2 Integration Testing with Testing Library

**コンポーネント統合テスト**:

```typescript
// src/components/LoginForm.test.tsx
import { render, screen, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { vi, describe, it, expect, beforeEach } from 'vitest'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { LoginForm } from './LoginForm'

describe('LoginForm', () => {
  let queryClient: QueryClient

  beforeEach(() => {
    queryClient = new QueryClient({
      defaultOptions: {
        queries: { retry: false },
        mutations: { retry: false }
      }
    })
  })

  const renderWithProviders = (ui: React.ReactElement) => {
    return render(
      <QueryClientProvider client={queryClient}>
        {ui}
      </QueryClientProvider>
    )
  }

  it('フォームが正しくレンダリングされる', () => {
    renderWithProviders(<LoginForm />)

    expect(screen.getByLabelText('メールアドレス')).toBeInTheDocument()
    expect(screen.getByLabelText('パスワード')).toBeInTheDocument()
    expect(screen.getByRole('button', { name: 'ログイン' })).toBeInTheDocument()
  })

  it('バリデーションエラーが表示される', async () => {
    const user = userEvent.setup()
    renderWithProviders(<LoginForm />)

    const submitButton = screen.getByRole('button', { name: 'ログイン' })
    await user.click(submitButton)

    expect(await screen.findByText('メールアドレスを入力してください')).toBeInTheDocument()
    expect(await screen.findByText('パスワードを入力してください')).toBeInTheDocument()
  })

  it('正しい入力でログインが成功する', async () => {
    const user = userEvent.setup()
    const mockOnSuccess = vi.fn()

    renderWithProviders(<LoginForm onSuccess={mockOnSuccess} />)

    await user.type(screen.getByLabelText('メールアドレス'), 'test@example.com')
    await user.type(screen.getByLabelText('パスワード'), 'password123')
    await user.click(screen.getByRole('button', { name: 'ログイン' }))

    await waitFor(() => {
      expect(mockOnSuccess).toHaveBeenCalledWith({
        email: 'test@example.com',
        token: expect.any(String)
      })
    })
  })

  it('ログイン失敗時にエラーメッセージが表示される', async () => {
    const user = userEvent.setup()

    // APIモック
    global.fetch = vi.fn(() =>
      Promise.resolve({
        ok: false,
        json: () => Promise.resolve({ error: '認証に失敗しました' })
      })
    ) as any

    renderWithProviders(<LoginForm />)

    await user.type(screen.getByLabelText('メールアドレス'), 'wrong@example.com')
    await user.type(screen.getByLabelText('パスワード'), 'wrongpassword')
    await user.click(screen.getByRole('button', { name: 'ログイン' }))

    expect(await screen.findByText('認証に失敗しました')).toBeInTheDocument()
  })
})
```

**APIとの統合テスト（MSW使用）**:

```typescript
// src/tests/mocks/handlers.ts
import { http, HttpResponse } from 'msw'

export const handlers = [
  http.post('/api/auth/login', async ({ request }) => {
    const { email, password } = await request.json()

    if (email === 'test@example.com' && password === 'password123') {
      return HttpResponse.json({
        user: { id: '1', email: 'test@example.com', name: 'Test User' },
        token: 'mock-jwt-token'
      })
    }

    return HttpResponse.json(
      { error: '認証に失敗しました' },
      { status: 401 }
    )
  }),

  http.get('/api/posts', () => {
    return HttpResponse.json([
      { id: '1', title: 'First Post', content: 'Content 1' },
      { id: '2', title: 'Second Post', content: 'Content 2' }
    ])
  })
]

// src/tests/setup.ts
import { afterAll, afterEach, beforeAll } from 'vitest'
import { setupServer } from 'msw/node'
import { handlers } from './mocks/handlers'
import '@testing-library/jest-dom/vitest'

export const server = setupServer(...handlers)

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }))
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

### 3.3 E2E Testing with Playwright

**Playwright のセットアップ**:

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',

  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure'
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] }
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] }
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] }
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] }
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 13'] }
    }
  ],

  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI
  }
})
```

**E2Eテストの実装**:

```typescript
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test'

test.describe('認証フロー', () => {
  test('ログイン → ダッシュボード表示 → ログアウト', async ({ page }) => {
    // ログインページへ
    await page.goto('/login')

    // ログインフォーム入力
    await page.getByLabel('メールアドレス').fill('test@example.com')
    await page.getByLabel('パスワード').fill('password123')
    await page.getByRole('button', { name: 'ログイン' }).click()

    // ダッシュボードにリダイレクトされる
    await expect(page).toHaveURL('/dashboard')
    await expect(page.getByRole('heading', { name: 'ダッシュボード' })).toBeVisible()

    // ユーザー情報が表示される
    await expect(page.getByText('Test User')).toBeVisible()

    // ログアウト
    await page.getByRole('button', { name: 'ログアウト' }).click()

    // ログインページにリダイレクト
    await expect(page).toHaveURL('/login')
  })

  test('未認証でダッシュボードにアクセスするとログインページにリダイレクト', async ({ page }) => {
    await page.goto('/dashboard')

    // ログインページにリダイレクトされる
    await expect(page).toHaveURL('/login')
    await expect(page.getByText('ログインが必要です')).toBeVisible()
  })
})

// e2e/post-crud.spec.ts
import { test, expect } from '@playwright/test'

test.describe('投稿CRUD', () => {
  test.beforeEach(async ({ page }) => {
    // ログイン処理
    await page.goto('/login')
    await page.getByLabel('メールアドレス').fill('test@example.com')
    await page.getByLabel('パスワード').fill('password123')
    await page.getByRole('button', { name: 'ログイン' }).click()
    await expect(page).toHaveURL('/dashboard')
  })

  test('新規投稿作成', async ({ page }) => {
    await page.goto('/posts/new')

    // フォーム入力
    await page.getByLabel('タイトル').fill('テスト投稿')
    await page.getByLabel('本文').fill('これはテスト投稿です')
    await page.getByRole('button', { name: '公開' }).click()

    // 成功メッセージ
    await expect(page.getByText('投稿を公開しました')).toBeVisible()

    // 投稿一覧に表示される
    await page.goto('/posts')
    await expect(page.getByText('テスト投稿')).toBeVisible()
  })

  test('投稿編集', async ({ page }) => {
    await page.goto('/posts')

    // 最初の投稿をクリック
    await page.getByRole('link', { name: 'テスト投稿' }).first().click()

    // 編集ボタンをクリック
    await page.getByRole('button', { name: '編集' }).click()

    // タイトル変更
    await page.getByLabel('タイトル').fill('更新されたテスト投稿')
    await page.getByRole('button', { name: '保存' }).click()

    // 成功メッセージ
    await expect(page.getByText('投稿を更新しました')).toBeVisible()

    // 更新後のタイトルが表示される
    await expect(page.getByRole('heading', { name: '更新されたテスト投稿' })).toBeVisible()
  })
})
```

### 3.4 テストピラミッド

**推奨するテスト配分**:

```typescript
// テストピラミッドの理想的な配分
const testPyramid = {
  unit: {
    ratio: '70%',
    target: 'Pure functions, hooks, utilities',
    tools: 'Vitest',
    speed: 'Fast (< 100ms/test)',
    cost: 'Low'
  },
  integration: {
    ratio: '20%',
    target: 'Components with API, State management',
    tools: 'Testing Library + MSW',
    speed: 'Medium (100-500ms/test)',
    cost: 'Medium'
  },
  e2e: {
    ratio: '10%',
    target: 'Critical user flows',
    tools: 'Playwright',
    speed: 'Slow (1-5s/test)',
    cost: 'High'
  }
}

// 実践例: E-Commerce サイトの場合
const ecommerceTests = {
  unit: [
    'formatCurrency()',
    'calculateDiscount()',
    'validateEmail()',
    'useCart hook',
    'useCheckout hook'
  ],
  integration: [
    '<ProductCard /> with API',
    '<ShoppingCart /> with state',
    '<CheckoutForm /> with validation'
  ],
  e2e: [
    'User journey: Browse → Add to Cart → Checkout → Payment',
    'User registration and first purchase',
    'Search products and filter'
  ]
}
```

**package.json のテストスクリプト**:

```json
{
  "scripts": {
    "test": "vitest",
    "test:unit": "vitest run --coverage",
    "test:integration": "vitest run --config vitest.integration.config.ts",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "test:ci": "npm run test:unit && npm run test:integration && npm run test:e2e",
    "test:watch": "vitest watch"
  }
}
```

---

## 4. 本番環境へのデプロイメント

### 4.1 Vercel へのデプロイ

**Vercel CLI によるデプロイ**:

```bash
# Vercel CLI インストール
npm install -g vercel

# 初回デプロイ
vercel

# 本番環境へのデプロイ
vercel --prod

# プレビュー環境へのデプロイ
vercel
```

**vercel.json の設定**:

```json
{
  "buildCommand": "npm run build",
  "devCommand": "npm run dev",
  "installCommand": "npm install",
  "framework": "nextjs",
  "regions": ["hnd1"],
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "X-Frame-Options",
          "value": "DENY"
        },
        {
          "key": "X-XSS-Protection",
          "value": "1; mode=block"
        },
        {
          "key": "Strict-Transport-Security",
          "value": "max-age=31536000; includeSubDomains"
        }
      ]
    }
  ],
  "redirects": [
    {
      "source": "/old-blog/:slug",
      "destination": "/blog/:slug",
      "permanent": true
    }
  ],
  "rewrites": [
    {
      "source": "/api/:path*",
      "destination": "https://api.example.com/:path*"
    }
  ]
}
```

### 4.2 環境変数の管理

**環境別の環境変数**:

```bash
# .env.local (開発環境)
NEXT_PUBLIC_API_URL=http://localhost:3001/api
DATABASE_URL=postgresql://localhost:5432/myapp_dev
STRIPE_SECRET_KEY=sk_test_xxxxx

# .env.production (本番環境 - Vercelで設定)
NEXT_PUBLIC_API_URL=https://api.example.com
DATABASE_URL=postgresql://prod-db.example.com:5432/myapp_prod
STRIPE_SECRET_KEY=sk_live_xxxxx

# .env.test (テスト環境)
NEXT_PUBLIC_API_URL=http://localhost:3001/api
DATABASE_URL=postgresql://localhost:5432/myapp_test
STRIPE_SECRET_KEY=sk_test_xxxxx
```

**型安全な環境変数**:

```typescript
// src/env.ts
import { z } from 'zod'

const envSchema = z.object({
  // Public (NEXT_PUBLIC_ prefix)
  NEXT_PUBLIC_API_URL: z.string().url(),
  NEXT_PUBLIC_APP_URL: z.string().url(),

  // Private (server-side only)
  DATABASE_URL: z.string(),
  STRIPE_SECRET_KEY: z.string().min(1),
  STRIPE_WEBHOOK_SECRET: z.string().min(1),

  // Optional
  SENTRY_DSN: z.string().optional(),
  ANALYTICS_ID: z.string().optional()
})

export const env = envSchema.parse({
  NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL,
  NEXT_PUBLIC_APP_URL: process.env.NEXT_PUBLIC_APP_URL,
  DATABASE_URL: process.env.DATABASE_URL,
  STRIPE_SECRET_KEY: process.env.STRIPE_SECRET_KEY,
  STRIPE_WEBHOOK_SECRET: process.env.STRIPE_WEBHOOK_SECRET,
  SENTRY_DSN: process.env.SENTRY_DSN,
  ANALYTICS_ID: process.env.ANALYTICS_ID
})

// 使用例
import { env } from '@/env'

async function fetchUser(id: string) {
  const res = await fetch(`${env.NEXT_PUBLIC_API_URL}/users/${id}`)
  return res.json()
}
```

### 4.3 プレビュー環境の活用

**Pull Request ごとのプレビュー**:

```yaml
# .github/workflows/preview.yml
name: Preview Deployment

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Vercel Preview
        id: deploy
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          scope: ${{ secrets.VERCEL_ORG_ID }}

      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v10
        with:
          urls: ${{ steps.deploy.outputs.preview-url }}
          uploadArtifacts: true

      - name: Comment PR with Preview URL
        uses: actions/github-script@v7
        with:
          script: |
            const { data: comments } = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
            })

            const botComment = comments.find(comment =>
              comment.user.type === 'Bot' &&
              comment.body.includes('Preview Deployed')
            )

            const commentBody = `## ✅ Preview Deployed

            **Preview URL**: ${{ steps.deploy.outputs.preview-url }}

            Lighthouse scores available in artifacts.`

            if (botComment) {
              await github.rest.issues.updateComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: botComment.id,
                body: commentBody
              })
            } else {
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.issue.number,
                body: commentBody
              })
            }
```

---

## 5. 監視とエラー対応

### 5.1 Sentry によるエラー監視

**Sentry のセットアップ**:

```bash
# インストール
npm install @sentry/nextjs
npx @sentry/wizard -i nextjs
```

**Sentry 設定**:

```typescript
// sentry.client.config.ts
import * as Sentry from '@sentry/nextjs'

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,

  // 本番環境のみ有効化
  enabled: process.env.NODE_ENV === 'production',

  // サンプリングレート
  tracesSampleRate: 1.0,

  // セッションリプレイ
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,

  // 環境情報
  environment: process.env.NEXT_PUBLIC_VERCEL_ENV || 'development',

  // リリースバージョン
  release: process.env.NEXT_PUBLIC_VERCEL_GIT_COMMIT_SHA,

  // エラーフィルタリング
  beforeSend(event, hint) {
    // 開発環境のエラーは送信しない
    if (process.env.NODE_ENV === 'development') {
      return null
    }

    // 特定のエラーを除外
    if (event.exception?.values?.[0]?.value?.includes('ResizeObserver')) {
      return null
    }

    return event
  }
})

// sentry.server.config.ts
import * as Sentry from '@sentry/nextjs'

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  enabled: process.env.NODE_ENV === 'production',
  tracesSampleRate: 1.0,
  environment: process.env.NEXT_PUBLIC_VERCEL_ENV || 'development',
  release: process.env.NEXT_PUBLIC_VERCEL_GIT_COMMIT_SHA
})
```

**エラーハンドリング**:

```typescript
// app/error.tsx (Next.js App Router)
'use client'

import * as Sentry from '@sentry/nextjs'
import { useEffect } from 'react'
import { Button } from '@/components/ui/button'

export default function Error({
  error,
  reset
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => {
    // Sentryにエラーを送信
    Sentry.captureException(error)
  }, [error])

  return (
    <div className="flex min-h-screen flex-col items-center justify-center">
      <h2 className="text-2xl font-bold">問題が発生しました</h2>
      <p className="mt-2 text-gray-600">
        エラーは記録されました。しばらくしてから再度お試しください。
      </p>
      {error.digest && (
        <p className="mt-1 text-sm text-gray-500">
          エラーID: {error.digest}
        </p>
      )}
      <Button onClick={reset} className="mt-4">
        再試行
      </Button>
    </div>
  )
}

// カスタムエラーハンドリング
import * as Sentry from '@sentry/nextjs'

export async function fetchUserData(userId: string) {
  try {
    const res = await fetch(`/api/users/${userId}`)
    if (!res.ok) throw new Error('Failed to fetch user')
    return await res.json()
  } catch (error) {
    // コンテキスト情報を追加
    Sentry.captureException(error, {
      tags: {
        section: 'user-data',
        userId
      },
      level: 'error'
    })

    throw error
  }
}
```

### 5.2 パフォーマンスモニタリング

**Vercel Analytics の導入**:

```typescript
// app/layout.tsx
import { Analytics } from '@vercel/analytics/react'
import { SpeedInsights } from '@vercel/speed-insights/next'

export default function RootLayout({
  children
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="ja">
      <body>
        {children}
        <Analytics />
        <SpeedInsights />
      </body>
    </html>
  )
}
```

**カスタムメトリクスの計測**:

```typescript
// lib/analytics.ts
import { track } from '@vercel/analytics'

export const trackEvent = {
  // ページビュー
  pageView: (url: string) => {
    track('page_view', { url })
  },

  // ユーザーアクション
  buttonClick: (buttonName: string) => {
    track('button_click', { button: buttonName })
  },

  // E-Commerce イベント
  addToCart: (productId: string, price: number) => {
    track('add_to_cart', { product_id: productId, price })
  },

  purchase: (orderId: string, total: number) => {
    track('purchase', { order_id: orderId, total })
  },

  // パフォーマンス
  apiLatency: (endpoint: string, duration: number) => {
    track('api_latency', { endpoint, duration })
  }
}

// 使用例
'use client'

import { trackEvent } from '@/lib/analytics'

export function AddToCartButton({ productId, price }: Props) {
  const handleClick = () => {
    trackEvent.addToCart(productId, price)
    // カート追加処理
  }

  return <button onClick={handleClick}>カートに追加</button>
}
```

**Web Vitals の計測**:

```typescript
// app/layout.tsx
'use client'

import { useReportWebVitals } from 'next/web-vitals'

export function WebVitals() {
  useReportWebVitals((metric) => {
    // Vercel Analytics に送信
    if (window.va) {
      window.va('event', {
        name: metric.name,
        value: metric.value,
        rating: metric.rating
      })
    }

    // Sentry に送信
    if (metric.rating === 'poor') {
      Sentry.captureMessage(`Poor ${metric.name}: ${metric.value}`, {
        level: 'warning',
        tags: {
          web_vital: metric.name,
          rating: metric.rating
        }
      })
    }
  })

  return null
}
```

### 5.3 ログ収集と分析

**構造化ロギング**:

```typescript
// lib/logger.ts
import pino from 'pino'

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  browser: {
    asObject: true
  },
  formatters: {
    level: (label) => {
      return { level: label }
    }
  }
})

// 使用例
import { logger } from '@/lib/logger'

export async function POST(request: Request) {
  const startTime = Date.now()

  try {
    const body = await request.json()

    logger.info({
      msg: 'API request started',
      method: 'POST',
      path: '/api/posts',
      userId: body.userId
    })

    const result = await createPost(body)

    logger.info({
      msg: 'API request completed',
      method: 'POST',
      path: '/api/posts',
      duration: Date.now() - startTime,
      statusCode: 200
    })

    return Response.json(result)

  } catch (error) {
    logger.error({
      msg: 'API request failed',
      method: 'POST',
      path: '/api/posts',
      duration: Date.now() - startTime,
      error: error instanceof Error ? error.message : 'Unknown error'
    })

    return Response.json(
      { error: 'Internal server error' },
      { status: 500 }
    )
  }
}
```

**ログ転送の設定**:

```typescript
// next.config.js
module.exports = {
  async rewrites() {
    return [
      {
        source: '/api/logs',
        destination: 'https://logs.example.com/ingest'
      }
    ]
  }
}

// lib/log-transport.ts
export async function sendLog(log: LogEntry) {
  if (process.env.NODE_ENV === 'production') {
    await fetch('/api/logs', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        ...log,
        timestamp: new Date().toISOString(),
        environment: process.env.NEXT_PUBLIC_VERCEL_ENV,
        release: process.env.NEXT_PUBLIC_VERCEL_GIT_COMMIT_SHA
      })
    })
  }
}
```

---

## まとめ

### 本章で学んだこと

✅ **Git Workflow**: GitHub Flow、コミット規約、PRベストプラクティス
✅ **CI/CD**: GitHub Actions による自動テスト・デプロイ
✅ **テスト戦略**: Unit/Integration/E2E テストの使い分け
✅ **デプロイメント**: Vercel への本番デプロイ、環境変数管理
✅ **監視**: Sentry によるエラー監視、パフォーマンス計測

### 実践チェックリスト

```typescript
const productionReadinessChecklist = {
  git: [
    '✓ ブランチ戦略を決定（GitHub Flow推奨）',
    '✓ コミット規約を導入（Conventional Commits）',
    '✓ PR テンプレートを作成',
    '✓ Commitlint + Husky をセットアップ'
  ],
  cicd: [
    '✓ GitHub Actions でリント・テストを自動化',
    '✓ Pull Request ごとにプレビュー環境を作成',
    '✓ main ブランチへのマージで自動デプロイ'
  ],
  testing: [
    '✓ Vitest でユニットテスト（70%カバレッジ目標）',
    '✓ Testing Library で統合テスト',
    '✓ Playwright でE2Eテスト（クリティカルフローのみ）'
  ],
  deployment: [
    '✓ Vercel にデプロイ',
    '✓ 環境変数を型安全に管理（Zod）',
    '✓ セキュリティヘッダーを設定'
  ],
  monitoring: [
    '✓ Sentry でエラー監視',
    '✓ Vercel Analytics でパフォーマンス計測',
    '✓ 構造化ロギングを実装'
  ]
}
```

### Next Steps

この本で学んだ内容を実践し、優れたReactアプリケーションを構築しましょう！

**推奨する学習リソース**:
- [Next.js 公式ドキュメント](https://nextjs.org/docs)
- [React 公式ドキュメント](https://react.dev)
- [Testing Library ドキュメント](https://testing-library.com)
- [Playwright ドキュメント](https://playwright.dev)
- [Vercel ドキュメント](https://vercel.com/docs)

---

**おわりに**

この本で学んだ内容を実践して、優れたReactアプリケーションを構築しましょう！
