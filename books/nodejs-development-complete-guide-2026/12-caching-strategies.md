---
title: "キャッシング戦略 - パフォーマンスを劇的に向上させる"
---

# キャッシング戦略 - パフォーマンスを劇的に向上させる

適切なキャッシング戦略により、データベース負荷を削減し、レスポンスタイムを大幅に改善する方法を学びます。

## キャッシングの基本原則

### キャッシュの種類

```
┌─────────────────────────────────────┐
│   クライアント                        │
│   └─ ブラウザキャッシュ                │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│   CDN / リバースプロキシ              │
│   └─ Nginx/CloudFlare キャッシュ      │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│   アプリケーション層                  │
│   ├─ メモリキャッシュ (LRU)           │
│   └─ Redis/Memcached                │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│   データベース層                      │
│   └─ クエリキャッシュ                 │
└─────────────────────────────────────┘
```

### キャッシュすべきデータの特徴

- 読み取り頻度が高い
- 更新頻度が低い
- 計算コストが高い
- データベースクエリが重い
- 外部API呼び出しが必要

## メモリキャッシュ（LRU Cache）

### node-lru-cacheの基本

```typescript
import LRU from 'lru-cache';

// 基本的な設定
const cache = new LRU<string, any>({
  max: 500, // 最大エントリ数
  ttl: 1000 * 60 * 5, // 5分でexpire
  allowStale: false, // 期限切れデータを返さない
  updateAgeOnGet: false, // 取得時にTTLをリセットしない
  updateAgeOnHas: false,
});

// データの保存
cache.set('user:123', { id: 123, name: 'John' });

// データの取得
const user = cache.get('user:123');

// キャッシュの削除
cache.delete('user:123');

// 全クリア
cache.clear();

// キャッシュサイズ
console.log(cache.size); // 現在のエントリ数
```

### 実践的なキャッシュパターン

```typescript
class UserService {
  private cache = new LRU<string, User>({
    max: 1000,
    ttl: 1000 * 60 * 10, // 10分
  });

  async getUser(id: string): Promise<User> {
    // キャッシュをチェック
    const cached = this.cache.get(`user:${id}`);
    if (cached) {
      return cached;
    }

    // キャッシュミス: DBから取得
    const user = await prisma.user.findUnique({
      where: { id },
    });

    if (user) {
      // キャッシュに保存
      this.cache.set(`user:${id}`, user);
    }

    return user;
  }

  async updateUser(id: string, data: Partial<User>): Promise<User> {
    const user = await prisma.user.update({
      where: { id },
      data,
    });

    // キャッシュを更新
    this.cache.set(`user:${id}`, user);

    return user;
  }

  async deleteUser(id: string): Promise<void> {
    await prisma.user.delete({
      where: { id },
    });

    // キャッシュを削除
    this.cache.delete(`user:${id}`);
  }
}
```

### TTLの動的設定

```typescript
class CacheService {
  private cache = new LRU<string, any>({
    max: 1000,
    // TTLを個別に設定可能
  });

  set(key: string, value: any, options?: { ttl?: number }) {
    this.cache.set(key, value, {
      ttl: options?.ttl || 1000 * 60 * 5, // デフォルト5分
    });
  }

  async getOrFetch<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttl?: number
  ): Promise<T> {
    // キャッシュチェック
    const cached = this.cache.get(key);
    if (cached !== undefined) {
      return cached;
    }

    // データ取得
    const data = await fetcher();

    // キャッシュに保存
    this.set(key, data, { ttl });

    return data;
  }
}

// 使用例
const cacheService = new CacheService();

// 頻繁に更新されるデータ: 短いTTL
const recentPosts = await cacheService.getOrFetch(
  'posts:recent',
  () => prisma.post.findMany({ take: 10, orderBy: { createdAt: 'desc' } }),
  1000 * 60 // 1分
);

// ほとんど更新されないデータ: 長いTTL
const categories = await cacheService.getOrFetch(
  'categories:all',
  () => prisma.category.findMany(),
  1000 * 60 * 60 // 1時間
);
```

