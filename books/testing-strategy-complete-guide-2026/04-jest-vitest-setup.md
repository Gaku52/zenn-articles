---
title: "Chapter 04 - Jest/Vitest セットアップ"
---

# Chapter 04 - Jest/Vitest セットアップ

## 学習目標

この章を読み終えた後、以下ができるようになります:

✅ JestとVitestの違いを理解し、適切に選択できる
✅ TypeScript統合を含む完全なセットアップができる
✅ ウォッチモードと並列実行で開発効率を最大化できる
✅ 実プロジェクトで即座に使える設定ファイルを作成できる

**難易度**: ★★☆☆☆ (初級〜中級)
**所要時間**: 30分

---

## 1. Jest vs Vitest 比較

### 1.1 概要

#### Jest

**特徴:**
- Facebook（Meta）が開発
- デファクトスタンダード
- 豊富なエコシステム
- ゼロ設定で使える

**適している場面:**
```
✅ React/Vue/Angularプロジェクト
✅ 既存のJestエコシステムを活用したい
✅ 安定性を重視
✅ 豊富なプラグインが必要
```

#### Vitest

**特徴:**
- Viteチームが開発
- **超高速**（Jest比で5-10倍）
- ViteのHMR技術を活用
- Jest互換API

**適している場面:**
```
✅ Viteを使っているプロジェクト
✅ 高速なフィードバックが必要
✅ ESMネイティブサポートが必要
✅ モノレポ構成
```

---

### 1.2 詳細比較

| 項目 | Jest | Vitest |
|------|------|--------|
| **速度（1000テスト）** | 45秒 | 8秒 |
| **ウォッチモード** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **ESMサポート** | 実験的 | ネイティブ |
| **TypeScript** | ts-jest必要 | 組み込み |
| **並列実行** | Worker Threads | Worker Threads |
| **スナップショット** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **カバレッジ** | Istanbul | c8（より高速） |
| **エコシステム** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **学習曲線** | なだらか | なだらか |

---

### 1.3 想定パフォーマンス

**テスト環境:**
- プロジェクトサイズ: 50,000行
- テストファイル: 500個
- テストケース: 2,500個

#### 初回実行

| ツール | 実行時間 | メモリ使用量 |
|-------|---------|-----------|
| Jest | 52.3秒 | 1.2GB |
| Vitest | 9.8秒 | 680MB |

#### ウォッチモード（再実行）

| ツール | 実行時間 |
|-------|---------|
| Jest | 8.2秒 |
| Vitest | **1.3秒** |

**結論:**
- Vitest は Jest の **約5倍高速**
- ウォッチモードでは **約6倍高速**

---

### 1.4 選択ガイド

**Jestを選ぶべき場合:**

```
✅ Create React App を使っている
✅ Next.js プロジェクト
✅ 既存のJest設定を移行したくない
✅ Jest専用プラグインが必要
✅ チームがJestに慣れている
```

**Vitestを選ぶべき場合:**

```
✅ Vite を使っている
✅ 新規プロジェクト
✅ 速度を最優先したい
✅ ESMネイティブサポートが必要
✅ モダンな開発体験が欲しい
```

---

## 2. Jestのセットアップ

### 2.1 基本インストール

```bash
# Jest本体
npm install --save-dev jest

# TypeScript対応
npm install --save-dev @types/jest ts-jest

# React Testing Library（React使用時）
npm install --save-dev @testing-library/react @testing-library/jest-dom
```

---

### 2.2 設定ファイル

#### jest.config.ts（完全版）

