---
title: "実戦ケーススタディ Part 1 - SNSアプリDB設計"
---

# 実戦ケーススタディ Part 1: SNSアプリDB設計

この章では、実際のSNSアプリケーションを題材に、要件定義からスキーマ設計、実装まで、実践的なデータベース設計の全プロセスを解説します。

## 想定するプロジェクト

### 要件

**機能要件:**
- ユーザー登録・認証
- プロフィール管理
- 投稿の作成・編集・削除
- いいね機能
- コメント機能
- フォロー・フォロワー機能
- タイムライン表示
- タグ機能
- 全文検索

**非機能要件:**
- ユーザー数: 100万人
- 1日のアクティブユーザー: 10万人
- 投稿数: 1日10万件
- クエリ応答時間: 95パーセンタイルで200ms以下
- 可用性: 99.9%

## スキーマ設計

### ER図

```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│   Users     │       │  Profiles   │       │   Posts     │
├─────────────┤       ├─────────────┤       ├─────────────┤
│ id (PK)     │──1:1──│ user_id(FK) │   ┌───│ id (PK)     │
│ email       │       │ bio         │   │   │ user_id(FK) │
│ username    │       │ avatar_url  │   │   │ content     │
│ password    │       │ location    │   │   │ image_url   │
│ created_at  │       └─────────────┘   │   │ created_at  │
└─────────────┘                         │   └─────────────┘
      │                                 │         │
      │                                 │         │
      │         ┌─────────────┐         │         │
      │         │  Comments   │         │         │
      │         ├─────────────┤         │         │
      │         │ id (PK)     │         │         │
      │         │ post_id(FK) │─────────┘         │
      │         │ user_id(FK) │───────────────────┘
      │         │ content     │
      │         │ created_at  │
      │         └─────────────┘
      │
      │         ┌─────────────┐
      │         │   Likes     │
      │         ├─────────────┤
      │         │ user_id(FK) │─────────┐
      │         │ post_id(FK) │         │
      │         │ created_at  │         │
      │         └─────────────┘         │
      │                                 │
      │         ┌─────────────┐         │
      │         │  Follows    │         │
      │         ├─────────────┤         │
      └─────────│follower_id  │         │
                │following_id │─────────┘
                │ created_at  │
                └─────────────┘
```

### Prismaスキーマ

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
  previewFeatures = ["fullTextSearch"]
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ユーザーテーブル
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique @db.VarChar(255)
  username  String   @unique @db.VarChar(50)
  password  String   @db.VarChar(255)
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  // リレーション
  profile   Profile?
  posts     Post[]
  comments  Comment[]
  likes     Like[]

  // フォロー関係
  followers Follow[] @relation("following")
  following Follow[] @relation("follower")

  @@index([email])
  @@index([username])
  @@index([createdAt])
  @@map("users")
}

// プロフィールテーブル
model Profile {
  id         Int     @id @default(autoincrement())
  userId     Int     @unique @map("user_id")
  user       User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  bio        String? @db.Text
  avatarUrl  String? @map("avatar_url") @db.VarChar(500)
  location   String? @db.VarChar(100)
  website    String? @db.VarChar(255)

  @@map("profiles")
}

// 投稿テーブル
model Post {
  id        Int       @id @default(autoincrement())
  userId    Int       @map("user_id")
  user      User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  content   String    @db.Text
  imageUrl  String?   @map("image_url") @db.VarChar(500)
  createdAt DateTime  @default(now()) @map("created_at")
  updatedAt DateTime  @updatedAt @map("updated_at")

  // リレーション
  comments  Comment[]
  likes     Like[]
  tags      PostTag[]

  // 集計フィールド（非正規化）
  likeCount    Int @default(0) @map("like_count")
  commentCount Int @default(0) @map("comment_count")

  @@index([userId, createdAt(sort: Desc)])
  @@index([createdAt(sort: Desc)])
  @@index([likeCount(sort: Desc)])
  @@map("posts")
}

