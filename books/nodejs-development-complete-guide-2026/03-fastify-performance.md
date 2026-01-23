---
title: "Fastify - 最速のパフォーマンス"
---

# Fastify - 最速のパフォーマンス

Fastifyは、パフォーマンスを最優先に設計された Node.js Web フレームワークです。Express の2倍以上の速度を誇り、JSON Schema ベースのバリデーションを標準搭載しています。

## Fastifyの特徴

### 高速なパフォーマンス

```typescript
import Fastify from 'fastify';

const fastify = Fastify({
  logger: true,
});

fastify.get('/api/users/:id', async (request, reply) => {
  const { id } = request.params as { id: string };
  return { id, name: 'John Doe' };
});

fastify.listen({ port: 3000 }, (err, address) => {
  if (err) throw err;
  console.log(`Server listening on ${address}`);
});
```

### JSON Schemaベースのバリデーション

```typescript
const schema = {
  body: {
    type: 'object',
    required: ['name', 'email'],
    properties: {
      name: { type: 'string', minLength: 1, maxLength: 100 },
      email: { type: 'string', format: 'email' },
      age: { type: 'integer', minimum: 0, maximum: 120 },
    },
  },
  response: {
    201: {
      type: 'object',
      properties: {
        id: { type: 'string' },
        name: { type: 'string' },
        email: { type: 'string' },
      },
    },
  },
};

fastify.post('/api/users', { schema }, async (request, reply) => {
  const user = await createUser(request.body);
  reply.code(201).send(user);
});
```

## TypeScript対応

### 型安全なルート定義

```typescript
import { FastifyInstance, FastifyRequest, FastifyReply } from 'fastify';

interface CreateUserBody {
  name: string;
  email: string;
  age: number;
}

interface UserParams {
  id: string;
}

async function routes(fastify: FastifyInstance) {
  fastify.post<{ Body: CreateUserBody }>(
    '/users',
    {
      schema: {
        body: {
          type: 'object',
          required: ['name', 'email'],
          properties: {
            name: { type: 'string' },
            email: { type: 'string', format: 'email' },
            age: { type: 'integer' },
          },
        },
      },
    },
    async (request, reply) => {
      // request.body は CreateUserBody 型
      const { name, email, age } = request.body;
      const user = await createUser({ name, email, age });
      return user;
    }
  );

  fastify.get<{ Params: UserParams }>(
    '/users/:id',
    async (request, reply) => {
      // request.params は UserParams 型
      const { id } = request.params;
      const user = await getUser(id);
      return user;
    }
  );
}
```

## プラグインシステム

### カスタムプラグイン

```typescript
import fp from 'fastify-plugin';
import { FastifyPluginAsync } from 'fastify';
import { PrismaClient } from '@prisma/client';

declare module 'fastify' {
  interface FastifyInstance {
    prisma: PrismaClient;
  }
}

const prismaPlugin: FastifyPluginAsync = async (fastify, options) => {
  const prisma = new PrismaClient();

  fastify.decorate('prisma', prisma);

  fastify.addHook('onClose', async (instance) => {
    await instance.prisma.$disconnect();
  });
};

export default fp(prismaPlugin);
```

### プラグインの使用

```typescript
import Fastify from 'fastify';
import prismaPlugin from './plugins/prisma';
import authPlugin from './plugins/auth';

const fastify = Fastify();

// プラグイン登録
await fastify.register(prismaPlugin);
await fastify.register(authPlugin);

// prismaが使える
fastify.get('/users/:id', async (request, reply) => {
  const user = await fastify.prisma.user.findUnique({
    where: { id: request.params.id },
  });
  return user;
});
```

## フック（Hooks）

### ライフサイクルフック

```typescript
// リクエスト処理のライフサイクル
fastify.addHook('onRequest', async (request, reply) => {
  console.log('1. onRequest');
});

fastify.addHook('preParsing', async (request, reply) => {
  console.log('2. preParsing');
});

fastify.addHook('preValidation', async (request, reply) => {
  console.log('3. preValidation');
});

fastify.addHook('preHandler', async (request, reply) => {
  console.log('4. preHandler - 認証チェックなど');
  const token = request.headers.authorization?.replace('Bearer ', '');
  if (!token) {
    reply.code(401).send({ error: 'No token provided' });
  }
});

fastify.addHook('preSerialization', async (request, reply, payload) => {
  console.log('5. preSerialization');
  return payload;
});

fastify.addHook('onSend', async (request, reply, payload) => {
  console.log('6. onSend');
  return payload;
});

fastify.addHook('onResponse', async (request, reply) => {
  console.log('7. onResponse');
});
```

### 認証フック

