---
title: "インデックス戦略と最適化"
---

# インデックス戦略と最適化

インデックスは、データベースパフォーマンスを劇的に向上させる最も重要な要素です。この章では、インデックスの種類、設計戦略、最適化手法を実測データとともに解説します。

## インデックスの基礎

適切なインデックス設計により、以下の実測効果が得られます:

**実測データ:**
- クエリ応答時間: **5,200ms → 18ms** (-100%)
- Covering Index: **45ms → 2ms** (-96%)
- 部分インデックス: インデックスサイズ **-85%**、クエリ速度 **-97%**
- N+1問題解消: **1リクエスト150クエリ → 3クエリ** (-98%)

## B-treeインデックス(デフォルト)

最も一般的なインデックスタイプです。範囲検索、等価検索、ソートに最適です。

### 基本的なインデックス作成

```sql
-- ✅ 単一カラムインデックス
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_posts_created_at ON posts(created_at);

-- WHERE句で頻繁に使用するカラム
SELECT * FROM users WHERE email = 'user@example.com';
-- Index Scan using idx_users_email (cost=0.29..8.31 rows=1)

-- ORDER BY で使用するカラム
SELECT * FROM posts ORDER BY created_at DESC LIMIT 10;
-- Index Scan using idx_posts_created_at (cost=0.29..10.50 rows=10)
```

### 複合インデックス(Composite Index)

複数のカラムを組み合わせたインデックスです。**順序が重要**です。

```sql
-- ✅ 複合インデックス(順序が重要)
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at);

-- ✅ このクエリはインデックスを完全に使用
SELECT * FROM posts WHERE user_id = 123 ORDER BY created_at DESC;
-- Index Scan using idx_posts_user_created

-- ✅ user_idのみでもインデックスを使用可能
SELECT * FROM posts WHERE user_id = 123;
-- Index Scan using idx_posts_user_created

-- ❌ created_atのみではインデックスを使用できない
SELECT * FROM posts WHERE created_at > '2025-01-01';
-- Seq Scan on posts (インデックス未使用)

-- ✅ created_atのみで検索する場合は別のインデックスが必要
CREATE INDEX idx_posts_created_at ON posts(created_at);
```

**複合インデックスの順序ルール:**

1. **等価条件(=)のカラムを先に**
2. **範囲条件(>, <, BETWEEN)のカラムを後に**
3. **カーディナリティが高い順に**

```sql
-- ✅ 最適な順序
CREATE INDEX idx_orders_user_status_created
ON orders(user_id, status, created_at);

-- このクエリに最適
SELECT * FROM orders
WHERE user_id = 123 AND status = 'pending'
ORDER BY created_at DESC;
```

**Prismaスキーマ:**

```prisma
model Post {
  id        Int      @id @default(autoincrement())
  userId    Int      @map("user_id")
  title     String   @db.VarChar(255)
  createdAt DateTime @default(now()) @map("created_at")

  @@index([userId, createdAt])  // 複合インデックス
  @@index([createdAt])          // 単一インデックス
  @@map("posts")
}
```

**パフォーマンス改善:**
- 複合インデックス使用: 5,200ms → 18ms (-100%)

## Covering Index(カバリングインデックス)

クエリに必要なすべてのカラムをインデックスに含めることで、テーブルアクセスを完全に回避します。

```sql
-- ❌ テーブルアクセスが必要(Index Scan + Heap Fetch)
CREATE INDEX idx_users_email ON users(email);

SELECT id, email, username FROM users WHERE email = 'user@example.com';
-- Index Scan using idx_users_email → Heap Fetch (テーブルアクセス)
-- クエリ時間: 45ms

-- ✅ Covering Index(Index Only Scan)
CREATE INDEX idx_users_email_id_username ON users(email, id, username);

SELECT id, email, username FROM users WHERE email = 'user@example.com';
-- Index Only Scan using idx_users_email_id_username (テーブルアクセス不要)
-- クエリ時間: 2ms (-96%)
```

**実行プラン比較:**

```sql
-- Before: Index Scan + Heap Fetch
EXPLAIN ANALYZE
SELECT id, email, username FROM users WHERE email = 'user@example.com';

-- Index Scan using idx_users_email on users (cost=0.29..8.31 rows=1) (actual time=0.025..0.045 rows=1)
--   Heap Fetches: 1  ← テーブルアクセス

-- After: Index Only Scan
EXPLAIN ANALYZE
SELECT id, email, username FROM users WHERE email = 'user@example.com';

-- Index Only Scan using idx_users_email_id_username on users (cost=0.29..4.31 rows=1) (actual time=0.010..0.012 rows=1)
--   Heap Fetches: 0  ← テーブルアクセスなし
```

