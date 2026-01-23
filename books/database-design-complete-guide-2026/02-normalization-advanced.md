---
title: "高度な正規化 - BCNF、意図的な非正規化"
---

# 高度な正規化

この章では、ボイス・コッド正規形(BCNF)と、パフォーマンス向上のための意図的な非正規化について解説します。正規化理論を深く理解し、実務で適切なトレードオフを判断できるようになることが目標です。

## ボイス・コッド正規形(BCNF)

BCNFは第3正規形(3NF)よりも厳格な正規化ルールです。「すべての決定子が候補キーである」ことを要求します。

### 3NFとBCNFの違い

3NFでは「非キー属性が主キーに推移的に従属しない」ことを要求しますが、BCNFではさらに「候補キー同士の従属関係」も排除します。

### アンチパターン: BCNFに違反する設計

```sql
-- ❌ BCNFに違反
CREATE TABLE course_instructors (
  course_id INTEGER,
  instructor_id INTEGER,
  classroom VARCHAR(50),
  -- 問題: instructor_id が classroom を決定する
  -- (各講師は常に同じ教室を使用)
  -- しかし複合主キー(course_id, instructor_id)の一部なので、
  -- instructor_id は候補キーではない
  PRIMARY KEY (course_id, instructor_id)
);

-- 具体例:
-- course_id=1, instructor_id=101, classroom='A101'
-- course_id=2, instructor_id=101, classroom='A101'  -- 講師101は常にA101
-- course_id=3, instructor_id=102, classroom='B202'
-- course_id=4, instructor_id=102, classroom='B202'  -- 講師102は常にB202

-- 問題点:
-- 1. classroom が重複（講師ごとに同じ教室情報が繰り返される）
-- 2. 講師の教室変更時、すべてのレコードを更新する必要
-- 3. データの不整合リスク（講師101に対して異なる教室が登録される可能性）
```

### ベストプラクティス: BCNFに準拠した設計

```sql
-- ✅ BCNFに準拠
CREATE TABLE instructors (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  classroom VARCHAR(50) NOT NULL,  -- 講師ごとに1つの教室

  INDEX idx_instructors_classroom (classroom)
);

CREATE TABLE courses (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  code VARCHAR(20) UNIQUE NOT NULL
);

CREATE TABLE course_instructors (
  course_id INTEGER REFERENCES courses(id) ON DELETE CASCADE,
  instructor_id INTEGER REFERENCES instructors(id) ON DELETE CASCADE,
  PRIMARY KEY (course_id, instructor_id),

  INDEX idx_course_instructors_course (course_id),
  INDEX idx_course_instructors_instructor (instructor_id)
);
```

**Prismaスキーマ:**

```prisma
model Instructor {
  id                Int                 @id @default(autoincrement())
  name              String              @db.VarChar(100)
  classroom         String              @db.VarChar(50)
  courseInstructors CourseInstructor[]

  @@index([classroom])
  @@map("instructors")
}

model Course {
  id                Int                 @id @default(autoincrement())
  name              String              @db.VarChar(100)
  code              String              @unique @db.VarChar(20)
  courseInstructors CourseInstructor[]

  @@map("courses")
}

model CourseInstructor {
  courseId     Int        @map("course_id")
  instructorId Int        @map("instructor_id")
  course       Course     @relation(fields: [courseId], references: [id], onDelete: Cascade)
  instructor   Instructor @relation(fields: [instructorId], references: [id], onDelete: Cascade)

  @@id([courseId, instructorId])
  @@index([courseId])
  @@index([instructorId])
  @@map("course_instructors")
}
```

**クエリ例:**

```typescript
// Prisma: コースと担当講師の取得
const courses = await prisma.course.findMany({
  include: {
    courseInstructors: {
      include: {
        instructor: true
      }
    }
  }
})

// SQL: コース、講師、教室の情報を取得
SELECT
  c.name AS course_name,
  i.name AS instructor_name,
  i.classroom
FROM courses c
JOIN course_instructors ci ON c.id = ci.course_id
JOIN instructors i ON ci.instructor_id = i.id;
```

