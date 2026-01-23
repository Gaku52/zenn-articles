---
title: "実践事例 Part 2: SaaSダッシュボードの改善"
---

# 実践事例 Part 2: SaaSダッシュボードの改善

データ可視化が中心のSaaSダッシュボードを最適化した事例を紹介します。

## プロジェクト概要

**サービス:** ビジネス分析SaaS
**ユーザー数:** 15,000社
**課題:** ダッシュボードの動作が重い、ユーザー満足度低下

## 初期状態の分析

### パフォーマンス指標

| 指標 | 値 | 目標 |
|------|-----|------|
| LCP | 3.2秒 | 2.5秒以下 |
| INP | 580ms | 200ms以下 |
| CLS | 0.18 | 0.1以下 |
| TTI | 4.8秒 | 3.0秒以下 |

### ユーザーフィードバック

- チャート表示が遅い
- フィルタリング時にフリーズ
- データテーブルのスクロールがカクつく

## Phase 1: Chart.js最適化（Week 1-2）

### 実施内容

```typescript
// Before: 全機能読み込み
import Chart from 'chart.js/auto'  // 242KB

// After: 必要な機能のみ
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
} from 'chart.js'  // 58KB

ChartJS.register(
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend
)

// 遅延読み込み
const ChartComponent = dynamic(
  () => import('./ChartComponent'),
  {
    ssr: false,
    loading: () => <ChartSkeleton />,
  }
)
```

### 結果

- バンドルサイズ: 242KB → 58KB (76%削減)
- LCP: 3.2秒 → 2.1秒 (34%改善)

## Phase 2: 仮想スクロール導入（Week 3）

### 実施内容

```typescript
// Before: すべての行をレンダリング
function DataTable({ data }) {
  return (
    <table>
      {data.map(row => (
        <tr key={row.id}>
          <td>{row.name}</td>
          <td>{row.value}</td>
        </tr>
      ))}
    </table>
  )
}

// After: 仮想スクロール
import { useVirtualizer } from '@tanstack/react-virtual'

function VirtualDataTable({ data }) {
  const parentRef = useRef(null)

  const virtualizer = useVirtualizer({
    count: data.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
    overscan: 5,
  })

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map(virtualRow => {
          const row = data[virtualRow.index]
          return (
            <div
              key={virtualRow.key}
              style={{
                position: 'absolute',
                transform: `translateY(${virtualRow.start}px)`,
                height: `${virtualRow.size}px`,
              }}
            >
              <span>{row.name}</span>
              <span>{row.value}</span>
            </div>
          )
        })}
      </div>
    </div>
  )
}
```

### 結果

- 10,000行表示時のレンダリング時間: 2800ms → 120ms (96%改善)
- INP: 580ms → 280ms (52%改善)
- スクロールがスムーズに

## Phase 3: useMemo/useCallback最適化（Week 4）

### 実施内容

```typescript
function Dashboard({ data, filter }) {
  // フィルタリング結果をメモ化
  const filteredData = useMemo(() => {
    return data.filter(item => item.category === filter)
  }, [data, filter])

  // ソート処理をメモ化
  const sortedData = useMemo(() => {
    return [...filteredData].sort((a, b) => b.value - a.value)
  }, [filteredData])

  // イベントハンドラーをメモ化
  const handleRowClick = useCallback((id) => {
    router.push(`/detail/${id}`)
  }, [router])

  return (
    <VirtualDataTable
      data={sortedData}
      onRowClick={handleRowClick}
    />
  )
}

// コンポーネントもメモ化
const DataRow = React.memo(({ row, onClick }) => {
  return (
    <div onClick={() => onClick(row.id)}>
      {row.name}: {row.value}
    </div>
  )
}, (prev, next) => {
  return prev.row.id === next.row.id && prev.row.value === next.row.value
})
```

### 結果

- フィルタリング時のレンダリング: 850ms → 45ms (95%改善)
- INP: 280ms → 150ms (46%改善)

## Phase 4: Web Worker活用（Week 5）

### 実施内容

```typescript
// worker.ts
import { expose } from 'comlink'

const api = {
  async processData(data: any[]) {
    // 重い集計処理
    return data.reduce((acc, item) => {
      // 複雑な計算
      return acc + item.value
    }, 0)
  },

  async generateReport(data: any[]) {
    // レポート生成
    return {
      total: data.length,
      sum: data.reduce((a, b) => a + b.value, 0),
      average: data.reduce((a, b) => a + b.value, 0) / data.length,
    }
  },
}

expose(api)

// main.ts
import { wrap } from 'comlink'

function Dashboard() {
  const [report, setReport] = useState(null)
  const workerRef = useRef(null)

  useEffect(() => {
    const worker = new Worker(new URL('./worker.ts', import.meta.url))
    workerRef.current = wrap(worker)

    return () => worker.terminate()
  }, [])

  const handleGenerateReport = async () => {
    const result = await workerRef.current.generateReport(data)
    setReport(result)
  }

  return (
    <button onClick={handleGenerateReport}>
      Generate Report
    </button>
  )
}
```

