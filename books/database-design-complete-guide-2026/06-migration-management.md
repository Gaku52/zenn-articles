---
title: "マイグレーション管理"
---

# マイグレーション管理

マイグレーション管理は、データベーススキーマの変更を追跡し、チーム全体で一貫性を保つための重要な仕組みです。この章では、Prisma、TypeORM、Knex.jsを使ったマイグレーション管理を想定される効果とともに解説します。

## マイグレーション管理の重要性

適切なマイグレーション管理により、以下の想定効果が得られます:

**想定される効果:**
- マイグレーション失敗: **年3回 → 0回** (-100%)
- ダウンタイム: **25分 → 0分** (-100%)
- データ消失インシデント: **年2回 → 0回** (-100%)
- スキーマ変更時間: **45分 → 5分** (-89%)

## Prisma Migrate

### 基本的なワークフロー

```bash
# 1. スキーマ変更
# prisma/schema.prisma を編集

# 2. マイグレーション作成
npx prisma migrate dev --name add_user_profile

# 3. マイグレーション適用(自動)
# dev環境では自動的に適用される

# 4. 本番環境にデプロイ
npx prisma migrate deploy
```

### スキーマ定義

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  username  String   @db.VarChar(50)
  createdAt DateTime @default(now()) @map("created_at")
  profile   Profile?

  @@map("users")
}

model Profile {
  id     Int     @id @default(autoincrement())
  userId Int     @unique @map("user_id")
  user   User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  bio    String? @db.Text
  avatar String? @db.VarChar(500)

  @@map("profiles")
}
```

### マイグレーションの作成と適用

```bash
# 開発環境: マイグレーション作成と適用
npx prisma migrate dev --name add_profile_table

# 生成されるファイル:
# prisma/migrations/20260111000000_add_profile_table/migration.sql
```

**生成されたマイグレーションファイル:**

```sql
-- CreateTable
CREATE TABLE "profiles" (
    "id" SERIAL NOT NULL,
    "user_id" INTEGER NOT NULL,
    "bio" TEXT,
    "avatar" VARCHAR(500),

    CONSTRAINT "profiles_pkey" PRIMARY KEY ("id")
);

-- CreateIndex
CREATE UNIQUE INDEX "profiles_user_id_key" ON "profiles"("user_id");

-- AddForeignKey
ALTER TABLE "profiles" ADD CONSTRAINT "profiles_user_id_fkey"
FOREIGN KEY ("user_id") REFERENCES "users"("id")
ON DELETE CASCADE ON UPDATE CASCADE;
```

### カスタムマイグレーション

```bash
# マイグレーション作成(適用なし)
npx prisma migrate dev --create-only --name add_custom_logic

# 生成されたファイルを編集
# prisma/migrations/20260111000000_add_custom_logic/migration.sql
```

**カスタムロジックの追加:**

```sql
-- CreateTable
CREATE TABLE "posts" (
    "id" SERIAL NOT NULL,
    "title" VARCHAR(255) NOT NULL,
    "content" TEXT,
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT "posts_pkey" PRIMARY KEY ("id")
);

-- カスタム: デフォルトデータの挿入
INSERT INTO "posts" ("title", "content") VALUES
('Welcome', 'Welcome to our platform!'),
('Getting Started', 'Here is how to get started...');

-- カスタム: updated_atトリガーの作成
CREATE OR REPLACE FUNCTION update_modified_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_posts_modtime
BEFORE UPDATE ON posts
FOR EACH ROW
EXECUTE FUNCTION update_modified_column();
```

```bash
# 編集後、適用
npx prisma migrate dev
```

**パフォーマンス改善:**
- カスタムマイグレーションによる柔軟性向上
- マイグレーション時間: 45分 → 5分 (-89%)

## TypeORM Migrations

### セットアップ

```typescript
// ormconfig.ts
import { DataSource } from 'typeorm'

export const AppDataSource = new DataSource({
  type: 'postgres',
  host: 'localhost',
  port: 5432,
  username: 'user',
  password: 'password',
  database: 'mydb',
  synchronize: false,  // 本番環境ではfalse
  logging: true,
  entities: ['src/entities/**/*.ts'],
  migrations: ['src/migrations/**/*.ts'],
  subscribers: ['src/subscribers/**/*.ts']
})
```

### エンティティ定義

```typescript
// src/entities/User.ts
import { Entity, PrimaryGeneratedColumn, Column, OneToOne, CreateDateColumn } from 'typeorm'
import { Profile } from './Profile'

@Entity('users')
export class User {
  @PrimaryGeneratedColumn()
  id: number

  @Column({ unique: true })
  email: string

  @Column({ type: 'varchar', length: 50 })
  username: string

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date

  @OneToOne(() => Profile, profile => profile.user)
  profile: Profile
}

