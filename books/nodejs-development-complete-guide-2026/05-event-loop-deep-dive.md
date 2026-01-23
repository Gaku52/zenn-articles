---
title: "イベントループ完全理解"
---

# イベントループ完全理解

Node.jsの核心であるイベントループを完全に理解することで、高パフォーマンスなコードを書けるようになります。

## イベントループとは

Node.jsはシングルスレッドですが、非同期I/Oにより高い同時実行性を実現しています。その中心にあるのがイベントループです。

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout, setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  I/O callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  内部処理
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  I/O イベント取得
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  setImmediate
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──│      close callbacks      │  socket.on('close', ...)
   └───────────────────────────┘
```

## フェーズ詳解

### 1. timers フェーズ

`setTimeout()` と `setInterval()` のコールバックを実行します。

```typescript
console.log('Start');

setTimeout(() => {
  console.log('Timer 1');
}, 0);

setTimeout(() => {
  console.log('Timer 2');
}, 0);

console.log('End');

// 出力:
// Start
// End
// Timer 1
// Timer 2
```

### 2. poll フェーズ

I/Oイベントを待機し、そのコールバックを実行します。

```typescript
import fs from 'fs';

console.log('Start');

fs.readFile('/path/to/file', (err, data) => {
  console.log('File read complete');
});

console.log('End');

// 出力:
// Start
// End
// File read complete
```

### 3. check フェーズ

`setImmediate()` のコールバックを実行します。

```typescript
setImmediate(() => {
  console.log('Immediate');
});

setTimeout(() => {
  console.log('Timeout');
}, 0);

// 出力は実行環境により異なる場合がある
// Timeout
// Immediate
// または
// Immediate
// Timeout
```

## process.nextTick() と Promise

これらはイベントループのフェーズとは別に、**マイクロタスクキュー**で処理されます。

```typescript
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve().then(() => console.log('3'));

process.nextTick(() => console.log('4'));

console.log('5');

// 出力:
// 1
// 5
// 4  <- process.nextTick（最優先）
// 3  <- Promise（マイクロタスク）
// 2  <- setTimeout（タイマーフェーズ）
```

### 優先順位

```
1. 同期コード
2. process.nextTick()
3. Promise（マイクロタスク）
4. setImmediate()
5. setTimeout() / setInterval()
```

## よくある落とし穴

### 問題1: process.nextTick()の無限ループ

```typescript
// ❌ 悪い例: イベントループをブロック
function infiniteNextTick() {
  process.nextTick(infiniteNextTick);
}

infiniteNextTick();

// pollフェーズに到達できず、I/Oがブロックされる
```

```typescript
// ✅ 良い例: setImmediate() を使う
function infiniteImmediate() {
  setImmediate(infiniteImmediate);
}

infiniteImmediate();

// 各イベントループサイクルで他の処理も実行される
```

### 問題2: 長時間のCPU処理

```typescript
// ❌ 悪い例: 10秒間イベントループをブロック
function heavyComputation() {
  const start = Date.now();
  while (Date.now() - start < 10000) {
    // 重い処理
  }
  return 'Done';
}

app.get('/heavy', (req, res) => {
  const result = heavyComputation();
  res.send(result);
});

// この間、他のリクエストを処理できない
```

```typescript
// ✅ 良い例: Worker Threads を使う
import { Worker } from 'worker_threads';

app.get('/heavy', (req, res) => {
  const worker = new Worker('./heavy-worker.js');

  worker.on('message', (result) => {
    res.send(result);
  });

  worker.on('error', (error) => {
    res.status(500).send(error.message);
  });
});

// メインスレッドは他のリクエストを処理できる
```

## パフォーマンス計測

### イベントループレイテンシーの測定

```typescript
import { performance } from 'perf_hooks';

let lastCheck = performance.now();

setInterval(() => {
  const now = performance.now();
  const delay = now - lastCheck - 1000; // 期待値は1000ms

  if (delay > 100) {
    console.warn(`Event loop delay: ${delay.toFixed(2)}ms`);
  }

  lastCheck = now;
}, 1000);
```

### イベントループ診断

```typescript
import { monitorEventLoopDelay } from 'perf_hooks';

const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();

setInterval(() => {
  console.log({
    min: h.min,
    max: h.max,
    mean: h.mean,
    stddev: h.stddev,
    p50: h.percentile(50),
    p99: h.percentile(99),
  });
  h.reset();
}, 10000);
```

## ベストプラクティス

### 1. 長時間処理は分割する

```typescript
// ❌ 悪い例
function processLargeArray(items) {
  for (let i = 0; i < items.length; i++) {
    heavyOperation(items[i]);
  }
}

// ✅ 良い例: チャンク処理
async function processLargeArray(items, chunkSize = 100) {
  for (let i = 0; i < items.length; i += chunkSize) {
    const chunk = items.slice(i, i + chunkSize);

    for (const item of chunk) {
      heavyOperation(item);
    }

    // 他の処理を実行させる
    await new Promise((resolve) => setImmediate(resolve));
  }
}
```

### 2. I/O処理は並列化する

```typescript
// ❌ 悪い例: 直列処理（3秒）
async function fetchUsersSerial(ids) {
  const users = [];
  for (const id of ids) {
    const user = await fetchUser(id); // 各1秒
    users.push(user);
  }
  return users;
}

// ✅ 良い例: 並列処理（1秒）
async function fetchUsersParallel(ids) {
  return Promise.all(ids.map((id) => fetchUser(id)));
}
```

### 3. setImmediate vs setTimeout

```typescript
// I/O処理後に実行したい場合
fs.readFile('/path/to/file', (err, data) => {
  // ✅ 良い: check フェーズで実行
  setImmediate(() => {
    processData(data);
  });

  // ❌ 悪い: timers フェーズまで待つ
  setTimeout(() => {
    processData(data);
  }, 0);
});
```

## まとめ

イベントループの重要ポイント:
- Node.jsはシングルスレッドだが、非同期I/Oで高い同時実行性を実現
- イベントループは複数のフェーズで構成される
- `process.nextTick()` は最優先で実行される
- 長時間のCPU処理はイベントループをブロックする
- Worker Threadsやチャンク処理で対処する

次の章では、Promise と async/await の正しい使い方を学びます。
