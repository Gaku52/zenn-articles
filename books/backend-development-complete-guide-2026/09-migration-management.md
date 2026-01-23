---
title: "Chapter 09: マイグレーション管理 - データベース変更の完全制御"
---

# Chapter 09: マイグレーション管理

## イントロダクション

データベースマイグレーションは、バックエンド開発において最も重要かつ慎重に扱うべき領域の一つです。スキーマ変更の管理を誤ると、データ消失、ダウンタイム、環境間の不整合など、深刻な問題を引き起こします。

本章では、Prisma Migrateを中心に、TypeORM、Knex.jsなど主要なマイグレーションツールの実践的な使用方法、ゼロダウンタイムマイグレーション、本番環境へのデプロイ戦略、データマイグレーションのベストプラクティスを詳しく解説します。

### 本章で学ぶこと

- Prisma Migrateの完全マスター
- TypeORM、Knex.jsのマイグレーション戦略
- ゼロダウンタイムマイグレーションの実装
- カスタムマイグレーションとデータ移行
- 本番環境への安全なデプロイ
- ロールバック戦略とバックアップ

---

## マイグレーションの基礎

### マイグレーションとは

データベースマイグレーションは、スキーマの変更を追跡・管理し、チーム全体で一貫性を保つための仕組みです。

#### 主な目的

1. **スキーマのバージョン管理** - Gitのようにスキーマ変更を履歴として管理
2. **環境間での一貫性確保** - 開発・ステージング・本番で同じスキーマ
3. **変更の可逆性** - ロールバック可能な設計
4. **チーム開発での競合回避** - 複数人での同時開発をサポート

#### マイグレーションのライフサイクル

```
┌─────────────────┐
│ 1. スキーマ変更  │
│   (schema.prisma)│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 2. マイグレー    │
│   ション作成     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 3. マイグレー    │
│   ション適用     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 4. 本番デプロイ  │
└─────────────────┘
```

### マイグレーションツール比較

| ツール | 対応DB | 自動生成 | ロールバック | トランザクション | 学習コスト |
|--------|--------|----------|-------------|-----------------|-----------|
| **Prisma Migrate** | PostgreSQL, MySQL, SQLite, SQL Server, MongoDB | ✅ | ❌ | ✅ | 低 |
| **TypeORM** | PostgreSQL, MySQL, MariaDB, SQLite, MS SQL, Oracle, CockroachDB | ✅ | ✅ | ✅ | 中 |
| **Knex.js** | PostgreSQL, MySQL, MariaDB, SQLite, Oracle, MS SQL | ❌ | ✅ | ✅ | 中 |
| **Sequelize** | PostgreSQL, MySQL, MariaDB, SQLite, MS SQL | ✅ | ✅ | ✅ | 中 |

---

## Prisma Migrate完全ガイド

### 基本的なワークフロー

Prisma Migrateは、スキーマファイル（`schema.prisma`）からマイグレーションを自動生成します。

#### 1. プロジェクトセットアップ

```bash
# Prismaのインストール
npm install @prisma/client
npm install -D prisma

# Prismaの初期化
npx prisma init
```

これにより、以下のファイルが生成されます：

```
project/
├── prisma/
│   └── schema.prisma
└── .env
```

#### 2. スキーマ定義

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
  password  String   @db.VarChar(255)
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")
  profile   Profile?
  posts     Post[]

  @@map("users")
}

model Profile {
  id        Int      @id @default(autoincrement())
  userId    Int      @unique @map("user_id")
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  bio       String?  @db.Text
  avatar    String?  @db.VarChar(500)
  website   String?  @db.VarChar(500)
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  @@map("profiles")
}

model Post {
  id          Int       @id @default(autoincrement())
  title       String    @db.VarChar(255)
  content     String?   @db.Text
  published   Boolean   @default(false)
  publishedAt DateTime? @map("published_at")
  authorId    Int       @map("author_id")
  author      User      @relation(fields: [authorId], references: [id], onDelete: Cascade)
  createdAt   DateTime  @default(now()) @map("created_at")
  updatedAt   DateTime  @updatedAt @map("updated_at")

  @@index([authorId])
  @@index([publishedAt])
  @@map("posts")
}
```

#### 3. マイグレーション作成と適用

```bash
# 開発環境：マイグレーション作成と適用
npx prisma migrate dev --name initial_schema

# 以下が自動的に実行されます：
# 1. マイグレーションファイル生成
# 2. データベースへの適用
# 3. Prisma Clientの再生成
```

生成されるマイグレーションファイル：

```sql
-- prisma/migrations/20260103120000_initial_schema/migration.sql

-- CreateTable
CREATE TABLE "users" (
    "id" SERIAL NOT NULL,
    "email" TEXT NOT NULL,
    "username" VARCHAR(50) NOT NULL,
    "password" VARCHAR(255) NOT NULL,
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_at" TIMESTAMP(3) NOT NULL,

    CONSTRAINT "users_pkey" PRIMARY KEY ("id")
);

-- CreateTable
CREATE TABLE "profiles" (
    "id" SERIAL NOT NULL,
    "user_id" INTEGER NOT NULL,
    "bio" TEXT,
    "avatar" VARCHAR(500),
    "website" VARCHAR(500),
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_at" TIMESTAMP(3) NOT NULL,

    CONSTRAINT "profiles_pkey" PRIMARY KEY ("id")
);

-- CreateTable
CREATE TABLE "posts" (
    "id" SERIAL NOT NULL,
    "title" VARCHAR(255) NOT NULL,
    "content" TEXT,
    "published" BOOLEAN NOT NULL DEFAULT false,
    "published_at" TIMESTAMP(3),
    "author_id" INTEGER NOT NULL,
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_at" TIMESTAMP(3) NOT NULL,

    CONSTRAINT "posts_pkey" PRIMARY KEY ("id")
);

-- CreateIndex
CREATE UNIQUE INDEX "users_email_key" ON "users"("email");

-- CreateIndex
CREATE UNIQUE INDEX "profiles_user_id_key" ON "profiles"("user_id");

-- CreateIndex
CREATE INDEX "posts_author_id_idx" ON "posts"("author_id");

-- CreateIndex
CREATE INDEX "posts_published_at_idx" ON "posts"("published_at");

-- AddForeignKey
ALTER TABLE "profiles" ADD CONSTRAINT "profiles_user_id_fkey"
    FOREIGN KEY ("user_id") REFERENCES "users"("id") ON DELETE CASCADE ON UPDATE CASCADE;

-- AddForeignKey
ALTER TABLE "posts" ADD CONSTRAINT "posts_author_id_fkey"
    FOREIGN KEY ("author_id") REFERENCES "users"("id") ON DELETE CASCADE ON UPDATE CASCADE;
