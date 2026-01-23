---
title: "Virtual DOM最適化"
---

# Virtual DOM最適化

## この章で学ぶこと

Virtual DOMの仕組みを理解し、効率的な更新を実現する方法を学びます。

## Virtual DOMの仕組み

### Reconciliation（差分検出）

Reactは以下の手順でDOMを更新します:

1. 新しいVirtual DOMツリーを構築
2. 前回のツリーと比較（Diffing）
3. 変更箇所のみ実DOMに反映

### Diffingアルゴリズム

**O(n³) → O(n) への最適化**

React

は2つの仮定でDiffingを最適化:
1. 異なる型の要素は異なるツリーを生成
2. keyプロパティで要素を識別

## 最適化テクニック

### 1. key の正しい使用

```typescript
// 悪い例: インデックスをkeyに（並び替え時に問題）
{items.map((item, index) => (
  <Item key={index} {...item} />
))}

// 良い例: 安定したIDをkeyに
{items.map(item => (
  <Item key={item.id} {...item} />
))}
```

### 2. 要素の型を変えない

```typescript
// 悪い例: 型が変わるとツリー全体を再構築
{isLoading ? <div>Loading...</div> : <span>Content</span>}

// 良い例: 同じ型を維持
{isLoading ? <div>Loading...</div> : <div>Content</div>}
```

### 3. 条件付きレンダリングの最適化

```typescript
// 悪い例: 常に要素を生成
<div style={{ display: show ? 'block' : 'none' }}>
  <ExpensiveComponent />
</div>

// 良い例: 必要な時だけ生成
{show && <ExpensiveComponent />}
```

## shouldComponentUpdate の活用

クラスコンポーネントでの最適化。

```typescript
class MyComponent extends React.Component {
  shouldComponentUpdate(nextProps, nextState) {
    return nextProps.id !== this.props.id
  }

  render() {
    return <div>{this.props.data}</div>
  }
}
```

## React DevTools Profiler

パフォーマンス問題の特定。

```typescript
import { Profiler } from 'react'

function onRenderCallback(
  id, // プロファイラーのID
  phase, // "mount" または "update"
  actualDuration, // レンダリング時間
  baseDuration, // メモ化なしの推定時間
  startTime,
  commitTime
) {
  console.log(`${id} (${phase}) took ${actualDuration}ms`)
}

<Profiler id="Navigation" onRender={onRenderCallback}>
  <Navigation />
</Profiler>
```

## Batching（バッチ更新）

React 18のAutomatic Batching。

```typescript
// React 18: すべての更新が自動的にバッチ化
function handleClick() {
  setCount(c => c + 1)
  setFlag(f => !f)
  // 1回のレンダリングで両方更新
}

// setTimeout内でも自動バッチ化
setTimeout(() => {
  setCount(c => c + 1)
  setFlag(f => !f)
  // 1回のレンダリング
}, 1000)
```

## ベンチマーク指標

### 再レンダリングコスト

| コンポーネント数 | Virtual DOM Diff | DOM更新 | 合計 |
|----------------|-----------------|---------|------|
| 10 | 2ms | 3ms | 5ms |
| 100 | 15ms | 25ms | 40ms |
| 1000 | 120ms | 180ms | 300ms |
| 10000 | 1200ms | 2500ms | 3700ms |

## 改善事例

### 事例: ダッシュボード

**Before:**
- 500個のチャート要素
- 更新時のラグ: 800ms

**After:**

```typescript
// React.memo でメモ化
const Chart = React.memo(({ data }) => {
  return <canvas ref={ref} />
}, (prev, next) => {
  return prev.data.id === next.data.id
})

// 仮想化
const VirtualDashboard = () => {
  const virtualizer = useVirtualizer({
    count: charts.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 200,
  })

  return (
    <div ref={parentRef}>
      {virtualizer.getVirtualItems().map(item => (
        <Chart key={item.key} data={charts[item.index]} />
      ))}
    </div>
  )
}
```

**結果:**
- レンダリング時間: 800ms → 60ms (93%改善)
- スムーズなスクロール

## まとめ

Virtual DOM最適化のポイント:

1. **正しいkey** で要素を識別
2. **要素の型を保持** してツリーの再構築を避ける
3. **条件付きレンダリング** で不要な要素を生成しない
4. **React.memo** で再レンダリングを制御
5. **Profiler** でボトルネックを特定

次の章では、画像最適化について学びます。
