---
title: "クラスタリングとスケーリング - 複数コアを活用する"
---

# クラスタリングとスケーリング - 複数コアを活用する

Node.jsアプリケーションを水平・垂直にスケールさせ、高いスループットと可用性を実現する方法を学びます。

## Node.jsのシングルスレッド問題

### 基本的な制約

Node.jsのイベントループはシングルスレッドで動作します:

```javascript
// シングルプロセスのNode.js
const express = require('express');
const app = express();

app.get('/api/heavy', (req, res) => {
  // このCPU集約的な処理は他のリクエストをブロックする
  const result = heavyComputation();
  res.json({ result });
});

app.listen(3000);
// このプロセスは1つのCPUコアしか使用しない
```

**問題点:**
- 1つのCPUコアしか使用できない
- CPU集約的な処理が他のリクエストをブロック
- マルチコアサーバーのリソースが無駄になる

## Cluster モジュールによる水平スケーリング

### 基本的なクラスタリング

```typescript
// cluster.ts
import cluster from 'cluster';
import os from 'os';
import { createServer } from './server';

const numCPUs = os.cpus().length;

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} is running`);

  // CPUコア数分のワーカーを起動
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died`);
    console.log('Starting a new worker');
    cluster.fork(); // 自動再起動
  });
} else {
  // ワーカープロセス: 実際のサーバーを起動
  createServer();
  console.log(`Worker ${process.pid} started`);
}
```

### ワーカー間の通信

```typescript
if (cluster.isPrimary) {
  const workers = [];

  for (let i = 0; i < numCPUs; i++) {
    const worker = cluster.fork();
    workers.push(worker);

    // ワーカーからのメッセージを受信
    worker.on('message', (msg) => {
      console.log(`Message from worker ${worker.id}:`, msg);

      // 全ワーカーにブロードキャスト
      if (msg.type === 'broadcast') {
        workers.forEach((w) => {
          if (w.id !== worker.id) {
            w.send(msg.data);
          }
        });
      }
    });
  }
} else {
  // ワーカープロセス
  const app = createServer();

  // プライマリにメッセージを送信
  process.send({
    type: 'broadcast',
    data: { event: 'cache-invalidated', key: 'user:123' },
  });

  // プライマリからのメッセージを受信
  process.on('message', (msg) => {
    console.log('Received from primary:', msg);
    // キャッシュを無効化など
    cache.del(msg.key);
  });
}
```

### グレースフルシャットダウン

```typescript
if (cluster.isPrimary) {
  // SIGTERM シグナルを受信
  process.on('SIGTERM', () => {
    console.log('SIGTERM received, shutting down gracefully');

    // 各ワーカーに終了を通知
    for (const id in cluster.workers) {
      cluster.workers[id].send('shutdown');
      cluster.workers[id].disconnect();
    }

    // 10秒後に強制終了
    setTimeout(() => {
      console.log('Forcing shutdown');
      process.exit(1);
    }, 10000);
  });
} else {
  const server = app.listen(3000);

  process.on('message', (msg) => {
    if (msg === 'shutdown') {
      console.log(`Worker ${process.pid} shutting down`);

      // 新しいリクエストを受け付けない
      server.close(() => {
        console.log('Server closed');
        process.exit(0);
      });

      // 既存のリクエストの完了を待つ（最大5秒）
      setTimeout(() => {
        console.log('Forcing shutdown');
        process.exit(1);
      }, 5000);
    }
  });
}
```

## PM2による本番運用

### PM2の基本設定

```javascript
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name: 'api-server',
      script: './dist/server.js',
      instances: 'max', // CPUコア数分起動
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
        PORT: 3000,
      },
      error_file: './logs/err.log',
      out_file: './logs/out.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
      merge_logs: true,
      autorestart: true,
      max_restarts: 10,
      min_uptime: '10s',
      max_memory_restart: '1G',
    },
  ],
};
```

### PM2コマンド

```bash
# アプリケーション起動
pm2 start ecosystem.config.js

# クラスタのスケーリング
pm2 scale api-server 8  # 8インスタンスに変更

# ゼロダウンタイムリロード
pm2 reload api-server

# ログ確認
pm2 logs api-server

# モニタリング
pm2 monit

# プロセス情報
pm2 list

# 停止
pm2 stop api-server

# 削除
pm2 delete api-server
```

### PM2のゼロダウンタイムデプロイ

```javascript
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name: 'api-server',
      script: './dist/server.js',
      instances: 4,
      exec_mode: 'cluster',
      wait_ready: true, // ready シグナルを待つ
      listen_timeout: 10000,
      kill_timeout: 5000,
    },
  ],
};
```