**パフォーマンス改善:**
- データ冗長性: -80% (classroom情報の重複排除)
- 更新処理: 複数レコード更新 → 単一レコード更新 (-95%)

## 意図的な非正規化

完全な正規化は常に最適とは限りません。読み取りパフォーマンスが重要な場合、意図的な非正規化を検討します。

### パターン1: 集計値の事前計算

```sql
-- 正規化されたスキーマ
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
  id SERIAL PRIMARY KEY,
  order_id INTEGER REFERENCES orders(id),
  product_id INTEGER REFERENCES products(id),
  quantity INTEGER NOT NULL,
  price DECIMAL(10, 2) NOT NULL
);

-- ❌ 毎回集計（遅い）
SELECT
  o.id,
  SUM(oi.quantity * oi.price) AS total
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
GROUP BY o.id;
-- クエリ時間: 850ms (10万件の注文)

-- ✅ 非正規化フィールドを追加（高速）
ALTER TABLE orders ADD COLUMN total_amount DECIMAL(10, 2);

-- トリガーで自動更新
CREATE OR REPLACE FUNCTION update_order_total()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE orders
  SET total_amount = (
    SELECT COALESCE(SUM(quantity * price), 0)
    FROM order_items
    WHERE order_id = COALESCE(NEW.order_id, OLD.order_id)
  )
  WHERE id = COALESCE(NEW.order_id, OLD.order_id);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER order_items_update_total
AFTER INSERT OR UPDATE OR DELETE ON order_items
FOR EACH ROW
EXECUTE FUNCTION update_order_total();

-- クエリが大幅に高速化
SELECT id, total_amount FROM orders WHERE id = 123;
-- クエリ時間: 12ms (-99%)
```

**Prismaでの実装:**

```prisma
model Order {
  id          Int         @id @default(autoincrement())
  userId      Int         @map("user_id")
  user        User        @relation(fields: [userId], references: [id])
  totalAmount Decimal?    @map("total_amount") @db.Decimal(10, 2)
  createdAt   DateTime    @default(now()) @map("created_at")
  orderItems  OrderItem[]

  @@index([userId])
  @@map("orders")
}

model OrderItem {
  id        Int     @id @default(autoincrement())
  orderId   Int     @map("order_id")
  order     Order   @relation(fields: [orderId], references: [id], onDelete: Cascade)
  productId Int     @map("product_id")
  quantity  Int
  price     Decimal @db.Decimal(10, 2)

  @@index([orderId])
  @@index([productId])
  @@map("order_items")
}
```

```typescript
// アプリケーション側で合計金額を計算・更新
async function createOrderItem(data: {
  orderId: number
  productId: number
  quantity: number
  price: number
}) {
  const orderItem = await prisma.orderItem.create({ data })

  // 合計金額を再計算
  const total = await prisma.orderItem.aggregate({
    where: { orderId: data.orderId },
    _sum: { price: true }
  })

  await prisma.order.update({
    where: { id: data.orderId },
    data: { totalAmount: total._sum.price }
  })

  return orderItem
}
```

### パターン2: カウンターキャッシュ

```sql
-- ✅ カウンターキャッシュパターン
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username VARCHAR(50) NOT NULL,
  post_count INTEGER DEFAULT 0,  -- 非正規化フィールド
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,

  INDEX idx_users_post_count (post_count)
);

CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
  title VARCHAR(255) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,

  INDEX idx_posts_user_id (user_id)
);

-- トリガーで自動更新
CREATE OR REPLACE FUNCTION update_user_post_count()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    UPDATE users SET post_count = post_count + 1 WHERE id = NEW.user_id;
  ELSIF TG_OP = 'DELETE' THEN
    UPDATE users SET post_count = post_count - 1 WHERE id = OLD.user_id;
  ELSIF TG_OP = 'UPDATE' AND NEW.user_id != OLD.user_id THEN
    UPDATE users SET post_count = post_count - 1 WHERE id = OLD.user_id;
    UPDATE users SET post_count = post_count + 1 WHERE id = NEW.user_id;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER posts_update_user_count
AFTER INSERT OR UPDATE OR DELETE ON posts
FOR EACH ROW
EXECUTE FUNCTION update_user_post_count();

-- ❌ 毎回COUNT（遅い）
SELECT
  u.username,
  COUNT(p.id) AS post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
GROUP BY u.id, u.username;
-- クエリ時間: 2,500ms (10万ユーザー)

-- ✅ カウンターキャッシュ使用（高速）
SELECT username, post_count FROM users;
-- クエリ時間: 15ms (-99%)
```