```

### マイグレーションコマンド詳細

#### 開発環境コマンド

```bash
# マイグレーション作成と適用
npx prisma migrate dev --name add_user_roles

# マイグレーション作成のみ（適用なし）
npx prisma migrate dev --create-only --name add_indexes

# スキーマとDBの同期（マイグレーション履歴なし）
# 開発初期のみ推奨
npx prisma db push
```

#### 本番環境コマンド

```bash
# マイグレーション適用（本番環境）
npx prisma migrate deploy

# マイグレーション状態確認
npx prisma migrate status

# マイグレーション履歴表示
npx prisma migrate history
```

#### トラブルシューティングコマンド

```bash
# マイグレーションをスキップ（失敗したマイグレーションを適用済みとしてマーク）
npx prisma migrate resolve --applied 20260103120000_failed_migration

# マイグレーションをロールバック（手動でDBを元に戻した後）
npx prisma migrate resolve --rolled-back 20260103120000_failed_migration

# 既存DBにベースラインを設定
npx prisma migrate resolve --applied 0_init
```

### カスタムマイグレーション

Prismaが自動生成したマイグレーションに、カスタムロジックを追加できます。

#### 1. カスタムマイグレーション作成

```bash
# マイグレーション作成（適用なし）
npx prisma migrate dev --create-only --name add_full_text_search
```

#### 2. 生成されたSQLファイルを編集

```sql
-- prisma/migrations/20260103120000_add_full_text_search/migration.sql

-- Prismaが生成したSQL
ALTER TABLE "posts" ADD COLUMN "search_vector" tsvector;

-- カスタムSQL: 全文検索用のトリガー
CREATE OR REPLACE FUNCTION posts_search_trigger() RETURNS trigger AS $$
BEGIN
  NEW.search_vector :=
    setweight(to_tsvector('english', coalesce(NEW.title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(NEW.content, '')), 'B');
  RETURN NEW;
END
$$ LANGUAGE plpgsql;

CREATE TRIGGER posts_search_update
  BEFORE INSERT OR UPDATE ON posts
  FOR EACH ROW
  EXECUTE FUNCTION posts_search_trigger();

-- インデックス作成
CREATE INDEX posts_search_idx ON posts USING GIN(search_vector);

-- 既存データの更新
UPDATE posts
SET search_vector =
  setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
  setweight(to_tsvector('english', coalesce(content, '')), 'B');
```

#### 3. マイグレーション適用

```bash
npx prisma migrate dev
```

### データマイグレーション

既存データを変換する必要がある場合のパターンです。

#### パターン1: カラム名変更（ゼロダウンタイム）

```bash
# Phase 1: 新カラム追加
npx prisma migrate dev --create-only --name add_full_name_column
```

```sql
-- migration.sql
-- Step 1: 新しいカラムを追加（NULL許可）
ALTER TABLE "users" ADD COLUMN "full_name" VARCHAR(200);

-- Step 2: 既存データから full_name を生成
UPDATE "users"
SET "full_name" = CONCAT("first_name", ' ', "last_name")
WHERE "full_name" IS NULL AND "first_name" IS NOT NULL AND "last_name" IS NOT NULL;

-- Step 3: NOT NULL制約を追加
ALTER TABLE "users" ALTER COLUMN "full_name" SET NOT NULL;

-- Step 4: 古いカラムを削除（次のマイグレーションで）
-- この時点ではまだ削除しない（アプリケーションコードの更新待ち）
```

アプリケーションコードの更新：

```typescript
// Phase 1: 両方のカラムに書き込み
await prisma.user.create({
  data: {
    firstName: 'John',
    lastName: 'Doe',
    fullName: 'John Doe',  // 新カラム
    email: 'john@example.com',
  },
})

// Phase 2: 新カラムから読み込み（フォールバック付き）
const user = await prisma.user.findUnique({ where: { id: 1 } })
const name = user.fullName || `${user.firstName} ${user.lastName}`

// Phase 3: 古いカラムを削除（十分な移行期間後）
```

```bash
# Phase 2: 古いカラム削除（2週間後など）
npx prisma migrate dev --create-only --name remove_old_name_columns
```

```sql
-- migration.sql
ALTER TABLE "users" DROP COLUMN "first_name";
ALTER TABLE "users" DROP COLUMN "last_name";
```

#### パターン2: 大量データの移行

```bash
npx prisma migrate dev --create-only --name migrate_user_status
```

```sql
-- migration.sql
-- Step 1: 新しいカラムを追加
ALTER TABLE "users" ADD COLUMN "status" VARCHAR(20) DEFAULT 'active';

-- Step 2: バッチ処理で既存データを変換
DO $$
DECLARE
  batch_size INTEGER := 1000;
  offset_val INTEGER := 0;
  rows_affected INTEGER;
BEGIN
  LOOP
    -- バッチで更新
    UPDATE "users"
    SET "status" = CASE
      WHEN "is_active" = true THEN 'active'
      WHEN "is_suspended" = true THEN 'suspended'
      ELSE 'inactive'
    END
    WHERE id IN (
      SELECT id FROM "users"
      WHERE "status" = 'active' -- デフォルト値のまま
      LIMIT batch_size
      OFFSET offset_val
    );

    GET DIAGNOSTICS rows_affected = ROW_COUNT;

    EXIT WHEN rows_affected = 0;

    offset_val := offset_val + batch_size;

    RAISE NOTICE 'Processed % rows', offset_val;
  END LOOP;
END $$;

-- Step 3: NOT NULL制約を追加
ALTER TABLE "users" ALTER COLUMN "status" SET NOT NULL;

-- Step 4: 古いカラムを削除
ALTER TABLE "users" DROP COLUMN "is_active";
ALTER TABLE "users" DROP COLUMN "is_suspended";
```

### Prismaのロールバック戦略

Prismaには公式なロールバック機能がないため、手動で対応する必要があります。

#### 方法1: 逆マイグレーションの作成

```bash
# 逆の変更を行う新しいマイグレーションを作成
npx prisma migrate dev --create-only --name revert_add_profile_table
```

```sql
-- migration.sql
-- 追加したテーブルを削除
DROP TABLE "profiles";
```

#### 方法2: データベースバックアップから復元

```bash
# バックアップから復元
pg_restore -U postgres -d mydb backup_20260103_120000.dump

# マイグレーション履歴を手動で修正
# _prisma_migrationsテーブルから該当レコードを削除
DELETE FROM "_prisma_migrations"
WHERE migration_name = '20260103120000_add_profile_table';
```

#### 方法3: resolve コマンドでスキップ

```bash
# マイグレーションを適用済みとしてマーク（手動でDBを戻した後）
npx prisma migrate resolve --rolled-back 20260103120000_add_profile_table
```

---

## TypeORM Migrations

TypeORMは、エンティティ定義からマイグレーションを自動生成できる、フル機能のORMです。

### セットアップ

```bash
# TypeORMのインストール
npm install typeorm reflect-metadata pg

# TypeScript設定
npm install -D @types/node ts-node typescript
```

#### データソース設定

```typescript
// src/data-source.ts
import { DataSource } from 'typeorm'
import { User } from './entities/User'
import { Profile } from './entities/Profile'
import { Post } from './entities/Post'

export const AppDataSource = new DataSource({
  type: 'postgres',
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT || '5432'),
  username: process.env.DB_USER || 'postgres',
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME || 'mydb',
  entities: [User, Profile, Post],
  migrations: ['src/migrations/**/*.ts'],
  synchronize: false, // 本番では絶対にfalse
  logging: process.env.NODE_ENV === 'development',
  migrationsRun: false, // 手動でマイグレーション実行
})
```

### エンティティ定義

```typescript
// src/entities/User.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  OneToOne,
  OneToMany,
  Index,
} from 'typeorm'
import { Profile } from './Profile'
import { Post } from './Post'

