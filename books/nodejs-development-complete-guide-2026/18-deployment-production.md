---
title: "デプロイと本番運用 - 実践的なプロダクション戦略"
---

# デプロイと本番運用 - 実践的なプロダクション戦略

Node.jsアプリケーションを安全に本番環境にデプロイし、運用するための実践的な手法を学びます。

## デプロイ前のチェックリスト

### 必須設定

```typescript
// 環境変数の検証
const requiredEnvVars = [
  'DATABASE_URL',
  'REDIS_URL',
  'JWT_SECRET',
  'NODE_ENV',
];

requiredEnvVars.forEach((varName) => {
  if (!process.env[varName]) {
    throw new Error(`Missing required environment variable: ${varName}`);
  }
});

// NODE_ENV の確認
if (process.env.NODE_ENV !== 'production') {
  console.warn('⚠️  NODE_ENV is not set to production');
}
```

### セキュリティチェック

```bash
# 脆弱性スキャン
npm audit
npm audit fix

# 依存関係の更新
npm outdated
npm update

# TypeScript のビルドエラーチェック
npm run build

# テスト実行
npm test

# Lintチェック
npm run lint
```

## Docker化

### Dockerfile (マルチステージビルド)

```dockerfile
# ビルドステージ
FROM node:20-alpine AS builder

WORKDIR /app

# 依存関係のインストール
COPY package*.json ./
RUN npm ci

# ソースコードをコピー
COPY . .

# Prisma クライアント生成
RUN npx prisma generate

# TypeScript ビルド
RUN npm run build

# 本番ステージ
FROM node:20-alpine

WORKDIR /app

# 本番依存関係のみインストール
COPY package*.json ./
RUN npm ci --only=production

# Prisma クライアント生成
COPY prisma ./prisma
RUN npx prisma generate

# ビルド成果物をコピー
COPY --from=builder /app/dist ./dist

# ヘルスチェック
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1))"

# 非rootユーザーで実行
USER node

EXPOSE 3000

CMD ["node", "dist/server.js"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - '3000:3000'
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://postgres:password@db:5432/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U postgres']
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

## CI/CD パイプライン

### GitHub Actions (完全版)

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test

  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_KEY }}
          script: |
            cd /app
            docker-compose pull
            docker-compose up -d
            docker-compose exec -T app npx prisma migrate deploy
```

## ゼロダウンタイムデプロイ

### PM2 によるクラスタリング

```javascript
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name: 'api',
      script: './dist/server.js',
      instances: 'max', // CPUコア数に応じて自動
      exec_mode: 'cluster',
      env_production: {
        NODE_ENV: 'production',
        PORT: 3000,
      },
      max_memory_restart: '500M',
      error_file: './logs/err.log',
      out_file: './logs/out.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    },
  ],
};
```

```bash
# デプロイコマンド
pm2 start ecosystem.config.js --env production

# ゼロダウンタイムリロード
pm2 reload api

# 監視
pm2 monit

# ログ確認
pm2 logs api
```

### Kubernetes デプロイ

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: ghcr.io/your-org/api:latest
          ports:
            - containerPort: 3000
          env:
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
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: LoadBalancer
```

## データベースマイグレーション

### 本番環境での安全なマイグレーション

```bash
# バックアップを取得
pg_dump $DATABASE_URL > backup-$(date +%Y%m%d-%H%M%S).sql

# マイグレーションをドライラン
npx prisma migrate deploy --preview-feature

# 本番マイグレーション
npx prisma migrate deploy

# ロールバック（必要な場合）
psql $DATABASE_URL < backup-20260124-150000.sql
```

### マイグレーション戦略

```typescript
// scripts/migrate.ts
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

async function runMigration() {
  try {
    console.log('Starting migration...');

    // マイグレーション実行
    await execAsync('npx prisma migrate deploy');

    console.log('Migration completed successfully');
  } catch (error) {
    console.error('Migration failed:', error);
    process.exit(1);
  }
}