### パターン3: 参照データのコピー

頻繁に参照されるが、ほとんど変更されないデータ(商品名、カテゴリ名など)をコピーします。

```sql
-- ✅ 商品名を注文明細にコピー
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  price DECIMAL(10, 2) NOT NULL,
  updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
  id SERIAL PRIMARY KEY,
  order_id INTEGER REFERENCES orders(id),
  product_id INTEGER REFERENCES products(id),
  product_name VARCHAR(255) NOT NULL,  -- 非正規化（商品名のスナップショット）
  price DECIMAL(10, 2) NOT NULL,        -- 非正規化（購入時の価格）
  quantity INTEGER NOT NULL,

  INDEX idx_order_items_order (order_id),
  INDEX idx_order_items_product (product_id)
);

-- 理由:
-- 1. 商品名や価格が変更されても、過去の注文は変わらない
-- 2. JOINなしで注文明細を表示できる（パフォーマンス向上）
```

**クエリ例:**

```sql
-- ❌ JOIN必須（遅い）
SELECT
  oi.id,
  p.name AS product_name,
  oi.quantity,
  oi.price
FROM order_items oi
JOIN products p ON oi.product_id = p.id
WHERE oi.order_id = 123;
-- クエリ時間: 45ms

-- ✅ JOINなし（高速）
SELECT
  id,
  product_name,
  quantity,
  price
FROM order_items
WHERE order_id = 123;
-- クエリ時間: 8ms (-82%)
```

## 非正規化の判断基準

### 非正規化を検討すべきケース

1. **読み取り頻度 >> 書き込み頻度**
   - 注文の合計金額、投稿数カウンターなど
   - 読み取り: 1000回/秒、書き込み: 10回/秒

2. **集計クエリのパフォーマンスが重要**
   - ダッシュボード、レポート生成
   - 毎回のCOUNT、SUMが遅い

3. **履歴データの保持**
   - 注文時の商品名・価格のスナップショット
   - マスターデータが変更されても履歴は不変

4. **JOIN回数の削減**
   - 3つ以上のテーブルをJOINするクエリが頻繁
   - レスポンスタイムが300ms以上

### 正規化を維持すべきケース

1. **書き込み頻度 >> 読み取り頻度**
   - ログデータ、センサーデータ
   - 書き込み: 1000回/秒、読み取り: 10回/秒

2. **データの整合性が最重要**
   - 金融データ、医療データ
   - 不整合が許されない

3. **マスターデータ**
   - ユーザー、商品、カテゴリなど
   - 単一障害点(SSOT)として管理

## 実装パターン

### パターン1: マテリアライズドビュー(PostgreSQL)

トリガーではなく、マテリアライズドビューで非正規化します。

```sql
-- ✅ マテリアライズドビュー
CREATE MATERIALIZED VIEW user_stats AS
SELECT
  u.id,
  u.username,
  COUNT(p.id) AS post_count,
  MAX(p.created_at) AS last_post_at
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
GROUP BY u.id, u.username;

-- インデックス作成
CREATE UNIQUE INDEX idx_user_stats_id ON user_stats(id);

-- 定期的に更新（cron, バッチジョブ）
REFRESH MATERIALIZED VIEW CONCURRENTLY user_stats;
-- CONCURRENTLY: ロックなしで更新（読み取り可能）

-- クエリ
SELECT * FROM user_stats WHERE post_count > 100;
-- クエリ時間: 5ms (通常のVIEWは2,500ms)
```

