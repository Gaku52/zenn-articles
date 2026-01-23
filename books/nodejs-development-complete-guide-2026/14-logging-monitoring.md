---
title: "ロギングとモニタリング - システムの健全性を保つ"
---

# ロギングとモニタリング - システムの健全性を保つ

効果的なロギングとモニタリングにより、問題の早期発見、パフォーマンス分析、セキュリティ監査を実現する方法を学びます。

## ロギングの基本原則

### ログレベルの定義

```typescript
enum LogLevel {
  ERROR = 0,   // システム障害
  WARN = 1,    // 警告（対処必要かも）
  INFO = 2,    // 通常の情報
  DEBUG = 3,   // デバッグ情報
  TRACE = 4,   // 詳細トレース
}

// 本番環境: ERROR, WARN, INFO
// 開発環境: すべて
const LOG_LEVEL = process.env.NODE_ENV === 'production'
  ? LogLevel.INFO
  : LogLevel.TRACE;
```

### 構造化ログの重要性

```typescript
// ❌ Bad: 文字列のみのログ
console.log('User login failed for john@example.com');

// ✅ Good: 構造化されたログ
logger.error('User login failed', {
  email: 'john@example.com',
  reason: 'invalid_password',
  ip: '192.168.1.100',
  timestamp: new Date().toISOString(),
});
```

## Winston によるロギング

### 基本設定

```typescript
import winston from 'winston';

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'api-server' },
  transports: [
    // エラーログ専用ファイル
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error',
      maxsize: 5242880, // 5MB
      maxFiles: 5,
    }),

    // すべてのログ
    new winston.transports.File({
      filename: 'logs/combined.log',
      maxsize: 5242880,
      maxFiles: 5,
    }),
  ],
});

// 開発環境: コンソール出力
if (process.env.NODE_ENV !== 'production') {
  logger.add(
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      ),
    })
  );
}
```

### リクエストロギング

```typescript
import { Request, Response, NextFunction } from 'express';

function requestLogger(req: Request, res: Response, next: NextFunction) {
  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;

    logger.info('HTTP Request', {
      method: req.method,
      url: req.url,
      statusCode: res.statusCode,
      duration,
      userAgent: req.headers['user-agent'],
      ip: req.ip,
      userId: req.user?.id,
    });
  });

  next();
}

app.use(requestLogger);
```

### エラーロギング

```typescript
function errorLogger(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
) {
  logger.error('Error occurred', {
    error: {
      name: err.name,
      message: err.message,
      stack: err.stack,
    },
    request: {
      method: req.method,
      url: req.url,
      headers: req.headers,
      body: req.body,
      query: req.query,
      params: req.params,
    },
    user: {
      id: req.user?.id,
      email: req.user?.email,
    },
  });

  next(err);
}

app.use(errorLogger);
```

## Pino による高速ロギング

### Pino の基本

```typescript
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: {
    target: 'pino-pretty',
    options: {
      colorize: true,
      translateTime: 'SYS:standard',
      ignore: 'pid,hostname',
    },
  },
});

// 使用例
logger.info('Server started');
logger.warn({ port: 3000 }, 'Port already in use');
logger.error({ err: error }, 'Database connection failed');
```

### Express との統合

```typescript
import expressPino from 'express-pino-logger';

const expressLogger = expressPino({
  logger,
  autoLogging: true,
  customLogLevel: (req, res, err) => {
    if (res.statusCode >= 500) return 'error';
    if (res.statusCode >= 400) return 'warn';
    if (res.statusCode >= 300) return 'info';
    return 'debug';
  },
  customSuccessMessage: (req, res) => {
    return `${req.method} ${req.url} ${res.statusCode}`;
  },
});

app.use(expressLogger);
```

### 子ロガーの作成