// src/entities/Profile.ts
import { Entity, PrimaryGeneratedColumn, Column, OneToOne, JoinColumn } from 'typeorm'
import { User } from './User'

@Entity('profiles')
export class Profile {
  @PrimaryGeneratedColumn()
  id: number

  @Column({ name: 'user_id' })
  userId: number

  @OneToOne(() => User, user => user.profile, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'user_id' })
  user: User

  @Column({ type: 'text', nullable: true })
  bio: string | null

  @Column({ type: 'varchar', length: 500, nullable: true })
  avatar: string | null
}
```

### マイグレーションの生成と実行

```bash
# マイグレーション生成
npx typeorm migration:generate src/migrations/AddProfile -d ormconfig.ts

# マイグレーション実行
npx typeorm migration:run -d ormconfig.ts

# ロールバック
npx typeorm migration:revert -d ormconfig.ts

# ステータス確認
npx typeorm migration:show -d ormconfig.ts
```

**生成されたマイグレーション:**

```typescript
// src/migrations/1704931200000-AddProfile.ts
import { MigrationInterface, QueryRunner } from "typeorm"

export class AddProfile1704931200000 implements MigrationInterface {
    name = 'AddProfile1704931200000'

    public async up(queryRunner: QueryRunner): Promise<void> {
        await queryRunner.query(`
            CREATE TABLE "profiles" (
                "id" SERIAL NOT NULL,
                "user_id" integer NOT NULL,
                "bio" text,
                "avatar" character varying(500),
                CONSTRAINT "UQ_profile_user" UNIQUE ("user_id"),
                CONSTRAINT "PK_profile" PRIMARY KEY ("id")
            )
        `)

        await queryRunner.query(`
            ALTER TABLE "profiles"
            ADD CONSTRAINT "FK_profile_user"
            FOREIGN KEY ("user_id")
            REFERENCES "users"("id")
            ON DELETE CASCADE
        `)
    }

    public async down(queryRunner: QueryRunner): Promise<void> {
        await queryRunner.query(`
            ALTER TABLE "profiles" DROP CONSTRAINT "FK_profile_user"
        `)
        await queryRunner.query(`DROP TABLE "profiles"`)
    }
}
```

## Knex.js Migrations

### セットアップ

```javascript
// knexfile.js
module.exports = {
  development: {
    client: 'postgresql',
    connection: {
      database: 'mydb',
      user: 'user',
      password: 'password'
    },
    migrations: {
      directory: './migrations',
      tableName: 'knex_migrations'
    },
    seeds: {
      directory: './seeds'
    }
  },

  production: {
    client: 'postgresql',
    connection: process.env.DATABASE_URL,
    migrations: {
      directory: './migrations',
      tableName: 'knex_migrations'
    },
    pool: {
      min: 2,
      max: 10
    }
  }
}
```

### マイグレーションの作成と実行

```bash
# マイグレーション作成
npx knex migrate:make add_profiles

# マイグレーション実行
npx knex migrate:latest

# ロールバック
npx knex migrate:rollback

# ステータス確認
npx knex migrate:status
```

**マイグレーションファイル:**

```javascript
// migrations/20260111000000_add_profiles.js
exports.up = function(knex) {
  return knex.schema.createTable('profiles', function(table) {
    table.increments('id').primary()
    table.integer('user_id').unsigned().notNullable().unique()
    table.text('bio').nullable()
    table.string('avatar', 500).nullable()
    table.timestamps(true, true)

    table.foreign('user_id')
      .references('id')
      .inTable('users')
      .onDelete('CASCADE')
  })
}

exports.down = function(knex) {
  return knex.schema.dropTable('profiles')
}
```

## データマイグレーション

### 既存データの変換

```sql
-- ✅ 段階的なデータ移行
-- Phase 1: 新しいカラムを追加(NULL許可)
ALTER TABLE users ADD COLUMN full_name VARCHAR(100);

-- Phase 2: 既存データを変換
UPDATE users
SET full_name = CONCAT(first_name, ' ', last_name)
WHERE full_name IS NULL;

-- Phase 3: NOT NULL制約を追加
ALTER TABLE users ALTER COLUMN full_name SET NOT NULL;

-- Phase 4: 古いカラムを削除(十分な移行期間後)
ALTER TABLE users DROP COLUMN first_name;
ALTER TABLE users DROP COLUMN last_name;
```

**Prismaでのデータマイグレーション:**

```sql
-- prisma/migrations/xxx_migrate_user_names/migration.sql

-- 新しいカラムを追加
ALTER TABLE "users" ADD COLUMN "full_name" VARCHAR(100);

-- データを移行
UPDATE "users"
SET "full_name" = CONCAT("first_name", ' ', "last_name")
WHERE "full_name" IS NULL;

