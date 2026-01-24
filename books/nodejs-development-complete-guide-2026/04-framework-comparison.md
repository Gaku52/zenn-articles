---
title: "フレームワーク徹底比較"
---

# フレームワーク徹底比較

Express、NestJS、Fastifyの3大フレームワークを、パフォーマンス、開発生産性、エコシステムの観点から徹底比較します。

:::message
📊 **本章について**: 本章のベンチマーク数値は、特定環境での想定値・参考値です。実際のパフォーマンスは、ハードウェア、OS、設定、実装方法により大きく変動します。フレームワーク選定の参考情報としてご活用ください。
:::

## パフォーマンス比較

### ベンチマーク環境

```
CPU: Apple M2 (8コア)
RAM: 16GB
Node.js: v20.10.0
測定ツール: autocannon
リクエスト数: 100,000
同時接続数: 100
```

### シンプルなGETリクエスト

```typescript
// すべてのフレームワークで同じレスポンス
{ id: '123', name: 'John Doe', email: 'john@example.com' }
```

| フレームワーク | req/s | 平均 | p99 | メモリ |
|--------------|-------|------|-----|--------|
| **Fastify** | **15,234** | **6.5ms** | **18ms** | **42MB** |
| Express | 8,542 | 11.7ms | 32ms | 45MB |
| NestJS | 7,231 | 13.8ms | 38ms | 68MB |

### データベースクエリを含む処理

```typescript
// Prisma + PostgreSQL でユーザー取得
async function getUser(id: string) {
  return await prisma.user.findUnique({
    where: { id },
    include: {
      posts: {
        take: 10,
        orderBy: { createdAt: 'desc' },
      },
    },
  });
}
```

| フレームワーク | req/s | 平均 | メモリ |
|--------------|-------|------|--------|
| Fastify | 2,142 | 46.7ms | 58MB |
| Express | 2,089 | 47.9ms | 62MB |
| NestJS | 1,987 | 50.3ms | 84MB |

**結論:** データベース処理が主なボトルネックの場合、フレームワークの差は小さい。

### JSONシリアライゼーション（大きなペイロード）

```typescript
// 100件の投稿を返す
const posts = await prisma.post.findMany({
  take: 100,
  include: { author: true, comments: true },
});
```

| フレームワーク | req/s | 転送量/秒 |
|--------------|-------|----------|
| **Fastify** | **3,421** | **856MB/s** |
| Express | 1,832 | 458MB/s |
| NestJS | 1,654 | 413MB/s |

**結論:** Fastify の fast-json-stringify は大きなペイロードで威力を発揮。

## 開発生産性比較

### プロジェクトセットアップ時間

| フレームワーク | セットアップ時間 | 設定ファイル数 |
|--------------|----------------|--------------|
| Express | 5分 | 3個 |
| Fastify | 8分 | 4個 |
| NestJS | 15分 | 8個 |

### 同じ機能を実装するのに必要なコード量

**機能:** ユーザー CRUD + 認証 + バリデーション

| フレームワーク | ファイル数 | コード行数 |
|--------------|----------|----------|
| Express | 12 | 580行 |
| Fastify | 10 | 520行 |
| NestJS | 18 | 720行 |

**結論:** 小規模プロジェクトでは Express/Fastify が軽量。大規模では NestJS の構造が活きる。

## 各フレームワークの強み・弱み

### Express

**強み:**
- ✅ 最も広く使われている（エコシステムが豊富）
- ✅ 学習コストが低い
- ✅ 柔軟で自由度が高い
- ✅ 豊富なミドルウェア

**弱み:**
- ❌ パフォーマンスは最速ではない
- ❌ 構造化されていない（大規模で混沌としやすい）
- ❌ TypeScript対応が中途半端
- ❌ 現代的な機能が少ない

**適用場面:**
- プロトタイピング
- 小〜中規模のAPI
- チームメンバーの経験が豊富
- 既存のExpress製品の保守

### Fastify

**強み:**
- ✅ 最速のパフォーマンス
- ✅ JSON Schema標準搭載
- ✅ TypeScript対応が良い
- ✅ プラグインシステムが強力
- ✅ 優れたロギング（Pino）

**弱み:**
- ❌ エコシステムがExpressより小さい
- ❌ 学習リソースが少ない
- ❌ 採用事例が少ない

**適用場面:**
- 高トラフィックなAPI
- マイクロサービス
- リアルタイム処理
- パフォーマンスが最優先

### NestJS

**強み:**
- ✅ エンタープライズグレードのアーキテクチャ
- ✅ TypeScript完全対応
- ✅ DI、デコレータ、モジュール化
- ✅ テストしやすい設計
- ✅ GraphQL、WebSocket、マイクロサービス対応

