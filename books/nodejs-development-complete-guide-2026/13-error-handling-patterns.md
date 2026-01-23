---
title: "エラーハンドリングパターン - 堅牢なアプリを作る"
---

# エラーハンドリングパターン - 堅牢なアプリを作る

適切なエラーハンドリングにより、予期しない障害からアプリケーションを保護し、優れたユーザー体験を提供する方法を学びます。

## エラーの種類

### Node.jsにおけるエラー分類

```typescript
// 1. 操作エラー（予測可能・回復可能）
class ValidationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'ValidationError';
  }
}

class NotFoundError extends Error {
  constructor(resource: string) {
    super(`${resource} not found`);
    this.name = 'NotFoundError';
  }
}

class UnauthorizedError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'UnauthorizedError';
  }
}

// 2. プログラマーエラー（バグ・修正が必要）
// - TypeError: undefined のプロパティアクセス
// - ReferenceError: 未定義の変数
// - SyntaxError: 構文エラー
```

### カスタムエラークラスの設計

```typescript
// 基底エラークラス
abstract class AppError extends Error {
  constructor(
    message: string,
    public statusCode: number,
    public isOperational: boolean = true
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

// 具体的なエラークラス
class BadRequestError extends AppError {
  constructor(message: string) {
    super(message, 400);
  }
}

class UnauthorizedError extends AppError {
  constructor(message: string = 'Unauthorized') {
    super(message, 401);
  }
}

class ForbiddenError extends AppError {
  constructor(message: string = 'Forbidden') {
    super(message, 403);
  }
}

class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource} not found`, 404);
  }
}

class ConflictError extends AppError {
  constructor(message: string) {
    super(message, 409);
  }
}

class InternalServerError extends AppError {
  constructor(message: string = 'Internal server error') {
    super(message, 500, false); // 操作エラーではない
  }
}
```

## Express でのエラーハンドリング

### グローバルエラーハンドラー

```typescript
import { Request, Response, NextFunction } from 'express';

// エラーハンドリングミドルウェア
function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
) {
  // ログ出力
  console.error('Error:', {
    name: err.name,
    message: err.message,
    stack: err.stack,
    url: req.url,
    method: req.method,
  });

  // AppError の場合
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      error: {
        message: err.message,
        status: err.statusCode,
      },
    });
  }

  // バリデーションエラー（Zod）
  if (err.name === 'ZodError') {
    return res.status(400).json({
      error: {
        message: 'Validation failed',
        details: err.errors,
      },
    });
  }

  // Prisma エラー
  if (err.name === 'PrismaClientKnownRequestError') {
    if (err.code === 'P2002') {
      return res.status(409).json({
        error: {
          message: 'Unique constraint violation',
        },
      });
    }
  }

  // その他のエラー: 500
  res.status(500).json({
    error: {
      message: 'Internal server error',
    },
  });
}

// アプリケーションに登録
app.use(errorHandler);
```

### 非同期エラーのハンドリング

```typescript
// asyncHandler ラッパー
function asyncHandler(fn: Function) {
  return (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

// 使用例
app.get(
  '/api/users/:id',
  asyncHandler(async (req, res) => {
    const user = await prisma.user.findUnique({
      where: { id: req.params.id },
    });

    if (!user) {
      throw new NotFoundError('User');
    }

    res.json(user);
  })
);

// または express-async-errors を使用
import 'express-async-errors';

// これで自動的に catch される
app.get('/api/users/:id', async (req, res) => {
  const user = await prisma.user.findUnique({
    where: { id: req.params.id },
  });

  if (!user) {
    throw new NotFoundError('User');
  }

  res.json(user);
});
```

### バリデーションエラー

```typescript
import { z } from 'zod';

const createUserSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  name: z.string().min(1),
});

app.post('/api/users', async (req, res) => {
  try {
    const data = createUserSchema.parse(req.body);

    const user = await prisma.user.create({
      data,
    });

    res.status(201).json(user);
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({
        error: {
          message: 'Validation failed',
          details: error.errors.map((e) => ({
            field: e.path.join('.'),
            message: e.message,
          })),
        },
      });
    }
    throw error;
  }
});

// またはミドルウェア化
function validate(schema: z.ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      req.body = schema.parse(req.body);
      next();
    } catch (error) {
      next(error); // エラーハンドラーに渡す
    }
  };
}