@Entity('users')
export class User {
  @PrimaryGeneratedColumn()
  id: number

  @Column({ unique: true })
  @Index()
  email: string

  @Column({ length: 50 })
  username: string

  @Column({ length: 255 })
  password: string

  @Column({ type: 'varchar', length: 20, default: 'active' })
  status: 'active' | 'inactive' | 'suspended'

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date

  @OneToOne(() => Profile, (profile) => profile.user)
  profile: Profile

  @OneToMany(() => Post, (post) => post.author)
  posts: Post[]
}
```

```typescript
// src/entities/Profile.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  OneToOne,
  JoinColumn,
  CreateDateColumn,
  UpdateDateColumn,
} from 'typeorm'
import { User } from './User'

@Entity('profiles')
export class Profile {
  @PrimaryGeneratedColumn()
  id: number

  @Column({ name: 'user_id', unique: true })
  userId: number

  @Column({ type: 'text', nullable: true })
  bio: string

  @Column({ length: 500, nullable: true })
  avatar: string

  @Column({ length: 500, nullable: true })
  website: string

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date

  @OneToOne(() => User, (user) => user.profile, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'user_id' })
  user: User
}
```

```typescript
// src/entities/Post.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  ManyToOne,
  JoinColumn,
  CreateDateColumn,
  UpdateDateColumn,
  Index,
} from 'typeorm'
import { User } from './User'

@Entity('posts')
@Index(['authorId'])
@Index(['publishedAt'])
export class Post {
  @PrimaryGeneratedColumn()
  id: number

  @Column({ length: 255 })
  title: string

  @Column({ type: 'text', nullable: true })
  content: string

  @Column({ default: false })
  published: boolean

  @Column({ name: 'published_at', nullable: true })
  publishedAt: Date

  @Column({ name: 'author_id' })
  authorId: number

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date

  @ManyToOne(() => User, (user) => user.posts, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'author_id' })
  author: User
}
```

### マイグレーション生成と実行

```bash
# マイグレーション自動生成
npx typeorm migration:generate src/migrations/InitialSchema -d src/data-source.ts

# 空のマイグレーション作成
npx typeorm migration:create src/migrations/AddCustomIndexes

# マイグレーション適用
npx typeorm migration:run -d src/data-source.ts

# ロールバック（最新1件）
npx typeorm migration:revert -d src/data-source.ts

# マイグレーション状態確認
npx typeorm migration:show -d src/data-source.ts
```

### 生成されたマイグレーション

```typescript
// src/migrations/1704297600000-InitialSchema.ts
import { MigrationInterface, QueryRunner } from 'typeorm'

export class InitialSchema1704297600000 implements MigrationInterface {
  name = 'InitialSchema1704297600000'

  public async up(queryRunner: QueryRunner): Promise<void> {
    // Users table
    await queryRunner.query(`
      CREATE TABLE "users" (
        "id" SERIAL NOT NULL,
        "email" character varying NOT NULL,
        "username" character varying(50) NOT NULL,
        "password" character varying(255) NOT NULL,
        "status" character varying(20) NOT NULL DEFAULT 'active',
        "created_at" TIMESTAMP NOT NULL DEFAULT now(),
        "updated_at" TIMESTAMP NOT NULL DEFAULT now(),
        CONSTRAINT "UQ_users_email" UNIQUE ("email"),
        CONSTRAINT "PK_users" PRIMARY KEY ("id")
      )
    `)

    await queryRunner.query(`
      CREATE INDEX "IDX_users_email" ON "users" ("email")
    `)

    // Profiles table
    await queryRunner.query(`
      CREATE TABLE "profiles" (
        "id" SERIAL NOT NULL,
        "user_id" integer NOT NULL,
        "bio" text,
        "avatar" character varying(500),
        "website" character varying(500),
        "created_at" TIMESTAMP NOT NULL DEFAULT now(),
        "updated_at" TIMESTAMP NOT NULL DEFAULT now(),
        CONSTRAINT "UQ_profiles_user_id" UNIQUE ("user_id"),
        CONSTRAINT "PK_profiles" PRIMARY KEY ("id")
      )
    `)

    // Posts table
    await queryRunner.query(`
      CREATE TABLE "posts" (
        "id" SERIAL NOT NULL,
        "title" character varying(255) NOT NULL,
        "content" text,
        "published" boolean NOT NULL DEFAULT false,
        "published_at" TIMESTAMP,
        "author_id" integer NOT NULL,
        "created_at" TIMESTAMP NOT NULL DEFAULT now(),
        "updated_at" TIMESTAMP NOT NULL DEFAULT now(),
        CONSTRAINT "PK_posts" PRIMARY KEY ("id")
      )
    `)

    await queryRunner.query(`
      CREATE INDEX "IDX_posts_author_id" ON "posts" ("author_id")
    `)

    await queryRunner.query(`
      CREATE INDEX "IDX_posts_published_at" ON "posts" ("published_at")
    `)

    // Foreign keys
    await queryRunner.query(`
      ALTER TABLE "profiles"
      ADD CONSTRAINT "FK_profiles_user_id"
      FOREIGN KEY ("user_id")
      REFERENCES "users"("id")
      ON DELETE CASCADE
    `)

    await queryRunner.query(`
      ALTER TABLE "posts"
      ADD CONSTRAINT "FK_posts_author_id"
      FOREIGN KEY ("author_id")
      REFERENCES "users"("id")
      ON DELETE CASCADE
    `)
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`
      ALTER TABLE "posts" DROP CONSTRAINT "FK_posts_author_id"
    `)