```typescript
// 特定のモジュール用ロガー
const dbLogger = logger.child({ module: 'database' });
const authLogger = logger.child({ module: 'auth' });

dbLogger.info('Query executed', { query: 'SELECT * FROM users' });
authLogger.warn('Login attempt failed', { email: 'user@example.com' });
```

## ログのフィルタリングと機密情報の保護

### 機密情報のマスキング

```typescript
import { redactObject } from 'pino-std-serializers';

const logger = pino({
  serializers: {
    req: (req) => ({
      method: req.method,
      url: req.url,
      headers: {
        ...req.headers,
        authorization: '[REDACTED]',
      },
    }),
    res: (res) => ({
      statusCode: res.statusCode,
    }),
  },
});

// カスタムマスキング
function maskSensitiveData(obj: any): any {
  const sensitiveKeys = ['password', 'token', 'secret', 'apiKey'];

  const masked = { ...obj };

  for (const key of Object.keys(masked)) {
    if (sensitiveKeys.includes(key)) {
      masked[key] = '[REDACTED]';
    } else if (typeof masked[key] === 'object') {
      masked[key] = maskSensitiveData(masked[key]);
    }
  }

  return masked;
}

logger.info('User data', maskSensitiveData(userData));
```

## アプリケーションメトリクス

### Prometheus によるメトリクス収集

```typescript
import client from 'prom-client';

// デフォルトメトリクス
client.collectDefaultMetrics({ prefix: 'nodejs_' });

// カスタムメトリクス
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.1, 0.5, 1, 2, 5],
});

const httpRequestTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
});

const activeUsers = new client.Gauge({
  name: 'active_users',
  help: 'Number of active users',
});

// ミドルウェア
app.use((req, res, next) => {
  const start = Date.now();

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;

    httpRequestDuration
      .labels(req.method, req.route?.path || req.path, res.statusCode.toString())
      .observe(duration);

    httpRequestTotal
      .labels(req.method, req.route?.path || req.path, res.statusCode.toString())
      .inc();
  });

  next();
});

// メトリクスエンドポイント
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});
```

### ビジネスメトリクス

```typescript
const userRegistrations = new client.Counter({
  name: 'user_registrations_total',
  help: 'Total number of user registrations',
});

const orderTotal = new client.Counter({
  name: 'order_total',
  help: 'Total value of orders',
  labelNames: ['status'],
});

const cacheHitRate = new client.Gauge({
  name: 'cache_hit_rate',
  help: 'Cache hit rate',
});

// 使用例
app.post('/api/users', async (req, res) => {
  const user = await createUser(req.body);
  userRegistrations.inc();
  res.json(user);
});

app.post('/api/orders', async (req, res) => {
  const order = await createOrder(req.body);
  orderTotal.labels(order.status).inc(order.amount);
  res.json(order);
});
```

## ヘルスチェック

### 基本的なヘルスチェック

```typescript
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
  });
});
```

### 詳細なヘルスチェック

