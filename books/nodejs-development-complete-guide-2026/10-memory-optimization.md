---
title: "メモリ最適化 - リークを防ぎ、効率的なアプリを作る"
---

# メモリ最適化 - リークを防ぎ、効率的なアプリを作る

Node.jsアプリケーションのメモリ使用量を最適化し、メモリリークを防ぐ実践的な方法を学びます。

## V8のメモリ構造を理解する

### メモリの種類

```
┌─────────────────────────────────┐
│   Resident Set Size (RSS)       │  ← プロセス全体のメモリ
├─────────────────────────────────┤
│  ┌─────────────────────────────┐│
│  │    Heap Memory (動的)       ││
│  │  ├─ New Space (若い世代)    ││  ← 新しいオブジェクト
│  │  ├─ Old Space (古い世代)    ││  ← 生き残ったオブジェクト
│  │  └─ Large Object Space      ││  ← 大きなオブジェクト
│  └─────────────────────────────┘│
│  ┌─────────────────────────────┐│
│  │    Stack (静的)             ││  ← 関数呼び出し、ローカル変数
│  └─────────────────────────────┘│
│  ┌─────────────────────────────┐│
│  │    Code Space               ││  ← コンパイル済みコード
│  └─────────────────────────────┘│
└─────────────────────────────────┘
```

### メモリ使用量の確認

```typescript
function formatMemoryUsage(bytes: number): string {
  return `${(bytes / 1024 / 1024).toFixed(2)} MB`;
}

function logMemoryUsage() {
  const usage = process.memoryUsage();

  console.log('Memory Usage:');
  console.log(`  RSS:        ${formatMemoryUsage(usage.rss)}`);
  console.log(`  Heap Total: ${formatMemoryUsage(usage.heapTotal)}`);
  console.log(`  Heap Used:  ${formatMemoryUsage(usage.heapUsed)}`);
  console.log(`  External:   ${formatMemoryUsage(usage.external)}`);
  console.log(`  Array Buffers: ${formatMemoryUsage(usage.arrayBuffers)}`);
}

// 定期的に監視
setInterval(logMemoryUsage, 10000);
```

**出力される情報:**
- **RSS**: プロセス全体のメモリ使用量
- **Heap Total**: 確保されたヒープメモリ総量
- **Heap Used**: 実際に使用中のヒープメモリ
- **External**: C++オブジェクトが使用するメモリ
- **Array Buffers**: ArrayBufferが使用するメモリ

## メモリリークの主な原因

### 1. グローバル変数の肥大化

```typescript
// ❌ 悪い例: グローバルキャッシュが無限に増える
const userCache = new Map<string, User>();

app.get('/api/users/:id', async (req, res) => {
  const { id } = req.params;

  if (!userCache.has(id)) {
    const user = await prisma.user.findUnique({ where: { id } });
    userCache.set(id, user); // 永遠に削除されない
  }

  res.json(userCache.get(id));
});

// 問題: アクセスされたユーザーが全て永続的にメモリに残る
```

```typescript
// ✅ 良い例: LRUキャッシュで自動削除
import LRU from 'lru-cache';

const userCache = new LRU<string, User>({
  max: 1000, // 最大1000エントリ
  ttl: 1000 * 60 * 10, // 10分でexpire
  updateAgeOnGet: true, // アクセス時にTTLリセット
});

app.get('/api/users/:id', async (req, res) => {
  const { id } = req.params;

  let user = userCache.get(id);
  if (!user) {
    user = await prisma.user.findUnique({ where: { id } });
    userCache.set(id, user);
  }

  res.json(user);
});

// メリット: 最大1000エントリに制限され、古いデータは自動削除される
```

### 2. イベントリスナーの解放忘れ

```typescript
// ❌ 悪い例: リスナーが蓄積していく
class DataProcessor {
  constructor(private eventEmitter: EventEmitter) {
    // 毎回新しいリスナーが追加される
    this.eventEmitter.on('data', (data) => {
      this.process(data);
    });
  }

  process(data: any) {
    // データ処理
  }
}

// 1000回インスタンス化すると1000個のリスナーが残る
for (let i = 0; i < 1000; i++) {
  new DataProcessor(emitter);
}

console.log(emitter.listenerCount('data')); // 1000
```

```typescript
// ✅ 良い例: 明示的に解放
class DataProcessor {
  private handler: (data: any) => void;

  constructor(private eventEmitter: EventEmitter) {
    this.handler = this.process.bind(this);
    this.eventEmitter.on('data', this.handler);
  }

  process(data: any) {
    // データ処理
  }

  dispose() {
    this.eventEmitter.removeListener('data', this.handler);
  }
}

// 使用後に解放
const processors: DataProcessor[] = [];
for (let i = 0; i < 1000; i++) {
  processors.push(new DataProcessor(emitter));
}

// クリーンアップ
processors.forEach(p => p.dispose());
console.log(emitter.listenerCount('data')); // 0
```

### 3. クロージャによるメモリ保持

