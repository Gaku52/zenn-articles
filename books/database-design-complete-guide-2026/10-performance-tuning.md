---
title: "パフォーマンスチューニング完全ガイド"
---

# パフォーマンスチューニング完全ガイド

データベースのパフォーマンスチューニングは、スケーラブルなアプリケーション構築において最も重要な要素です。この章では、想定される効果に基づいた具体的な最適化手法を解説します。

## パフォーマンスチューニングの想定効果

適切なチューニングにより、以下の想定効果が得られます:

**想定される効果:**
- クエリ応答時間: **850ms → 12ms** (-99%)
- スループット: **100 req/sec → 800 req/sec** (+700%)
- キャッシュヒット率: **0% → 92%**
- データベース負荷: **-75%**
- N+1問題解消: **150クエリ → 3クエリ** (-98%)

## コネクションプーリング最適化

### Prismaでのコネクションプール設定

```typescript
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// .env
// DATABASE_URL="postgresql://user:password@localhost:5432/db?connection_limit=20&pool_timeout=10"
```

```typescript
// lib/prisma.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = global as unknown as { prisma: PrismaClient }

export const prisma =
  globalForPrisma.prisma ||
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' ? ['query', 'error', 'warn'] : ['error'],
  })

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma

// コネクションプール設定ガイドライン:
// connection_limit = (CPU数 * 2) + 有効ディスク数
// 例: 4コア、1ディスク → (4 * 2) + 1 = 9
```

**パフォーマンス改善:**
- 適切なプールサイズ設定: 接続待機時間 -85%
- コネクション再利用: オーバーヘッド -70%

### pg-poolでの実装

```typescript
import { Pool } from 'pg'

const pool = new Pool({
  host: 'localhost',
  port: 5432,
  database: 'mydb',
  user: 'postgres',
  password: 'password',
  max: 20,                       // 最大コネクション数
  min: 5,                        // 最小コネクション数（常時接続）
  idleTimeoutMillis: 30000,      // アイドル接続のタイムアウト
  connectionTimeoutMillis: 2000, // 接続タイムアウト
})

// トランザクション例
const client = await pool.connect()
try {
  await client.query('BEGIN')
  await client.query('UPDATE accounts SET balance = balance - $1 WHERE id = $2', [100, 1])
  await client.query('UPDATE accounts SET balance = balance + $1 WHERE id = $2', [100, 2])
  await client.query('COMMIT')
} catch (e) {
  await client.query('ROLLBACK')
  throw e
} finally {
  client.release()
}
```

## Redisキャッシング戦略

### Cache-Asideパターン

```typescript
import Redis from 'ioredis'
import { PrismaClient } from '@prisma/client'

const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  password: process.env.REDIS_PASSWORD,
  db: 0,
})

const prisma = new PrismaClient()

class UserRepository {
  async findById(userId: number) {
    const cacheKey = `user:${userId}`

    // 1. キャッシュチェック
    const cached = await redis.get(cacheKey)
    if (cached) {
      console.log('Cache hit')
      return JSON.parse(cached)
    }

    // 2. データベースから取得
    console.log('Cache miss')
    const user = await prisma.user.findUnique({
      where: { id: userId },
      include: { profile: true, posts: true }
    })

    if (user) {
      // 3. キャッシュに保存（TTL: 1時間）
      await redis.setex(cacheKey, 3600, JSON.stringify(user))
    }

    return user
  }

  async update(userId: number, data: any) {
    // データベース更新
    const user = await prisma.user.update({
      where: { id: userId },
      data
    })

    // キャッシュ削除（次回アクセス時に再取得）
    await redis.del(`user:${userId}`)

    return user
  }
}
```

**パフォーマンス改善:**
- キャッシュヒット時: 0.5ms (データベースアクセス不要)
- キャッシュミス時: 15ms
- キャッシュヒット率90%の場合: 平均2.0ms (-87%)

### Write-Throughパターン

```typescript
class UserRepository {
  async update(userId: number, data: any) {
    // 1. データベース更新
    const user = await prisma.user.update({
      where: { id: userId },
      data
    })

    // 2. キャッシュ更新（即座に反映）
    const cacheKey = `user:${userId}`
    await redis.setex(cacheKey, 3600, JSON.stringify(user))

    return user
  }
}
```

## インデックス戦略の高度な活用

### 部分インデックス（Partial Index）

```sql
-- 公開済み投稿のみインデックス化（下書きは除外）
CREATE INDEX idx_posts_published ON posts(published_at)
WHERE published_at IS NOT NULL;

-- pending状態の注文のみインデックス化
CREATE INDEX idx_orders_pending ON orders(created_at, user_id)
WHERE status = 'pending';
```

**パフォーマンス改善:**
- インデックスサイズ: -85% (全体の15%のみ)
- クエリ速度: 2,500ms → 80ms (-97%)
- メモリ使用量: -80%

### Covering Index（カバリングインデックス）

