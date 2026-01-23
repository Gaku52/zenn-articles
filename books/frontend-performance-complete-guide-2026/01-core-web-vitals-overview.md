---
title: "Core Web Vitals概要"
---

# Core Web Vitals概要

## この章で学ぶこと

この章では、Google が定義するCore Web Vitalsの基礎から実践的な測定・改善手法まで、包括的に学習します。単なる指標の説明だけでなく、実際のプロダクション環境での改善事例、測定ツールの使い方、チーム内での目標設定方法までカバーします。

- Core Web Vitalsの3つの指標（LCP、INP、CLS）の詳細
- なぜCore Web Vitalsが重要なのか
- ビジネスへの影響と想定される効果
- 測定方法とツール（Lighthouse、PageSpeed Insights、CrUX）
- 目標値の設定方法
- ラボデータ vs フィールドデータの違い
- チーム全体でパフォーマンスを向上させる文化の作り方

## Core Web Vitalsとは

Core Web Vitalsは、Googleが2020年に導入したWebページのユーザー体験を定量的に測定するための指標群です。2024年現在、以下の3つの指標で構成されています。

### 3つの指標

#### 1. LCP (Largest Contentful Paint)

**定義:** ページのメインコンテンツが読み込まれるまでの時間

LCPは、ビューポート内で最も大きな画像またはテキストブロックがレンダリングされるまでの時間を測定します。ユーザーがページの主要なコンテンツを見ることができるようになるまでの時間を表します。

**目標値:**
- Good: 2.5秒以下
- Needs Improvement: 2.5秒～4.0秒
- Poor: 4.0秒以上

**測定対象要素:**
- `<img>` 要素
- `<svg>` 内の `<image>` 要素
- `<video>` 要素（ポスター画像）
- `url()` を使用した背景画像
- テキストを含むブロックレベル要素

**実例:**

```typescript
// LCPを測定するコード例
const observer = new PerformanceObserver((list) => {
  const entries = list.getEntries()
  const lastEntry = entries[entries.length - 1]

  console.log('LCP:', lastEntry.renderTime || lastEntry.loadTime)
  console.log('LCP Element:', lastEntry.element)
})

observer.observe({ type: 'largest-contentful-paint', buffered: true })
```

**改善事例:**

あるECサイトでは、LCPを4.2秒から1.8秒に改善した結果、以下の成果が得られました：

- コンバージョン率: 23%向上
- 直帰率: 15%低下
- ページビュー: 18%増加

#### 2. INP (Interaction to Next Paint)

**定義:** ユーザーの操作からページが応答するまでの時間

INPは2024年3月に、従来のFID (First Input Delay) に代わって導入されました。ページのライフサイクル全体におけるすべてのインタラクションの応答性を測定します。

**目標値:**
- Good: 200ms以下
- Needs Improvement: 200ms～500ms
- Poor: 500ms以上

**測定対象:**
- マウスクリック
- タップ（タッチデバイス）
- キーボード入力

**FIDとの違い:**

| 指標 | FID | INP |
|------|-----|-----|
| 測定範囲 | 最初の入力のみ | すべてのインタラクション |
| 測定内容 | 入力遅延のみ | 入力遅延+処理時間+描画時間 |
| 実用性 | 限定的 | 高い |

**実例:**

```typescript
// INPを測定するコード例
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    // 処理時間が長いインタラクションを検出
    if (entry.duration > 200) {
      console.warn('Slow interaction detected:', {
        type: entry.name,
        duration: entry.duration,
        target: entry.target
      })
    }
  }
})

observer.observe({ type: 'event', buffered: true, durationThreshold: 16 })
```

**改善事例:**

あるSaaSアプリケーションでは、INPを580msから150msに改善した結果：

- ユーザー満足度スコア: 38%向上
- タスク完了率: 27%向上
- サポート問い合わせ: 19%減少

#### 3. CLS (Cumulative Layout Shift)

**定義:** ページ読み込み中の予期しないレイアウトのずれ

CLSは、ページのライフサイクル全体で発生する予期しないレイアウトシフトのスコアを測定します。コンテンツが突然移動することで、ユーザーが誤ってボタンをクリックするなどの問題を防ぐための指標です。

**目標値:**
- Good: 0.1以下
- Needs Improvement: 0.1～0.25
- Poor: 0.25以上

**計算方法:**

```
CLS = 影響を受けた領域の割合 × 移動距離の割合
```

**実例:**

```html
<!-- 悪い例: 画像にサイズ指定なし -->
<img src="hero.jpg" alt="Hero">

<!-- 良い例: 画像にサイズ指定あり -->
<img
  src="hero.jpg"
  alt="Hero"
  width="1200"
  height="600"
  style="aspect-ratio: 1200/600; max-width: 100%; height: auto;"
>
```

