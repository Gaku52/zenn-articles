---
title: "Node.js初心者が必ず躓く非同期処理の罠5選と解決策"
emoji: "🪤"
type: "tech"
topics: ["nodejs", "javascript", "async", "promise", "typescript"]
published: false
---

## はじめに

Node.jsの非同期処理は強力ですが、その仕組みを正しく理解していないと思わぬバグや パフォーマンス問題を引き起こします。

この記事では、Node.js開発においてよく見られる非同期処理の典型的な落とし穴と、それを回避するためのベストプラクティスを紹介します。

## 1. Promise地獄 - async/awaitの誤った使い方

### よくある間違い

Callback地獄から脱却するためにPromiseやasync/awaitを導入したのに、結局ネストが深くなってしまうケースです。

```typescript
// ❌ Promise地獄
async function processUserData(userId: string) {
  return getUser(userId).then(user => {
    return getUserPosts(user.id).then(posts => {
      return getPostComments(posts[0].id).then(comments => {
        return {
          user,
          posts,
          comments
        };
      });
    });
  });
}
```

### 解決策: フラットなasync/await

```typescript
// ✅ フラットで読みやすい
async function processUserData(userId: string) {
  const user = await getUser(userId);
  const posts = await getUserPosts(user.id);
  const comments = await getPostComments(posts[0].id);

  return { user, posts, comments };
}
```

**学習ポイント:**
- `.then()`のネストは避ける
- async/awaitで直列処理を明示的に記述
- コードの可読性とメンテナンス性が向上

## 2. 並列処理と直列処理の混同

### よくある間違い

複数の非同期処理を実行する際、不必要に直列化してしまい、パフォーマンスを損なうケースです。

```typescript
// ❌ 不必要な直列処理（想定実行時間: 3秒）
async function fetchAllData() {
  const users = await fetchUsers();      // 1秒
  const products = await fetchProducts(); // 1秒
  const orders = await fetchOrders();     // 1秒

  return { users, products, orders };
}
```

### 解決策: Promise.allで並列化

```typescript
// ✅ 並列処理（想定実行時間: 1秒）
async function fetchAllData() {
  const [users, products, orders] = await Promise.all([
    fetchUsers(),
    fetchProducts(),
    fetchOrders()
  ]);

  return { users, products, orders };
}

// ✅ 依存関係がある場合は段階的に並列化
async function fetchUserRelatedData(userId: string) {
  // 最初にユーザー情報を取得（必須）
  const user = await getUser(userId);

  // ユーザーIDを使った処理は並列化
  const [posts, followers, settings] = await Promise.all([
    getUserPosts(user.id),
    getUserFollowers(user.id),
    getUserSettings(user.id)
  ]);

  return { user, posts, followers, settings };
}
```

**学習ポイント:**
- 独立した処理は`Promise.all`で並列化
- 依存関係がある処理は直列化
- 理論的には最大3倍のパフォーマンス向上が見込める

## 3. エラーハンドリングの抜け - 未処理のPromise拒否

### よくある間違い

async/awaitでエラーハンドリングを忘れると、アプリケーションがクラッシュする可能性があります。

```typescript
// ❌ エラーハンドリングなし
async function updateUserProfile(userId: string, data: ProfileData) {
  const user = await getUser(userId); // ここで例外が発生する可能性
  await updateDatabase(user.id, data);
  return user;
}

// ❌ Promise.allでの部分的な失敗を考慮していない
async function syncAllData() {
  await Promise.all([
    syncUsers(),    // これが失敗すると全体が失敗
    syncProducts(),
    syncOrders()
  ]);
}
```

### 解決策: 適切なtry-catchとPromise.allSettled

```typescript
// ✅ try-catchで適切にエラーハンドリング
async function updateUserProfile(userId: string, data: ProfileData) {
  try {
    const user = await getUser(userId);
    await updateDatabase(user.id, data);
    return { success: true, user };
  } catch (error) {
    console.error('Failed to update profile:', error);
    return { success: false, error: error.message };
  }
}

// ✅ Promise.allSettledで部分的な失敗を許容
async function syncAllData() {
  const results = await Promise.allSettled([
    syncUsers(),
    syncProducts(),
    syncOrders()
  ]);

  const failures = results.filter(r => r.status === 'rejected');

  if (failures.length > 0) {
    console.warn(`${failures.length} sync operations failed`);
  }

  return {
    total: results.length,
    succeeded: results.filter(r => r.status === 'fulfilled').length,
    failed: failures.length
  };
}
```

**学習ポイント:**
- async関数は必ずtry-catchで囲む
- Promise.all: 1つでも失敗したら全体が失敗
- Promise.allSettled: 全ての結果を取得（成功/失敗問わず）