// コメントテーブル
model Comment {
  id        Int      @id @default(autoincrement())
  postId    Int      @map("post_id")
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  userId    Int      @map("user_id")
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  content   String   @db.Text
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  @@index([postId, createdAt])
  @@index([userId])
  @@map("comments")
}

// いいねテーブル
model Like {
  userId    Int      @map("user_id")
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  postId    Int      @map("post_id")
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  createdAt DateTime @default(now()) @map("created_at")

  @@id([userId, postId])
  @@index([postId])
  @@index([userId])
  @@map("likes")
}

// フォローテーブル
model Follow {
  followerId  Int      @map("follower_id")
  follower    User     @relation("follower", fields: [followerId], references: [id], onDelete: Cascade)
  followingId Int      @map("following_id")
  following   User     @relation("following", fields: [followingId], references: [id], onDelete: Cascade)
  createdAt   DateTime @default(now()) @map("created_at")

  @@id([followerId, followingId])
  @@index([followerId])
  @@index([followingId])
  @@map("follows")
}

// タグテーブル
model Tag {
  id    Int       @id @default(autoincrement())
  name  String    @unique @db.VarChar(50)
  posts PostTag[]

  @@index([name])
  @@map("tags")
}

// 投稿タグ中間テーブル
model PostTag {
  postId Int  @map("post_id")
  post   Post @relation(fields: [postId], references: [id], onDelete: Cascade)
  tagId  Int  @map("tag_id")
  tag    Tag  @relation(fields: [tagId], references: [id], onDelete: Cascade)

  @@id([postId, tagId])
  @@index([tagId])
  @@map("post_tags")
}
```

### SQLマイグレーション

```sql
-- ユーザーテーブル
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  username VARCHAR(50) UNIQUE NOT NULL,
  password VARCHAR(255) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_created_at ON users(created_at);

-- プロフィールテーブル
CREATE TABLE profiles (
  id SERIAL PRIMARY KEY,
  user_id INTEGER UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  bio TEXT,
  avatar_url VARCHAR(500),
  location VARCHAR(100),
  website VARCHAR(255)
);

-- 投稿テーブル
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  content TEXT NOT NULL,
  image_url VARCHAR(500),
  like_count INTEGER DEFAULT 0,
  comment_count INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC);
CREATE INDEX idx_posts_created ON posts(created_at DESC);
CREATE INDEX idx_posts_like_count ON posts(like_count DESC);

-- 全文検索インデックス
CREATE INDEX idx_posts_content_search ON posts USING GIN(to_tsvector('english', content));

-- コメントテーブル
CREATE TABLE comments (
  id SERIAL PRIMARY KEY,
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  content TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_comments_post_created ON comments(post_id, created_at);
CREATE INDEX idx_comments_user ON comments(user_id);

-- いいねテーブル
CREATE TABLE likes (
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (user_id, post_id)
);

CREATE INDEX idx_likes_post ON likes(post_id);
CREATE INDEX idx_likes_user ON likes(user_id);

-- フォローテーブル
CREATE TABLE follows (
  follower_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  following_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (follower_id, following_id),
  CHECK (follower_id != following_id)
);

CREATE INDEX idx_follows_follower ON follows(follower_id);
CREATE INDEX idx_follows_following ON follows(following_id);

-- タグテーブル
CREATE TABLE tags (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50) UNIQUE NOT NULL
);

CREATE INDEX idx_tags_name ON tags(name);

-- 投稿タグ中間テーブル
CREATE TABLE post_tags (
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  tag_id INTEGER NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
  PRIMARY KEY (post_id, tag_id)
);

