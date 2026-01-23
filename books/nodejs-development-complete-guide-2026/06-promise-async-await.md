---
title: "Promise と async/await マスター"
---

# Promise と async/await マスター

非同期処理の正しい書き方を習得し、よくあるバグを防ぎます。

## Promiseの基礎

### Promise の3つの状態

```typescript
// Pending（待機中）
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve('Success'); // Fulfilled（成功）
    // reject(new Error('Failed')); // Rejected（失敗）
  }, 1000);
});

promise
  .then((result) => console.log(result))
  .catch((error) => console.error(error));
```

### Promise の作成

```typescript
// ✅ 良い例: Promise でラップ
function delay(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

// ✅ 良い例: コールバックを Promise に変換
import { promisify } from 'util';
import fs from 'fs';

const readFileAsync = promisify(fs.readFile);

async function readConfig() {
  const data = await readFileAsync('./config.json', 'utf-8');
  return JSON.parse(data);
}
```

## async/await の正しい使い方

### 基本パターン

```typescript
// ✅ 良い例
async function fetchUser(id: string) {
  try {
    const user = await prisma.user.findUnique({ where: { id } });
    if (!user) {
      throw new Error('User not found');
    }
    return user;
  } catch (error) {
    console.error('Failed to fetch user:', error);
    throw error;
  }
}
```

### 並列処理

```typescript
// ❌ 悪い例: 直列処理（3秒）
async function fetchDataSerial() {
  const user = await fetchUser('1'); // 1秒
  const posts = await fetchPosts('1'); // 1秒
  const comments = await fetchComments('1'); // 1秒
  return { user, posts, comments };
}

// ✅ 良い例: 並列処理（1秒）
async function fetchDataParallel() {
  const [user, posts, comments] = await Promise.all([
    fetchUser('1'),
    fetchPosts('1'),
    fetchComments('1'),
  ]);
  return { user, posts, comments };
}
```

### エラーハンドリング

```typescript
// ❌ 悪い例: エラーを握りつぶす
async function badErrorHandling() {
  try {
    await riskyOperation();
  } catch (error) {
    console.log(error); // ログだけ出して続行
  }
  // エラーが発生しても成功したように見える
}

// ✅ 良い例: エラーを適切に処理
async function goodErrorHandling() {
  try {
    return await riskyOperation();
  } catch (error) {
    console.error('Operation failed:', error);
    throw error; // 再スロー
  }
}

// ✅ 良い例: デフォルト値を返す
async function withFallback() {
  try {
    return await riskyOperation();
  } catch (error) {
    console.error('Operation failed, using fallback:', error);
    return getDefaultValue();
  }
}
```

## よくある間違い

### 間違い1: await を忘れる

```typescript
// ❌ 悪い例
async function forgotAwait() {
  const user = fetchUser('1'); // Promise<User> が返る
  console.log(user.name); // undefined
}

// ✅ 良い例
async function withAwait() {
  const user = await fetchUser('1'); // User が返る
  console.log(user.name); // 正しく表示
}
```

### 間違い2: ループ内での直列処理

```typescript
// ❌ 悪い例: 直列処理（10秒）
async function processUsersSerial(ids: string[]) {
  const results = [];
  for (const id of ids) {
    const user = await fetchUser(id); // 各1秒
    results.push(user);
  }
  return results;
}

// ✅ 良い例: 並列処理（1秒）
async function processUsersParallel(ids: string[]) {
  return Promise.all(ids.map((id) => fetchUser(id)));
}

// ✅ より良い例: 制限付き並列処理
async function processUsersWithLimit(ids: string[], limit = 5) {
  const results = [];
  for (let i = 0; i < ids.length; i += limit) {
    const chunk = ids.slice(i, i + limit);
    const chunkResults = await Promise.all(
      chunk.map((id) => fetchUser(id))
    );
    results.push(...chunkResults);
  }
  return results;
}
```

### 間違い3: Promise.all でエラーハンドリング