## Redis キャッシュ

### Redis の基本設定

```typescript
import { createClient } from 'redis';

// Redis クライアントの作成
const redisClient = createClient({
  url: process.env.REDIS_URL || 'redis://localhost:6379',
  socket: {
    reconnectStrategy: (retries) => {
      if (retries > 10) {
        return new Error('Redis connection failed');
      }
      return Math.min(retries * 100, 3000);
    },
  },
});

redisClient.on('error', (err) => console.error('Redis Error:', err));
redisClient.on('connect', () => console.log('Redis connected'));
redisClient.on('reconnecting', () => console.log('Redis reconnecting'));

await redisClient.connect();

// 基本操作
await redisClient.set('key', 'value');
await redisClient.setEx('key', 3600, 'value'); // 1時間後に期限切れ

const value = await redisClient.get('key');

await redisClient.del('key');
```

### Redis Cache Aside パターン

```typescript
class RedisCacheService {
  constructor(private redis: ReturnType<typeof createClient>) {}

  async get<T>(key: string): Promise<T | null> {
    const cached = await this.redis.get(key);
    if (!cached) return null;

    try {
      return JSON.parse(cached);
    } catch {
      return null;
    }
  }

  async set(key: string, value: any, ttl?: number): Promise<void> {
    const serialized = JSON.stringify(value);

    if (ttl) {
      await this.redis.setEx(key, ttl, serialized);
    } else {
      await this.redis.set(key, serialized);
    }
  }

  async delete(key: string): Promise<void> {
    await this.redis.del(key);
  }

  async invalidatePattern(pattern: string): Promise<void> {
    // パターンに一致するキーを全て削除
    const keys = await this.redis.keys(pattern);
    if (keys.length > 0) {
      await this.redis.del(keys);
    }
  }
}

// 使用例
class ProductService {
  constructor(private cache: RedisCacheService) {}

  async getProduct(id: string): Promise<Product> {
    const cacheKey = `product:${id}`;

    // キャッシュチェック
    const cached = await this.cache.get<Product>(cacheKey);
    if (cached) {
      return cached;
    }

    // DBから取得
    const product = await prisma.product.findUnique({
      where: { id },
      include: { category: true, reviews: true },
    });

    // キャッシュに保存（1時間）
    await this.cache.set(cacheKey, product, 3600);

    return product;
  }

  async updateProduct(id: string, data: Partial<Product>): Promise<Product> {
    const product = await prisma.product.update({
      where: { id },
      data,
    });

    // キャッシュを無効化（関連するキャッシュも削除）
    await this.cache.delete(`product:${id}`);
    await this.cache.invalidatePattern('products:list:*');

    return product;
  }
}
```

### Redis Hash を使った複雑なキャッシュ

```typescript
class UserCacheService {
  constructor(private redis: ReturnType<typeof createClient>) {}

  async cacheUser(user: User): Promise<void> {
    const key = `user:${user.id}`;

    // Hashとして保存
    await this.redis.hSet(key, {
      id: user.id,
      name: user.name,
      email: user.email,
      role: user.role,
    });

    // 1時間で期限切れ
    await this.redis.expire(key, 3600);
  }

  async getUser(id: string): Promise<User | null> {
    const key = `user:${id}`;
    const data = await this.redis.hGetAll(key);

    if (!data || Object.keys(data).length === 0) {
      return null;
    }

    return data as unknown as User;
  }

  async updateUserField(id: string, field: string, value: string): Promise<void> {
    const key = `user:${id}`;
    await this.redis.hSet(key, field, value);
  }
}
```

## キャッシュ無効化戦略

### Time-based Invalidation

```typescript
// TTLベースの自動期限切れ
cache.set('data', value, { ttl: 1000 * 60 * 5 }); // 5分後に自動削除
```

### Write-through Cache

