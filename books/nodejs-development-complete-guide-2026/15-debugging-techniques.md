---
title: "デバッグテクニック - 効率的な問題解決"
---

# デバッグテクニック - 効率的な問題解決

Node.jsアプリケーションのデバッグを効率的に行うための実践的なテクニックとツールを学びます。

## Chrome DevTools によるデバッグ

### 基本的な起動方法

```bash
# --inspect フラグで起動
node --inspect server.js

# 特定のポートを指定
node --inspect=9229 server.js

# 起動直後にブレークポイントで停止
node --inspect-brk server.js
```

Chrome で `chrome://inspect` を開いてデバッグ開始。

### VS Code での統合デバッグ

`.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Launch Program",
      "skipFiles": ["<node_internals>/**"],
      "program": "${workspaceFolder}/src/server.ts",
      "preLaunchTask": "tsc: build - tsconfig.json",
      "outFiles": ["${workspaceFolder}/dist/**/*.js"],
      "env": {
        "NODE_ENV": "development"
      }
    },
    {
      "type": "node",
      "request": "attach",
      "name": "Attach to Process",
      "port": 9229,
      "skipFiles": ["<node_internals>/**"]
    },
    {
      "type": "node",
      "request": "launch",
      "name": "Jest Tests",
      "program": "${workspaceFolder}/node_modules/.bin/jest",
      "args": ["--runInBand", "--no-cache"],
      "console": "integratedTerminal"
    }
  ]
}
```

### ブレークポイントの活用

```typescript
// コード内ブレークポイント
function processUser(user: User) {
  debugger; // デバッガがここで停止

  const validated = validateUser(user);
  const transformed = transformUser(validated);

  return transformed;
}

// 条件付きブレークポイント
function processItems(items: Item[]) {
  for (const item of items) {
    // VS Code: ブレークポイント右クリック > "Edit Breakpoint"
    // 条件: item.id === '123'
    processItem(item);
  }
}

// ログポイント（実行を止めずにログ出力）
// VS Code: ブレークポイント右クリック > "Add Logpoint"
// Message: "Processing item {item.id}"
```

## デバッグ用ログの戦略的配置

### デバッグログの追加

```typescript
import debug from 'debug';

// モジュールごとにデバッガを作成
const dbDebug = debug('app:db');
const authDebug = debug('app:auth');
const apiDebug = debug('app:api');

// 使用例
async function findUser(id: string) {
  dbDebug('Finding user with id: %s', id);

  const user = await prisma.user.findUnique({
    where: { id },
  });

  dbDebug('Found user: %O', user);

  return user;
}

// 実行時に有効化
// DEBUG=app:* node server.js
// DEBUG=app:db,app:auth node server.js
```

### コンテキスト付きログ

```typescript
class Logger {
  constructor(private context: Record<string, any> = {}) {}

  withContext(additionalContext: Record<string, any>): Logger {
    return new Logger({ ...this.context, ...additionalContext });
  }

  debug(message: string, data?: any): void {
    console.debug({
      level: 'debug',
      message,
      ...this.context,
      ...data,
      timestamp: new Date().toISOString(),
    });
  }
}

// 使用例
const baseLogger = new Logger({ service: 'api' });

app.use((req, res, next) => {
  req.logger = baseLogger.withContext({
    requestId: req.id,
    userId: req.user?.id,
  });
  next();
});

app.get('/api/users/:id', async (req, res) => {
  req.logger.debug('Fetching user', { userId: req.params.id });
  // ...
});
```

## メモリリークのデバッグ

### ヒープスナップショットの取得

```typescript
import v8 from 'v8';
import fs from 'fs';

// ヒープスナップショットを保存
function takeHeapSnapshot(filename: string) {
  const snapshotStream = v8.writeHeapSnapshot(filename);
  console.log('Heap snapshot written to:', snapshotStream);
}

// 定期的にスナップショット取得
setInterval(() => {
  const filename = `heap-${Date.now()}.heapsnapshot`;
  takeHeapSnapshot(filename);
}, 60000 * 5); // 5分ごと

// API経由で取得
app.get('/debug/heap-snapshot', (req, res) => {
  const filename = `heap-${Date.now()}.heapsnapshot`;
  takeHeapSnapshot(filename);
  res.json({ snapshot: filename });
});
```