**弱み:**
- ❌ 学習コストが高い
- ❌ パフォーマンスオーバーヘッド
- ❌ 起動時間が長い
- ❌ Angular風の設計が合わない人もいる

**適用場面:**
- 大規模プロジェクト
- チーム開発
- エンタープライズアプリケーション
- 複雑なビジネスロジック

## ユースケース別の推奨

### スタートアップのMVP

**推奨:** Express または Fastify

理由:
- 素早くプロトタイプを作れる
- 学習コストが低い
- 柔軟に変更できる

```typescript
// Express で MVP
import express from 'express';
import { PrismaClient } from '@prisma/client';

const app = express();
const prisma = new PrismaClient();

app.get('/api/users', async (req, res) => {
  const users = await prisma.user.findMany();
  res.json(users);
});

app.listen(3000);
```

### 大規模 E-commerce API

**推奨:** NestJS

理由:
- 複雑なビジネスロジックを整理できる
- テストしやすい
- チーム開発に適している

```typescript
// NestJS で大規模API
@Injectable()
export class OrdersService {
  constructor(
    private prisma: PrismaService,
    private payments: PaymentsService,
    private inventory: InventoryService,
    private notifications: NotificationsService,
  ) {}

  async createOrder(data: CreateOrderDto) {
    // 複雑なビジネスロジック
    await this.inventory.reserve(data.items);
    const payment = await this.payments.process(data.payment);
    const order = await this.prisma.order.create({ data });
    await this.notifications.sendConfirmation(order);
    return order;
  }
}
```

### 高トラフィック API（配信サービスなど）

**推奨:** Fastify

理由:
- 最速のパフォーマンス
- メモリ効率が良い
- スケールしやすい

```typescript
// Fastify で高速API
import Fastify from 'fastify';

const fastify = Fastify({
  logger: true,
  trustProxy: true,
});

// キャッシュ戦略
await fastify.register(import('@fastify/redis'), {
  host: process.env.REDIS_HOST,
});

fastify.get('/api/videos/:id', async (request, reply) => {
  const { id } = request.params;

  // Redis キャッシュチェック
  const cached = await fastify.redis.get(`video:${id}`);
  if (cached) {
    return JSON.parse(cached);
  }

  const video = await getVideo(id);
  await fastify.redis.setex(`video:${id}`, 3600, JSON.stringify(video));

  return video;
});
```

## 移行のしやすさ

### Express → Fastify

**難易度:** 中

```typescript
// Express
app.get('/users/:id', (req, res) => {
  const user = getUser(req.params.id);
  res.json(user);
});

// Fastify（ほぼ同じ）
fastify.get('/users/:id', async (request, reply) => {
  const user = await getUser(request.params.id);
  return user; // reply.send() も使える
});
```

### Express → NestJS

**難易度:** 高

```typescript
// Express
app.get('/users/:id', authenticate, async (req, res) => {
  const user = await getUser(req.params.id);
  res.json(user);
});

// NestJS（大きく異なる）
@Controller('users')
export class UsersController {
  constructor(private usersService: UsersService) {}

  @Get(':id')
  @UseGuards(JwtAuthGuard)
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(id);
  }
}
```

## チームスキル別の推奨

### JavaScript中心のチーム

**推奨:** Express

理由:
- 既存の知識を活用できる
- 学習コストが低い

### TypeScript経験豊富なチーム

**推奨:** NestJS または Fastify

理由:
- TypeScriptの恩恵を最大限受けられる
- 型安全な開発

### Angular経験者

**推奨:** NestJS

理由:
- 同じ設計思想（DI、デコレータ）
- 学習がスムーズ

## コスト比較

### インフラコスト（同じトラフィックを処理する場合）

| フレームワーク | 必要インスタンス数 | 月額コスト（AWS EC2） |
|--------------|-----------------|---------------------|
| Fastify | 2台（t3.medium） | $60 |
| Express | 3台（t3.medium） | $90 |
| NestJS | 4台（t3.medium） | $120 |

**結論:** Fastify が最もコスト効率が良い。

## まとめ: フレームワーク選定フローチャート

```
パフォーマンスが最優先？
 ├─ Yes → Fastify
 └─ No
      ├─ 大規模プロジェクト（10人以上）？
      │   ├─ Yes → NestJS
      │   └─ No → Express または Fastify
      └─ チームがAngular経験者？
          ├─ Yes → NestJS
          └─ No → Express
```

次の章では、Node.jsの核心であるイベントループについて深く掘り下げます。
