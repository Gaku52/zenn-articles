---
title: "JavaScript最適化"
---

# JavaScript最適化

JavaScriptはパース・コンパイル・実行に時間がかかります。効率的な最適化を学びます。

## バンドルサイズ削減

### lodashの最適化

```typescript
// Before: 72KB
import _ from 'lodash'

// After: 3KB
import uniq from 'lodash/uniq'
```

### moment.js → day.js

```typescript
// Before: 71KB
import moment from 'moment'

// After: 3KB
import dayjs from 'dayjs'
```

## コード分割

```typescript
// 動的インポート
const handleExport = async () => {
  const { exportPDF } = await import('./export')
  await exportPDF(data)
}
```

## Minification

```javascript
// terser設定
module.exports = {
  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            drop_console: true,
            dead_code: true,
          },
        },
      }),
    ],
  },
}
```

## WebAssembly活用

```typescript
// 計算集約的な処理をWasmで
const wasm = await WebAssembly.instantiateStreaming(
  fetch('compute.wasm')
)
const result = wasm.instance.exports.compute(data)
```

## 改善事例

**Before:** 全機能バンドル、1.2MB、TTI 4.8秒
**After:** コード分割 + 最適化、320KB、TTI 2.1秒（56%改善）