runMigration();
```

## 本番環境の監視

### ヘルスチェックエンドポイント

```typescript
app.get('/health', async (req, res) => {
  const checks = {
    uptime: process.uptime(),
    timestamp: Date.now(),
    database: 'unknown',
    redis: 'unknown',
  };

  try {
    await prisma.$queryRaw`SELECT 1`;
    checks.database = 'ok';
  } catch {
    checks.database = 'error';
  }

  try {
    await redis.ping();
    checks.redis = 'ok';
  } catch {
    checks.redis = 'error';
  }

  const isHealthy = checks.database === 'ok' && checks.redis === 'ok';

  res.status(isHealthy ? 200 : 503).json(checks);
});
```

### Prometheus メトリクス

```typescript
import client from 'prom-client';

// デフォルトメトリクス
client.collectDefaultMetrics();

// カスタムメトリクス
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});
```

## ログ管理

### 構造化ログ

```typescript
import winston from 'winston';

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

// 本番環境ではコンソール出力も追加
if (process.env.NODE_ENV === 'production') {
  logger.add(
    new winston.transports.Console({
      format: winston.format.simple(),
    })
  );
}
```

## 環境別設定

### config.ts

```typescript
interface Config {
  port: number;
  database: {
    url: string;
    maxConnections: number;
  };
  redis: {
    url: string;
  };
  jwt: {
    secret: string;
    expiresIn: string;
  };
}

const configs: Record<string, Config> = {
  development: {
    port: 3000,
    database: {
      url: process.env.DATABASE_URL!,
      maxConnections: 5,
    },
    redis: {
      url: 'redis://localhost:6379',
    },
    jwt: {
      secret: 'dev-secret',
      expiresIn: '7d',
    },
  },
  production: {
    port: parseInt(process.env.PORT || '3000'),
    database: {
      url: process.env.DATABASE_URL!,
      maxConnections: 20,
    },
    redis: {
      url: process.env.REDIS_URL!,
    },
    jwt: {
      secret: process.env.JWT_SECRET!,
      expiresIn: '1d',
    },
  },
};

export const config = configs[process.env.NODE_ENV || 'development'];
```

## バックアップ戦略

### 自動バックアップスクリプト

```bash
#!/bin/bash
# scripts/backup.sh

DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="/backups"
BACKUP_FILE="$BACKUP_DIR/backup-$DATE.sql"

# PostgreSQL バックアップ
pg_dump $DATABASE_URL > $BACKUP_FILE

# 圧縮
gzip $BACKUP_FILE

# S3にアップロード
aws s3 cp $BACKUP_FILE.gz s3://my-backups/

# 7日以上古いバックアップを削除
find $BACKUP_DIR -name "*.sql.gz" -mtime +7 -delete

echo "Backup completed: $BACKUP_FILE.gz"
```

### Cron で自動化

```bash
# 毎日 2:00 AM にバックアップ
0 2 * * * /app/scripts/backup.sh
```

## スケーリング戦略

### 水平スケーリング

```yaml
# docker-compose.yml
services:
  app:
    deploy:
      replicas: 3
    # ... その他の設定

  nginx:
    image: nginx:alpine
    ports:
      - '80:80'
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - app
```

### Nginx ロードバランサー

```nginx
# nginx.conf
upstream app_servers {
    least_conn;
    server app1:3000;
    server app2:3000;
    server app3:3000;
}

server {
    listen 80;

    location / {
        proxy_pass http://app_servers;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

## まとめ

デプロイと本番運用のベストプラクティス:

- ✅ Docker でコンテナ化
- ✅ CI/CD で自動デプロイ
- ✅ ゼロダウンタイムデプロイ
- ✅ データベースマイグレーションの安全な実行
- ✅ ヘルスチェックとメトリクス監視
- ✅ 構造化ログで分析可能に
- ✅ 定期的なバックアップ
- ✅ 水平スケーリングに対応
- ❌ 本番環境で直接作業しない
- ❌ バックアップなしでマイグレーションしない

次の章では、実践的なケーススタディ（前編）を学びます。