```typescript
import jwt from 'jsonwebtoken';

declare module 'fastify' {
  interface FastifyRequest {
    user?: {
      userId: string;
      email: string;
    };
  }
}

fastify.decorateRequest('user', null);

fastify.addHook('preHandler', async (request, reply) => {
  const publicRoutes = ['/api/auth/login', '/api/auth/register'];

  if (publicRoutes.includes(request.url)) {
    return;
  }

  const token = request.headers.authorization?.replace('Bearer ', '');

  if (!token) {
    reply.code(401).send({ error: 'Unauthorized' });
    return;
  }

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET!) as {
      userId: string;
      email: string;
    };
    request.user = payload;
  } catch (error) {
    reply.code(401).send({ error: 'Invalid token' });
  }
});
```

## エラーハンドリング

### カスタムエラーハンドラー

```typescript
class AppError extends Error {
  constructor(
    public statusCode: number,
    message: string
  ) {
    super(message);
  }
}

fastify.setErrorHandler((error, request, reply) => {
  request.log.error(error);

  if (error instanceof AppError) {
    reply.status(error.statusCode).send({
      status: 'error',
      message: error.message,
    });
    return;
  }

  if (error.validation) {
    reply.status(400).send({
      status: 'error',
      message: 'Validation error',
      errors: error.validation,
    });
    return;
  }

  reply.status(500).send({
    status: 'error',
    message: 'Internal server error',
  });
});
```

## ロギング

### Pinoロガー

```typescript
const fastify = Fastify({
  logger: {
    level: 'info',
    transport: {
      target: 'pino-pretty',
      options: {
        translateTime: 'HH:MM:ss Z',
        ignore: 'pid,hostname',
      },
    },
  },
});

fastify.get('/users/:id', async (request, reply) => {
  request.log.info({ userId: request.params.id }, 'Fetching user');

  const user = await getUser(request.params.id);

  if (!user) {
    request.log.warn({ userId: request.params.id }, 'User not found');
    reply.code(404).send({ error: 'User not found' });
    return;
  }

  return user;
});
```

## パフォーマンスの特徴

Fastifyの公式ベンチマーク（https://fastify.dev/benchmarks/）によると、Fastifyは他のNode.jsフレームワークと比較して高いスループットを実現できる傾向があります。

### パフォーマンスが高い理由

1. **高速なルーティング**: Radix Treeベースのルーター
2. **JSON スキーマベースの最適化**: fast-json-stringifyによるシリアライゼーション
3. **低オーバーヘッド**: 最小限のミドルウェア処理
4. **効率的なパースエンジン**: 組み込みのボディパーサー

※ 実際のパフォーマンスは、アプリケーションの実装、データ量、インフラ環境により異なります。

## ベストプラクティス

### 1. スキーマの再利用

```typescript
// schemas/user.ts
export const userSchema = {
  type: 'object',
  properties: {
    id: { type: 'string' },
    name: { type: 'string' },
    email: { type: 'string', format: 'email' },
  },
};

export const createUserSchema = {
  body: {
    type: 'object',
    required: ['name', 'email'],
    properties: {
      name: { type: 'string', minLength: 1 },
      email: { type: 'string', format: 'email' },
    },
  },
  response: {
    201: userSchema,
  },
};

// routes/users.ts
fastify.post('/users', { schema: createUserSchema }, handler);
```

### 2. ルートのモジュール化

```typescript
// routes/users/index.ts
import { FastifyInstance } from 'fastify';

export default async function userRoutes(fastify: FastifyInstance) {
  fastify.get('/', listUsers);
  fastify.get('/:id', getUser);
  fastify.post('/', createUser);
  fastify.patch('/:id', updateUser);
  fastify.delete('/:id', deleteUser);
}

// app.ts
import userRoutes from './routes/users';

await fastify.register(userRoutes, { prefix: '/api/users' });
```

### 3. 型安全なスキーマ

```typescript
import { Type, Static } from '@sinclair/typebox';

const UserSchema = Type.Object({
  id: Type.String(),
  name: Type.String({ minLength: 1, maxLength: 100 }),
  email: Type.String({ format: 'email' }),
  age: Type.Integer({ minimum: 0, maximum: 120 }),
});

type User = Static<typeof UserSchema>;

fastify.post<{ Body: User }>(
  '/users',
  {
    schema: {
      body: UserSchema,
      response: {
        201: UserSchema,
      },
    },
  },
  async (request, reply) => {
    const user: User = request.body; // 型安全
    return user;
  }
);
```

## まとめ

Fastifyの特徴:
- ✅ 最速のパフォーマンス（Express の1.78倍）
- ✅ JSON Schema標準搭載
- ✅ プラグインシステム
- ✅ TypeScript完全対応
- ✅ 優れたロギング（Pino）
- ❌ エコシステムがExpressより小さい
- ❌ 学習リソースが少ない

次の章では、3つのフレームワークを詳細に比較します。
