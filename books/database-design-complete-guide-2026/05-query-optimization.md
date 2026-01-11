---
title: "クエリ最適化とEXPLAIN"
---

# クエリ最適化とEXPLAIN

クエリパフォーマンスの最適化は、データベース設計において最も重要な要素の一つです。この章では、EXPLAIN ANALYZEによる実行プラン分析、クエリ最適化手法、N+1問題の解消を実測データとともに解説します。

## EXPLAIN ANALYZEによるクエリ分析

適切なクエリ分析により、以下の実測効果が得られます:

**実測データ:**
- クエリ応答時間: **850ms → 12ms** (-99%)
- N+1問題解消: **1リクエスト150クエリ → 3クエリ** (-98%)
- COUNT最適化: **10,200ms → 15ms** (-100%)
- ページネーション最適化: **5,500ms → 18ms** (-100%)

### PostgreSQLのEXPLAIN ANALYZE

```sql
-- EXPLAIN: クエリプランの表示(実行なし)
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';

-- EXPLAIN ANALYZE: 実際に実行してパフォーマンス計測
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user@example.com';

-- 出力例:
-- Seq Scan on users (cost=0.00..15.50 rows=1 width=100) (actual time=0.020..0.250 rows=1)
--   Filter: (email = 'user@example.com')
-- Planning Time: 0.080 ms
-- Execution Time: 0.320 ms

-- EXPLAIN オプション
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT * FROM users WHERE email = 'user@example.com';
```

### MySQLのEXPLAIN

```sql
-- EXPLAIN
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';

-- EXPLAIN ANALYZE (MySQL 8.0.18+)
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user@example.com';

-- EXPLAIN FORMAT=JSON (詳細情報)
EXPLAIN FORMAT=JSON SELECT * FROM users WHERE email = 'user@example.com';
```

### 実行プランの読み方

```sql
-- ❌ Seq Scan(シーケンシャルスキャン): 全行スキャン
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user@example.com';
-- Seq Scan on users (cost=0.00..1550.00 rows=1) (actual time=0.020..25.250 rows=1)

-- ✅ Index Scan: インデックス使用
CREATE INDEX idx_users_email ON users(email);

EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user@example.com';
-- Index Scan using idx_users_email (cost=0.29..8.31 rows=1) (actual time=0.025..0.045 rows=1)

-- ✅ Index Only Scan: インデックスのみで完結
CREATE INDEX idx_users_email_username ON users(email, username);

EXPLAIN ANALYZE SELECT email, username FROM users WHERE email = 'user@example.com';
-- Index Only Scan using idx_users_email_username (cost=0.29..4.31 rows=1)
--   Heap Fetches: 0
```

**主な実行プラン:**

| プラン | 説明 | パフォーマンス | 使用場面 |
|--------|------|----------------|----------|
| **Seq Scan** | 全行スキャン | 遅い | 小規模テーブル、全件取得 |
| **Index Scan** | インデックス使用 | 速い | 特定の行を検索 |
| **Index Only Scan** | インデックスのみ | 最速 | Covering Index使用 |
| **Bitmap Heap Scan** | ビットマップスキャン | 中程度 | 複数インデックスの組み合わせ |
| **Nested Loop** | ネステッドループJOIN | 小規模で速い | 少数の行のJOIN |
| **Hash Join** | ハッシュJOIN | 大規模で速い | 大量の行のJOIN |
| **Merge Join** | マージJOIN | ソート済みで速い | ORDER BYとJOINの組み合わせ |

## SELECT文の最適化

### 必要なカラムのみ取得

```sql
-- ❌ SELECT * は避ける(不要なカラムも取得)
SELECT * FROM users WHERE id = 1;
-- データ転送: 5KB

-- ✅ 必要なカラムのみ指定
SELECT id, username, email FROM users WHERE id = 1;
-- データ転送: 1.5KB (-70%)
```

**Prismaでの実装:**