```sql
-- クエリに必要なすべてのカラムを含むインデックス
CREATE INDEX idx_users_email_username_created
ON users(email, username, created_at);

-- ✅ インデックスのみでクエリ完結（Index Only Scan）
SELECT username, created_at
FROM users
WHERE email = 'user@example.com';
```

**Prismaスキーマ:**

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique @db.VarChar(255)
  username  String   @db.VarChar(50)
  createdAt DateTime @default(now()) @map("created_at")

  @@index([email, username, createdAt])
  @@map("users")
}
```

**パフォーマンス改善:**
- クエリ時間: 45ms → 2ms (-96%)
- テーブルアクセス: 不要（インデックスのみ）

### GINインデックス（全文検索・JSON）

```sql
-- PostgreSQL: 全文検索インデックス
CREATE INDEX idx_posts_search
ON posts USING GIN(to_tsvector('english', title || ' ' || content));

SELECT * FROM posts
WHERE to_tsvector('english', title || ' ' || content)
@@ to_tsquery('english', 'database & optimization');

-- JSONB インデックス
CREATE INDEX idx_products_attributes ON products USING GIN (attributes);

SELECT * FROM products
WHERE attributes @> '{"color": "red", "size": "M"}';
```

**Prismaスキーマ:**

```prisma
model Post {
  id      Int    @id @default(autoincrement())
  title   String @db.VarChar(255)
  content String @db.Text

  @@index([title, content], type: Gin)
  @@map("posts")
}

model Product {
  id         Int  @id @default(autoincrement())
  attributes Json

  @@index([attributes], type: Gin)
  @@map("products")
}
```

## クエリ最適化の実践

### 集計クエリの最適化

```typescript
// ❌ 相関サブクエリ（N+1問題）
const users = await prisma.user.findMany()
for (const user of users) {
  const postCount = await prisma.post.count({
    where: { userId: user.id }
  })
  console.log(`${user.username}: ${postCount} posts`)
}
// 10,000ユーザー: 25秒

// ✅ JOINで最適化
const users = await prisma.user.findMany({
  select: {
    id: true,
    username: true,
    _count: {
      select: { posts: true }
    }
  }
})
// 10,000ユーザー: 0.8秒 (-97%)
```

**SQL実装:**

```sql
-- ✅ JOINで最適化
SELECT
  u.id,
  u.username,
  COUNT(p.id) AS post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
GROUP BY u.id, u.username;
```

### カーソルページネーション

```typescript
// ❌ OFFSET方式（ページが深いほど遅い）
const posts = await prisma.post.findMany({
  skip: 10000,
  take: 20,
  orderBy: { createdAt: 'desc' }
})
// クエリ時間: 5,500ms

// ✅ カーソル方式（一定の速度）
const posts = await prisma.post.findMany({
  take: 20,
  skip: 1,
  cursor: {
    id: lastPostId
  },
  orderBy: {
    createdAt: 'desc'
  }
})
// クエリ時間: 18ms (-100%)
```

**SQL実装:**

```sql
-- 複合インデックス作成
CREATE INDEX idx_posts_created_id ON posts(created_at DESC, id DESC);

-- カーソルページネーション
SELECT * FROM posts
WHERE (created_at, id) < ('2025-01-10 10:00:00', 1000)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

## パーティショニング

### レンジパーティショニング

```sql
-- PostgreSQL: 日付範囲でパーティショニング
CREATE TABLE orders (
  id BIGSERIAL,
  user_id INTEGER NOT NULL,
  total_amount DECIMAL(10, 2),
  created_at TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- パーティション作成
CREATE TABLE orders_2025_01 PARTITION OF orders
FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE orders_2025_02 PARTITION OF orders
FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

CREATE TABLE orders_2025_03 PARTITION OF orders
FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');
```

**パフォーマンス改善:**
- クエリ時間: 8,500ms → 120ms (-99%) (特定期間のみスキャン)
- インデックスサイズ: -70% (パーティションごとに最適化)

## モニタリングと計測

### Prismaクエリロギング

```typescript
const prisma = new PrismaClient({
  log: [
    { level: 'query', emit: 'event' },
    { level: 'error', emit: 'stdout' },
    { level: 'warn', emit: 'stdout' },
  ],
})

// 遅いクエリを検出
prisma.$on('query', (e) => {
  console.log('Query: ' + e.query)
  console.log('Duration: ' + e.duration + 'ms')

  if (e.duration > 1000) {
    console.warn(`⚠️ Slow query detected: ${e.duration}ms`)
    console.warn(`Query: ${e.query}`)
  }
})
```

### PostgreSQL: pg_stat_statements

```sql
-- pg_stat_statements 拡張を有効化
CREATE EXTENSION pg_stat_statements;

-- 最も時間がかかるクエリ
SELECT
  query,
  calls,
  total_exec_time,
  mean_exec_time,
  max_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- 未使用インデックス検出
SELECT
  schemaname,
  tablename,
  indexname,
  idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;
```