```typescript
interface HealthCheck {
  status: 'healthy' | 'unhealthy' | 'degraded';
  checks: {
    database: HealthStatus;
    redis: HealthStatus;
    disk: HealthStatus;
    memory: HealthStatus;
  };
  timestamp: string;
}

interface HealthStatus {
  status: 'ok' | 'error';
  message?: string;
  responseTime?: number;
}

app.get('/health/detailed', async (req, res) => {
  const health: HealthCheck = {
    status: 'healthy',
    checks: {
      database: await checkDatabase(),
      redis: await checkRedis(),
      disk: await checkDisk(),
      memory: checkMemory(),
    },
    timestamp: new Date().toISOString(),
  };

  // いずれかが error なら unhealthy
  if (Object.values(health.checks).some((c) => c.status === 'error')) {
    health.status = 'unhealthy';
  }

  const statusCode = health.status === 'healthy' ? 200 : 503;
  res.status(statusCode).json(health);
});

async function checkDatabase(): Promise<HealthStatus> {
  const start = Date.now();
  try {
    await prisma.$queryRaw`SELECT 1`;
    return {
      status: 'ok',
      responseTime: Date.now() - start,
    };
  } catch (error) {
    return {
      status: 'error',
      message: error.message,
    };
  }
}

async function checkRedis(): Promise<HealthStatus> {
  const start = Date.now();
  try {
    await redis.ping();
    return {
      status: 'ok',
      responseTime: Date.now() - start,
    };
  } catch (error) {
    return {
      status: 'error',
      message: error.message,
    };
  }
}

function checkMemory(): HealthStatus {
  const usage = process.memoryUsage();
  const usedMB = usage.heapUsed / 1024 / 1024;
  const totalMB = usage.heapTotal / 1024 / 1024;
  const percentage = (usedMB / totalMB) * 100;

  if (percentage > 90) {
    return {
      status: 'error',
      message: `Memory usage at ${percentage.toFixed(2)}%`,
    };
  }

  return { status: 'ok' };
}

async function checkDisk(): Promise<HealthStatus> {
  const diskUsage = await import('check-disk-space');
  const info = await diskUsage.default('/');

  const percentage = ((info.size - info.free) / info.size) * 100;

  if (percentage > 90) {
    return {
      status: 'error',
      message: `Disk usage at ${percentage.toFixed(2)}%`,
    };
  }

  return { status: 'ok' };
}
```

## APM (Application Performance Monitoring)

### New Relic 統合

```typescript
// newrelic.js
exports.config = {
  app_name: ['My Application'],
  license_key: process.env.NEW_RELIC_LICENSE_KEY,
  logging: {
    level: 'info',
  },
  allow_all_headers: true,
  attributes: {
    exclude: [
      'request.headers.cookie',
      'request.headers.authorization',
    ],
  },
};

// アプリケーション起動時（最初に読み込む）
require('newrelic');
```

### カスタムトランザクション

```typescript
import newrelic from 'newrelic';

app.post('/api/process-order', async (req, res) => {
  // カスタムトランザクション
  await newrelic.startBackgroundTransaction('process-order', async () => {
    // 計測したい処理
    const order = await processOrder(req.body);

    // カスタム属性
    newrelic.addCustomAttribute('orderId', order.id);
    newrelic.addCustomAttribute('amount', order.amount);

    res.json(order);
  });
});
```

## ログ集約とクエリ

### ELK Stack (Elasticsearch, Logstash, Kibana)

```typescript
import winston from 'winston';
import { ElasticsearchTransport } from 'winston-elasticsearch';

const esTransport = new ElasticsearchTransport({
  level: 'info',
  clientOpts: {
    node: process.env.ELASTICSEARCH_URL,
    auth: {
      username: process.env.ELASTICSEARCH_USER,
      password: process.env.ELASTICSEARCH_PASSWORD,
    },
  },
  index: 'app-logs',
});

const logger = winston.createLogger({
  transports: [esTransport],
});
```

### CloudWatch Logs (AWS)

```typescript
import winston from 'winston';
import CloudWatchTransport from 'winston-cloudwatch';

const cloudWatchTransport = new CloudWatchTransport({
  logGroupName: '/aws/app/my-application',
  logStreamName: `${process.env.NODE_ENV}-${new Date().toISOString().split('T')[0]}`,
  awsRegion: process.env.AWS_REGION,
  jsonMessage: true,
});

const logger = winston.createLogger({
  transports: [cloudWatchTransport],
});
```

## パフォーマンス計測

### カスタムタイマー

```typescript
class PerformanceTimer {
  private timers = new Map<string, number>();

  start(label: string): void {
    this.timers.set(label, Date.now());
  }

  end(label: string): number {
    const start = this.timers.get(label);
    if (!start) {
      throw new Error(`Timer ${label} not started`);
    }

    const duration = Date.now() - start;
    this.timers.delete(label);

    logger.debug('Performance', { label, duration });

    return duration;
  }
}

// 使用例
const timer = new PerformanceTimer();

app.get('/api/data', async (req, res) => {
  timer.start('fetch-data');
  const data = await fetchData();
  timer.end('fetch-data');

  timer.start('process-data');
  const processed = await processData(data);
  timer.end('process-data');

  res.json(processed);
});
```

