---
title: "Tree ShakingとDead Code Elimination"
---

# Tree ShakingとDead Code Elimination

## この章で学ぶこと

- Tree Shakingの仕組み
- ESモジュールの重要性
- sideEffectsフラグの活用
- 使用されていないコードの検出
- Terserでの最適化設定
- 実践的な最適化例

## Tree Shakingとは

Tree Shakingは、使用されていないコードを自動的に削除するビルドプロセスの最適化手法です。

### 基本原理

```typescript
// utils.ts
export function usedFunction() {
  return 'Used'
}

export function unusedFunction() {
  return 'Unused'
}

// main.ts
import { usedFunction } from './utils'
console.log(usedFunction())
// unusedFunction はバンドルに含まれない
```

## ESモジュールの重要性

Tree ShakingはESモジュールの静的構造に依存しています。

### CommonJS vs ESモジュール

**悪い例（CommonJS）:**

```javascript
// Tree Shaking不可
const utils = require('./utils')
utils.someFunction()
```

**良い例（ESモジュール）:**

```javascript
// Tree Shaking可能
import { someFunction } from './utils'
someFunction()
```

### package.jsonの設定

```json
{
  "name": "my-library",
  "version": "1.0.0",
  "main": "./dist/index.cjs",
  "module": "./dist/index.mjs",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  },
  "sideEffects": false
}
```

## sideEffectsフラグ

### 基本設定

```json
// package.json
{
  "sideEffects": false  // 全ファイルに副作用なし
}
```

### 特定ファイルを除外

```json
{
  "sideEffects": [
    "*.css",
    "*.scss",
    "./src/polyfills.ts"
  ]
}
```

### 実例

```typescript
// Button.tsx
import './Button.css'  // 副作用あり
export function Button() {
  return <button>Click</button>
}
```

```json
{
  "sideEffects": ["**/*.css", "**/*.scss"]
}
```

## 実践的な最適化

### 1. Lodashの最適化

**Before:**

```typescript
import _ from 'lodash'
const result = _.uniq(array)
```

**Bundle size:** 72KB

**After:**

```typescript
import uniq from 'lodash-es/uniq'
const result = uniq(array)
```

**Bundle size:** 3KB
**削減率:** 96%

### 2. Material-UIの最適化

**Before:**

```typescript
import { Button, TextField } from '@mui/material'
```

**Bundle size:** 420KB

**After:**

```typescript
import Button from '@mui/material/Button'
import TextField from '@mui/material/TextField'
```

**Bundle size:** 85KB
**削減率:** 80%

### 3. Date-fnsの最適化

**Before:**

```typescript
import * as dateFns from 'date-fns'
dateFns.format(new Date(), 'yyyy-MM-dd')
```

**After:**

```typescript
import { format } from 'date-fns'
format(new Date(), 'yyyy-MM-dd')
```

## Webpack設定

```javascript
// webpack.config.js
module.exports = {
  mode: 'production',
  optimization: {
    usedExports: true,  // Tree Shaking有効化
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            dead_code: true,
            drop_console: true,
            drop_debugger: true,
            pure_funcs: ['console.log'],
          },
        },
      }),
    ],
  },
}
```

## Rollup設定

```javascript
// rollup.config.js
import { terser } from 'rollup-plugin-terser'

export default {
  input: 'src/index.js',
  output: {
    file: 'dist/bundle.js',
    format: 'esm',
  },
  plugins: [
    terser({
      compress: {
        dead_code: true,
        drop_console: true,
      },
    }),
  ],
  treeshake: {
    moduleSideEffects: false,
    propertyReadSideEffects: false,
  },
}
```

## 検証方法

### Bundle Analyzerで確認

```bash
npm run build
# bundle-report.htmlで使用されているコードを確認
```

### コンソールでの確認

```bash
# Before
ls -lh dist/bundle.js
# 450KB

# After (Tree Shaking)
ls -lh dist/bundle.js
# 120KB
```

## 改善事例

### 事例: React アプリケーション

**Before:**
- Bundle: 890KB
- Tree Shaking未設定

**After:**

```json
// package.json
{
  "sideEffects": ["*.css", "src/polyfills.ts"]
}
```

```javascript
// webpack.config.js
optimization: {
  usedExports: true,
  sideEffects: true,
}
```

**結果:**
- Bundle: 890KB → 320KB (64%削減)
- LCP: 3.1秒 → 1.4秒 (55%改善)

## まとめ

Tree Shakingの重要ポイント:

1. **ESモジュール使用**
   - import/export構文
   - CommonJS避ける

2. **sideEffectsフラグ設定**
   - package.jsonで明示

3. **named importの使用**
   - default importよりTreeShaking効果大

4. **定期的な検証**
   - Bundle Analyzer
   - ビルドサイズ監視

次の章では、Dynamic Importsについて学びます。
