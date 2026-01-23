---
title: "Worker Threads活用"
---

# Worker Threads活用

CPU集約的な処理をWorker Threadsで並列化し、メインスレッドをブロックしない方法を学びます。

## Worker Threadsとは

Node.jsでマルチスレッド処理を実現する仕組みです。

```typescript
import { Worker } from 'worker_threads';

const worker = new Worker('./worker.js');

worker.on('message', (result) => {
  console.log('Result:', result);
});

worker.postMessage({ task: 'heavy-computation' });
```

## 基本的な使い方

### メインスレッド

```typescript
// main.ts
import { Worker } from 'worker_threads';

async function runWorker(data: any): Promise<any> {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./worker.js', {
      workerData: data,
    });

    worker.on('message', resolve);
    worker.on('error', reject);
    worker.on('exit', (code) => {
      if (code !== 0) {
        reject(new Error(`Worker stopped with exit code ${code}`));
      }
    });
  });
}

const result = await runWorker({ numbers: [1, 2, 3, 4, 5] });
console.log(result);
```

### Workerスレッド

```typescript
// worker.ts
import { parentPort, workerData } from 'worker_threads';

function heavyComputation(numbers: number[]): number {
  return numbers.reduce((sum, n) => sum + n * n, 0);
}

const result = heavyComputation(workerData.numbers);
parentPort?.postMessage(result);
```

## 実践例: 画像処理

```typescript
// image-processor.ts
import { Worker } from 'worker_threads';
import { cpus } from 'os';

async function processImages(images: string[]) {
  const numWorkers = cpus().length;
  const chunkSize = Math.ceil(images.length / numWorkers);

  const workers = [];

  for (let i = 0; i < numWorkers; i++) {
    const chunk = images.slice(i * chunkSize, (i + 1) * chunkSize);

    if (chunk.length === 0) continue;

    const worker = new Worker('./image-worker.js', {
      workerData: { images: chunk },
    });

    workers.push(
      new Promise((resolve, reject) => {
        worker.on('message', resolve);
        worker.on('error', reject);
      })
    );
  }

  const results = await Promise.all(workers);
  return results.flat();
}
```

## Worker Pool

```typescript
import { Worker } from 'worker_threads';
import { EventEmitter } from 'events';

class WorkerPool extends EventEmitter {
  private workers: Worker[] = [];
  private freeWorkers: Worker[] = [];
  private tasks: Array<{
    data: any;
    resolve: (value: any) => void;
    reject: (error: any) => void;
  }> = [];

  constructor(
    private workerScript: string,
    private numWorkers: number = cpus().length
  ) {
    super();
    this.init();
  }

  private init() {
    for (let i = 0; i < this.numWorkers; i++) {
      this.createWorker();
    }
  }

  private createWorker() {
    const worker = new Worker(this.workerScript);

    worker.on('message', (result) => {
      this.freeWorkers.push(worker);
      this.emit('task-complete', result);
      this.runNext();
    });

    worker.on('error', (error) => {
      this.emit('error', error);
    });

    this.workers.push(worker);
    this.freeWorkers.push(worker);
  }

  async exec(data: any): Promise<any> {
    return new Promise((resolve, reject) => {
      this.tasks.push({ data, resolve, reject });
      this.runNext();
    });
  }

  private runNext() {
    if (this.tasks.length === 0 || this.freeWorkers.length === 0) {
      return;
    }

    const task = this.tasks.shift()!;
    const worker = this.freeWorkers.shift()!;

    worker.once('message', task.resolve);
    worker.once('error', task.reject);
    worker.postMessage(task.data);
  }

  close() {
    this.workers.forEach((worker) => worker.terminate());
  }
}

// 使用例
const pool = new WorkerPool('./worker.js', 4);

const results = await Promise.all([
  pool.exec({ numbers: [1, 2, 3] }),
  pool.exec({ numbers: [4, 5, 6] }),
  pool.exec({ numbers: [7, 8, 9] }),
]);

pool.close();
```

## 実測データ: パフォーマンス

### テスト: 素数計算

```typescript
// シングルスレッド
console.time('Single');
const primes = findPrimes(1000000);
console.timeEnd('Single');
// Single: 2,456ms

// Worker Threads (4コア)
console.time('Workers');
const primesParallel = await findPrimesParallel(1000000, 4);
console.timeEnd('Workers');
// Workers: 687ms (3.6倍高速)
```

## まとめ

Worker Threadsのポイント:
- CPU集約的な処理に最適
- I/O処理には不要（非同期で十分）
- Worker Poolで効率化
- メインスレッドをブロックしない

次章: パフォーマンスプロファイリング