```typescript
import type { Config } from 'jest'

const config: Config = {
  // TypeScript設定
  preset: 'ts-jest',
  testEnvironment: 'node',

  // テストファイルの場所
  roots: ['<rootDir>/src'],
  testMatch: [
    '**/__tests__/**/*.ts',
    '**/?(*.)+(spec|test).ts',
  ],

  // TypeScript変換
  transform: {
    '^.+\\.tsx?$': 'ts-jest',
  },

  // モジュール解決
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '^@components/(.*)$': '<rootDir>/src/components/$1',
    '^@utils/(.*)$': '<rootDir>/src/utils/$1',
  },

  // カバレッジ設定
  collectCoverage: true,
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/*.test.ts',
    '!src/**/*.spec.ts',
    '!src/index.ts',
    '!src/**/types.ts',
  ],

  // カバレッジしきい値
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
    // 重要なモジュールは高めに設定
    './src/services/': {
      branches: 90,
      functions: 90,
      lines: 90,
      statements: 90,
    },
  },

  // レポート形式
  coverageReporters: [
    'text',
    'text-summary',
    'html',
    'lcov',
    'json-summary',
  ],

  // セットアップファイル
  setupFilesAfterEnv: ['<rootDir>/jest.setup.ts'],

  // タイムアウト設定
  testTimeout: 10000,

  // 並列実行
  maxWorkers: '50%',

  // キャッシュ
  cache: true,
  cacheDirectory: '<rootDir>/.jest-cache',

  // 詳細設定
  verbose: true,
  clearMocks: true,
  resetMocks: true,
  restoreMocks: true,
}

export default config
```

---

### 2.3 セットアップファイル

#### jest.setup.ts

```typescript
// Jest DOM（DOMアサーションの拡張）
import '@testing-library/jest-dom'

// タイムアウト設定
jest.setTimeout(10000)

// グローバルなモック
global.fetch = jest.fn()

// コンソール警告を抑制
global.console = {
  ...console,
  warn: jest.fn(),
  error: jest.fn(),
}

// 各テスト後のクリーンアップ
afterEach(() => {
  jest.clearAllMocks()
  jest.restoreAllMocks()
})

// 環境変数の設定
process.env.NODE_ENV = 'test'
process.env.API_URL = 'http://localhost:3000'
```

---

### 2.4 TypeScript設定

#### tsconfig.json（テスト用）

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "types": ["jest", "@testing-library/jest-dom"]
  },
  "include": [
    "src/**/*.ts",
    "src/**/*.test.ts",
    "jest.setup.ts"
  ]
}
```

---

### 2.5 package.json設定

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:ci": "jest --ci --coverage --maxWorkers=2",
    "test:debug": "node --inspect-brk node_modules/.bin/jest --runInBand"
  },
  "jest": {
    "collectCoverageFrom": [
      "src/**/*.ts",
      "!src/**/*.d.ts"
    ]
  }
}
```

---

### 2.6 React向け設定

#### jest.config.ts（React用）

```typescript
import type { Config } from 'jest'

const config: Config = {
  preset: 'ts-jest',
  testEnvironment: 'jsdom', // ← Reactの場合はjsdom

  // React Testing Library設定
  setupFilesAfterEnv: ['<rootDir>/jest.setup.ts'],

  // CSS/画像モック
  moduleNameMapper: {
    '\\.(css|less|scss|sass)$': 'identity-obj-proxy',
    '\\.(jpg|jpeg|png|gif|svg)$': '<rootDir>/__mocks__/fileMock.js',
  },

  // JSX変換
  transform: {
    '^.+\\.tsx?$': [
      'ts-jest',
      {
        tsconfig: {
          jsx: 'react',
        },
      },
    ],
  },

  testMatch: [
    '**/__tests__/**/*.{ts,tsx}',
    '**/?(*.)+(spec|test).{ts,tsx}',
  ],
}

export default config
```

#### __mocks__/fileMock.js

```javascript
module.exports = 'test-file-stub'
```

---

## 3. Vitestのセットアップ

### 3.1 基本インストール

```bash
# Vitest本体
npm install --save-dev vitest

# UI（オプション）
npm install --save-dev @vitest/ui

# カバレッジ
npm install --save-dev @vitest/coverage-c8
```

---

### 3.2 設定ファイル

#### vite.config.ts（完全版）

```typescript
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],

  // パス解決
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@components': path.resolve(__dirname, './src/components'),
      '@utils': path.resolve(__dirname, './src/utils'),
    },
  },

  // Vitest設定
  test: {
    // テスト環境
    environment: 'jsdom',

    // グローバルAPI（describeなど）をインポート不要に
    globals: true,

    // セットアップファイル
    setupFiles: ['./vitest.setup.ts'],

    // カバレッジ設定
    coverage: {
      provider: 'c8',
      reporter: ['text', 'json', 'html', 'lcov'],
      include: ['src/**/*.ts', 'src/**/*.tsx'],
      exclude: [
        'src/**/*.d.ts',
        'src/**/*.test.ts',
        'src/**/*.spec.ts',
        'src/index.ts',
        'src/**/types.ts',
      ],
      all: true,
      lines: 80,
      functions: 80,
      branches: 80,
      statements: 80,
    },

    // 並列実行
    threads: true,
    isolate: true,

    // タイムアウト
    testTimeout: 10000,
    hookTimeout: 10000,

    // レポーター
    reporters: ['verbose'],

    // ウォッチモード設定
    watch: false,
    watchExclude: ['**/node_modules/**', '**/dist/**'],

    // グローバルモック
    mockReset: true,
    clearMocks: true,
    restoreMocks: true,

    // スナップショット
    resolveSnapshotPath: (testPath, snapExtension) => {
      return testPath.replace(/\.test\.([tj]sx?)/, `${snapExtension}.$1`)
    },
  },
})
```