```typescript
// server.ts
import express from 'express';

const app = express();

// ... ミドルウェア、ルート設定

const server = app.listen(3000, () => {
  console.log('Server started on port 3000');

  // PM2に準備完了を通知
  if (process.send) {
    process.send('ready');
  }
});

// グレースフルシャットダウン
process.on('SIGINT', () => {
  console.log('SIGINT received, shutting down gracefully');

  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
});
```

## 負荷分散戦略

### Nginxによるロードバランシング

```nginx
# /etc/nginx/nginx.conf
upstream nodejs_cluster {
    # ラウンドロビン（デフォルト）
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;

    # keepalive接続をプール
    keepalive 64;
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://nodejs_cluster;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### IP HashによるSession Affinity

```nginx
upstream nodejs_cluster {
    # クライアントIPに基づいて同じサーバーにルーティング
    ip_hash;

    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
```

### Least Connections

```nginx
upstream nodejs_cluster {
    # 最も接続数が少ないサーバーにルーティング
    least_conn;

    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
```

## セッション管理

### Redisを使った共有セッション

```typescript
import express from 'express';
import session from 'express-session';
import RedisStore from 'connect-redis';
import { createClient } from 'redis';

const app = express();

// Redis クライアント
const redisClient = createClient({
  url: 'redis://localhost:6379',
});
redisClient.connect();

// Redisベースのセッションストア
app.use(
  session({
    store: new RedisStore({ client: redisClient }),
    secret: process.env.SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
    cookie: {
      secure: process.env.NODE_ENV === 'production',
      httpOnly: true,
      maxAge: 1000 * 60 * 60 * 24, // 24時間
    },
  })
);

app.get('/api/profile', (req, res) => {
  // どのワーカーでも同じセッションにアクセス可能
  if (!req.session.userId) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  res.json({ userId: req.session.userId });
});
```

### JWTによるステートレス認証

```typescript
import jwt from 'jsonwebtoken';

// ログイン時にJWTを発行
app.post('/api/login', async (req, res) => {
  const { email, password } = req.body;

  const user = await authenticateUser(email, password);
  if (!user) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // JWT トークンを生成
  const token = jwt.sign(
    { userId: user.id, email: user.email },
    process.env.JWT_SECRET,
    { expiresIn: '7d' }
  );

  res.json({ token });
});

// 認証ミドルウェア
const authenticate = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];

  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

app.get('/api/profile', authenticate, (req, res) => {
  // セッションストア不要、完全にステートレス
  res.json({ userId: req.user.userId });
});
```

## ヘルスチェックとモニタリング

### ヘルスチェックエンドポイント

```typescript
app.get('/health', async (req, res) => {
  const health = {
    uptime: process.uptime(),
    timestamp: Date.now(),
    status: 'OK',
    checks: {},
  };

  try {
    // データベース接続チェック
    await prisma.$queryRaw`SELECT 1`;
    health.checks.database = 'OK';
  } catch (error) {
    health.checks.database = 'FAIL';
    health.status = 'DEGRADED';
  }

  try {
    // Redis接続チェック
    await redisClient.ping();
    health.checks.redis = 'OK';
  } catch (error) {
    health.checks.redis = 'FAIL';
    health.status = 'DEGRADED';
  }

  const statusCode = health.status === 'OK' ? 200 : 503;
  res.status(statusCode).json(health);
});

// Readiness probe（準備完了）
app.get('/ready', (req, res) => {
  if (!isReady) {
    return res.status(503).json({ status: 'NOT_READY' });
  }
  res.json({ status: 'READY' });
});

// Liveness probe（生存確認）
app.get('/live', (req, res) => {
  res.json({ status: 'ALIVE' });
});
```

## Kubernetes でのスケーリング

### Deployment 設定

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nodejs-api
  template:
    metadata:
      labels:
        app: nodejs-api
    spec:
      containers:
        - name: api
          image: your-registry/nodejs-api:latest
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              value: 'production'
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: api-secrets
                  key: database-url
          resources:
            requests:
              memory: '256Mi'
              cpu: '250m'
            limits:
              memory: '512Mi'
              cpu: '500m'
          livenessProbe:
            httpGet:
              path: /live
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: nodejs-api-service
spec:
  selector:
    app: nodejs-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: LoadBalancer
```

### Horizontal Pod Autoscaler

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nodejs-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nodejs-api
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

## まとめ

クラスタリングとスケーリングのベストプラクティス:

- ✅ Cluster モジュールで複数コアを活用
- ✅ PM2で本番運用を簡素化
- ✅ Nginxで負荷分散
- ✅ Redisで共有セッションを管理
- ✅ JWTでステートレス認証
- ✅ ヘルスチェックで可用性を確保
- ✅ Kubernetesで自動スケーリング
- ✅ グレースフルシャットダウンを実装
- ❌ シングルプロセスでマルチコアを無駄にしない
- ❌ メモリ内セッションをクラスタ環境で使わない

次の章では、キャッシング戦略について詳しく学びます。