    await queryRunner.query(`
      ALTER TABLE "profiles" DROP CONSTRAINT "FK_profiles_user_id"
    `)

    await queryRunner.query(`DROP INDEX "IDX_posts_published_at"`)
    await queryRunner.query(`DROP INDEX "IDX_posts_author_id"`)
    await queryRunner.query(`DROP TABLE "posts"`)

    await queryRunner.query(`DROP TABLE "profiles"`)

    await queryRunner.query(`DROP INDEX "IDX_users_email"`)
    await queryRunner.query(`DROP TABLE "users"`)
  }
}
```

### データマイグレーション（TypeORM）

```typescript
// src/migrations/1704297700000-MigrateUserData.ts
import { MigrationInterface, QueryRunner } from 'typeorm'

export class MigrateUserData1704297700000 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    // トランザクション内で実行
    await queryRunner.startTransaction()

    try {
      // 新しいカラムを追加
      await queryRunner.query(`
        ALTER TABLE "users" ADD COLUMN "full_name" VARCHAR(200)
      `)

      // バッチ処理で既存データを変換
      const batchSize = 1000
      let offset = 0
      let hasMore = true

      while (hasMore) {
        const users = await queryRunner.query(`
          SELECT id, first_name, last_name
          FROM users
          WHERE full_name IS NULL
          LIMIT $1 OFFSET $2
        `, [batchSize, offset])

        if (users.length === 0) {
          hasMore = false
          break
        }

        // バッチで更新
        for (const user of users) {
          const fullName = `${user.first_name} ${user.last_name}`.trim()

          await queryRunner.query(`
            UPDATE users
            SET full_name = $1
            WHERE id = $2
          `, [fullName, user.id])
        }

        offset += batchSize
        console.log(`Processed ${offset} users`)
      }

      // NOT NULL制約を追加
      await queryRunner.query(`
        ALTER TABLE "users" ALTER COLUMN "full_name" SET NOT NULL
      `)

      // 古いカラムを削除
      await queryRunner.query(`
        ALTER TABLE "users" DROP COLUMN "first_name"
      `)
      await queryRunner.query(`
        ALTER TABLE "users" DROP COLUMN "last_name"
      `)

      await queryRunner.commitTransaction()
    } catch (error) {
      await queryRunner.rollbackTransaction()
      throw error
    }
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.startTransaction()

    try {
      // 古いカラムを復元
      await queryRunner.query(`
        ALTER TABLE "users" ADD COLUMN "first_name" VARCHAR(50)
      `)
      await queryRunner.query(`
        ALTER TABLE "users" ADD COLUMN "last_name" VARCHAR(50)
      `)

      // full_nameから first_name と last_name を復元
      await queryRunner.query(`
        UPDATE users
        SET
          first_name = SPLIT_PART(full_name, ' ', 1),
          last_name = SPLIT_PART(full_name, ' ', 2)
      `)

      // full_nameを削除
      await queryRunner.query(`
        ALTER TABLE "users" DROP COLUMN "full_name"
      `)

      await queryRunner.commitTransaction()
    } catch (error) {
      await queryRunner.rollbackTransaction()
      throw error
    }
  }
}
```

---

## Knex.js Migrations

Knex.jsは、軽量なクエリビルダーで、シンプルなマイグレーション機能を提供します。

### セットアップ

```bash
# Knex.jsのインストール
npm install knex pg

# Knex CLI（グローバル）
npm install -g knex
```

#### Knex設定ファイル

```typescript
// knexfile.ts
import type { Knex } from 'knex'

const config: { [key: string]: Knex.Config } = {
  development: {
    client: 'postgresql',
    connection: {
      host: process.env.DB_HOST || 'localhost',
      port: parseInt(process.env.DB_PORT || '5432'),
      user: process.env.DB_USER || 'postgres',
      password: process.env.DB_PASSWORD,
      database: process.env.DB_NAME || 'mydb',
    },
    migrations: {
      directory: './migrations',
      extension: 'ts',
      tableName: 'knex_migrations',
    },
    seeds: {
      directory: './seeds',
      extension: 'ts',
    },
  },

  production: {
    client: 'postgresql',
    connection: process.env.DATABASE_URL,
    migrations: {
      directory: './migrations',
      extension: 'ts',
    },
    pool: {
      min: 2,
      max: 10,
    },
  },
}

export default config
```

### マイグレーション作成

```bash
# マイグレーション作成
npx knex migrate:make initial_schema --knexfile knexfile.ts
```

```typescript
// migrations/20260103120000_initial_schema.ts
import type { Knex } from 'knex'

export async function up(knex: Knex): Promise<void> {
  // Users table
  await knex.schema.createTable('users', (table) => {
    table.increments('id').primary()
    table.string('email', 255).unique().notNullable()
    table.string('username', 50).notNullable()
    table.string('password', 255).notNullable()
    table.enu('status', ['active', 'inactive', 'suspended']).defaultTo('active')
    table.timestamps(true, true, true) // created_at, updated_at with default now()

    table.index(['email'])
  })

  // Profiles table
  await knex.schema.createTable('profiles', (table) => {
    table.increments('id').primary()
    table.integer('user_id').unsigned().notNullable().unique()
    table.text('bio')
    table.string('avatar', 500)
    table.string('website', 500)
    table.timestamps(true, true, true)

    table.foreign('user_id').references('id').inTable('users').onDelete('CASCADE')
  })

  // Posts table
  await knex.schema.createTable('posts', (table) => {
    table.increments('id').primary()
    table.string('title', 255).notNullable()
    table.text('content')
    table.boolean('published').defaultTo(false)
    table.timestamp('published_at')
    table.integer('author_id').unsigned().notNullable()
    table.timestamps(true, true, true)

    table.foreign('author_id').references('id').inTable('users').onDelete('CASCADE')
    table.index(['author_id'])
    table.index(['published_at'])
  })
}

export async function down(knex: Knex): Promise<void> {
  await knex.schema.dropTableIfExists('posts')
  await knex.schema.dropTableIfExists('profiles')
  await knex.schema.dropTableIfExists('users')
}
```

### マイグレーション実行

```bash
# 最新までマイグレーション
npx knex migrate:latest --knexfile knexfile.ts

# ロールバック（最新1件）
npx knex migrate:rollback --knexfile knexfile.ts

# 全てロールバック
npx knex migrate:rollback --all --knexfile knexfile.ts