```typescript
class WriteThroughCache {
  async updateUser(id: string, data: Partial<User>): Promise<User> {
    // DBとキャッシュを同時更新
    const [user] = await Promise.all([
      prisma.user.update({ where: { id }, data }),
      this.cache.set(`user:${id}`, { ...data, id }),
    ]);

    return user;
  }
}
```

### Cache Invalidation on Update

```typescript
class CacheInvalidationService {
  async updatePost(id: string, data: Partial<Post>): Promise<Post> {
    const post = await prisma.post.update({
      where: { id },
      data,
    });

    // 関連するキャッシュを全て無効化
    await Promise.all([
      this.cache.delete(`post:${id}`),
      this.cache.delete(`posts:recent`),
      this.cache.delete(`posts:by-category:${post.categoryId}`),
      this.cache.delete(`posts:by-author:${post.authorId}`),
    ]);

    return post;
  }
}
```

### Tag-based Invalidation

```typescript
class TaggedCache {
  private cache = new LRU<string, any>({ max: 1000 });
  private tags = new Map<string, Set<string>>();

  set(key: string, value: any, tags: string[]): void {
    this.cache.set(key, value);

    // タグとキーの関連付け
    tags.forEach((tag) => {
      if (!this.tags.has(tag)) {
        this.tags.set(tag, new Set());
      }
      this.tags.get(tag)!.add(key);
    });
  }

  invalidateTag(tag: string): void {
    const keys = this.tags.get(tag);
    if (keys) {
      keys.forEach((key) => this.cache.delete(key));
      this.tags.delete(tag);
    }
  }
}

// 使用例
const cache = new TaggedCache();

cache.set('user:123', userData, ['user', 'user:123']);
cache.set('user:123:posts', postsData, ['user', 'user:123', 'posts']);

// 特定ユーザーに関連する全キャッシュを無効化
cache.invalidateTag('user:123');
```

## HTTP キャッシュヘッダー

### Cache-Control の設定

```typescript
app.get('/api/public/posts', async (req, res) => {
  // 公開データ: 5分間キャッシュ可能
  res.set('Cache-Control', 'public, max-age=300');

  const posts = await getPublicPosts();
  res.json(posts);
});

app.get('/api/user/profile', authenticate, async (req, res) => {
  // プライベートデータ: ブラウザのみキャッシュ
  res.set('Cache-Control', 'private, max-age=60');

  const profile = await getUserProfile(req.user.id);
  res.json(profile);
});

app.get('/api/sensitive', authenticate, async (req, res) => {
  // 機密データ: キャッシュしない
  res.set('Cache-Control', 'no-store, no-cache, must-revalidate');

  const data = await getSensitiveData(req.user.id);
  res.json(data);
});
```

### ETag による条件付きリクエスト

```typescript
import crypto from 'crypto';

app.get('/api/posts/:id', async (req, res) => {
  const post = await prisma.post.findUnique({
    where: { id: req.params.id },
  });

  if (!post) {
    return res.status(404).json({ error: 'Not found' });
  }

  // ETagを生成
  const etag = crypto
    .createHash('md5')
    .update(JSON.stringify(post))
    .digest('hex');

  // If-None-Match ヘッダーをチェック
  if (req.headers['if-none-match'] === etag) {
    // データが変更されていない
    return res.status(304).end();
  }

  // ETagを設定してレスポンス
  res.set('ETag', etag);
  res.set('Cache-Control', 'private, max-age=60');
  res.json(post);
});
```

## クエリ結果のキャッシング

### Prismaのクエリキャッシュ

