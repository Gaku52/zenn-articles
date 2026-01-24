# アクセシビリティテスト - 品質を保証する

## テストの重要性

アクセシビリティは、実装だけでなくテストも重要です。自動テストと手動テストを組み合わせることで、WCAG 2.1準拠を確実にします。

## 自動テストツール

### 1. axe DevTools

**Chrome/Firefox拡張機能**

インストール:
- Chrome Web Store: "axe DevTools"で検索
- Firefox Add-ons: "axe DevTools"で検索

使い方:
1. DevToolsを開く
2. "axe DevTools"タブを選択
3. "Scan ALL of my page"をクリック

```bash
# CLIでの使用
npm install -g @axe-core/cli
axe https://example.com
```

### 2. Lighthouse

**Chrome DevTools標準装備**

使い方:
1. DevToolsを開く
2. "Lighthouse"タブを選択
3. "Accessibility"にチェック
4. "Analyze page load"をクリック

目標スコア: 90以上

### 3. eslint-plugin-jsx-a11y

**開発時の自動チェック**

```bash
pnpm add -D eslint-plugin-jsx-a11y
```

```json
// .eslintrc.json
{
  "extends": [
    "next/core-web-vitals",
    "plugin:jsx-a11y/recommended"
  ],
  "rules": {
    "jsx-a11y/alt-text": "error",
    "jsx-a11y/aria-props": "error",
    "jsx-a11y/aria-proptypes": "error",
    "jsx-a11y/aria-unsupported-elements": "error",
    "jsx-a11y/role-has-required-aria-props": "error",
    "jsx-a11y/role-supports-aria-props": "error"
  }
}
```

### 4. @axe-core/react

**開発環境で自動検査**

```bash
pnpm add -D @axe-core/react
```

```tsx
// app/layout.tsx（開発環境のみ）
if (process.env.NODE_ENV !== 'production') {
  import('@axe-core/react').then((axe) => {
    axe.default(React, ReactDOM, 1000)
  })
}
```

### 5. Playwright + axe

**E2Eテストでアクセシビリティチェック**

```bash
pnpm add -D @playwright/test @axe-core/playwright
```

```typescript
// tests/accessibility.spec.ts
import { test, expect } from '@playwright/test'
import AxeBuilder from '@axe-core/playwright'

test('ホームページのアクセシビリティ', async ({ page }) => {
  await page.goto('/')

  const accessibilityScanResults = await new AxeBuilder({ page })
    .analyze()

  expect(accessibilityScanResults.violations).toEqual([])
})

test('フォームのアクセシビリティ', async ({ page }) => {
  await page.goto('/contact')

  const accessibilityScanResults = await new AxeBuilder({ page })
    .include('#contact-form')
    .analyze()

  expect(accessibilityScanResults.violations).toEqual([])
})
```

## 手動テスト

自動テストでは検出できない問題も多いため、手動テストが必須です。

### 1. キーボード操作テスト

**手順:**
1. マウスを使わず、Tabキーだけで全ての機能を操作
2. Enter/Spaceで全てのボタンが動作するか確認
3. Escapeでモーダルが閉じるか確認
4. Arrow keysでラジオボタン、タブ等が操作できるか確認

**チェック項目:**
- 全要素にTabで到達できる
- フォーカスインジケーターが視認できる
- フォーカス順序が論理的
- キーボードトラップがない（意図的なフォーカストラップ除く）

### 2. スクリーンリーダーテスト

**macOS: VoiceOver**

起動: Command + F5

基本操作:
- 次の要素: VO + →（VO = Control + Option）
- 前の要素: VO + ←
- クリック: VO + Space
- ローター: VO + U

**Windows: NVDA（無料）**

ダウンロード: https://www.nvaccess.org/

起動: Control + Alt + N

基本操作:
- 次の要素: ↓
- 前の要素: ↑
- クリック: Enter
- 要素リスト: NVDA + F7

**チェック項目:**
- 全てのコンテンツが読み上げられる
- 画像のalt属性が適切
- フォームラベルが読み上げられる
- 見出し階層が論理的
- ライブリージョンが機能する

### 3. ズームテスト

**手順:**
1. ブラウザのズームを200%に設定
2. 全てのコンテンツが表示されるか確認
3. 水平スクロールが発生しないか確認

**Chrome:**
- 拡大: Command + + (macOS) / Control + + (Windows)
- 縮小: Command + - (macOS) / Control + - (Windows)

**チェック項目:**
- 200%ズームで全コンテンツが表示される
- 水平スクロールなし（320px幅）
- テキストが重ならない

### 4. カラーコントラストテスト

**ツール:**
- Chrome DevTools: 要素検査 > Styles > カラーピッカー
- WebAIM Contrast Checker: https://webaim.org/resources/contrastchecker/

**チェック項目:**
- 通常テキスト: 4.5:1以上
- 大きなテキスト: 3:1以上
- UI要素: 3:1以上

### 5. 色覚多様性テスト

**Chrome拡張機能:**
- "Colorblindly"
- "Color Blind Pal"

**チェック項目:**
- 色だけで情報を伝えていない
- アイコンやテキストでも情報を提供
- リンクが下線等で識別可能

## テストチェックリスト

### 自動テスト

- axe DevTools: エラー0件
- Lighthouse: Accessibilityスコア90以上
- ESLint jsx-a11y: エラー0件
- Playwright + axe: 全テストパス

### 手動テスト

#### キーボード操作
- Tabで全要素に到達できる
- Enter/Spaceで全ボタンが動作
- フォーカスインジケーターが視認できる
- モーダルでフォーカストラップが動作
- Escapeでモーダルが閉じる

#### スクリーンリーダー
- VoiceOverで全コンテンツが読み上げられる
- NVDAで全コンテンツが読み上げられる（可能であれば）
- 画像のalt属性が適切
- フォームラベルが読み上げられる
- 見出し階層が論理的

#### ズーム・リフロー
- 200%ズームで情報が失われない
- 320px幅で水平スクロールなし
- テキストが重ならない

#### カラー
- 通常テキスト: 4.5:1以上
- UI要素: 3:1以上
- 色だけで情報を伝えていない

## CI/CDへの統合

### GitHub Actions

```yaml
# .github/workflows/accessibility.yml
name: Accessibility Tests

on: [push, pull_request]

jobs:
  accessibility:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'

      - name: Install dependencies
        run: pnpm install

      - name: Build
        run: pnpm build

      - name: Run Playwright tests
        run: pnpm playwright test

      - name: Run axe CLI
        run: |
          pnpm add -g @axe-core/cli
          axe http://localhost:3000 --exit
```

### pre-commit hook

```bash
# Huskyのインストール
pnpm add -D husky lint-staged

# package.json
{
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ]
  }
}
```

## まとめ

アクセシビリティテストは、自動テストと手動テストの組み合わせが重要です。

### テスト戦略

1. **開発時**: ESLint、@axe-core/react
2. **ローカルテスト**: axe DevTools、Lighthouse、キーボード操作
3. **PR時**: Playwright + axe、GitHub Actions
4. **リリース前**: スクリーンリーダー、ズーム、カラーコントラスト

### チェックリスト

- 自動テストツールでエラー0件
- キーボードだけで全機能が操作可能
- スクリーンリーダーで全コンテンツが読み上げられる
- 200%ズームで情報が失われない
- カラーコントラストが基準を満たす

## 次のステップ

Part 3では、React/Vue/Next.jsのフレームワーク選定とベストプラクティスについて解説します。