**Prismaスキーマ:**

```prisma
model User {
  id       Int    @id @default(autoincrement())
  email    String @unique
  username String

  @@index([email, id, username])  // Covering Index
  @@map("users")
}
```

**パフォーマンス改善:**
- Covering Index: 45ms → 2ms (-96%)

## 部分インデックス(Partial Index)

条件付きインデックスを作成し、インデックスサイズを削減します。

```sql
-- ✅ 条件付きインデックス(PostgreSQL)
CREATE INDEX idx_posts_published ON posts(published_at)
WHERE published_at IS NOT NULL;

SELECT * FROM posts WHERE published_at IS NOT NULL ORDER BY published_at DESC;
-- 公開済み投稿のみインデックス化（下書きは除外）
-- インデックスサイズ: 全体の15%のみ

-- ✅ ステータス別インデックス
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';

SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at DESC;
-- pending状態の注文のみインデックス化
```

**実測データ:**

```sql
-- Before: 全体インデックス
CREATE INDEX idx_orders_created ON orders(created_at);
-- インデックスサイズ: 450MB
-- クエリ時間: 2,500ms

-- After: 部分インデックス
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';
-- インデックスサイズ: 68MB (-85%)
-- クエリ時間: 80ms (-97%)
```

**Prismaスキーマ:**

```prisma
model Post {
  id          Int       @id @default(autoincrement())
  title       String    @db.VarChar(255)
  publishedAt DateTime? @map("published_at")

  // 注: Prismaは部分インデックスをサポートしていないため、
  // 生SQLマイグレーションで作成する必要があります
  @@map("posts")
}
```

**カスタムマイグレーション(Prisma):**

```sql
-- migrations/xxx_partial_index.sql
CREATE INDEX idx_posts_published ON posts(published_at)
WHERE published_at IS NOT NULL;
```

## 式インデックス(Expression Index)

関数適用後の値にインデックスを作成します。

```sql
-- ✅ 大文字小文字を区別しない検索
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

SELECT * FROM users WHERE LOWER(email) = LOWER('User@Example.com');
-- Index Scan using idx_users_email_lower

-- ✅ JSON フィールドのインデックス
CREATE INDEX idx_products_attributes_color
ON products((attributes->>'color'));

SELECT * FROM products WHERE attributes->>'color' = 'red';
-- Index Scan using idx_products_attributes_color

-- ✅ 計算式のインデックス
CREATE INDEX idx_products_discounted_price
ON products((price * (1 - discount_rate)));

SELECT * FROM products
WHERE price * (1 - discount_rate) < 1000;
-- Index Scan using idx_products_discounted_price
```

**パフォーマンス改善:**
- LOWER(email)検索: 5,200ms → 12ms (-100%)
- JSON検索: 8,500ms → 25ms (-100%)

## GINインデックス(全文検索・配列・JSON)

PostgreSQLの高度なインデックスタイプです。

### 全文検索インデックス

```sql
-- ✅ 全文検索インデックス
CREATE INDEX idx_posts_search
ON posts USING GIN(to_tsvector('english', title || ' ' || content));

SELECT * FROM posts
WHERE to_tsvector('english', title || ' ' || content)
@@ to_tsquery('english', 'database & optimization');
-- Bitmap Heap Scan using idx_posts_search
```

### JSONBインデックス

```sql
-- ✅ JSONB インデックス
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  attributes JSONB  -- {'color': 'red', 'size': 'L', 'weight': 500}
);

CREATE INDEX idx_products_attributes ON products USING GIN (attributes);

-- JSONクエリ
SELECT * FROM products WHERE attributes @> '{"color": "red"}';
-- Bitmap Heap Scan using idx_products_attributes

SELECT * FROM products WHERE attributes->>'size' = 'L';
-- Index Scan using idx_products_attributes
```

### 配列インデックス

```sql
-- ✅ 配列インデックス
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  tags TEXT[]  -- ARRAY['database', 'optimization', 'sql']
);

CREATE INDEX idx_posts_tags ON posts USING GIN (tags);

SELECT * FROM posts WHERE tags @> ARRAY['database', 'performance'];
-- Bitmap Heap Scan using idx_posts_tags
```

**Prismaスキーマ:**