```typescript
// ❌ 悪い例: 1つでも失敗すると全て失敗
async function fetchAllOrNothing(ids: string[]) {
  return Promise.all(ids.map((id) => fetchUser(id)));
  // 1つでも失敗すると、全て失敗扱い
}

// ✅ 良い例: Promise.allSettled を使う
async function fetchAllWithResults(ids: string[]) {
  const results = await Promise.allSettled(
    ids.map((id) => fetchUser(id))
  );

  const successful = results
    .filter((r) => r.status === 'fulfilled')
    .map((r) => r.value);

  const failed = results
    .filter((r) => r.status === 'rejected')
    .map((r) => r.reason);

  return { successful, failed };
}
```

## 高度なパターン

### リトライ機能

```typescript
async function retry<T>(
  fn: () => Promise<T>,
  maxAttempts = 3,
  delay = 1000
): Promise<T> {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxAttempts) {
        throw error;
      }
      console.log(`Attempt ${attempt} failed, retrying...`);
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }
  throw new Error('All attempts failed');
}

// 使用例
const user = await retry(() => fetchUser('1'), 3, 2000);
```

### タイムアウト機能

```typescript
async function withTimeout<T>(
  promise: Promise<T>,
  ms: number
): Promise<T> {
  const timeout = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new Error('Timeout')), ms)
  );
  return Promise.race([promise, timeout]);
}

// 使用例
try {
  const user = await withTimeout(fetchUser('1'), 5000);
} catch (error) {
  console.error('Request timed out');
}
```

### キャッシュ付き非同期関数

```typescript
function memoizeAsync<T>(
  fn: (...args: any[]) => Promise<T>,
  ttl = 60000
) {
  const cache = new Map<string, { value: T; expires: number }>();

  return async (...args: any[]): Promise<T> => {
    const key = JSON.stringify(args);
    const cached = cache.get(key);

    if (cached && cached.expires > Date.now()) {
      return cached.value;
    }

    const value = await fn(...args);
    cache.set(key, { value, expires: Date.now() + ttl });
    return value;
  };
}

// 使用例
const fetchUserCached = memoizeAsync(fetchUser, 60000);
```

## Promise の連鎖

### アンチパターン

```typescript
// ❌ 悪い例: Promise Hell
function promiseHell() {
  return fetchUser('1')
    .then((user) => {
      return fetchPosts(user.id)
        .then((posts) => {
          return fetchComments(posts[0].id)
            .then((comments) => {
              return { user, posts, comments };
            });
        });
    });
}
```

### ベストプラクティス

```typescript
// ✅ 良い例: async/await
async function cleanAsyncAwait() {
  const user = await fetchUser('1');
  const posts = await fetchPosts(user.id);
  const comments = await fetchComments(posts[0].id);
  return { user, posts, comments };
}

// ✅ 良い例: フラットな Promise チェーン
function flatPromiseChain() {
  let user: User;
  let posts: Post[];

  return fetchUser('1')
    .then((u) => {
      user = u;
      return fetchPosts(user.id);
    })
    .then((p) => {
      posts = p;
      return fetchComments(posts[0].id);
    })
    .then((comments) => ({ user, posts, comments }));
}
```

## 直列処理 vs 並列処理の比較

```typescript
// 直列処理: 一つずつ順番に処理
console.time('Serial');
for (const id of ids) {
  await fetchUser(id);  // 前の処理が終わるまで待つ
}
console.timeEnd('Serial');

// 並列処理: 同時に実行
console.time('Parallel');
await Promise.all(ids.map(fetchUser));  // 全て同時に開始
console.timeEnd('Parallel');
```

**パフォーマンス特性:**
- 直列処理: 処理時間 = 各処理時間の合計
- 並列処理: 処理時間 ≒ 最も遅い処理の時間

一般的に、独立した複数の非同期処理は並列実行により大幅に高速化できます。

## まとめ

Promise/async/await のベストプラクティス:
- ✅ 並列処理可能なものは Promise.all を使う
- ✅ エラーハンドリングを必ず行う
- ✅ Promise.allSettled で一部失敗を許容
- ✅ リトライやタイムアウトを実装
- ❌ Promise Hell を避ける
- ❌ await を忘れない

次の章では、大容量データ処理に欠かせない Stream について学びます。