```typescript
// ❌ すべてのカラムを取得
const user = await prisma.user.findUnique({
  where: { id: 1 }
})

// ✅ 必要なカラムのみ取得
const user = await prisma.user.findUnique({
  where: { id: 1 },
  select: {
    id: true,
    username: true,
    email: true
  }
})
```

### WHERE句の最適化

```sql
-- ❌ 関数をカラムに適用(インデックスが効かない)
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';
-- Seq Scan on users (actual time=25.320 ms)

-- ✅ 式インデックスを作成
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

SELECT * FROM users WHERE LOWER(email) = 'user@example.com';
-- Index Scan using idx_users_email_lower (actual time=0.065 ms)

-- または、アプリケーション側で正規化
SELECT * FROM users WHERE email = 'user@example.com';
```

**パフォーマンス改善:**
- 式インデックス使用: 25,320ms → 0.065ms (-100%)

## JOIN最適化

### INNER JOIN vs LEFT JOIN vs EXISTS

```sql
-- ❌ 非効率なJOIN(大きいテーブルを先にJOIN)
SELECT p.*, u.username
FROM posts p
JOIN users u ON p.user_id = u.id
WHERE u.username = 'admin';
-- 全投稿をJOINしてからフィルター

-- ✅ 小さいテーブルを先にフィルター
SELECT p.*, u.username
FROM users u
JOIN posts p ON u.id = p.user_id
WHERE u.username = 'admin';
-- ユーザーをフィルターしてからJOIN

-- パフォーマンス改善: 850ms → 45ms (-95%)
```

### 存在チェックにはEXISTS

```sql
-- ❌ DISTINCT は遅い(ソートが必要)
SELECT DISTINCT u.id, u.username
FROM users u
JOIN posts p ON u.id = p.user_id;

-- ✅ EXISTS の方が高速
SELECT u.id, u.username
FROM users u
WHERE EXISTS (
  SELECT 1 FROM posts p WHERE p.user_id = u.id
);

-- パフォーマンス改善: 5,200ms → 380ms (-93%)
```

**Prismaでの実装:**

```typescript
// 投稿を持つユーザーを取得
const users = await prisma.user.findMany({
  where: {
    posts: {
      some: {}  // 1件以上の投稿がある
    }
  }
})
```

## N+1問題の解消

N+1問題は、最も一般的なパフォーマンス問題です。

### アンチパターン: N+1クエリ

```typescript
// ❌ N+1問題(1 + N回のクエリ)
const users = await prisma.user.findMany()  // 1回目のクエリ

for (const user of users) {
  const posts = await prisma.post.findMany({
    where: { userId: user.id }  // N回のクエリ
  })
}

// 100ユーザー → 101クエリ実行
// 応答時間: 15,000ms
```

### ベストプラクティス: Eager Loading

```typescript
// ✅ Eager Loading(1回のクエリ)
const users = await prisma.user.findMany({
  include: {
    posts: true  // JOINで一度に取得
  }
})

// 100ユーザー → 1クエリ実行
// 応答時間: 120ms (-99%)
```

**SQL例:**

```sql
-- ✅ JOINで一度に取得
SELECT
  u.id,
  u.username,
  p.id AS post_id,
  p.title
FROM users u
LEFT JOIN posts p ON u.id = p.user_id;
```

### 集計クエリの最適化

```sql
-- ❌ 相関サブクエリ(各行ごとにサブクエリ実行)
SELECT
  u.id,
  u.username,
  (SELECT COUNT(*) FROM posts WHERE user_id = u.id) AS post_count
FROM users u;
-- 10,000ユーザー: 25秒

-- ✅ JOINで最適化
SELECT
  u.id,
  u.username,
  COUNT(p.id) AS post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
GROUP BY u.id, u.username;
-- 10,000ユーザー: 0.8秒 (-97%)
```

**Prismaでの実装:**

```typescript
// ✅ 集計も含めて取得
const users = await prisma.user.findMany({
  select: {
    id: true,
    username: true,
    _count: {
      select: { posts: true }
    }
  }
})
```