### メモリ使用量のモニタリング

```typescript
import { performance } from 'perf_hooks';

function monitorMemory() {
  const usage = process.memoryUsage();

  console.log({
    rss: `${(usage.rss / 1024 / 1024).toFixed(2)} MB`, // 総メモリ
    heapTotal: `${(usage.heapTotal / 1024 / 1024).toFixed(2)} MB`,
    heapUsed: `${(usage.heapUsed / 1024 / 1024).toFixed(2)} MB`,
    external: `${(usage.external / 1024 / 1024).toFixed(2)} MB`,
  });
}

// 定期的にモニタリング
setInterval(monitorMemory, 10000);

// メモリ使用量が閾値を超えたら警告
function checkMemoryThreshold() {
  const usage = process.memoryUsage();
  const heapUsedMB = usage.heapUsed / 1024 / 1024;

  if (heapUsedMB > 500) {
    console.warn('⚠️  High memory usage:', heapUsedMB, 'MB');
    // ヒープスナップショット取得
    takeHeapSnapshot(`high-memory-${Date.now()}.heapsnapshot`);
  }
}
```

### よくあるメモリリークパターン

```typescript
// ❌ Bad: グローバル変数にデータを蓄積
const cache = {};

app.get('/api/users/:id', async (req, res) => {
  cache[req.params.id] = await getUser(req.params.id);
  res.json(cache[req.params.id]);
});

// ✅ Good: LRUキャッシュを使用
import LRU from 'lru-cache';

const cache = new LRU({ max: 1000, ttl: 1000 * 60 * 5 });

// ❌ Bad: イベントリスナーの解除忘れ
function setupListener() {
  const emitter = new EventEmitter();
  emitter.on('data', (data) => {
    // 処理
  });
  // emitter.removeAllListeners() を忘れている
}

// ✅ Good: 適切にクリーンアップ
function setupListener() {
  const emitter = new EventEmitter();
  const handler = (data) => {
    // 処理
  };

  emitter.on('data', handler);

  return () => {
    emitter.removeListener('data', handler);
  };
}
```

## 非同期処理のデバッグ

### Promise の未処理拒否を追跡

```typescript
// 未処理の Promise 拒否を検出
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise);
  console.error('Reason:', reason);

  // スタックトレースを保存
  if (reason instanceof Error) {
    console.error('Stack:', reason.stack);
  }
});

// 例: 未処理の Promise
async function problematicFunction() {
  throw new Error('This will be unhandled');
}

// ❌ Bad: catch していない
problematicFunction();

// ✅ Good: 適切に catch
problematicFunction().catch((error) => {
  console.error('Caught error:', error);
});
```

### 非同期スタックトレースの改善

```typescript
// Node.js の --async-stack-traces フラグを有効化
// node --async-stack-traces server.js

// または環境変数で設定
// NODE_OPTIONS=--async-stack-traces node server.js

// エラーにコンテキストを追加
class ContextError extends Error {
  constructor(message: string, public context: any) {
    super(message);
    this.name = 'ContextError';
  }
}

async function fetchUserData(userId: string) {
  try {
    return await prisma.user.findUnique({
      where: { id: userId },
    });
  } catch (error) {
    throw new ContextError('Failed to fetch user', {
      userId,
      originalError: error,
    });
  }
}
```

## パフォーマンスデバッグ

### CPU プロファイリング