app.post('/api/users', validate(createUserSchema), async (req, res) => {
  const user = await prisma.user.create({
    data: req.body,
  });

  res.status(201).json(user);
});
```

## NestJS でのエラーハンドリング

### カスタム例外フィルター

```typescript
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message = 'Internal server error';

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const exceptionResponse = exception.getResponse();

      message =
        typeof exceptionResponse === 'string'
          ? exceptionResponse
          : (exceptionResponse as any).message;
    }

    // ログ出力
    console.error('Exception:', {
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      method: request.method,
      message,
    });

    response.status(status).json({
      error: {
        statusCode: status,
        message,
        timestamp: new Date().toISOString(),
        path: request.url,
      },
    });
  }
}

// main.ts で登録
app.useGlobalFilters(new AllExceptionsFilter());
```

### ビジネスロジックでの例外

```typescript
import { Injectable, NotFoundException, ConflictException } from '@nestjs/common';

@Injectable()
export class UsersService {
  async findOne(id: string): Promise<User> {
    const user = await this.prisma.user.findUnique({
      where: { id },
    });

    if (!user) {
      throw new NotFoundException(`User with ID ${id} not found`);
    }

    return user;
  }

  async create(createUserDto: CreateUserDto): Promise<User> {
    const existingUser = await this.prisma.user.findUnique({
      where: { email: createUserDto.email },
    });

    if (existingUser) {
      throw new ConflictException('Email already exists');
    }

    return this.prisma.user.create({
      data: createUserDto,
    });
  }
}
```

## データベースエラーのハンドリング

### Prisma エラーの処理

```typescript
import { Prisma } from '@prisma/client';

async function handlePrismaError(error: unknown): never {
  if (error instanceof Prisma.PrismaClientKnownRequestError) {
    switch (error.code) {
      case 'P2002':
        // Unique constraint violation
        throw new ConflictError(
          `Unique constraint failed on ${error.meta?.target}`
        );

      case 'P2025':
        // Record not found
        throw new NotFoundError('Record');

      case 'P2003':
        // Foreign key constraint failed
        throw new BadRequestError('Related record not found');

      default:
        throw new InternalServerError('Database operation failed');
    }
  }

  if (error instanceof Prisma.PrismaClientValidationError) {
    throw new BadRequestError('Invalid data provided');
  }

  throw error;
}

// 使用例
app.post('/api/users', async (req, res) => {
  try {
    const user = await prisma.user.create({
      data: req.body,
    });
    res.status(201).json(user);
  } catch (error) {
    handlePrismaError(error);
  }
});
```

## 外部API呼び出しのエラーハンドリング

### リトライロジック

```typescript
async function fetchWithRetry<T>(
  url: string,
  options: RequestInit = {},
  maxRetries: number = 3
): Promise<T> {
  let lastError: Error;

  for (let i = 0; i < maxRetries; i++) {
    try {
      const response = await fetch(url, options);

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      return await response.json();
    } catch (error) {
      lastError = error;
      console.log(`Retry ${i + 1}/${maxRetries} failed:`, error.message);

      // 最終試行でなければ待機
      if (i < maxRetries - 1) {
        await new Promise((resolve) => setTimeout(resolve, 1000 * (i + 1)));
      }
    }
  }

  throw new Error(`Failed after ${maxRetries} retries: ${lastError.message}`);
}

// 使用例
app.get('/api/external-data', async (req, res) => {
  try {
    const data = await fetchWithRetry('https://api.example.com/data');
    res.json(data);
  } catch (error) {
    throw new InternalServerError('Failed to fetch external data');
  }
});
```

### タイムアウト処理

```typescript
async function fetchWithTimeout<T>(
  url: string,
  timeout: number = 5000
): Promise<T> {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeout);

  try {
    const response = await fetch(url, {
      signal: controller.signal,
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    return await response.json();
  } catch (error) {
    if (error.name === 'AbortError') {
      throw new Error('Request timeout');
    }
    throw error;
  } finally {
    clearTimeout(timeoutId);
  }
}
```

## 未処理のエラーをキャッチ

### プロセスレベルのエラーハンドリング

```typescript
// 未処理の Promise 拒否
process.on('unhandledRejection', (reason: Error, promise: Promise<any>) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);

  // ログ送信
  sendErrorLog({
    type: 'unhandledRejection',
    error: reason,
    stack: reason.stack,
  });

  // グレースフルシャットダウン
  gracefulShutdown();
});

// 未処理の例外
process.on('uncaughtException', (error: Error) => {
  console.error('Uncaught Exception:', error);

  // ログ送信
  sendErrorLog({
    type: 'uncaughtException',
    error: error,
    stack: error.stack,
  });

  // プロセスを終了（必須）
  process.exit(1);
});