---

### 3.3 セットアップファイル

#### vitest.setup.ts

```typescript
import { expect, afterEach } from 'vitest'
import { cleanup } from '@testing-library/react'
import * as matchers from '@testing-library/jest-dom/matchers'

// Jest DOM マッチャーの追加
expect.extend(matchers)

// 各テスト後のクリーンアップ
afterEach(() => {
  cleanup()
})

// グローバルなモック
global.fetch = vi.fn()

// 環境変数
process.env.NODE_ENV = 'test'
```

---

### 3.4 package.json設定

```json
{
  "scripts": {
    "test": "vitest",
    "test:ui": "vitest --ui",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage",
    "test:watch": "vitest watch"
  }
}
```

---

### 3.5 TypeScript設定

#### tsconfig.json

```json
{
  "compilerOptions": {
    "types": ["vitest/globals", "@testing-library/jest-dom"]
  }
}
```

---

## 4. ウォッチモード

### 4.1 Jestウォッチモード

```bash
# ウォッチモード起動
npm run test:watch

# インタラクティブモード
Watch Usage
 › Press f to run only failed tests.
 › Press o to only run tests related to changed files.
 › Press p to filter by a filename regex pattern.
 › Press t to filter by a test name regex pattern.
 › Press q to quit watch mode.
 › Press Enter to trigger a test run.
```

**便利なコマンド:**

```bash
# 変更されたファイルのみテスト
npm run test:watch -- -o

# 特定のパターンのみ
npm run test:watch -- --testPathPattern=user

# カバレッジも表示
npm run test:watch -- --coverage
```

---

### 4.2 Vitestウォッチモード

```bash
# ウォッチモード起動
npm run test:watch

# UIモード起動
npm run test:ui
```

**Vitest UI:**
- ブラウザベースのインターフェース
- テスト結果の可視化
- カバレッジマップ
- テストの再実行

**アクセス:**
```
http://localhost:51204/__vitest__/
```

---

### 4.3 パフォーマンス最適化

#### Jestの最適化

```typescript
// jest.config.ts
export default {
  // ワーカー数を制限（メモリ節約）
  maxWorkers: '50%',

  // 並列実行の最適化
  maxConcurrency: 5,

  // キャッシュ利用
  cache: true,

  // 不要なファイルをスキップ
  testPathIgnorePatterns: [
    '/node_modules/',
    '/.next/',
    '/dist/',
  ],
}
```

#### Vitestの最適化

```typescript
// vite.config.ts
export default defineConfig({
  test: {
    // スレッド数を最適化
    threads: true,
    maxThreads: 4,
    minThreads: 2,

    // 分離モード
    isolate: true,

    // 並列実行
    sequence: {
      shuffle: false,
    },
  },
})
```

---

## 5. 並列実行

### 5.1 Jestの並列実行

**デフォルト:**
- 各テストファイルが並列実行
- Worker Threadsを使用

**設定:**

```typescript
// jest.config.ts
export default {
  // CPUコア数の50%を使用
  maxWorkers: '50%',

  // または固定値
  // maxWorkers: 4,

  // 同時実行の最大数
  maxConcurrency: 5,
}
```

**CI環境向け:**

```json
{
  "scripts": {
    "test:ci": "jest --ci --maxWorkers=2"
  }
}
```

---

### 5.2 Vitestの並列実行

**デフォルト:**
- テストファイルとテストケースの両方が並列実行
- より高速

**設定:**

```typescript
// vite.config.ts
export default defineConfig({
  test: {
    // スレッド実行（デフォルト: true）
    threads: true,

    // 最大スレッド数
    maxThreads: 4,
    minThreads: 2,

    // テストの順序
    sequence: {
      shuffle: false, // ランダム実行
    },
  },
})
```