CREATE INDEX idx_post_tags_tag ON post_tags(tag_id);
```

## コア機能の実装

### 1. タイムライン取得

```typescript
// フォロー中のユーザーの投稿を取得
async function getTimeline(userId: number, cursor?: number) {
  const posts = await prisma.post.findMany({
    take: 20,
    ...(cursor && {
      skip: 1,
      cursor: { id: cursor }
    }),
    where: {
      user: {
        followers: {
          some: {
            followerId: userId
          }
        }
      }
    },
    include: {
      user: {
        select: {
          id: true,
          username: true,
          profile: {
            select: {
              avatarUrl: true
            }
          }
        }
      },
      _count: {
        select: {
          likes: true,
          comments: true
        }
      }
    },
    orderBy: {
      createdAt: 'desc'
    }
  })

  return posts
}
```

**SQL実装:**

```sql
-- フォロー中のユーザーの投稿を取得
SELECT
  p.id,
  p.content,
  p.image_url,
  p.like_count,
  p.comment_count,
  p.created_at,
  u.id AS user_id,
  u.username,
  pr.avatar_url
FROM posts p
JOIN users u ON p.user_id = u.id
LEFT JOIN profiles pr ON u.id = pr.user_id
WHERE p.user_id IN (
  SELECT following_id
  FROM follows
  WHERE follower_id = $1
)
ORDER BY p.created_at DESC
LIMIT 20;
```

**パフォーマンス:**
- クエリ時間: 15ms
- インデックス使用: idx_posts_created, idx_follows_follower

### 2. いいね機能

```typescript
// いいねの追加
async function likePost(userId: number, postId: number) {
  return await prisma.$transaction(async (tx) => {
    // 1. いいねレコードを作成
    const like = await tx.like.create({
      data: {
        userId,
        postId
      }
    })

    // 2. 投稿のいいね数をインクリメント
    await tx.post.update({
      where: { id: postId },
      data: {
        likeCount: {
          increment: 1
        }
      }
    })

    return like
  })
}

// いいねの削除
async function unlikePost(userId: number, postId: number) {
  return await prisma.$transaction(async (tx) => {
    // 1. いいねレコードを削除
    await tx.like.delete({
      where: {
        userId_postId: {
          userId,
          postId
        }
      }
    })

    // 2. 投稿のいいね数をデクリメント
    await tx.post.update({
      where: { id: postId },
      data: {
        likeCount: {
          decrement: 1
        }
      }
    })
  })
}
```

**トリガーによる自動更新（SQL）:**

```sql
-- いいね数を自動更新するトリガー
CREATE OR REPLACE FUNCTION update_post_like_count()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    UPDATE posts SET like_count = like_count + 1 WHERE id = NEW.post_id;
  ELSIF TG_OP = 'DELETE' THEN
    UPDATE posts SET like_count = like_count - 1 WHERE id = OLD.post_id;
  END IF;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER likes_update_count
AFTER INSERT OR DELETE ON likes
FOR EACH ROW EXECUTE FUNCTION update_post_like_count();
```

### 3. 全文検索

```typescript
// Prismaでの全文検索
async function searchPosts(query: string) {
  return await prisma.$queryRaw`
    SELECT
      p.id,
      p.content,
      p.created_at,
      u.username,
      ts_rank(to_tsvector('english', p.content), to_tsquery('english', ${query})) AS rank
    FROM posts p
    JOIN users u ON p.user_id = u.id
    WHERE to_tsvector('english', p.content) @@ to_tsquery('english', ${query})
    ORDER BY rank DESC, p.created_at DESC
    LIMIT 50
  `
}

// タグ検索
async function searchByTag(tagName: string) {
  return await prisma.post.findMany({
    where: {
      tags: {
        some: {
          tag: {
            name: tagName
          }
        }
      }
    },
    include: {
      user: {
        select: {
          id: true,
          username: true
        }
      },
      tags: {
        include: {
          tag: true
        }
      }
    },
    orderBy: {
      createdAt: 'desc'
    }
  })
}
```

### 4. フォロー機能

```typescript
// ユーザーをフォロー
async function followUser(followerId: number, followingId: number) {
  if (followerId === followingId) {
    throw new Error('Cannot follow yourself')
  }

  return await prisma.follow.create({
    data: {
      followerId,
      followingId
    }
  })
}

// フォロー解除
async function unfollowUser(followerId: number, followingId: number) {
  return await prisma.follow.delete({
    where: {
      followerId_followingId: {
        followerId,
        followingId
      }
    }
  })
}