// グレースフルシャットダウン
async function gracefulShutdown() {
  console.log('Starting graceful shutdown...');

  // 新しいリクエストを受け付けない
  server.close(async () => {
    console.log('Closed server');

    // DB接続を閉じる
    await prisma.$disconnect();

    process.exit(1);
  });

  // 10秒後に強制終了
  setTimeout(() => {
    console.error('Forced shutdown');
    process.exit(1);
  }, 10000);
}
```

## エラーログとモニタリング

### 構造化ログ

```typescript
interface ErrorLog {
  timestamp: string;
  level: 'error' | 'warn' | 'info';
  message: string;
  error?: {
    name: string;
    message: string;
    stack?: string;
  };
  context?: {
    userId?: string;
    requestId?: string;
    url?: string;
    method?: string;
  };
}

function logError(error: Error, context?: any): void {
  const log: ErrorLog = {
    timestamp: new Date().toISOString(),
    level: 'error',
    message: error.message,
    error: {
      name: error.name,
      message: error.message,
      stack: error.stack,
    },
    context,
  };

  // 本番環境: 外部サービスに送信
  if (process.env.NODE_ENV === 'production') {
    sendToSentry(log);
  }

  // ローカル: コンソール出力
  console.error(JSON.stringify(log, null, 2));
}
```

### Sentry 統合

```typescript
import * as Sentry from '@sentry/node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 1.0,
});

// Express統合
app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.tracingHandler());

// エラーハンドラーの前に配置
app.use(Sentry.Handlers.errorHandler());

// カスタムコンテキスト
app.use((req, res, next) => {
  Sentry.setContext('user', {
    id: req.user?.id,
    email: req.user?.email,
  });
  next();
});
```

## リソースクリーンアップ

### try-finally パターン

```typescript
async function processFile(filePath: string): Promise<void> {
  let fileHandle: FileHandle | null = null;

  try {
    fileHandle = await fs.promises.open(filePath, 'r');
    const data = await fileHandle.readFile('utf-8');
    await processData(data);
  } catch (error) {
    console.error('Error processing file:', error);
    throw error;
  } finally {
    // 必ずリソースを解放
    if (fileHandle) {
      await fileHandle.close();
    }
  }
}
```

### using (Explicit Resource Management)

```typescript
// TypeScript 5.2+ の using キーワード
async function processFileModern(filePath: string): Promise<void> {
  await using file = await fs.promises.open(filePath, 'r');
  const data = await file.readFile('utf-8');
  await processData(data);
  // file は自動的に close される
}
```

## ユーザーフレンドリーなエラーメッセージ

### 本番環境でのエラー表示

```typescript
function formatErrorForUser(error: Error): string {
  // 本番環境: 詳細を隠す
  if (process.env.NODE_ENV === 'production') {
    if (error instanceof AppError && error.isOperational) {
      return error.message; // ユーザー向けメッセージ
    }
    return 'An unexpected error occurred. Please try again later.';
  }

  // 開発環境: 詳細を表示
  return error.message;
}

app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  const statusCode = err instanceof AppError ? err.statusCode : 500;

  res.status(statusCode).json({
    error: {
      message: formatErrorForUser(err),
      ...(process.env.NODE_ENV === 'development' && {
        stack: err.stack,
        details: err,
      }),
    },
  });
});
```

## サーキットブレーカーパターン

### 外部サービス保護

```typescript
class CircuitBreaker {
  private failureCount = 0;
  private lastFailureTime?: number;
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';

  constructor(
    private threshold: number = 5,
    private timeout: number = 60000
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime! > this.timeout) {
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }

  private onFailure(): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();

    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN';
    }
  }
}

// 使用例
const breaker = new CircuitBreaker();

app.get('/api/external', async (req, res) => {
  try {
    const data = await breaker.execute(() =>
      fetch('https://api.example.com/data')
    );
    res.json(data);
  } catch (error) {
    throw new InternalServerError('External service unavailable');
  }
});
```

## まとめ

エラーハンドリングのベストプラクティス:

- ✅ カスタムエラークラスで明確な分類
- ✅ グローバルエラーハンドラーで一元管理
- ✅ 非同期エラーを確実にキャッチ
- ✅ バリデーションエラーは詳細を返す
- ✅ データベースエラーを適切に変換
- ✅ リトライとタイムアウトを実装
- ✅ 未処理エラーをプロセスレベルでキャッチ
- ✅ 構造化ログで分析可能に
- ✅ 本番環境では詳細を隠す
- ✅ サーキットブレーカーで外部依存を保護
- ❌ エラーを握りつぶさない
- ❌ スタックトレースを本番で露出しない

次の章では、ロギングとモニタリングについて詳しく学びます。