```typescript
class QueryCacheService {
  private cache = new LRU<string, any>({ max: 1000, ttl: 1000 * 60 * 5 });

  async cachedQuery<T>(
    cacheKey: string,
    query: () => Promise<T>,
    ttl?: number
  ): Promise<T> {
    // キャッシュチェック
    const cached = this.cache.get(cacheKey);
    if (cached !== undefined) {
      return cached;
    }

    // クエリ実行
    const result = await query();

    // キャッシュに保存
    this.cache.set(cacheKey, result, { ttl });

    return result;
  }
}

// 使用例
const queryCache = new QueryCacheService();

app.get('/api/stats', async (req, res) => {
  const stats = await queryCache.cachedQuery(
    'stats:dashboard',
    async () => {
      // 重いクエリ
      const [userCount, postCount, commentCount] = await Promise.all([
        prisma.user.count(),
        prisma.post.count(),
        prisma.comment.count(),
      ]);

      return { userCount, postCount, commentCount };
    },
    1000 * 60 * 10 // 10分
  );

  res.json(stats);
});
```

### ページネーションのキャッシング

```typescript
class PaginatedCache {
  async getPosts(page: number, pageSize: number): Promise<Post[]> {
    const cacheKey = `posts:page:${page}:size:${pageSize}`;

    return queryCache.cachedQuery(
      cacheKey,
      () =>
        prisma.post.findMany({
          skip: (page - 1) * pageSize,
          take: pageSize,
          orderBy: { createdAt: 'desc' },
        }),
      1000 * 60 // 1分
    );
  }
}
```

## CDN とリバースプロキシキャッシュ

### Nginx でのキャッシング

```nginx
# /etc/nginx/nginx.conf
http {
    # キャッシュディレクトリの設定
    proxy_cache_path /var/cache/nginx
                     levels=1:2
                     keys_zone=api_cache:10m
                     max_size=1g
                     inactive=60m
                     use_temp_path=off;

    server {
        listen 80;
        server_name api.example.com;

        location /api/public/ {
            proxy_pass http://localhost:3000;

            # キャッシュを有効化
            proxy_cache api_cache;
            proxy_cache_valid 200 5m;
            proxy_cache_valid 404 1m;
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503;

            # キャッシュキー
            proxy_cache_key "$scheme$request_method$host$request_uri";

            # キャッシュステータスをヘッダーに追加
            add_header X-Cache-Status $upstream_cache_status;
        }

        location /api/private/ {
            proxy_pass http://localhost:3000;
            # プライベートAPIはキャッシュしない
            proxy_no_cache 1;
            proxy_cache_bypass 1;
        }
    }
}
```

## キャッシュ戦略の選択

### データ特性による選択

```typescript
class CacheStrategySelector {
  selectStrategy(dataType: string): CacheConfig {
    const strategies = {
      // 静的データ: 長いTTL
      static: { ttl: 1000 * 60 * 60 * 24, layer: 'cdn' }, // 24時間

      // ユーザー固有データ: 中程度のTTL
      userSpecific: { ttl: 1000 * 60 * 5, layer: 'redis' }, // 5分

      // リアルタイムデータ: 短いTTL
      realtime: { ttl: 1000 * 10, layer: 'memory' }, // 10秒

      // 計算コストの高いデータ: 長めのTTL
      expensive: { ttl: 1000 * 60 * 30, layer: 'redis' }, // 30分

      // 頻繁に更新されるデータ: キャッシュしない
      volatile: { ttl: 0, layer: 'none' },
    };

    return strategies[dataType] || strategies.userSpecific;
  }
}
```

## まとめ

キャッシング戦略のベストプラクティス:

- ✅ LRU Cacheでメモリ効率的なキャッシング
- ✅ Redisで複数サーバー間の共有キャッシュ
- ✅ 適切なTTLを設定（データの特性による）
- ✅ キャッシュ無効化戦略を明確にする
- ✅ HTTPキャッシュヘッダーを活用
- ✅ CDN/Nginxでエッジキャッシング
- ✅ ETagで条件付きリクエストを実装
- ✅ タグベースの無効化で柔軟性を確保
- ❌ 機密データをパブリックキャッシュに保存しない
- ❌ 無期限のキャッシュを避ける

次の章では、エラーハンドリングパターンについて学びます。