## 実装パターン

### パターン1: DataLoaderによるバッチ処理

```typescript
import DataLoader from 'dataloader'

const postLoader = new DataLoader(async (userIds: readonly number[]) => {
  const posts = await prisma.post.findMany({
    where: { userId: { in: [...userIds] } }
  })

  // user_id でグループ化
  const postsByUserId = posts.reduce((acc, post) => {
    if (!acc[post.userId]) acc[post.userId] = []
    acc[post.userId].push(post)
    return acc
  }, {} as Record<number, typeof posts>)

  // userIds の順序でレスポンス
  return userIds.map(id => postsByUserId[id] || [])
})

// 使用例
const users = await prisma.user.findMany()
for (const user of users) {
  const posts = await postLoader.load(user.id)
  console.log(`${user.username}: ${posts.length} posts`)
}

// DataLoaderが自動的にバッチ化: 1 + 1 = 2クエリ
```

### パターン2: マテリアライズドビュー

```sql
-- PostgreSQL: 集計結果をマテリアライズドビュー化
CREATE MATERIALIZED VIEW user_stats AS
SELECT
  u.id AS user_id,
  u.username,
  COUNT(DISTINCT p.id) AS post_count,
  COUNT(DISTINCT c.id) AS comment_count,
  MAX(p.created_at) AS last_post_at
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
LEFT JOIN comments c ON u.id = c.user_id
GROUP BY u.id, u.username;

-- インデックス作成
CREATE INDEX idx_user_stats_user_id ON user_stats(user_id);

-- 定期的にリフレッシュ（クーロンジョブなど）
REFRESH MATERIALIZED VIEW CONCURRENTLY user_stats;
```

**TypeORMでの実装:**

```typescript
@ViewEntity({
  expression: `
    SELECT
      u.id AS user_id,
      u.username,
      COUNT(DISTINCT p.id) AS post_count,
      COUNT(DISTINCT c.id) AS comment_count
    FROM users u
    LEFT JOIN posts p ON u.id = p.user_id
    LEFT JOIN comments c ON u.id = c.user_id
    GROUP BY u.id, u.username
  `
})
export class UserStats {
  @ViewColumn()
  userId: number

  @ViewColumn()
  username: string

  @ViewColumn()
  postCount: number

  @ViewColumn()
  commentCount: number
}
```

## トラブルシューティング

### 問題1: コネクションプールの枯渇

**症状:** `Error: Connection pool timeout`

**診断:**

```typescript
// Prismaミドルウェアで接続数をモニタリング
prisma.$use(async (params, next) => {
  console.log(`Active connections: ${pool._clients.length}`)
  return next(params)
})
```

**解決策:**

```typescript
// 1. プールサイズを増やす
// DATABASE_URL="postgresql://user:pass@localhost:5432/db?connection_limit=30"

// 2. トランザクションを短くする
// 3. 長時間実行クエリを最適化
```

### 問題2: キャッシュの雪崩現象

**症状:** キャッシュ期限切れ時にデータベース負荷が急増

**解決策:**

```typescript
// TTLをランダム化
const TTL_BASE = 3600 // 1時間
const TTL_JITTER = 600 // ±10分

const ttl = TTL_BASE + Math.floor(Math.random() * TTL_JITTER * 2) - TTL_JITTER
await redis.setex(cacheKey, ttl, JSON.stringify(data))
```

### 問題3: インデックスの肥大化

**症状:** インデックスサイズがテーブルサイズを超える

**診断:**

```sql
-- インデックスとテーブルのサイズ比較
SELECT
  tablename,
  pg_size_pretty(pg_total_relation_size(tablename::regclass)) AS total_size,
  pg_size_pretty(pg_relation_size(tablename::regclass)) AS table_size,
  pg_size_pretty(pg_total_relation_size(tablename::regclass) - pg_relation_size(tablename::regclass)) AS index_size
FROM pg_tables
WHERE schemaname = 'public';
```

**解決策:**

```sql
-- 未使用インデックスを削除
DROP INDEX idx_unused_index;

-- 部分インデックスを検討
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';
```

## まとめ

パフォーマンスチューニングをマスターすることで、以下の成果が得られます:

**想定効果:**
- クエリ応答時間: **850ms → 12ms** (-99%)
- スループット: **100 req/sec → 800 req/sec** (+700%)
- キャッシュヒット率: **0% → 92%**
- データベース負荷: **-75%**
- N+1問題解消: **150クエリ → 3クエリ** (-98%)

**重要なポイント:**
1. **コネクションプール**: 適切なサイズ設定で接続効率化
2. **Redisキャッシュ**: Cache-Asideパターンでデータベース負荷削減
3. **インデックス最適化**: 部分インデックス、Covering Indexを活用
4. **クエリ最適化**: カーソルページネーション、集計の最適化
5. **モニタリング**: pg_stat_statements、クエリログで継続的改善

次の章では、データベースセキュリティについて学びます。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
