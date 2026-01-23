---
title: "バンドルサイズ分析"
---

# バンドルサイズ分析

## この章で学ぶこと

バンドルサイズはページの読み込み速度に直接影響します。この章では、バンドルサイズを分析し、最適化するための実践的な手法を学びます。

- バンドルサイズが

パフォーマンスに与える影響
- Webpack Bundle Analyzerの使い方
- source-map-explorerでの分析
- バンドルサイズの目標設定
- 不要な依存関係の特定
- Tree Shakingの効果測定
- 実際の最適化事例

## バンドルサイズの影響

### ダウンロード時間

| バンドルサイズ | 3G (750kbps) | 4G (4Mbps) | Fiber (50Mbps) |
|---------------|--------------|------------|----------------|
| 100KB | 1.1秒 | 0.2秒 | 0.02秒 |
| 500KB | 5.3秒 | 1.0秒 | 0.08秒 |
| 1MB | 10.7秒 | 2.0秒 | 0.16秒 |
| 3MB | 32秒 | 6.0秒 | 0.48秒 |

### パース・コンパイル時間

JavaScriptは解凍→パース→コンパイル→実行の順で処理されます。

```
1MBのJavaScript = ダウンロード2秒 + パース0.5秒 + コンパイル0.3秒 = 計2.8秒
```

## Webpack Bundle Analyzer

### インストールと設定

```bash
npm install --save-dev webpack-bundle-analyzer
```

```javascript
// webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      openAnalyzer: false,
      reportFilename: 'bundle-report.html',
    }),
  ],
}
```

### Next.jsでの使用

```bash
npm install --save-dev @next/bundle-analyzer
```

```javascript
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
})

module.exports = withBundleAnalyzer({
  // Next.js config
})
```

```bash
# 実行
ANALYZE=true npm run build
```

### Viteでの使用

```bash
npm install --save-dev rollup-plugin-visualizer
```

```javascript
// vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer'

export default {
  plugins: [
    visualizer({
      open: true,
      gzipSize: true,
      brotliSize: true,
    }),
  ],
}
```

## バンドルサイズの分析手法

### 1. 大きな依存関係の特定

```bash
# 各パッケージのサイズを確認
npm install -g cost-of-modules
cost-of-modules

# またはbundlephobia
npx bundle-phobia <package-name>
```

**例: moment.js vs date-fns**

| ライブラリ | サイズ（minified） | サイズ（gzipped） |
|-----------|------------------|------------------|
| moment.js | 232KB | 71KB |
| date-fns | 76KB | 22KB |
| dayjs | 7KB | 3KB |

**置き換え例:**

```typescript
// Before: moment.js (71KB gzipped)
import moment from 'moment'
const date = moment().format('YYYY-MM-DD')

// After: date-fns (22KB gzipped)
import { format } from 'date-fns'
const date = format(new Date(), 'yyyy-MM-dd')

// Better: dayjs (3KB gzipped)
import dayjs from 'dayjs'
const date = dayjs().format('YYYY-MM-DD')
```

### 2. 重複コードの検出

```bash
npm install --save-dev duplicate-package-checker-webpack-plugin
```

```javascript
// webpack.config.js
const DuplicatePackageCheckerPlugin = require('duplicate-package-checker-webpack-plugin')

module.exports = {
  plugins: [
    new DuplicatePackageCheckerPlugin({
      verbose: true,
      emitError: true,
    }),
  ],
}
```

### 3. 使用していないコードの検出

```bash
npx depcheck
```

```javascript
// .depcheckrc
{
  "ignores": [
    "@types/*",
    "eslint*",
    "prettier"
  ],
  "skip-missing": false
}
```

## バンドルサイズの目標設定

### Google推奨値

- **初回ページ読み込み:** 200KB以下（gzipped）
- **JavaScriptバンドル:** 170KB以下（gzipped）
- **CSSバンドル:** 30KB以下（gzipped）

### パフォーマンスバジェット

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    assert: {
      assertions: {
        'total-byte-weight': ['error', { maxNumericValue: 1000000 }], // 1MB
        'dom-size': ['error', { maxNumericValue: 1500 }],
        'bootup-time': ['error', { maxNumericValue: 3500 }],
      },
    },
  },
}
```

## 実際の最適化事例

### 事例1: lodashの最適化

**Before:**

```typescript
import _ from 'lodash'

const result = _.uniq(array)
const sorted = _.sortBy(data, 'name')
```

**Bundle size:** 72KB (minified + gzipped)

**After:**

```typescript
import uniq from 'lodash/uniq'
import sortBy from 'lodash/sortBy'

const result = uniq(array)
const sorted = sortBy(data, 'name')
```

**Bundle size:** 8KB (minified + gzipped)
**削減率:** 89%

### 事例2: アイコンライブラリの最適化

**Before:**

```typescript
import { FaHome, FaUser, FaSettings } from 'react-icons/fa'
```

**Bundle size:** 680KB

**After:**

```typescript
import FaHome from 'react-icons/fa/FaHome'
import FaUser from 'react-icons/fa/FaUser'
import FaSettings from 'react-icons/fa/FaSettings'
```

**Bundle size:** 12KB
**削減率:** 98%

### 事例3: Chart.jsの最適化

**Before:**

```typescript
import Chart from 'chart.js/auto'
```

**Bundle size:** 242KB

**After:**

```typescript
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  BarElement,
  Title,
  Tooltip,
} from 'chart.js'

ChartJS.register(CategoryScale, LinearScale, BarElement, Title, Tooltip)
```

**Bundle size:** 58KB
**削減率:** 76%

## バンドルサイズ監視の自動化

### GitHub Actions

```yaml
# .github/workflows/bundle-size.yml
name: Bundle Size Check

on: [pull_request]

jobs:
  check-size:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install dependencies
        run: npm ci
      - name: Build
        run: npm run build
      - name: Check bundle size
        uses: andresz1/size-limit-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

### Size Limit

```bash
npm install --save-dev @size-limit/preset-app
```

```json
// package.json
{
  "size-limit": [
    {
      "path": "dist/bundle.js",
      "limit": "170 KB"
    },
    {
      "path": "dist/styles.css",
      "limit": "30 KB"
    }
  ]
}
```

## まとめ

バンドルサイズ分析の重要なポイント:

1. **定期的な分析**
   - Webpack Bundle Analyzer
   - Lighthouse CI
   - Size Limit

2. **大きな依存関係の置き換え**
   - moment.js → dayjs
   - lodash → lodash/es + Tree Shaking

3. **パフォーマンスバジェットの設定**
   - 目標値: 200KB以下
   - CI/CDで自動チェック

4. **継続的な監視**
   - GitHub Actions
   - Bundlephobia
   - npm audit

次の章では、コード分割の戦略について学びます。