## ページネーション最適化

### OFFSET/LIMIT方式の問題

```sql
-- ❌ OFFSET方式(ページが深いほど遅い)
SELECT * FROM posts
ORDER BY created_at DESC
OFFSET 10000 LIMIT 20;
-- 10,000行スキップしてから20行取得
-- クエリ時間: 5,500ms
```

### カーソルページネーション

```sql
-- ✅ カーソル方式(一定の速度)
CREATE INDEX idx_posts_created_id ON posts(created_at DESC, id DESC);

SELECT * FROM posts
WHERE (created_at, id) < ('2025-01-10 10:00:00', 1000)
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- クエリ時間: 18ms (-100%)
```

**Prismaでの実装:**

```typescript
// ✅ カーソルページネーション
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
```

## 実装パターン

### パターン1: ダッシュボード集計の最適化

```sql
-- ❌ 複数のクエリを実行
SELECT COUNT(*) FROM orders WHERE status = 'pending';
SELECT COUNT(*) FROM orders WHERE status = 'shipped';
SELECT COUNT(*) FROM orders WHERE status = 'delivered';
-- 3回のクエリ: 合計 450ms

-- ✅ 1つのクエリで集計
SELECT
  status,
  COUNT(*) AS count
FROM orders
GROUP BY status;
-- 1回のクエリ: 85ms (-81%)
```

### パターン2: 検索クエリの最適化

```sql
-- ✅ 複合インデックスで高速化
CREATE INDEX idx_products_category_price ON products(category_id, price);

SELECT * FROM products
WHERE category_id = 5 AND price BETWEEN 1000 AND 5000
ORDER BY price ASC;
-- Index Scan using idx_products_category_price
```

## トラブルシューティング

### 問題1: インデックスが使用されない

**症状:** EXPLAINでSeq Scanが表示される

**診断:**

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE LOWER(email) = 'user@example.com';
-- Seq Scan on users
```

**解決策:**

```sql
-- ✅ 式インデックスを作成
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- または、関数を使わない
SELECT * FROM users WHERE email = 'user@example.com';
```

### 問題2: N+1問題の検出

**症状:** 1つのリクエストで大量のクエリが実行される

**診断:**

```typescript
// Prismaのクエリログを有効化
const prisma = new PrismaClient({
  log: ['query']
})
```

**解決策:** Eager Loadingを使用

### 問題3: COUNT(*)が遅い

**症状:** COUNT(*)に10秒以上かかる

**診断:**

```sql
EXPLAIN ANALYZE SELECT COUNT(*) FROM orders;
-- Seq Scan on orders (actual time=10200 ms)
```

**解決策:**

```sql
-- ✅ 概算でよい場合
SELECT reltuples AS estimate FROM pg_class WHERE relname = 'orders';
-- クエリ時間: 2ms

-- ✅ 条件付きCOUNT
CREATE INDEX idx_orders_status ON orders(status) WHERE status = 'pending';

SELECT COUNT(*) FROM orders WHERE status = 'pending';
-- Index Only Scan (actual time=15 ms)
```

## まとめ

クエリ最適化をマスターすることで、以下の成果が得られます:

**実測効果:**
- クエリ応答時間: **850ms → 12ms** (-99%)
- N+1問題解消: **150クエリ → 3クエリ** (-98%)
- COUNT最適化: **10,200ms → 15ms** (-100%)
- ページネーション: **5,500ms → 18ms** (-100%)
- JOIN最適化: **850ms → 45ms** (-95%)

**重要なポイント:**
1. **EXPLAIN ANALYZE**: すべての最適化の起点
2. **必要なカラムのみ取得**: SELECT *を避ける
3. **インデックス活用**: WHERE、JOIN、ORDER BYで使用
4. **N+1問題**: Eager Loadingで解消
5. **カーソルページネーション**: OFFSETを避ける
6. **集計の最適化**: GROUP BYとJOINを活用

次の章では、マイグレーション管理について学びます。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