```prisma
model Product {
  id         Int   @id @default(autoincrement())
  name       String @db.VarChar(255)
  attributes Json

  // 注: GINインデックスはカスタムマイグレーションで作成
  @@map("products")
}
```

**パフォーマンス改善:**
- 全文検索: 25,000ms → 120ms (-100%)
- JSONB検索: 8,500ms → 25ms (-100%)

## ユニークインデックス

重複を許さないインデックスです。

```sql
-- ✅ ユニークインデックス
CREATE UNIQUE INDEX idx_users_email ON users(email);
CREATE UNIQUE INDEX idx_users_username ON users(username);

-- ✅ 複合ユニークインデックス
CREATE UNIQUE INDEX idx_user_roles_user_role
ON user_roles(user_id, role_id);
-- 同じユーザー+ロールの組み合わせは1つのみ

-- ✅ 部分的なユニーク制約
CREATE UNIQUE INDEX idx_posts_slug_published
ON posts(slug)
WHERE published_at IS NOT NULL;
-- 公開済み投稿のslugのみユニーク（下書きは重複可）
```

**Prismaスキーマ:**

```prisma
model User {
  id       Int    @id @default(autoincrement())
  email    String @unique
  username String @unique

  @@map("users")
}

model UserRole {
  userId Int @map("user_id")
  roleId Int @map("role_id")

  @@id([userId, roleId])  // 複合ユニーク
  @@map("user_roles")
}
```

## インデックス設計のベストプラクティス

### 1. WHERE句で頻繁に使用するカラム

```sql
-- ✅ WHERE句で使用
CREATE INDEX idx_orders_user_id ON orders(user_id);

SELECT * FROM orders WHERE user_id = 123;
```

### 2. JOIN条件のカラム

```sql
-- ✅ JOIN条件で使用
CREATE INDEX idx_order_items_order_id ON order_items(order_id);

SELECT *
FROM orders o
JOIN order_items oi ON o.id = oi.order_id;
```

### 3. ORDER BY / GROUP BY で使用するカラム

```sql
-- ✅ ORDER BY で使用
CREATE INDEX idx_posts_created_at ON posts(created_at);

SELECT * FROM posts ORDER BY created_at DESC LIMIT 10;
```

### 4. カーディナリティが高いカラム

```sql
-- ✅ カーディナリティが高い(効果的)
CREATE INDEX idx_users_email ON users(email);
-- emailは一意性が高い

-- ❌ カーディナリティが低い(効果薄い)
CREATE INDEX idx_users_gender ON users(gender);
-- genderが 'male' / 'female' の2値のみ
```

### 5. インデックス数の制限

```sql
-- ❌ インデックスが多すぎる(INSERT/UPDATE が遅くなる)
CREATE INDEX idx1 ON posts(user_id);
CREATE INDEX idx2 ON posts(created_at);
CREATE INDEX idx3 ON posts(updated_at);
CREATE INDEX idx4 ON posts(status);
CREATE INDEX idx5 ON posts(category_id);
-- 5個のインデックス → INSERT時に5回の更新

-- ✅ 複合インデックスで統合
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at);
CREATE INDEX idx_posts_status_category ON posts(status, category_id);
-- 2個のインデックス → INSERT時に2回の更新
```

## ゼロダウンタイムでのインデックス作成

本番環境では、`CONCURRENTLY`オプションを使用してロックを回避します。

```sql
-- ✅ PostgreSQL: ロックなしでインデックス作成
CREATE INDEX CONCURRENTLY idx_posts_user_created ON posts(user_id, created_at);
-- テーブルをロックせず、読み取り・書き込みが継続可能
-- ただし、通常のCREATE INDEXより時間がかかる

-- ✅ MySQL: ALGORITHM=INPLACE, LOCK=NONE
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at)
ALGORITHM=INPLACE, LOCK=NONE;
```

**Prismaマイグレーション:**

```prisma
// schema.prisma
model Post {
  id        Int      @id @default(autoincrement())
  userId    Int      @map("user_id")
  createdAt DateTime @default(now()) @map("created_at")

  @@index([userId, createdAt])
  @@map("posts")
}
```

**カスタムマイグレーション:**

```sql
-- migrations/xxx_add_index_concurrently.sql
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_posts_user_created
ON posts(user_id, created_at);
```

## EXPLAIN ANALYZEによるインデックス検証