# 特定のバージョンまでロールバック
npx knex migrate:rollback --to 20260103120000 --knexfile knexfile.ts

# ステータス確認
npx knex migrate:status --knexfile knexfile.ts

# マイグレーション一覧
npx knex migrate:list --knexfile knexfile.ts
```

### 高度なマイグレーション

```typescript
// migrations/20260103120100_add_full_text_search.ts
import type { Knex } from 'knex'

export async function up(knex: Knex): Promise<void> {
  // PostgreSQL全文検索カラム追加
  await knex.schema.alterTable('posts', (table) => {
    table.specificType('search_vector', 'tsvector')
  })

  // トリガー作成（生SQL）
  await knex.raw(`
    CREATE OR REPLACE FUNCTION posts_search_trigger() RETURNS trigger AS $$
    BEGIN
      NEW.search_vector :=
        setweight(to_tsvector('english', coalesce(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(NEW.content, '')), 'B');
      RETURN NEW;
    END
    $$ LANGUAGE plpgsql;

    CREATE TRIGGER posts_search_update
      BEFORE INSERT OR UPDATE ON posts
      FOR EACH ROW
      EXECUTE FUNCTION posts_search_trigger();
  `)

  // GINインデックス作成
  await knex.raw(`
    CREATE INDEX posts_search_idx ON posts USING GIN(search_vector)
  `)

  // 既存データの更新
  await knex.raw(`
    UPDATE posts
    SET search_vector =
      setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
      setweight(to_tsvector('english', coalesce(content, '')), 'B')
  `)
}

export async function down(knex: Knex): Promise<void> {
  await knex.raw(`DROP TRIGGER IF EXISTS posts_search_update ON posts`)
  await knex.raw(`DROP FUNCTION IF EXISTS posts_search_trigger()`)
  await knex.raw(`DROP INDEX IF EXISTS posts_search_idx`)

  await knex.schema.alterTable('posts', (table) => {
    table.dropColumn('search_vector')
  })
}
```

---

## ゼロダウンタイムマイグレーション

本番環境でサービスを停止せずにマイグレーションを適用する手法です。

### Expand-Contractパターン

後方互換性を維持しながら、段階的にスキーマを変更します。

#### ケース1: カラム名変更

```sql
-- ❌ ダウンタイム発生（従来の方法）
-- 1ステップで変更すると、古いコードがエラーになる
ALTER TABLE users RENAME COLUMN email TO email_address;

-- ✅ ゼロダウンタイム（Expand-Contractパターン）

-- === EXPAND PHASE ===
-- Migration 1: 新しいカラムを追加（NULL許可）
ALTER TABLE users ADD COLUMN email_address VARCHAR(255);

-- Migration 2: 既存データをコピー
UPDATE users SET email_address = email WHERE email_address IS NULL;

-- Migration 3: トリガーで両カラムを同期
CREATE OR REPLACE FUNCTION sync_email_columns()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.email IS DISTINCT FROM OLD.email THEN
    NEW.email_address = NEW.email;
  END IF;
  IF NEW.email_address IS DISTINCT FROM OLD.email_address THEN
    NEW.email = NEW.email_address;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER sync_email_before_update
BEFORE UPDATE ON users
FOR EACH ROW EXECUTE FUNCTION sync_email_columns();

-- Application Code Update 1: 両方のカラムに書き込み
-- await prisma.user.update({
--   where: { id },
--   data: { email: newEmail, emailAddress: newEmail }
-- })

-- Application Code Update 2: 新しいカラムから読み込み（フォールバック付き）
-- const email = user.emailAddress || user.email

-- === CONTRACT PHASE ===
-- Migration 4: NOT NULL制約を追加
ALTER TABLE users ALTER COLUMN email_address SET NOT NULL;

-- Migration 5: UNIQUE制約を追加
CREATE UNIQUE INDEX users_email_address_idx ON users(email_address);

-- Application Code Update 3: 新しいカラムのみ使用
-- await prisma.user.update({
--   where: { id },
--   data: { emailAddress: newEmail }
-- })

-- Migration 6: トリガー削除
DROP TRIGGER sync_email_before_update ON users;
DROP FUNCTION sync_email_columns();

-- Migration 7: 古いカラムを削除（十分な期間後、例: 2週間）
ALTER TABLE users DROP COLUMN email;
```

#### ケース2: NOT NULL制約追加

```sql
-- ❌ ダウンタイム発生
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NOT NULL;
-- 既存レコードにNULLが入るためエラー

-- ✅ ゼロダウンタイム
-- Migration 1: NULL許可で追加
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Migration 2: アプリケーションコード更新
-- 新規作成時に必ず phone を設定するように変更

-- Migration 3: デフォルト値を設定（既存データ用）
UPDATE users SET phone = '000-0000-0000' WHERE phone IS NULL;

-- Migration 4: NOT NULL制約を追加
ALTER TABLE users ALTER COLUMN phone SET NOT NULL;

-- Migration 5: デフォルト値を削除（任意）
ALTER TABLE users ALTER COLUMN phone DROP DEFAULT;
```

#### ケース3: インデックス作成

```sql
-- ❌ ダウンタイム発生（テーブルロック）
CREATE INDEX idx_posts_author_id ON posts(author_id);
-- 大きなテーブルでは数分〜数時間ロック

-- ✅ ゼロダウンタイム（PostgreSQL）
CREATE INDEX CONCURRENTLY idx_posts_author_id ON posts(author_id);
-- 書き込みを許可しながらインデックス作成

-- ✅ ゼロダウンタイム（MySQL）
ALTER TABLE posts
ADD INDEX idx_posts_author_id (author_id),
ALGORITHM=INPLACE,
LOCK=NONE;
```

#### ケース4: 外部キー制約追加

```sql
-- ❌ ダウンタイム発生（既存データのバリデーションでロック）
ALTER TABLE posts ADD CONSTRAINT fk_posts_author_id
FOREIGN KEY (author_id) REFERENCES users(id);

-- ✅ ゼロダウンタイム（PostgreSQL）
-- Step 1: NOT VALID で制約追加（既存データチェックなし）
ALTER TABLE posts ADD CONSTRAINT fk_posts_author_id
FOREIGN KEY (author_id) REFERENCES users(id) NOT VALID;

-- Step 2: バリデーション（バックグラウンドで実行、読み書き可能）
ALTER TABLE posts VALIDATE CONSTRAINT fk_posts_author_id;
```

### Blue-Greenデプロイ

アプリケーションとデータベースを同時に切り替える手法です。

```bash
#!/bin/bash
# Blue-Green デプロイスクリプト

echo "=== Blue-Green Deployment ==="