## 4. async/awaitの暗黙的なPromise変換の誤解

### よくある間違い

async関数は常にPromiseを返すことを理解せず、想定外の挙動に戸惑うケースです。

```typescript
// ❌ 同期的な値を返しているつもりが、Promiseになる
async function getUserName(userId: string): string { // 型定義が間違っている
  const user = await getUser(userId);
  return user.name; // Promiseでラップされる
}

// この関数を使う側
const name = getUserName('123'); // nameはPromise<string>
console.log(name.toUpperCase()); // エラー！nameはPromiseです
```

### 解決策: 正しい型定義と使用方法

```typescript
// ✅ 正しい型定義
async function getUserName(userId: string): Promise<string> {
  const user = await getUser(userId);
  return user.name;
}

// ✅ 呼び出し側もawaitする
async function displayUserName(userId: string) {
  const name = await getUserName(userId);
  console.log(name.toUpperCase()); // 正しく動作
}

// ✅ 本当に同期的な値を返したい場合はasyncを使わない
function formatUserName(name: string): string {
  return name.toUpperCase();
}
```

**学習ポイント:**
- async関数は必ずPromiseを返す
- 戻り値の型は`Promise<T>`と明示
- TypeScriptの型チェックを活用

## 5. Event Loopのブロッキング - 重い処理の誤った配置

### よくある間違い

async/awaitを使っているから非同期だと勘違いし、CPU集約的な処理をメインスレッドで実行してしまうケースです。

```typescript
// ❌ async/awaitでもCPU集約的な処理はブロッキングする
async function processLargeData(data: number[]) {
  // この処理は非同期ではない！イベントループをブロックする
  const result = data.map(n => {
    // 重い計算処理
    for (let i = 0; i < 1000000; i++) {
      n = Math.sqrt(n * n + i);
    }
    return n;
  });

  return result;
}

// 他のリクエストが全て待たされる！
app.get('/process', async (req, res) => {
  const result = await processLargeData(req.body.data);
  res.json({ result });
});
```

### 解決策: Worker ThreadsやWeb Workersの活用

```typescript
import { Worker } from 'worker_threads';

// ✅ Worker Threadsで重い処理を分離
function processLargeDataAsync(data: number[]): Promise<number[]> {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./heavy-calculation-worker.js', {
      workerData: data
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

// メインスレッドをブロックしない
app.get('/process', async (req, res) => {
  const result = await processLargeDataAsync(req.body.data);
  res.json({ result });
});
```

**heavy-calculation-worker.js**
```javascript
const { parentPort, workerData } = require('worker_threads');

// 重い計算処理
const result = workerData.map(n => {
  for (let i = 0; i < 1000000; i++) {
    n = Math.sqrt(n * n + i);
  }
  return n;
});

parentPort.postMessage(result);
```

**学習ポイント:**
- async/awaitは「非同期I/O」を扱いやすくするもの
- CPU集約的な処理は依然としてブロッキングする
- Worker Threadsで別スレッドに処理を委譲

## まとめ

Node.jsの非同期処理における典型的な落とし穴を5つ紹介しました。

| 落とし穴 | 影響 | 解決策 |
|---------|------|--------|
| Promise地獄 | 可読性低下 | async/awaitをフラットに |
| 並列/直列の混同 | パフォーマンス低下 | Promise.all活用 |
| エラーハンドリング不足 | クラッシュリスク | try-catch、allSettled |
| 暗黙的Promise変換 | 型エラー | 正しい型定義 |
| Event Loopブロッキング | 全体の遅延 | Worker Threads |

これらのパターンを理解することで、より堅牢で高性能なNode.jsアプリケーションを構築できます。

### より深く学びたい方へ

この記事では非同期処理の基本的な落とし穴を紹介しましたが、実務ではさらに以下のような内容も重要です:

- Event Loopの内部動作の詳細
- Promise、async/awaitの実装原理
- ストリーム処理とバックプレッシャー
- 並行処理の限界とスケーリング戦略

これらの内容を体系的に学びたい方は、拙著「[Node.js開発完全ガイド 2026](https://zenn.dev/gaku52/books/nodejs-development-complete-guide-2026)」をご参照ください。非同期処理の理論から実践まで、20万文字超のボリュームで詳しく解説しています。

---

**関連記事**
- [Node.jsパフォーマンス最適化の実践 - レスポンス時間を最大67%改善する5つの手法](https://zenn.dev/gaku52/articles/nodejs-performance-optimization-5-methods)
- [本番環境で泣かないためのNode.jsエラーハンドリング実践ガイド](https://zenn.dev/gaku52/articles/nodejs-error-handling-production-guide)
