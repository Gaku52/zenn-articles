---
title: "Express基礎 - シンプルで柔軟な設計"
---

# Express基礎 - シンプルで柔軟な設計

Expressは、Node.jsで最も広く使われているWebフレームワークです。シンプルで柔軟な設計が特徴で、小規模から大規模まで幅広いプロジェクトに対応できます。

## Expressの特徴

### シンプルなAPI

```typescript
import express from 'express';

const app = express();

app.get('/api/users/:id', (req, res) => {
  const { id } = req.params;
  res.json({ id, name: 'John Doe' });
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

### ミドルウェアベースのアーキテクチャ

Expressの核心は、ミドルウェアチェーンです。

```typescript
// リクエスト処理の流れ
app.use(express.json());              // 1. JSONパース
app.use(logger);                       // 2. ロギング
app.use(authenticate);                 // 3. 認証
app.use('/api', apiRouter);           // 4. ルーティング
app.use(errorHandler);                 // 5. エラーハンドリング
```

## TypeScriptでのExpress開発

### プロジェクトセットアップ

```bash
npm init -y
npm install express
npm install -D typescript @types/express @types/node ts-node-dev

npx tsc --init
```

### 型安全なルーティング

```typescript
// types/express.d.ts
import { User } from './models';

declare global {
  namespace Express {
    interface Request {
      user?: User;
    }
  }
}

// routes/users.ts
import { Router, Request, Response } from 'express';
import { z } from 'zod';

const router = Router();

const userSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
  age: z.number().int().min(0).max(120),
});

router.post('/users', async (req: Request, res: Response) => {
  try {
    const data = userSchema.parse(req.body);
    const user = await createUser(data);
    res.status(201).json(user);
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({ errors: error.errors });
    }
    throw error;
  }
});

export default router;
```

## ミドルウェアの実装

### カスタムロガーミドルウェア

```typescript
import { Request, Response, NextFunction } from 'express';

interface LogEntry {
  timestamp: string;
  method: string;
  url: string;
  status: number;
  duration: number;
}

export function logger(req: Request, res: Response, next: NextFunction) {
  const start = Date.now();

  // レスポンス完了時にログ出力
  res.on('finish', () => {
    const duration = Date.now() - start;
    const log: LogEntry = {
      timestamp: new Date().toISOString(),
      method: req.method,
      url: req.url,
      status: res.statusCode,
      duration,
    };
    console.log(JSON.stringify(log));
  });

  next();
}
```

### 認証ミドルウェア

```typescript
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';

interface JwtPayload {
  userId: string;
  email: string;
}

export async function authenticate(
  req: Request,
  res: Response,
  next: NextFunction
) {
  try {
    const token = req.headers.authorization?.replace('Bearer ', '');

    if (!token) {
      return res.status(401).json({ error: 'No token provided' });
    }

    const payload = jwt.verify(token, process.env.JWT_SECRET!) as JwtPayload;

    const user = await prisma.user.findUnique({
      where: { id: payload.userId },
    });

    if (!user) {
      return res.status(401).json({ error: 'User not found' });
    }

    req.user = user;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
}
```

## エラーハンドリング

### グローバルエラーハンドラー

```typescript
import { Request, Response, NextFunction } from 'express';

class AppError extends Error {
  constructor(
    public statusCode: number,
    message: string,
    public isOperational = true
  ) {
    super(message);
    Object.setPrototypeOf(this, AppError.prototype);
  }
}

export function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
) {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      status: 'error',
      message: err.message,
    });
  }

  // 予期しないエラー
  console.error('ERROR 💥:', err);

  return res.status(500).json({
    status: 'error',
    message: 'Something went wrong',
  });
}

// 使用例
app.get('/api/users/:id', async (req, res, next) => {
  try {
    const user = await prisma.user.findUnique({
      where: { id: req.params.id },
    });

    if (!user) {
      throw new AppError(404, 'User not found');
    }

    res.json(user);
  } catch (error) {
    next(error);
  }
});
```

## ルーティングの設計

### モジュール化されたルーター

```typescript
// routes/index.ts
import { Router } from 'express';
import userRouter from './users';
import postRouter from './posts';
import authRouter from './auth';

const router = Router();

router.use('/auth', authRouter);
router.use('/users', userRouter);
router.use('/posts', postRouter);

export default router;

// app.ts
import express from 'express';
import routes from './routes';

const app = express();

app.use(express.json());
app.use('/api/v1', routes);
```

### RESTful API設計

```typescript
// routes/posts.ts
import { Router } from 'express';
import { authenticate } from '../middleware/auth';
import * as postController from '../controllers/posts';

const router = Router();

// 公開エンドポイント
router.get('/', postController.listPosts);
router.get('/:id', postController.getPost);

// 認証が必要なエンドポイント
router.use(authenticate);
router.post('/', postController.createPost);
router.patch('/:id', postController.updatePost);
router.delete('/:id', postController.deletePost);

export default router;
```

## ベストプラクティス

### 1. 環境変数の管理

```typescript
// config/env.ts
import { z } from 'zod';
import dotenv from 'dotenv';

dotenv.config();

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']),
  PORT: z.string().transform(Number),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  REDIS_URL: z.string().url(),
});

export const env = envSchema.parse(process.env);
```

### 2. レスポンスの標準化

```typescript
// utils/response.ts
import { Response } from 'express';

interface ApiResponse<T> {
  status: 'success' | 'error';
  data?: T;
  message?: string;
  errors?: unknown;
}

export function sendSuccess<T>(res: Response, data: T, statusCode = 200) {
  const response: ApiResponse<T> = {
    status: 'success',
    data,
  };
  return res.status(statusCode).json(response);
}

export function sendError(res: Response, message: string, statusCode = 500) {
  const response: ApiResponse<never> = {
    status: 'error',
    message,
  };
  return res.status(statusCode).json(response);
}
```

### 3. リクエストバリデーション

```typescript
import { Request, Response, NextFunction } from 'express';
import { z, ZodSchema } from 'zod';

export function validate(schema: ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      schema.parse(req.body);
      next();
    } catch (error) {
      if (error instanceof z.ZodError) {
        return res.status(400).json({
          status: 'error',
          errors: error.errors,
        });
      }
      next(error);
    }
  };
}

// 使用例
const createUserSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
});

router.post('/users', validate(createUserSchema), createUser);
```

## まとめ

Expressの特徴:
- ✅ シンプルで学習コストが低い
- ✅ 柔軟で自由度が高い
- ✅ エコシステムが豊富
- ✅ TypeScriptとの相性が良い

次の章では、より構造化されたアプローチを提供するNestJSについて学びます。