# 現在の環境
CURRENT_ENV="blue"
NEW_ENV="green"

# 1. Green環境にアプリケーションをデプロイ
echo "Deploying to $NEW_ENV environment..."
kubectl apply -f k8s/green-deployment.yaml

# 2. Green環境でマイグレーション適用
echo "Running migrations on $NEW_ENV..."
kubectl exec -it deployment/green-api -- npx prisma migrate deploy

# 3. Green環境のヘルスチェック
echo "Health checking $NEW_ENV environment..."
for i in {1..30}; do
  if curl -f http://green-api/health; then
    echo "$NEW_ENV is healthy"
    break
  fi
  echo "Waiting for $NEW_ENV to be ready..."
  sleep 2
done

# 4. ロードバランサーをGreen環境に切り替え
echo "Switching traffic to $NEW_ENV..."
kubectl patch service api-service -p '{"spec":{"selector":{"version":"green"}}}'

# 5. 動作確認（5分間監視）
echo "Monitoring $NEW_ENV for 5 minutes..."
sleep 300

# 6. エラー率確認
ERROR_RATE=$(curl -s http://monitoring/api/error-rate)
if [ "$ERROR_RATE" -lt 1 ]; then
  echo "Deployment successful. Shutting down $CURRENT_ENV..."
  kubectl delete deployment blue-api
else
  echo "High error rate detected. Rolling back to $CURRENT_ENV..."
  kubectl patch service api-service -p '{"spec":{"selector":{"version":"blue"}}}'
  kubectl delete deployment green-api
  exit 1
fi
```

---

## 本番環境へのデプロイ

本番環境でのマイグレーションは、慎重な計画と実行が必要です。

### デプロイ前チェックリスト

```markdown
## マイグレーション実行前チェックリスト

### 事前準備
- [ ] バックアップ作成（データベース全体）
- [ ] ステージング環境で事前テスト
- [ ] マイグレーション計画書作成
- [ ] ダウンタイム見積もり
- [ ] ロールバック手順の文書化
- [ ] チームメンバーへの通知

### 実行前確認
- [ ] メンテナンスモード設定（必要な場合）
- [ ] モニタリングツール準備
- [ ] アラート設定確認
- [ ] オンコール体制確認

### 実行
- [ ] マイグレーション実行
- [ ] 動作確認（主要機能）
- [ ] パフォーマンス確認
- [ ] エラーログ確認

### 実行後
- [ ] メンテナンスモード解除
- [ ] モニタリング（最低1時間）
- [ ] ユーザーへの通知
- [ ] デプロイ報告書作成
```

### バックアップ作成

```bash
#!/bin/bash
# backup.sh - データベースバックアップスクリプト

BACKUP_DIR="/mnt/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DB_NAME="mydb"
DB_USER="postgres"

echo "=== Database Backup Script ==="
echo "Date: $(date)"
echo "Database: $DB_NAME"

# PostgreSQL: フルバックアップ（カスタム形式）
pg_dump -U $DB_USER -d $DB_NAME -F c -f "$BACKUP_DIR/backup_$TIMESTAMP.dump"

# スキーマのみバックアップ
pg_dump -U $DB_USER -d $DB_NAME -s -f "$BACKUP_DIR/schema_$TIMESTAMP.sql"

# データのみバックアップ
pg_dump -U $DB_USER -d $DB_NAME -a -f "$BACKUP_DIR/data_$TIMESTAMP.sql"

# S3にアップロード（オプション）
aws s3 cp "$BACKUP_DIR/backup_$TIMESTAMP.dump" \
  s3://my-bucket/db-backups/backup_$TIMESTAMP.dump

echo "Backup completed: backup_$TIMESTAMP.dump"

# 30日以上古いバックアップを削除
find "$BACKUP_DIR" -name "backup_*.dump" -mtime +30 -delete

echo "Old backups cleaned up."
```

### マイグレーション実行スクリプト

```bash
#!/bin/bash
# deploy_migration.sh - 本番マイグレーション実行スクリプト

set -e  # エラー時に停止

echo "=== Database Migration Script ==="
echo "Date: $(date)"
echo "Environment: $ENVIRONMENT"
echo ""

# 環境変数チェック
if [ -z "$DATABASE_URL" ]; then
  echo "ERROR: DATABASE_URL is not set"
  exit 1
fi

# 1. バックアップ作成
echo "Step 1: Creating backup..."
BACKUP_FILE="backup_$(date +%Y%m%d_%H%M%S).dump"
pg_dump -U $DB_USER -d $DB_NAME -F c -f "/tmp/$BACKUP_FILE"

if [ $? -eq 0 ]; then
  echo "Backup created: /tmp/$BACKUP_FILE"
  # S3にアップロード
  aws s3 cp "/tmp/$BACKUP_FILE" "s3://my-bucket/db-backups/$BACKUP_FILE"
else
  echo "ERROR: Backup failed"
  exit 1
fi

echo ""

# 2. ドライラン（SQL生成）
echo "Step 2: Generating migration SQL..."
npx prisma migrate deploy --dry-run > /tmp/migration.sql

echo "Generated SQL:"
cat /tmp/migration.sql
echo ""

# 3. 確認プロンプト
read -p "Continue with migration? (yes/no) " -r
echo ""
if [[ ! $REPLY =~ ^[Yy]es$ ]]; then
  echo "Migration cancelled."
  exit 0
fi

# 4. マイグレーション実行
echo "Step 3: Running migrations..."
START_TIME=$(date +%s)

npx prisma migrate deploy

if [ $? -eq 0 ]; then
  END_TIME=$(date +%s)
  DURATION=$((END_TIME - START_TIME))
  echo "Migrations completed successfully in ${DURATION}s"
else
  echo "ERROR: Migration failed"
  echo "Restoring from backup..."
  pg_restore -U $DB_USER -d $DB_NAME -c "/tmp/$BACKUP_FILE"
  exit 1
fi

echo ""

# 5. ヘルスチェック
echo "Step 4: Health check..."
for i in {1..30}; do
  if curl -f http://localhost:3000/health; then
    echo "Health check passed"
    break
  fi

  if [ $i -eq 30 ]; then
    echo "ERROR: Health check failed"
    echo "Consider rollback"
    exit 1
  fi

  echo "Waiting for service to be ready..."
  sleep 2
done

echo ""

# 6. 完了通知
echo "=== Migration completed successfully ==="
echo "Backup: s3://my-bucket/db-backups/$BACKUP_FILE"
echo "Duration: ${DURATION}s"

# Slackに通知（オプション）
curl -X POST -H 'Content-type: application/json' \
  --data "{\"text\":\"Migration completed successfully in ${DURATION}s\"}" \
  $SLACK_WEBHOOK_URL
```

### CI/CDパイプライン統合

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Generate Prisma Client
        run: npx prisma generate

      - name: Backup Database
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: |
          npm run db:backup

      - name: Run Migrations
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: |
          npx prisma migrate deploy

      - name: Deploy Application
        run: |
          npm run deploy:production

      - name: Health Check
        run: |
          npm run health-check

      - name: Notify Slack
        if: always()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Deployment ${{ job.status }}'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

---

## シードデータ

開発環境やテスト環境で初期データを投入する仕組みです。

### Prismaシード

```typescript
// prisma/seed.ts
import { PrismaClient } from '@prisma/client'
import { hash } from 'bcrypt'

const prisma = new PrismaClient()

async function main() {
  console.log('Seeding database...')

  // ユーザー作成
  const adminPassword = await hash('admin123', 12)
  const userPassword = await hash('user123', 12)

  const admin = await prisma.user.upsert({
    where: { email: 'admin@example.com' },
    update: {},
    create: {
      email: 'admin@example.com',
      username: 'admin',
      password: adminPassword,
      status: 'active',
      profile: {
        create: {
          bio: 'System Administrator',
          avatar: 'https://example.com/avatars/admin.png',
        },
      },
    },
  })

  const user1 = await prisma.user.upsert({
    where: { email: 'john@example.com' },
    update: {},
    create: {
      email: 'john@example.com',
      username: 'john_doe',
      password: userPassword,
      status: 'active',
      profile: {
        create: {
          bio: 'Software Engineer',
          avatar: 'https://example.com/avatars/john.png',
          website: 'https://johndoe.com',
        },
      },
    },
  })

  const user2 = await prisma.user.upsert({
    where: { email: 'jane@example.com' },
    update: {},
    create: {
      email: 'jane@example.com',
      username: 'jane_smith',
      password: userPassword,
      status: 'active',
    },
  })

  console.log({ admin, user1, user2 })

  // 投稿作成
  const posts = await Promise.all([
    prisma.post.create({
      data: {
        title: 'Welcome to our platform',
        content: 'This is the first post! We are excited to have you here.',
        published: true,
        publishedAt: new Date(),
        authorId: admin.id,
      },
    }),
    prisma.post.create({
      data: {
        title: 'Getting Started with Prisma',
        content: 'Prisma is a modern database toolkit that makes database access easy.',
        published: true,
        publishedAt: new Date(),
        authorId: user1.id,
      },
    }),
    prisma.post.create({
      data: {
        title: 'My First Post',
        content: 'Hello world! This is my first post.',
        published: false,
        authorId: user2.id,
      },
    }),
  ])

  console.log({ posts })

  console.log('Seeding completed!')
}

main()
  .catch((e) => {
    console.error('Error seeding database:', e)
    process.exit(1)
  })
  .finally(async () => {
    await prisma.$disconnect()
  })
```

```json
// package.json
{
  "name": "my-app",
  "prisma": {
    "seed": "ts-node prisma/seed.ts"
  },
  "scripts": {
    "db:seed": "npx prisma db seed"
  }
}
```

```bash
# シード実行
npm run db:seed
```

### TypeORMシード

```typescript
// src/seeds/01_users.seed.ts
import { DataSource } from 'typeorm'
import { User } from '../entities/User'
import { Profile } from '../entities/Profile'
import { hash } from 'bcrypt'

export async function seedUsers(dataSource: DataSource): Promise<void> {
  const userRepository = dataSource.getRepository(User)
  const profileRepository = dataSource.getRepository(Profile)

  // 既存データをクリア（開発環境のみ）
  if (process.env.NODE_ENV === 'development') {
    await profileRepository.delete({})
    await userRepository.delete({})
  }

  // ユーザー作成
  const adminPassword = await hash('admin123', 12)
  const admin = userRepository.create({
    email: 'admin@example.com',
    username: 'admin',
    password: adminPassword,
    status: 'active',
  })
  await userRepository.save(admin)

  // プロフィール作成
  const adminProfile = profileRepository.create({
    userId: admin.id,
    bio: 'System Administrator',
    avatar: 'https://example.com/avatars/admin.png',
  })
  await profileRepository.save(adminProfile)

  console.log('Users seeded successfully')
}
```

```typescript
// src/seeds/index.ts
import { AppDataSource } from '../data-source'
import { seedUsers } from './01_users.seed'
import { seedPosts } from './02_posts.seed'

async function runSeeds() {
  try {
    await AppDataSource.initialize()
    console.log('Data Source initialized')

    await seedUsers(AppDataSource)
    await seedPosts(AppDataSource)

    console.log('All seeds completed successfully')
  } catch (error) {
    console.error('Error running seeds:', error)
    process.exit(1)
  } finally {
    await AppDataSource.destroy()
  }
}

runSeeds()
```

```json
// package.json
{
  "scripts": {
    "seed": "ts-node src/seeds/index.ts"
  }
}
```

### Knex.jsシード

```typescript
// seeds/01_users.ts
import { Knex } from 'knex'
import { hash } from 'bcrypt'

export async function seed(knex: Knex): Promise<void> {
  // テーブルをクリア
  await knex('profiles').del()
  await knex('posts').del()
  await knex('users').del()

  // ユーザー挿入
  const adminPassword = await hash('admin123', 12)
  const userPassword = await hash('user123', 12)

  const [admin, user1, user2] = await knex('users')
    .insert([
      {
        email: 'admin@example.com',
        username: 'admin',
        password: adminPassword,
        status: 'active',
      },
      {
        email: 'john@example.com',
        username: 'john_doe',
        password: userPassword,
        status: 'active',
      },
      {
        email: 'jane@example.com',
        username: 'jane_smith',
        password: userPassword,
        status: 'active',
      },
    ])
    .returning('*')

  // プロフィール挿入
  await knex('profiles').insert([
    {
      user_id: admin.id,
      bio: 'System Administrator',
      avatar: 'https://example.com/avatars/admin.png',
    },
    {
      user_id: user1.id,
      bio: 'Software Engineer',
      avatar: 'https://example.com/avatars/john.png',
      website: 'https://johndoe.com',
    },
  ])

  // 投稿挿入
  await knex('posts').insert([
    {
      title: 'Welcome to our platform',
      content: 'This is the first post!',
      published: true,
      published_at: new Date(),
      author_id: admin.id,
    },
    {
      title: 'Getting Started',
      content: 'Here is how to get started...',
      published: true,
      published_at: new Date(),
      author_id: user1.id,
    },
  ])
}
```

```bash
# シード実行
npx knex seed:run --knexfile knexfile.ts
```

---

## よくあるトラブルと解決策

### 1. マイグレーション適用順序が間違っている

**症状:** 外部キー制約エラー

```sql
-- ❌ 参照先テーブルが存在しない
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id)  -- usersテーブルがまだ存在しない
);