### Node.js Performance Hooks

```typescript
import { performance, PerformanceObserver } from 'perf_hooks';

const obs = new PerformanceObserver((items) => {
  items.getEntries().forEach((entry) => {
    logger.info('Performance measurement', {
      name: entry.name,
      duration: entry.duration,
    });
  });
});

obs.observe({ entryTypes: ['measure'] });

// 使用例
async function processData(data: any) {
  performance.mark('start-process');

  // 処理
  const result = await heavyComputation(data);

  performance.mark('end-process');
  performance.measure('process-data', 'start-process', 'end-process');

  return result;
}
```

## アラートとノーティフィケーション

### エラー率のモニタリング

```typescript
import nodemailer from 'nodemailer';

class AlertService {
  private errorCount = 0;
  private lastAlertTime = 0;
  private alertThreshold = 10; // 10エラー/分
  private alertCooldown = 300000; // 5分

  async checkAndAlert(error: Error): Promise<void> {
    this.errorCount++;

    const now = Date.now();
    const timeSinceLastAlert = now - this.lastAlertTime;

    if (
      this.errorCount >= this.alertThreshold &&
      timeSinceLastAlert > this.alertCooldown
    ) {
      await this.sendAlert(error);
      this.lastAlertTime = now;
      this.errorCount = 0;
    }
  }

  private async sendAlert(error: Error): Promise<void> {
    const transporter = nodemailer.createTransporter({
      host: process.env.SMTP_HOST,
      port: 587,
      auth: {
        user: process.env.SMTP_USER,
        pass: process.env.SMTP_PASS,
      },
    });

    await transporter.sendMail({
      from: 'alerts@example.com',
      to: 'team@example.com',
      subject: '🚨 High error rate detected',
      text: `Error count: ${this.errorCount}\nLast error: ${error.message}`,
    });
  }
}

const alertService = new AlertService();

app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  alertService.checkAndAlert(err);
  next(err);
});
```

## 分散トレーシング

### OpenTelemetry

```typescript
import { NodeTracerProvider } from '@opentelemetry/sdk-trace-node';
import { registerInstrumentations } from '@opentelemetry/instrumentation';
import { HttpInstrumentation } from '@opentelemetry/instrumentation-http';
import { ExpressInstrumentation } from '@opentelemetry/instrumentation-express';

const provider = new NodeTracerProvider();
provider.register();

registerInstrumentations({
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation(),
  ],
});

// カスタムスパン
import { trace } from '@opentelemetry/api';

const tracer = trace.getTracer('my-application');

async function fetchUserData(userId: string) {
  const span = tracer.startSpan('fetchUserData');
  span.setAttribute('userId', userId);

  try {
    const user = await prisma.user.findUnique({
      where: { id: userId },
    });

    span.setStatus({ code: 0 }); // OK
    return user;
  } catch (error) {
    span.setStatus({ code: 2, message: error.message }); // ERROR
    throw error;
  } finally {
    span.end();
  }
}
```

## まとめ

ロギングとモニタリングのベストプラクティス:

- ✅ 構造化ログで検索・分析を容易に
- ✅ 適切なログレベルを設定
- ✅ 機密情報をマスキング
- ✅ メトリクスでシステム健全性を可視化
- ✅ ヘルスチェックで自動復旧を実現
- ✅ APMで詳細なパフォーマンス分析
- ✅ ログ集約で一元管理
- ✅ アラートで迅速な対応
- ✅ 分散トレーシングでボトルネック特定
- ❌ ログに機密情報を含めない
- ❌ 過剰なログでストレージを圧迫しない

次の章では、デバッグテクニックについて学びます。