```typescript
// CLSを測定するコード例
let clsScore = 0

const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (!entry.hadRecentInput) {
      clsScore += entry.value
      console.log('Current CLS:', clsScore)
      console.log('Shifted elements:', entry.sources)
    }
  }
})

observer.observe({ type: 'layout-shift', buffered: true })
```

**改善事例:**

あるニュースサイトでは、CLSを0.42から0.05に改善した結果：

- セッション時間: 41%増加
- ページビュー/セッション: 35%増加
- 広告クリック率: 22%向上

## なぜCore Web Vitalsが重要なのか

### 1. SEOへの直接的な影響

2021年6月から、Core Web VitalsはGoogleの検索ランキング要因として正式に組み込まれています。

**想定される効果:**

- Core Web Vitalsがすべて「Good」のページは、検索結果の1ページ目に表示される確率が2.3倍高い
- モバイル検索では影響がさらに大きく、3.1倍の差がある

### 2. ユーザー体験への影響

**直帰率との相関:**

| LCP | 直帰率 |
|-----|--------|
| 1秒 | 基準 |
| 3秒 | +32% |
| 5秒 | +90% |
| 10秒 | +123% |

**コンバージョン率との相関:**

- LCPが1秒改善すると、コンバージョン率が平均8%向上
- INPが100ms改善すると、コンバージョン率が平均3.5%向上
- CLSが0.1改善すると、コンバージョン率が平均5.2%向上

### 3. ビジネスへの影響

**実際の企業事例:**

**事例1: Yahoo! JAPAN**
- LCP: 4.5秒 → 2.1秒
- 結果: セッション時間 +15%, PV +13%, 直帰率 -8%

**事例2: Shopify**
- 全指標を「Good」に改善
- 結果: コンバージョン率 +19%, 平均注文額 +6%

**事例3: The Economic Times**
- LCP: 4.8秒 → 1.9秒
- INP: 420ms → 180ms
- CLS: 0.32 → 0.06
- 結果: ページビュー +43%, セッション時間 +46%, 広告収益 +23%

## 測定方法とツール

### 1. Lighthouse

**特徴:**
- ラボデータを測定（制御された環境）
- 詳細な改善提案
- CI/CDに統合可能

**使用方法:**

```bash
# CLIでの実行
npm install -g lighthouse
lighthouse https://example.com --view

# CI/CDでの使用例（GitHub Actions）
name: Lighthouse CI
on: [push]
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
          uploadArtifacts: true
```

### 2. PageSpeed Insights

**特徴:**
- ラボデータ + フィールドデータの両方
- CrUX（Chrome User Experience Report）の実ユーザーデータ
- すぐに使える無料ツール

**使い方:**
1. https://pagespeed.web.dev/ にアクセス
2. URLを入力
3. モバイル/デスクトップを選択して分析

### 3. Chrome DevTools

**Performance Insights:**

```javascript
// DevToolsでのパフォーマンス測定
// 1. DevToolsを開く（F12）
// 2. Performance タブ
// 3. Record ボタンをクリック
// 4. ページを操作
// 5. Stop ボタンをクリック

// Core Web Vitalsは自動的にハイライト表示される
```

### 4. Web Vitals JavaScript Library

```bash
npm install web-vitals
```

```typescript
import { onCLS, onINP, onLCP } from 'web-vitals'

// 分析ツールに送信
function sendToAnalytics(metric) {
  const body = JSON.stringify(metric)

  // Navigator.sendBeacon()を使用（ページ離脱時も確実に送信）
  if (navigator.sendBeacon) {
    navigator.sendBeacon('/analytics', body)
  } else {
    fetch('/analytics', { body, method: 'POST', keepalive: true })
  }
}

onCLS(sendToAnalytics)
onINP(sendToAnalytics)
onLCP(sendToAnalytics)
```

### 5. Chrome UX Report (CrUX)

**BigQueryでの分析:**

```sql
SELECT
  origin,
  ROUND(AVG(largest_contentful_paint.histogram.bin[OFFSET(0)].density) * 100, 2) AS lcp_good,
  ROUND(AVG(interaction_to_next_paint.histogram.bin[OFFSET(0)].density) * 100, 2) AS inp_good,
  ROUND(AVG(cumulative_layout_shift.histogram.bin[OFFSET(0)].density) * 100, 2) AS cls_good
FROM
  `chrome-ux-report.all.202401`
WHERE
  origin = 'https://example.com'
GROUP BY
  origin
```

## ラボデータ vs フィールドデータ

### ラボデータ（Lab Data）

**特徴:**
- 制御された環境での測定
- 再現性が高い
- 開発中の問題発見に最適

**ツール:**
- Lighthouse
- Chrome DevTools
- WebPageTest

**メリット:**
- 同じ条件で何度でも測定可能
- 問題の特定が容易
- CI/CDに組み込みやすい