-- ✅ 正しい順序
CREATE TABLE users (
  id SERIAL PRIMARY KEY
);

CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id)
);
```

### 2. マイグレーションが途中で失敗

**症状:** 一部のテーブルのみ作成され、不整合が発生

**解決策:** トランザクション使用

```typescript
// TypeORM
export class SafeMigration1704297600000 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.startTransaction()

    try {
      await queryRunner.query(`CREATE TABLE users (...)`)
      await queryRunner.query(`CREATE TABLE posts (...)`)

      await queryRunner.commitTransaction()
    } catch (error) {
      await queryRunner.rollbackTransaction()
      throw error
    }
  }
}
```

### 3. 本番データを失う

**症状:** DROP TABLEで既存データが消える

**解決策:** バックアップ作成

```bash
# ✅ 本番環境では必ずバックアップ
pg_dump -U postgres -d mydb -F c -f backup_before_migration.dump

# マイグレーション適用
npx prisma migrate deploy

# 問題があれば復元
pg_restore -U postgres -d mydb -c backup_before_migration.dump
```

### 4. カラム名変更で既存データが消える

**症状:** RENAME COLUMNのつもりが、DROP → ADDで実行される

**解決策:**

```sql
-- ❌ データ消失のリスク（ツールによってはこう解釈される）
ALTER TABLE users DROP COLUMN email;
ALTER TABLE users ADD COLUMN email_address VARCHAR(255);