```typescript
// ❌ 悪い例: 大きなオブジェクトを保持し続ける
function createHandlers() {
  const largeData = new Array(1000000).fill({
    id: crypto.randomUUID(),
    data: 'x'.repeat(1000),
  });

  return {
    getCount: () => largeData.length, // largeData全体を保持
    getFirst: () => largeData[0],
  };
}

// 問題: クロージャがlargeData全体への参照を保持し続ける
```

```typescript
// ✅ 良い例: 必要なデータのみ保持
function createHandlers() {
  const largeData = new Array(1000000).fill({
    id: crypto.randomUUID(),
    data: 'x'.repeat(1000),
  });

  const count = largeData.length;
  const first = largeData[0];

  // largeDataはGCされる
  return {
    getCount: () => count,
    getFirst: () => first,
  };
}

// メリット: largeDataへの参照がなくなり、GCで回収される
```

### 4. タイマーのクリア忘れ

```typescript
// ❌ 悪い例: タイマーが蓄積
class PollingService {
  private timerId?: NodeJS.Timeout;

  start() {
    this.timerId = setInterval(() => {
      this.poll();
    }, 5000);
  }

  poll() {
    // ポーリング処理
  }
}

// 100回インスタンス化すると100個のタイマーが動作
const services: PollingService[] = [];
for (let i = 0; i < 100; i++) {
  const service = new PollingService();
  service.start();
  services.push(service);
}
```

```typescript
// ✅ 良い例: タイマーをクリア
class PollingService {
  private timerId?: NodeJS.Timeout;

  start() {
    if (this.timerId) {
      clearInterval(this.timerId);
    }
    this.timerId = setInterval(() => {
      this.poll();
    }, 5000);
  }

  poll() {
    // ポーリング処理
  }

  stop() {
    if (this.timerId) {
      clearInterval(this.timerId);
      this.timerId = undefined;
    }
  }
}

// 使用後に停止
services.forEach(s => s.stop());
```

## メモリリークの検出

### 1. Heapdumpで分析

```typescript
import v8 from 'v8';
import fs from 'fs';
import path from 'path';

function takeHeapSnapshot() {
  const filename = path.join(
    __dirname,
    `heap-${Date.now()}.heapsnapshot`
  );

  v8.writeHeapSnapshot(filename);
  console.log(`Heap snapshot saved to ${filename}`);

  const stats = fs.statSync(filename);
  console.log(`File size: ${(stats.size / 1024 / 1024).toFixed(2)} MB`);
}

// APIエンドポイントで手動取得
app.post('/admin/heap-snapshot', (req, res) => {
  takeHeapSnapshot();
  res.json({ message: 'Snapshot taken' });
});

// 定期的に自動取得（開発環境のみ）
if (process.env.NODE_ENV === 'development') {
  setInterval(takeHeapSnapshot, 60000); // 1分ごと
}
```

**Chrome DevToolsでの分析手順:**
1. Chrome DevToolsを開く
2. Memory タブ → Load
3. heapsnapshotファイルを選択
4. Comparison viewで差分を確認
5. Detached DOM nodesやRetained Sizeが大きいオブジェクトを調査

### 2. memwatch-nextで自動検出

```typescript
import memwatch from '@airbnb/node-memwatch';

// メモリリーク検出
memwatch.on('leak', (info) => {
  console.error('⚠️  Memory leak detected!');
  console.error(JSON.stringify(info, null, 2));

  // アラート送信
  sendAlert({
    type: 'memory-leak',
    growth: info.growth,
    reason: info.reason,
  });
});

// GC統計
memwatch.on('stats', (stats) => {
  const trend = stats.usage_trend > 0 ? '📈' : '📉';
  console.log(`${trend} Heap Usage Trend: ${stats.usage_trend}%`);
  console.log(`Current: ${(stats.current_base / 1024 / 1024).toFixed(2)} MB`);
  console.log(`Estimated: ${(stats.estimated_base / 1024 / 1024).toFixed(2)} MB`);
});
```

### 3. 継続的モニタリング

```typescript
import { performance, PerformanceObserver } from 'perf_hooks';

class MemoryMonitor {
  private measurements: Array<{
    timestamp: number;
    heapUsed: number;
    heapTotal: number;
    external: number;
  }> = [];

  start() {
    setInterval(() => {
      const usage = process.memoryUsage();

      this.measurements.push({
        timestamp: Date.now(),
        heapUsed: usage.heapUsed,
        heapTotal: usage.heapTotal,
        external: usage.external,
      });

      // 直近100件のみ保持
      if (this.measurements.length > 100) {
        this.measurements.shift();
      }

      this.checkForLeak();
    }, 10000); // 10秒ごと
  }

  private checkForLeak() {
    if (this.measurements.length < 10) return;

    const recent = this.measurements.slice(-10);
    const growth = recent.map((m, i) => {
      if (i === 0) return 0;
      return m.heapUsed - recent[i - 1].heapUsed;
    });

    const avgGrowth = growth.reduce((a, b) => a + b, 0) / growth.length;

    // 平均して1MB/10秒以上増加している場合
    if (avgGrowth > 1024 * 1024) {
      console.warn('⚠️  Potential memory leak detected!');
      console.warn(`Average growth: ${(avgGrowth / 1024 / 1024).toFixed(2)} MB/10s`);
    }
  }
}

const monitor = new MemoryMonitor();
monitor.start();
```