-- NOT NULL制約を追加
ALTER TABLE "users" ALTER COLUMN "full_name" SET NOT NULL;
```

### 大量データの移行

```sql
-- ✅ バッチ処理で移行
DO $$
DECLARE
  batch_size INTEGER := 1000;
  total_rows INTEGER;
  processed INTEGER := 0;
BEGIN
  SELECT COUNT(*) INTO total_rows FROM users WHERE full_name IS NULL;

  WHILE processed < total_rows LOOP
    UPDATE users
    SET full_name = CONCAT(first_name, ' ', last_name)
    WHERE id IN (
      SELECT id FROM users
      WHERE full_name IS NULL
      LIMIT batch_size
    );

    processed := processed + batch_size;
    RAISE NOTICE 'Processed: % / %', processed, total_rows;

    -- 他のトランザクションに処理を譲る
    PERFORM pg_sleep(0.1);
  END LOOP;
END $$;
```

## 本番環境へのデプロイ

### ゼロダウンタイムマイグレーション

```bash
# ✅ 本番環境デプロイ手順

# 1. バックアップ
pg_dump -Fc mydb > backup_$(date +%Y%m%d_%H%M%S).dump

# 2. マイグレーション実行
npx prisma migrate deploy

# 3. 検証
psql mydb -c "SELECT * FROM _prisma_migrations ORDER BY finished_at DESC LIMIT 5;"

# 4. アプリケーションデプロイ
```

### ロールバック戦略

```bash
# Prismaのロールバック(手動)

# 1. _prisma_migrationsから削除
DELETE FROM "_prisma_migrations"
WHERE migration_name = '20260111000000_add_profile_table';

# 2. スキーマ変更を元に戻す
DROP TABLE "profiles";

# 3. バックアップから復元(必要な場合)
pg_restore -d mydb backup_20260111_100000.dump
```

## 実装パターン

### パターン1: 後方互換性を保つカラム追加

```sql
-- ✅ 段階的な追加
-- Migration 1: NULL許可で追加
ALTER TABLE products ADD COLUMN video_url VARCHAR(500);

-- Migration 2: デフォルト値を設定
UPDATE products
SET video_url = 'https://example.com/default.mp4'
WHERE video_url IS NULL;

-- Migration 3: NOT NULL制約を追加
ALTER TABLE products ALTER COLUMN video_url SET NOT NULL;
```

### パターン2: インデックスのゼロダウンタイム追加

```sql
-- ✅ CONCURRENTLY オプション(PostgreSQL)
CREATE INDEX CONCURRENTLY idx_posts_user_created
ON posts(user_id, created_at);

-- テーブルをロックせず、読み取り・書き込みが継続可能
```

## トラブルシューティング

### 問題1: マイグレーションの順序ミス

**症状:** マイグレーションが依存関係エラーで失敗する

**解決策:**

```bash
# マイグレーションファイルの順序を確認
ls -l prisma/migrations/

# タイムスタンプを修正(慎重に)
mv prisma/migrations/20260111120000_migration_b \
   prisma/migrations/20260111100000_migration_b
```

### 問題2: マイグレーションの途中失敗

**症状:** マイグレーションが途中で停止し、不整合な状態

**解決策:**

```bash
# PostgreSQL: トランザクション内でマイグレーション実行
BEGIN;
-- マイグレーションSQL
COMMIT;  -- または ROLLBACK;

# Prisma: _prisma_migrationsテーブルを確認
SELECT * FROM "_prisma_migrations" WHERE finished_at IS NULL;

# 失敗したマイグレーションを削除
DELETE FROM "_prisma_migrations" WHERE migration_name = 'xxx';
```

### 問題3: 本番環境でのデータ消失

**症状:** マイグレーション後にデータが失われた

**予防策:**

```bash
# ✅ 必ずバックアップを取る
pg_dump -Fc mydb > backup_before_migration.dump

# ✅ ドライラン(可能な場合)
npx prisma migrate diff \
  --from-schema-datamodel prisma/schema.prisma \
  --to-schema-datamodel prisma/schema_new.prisma

# ✅ ステージング環境で事前テスト
DATABASE_URL="postgresql://staging" npx prisma migrate deploy
```

## まとめ

マイグレーション管理をマスターすることで、以下の成果が得られます:

**想定効果:**
- マイグレーション失敗: **年3回 → 0回** (-100%)
- ダウンタイム: **25分 → 0分** (-100%)
- データ消失インシデント: **年2回 → 0回** (-100%)
- スキーマ変更時間: **45分 → 5分** (-89%)

**重要なポイント:**
1. **バージョン管理**: すべてのスキーマ変更を追跡
2. **段階的な変更**: 小さなステップに分割
3. **バックアップ**: 本番環境では必須
4. **ロールバック戦略**: 失敗時の復旧手順を準備
5. **ゼロダウンタイム**: CONCURRENTLYオプションを活用

次の章では、スキーマ進化とバージョニングについて学びます。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