```typescript
import { Session } from 'inspector';
import fs from 'fs';

function startCPUProfile(duration: number = 10000) {
  const session = new Session();
  session.connect();

  session.post('Profiler.enable', () => {
    session.post('Profiler.start', () => {
      console.log('CPU profiling started');

      setTimeout(() => {
        session.post('Profiler.stop', (err, { profile }) => {
          if (!err) {
            fs.writeFileSync('cpu-profile.cpuprofile', JSON.stringify(profile));
            console.log('CPU profile saved');
          }
          session.disconnect();
        });
      }, duration);
    });
  });
}

// API経由でプロファイリング開始
app.post('/debug/profile-cpu', (req, res) => {
  const duration = req.body.duration || 10000;
  startCPUProfile(duration);
  res.json({ message: 'CPU profiling started', duration });
});
```

### 遅いクエリの検出

```typescript
// Prisma のログを有効化
const prisma = new PrismaClient({
  log: [
    {
      emit: 'event',
      level: 'query',
    },
  ],
});

// 遅いクエリを記録
prisma.$on('query', (e) => {
  if (e.duration > 1000) {
    // 1秒以上
    console.warn('Slow query detected:', {
      query: e.query,
      duration: e.duration,
      params: e.params,
    });
  }
});

// カスタムミドルウェアで計測
prisma.$use(async (params, next) => {
  const start = Date.now();
  const result = await next(params);
  const duration = Date.now() - start;

  if (duration > 1000) {
    console.warn('Slow operation:', {
      model: params.model,
      action: params.action,
      duration,
    });
  }

  return result;
});
```

## リクエストのトレーシング

### リクエストIDの追跡

```typescript
import { randomUUID } from 'crypto';

// リクエストIDをミドルウェアで生成
app.use((req, res, next) => {
  req.id = req.headers['x-request-id'] || randomUUID();
  res.setHeader('X-Request-ID', req.id);
  next();
});

// すべてのログにリクエストIDを含める
app.use((req, res, next) => {
  const originalLog = console.log;
  console.log = (...args) => {
    originalLog(`[${req.id}]`, ...args);
  };
  next();
});

// ログにリクエストIDを含める
app.get('/api/users/:id', async (req, res) => {
  console.log('Fetching user'); // [uuid] Fetching user
  const user = await getUser(req.params.id);
  res.json(user);
});
```

### 分散トレーシング (OpenTelemetry)

```typescript
import { trace } from '@opentelemetry/api';

const tracer = trace.getTracer('my-app');

app.get('/api/complex-operation', async (req, res) => {
  const span = tracer.startSpan('complex-operation');

  try {
    // 子スパンを作成
    const childSpan1 = tracer.startSpan('fetch-data', {
      parent: span,
    });
    const data = await fetchData();
    childSpan1.end();

    const childSpan2 = tracer.startSpan('process-data', {
      parent: span,
    });
    const result = await processData(data);
    childSpan2.end();

    res.json(result);
  } finally {
    span.end();
  }
});
```

## データベースデバッグ

### Prisma のデバッグ

```typescript
// 詳細なログを有効化
const prisma = new PrismaClient({
  log: [
    { level: 'query', emit: 'event' },
    { level: 'error', emit: 'stdout' },
    { level: 'warn', emit: 'stdout' },
  ],
});

prisma.$on('query', (e) => {
  console.log('Query:', e.query);
  console.log('Params:', e.params);
  console.log('Duration:', e.duration, 'ms');
});

// クエリの実行計画を表示
const result = await prisma.$queryRaw`
  EXPLAIN ANALYZE
  SELECT * FROM users WHERE email = 'test@example.com'
`;
```

### トランザクションのデバッグ

```typescript
async function debugTransaction() {
  try {
    await prisma.$transaction(async (tx) => {
      console.log('Transaction started');

      const user = await tx.user.create({
        data: { email: 'test@example.com', name: 'Test' },
      });
      console.log('User created:', user.id);

      const profile = await tx.profile.create({
        data: { userId: user.id, bio: 'Test bio' },
      });
      console.log('Profile created:', profile.id);

      console.log('Transaction will commit');
    });

    console.log('Transaction committed');
  } catch (error) {
    console.error('Transaction rolled back:', error);
  }
}
```