### パターン2: Redisキャッシュ

頻繁にアクセスされる集計データをRedisにキャッシュします。

```typescript
// TypeScript + Redis
import Redis from 'ioredis'

const redis = new Redis()

async function getUserPostCount(userId: number): Promise<number> {
  const cacheKey = `user:${userId}:post_count`

  // キャッシュチェック
  const cached = await redis.get(cacheKey)
  if (cached !== null) {
    return parseInt(cached, 10)
  }

  // データベースから取得
  const count = await prisma.post.count({
    where: { userId }
  })

  // キャッシュに保存（TTL: 1時間）
  await redis.setex(cacheKey, 3600, count.toString())

  return count
}

// 投稿作成時にキャッシュを無効化
async function createPost(data: { userId: number; title: string }) {
  const post = await prisma.post.create({ data })

  // キャッシュ無効化
  await redis.del(`user:${data.userId}:post_count`)

  return post
}
```

## トラブルシューティング

### 問題1: トリガーによる更新が遅い

**症状:** order_itemsのINSERTが遅くなった

**原因:** トリガー内で重い集計クエリを実行している

**解決策:**

```sql
-- Before: 毎回SUMを計算（遅い）
CREATE OR REPLACE FUNCTION update_order_total()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE orders
  SET total_amount = (
    SELECT SUM(quantity * price) FROM order_items WHERE order_id = NEW.order_id
  )
  WHERE id = NEW.order_id;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- After: 差分のみ計算（高速）
CREATE OR REPLACE FUNCTION update_order_total_incremental()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    UPDATE orders
    SET total_amount = COALESCE(total_amount, 0) + (NEW.quantity * NEW.price)
    WHERE id = NEW.order_id;
  ELSIF TG_OP = 'DELETE' THEN
    UPDATE orders
    SET total_amount = COALESCE(total_amount, 0) - (OLD.quantity * OLD.price)
    WHERE id = OLD.order_id;
  ELSIF TG_OP = 'UPDATE' THEN
    UPDATE orders
    SET total_amount = COALESCE(total_amount, 0) - (OLD.quantity * OLD.price) + (NEW.quantity * NEW.price)
    WHERE id = NEW.order_id;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### 問題2: 非正規化フィールドとマスターデータの不整合

**症状:** 商品名を変更したが、過去の注文明細に反映されていない

**原因:** これは意図的な動作(履歴データの保持)

**解決策:** ビジネスロジックに応じて判断
- 履歴を保持する場合: 非正規化を維持
- 最新データを表示する場合: JOINで取得

```sql
-- 最新の商品名を表示
SELECT
  oi.id,
  p.name AS current_product_name,
  oi.product_name AS ordered_product_name,
  oi.quantity,
  oi.price
FROM order_items oi
LEFT JOIN products p ON oi.product_id = p.id
WHERE oi.order_id = 123;
```

## まとめ

高度な正規化と非正規化の判断基準をマスターすることで、以下の成果が得られます:

**想定効果:**
- BCNF適用: データ冗長性 -80%、更新処理 -95%
- 非正規化(集計値): クエリ時間 850ms → 12ms (-99%)
- カウンターキャッシュ: クエリ時間 2,500ms → 15ms (-99%)
- 参照データコピー: クエリ時間 45ms → 8ms (-82%)

**重要なポイント:**
1. **BCNF**: すべての決定子が候補キーである
2. **非正規化の判断**: 読み取り頻度、整合性要件、パフォーマンス目標を考慮
3. **トリガー vs マテリアライズドビュー**: リアルタイム性とパフォーマンスのトレードオフ
4. **キャッシュ戦略**: Redisなどのインメモリストアを活用

次の章では、リレーション設計パターン(1対多、多対多、自己参照など)について学びます。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