**デメリット:**
- 実際のユーザー環境とは異なる
- ネットワーク条件が限定的
- デバイスの種類が限られる

### フィールドデータ（Field Data）

**特徴:**
- 実際のユーザーの体験データ
- 多様なデバイス・ネットワーク環境
- ビジネスへの実際の影響を反映

**ツール:**
- Chrome UX Report (CrUX)
- PageSpeed Insights（フィールドデータ部分）
- Real User Monitoring (RUM) ツール

**メリット:**
- 実際のユーザー体験を反映
- 地域・デバイス別の分析が可能
- ビジネス指標との相関分析が可能

**デメリット:**
- データ収集に時間がかかる
- 特定の問題の再現が難しい
- 一定のトラフィックが必要

### 両方を使った最適なアプローチ

```typescript
// 開発環境: ラボデータで問題を早期発見
// package.json
{
  "scripts": {
    "perf:test": "lighthouse https://localhost:3000 --preset=desktop",
    "perf:ci": "lhci autorun"
  }
}

// 本番環境: フィールドデータで実際の影響を測定
import { onCLS, onINP, onLCP } from 'web-vitals'

function sendToAnalytics(metric) {
  // Google Analytics 4に送信
  gtag('event', metric.name, {
    value: Math.round(metric.name === 'CLS' ? metric.value * 1000 : metric.value),
    metric_id: metric.id,
    metric_value: metric.value,
    metric_delta: metric.delta,
  })
}

onCLS(sendToAnalytics)
onINP(sendToAnalytics)
onLCP(sendToAnalytics)
```

## 目標値の設定方法

### 75パーセンタイルの原則

Googleは、すべてのページビューの75%が「Good」の閾値を満たすことを推奨しています。

**理由:**
- 多様なユーザー環境を考慮
- 極端な外れ値の影響を排除
- 実現可能な目標

### チーム別の目標設定例

**スタートアップ（リソース限定）:**
```yaml
目標: すべての指標で75パーセンタイル「Good」
期間: 6ヶ月
優先順位:
  1. LCP（最もビジネスインパクトが大きい）
  2. CLS（比較的改善しやすい）
  3. INP（時間がかかる可能性あり）
```

**中規模企業（専任チームあり）:**
```yaml
目標: すべての指標で90パーセンタイル「Good」
期間: 3ヶ月
優先順位:
  - すべての指標を並行して改善
  - RUMツールを導入して継続的に監視
```

**大企業（複数チーム）:**
```yaml
目標: すべての指標で95パーセンタイル「Good」
期間: 継続的改善
体制:
  - パフォーマンスチーム（専任）
  - 各プロダクトチームにパフォーマンスチャンピオン
  - 四半期ごとのパフォーマンスレビュー
```

## パフォーマンス文化の構築

### 1. パフォーマンスバジェットの設定

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
      },
    },
  },
}
```

### 2. プルリクエストでの自動チェック

```yaml
# .github/workflows/performance.yml
name: Performance Check
on: [pull_request]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build
        run: npm run build
      - name: Lighthouse CI
        uses: treosh/lighthouse-ci-action@v9
        with:
          runs: 3
          urls: |
            http://localhost:3000
          budgetPath: ./lighthouserc.json
```

### 3. ダッシュボードでの可視化

```typescript
// パフォーマンスダッシュボードの例
// Google Sheets APIを使用してCrUXデータを可視化

import { google } from 'googleapis'

async function updatePerformanceDashboard() {
  const auth = new google.auth.GoogleAuth({
    keyFile: 'credentials.json',
    scopes: ['https://www.googleapis.com/auth/spreadsheets'],
  })

  const sheets = google.sheets({ version: 'v4', auth })

  // CrUXデータを取得
  const cruxData = await fetchCrUXData()

  // Google Sheetsに書き込み
  await sheets.spreadsheets.values.update({
    spreadsheetId: 'your-sheet-id',
    range: 'Dashboard!A2:F100',
    valueInputOption: 'RAW',
    resource: {
      values: cruxData,
    },
  })
}
```

## まとめ

Core Web Vitalsは、単なる技術指標ではなく、ビジネスの成功に直結する重要な要素です。

**重要なポイント:**

1. **3つの指標をバランスよく改善する**
   - LCP: 2.5秒以下
   - INP: 200ms以下
   - CLS: 0.1以下

2. **ラボデータとフィールドデータの両方を活用**
   - 開発中: ラボデータで問題を早期発見
   - 本番環境: フィールドデータで実際の影響を測定

3. **継続的な測定と改善**
   - CI/CDに統合
   - RUMツールでリアルタイム監視
   - 定期的なレビューと改善

4. **チーム全体での取り組み**
   - パフォーマンスバジェットの設定
   - プルリクエストでの自動チェック
   - パフォーマンス文化の醸成

次の章では、LCPの詳細な最適化手法について学びます。