## メモリ最適化テクニック

### 1. オブジェクトプーリング

```typescript
class ObjectPool<T> {
  private available: T[] = [];
  private inUse = new Set<T>();

  constructor(
    private create: () => T,
    private reset: (obj: T) => void,
    initialSize = 10
  ) {
    for (let i = 0; i < initialSize; i++) {
      this.available.push(this.create());
    }
  }

  acquire(): T {
    let obj: T;

    if (this.available.length > 0) {
      obj = this.available.pop()!;
    } else {
      obj = this.create();
    }

    this.inUse.add(obj);
    return obj;
  }

  release(obj: T) {
    if (!this.inUse.has(obj)) {
      throw new Error('Object not in use');
    }

    this.inUse.delete(obj);
    this.reset(obj);
    this.available.push(obj);
  }

  get stats() {
    return {
      available: this.available.length,
      inUse: this.inUse.size,
      total: this.available.length + this.inUse.size,
    };
  }
}

// Bufferプールの例
const bufferPool = new ObjectPool(
  () => Buffer.allocUnsafe(64 * 1024), // 64KB
  (buffer) => buffer.fill(0),
  100
);

async function processFile(filePath: string) {
  const buffer = bufferPool.acquire();

  try {
    const fd = await fs.promises.open(filePath, 'r');
    await fd.read(buffer, 0, buffer.length, 0);
    await fd.close();

    // bufferを使った処理
    return processBuffer(buffer);
  } finally {
    bufferPool.release(buffer);
  }
}

console.log(bufferPool.stats);
// { available: 98, inUse: 2, total: 100 }
```

### 2. WeakMapで弱参照

```typescript
// ❌ 悪い例: Mapは参照を保持し続ける
const metadata = new Map<object, any>();

class User {
  constructor(public id: string, public name: string) {
    metadata.set(this, {
      created: Date.now(),
      accessCount: 0,
    });
  }
}

let user = new User('1', 'John');
user = null; // Userオブジェクトは解放されない（Mapが保持）
```

```typescript
// ✅ 良い例: WeakMapは弱参照
const metadata = new WeakMap<object, any>();

class User {
  constructor(public id: string, public name: string) {
    metadata.set(this, {
      created: Date.now(),
      accessCount: 0,
    });
  }
}

let user = new User('1', 'John');
user = null; // Userオブジェクトは自動的にGCされる
```

### 3. Streamで大容量データ処理

```typescript
// ❌ 悪い例: 全てメモリにロード
async function processLargeJSON(filePath: string) {
  const data = await fs.promises.readFile(filePath, 'utf-8');
  const parsed = JSON.parse(data); // 1GB のファイル = 1GB メモリ消費

  return parsed.map(item => transform(item));
}
```

```typescript
// ✅ 良い例: Streamで処理
import { pipeline } from 'stream/promises';
import { createReadStream, createWriteStream } from 'fs';
import { Transform } from 'stream';
import JSONStream from 'JSONStream';

async function processLargeJSON(inputPath: string, outputPath: string) {
  await pipeline(
    createReadStream(inputPath),
    JSONStream.parse('*'), // JSON配列をパース
    new Transform({
      objectMode: true,
      transform(item, encoding, callback) {
        const transformed = transform(item);
        callback(null, JSON.stringify(transformed) + '\n');
      },
    }),
    createWriteStream(outputPath)
  );
}

// メリット: データをチャンク単位で処理するため、ファイルサイズに関わらずメモリ使用量が一定
```

## ガベージコレクションの最適化

### Node.jsのGCフラグ

```bash
# Old Spaceサイズを4GBに増やす（デフォルト: 512MB）
node --max-old-space-size=4096 app.js

# New Spaceサイズを調整
node --max-semi-space-size=64 app.js

# GCログを出力
node --trace-gc app.js

# GCの詳細情報
node --trace-gc-verbose --trace-gc-nvp app.js
```

### GC統計の監視

```typescript
import { PerformanceObserver } from 'perf_hooks';

const gcObserver = new PerformanceObserver((list) => {
  const entries = list.getEntries();

  entries.forEach((entry) => {
    console.log(`GC: ${entry.name}`);
    console.log(`  Duration: ${entry.duration.toFixed(2)}ms`);
    console.log(`  Kind: ${entry.detail?.kind}`);
  });
});

gcObserver.observe({ entryTypes: ['gc'], buffered: true });
```

## まとめ

メモリ最適化のベストプラクティス:
- ✅ LRUキャッシュで自動削除
- ✅ イベントリスナーを必ず解放
- ✅ クロージャで必要なデータのみ保持
- ✅ オブジェクトプーリングで再利用
- ✅ WeakMapで弱参照を活用
- ✅ Streamで大容量データを処理
- ✅ Heapdumpで定期的に分析
- ❌ グローバル変数の肥大化を避ける

次の章では、Node.jsアプリケーションのクラスタリングとスケーリング戦略を学びます。