### 結果

- レポート生成中もUIがフリーズしない
- INP: 150ms維持（重い処理中も）
- ユーザー体験が大幅に改善

## Phase 5: SSR → ISR移行（Week 6）

### 実施内容

```typescript
// Before: SSR（毎回サーバーで生成）
export async function getServerSideProps(context) {
  const data = await fetchDashboardData(context.params.userId)
  return { props: { data } }
}

// After: ISR（キャッシュ + 定期更新）
export async function getStaticProps({ params }) {
  const data = await fetchDashboardData(params.userId)

  return {
    props: { data },
    revalidate: 60, // 1分ごとに再生成
  }
}

export async function getStaticPaths() {
  return {
    paths: [],
    fallback: 'blocking', // 初回アクセス時に生成
  }
}

// クライアントで最新データを取得
function Dashboard({ initialData }) {
  const { data } = useSWR('/api/dashboard', fetcher, {
    fallbackData: initialData,
    refreshInterval: 30000, // 30秒ごと
  })

  return <DashboardView data={data} />
}
```

### 結果

- TTFB: 1200ms → 180ms (85%改善)
- サーバー負荷: 70% → 15%削減
- インフラコスト: 30%削減

## Phase 6: startTransition活用（Week 7）

### 実施内容

```typescript
import { startTransition, useState } from 'react'

function SearchDashboard() {
  const [query, setQuery] = useState('')
  const [results, setResults] = useState([])

  const handleSearch = (e) => {
    // 入力は即座に反映（高優先度）
    setQuery(e.target.value)

    // 検索結果の更新は低優先度
    startTransition(() => {
      const filtered = filterData(e.target.value)
      setResults(filtered)
    })
  }

  return (
    <>
      <input
        value={query}
        onChange={handleSearch}
        placeholder="Search..."
      />
      <ResultsList results={results} />
    </>
  )
}
```

### 結果

- 入力時のラグが完全に解消
- INP: 150ms → 85ms (43%改善)
- ユーザー満足度スコア: +45%

## 最終結果

### パフォーマンス指標

| 指標 | Before | After | 改善率 |
|------|--------|-------|--------|
| LCP | 3.2秒 | 1.1秒 | 66% |
| INP | 580ms | 85ms | 85% |
| CLS | 0.18 | 0.04 | 78% |
| TTI | 4.8秒 | 1.8秒 | 63% |
| TTFB | 1200ms | 180ms | 85% |

### ビジネス指標

| 指標 | Before | After | 改善 |
|------|--------|-------|------|
| ユーザー満足度 | 3.2/5 | 4.6/5 | +44% |
| 月間アクティブ率 | 62% | 81% | +19pt |
| タスク完了率 | 73% | 94% | +21pt |
| チャーン率 | 8.2% | 4.1% | -50% |

### コスト削減

- インフラコスト: 月額¥1.2M → ¥840K (30%削減)
- サポート問い合わせ: -42%
- 開発コスト: 約¥4M
- ROI: 5ヶ月で回収

## 重要な学び

### 1. パフォーマンスは機能

ユーザーにとってパフォーマンスは機能の一部。遅いアプリは機能していないのと同じ。

### 2. 測定が最重要

- 改善前後を必ず測定
- ビジネス指標との相関を追跡
- RUMで実ユーザーのデータを収集

### 3. 段階的改善

一度にすべて変更せず、段階的に改善して効果を確認。

### 4. 継続的最適化

```yaml
# 現在も実施中
- Lighthouse CI: PRごと
- Bundle size monitoring
- RUM: 24/7監視
- 月次パフォーマンスレビュー
- 四半期ごとの最適化スプリント
```

## まとめ

本書で学んだすべてのテクニックを実践することで、どんなWebアプリケーションでもパフォーマンスを劇的に改善できます。

**重要なポイント:**

1. **Core Web Vitalsを最優先**
   - LCP、INP、CLSすべて"Good"を目指す

2. **ユーザー体験を常に意識**
   - パフォーマンスはユーザー満足度に直結

3. **継続的な測定と改善**
   - 一度改善して終わりではない
   - CI/CDに組み込んで自動化

4. **チーム全体で取り組む**
   - パフォーマンスバジェット
   - 定期的なレビュー
   - 文化として根付かせる

フロントエンドパフォーマンス最適化は、ユーザーにもビジネスにも大きな価値をもたらします。本書で学んだ知識を活かして、より高速で快適なWebアプリケーションを構築してください！
