# アクセシビリティ自動化 - CI/CDへの統合

## 自動化の重要性

アクセシビリティテストをCI/CDパイプラインに統合することで、継続的に品質を保証できます。

## GitHub Actions設定

### 基本的なワークフロー

```yaml
# .github/workflows/accessibility.yml
name: Accessibility Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  accessibility:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Setup pnpm
        uses: pnpm/action-setup@v2
        with:
          version: 8

      - name: Install dependencies
        run: pnpm install

      - name: ESLint
        run: pnpm lint

      - name: Build
        run: pnpm build

      - name: Start server
        run: |
          pnpm start &
          sleep 10

      - name: Run Playwright tests
        run: pnpm playwright test

      - name: Upload Playwright report
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: playwright-report/
```

### Lighthouseの統合

```yaml
# .github/workflows/lighthouse.yml
name: Lighthouse CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lighthouse:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Setup pnpm
        uses: pnpm/action-setup@v2
        with:
          version: 8

      - name: Install dependencies
        run: pnpm install

      - name: Build
        run: pnpm build

      - name: Run Lighthouse CI
        run: |
          pnpm add -g @lhci/cli
          lhci autorun
```

### lighthouserc.js

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      startServerCommand: 'pnpm start',
      url: ['http://localhost:3000'],
      numberOfRuns: 3,
    },
    assert: {
      assertions: {
        'categories:accessibility': ['error', { minScore: 0.9 }],
        'categories:performance': ['warn', { minScore: 0.8 }],
        'categories:seo': ['warn', { minScore: 0.9 }],
      },
    },
    upload: {
      target: 'temporary-public-storage',
    },
  },
}
```

## Playwrightでのアクセシビリティテスト

### playwright.config.ts

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
  },

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
  ],

  webServer: {
    command: 'pnpm dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
})
```

### テストの実装

```typescript
// tests/accessibility.spec.ts
import { test, expect } from '@playwright/test'
import AxeBuilder from '@axe-core/playwright'

test.describe('アクセシビリティテスト', () => {
  test('ホームページ', async ({ page }) => {
    await page.goto('/')

    const accessibilityScanResults = await new AxeBuilder({ page })
      .analyze()

    expect(accessibilityScanResults.violations).toEqual([])
  })

  test('お問い合わせページ', async ({ page }) => {
    await page.goto('/contact')

    const accessibilityScanResults = await new AxeBuilder({ page })
      .analyze()

    expect(accessibilityScanResults.violations).toEqual([])
  })

  test('ダッシュボード（認証後）', async ({ page }) => {
    // ログイン
    await page.goto('/login')
    await page.fill('#email', 'test@example.com')
    await page.fill('#password', 'password')
    await page.click('button[type="submit"]')

    await page.waitForURL('/dashboard')

    const accessibilityScanResults = await new AxeBuilder({ page })
      .analyze()

    expect(accessibilityScanResults.violations).toEqual([])
  })

  test('特定の要素のみテスト', async ({ page }) => {
    await page.goto('/')

    const accessibilityScanResults = await new AxeBuilder({ page })
      .include('#header') // header要素のみテスト
      .analyze()

    expect(accessibilityScanResults.violations).toEqual([])
  })

  test('特定のルールを除外', async ({ page }) => {
    await page.goto('/')

    const accessibilityScanResults = await new AxeBuilder({ page })
      .disableRules(['color-contrast']) // 一時的に除外
      .analyze()

    expect(accessibilityScanResults.violations).toEqual([])
  })
})
```

## pre-commit hook

### Huskyのセットアップ

```bash
pnpm add -D husky lint-staged
pnpm dlx husky init
```

```json
// package.json
{
  "scripts": {
    "prepare": "husky install"
  },
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ]
  }
}
```

```bash
# .husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

pnpm lint-staged
```

## ESLint設定

