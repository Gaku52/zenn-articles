---
title: "パフォーマンスバジェット"
---

# パフォーマンスバジェット

パフォーマンスの目標値を設定し、それを守るための仕組みを構築します。

## バジェットの設定

### Core Web Vitals

```javascript
const performanceBudget = {
  'largest-contentful-paint': 2500, // ms
  'interaction-to-next-paint': 200, // ms
  'cumulative-layout-shift': 0.1, // score
  'first-contentful-paint': 1800, // ms
  'time-to-first-byte': 800, // ms
}
```

### リソースサイズ

```javascript
const sizeBudget = {
  'total-byte-weight': 1000000, // 1MB
  'dom-size': 1500, // nodes
  'script-size': 170000, // 170KB
  'stylesheet-size': 30000, // 30KB
  'image-size': 500000, // 500KB
}
```

## Lighthouse CI

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      numberOfRuns: 3,
    },
    assert: {
      preset: 'lighthouse:recommended',
      assertions: {
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'interaction-to-next-paint': ['error', { maxNumericValue: 200 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-byte-weight': ['error', { maxNumericValue: 1000000 }],
        'categories:performance': ['error', { minScore: 0.9 }],
      },
    },
    upload: {
      target: 'temporary-public-storage',
    },
  },
}
```

## Size Limit

```json
// package.json
{
  "size-limit": [
    {
      "path": "dist/app.js",
      "limit": "170 KB",
      "webpack": true
    },
    {
      "path": "dist/styles.css",
      "limit": "30 KB"
    }
  ],
  "scripts": {
    "size": "size-limit"
  }
}
```

## bundlesize

```json
// package.json
{
  "bundlesize": [
    {
      "path": "./dist/*.js",
      "maxSize": "170 kB"
    },
    {
      "path": "./dist/*.css",
      "maxSize": "30 kB"
    }
  ]
}
```

## CI/CD統合

```yaml
# .github/workflows/performance-budget.yml
name: Performance Budget
on: [pull_request]

jobs:
  check-budget:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Check bundle size
        run: npm run size

      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v9
        with:
          urls: |
            http://localhost:3000
          budgetPath: ./lighthouserc.js
          uploadArtifacts: true
```

## Webpack Bundle Analyzer

```javascript
// webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      reportFilename: 'bundle-report.html',
      openAnalyzer: false,
      generateStatsFile: true,
      statsOptions: { source: false },
    }),
  ],
}
```

## チーム内での運用

### バジェット超過時のフロー

1. **PR作成時**: 自動チェック実行
2. **バジェット超過**: PRをブロック
3. **対応方法の検討**:
   - コード最適化
   - 遅延読み込み
   - バジェットの見直し（正当な理由がある場合）
4. **承認**: バジェット内に収まったらマージ

### 定期レビュー

```yaml
# 毎週月曜日にパフォーマンスレポートを生成
schedule:
  - cron: '0 9 * * 1'

jobs:
  weekly-report:
    runs-on: ubuntu-latest
    steps:
      - name: Generate report
        run: npm run perf:report

      - name: Send to Slack
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -H 'Content-Type: application/json' \
            -d @./report.json
```

## 改善事例

**導入前:** バンドルサイズが徐々に増加、気づいたら2MB超え
**導入後:** PRごとに自動チェック、バジェット超過をマージ前に検出、バンドルサイズを300KB以下に維持