---

### 5.3 並列実行の注意点

**避けるべきパターン:**

```typescript
// ❌ 悪い例: グローバル変数を共有
let sharedState = 0

test('test 1', () => {
  sharedState++
  expect(sharedState).toBe(1) // 並列実行で失敗する可能性
})

test('test 2', () => {
  sharedState++
  expect(sharedState).toBe(2) // 並列実行で失敗する可能性
})
```

**正しいパターン:**

```typescript
// ✅ 良い例: 各テストで状態を初期化
describe('Feature', () => {
  let state: number

  beforeEach(() => {
    state = 0 // 毎回リセット
  })

  test('test 1', () => {
    state++
    expect(state).toBe(1)
  })

  test('test 2', () => {
    state++
    expect(state).toBe(1) // 独立して実行
  })
})
```

---

## 6. 実践的な設定例

### 6.1 モノレポ構成

#### packages/api/jest.config.ts

```typescript
import type { Config } from 'jest'
import baseConfig from '../../jest.config.base'

const config: Config = {
  ...baseConfig,
  displayName: 'api',
  testMatch: ['<rootDir>/src/**/*.test.ts'],
}

export default config
```

#### packages/web/vitest.config.ts

```typescript
import { defineConfig } from 'vitest/config'
import baseConfig from '../../vitest.config.base'

export default defineConfig({
  ...baseConfig,
  test: {
    ...baseConfig.test,
    name: 'web',
  },
})
```

---

### 6.2 環境別設定

```typescript
// jest.config.ts
import type { Config } from 'jest'

const isCI = process.env.CI === 'true'

const config: Config = {
  // CI環境では並列数を減らす
  maxWorkers: isCI ? 2 : '50%',

  // CI環境ではカバレッジ必須
  collectCoverage: isCI,

  // CI環境ではバイルカバレッジレポート
  coverageReporters: isCI
    ? ['text', 'lcov']
    : ['text', 'html'],

  // CI環境では詳細ログ
  verbose: isCI,
}

export default config
```

---

## 7. トラブルシューティング

### 7.1 よくある問題

#### 問題1: テストが遅い

**原因:**
- すべてのテストが直列実行されている
- モックが不足

**解決:**

```bash
# 並列実行を有効化
jest --maxWorkers=4

# または
vitest --threads
```

#### 問題2: ESMエラー

**原因:**
- JestのESMサポートが不完全

**解決:**

```typescript
// jest.config.ts
export default {
  extensionsToTreatAsEsm: ['.ts'],
  transform: {
    '^.+\\.tsx?$': [
      'ts-jest',
      {
        useESM: true,
      },
    ],
  },
}
```

**または Vitestに移行:**

```bash
npm install --save-dev vitest
```

#### 問題3: カバレッジが正確でない

**原因:**
- 除外設定が不適切

**解決:**

```typescript
// jest.config.ts
collectCoverageFrom: [
  'src/**/*.ts',
  '!src/**/*.d.ts',
  '!src/**/*.test.ts',
  '!src/**/index.ts', // バレルエクスポートを除外
  '!src/**/types.ts', // 型定義を除外
]
```

---

## 8. 実践演習

### 演習1: Jest環境構築

1. 新しいプロジェクトでJestをセットアップ
2. TypeScript統合を設定
3. カバレッジ80%のしきい値を設定
4. ウォッチモードで開発

### 演習2: Vitest環境構築

1. Viteプロジェクトを作成
2. Vitestをセットアップ
3. UIモードを起動
4. カバレッジレポートを確認

### 演習3: 移行

既存のJestプロジェクトをVitestに移行:

1. 依存関係をインストール
2. 設定ファイルを作成
3. テストを実行して動作確認
4. パフォーマンスを比較

---

## まとめ

この章で学んだこと:

✅ **Jest vs Vitest**の詳細比較（Vitestは5倍高速）
✅ **完全な設定ファイル**の作成方法
✅ **TypeScript統合**のベストプラクティス
✅ **ウォッチモード**と**並列実行**による効率化
✅ **実践的な設定例**（モノレポ、環境別設定）

次のChapterでは、実際のテスト作成に入り、より高度なテクニックを学びます。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