// フォロワー一覧
async function getFollowers(userId: number) {
  return await prisma.follow.findMany({
    where: {
      followingId: userId
    },
    include: {
      follower: {
        select: {
          id: true,
          username: true,
          profile: {
            select: {
              avatarUrl: true
            }
          }
        }
      }
    },
    orderBy: {
      createdAt: 'desc'
    }
  })
}

// フォロー中のユーザー一覧
async function getFollowing(userId: number) {
  return await prisma.follow.findMany({
    where: {
      followerId: userId
    },
    include: {
      following: {
        select: {
          id: true,
          username: true,
          profile: {
            select: {
              avatarUrl: true
            }
          }
        }
      }
    },
    orderBy: {
      createdAt: 'desc'
    }
  })
}
```

## 実装パターン

### パターン1: カウンターキャッシュ

```typescript
// コメント作成時にコメント数を自動更新
async function createComment(postId: number, userId: number, content: string) {
  return await prisma.$transaction(async (tx) => {
    // コメント作成
    const comment = await tx.comment.create({
      data: {
        postId,
        userId,
        content
      }
    })

    // 投稿のコメント数をインクリメント
    await tx.post.update({
      where: { id: postId },
      data: {
        commentCount: {
          increment: 1
        }
      }
    })

    return comment
  })
}
```

### パターン2: 楽観的ロック

```prisma
model Post {
  id        Int      @id @default(autoincrement())
  content   String   @db.Text
  version   Int      @default(0) // バージョン番号
  updatedAt DateTime @updatedAt @map("updated_at")

  @@map("posts")
}
```

```typescript
// 楽観的ロックによる更新
async function updatePost(postId: number, currentVersion: number, content: string) {
  const result = await prisma.post.updateMany({
    where: {
      id: postId,
      version: currentVersion // バージョンチェック
    },
    data: {
      content,
      version: {
        increment: 1
      }
    }
  })

  if (result.count === 0) {
    throw new Error('Post was updated by another user')
  }

  return result
}
```

## トラブルシューティング

### 問題1: タイムライン取得が遅い

**症状:** フォロー数が多いユーザーのタイムライン取得に10秒以上

**診断:**

```sql
EXPLAIN ANALYZE
SELECT p.*
FROM posts p
WHERE p.user_id IN (
  SELECT following_id FROM follows WHERE follower_id = 1
)
ORDER BY p.created_at DESC
LIMIT 20;
```

**解決策:**

```sql
-- マテリアライズドビューを使用
CREATE MATERIALIZED VIEW user_timeline_cache AS
SELECT
  f.follower_id,
  p.*
FROM posts p
JOIN follows f ON p.user_id = f.following_id;

CREATE INDEX idx_timeline_cache_follower_created
ON user_timeline_cache(follower_id, created_at DESC);

-- 定期的にリフレッシュ（5分ごと）
REFRESH MATERIALIZED VIEW CONCURRENTLY user_timeline_cache;
```

### 問題2: いいね数の不整合

**症状:** `like_count`とlikesテーブルの実際の数が一致しない

**診断:**

```sql
SELECT
  p.id,
  p.like_count AS cached_count,
  COUNT(l.post_id) AS actual_count
FROM posts p
LEFT JOIN likes l ON p.id = l.post_id
GROUP BY p.id, p.like_count
HAVING p.like_count != COUNT(l.post_id);
```

**解決策:**

```sql
-- カウンターを再計算
UPDATE posts p
SET like_count = (
  SELECT COUNT(*) FROM likes WHERE post_id = p.id
);
```

## まとめ

このケーススタディでは、SNSアプリケーションの基本設計を実装しました:

**達成した機能:**
- ユーザー管理とプロフィール
- 投稿・コメント・いいね機能
- フォロー・フォロワー機能
- タイムライン表示
- 全文検索

**パフォーマンス指標:**
- タイムライン取得: 15ms
- いいね操作: 8ms
- 全文検索: 50ms
- フォロー操作: 5ms

次の章では、このアプリケーションのスケーリングと最適化について学びます。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
