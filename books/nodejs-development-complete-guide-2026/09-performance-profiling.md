---
title: "パフォーマンスプロファイリング"
---

# パフォーマンスプロファイリング

アプリケーションのボトルネックを特定し、最適化する方法を学びます。

## プロファイリングツール

### 1. Node.js内蔵プロファイラー

```bash
node --prof app.js
node --prof-process isolate-0x*.log > processed.txt
```

### 2. Chrome DevTools

```bash
node --inspect app.js
# Chrome で chrome://inspect を開く
```

### 3. clinic.js

```bash
npm install -g clinic
clinic doctor -- node app.js
clinic flame -- node app.js
clinic bubbleprof -- node app.js
```

## パフォーマンス計測

### performance API

```typescript
import { performance } from 'perf_hooks';

const start = performance.now();
await heavyOperation();
const end = performance.now();

console.log(`Duration: ${end - start}ms`);
```

### メモリ使用量

```typescript
console.log(process.memoryUsage());
// {
//   rss: 34603008,        // 総メモリ
//   heapTotal: 6537216,   // ヒープ合計
//   heapUsed: 4308496,    // ヒープ使用量
//   external: 1236952     // C++オブジェクト
// }
```

## ボトルネック特定

### 1. 関数の実行時間

```typescript
function measure<T>(fn: () => T, label: string): T {
  const start = performance.now();
  const result = fn();
  const end = performance.now();
  console.log(`${label}: ${(end - start).toFixed(2)}ms`);
  return result;
}

const users = measure(() => getUsers(), 'getUsers');
```

### 2. APIエンドポイントの計測

```typescript
import express from 'express';

app.use((req, res, next) => {
  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(`${req.method} ${req.url} - ${duration}ms`);
  });

  next();
});
```

## 最適化の実例

### Before

```typescript
// 遅いコード
async function getPostsWithAuthors() {
  const posts = await prisma.post.findMany();

  for (const post of posts) {
    post.author = await prisma.user.findUnique({
      where: { id: post.authorId },
    });
  }

  return posts;
}
// 想定例: 100件で約2,300ms（N+1問題）
```

### After

```typescript
// 高速なコード
async function getPostsWithAuthors() {
  return prisma.post.findMany({
    include: { author: true },
  });
}
// 想定例: 100件で約45ms（最適化後、理論上約51倍高速）
```

## まとめ

プロファイリングのポイント:
- まず計測、次に最適化
- ボトルネックを特定してから対処
- 最適化後は必ず再計測

次章: メモリ最適化