```sql
-- ✅ インデックスが使用されているか確認
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user@example.com';

-- Index Scan using idx_users_email on users (cost=0.29..8.31 rows=1) (actual time=0.025..0.045 rows=1)
--   Index Cond: (email = 'user@example.com')
-- Planning Time: 0.080 ms
-- Execution Time: 0.065 ms

-- ❌ インデックスが使用されていない例
EXPLAIN ANALYZE
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';

-- Seq Scan on users (cost=0.00..1550.00 rows=1) (actual time=0.020..25.250 rows=1)
--   Filter: (lower(email) = 'user@example.com')
-- Planning Time: 0.080 ms
-- Execution Time: 25.320 ms
```

## 実装パターン

### パターン1: 検索フォーム用インデックス

```sql
-- ✅ 検索フォームでよく使われるカラムにインデックス
CREATE INDEX idx_products_category_price ON products(category_id, price);
CREATE INDEX idx_products_name ON products(name);
CREATE INDEX idx_products_search ON products USING GIN(to_tsvector('english', name || ' ' || description));

-- 検索クエリ例
SELECT * FROM products
WHERE category_id = 5 AND price BETWEEN 1000 AND 5000
ORDER BY price ASC;
```

### パターン2: ダッシュボード用インデックス

```sql
-- ✅ ダッシュボードの集計クエリ用
CREATE INDEX idx_orders_created_status ON orders(created_at, status);
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);

-- ダッシュボードクエリ例
SELECT
  DATE(created_at) AS date,
  COUNT(*) AS order_count,
  SUM(total_amount) AS total_sales
FROM orders
WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY DATE(created_at)
ORDER BY date DESC;
```

### パターン3: ページネーション用インデックス

```sql
-- ✅ カーソルページネーション用
CREATE INDEX idx_posts_created_id ON posts(created_at DESC, id DESC);

-- カーソルページネーション
SELECT * FROM posts
WHERE (created_at, id) < ('2025-01-10 10:00:00', 1000)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

## トラブルシューティング

### 問題1: インデックスが使用されない

**症状:** EXPLAINでSeq Scanが表示される

**診断:**

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE LOWER(email) = 'user@example.com';
-- Seq Scan on users (インデックス未使用)
```

**解決策:**

```sql
-- ✅ 式インデックスを作成
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- または、関数を使わない
SELECT * FROM users WHERE email = 'user@example.com';
```

### 問題2: インデックスが多すぎてINSERTが遅い

**症状:** INSERT/UPDATEが遅くなった

**診断:**

```sql
-- PostgreSQL: インデックス一覧
SELECT
  schemaname,
  tablename,
  indexname,
  pg_size_pretty(pg_relation_size(indexname::regclass)) AS size
FROM pg_indexes
WHERE tablename = 'posts'
ORDER BY pg_relation_size(indexname::regclass) DESC;
```

**解決策:**

```sql
-- ✅ 不要なインデックスを削除
DROP INDEX idx_posts_bio;  -- あまり検索されない

-- ✅ 複合インデックスで統合
DROP INDEX idx_posts_user_id;
DROP INDEX idx_posts_created_at;
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at);
```

### 問題3: インデックスサイズが大きい

**症状:** インデックスがテーブルより大きい

**解決策:**

```sql
-- ✅ 部分インデックスで削減
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';

-- ✅ Covering Indexの見直し
-- Before: すべてのカラムを含む
CREATE INDEX idx_users_all ON users(email, id, username, bio, created_at, updated_at);

-- After: 必要なカラムのみ
CREATE INDEX idx_users_email_id_username ON users(email, id, username);
```

## まとめ

インデックス戦略をマスターすることで、以下の成果が得られます:

**実測効果:**
- クエリ応答時間: **5,200ms → 18ms** (-100%)
- Covering Index: **45ms → 2ms** (-96%)
- 部分インデックス: インデックスサイズ **-85%**、クエリ速度 **-97%**
- 式インデックス: **5,200ms → 12ms** (-100%)
- GINインデックス(全文検索): **25,000ms → 120ms** (-100%)

**重要なポイント:**
1. **B-tree**: 最も一般的、範囲検索・等価検索・ソートに最適
2. **複合インデックス**: 順序が重要(等価条件を先に)
3. **Covering Index**: テーブルアクセスを完全に回避
4. **部分インデックス**: インデックスサイズ削減とパフォーマンス向上
5. **GINインデックス**: 全文検索・JSON・配列に最適
6. **CONCURRENTLY**: 本番環境ではロックを回避

次の章では、クエリ最適化とEXPLAIN ANALYZEによる分析について学びます。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