```json
// .eslintrc.json
{
  "extends": [
    "next/core-web-vitals",
    "plugin:jsx-a11y/recommended"
  ],
  "rules": {
    "jsx-a11y/alt-text": "error",
    "jsx-a11y/anchor-has-content": "error",
    "jsx-a11y/anchor-is-valid": "error",
    "jsx-a11y/aria-activedescendant-has-tabindex": "error",
    "jsx-a11y/aria-props": "error",
    "jsx-a11y/aria-proptypes": "error",
    "jsx-a11y/aria-role": "error",
    "jsx-a11y/aria-unsupported-elements": "error",
    "jsx-a11y/click-events-have-key-events": "error",
    "jsx-a11y/heading-has-content": "error",
    "jsx-a11y/html-has-lang": "error",
    "jsx-a11y/iframe-has-title": "error",
    "jsx-a11y/img-redundant-alt": "error",
    "jsx-a11y/interactive-supports-focus": "error",
    "jsx-a11y/label-has-associated-control": "error",
    "jsx-a11y/media-has-caption": "warn",
    "jsx-a11y/mouse-events-have-key-events": "error",
    "jsx-a11y/no-access-key": "error",
    "jsx-a11y/no-autofocus": "warn",
    "jsx-a11y/no-noninteractive-element-interactions": "error",
    "jsx-a11y/no-noninteractive-tabindex": "error",
    "jsx-a11y/no-redundant-roles": "error",
    "jsx-a11y/no-static-element-interactions": "error",
    "jsx-a11y/role-has-required-aria-props": "error",
    "jsx-a11y/role-supports-aria-props": "error",
    "jsx-a11y/scope": "error",
    "jsx-a11y/tabindex-no-positive": "error"
  }
}
```

## 自動化のベストプラクティス

### 段階的な導入

1. **開発時**: ESLint、@axe-core/react
2. **コミット時**: lint-staged、Husky
3. **PR時**: Playwright + axe、Lighthouse CI
4. **リリース前**: 手動テスト（スクリーンリーダー等）

### テスト戦略

| テストタイプ | 実行タイミング | ツール |
|------------|--------------|-------|
| Lint | 開発時、コミット時 | ESLint |
| 自動テスト | PR時 | Playwright + axe |
| Lighthouse | PR時 | Lighthouse CI |
| 手動テスト | リリース前 | VoiceOver、NVDA |

### CI/CDパイプライン

```
開発 → コミット → PR → マージ → デプロイ
  ↓       ↓       ↓      ↓        ↓
ESLint  Husky  Playwright Main  手動テスト
              Lighthouse  Build
```

## レポート生成

### Playwright HTMLレポート

```bash
pnpm playwright show-report
```

### Lighthouse CI レポート

GitHub ActionsでLighthouse CIを実行すると、一時的なURLでレポートが公開されます。

## まとめ

アクセシビリティテストの自動化は、品質を継続的に保証するために不可欠です。

### チェックリスト

- ESLint jsx-a11yプラグインが設定されている
- Playwright + axeでE2Eテスト
- Lighthouse CIでスコア監視
- GitHub Actionsでテスト自動実行
- pre-commit hookでlint-staged実行
- 手動テストの定期実行（リリース前）

### 最終チェックリスト

本書全体を通じて学んだことの確認：

#### Part 1: WCAG 2.1準拠
- WCAG 2.1の4原則（POUR）を理解
- Level AA準拠を目標
- 78の達成基準を確認

#### Part 2: アクセシビリティ実装
- セマンティックHTMLを使用
- ARIA属性を適切に使用
- キーボード操作可能
- スクリーンリーダー対応
- カラーコントラスト4.5:1以上
- アクセシビリティテスト実施

#### Part 3: モダンWeb開発
- フレームワーク選定（Next.js推奨）
- React/Vue/Next.jsのベストプラクティス
- 状態管理（Zustand、TanStack Query）

#### Part 4: 技術ドキュメント
- 誠実性と正確性を重視
- READMEが充実
- API仕様書が明確
- アーキテクチャ図がある

#### Part 5: 自動化
- CI/CDパイプライン構築
- 自動テスト + 手動テスト
- 継続的な品質保証

## おわりに

Web品質とアクセシビリティは、全てのユーザーにとって使いやすいWebサイトを作るための基盤です。本書で学んだ知識を活用して、より良いWebアプリケーションを開発してください。

アクセシビリティは、特定のユーザーのためだけでなく、全てのユーザーの体験を向上させます。継続的な改善と学習を心がけ、誰もが使えるWebを実現しましょう。

---

ご購読ありがとうございました。質問やフィードバックは、GitHubのIssuesやDiscussionsでお待ちしています。

_Last updated: 2026-01-24_