-- ✅ RENAME使用
ALTER TABLE users RENAME COLUMN email TO email_address;
```

### 5. NOT NULL制約違反

**症状:** 既存データがNULLで制約追加に失敗

**解決策:**

```sql
-- ❌ 既存データがNULLで失敗
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NOT NULL;

-- ✅ 段階的に追加
-- Step 1: NULL許可で追加
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Step 2: デフォルト値を設定
UPDATE users SET phone = '000-0000-0000' WHERE phone IS NULL;

-- Step 3: NOT NULL制約追加
ALTER TABLE users ALTER COLUMN phone SET NOT NULL;
```

### 6. インデックス作成でロックが長時間発生

**症状:** CREATE INDEXで長時間テーブルロック

**解決策:**

```sql
-- ❌ テーブルロック（書き込み不可）
CREATE INDEX idx_posts_author_id ON posts(author_id);

-- ✅ CONCURRENTLY使用（PostgreSQL）
CREATE INDEX CONCURRENTLY idx_posts_author_id ON posts(author_id);
-- 書き込みを許可しながらインデックス作成
```

### 7. マイグレーション履歴の不整合

**症状:** 開発環境と本番環境でマイグレーション履歴が異なる

**解決策:**

```bash
# ✅ ベースラインを作成（Prisma）
npx prisma migrate resolve --applied "20260103120000_initial"

# ✅ 環境ごとにマイグレーション履歴を確認
npx prisma migrate status
```

### 8. ロールバックができない

**症状:** Prismaにロールバック機能がない

**解決策:**

```typescript
// ✅ 逆マイグレーションを作成
// migrations/20260103120001_remove_profile_table/migration.sql
DROP TABLE "profiles";
```

```bash
# または、手動でロールバック
npx prisma migrate resolve --rolled-back "20260103120000_add_profile_table"
```

### 9. 環境変数の設定ミス

**症状:** DATABASE_URLが間違っていてマイグレーション失敗

**解決策:**

```bash
# ✅ 環境変数を確認
echo $DATABASE_URL

# ✅ .envファイルを確認
cat .env

# ✅ 接続テスト
npx prisma db pull
```

### 10. 大量データでマイグレーションがタイムアウト

**症状:** データ移行で時間がかかりすぎる

**解決策:** バッチ処理（前述のTypeORMのデータマイグレーション参照）

---

## ベンチマーク指標

### 想定シナリオ: ECサイトのマイグレーション管理改善効果

#### 導入前の想定課題
- スキーマ変更が手動で、環境間で不整合が頻発
- 本番環境でのマイグレーション失敗: 年3回
- データ消失のインシデント: 年2回
- マイグレーション時間: 平均25分（ダウンタイム）
- ロールバック: 手動で平均30分

#### 導入後に期待できる効果（6ヶ月想定）

| 指標 | 導入前 | 導入後 | 改善率 |
|------|--------|--------|--------|
| 環境間の不整合 | 15件/月 | 0件 | **-100%** |
| マイグレーション失敗 | 年3回 | 年0回 | **-100%** |
| データ消失インシデント | 年2回 | 年0回 | **-100%** |
| マイグレーションダウンタイム | 25分 | 0分 | **-100%** |
| ロールバック時間 | 30分 | 2分 | **-93%** |
| スキーマ変更時間 | 45分 | 5分 | **-89%** |

#### 想定される改善策

1. **Prisma Migrate導入** - スキーマ変更の自動化
2. **ゼロダウンタイムマイグレーション** - Expand-Contractパターン
3. **自動バックアップ** - マイグレーション前に自動バックアップ
4. **CI/CDパイプライン統合** - GitHub ActionsでマイグレーションをCI/CDに組み込み
5. **マイグレーション計画書テンプレート** - 本番デプロイ手順を標準化

---

## まとめ

### マイグレーション管理の成功の鍵

1. **ツール選定** - プロジェクトに合ったマイグレーションツールを選択
2. **バージョン管理** - マイグレーションファイルをGitで管理
3. **ゼロダウンタイム** - 後方互換性を維持した段階的変更
4. **バックアップ** - 本番環境では必ずバックアップを取る
5. **自動化** - CI/CDパイプラインに組み込む

### 次のステップ

1. **今すぐ実装**: Prisma Migrate導入
2. **ゼロダウンタイム**: Expand-Contractパターンの実践
3. **バックアップ**: 自動バックアップスクリプトの作成
4. **CI/CD統合**: GitHub Actionsでマイグレーション自動化
5. **ドキュメント化**: マイグレーション計画書のテンプレート作成

### 参考資料

- [Prisma Migrate Documentation](https://www.prisma.io/docs/concepts/components/prisma-migrate)
- [TypeORM Migrations](https://typeorm.io/migrations)
- [Knex.js Migrations](https://knexjs.org/guide/migrations.html)
- [Zero-Downtime Database Migrations](https://spring.io/blog/2016/05/31/zero-downtime-deployment-with-a-database)

---

**文字数: 約27,500文字**
