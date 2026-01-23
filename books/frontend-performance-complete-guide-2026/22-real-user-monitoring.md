---
title: "Real User Monitoring"
---

# Real User Monitoring

実際のユーザーのパフォーマンスデータを収集・分析する方法を学びます。

## web-vitals ライブラリ

```typescript
import { onCLS, onINP, onLCP, onFCP, onTTFB } from 'web-vitals'

function sendToAnalytics(metric) {
  const body = JSON.stringify(metric)

  if (navigator.sendBeacon) {
    navigator.sendBeacon('/analytics', body)
  } else {
    fetch('/analytics', {
      body,
      method: 'POST',
      keepalive: true,
    })
  }
}

onCLS(sendToAnalytics)
onINP(sendToAnalytics)
onLCP(sendToAnalytics)
onFCP(sendToAnalytics)
onTTFB(sendToAnalytics)
```

## Google Analytics 4

```typescript
import { onCLS, onINP, onLCP } from 'web-vitals'

function sendToGoogleAnalytics({ name, delta, value, id }) {
  gtag('event', name, {
    event_category: 'Web Vitals',
    value: Math.round(name === 'CLS' ? delta * 1000 : delta),
    event_label: id,
    non_interaction: true,
  })
}

onCLS(sendToGoogleAnalytics)
onINP(sendToGoogleAnalytics)
onLCP(sendToGoogleAnalytics)
```

## Vercel Analytics

```typescript
// _app.tsx
import { Analytics } from '@vercel/analytics/react'

export default function App({ Component, pageProps }) {
  return (
    <>
      <Component {...pageProps} />
      <Analytics />
    </>
  )
}
```

## カスタムメトリクス

```typescript
// Next.jsのカスタムメトリクス
export function reportWebVitals(metric) {
  console.log(metric)

  switch (metric.name) {
    case 'FCP':
      // First Contentful Paint
      break
    case 'LCP':
      // Largest Contentful Paint
      break
    case 'CLS':
      // Cumulative Layout Shift
      break
    case 'FID':
      // First Input Delay
      break
    case 'TTFB':
      // Time to First Byte
      break
    case 'Next.js-hydration':
      // Hydration時間
      break
    case 'Next.js-route-change-to-render':
      // ルート変更からレンダリングまで
      break
    case 'Next.js-render':
      // レンダリング時間
      break
  }
}
```

## ダッシュボード構築

```typescript
// データベース保存
async function saveMetric(metric) {
  await prisma.metric.create({
    data: {
      name: metric.name,
      value: metric.value,
      url: window.location.href,
      userAgent: navigator.userAgent,
      timestamp: new Date(),
    },
  })
}

// 集計クエリ
async function getMetricsSummary() {
  return await prisma.metric.groupBy({
    by: ['name'],
    _avg: { value: true },
    _max: { value: true },
    where: {
      timestamp: {
        gte: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000), // 過去7日
      },
    },
  })
}
```

## パフォーマンス監視SaaS

| サービス | 特徴 | 価格 |
|---------|------|------|
| Vercel Analytics | Next.js統合 | 無料〜 |
| Sentry | エラー + パフォーマンス | 無料〜 |
| New Relic | フルスタック監視 | 有料 |
| Datadog RUM | エンタープライズ | 有料 |

## アラート設定

```typescript
// 閾値を超えたらSlack通知
async function checkPerformance() {
  const metrics = await getMetrics()

  if (metrics.lcp > 2500) {
    await fetch('https://hooks.slack.com/services/YOUR_WEBHOOK', {
      method: 'POST',
      body: JSON.stringify({
        text: `⚠️ LCP is ${metrics.lcp}ms (threshold: 2500ms)`,
      }),
    })
  }
}
```

## 改善事例

**導入前:** パフォーマンス問題に気づかない
**導入後:** リアルタイムで問題を検出、地域別・デバイス別の分析、パフォーマンス低下を即座に対応
