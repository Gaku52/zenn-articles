---
title: "LighthouseとWebPageTest"
---

# LighthouseとWebPageTest

パフォーマンス測定ツールの使い方と、継続的な監視方法を学びます。

## Lighthouse

### CLI での実行

```bash
npm install -g lighthouse
lighthouse https://example.com --view

# CI/CD用
lighthouse https://example.com \
  --output=json \
  --output-path=./report.json \
  --chrome-flags="--headless"
```

### プログラマティック実行

```typescript
import lighthouse from 'lighthouse'
import * as chromeLauncher from 'chrome-launcher'

async function runLighthouse(url: string) {
  const chrome = await chromeLauncher.launch({ chromeFlags: ['--headless'] })

  const options = {
    logLevel: 'info',
    output: 'html',
    port: chrome.port,
  }

  const runnerResult = await lighthouse(url, options)

  console.log('Performance score:', runnerResult.lhr.categories.performance.score * 100)

  await chrome.kill()
}
```

### Lighthouse CI

```yaml
# .github/workflows/lighthouse.yml
name: Lighthouse CI
on: [pull_request]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run Lighthouse
        uses: treosh/lighthouse-ci-action@v9
        with:
          urls: |
            https://example.com
          budgetPath: ./budget.json
          uploadArtifacts: true
```

```json
// budget.json
{
  "performance": 90,
  "accessibility": 100,
  "best-practices": 90,
  "seo": 100
}
```

## WebPageTest

### API使用

```bash
# テスト実行
curl "https://www.webpagetest.org/runtest.php?url=https://example.com&k=YOUR_API_KEY&f=json"

# 結果取得
curl "https://www.webpagetest.org/jsonResult.php?test=TEST_ID"
```

### Node.js SDK

```typescript
import WebPageTest from 'webpagetest'

const wpt = new WebPageTest('www.webpagetest.org', 'YOUR_API_KEY')

wpt.runTest('https://example.com', {
  location: 'Dulles:Chrome',
  connectivity: '4G',
  runs: 3,
}, (err, result) => {
  console.log('Test ID:', result.data.testId)
  console.log('Summary URL:', result.data.summary)
})
```

## パフォーマンスバジェット

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      numberOfRuns: 3,
    },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'interaction-to-next-paint': ['error', { maxNumericValue: 200 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-byte-weight': ['error', { maxNumericValue: 1000000 }],
      },
    },
  },
}
```

## 継続的監視

### Speedlify

```javascript
// speedlify.config.js
module.exports = {
  urls: [
    'https://example.com',
    'https://example.com/about',
    'https://example.com/products',
  ],
  schedule: '0 */6 * * *', // 6時間ごと
}
```

## 改善事例

**導入前:** パフォーマンス低下に気づかず
**導入後:** CI/CDで自動チェック、デプロイ前に問題を検出、パフォーマンススコア90以上を維持