## HTTPリクエストのデバッグ

### リクエスト・レスポンスのロギング

```typescript
import morgan from 'morgan';

// カスタムフォーマット
morgan.token('body', (req) => JSON.stringify(req.body));
morgan.token('response-body', (req, res) => res.locals.body);

app.use(
  morgan(':method :url :status :response-time ms - :body', {
    skip: (req) => req.url === '/health',
  })
);

// レスポンスボディをログ
app.use((req, res, next) => {
  const originalSend = res.send;

  res.send = function (data) {
    res.locals.body = data;
    return originalSend.call(this, data);
  };

  next();
});
```

### 外部API呼び出しのデバッグ

```typescript
import axios from 'axios';

// axios インターセプター
axios.interceptors.request.use((config) => {
  console.log('Request:', {
    method: config.method,
    url: config.url,
    headers: config.headers,
    data: config.data,
  });
  return config;
});

axios.interceptors.response.use(
  (response) => {
    console.log('Response:', {
      status: response.status,
      data: response.data,
    });
    return response;
  },
  (error) => {
    console.error('Request failed:', {
      url: error.config?.url,
      status: error.response?.status,
      data: error.response?.data,
    });
    return Promise.reject(error);
  }
);
```

## 本番環境でのデバッグ

### リモートデバッグの有効化

```bash
# SSH経由でリモートサーバーに接続
ssh -L 9229:localhost:9229 user@remote-server

# リモートサーバーでアプリを起動
node --inspect=0.0.0.0:9229 server.js

# ローカルのChrome DevToolsで接続
```

### 本番環境での安全なデバッグ

```typescript
// デバッグエンドポイントを認証で保護
app.get('/debug/info', authenticate, authorize('admin'), (req, res) => {
  res.json({
    memory: process.memoryUsage(),
    uptime: process.uptime(),
    env: process.env.NODE_ENV,
    version: process.version,
  });
});

// 一時的なデバッグログの有効化
let debugEnabled = false;

app.post('/debug/enable', authenticate, authorize('admin'), (req, res) => {
  debugEnabled = true;
  setTimeout(() => {
    debugEnabled = false;
  }, 300000); // 5分後に自動無効化

  res.json({ message: 'Debug logging enabled for 5 minutes' });
});

// デバッグログ
function debugLog(message: string, data?: any) {
  if (debugEnabled) {
    console.log('[DEBUG]', message, data);
  }
}
```

## デバッグツール

### Node.js Inspector

```bash
# Chrome DevTools を使用
node --inspect server.js

# VS Code でアタッチ
node --inspect-brk server.js
```

### clinic.js によるパフォーマンス診断

```bash
# インストール
npm install -g clinic

# Doctor: 全体的な健全性チェック
clinic doctor -- node server.js

# Bubbleprof: 非同期処理の可視化
clinic bubbleprof -- node server.js

# Flame: CPU使用率の可視化
clinic flame -- node server.js

# Heap Profiler: メモリ使用量
clinic heapprofiler -- node server.js
```

### 0x による火炎グラフ生成

```bash
# インストール
npm install -g 0x

# プロファイリング実行
0x server.js

# ブラウザで火炎グラフを表示
```

## まとめ

デバッグのベストプラクティス:

- ✅ Chrome DevTools/VS Code で効率的にデバッグ
- ✅ 戦略的にログを配置
- ✅ メモリリークを定期的に検出
- ✅ 非同期エラーを確実にキャッチ
- ✅ CPU/メモリプロファイリングで最適化
- ✅ リクエストIDで追跡可能に
- ✅ データベースクエリを監視
- ✅ 本番環境でも安全にデバッグ
- ✅ clinic.js/0x でパフォーマンス分析
- ❌ 本番環境でデバッグポートを公開しない
- ❌ 機密情報をログに出力しない

次の章では、セキュリティベストプラクティスについて学びます。
