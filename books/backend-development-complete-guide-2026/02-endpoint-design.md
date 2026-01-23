---
title: "エンドポイント設計とベストプラクティス"
---

# エンドポイント設計とベストプラクティス

## この章で学ぶこと

第1章でREST APIの基礎原則を学びました。この章では、実際のプロダクション環境で必要となる、より高度なエンドポイント設計パターンとベストプラクティスを学習します。

- 複雑なリソース関係の設計
- バッチ操作とバルク処理
- 高度な検索・フィルタリング
- ファイルアップロード/ダウンロード
- 非同期処理とジョブ管理
- Webhook実装
- APIレート制限
- リクエスト/レスポンスバリデーション
- セキュリティベストプラクティス

## 複雑なリソース関係の設計

実際のアプリケーションでは、リソース間に複雑な関係が存在します。これらを適切に設計することが重要です。

### 1対多の関係

ユーザーと投稿の関係など、最も一般的なパターンです。

```typescript
// prisma/schema.prisma
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String
  posts     Post[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Post {
  id        String   @id @default(cuid())
  title     String
  content   String
  published Boolean  @default(false)
  authorId  String
  author    User     @relation(fields: [authorId], references: [id])
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

#### エンドポイント設計の選択肢

**オプション1: ネストされたルート（推奨）**

```typescript
// src/routes/user.routes.ts
import { Router } from 'express'
import { PrismaClient } from '@prisma/client'

const router = Router()
const prisma = new PrismaClient()

// GET /users/:userId/posts - 特定ユーザーの投稿一覧
router.get('/:userId/posts', async (req, res) => {
  const { userId } = req.params
  const page = parseInt(req.query.page as string) || 1
  const limit = parseInt(req.query.limit as string) || 10
  const skip = (page - 1) * limit

  const [posts, total] = await Promise.all([
    prisma.post.findMany({
      where: { authorId: userId },
      skip,
      take: limit,
      orderBy: { createdAt: 'desc' },
      include: {
        author: {
          select: {
            id: true,
            name: true,
            email: true,
          },
        },
      },
    }),
    prisma.post.count({ where: { authorId: userId } }),
  ])

  res.json({
    success: true,
    data: posts,
    meta: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit),
    },
  })
})

// POST /users/:userId/posts - 特定ユーザーの投稿作成
router.post('/:userId/posts', async (req, res) => {
  const { userId } = req.params
  const { title, content, published } = req.body

  // ユーザーの存在確認
  const user = await prisma.user.findUnique({
    where: { id: userId },
  })

  if (!user) {
    return res.status(404).json({
      success: false,
      error: {
        code: 'USER_NOT_FOUND',
        message: 'User not found',
      },
    })
  }

  const post = await prisma.post.create({
    data: {
      title,
      content,
      published: published || false,
      authorId: userId,
    },
    include: {
      author: {
        select: {
          id: true,
          name: true,
          email: true,
        },
      },
    },
  })

  res.status(201)
    .setHeader('Location', `/posts/${post.id}`)
    .json({
      success: true,
      data: post,
    })
})

export default router
```

**オプション2: クエリパラメータによるフィルタリング**

```typescript
// GET /posts?authorId=123
router.get('/', async (req, res) => {
  const authorId = req.query.authorId as string
  const page = parseInt(req.query.page as string) || 1
  const limit = parseInt(req.query.limit as string) || 10
  const skip = (page - 1) * limit

  const where = authorId ? { authorId } : {}

  const [posts, total] = await Promise.all([
    prisma.post.findMany({
      where,
      skip,
      take: limit,
      orderBy: { createdAt: 'desc' },
      include: {
        author: {
          select: {
            id: true,
            name: true,
            email: true,
          },
        },
      },
    }),
    prisma.post.count({ where }),
  ])

  res.json({
    success: true,
    data: posts,
    meta: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit),
    },
  })
})
```

**どちらを選ぶべきか？**

| 状況 | 推奨 |
|-----|------|
| 親リソースのコンテキストが重要 | ネストされたルート |
| 柔軟なフィルタリングが必要 | クエリパラメータ |
| 複数の親からの取得が必要 | クエリパラメータ |
| RESTful性を重視 | ネストされたルート |

### 多対多の関係

記事とタグの関係など。

```typescript
// prisma/schema.prisma
model Post {
  id        String     @id @default(cuid())
  title     String
  content   String
  tags      PostTag[]
  createdAt DateTime   @default(now())
}

model Tag {
  id    String    @id @default(cuid())
  name  String    @unique
  posts PostTag[]
}

model PostTag {
  postId String
  tagId  String
  post   Post   @relation(fields: [postId], references: [id])
  tag    Tag    @relation(fields: [tagId], references: [id])

  @@id([postId, tagId])
}
```

```typescript
// src/routes/post.routes.ts
import { Router } from 'express'
import { PrismaClient } from '@prisma/client'

const router = Router()
const prisma = new PrismaClient()

// GET /posts/:postId/tags - 投稿のタグ一覧
router.get('/:postId/tags', async (req, res) => {
  const { postId } = req.params

  const post = await prisma.post.findUnique({
    where: { id: postId },
    include: {
      tags: {
        include: {
          tag: true,
        },
      },
    },
  })

  if (!post) {
    return res.status(404).json({
      success: false,
      error: {
        code: 'POST_NOT_FOUND',
        message: 'Post not found',
      },
    })
  }

  const tags = post.tags.map((pt) => pt.tag)

  res.json({
    success: true,
    data: tags,
  })
})

// POST /posts/:postId/tags - 投稿にタグを追加
router.post('/:postId/tags', async (req, res) => {
  const { postId } = req.params
  const { tagIds } = req.body // 配列で複数のタグIDを受け取る

  // トランザクションで複数のタグを追加
  const result = await prisma.$transaction(
    tagIds.map((tagId: string) =>
      prisma.postTag.create({
        data: {
          postId,
          tagId,
        },
        include: {
          tag: true,
        },
      })
    )
  )

  res.status(201).json({
    success: true,
    data: result.map((pt) => pt.tag),
  })
})

// DELETE /posts/:postId/tags/:tagId - 投稿からタグを削除
router.delete('/:postId/tags/:tagId', async (req, res) => {
  const { postId, tagId } = req.params

  try {
    await prisma.postTag.delete({
      where: {
        postId_tagId: {
          postId,
          tagId,
        },
      },
    })

    res.status(204).send()
  } catch (error) {
    res.status(404).json({
      success: false,
      error: {
        code: 'ASSOCIATION_NOT_FOUND',
        message: 'Post-Tag association not found',
      },
    })
  }
})

export default router
```

### 深いネスト構造の回避

3階層以上のネストは避けるべきです。

**❌ 避けるべき設計:**
```
GET /users/:userId/posts/:postId/comments/:commentId/likes
```

**✅ 推奨される設計:**
```
GET /comments/:commentId/likes
GET /likes?commentId=123
```

```typescript
// src/routes/comment.routes.ts
router.get('/:commentId/likes', async (req, res) => {
  const { commentId } = req.params

  const likes = await prisma.like.findMany({
    where: { commentId },
    include: {
      user: {
        select: {
          id: true,
          name: true,
        },
      },
    },
  })

  res.json({
    success: true,
    data: likes,
  })
})
```

## バッチ操作とバルク処理

複数のリソースを一度に処理する場合のパターンです。

### バッチ作成

```typescript
// POST /posts/batch
interface BatchCreatePostDto {
  posts: Array<{
    title: string
    content: string
    published?: boolean
  }>
}

router.post('/batch', async (req, res) => {
  const { posts }: BatchCreatePostDto = req.body

  // バリデーション
  if (!Array.isArray(posts) || posts.length === 0) {
    return res.status(400).json({
      success: false,
      error: {
        code: 'INVALID_INPUT',
        message: 'Posts array is required and must not be empty',
      },
    })
  }

  // 最大件数制限
  if (posts.length > 100) {
    return res.status(400).json({
      success: false,
      error: {
        code: 'BATCH_SIZE_EXCEEDED',
        message: 'Maximum 100 posts can be created at once',
      },
    })
  }

  try {
    // トランザクションで一括作成
    const result = await prisma.$transaction(
      posts.map((post) =>
        prisma.post.create({
          data: {
            title: post.title,
            content: post.content,
            published: post.published || false,
            authorId: req.user!.userId,
          },
        })
      )
    )

    res.status(201).json({
      success: true,
      data: result,
      meta: {
        created: result.length,
      },
    })
  } catch (error) {
    res.status(500).json({
      success: false,
      error: {
        code: 'BATCH_CREATE_FAILED',
        message: 'Failed to create posts in batch',
      },
    })
  }
})
```

### バッチ更新

```typescript
// PATCH /posts/batch
interface BatchUpdatePostDto {
  updates: Array<{
    id: string
    data: {
      title?: string
      content?: string
      published?: boolean
    }
  }>
}

router.patch('/batch', async (req, res) => {
  const { updates }: BatchUpdatePostDto = req.body

  if (!Array.isArray(updates) || updates.length === 0) {
    return res.status(400).json({
      success: false,
      error: {
        code: 'INVALID_INPUT',
        message: 'Updates array is required',
      },
    })
  }

  if (updates.length > 100) {
    return res.status(400).json({
      success: false,
      error: {
        code: 'BATCH_SIZE_EXCEEDED',
        message: 'Maximum 100 posts can be updated at once',
      },
    })
  }

  try {
    const results = await prisma.$transaction(
      updates.map((update) =>
        prisma.post.update({
          where: { id: update.id },
          data: update.data,
        })
      )
    )

    res.json({
      success: true,
      data: results,
      meta: {
        updated: results.length,
      },
    })
  } catch (error) {
    res.status(500).json({
      success: false,
      error: {
        code: 'BATCH_UPDATE_FAILED',
        message: 'Failed to update posts in batch',
      },
    })
  }
})
```

### バッチ削除

```typescript
// DELETE /posts/batch
interface BatchDeleteDto {
  ids: string[]
}

router.delete('/batch', async (req, res) => {
  const { ids }: BatchDeleteDto = req.body

  if (!Array.isArray(ids) || ids.length === 0) {
    return res.status(400).json({
      success: false,
      error: {
        code: 'INVALID_INPUT',
        message: 'IDs array is required',
      },
    })
  }

  try {
    const result = await prisma.post.deleteMany({
      where: {
        id: {
          in: ids,
        },
        // セキュリティ: 自分の投稿のみ削除可能
        authorId: req.user!.userId,
      },
    })

    res.json({
      success: true,
      meta: {
        deleted: result.count,
      },
    })
  } catch (error) {
    res.status(500).json({
      success: false,
      error: {
        code: 'BATCH_DELETE_FAILED',
        message: 'Failed to delete posts in batch',
      },
    })
  }
})
```

## 高度な検索・フィルタリング

### 全文検索

```typescript
// src/routes/search.routes.ts
import { Router } from 'express'
import { PrismaClient } from '@prisma/client'

const router = Router()
const prisma = new PrismaClient()

// GET /search/posts?q=keyword
router.get('/posts', async (req, res) => {
  const query = req.query.q as string
  const page = parseInt(req.query.page as string) || 1
  const limit = parseInt(req.query.limit as string) || 10
  const skip = (page - 1) * limit

  if (!query || query.trim().length < 2) {
    return res.status(400).json({
      success: false,
      error: {
        code: 'INVALID_QUERY',
        message: 'Search query must be at least 2 characters',
      },
    })
  }

  // PostgreSQLの全文検索を使用
  const [posts, total] = await Promise.all([
    prisma.$queryRaw`
      SELECT p.*, u.name as author_name
      FROM posts p
      JOIN users u ON p."authorId" = u.id
      WHERE
        to_tsvector('english', p.title || ' ' || p.content)
        @@ plainto_tsquery('english', ${query})
      ORDER BY
        ts_rank(
          to_tsvector('english', p.title || ' ' || p.content),
          plainto_tsquery('english', ${query})
        ) DESC
      LIMIT ${limit}
      OFFSET ${skip}
    `,
    prisma.$queryRaw`
      SELECT COUNT(*)::int as count
      FROM posts p
      WHERE
        to_tsvector('english', p.title || ' ' || p.content)
        @@ plainto_tsquery('english', ${query})
    `,
  ])

  res.json({
    success: true,
    data: posts,
    meta: {
      query,
      page,
      limit,
      total: (total as any)[0].count,
      totalPages: Math.ceil((total as any)[0].count / limit),
    },
  })
})

export default router
```

### 複雑なフィルタリング

```typescript
// src/utils/query-builder.ts
export interface PostFilters {
  search?: string
  authorId?: string
  published?: boolean
  tags?: string[]
  createdAfter?: Date
  createdBefore?: Date
  minViews?: number
  maxViews?: number
}

export function buildPostWhereClause(filters: PostFilters) {
  const where: any = {}

  // 検索
  if (filters.search) {
    where.OR = [
      { title: { contains: filters.search, mode: 'insensitive' } },
      { content: { contains: filters.search, mode: 'insensitive' } },
    ]
  }

  // 著者フィルター
  if (filters.authorId) {
    where.authorId = filters.authorId
  }

  // 公開状態
  if (filters.published !== undefined) {
    where.published = filters.published
  }

  // タグフィルター
  if (filters.tags && filters.tags.length > 0) {
    where.tags = {
      some: {
        tag: {
          name: {
            in: filters.tags,
          },
        },
      },
    }
  }

  // 作成日フィルター
  if (filters.createdAfter || filters.createdBefore) {
    where.createdAt = {}
    if (filters.createdAfter) {
      where.createdAt.gte = filters.createdAfter
    }
    if (filters.createdBefore) {
      where.createdAt.lte = filters.createdBefore
    }
  }

  // 閲覧数フィルター
  if (filters.minViews !== undefined || filters.maxViews !== undefined) {
    where.views = {}
    if (filters.minViews !== undefined) {
      where.views.gte = filters.minViews
    }
    if (filters.maxViews !== undefined) {
      where.views.lte = filters.maxViews
    }
  }

  return where
}

// 使用例
router.get('/', async (req, res) => {
  const filters: PostFilters = {
    search: req.query.search as string,
    authorId: req.query.authorId as string,
    published: req.query.published === 'true',
    tags: req.query.tags ? (req.query.tags as string).split(',') : undefined,
    createdAfter: req.query.createdAfter
      ? new Date(req.query.createdAfter as string)
      : undefined,
    createdBefore: req.query.createdBefore
      ? new Date(req.query.createdBefore as string)
      : undefined,
    minViews: req.query.minViews
      ? parseInt(req.query.minViews as string)
      : undefined,
    maxViews: req.query.maxViews
      ? parseInt(req.query.maxViews as string)
      : undefined,
  }

  const where = buildPostWhereClause(filters)
  const page = parseInt(req.query.page as string) || 1
  const limit = parseInt(req.query.limit as string) || 10
  const skip = (page - 1) * limit

  const [posts, total] = await Promise.all([
    prisma.post.findMany({
      where,
      skip,
      take: limit,
      include: {
        author: {
          select: {
            id: true,
            name: true,
          },
        },
        tags: {
          include: {
            tag: true,
          },
        },
      },
    }),
    prisma.post.count({ where }),
  ])

  res.json({
    success: true,
    data: posts,
    meta: {
      filters,
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit),
    },
  })
})

// リクエスト例:
// GET /posts?search=typescript&published=true&tags=backend,api&createdAfter=2024-01-01&minViews=100&page=1&limit=20
```

## ファイルアップロード/ダウンロード

### シングルファイルアップロード

```typescript
// src/routes/upload.routes.ts
import { Router } from 'express'
import multer from 'multer'
import path from 'path'
import crypto from 'crypto'
import fs from 'fs/promises'

const router = Router()

// ストレージ設定
const storage = multer.diskStorage({
  destination: async (req, file, cb) => {
    const uploadDir = path.join(__dirname, '../../uploads')
    await fs.mkdir(uploadDir, { recursive: true })
    cb(null, uploadDir)
  },
  filename: (req, file, cb) => {
    const uniqueName = `${crypto.randomUUID()}${path.extname(file.originalname)}`
    cb(null, uniqueName)
  },
})

// ファイルフィルター
const fileFilter = (req: any, file: Express.Multer.File, cb: multer.FileFilterCallback) => {
  const allowedMimes = [
    'image/jpeg',
    'image/png',
    'image/gif',
    'image/webp',
    'application/pdf',
  ]

  if (allowedMimes.includes(file.mimetype)) {
    cb(null, true)
  } else {
    cb(new Error('Invalid file type'))
  }
}

const upload = multer({
  storage,
  fileFilter,
  limits: {
    fileSize: 5 * 1024 * 1024, // 5MB
  },
})

// POST /upload - シングルファイルアップロード
router.post('/', upload.single('file'), async (req, res) => {
  if (!req.file) {
    return res.status(400).json({
      success: false,
      error: {
        code: 'FILE_REQUIRED',
        message: 'File is required',
      },
    })
  }

  // データベースにファイル情報を保存
  const file = await prisma.file.create({
    data: {
      originalName: req.file.originalname,
      filename: req.file.filename,
      mimetype: req.file.mimetype,
      size: req.file.size,
      path: req.file.path,
      uploadedBy: req.user!.userId,
    },
  })

  res.status(201).json({
    success: true,
    data: {
      id: file.id,
      originalName: file.originalName,
      filename: file.filename,
      mimetype: file.mimetype,
      size: file.size,
      url: `/files/${file.id}`,
      createdAt: file.createdAt,
    },
  })
})

// エラーハンドリング
router.use((error: any, req: any, res: any, next: any) => {
  if (error instanceof multer.MulterError) {
    if (error.code === 'LIMIT_FILE_SIZE') {
      return res.status(400).json({
        success: false,
        error: {
          code: 'FILE_TOO_LARGE',
          message: 'File size must not exceed 5MB',
        },
      })
    }
  }

  if (error.message === 'Invalid file type') {
    return res.status(400).json({
      success: false,
      error: {
        code: 'INVALID_FILE_TYPE',
        message: 'Only JPEG, PNG, GIF, WebP, and PDF files are allowed',
      },
    })
  }

  next(error)
})

export default router
```

### マルチファイルアップロード

```typescript
// POST /upload/multiple - 複数ファイルアップロード
router.post('/multiple', upload.array('files', 10), async (req, res) => {
  const files = req.files as Express.Multer.File[]

  if (!files || files.length === 0) {
    return res.status(400).json({
      success: false,
      error: {
        code: 'FILES_REQUIRED',
        message: 'At least one file is required',
      },
    })
  }

  // トランザクションで一括保存
  const savedFiles = await prisma.$transaction(
    files.map((file) =>
      prisma.file.create({
        data: {
          originalName: file.originalname,
          filename: file.filename,
          mimetype: file.mimetype,
          size: file.size,
          path: file.path,
          uploadedBy: req.user!.userId,
        },
      })
    )
  )

  res.status(201).json({
    success: true,
    data: savedFiles.map((file) => ({
      id: file.id,
      originalName: file.originalName,
      filename: file.filename,
      mimetype: file.mimetype,
      size: file.size,
      url: `/files/${file.id}`,
      createdAt: file.createdAt,
    })),
    meta: {
      uploaded: savedFiles.length,
    },
  })
})
```

### ファイルダウンロード

```typescript
// GET /files/:id/download - ファイルダウンロード
router.get('/:id/download', async (req, res) => {
  const { id } = req.params

  const file = await prisma.file.findUnique({
    where: { id },
  })

  if (!file) {
    return res.status(404).json({
      success: false,
      error: {
        code: 'FILE_NOT_FOUND',
        message: 'File not found',
      },
    })
  }

  // ファイルの存在確認
  try {
    await fs.access(file.path)
  } catch (error) {
    return res.status(404).json({
      success: false,
      error: {
        code: 'FILE_NOT_FOUND',
        message: 'File not found on server',
      },
    })
  }

  // ダウンロード
  res.setHeader('Content-Type', file.mimetype)
  res.setHeader('Content-Disposition', `attachment; filename="${file.originalName}"`)
  res.setHeader('Content-Length', file.size)

  const fileStream = require('fs').createReadStream(file.path)
  fileStream.pipe(res)
})

// GET /files/:id - ファイル情報取得
router.get('/:id', async (req, res) => {
  const file = await prisma.file.findUnique({
    where: { id: req.params.id },
    include: {
      uploader: {
        select: {
          id: true,
          name: true,
        },
      },
    },
  })

  if (!file) {
    return res.status(404).json({
      success: false,
      error: {
        code: 'FILE_NOT_FOUND',
        message: 'File not found',
      },
    })
  }

  res.json({
    success: true,
    data: {
      id: file.id,
      originalName: file.originalName,
      mimetype: file.mimetype,
      size: file.size,
      url: `/files/${file.id}`,
      downloadUrl: `/files/${file.id}/download`,
      uploader: file.uploader,
      createdAt: file.createdAt,
    },
  })
})

// DELETE /files/:id - ファイル削除
router.delete('/:id', async (req, res) => {
  const { id } = req.params

  const file = await prisma.file.findUnique({
    where: { id },
  })

  if (!file) {
    return res.status(404).json({
      success: false,
      error: {
        code: 'FILE_NOT_FOUND',
        message: 'File not found',
      },
    })
  }

  // 権限チェック
  if (file.uploadedBy !== req.user!.userId && req.user!.role !== 'admin') {
    return res.status(403).json({
      success: false,
      error: {
        code: 'FORBIDDEN',
        message: 'You do not have permission to delete this file',
      },
    })
  }

  // ファイルシステムから削除
  try {
    await fs.unlink(file.path)
  } catch (error) {
    console.error('Failed to delete file from filesystem:', error)
  }

  // データベースから削除
  await prisma.file.delete({
    where: { id },
  })

  res.status(204).send()
})
```

## 非同期処理とジョブ管理

長時間かかる処理は非同期で実行し、ジョブとして管理します。

### ジョブキューの実装（Bull使用）

```typescript
// src/queues/email.queue.ts
import Bull from 'bull'
import { sendEmail } from '../services/email.service'

export const emailQueue = new Bull('email', {
  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT || '6379'),
  },
})

// ジョブプロセッサー
emailQueue.process(async (job) => {
  const { to, subject, body } = job.data

  console.log(`Processing email job ${job.id}`)

  await sendEmail(to, subject, body)

  return { sent: true, timestamp: new Date() }
})

// エラーハンドリング
emailQueue.on('failed', (job, err) => {
  console.error(`Job ${job.id} failed:`, err)
})

emailQueue.on('completed', (job, result) => {
  console.log(`Job ${job.id} completed:`, result)
})
```

```typescript
// src/routes/job.routes.ts
import { Router } from 'express'
import { emailQueue } from '../queues/email.queue'
import { PrismaClient } from '@prisma/client'

const router = Router()
const prisma = new PrismaClient()

// POST /jobs/send-email - メール送信ジョブの作成
router.post('/send-email', async (req, res) => {
  const { to, subject, body } = req.body

  // ジョブをキューに追加
  const job = await emailQueue.add(
    {
      to,
      subject,
      body,
    },
    {
      attempts: 3, // 最大3回リトライ
      backoff: {
        type: 'exponential',
        delay: 2000, // 2秒から開始
      },
    }
  )

  // ジョブ情報をデータベースに保存
  const jobRecord = await prisma.job.create({
    data: {
      id: job.id.toString(),
      type: 'email',
      status: 'pending',
      data: { to, subject, body },
      createdBy: req.user!.userId,
    },
  })

  res.status(202).json({
    success: true,
    data: {
      jobId: job.id,
      status: 'pending',
      statusUrl: `/jobs/${job.id}`,
    },
  })
})

// GET /jobs/:id - ジョブステータス確認
router.get('/:id', async (req, res) => {
  const { id } = req.params

  const job = await emailQueue.getJob(id)

  if (!job) {
    return res.status(404).json({
      success: false,
      error: {
        code: 'JOB_NOT_FOUND',
        message: 'Job not found',
      },
    })
  }

  const state = await job.getState()
  const progress = job.progress()
  const result = job.returnvalue

  res.json({
    success: true,
    data: {
      id: job.id,
      type: job.name,
      status: state,
      progress,
      result,
      createdAt: new Date(job.timestamp),
      processedAt: job.processedOn ? new Date(job.processedOn) : null,
      finishedAt: job.finishedOn ? new Date(job.finishedOn) : null,
    },
  })
})

// GET /jobs - ジョブ一覧
router.get('/', async (req, res) => {
  const status = req.query.status as string
  const page = parseInt(req.query.page as string) || 1
  const limit = parseInt(req.query.limit as string) || 10

  let jobs: Bull.Job[] = []

  if (status) {
    jobs = await emailQueue.getJobs([status], 0, (page * limit) - 1)
  } else {
    const [waiting, active, completed, failed] = await Promise.all([
      emailQueue.getJobs(['waiting'], 0, (page * limit) - 1),
      emailQueue.getJobs(['active'], 0, (page * limit) - 1),
      emailQueue.getJobs(['completed'], 0, (page * limit) - 1),
      emailQueue.getJobs(['failed'], 0, (page * limit) - 1),
    ])
    jobs = [...waiting, ...active, ...completed, ...failed]
  }

  const data = await Promise.all(
    jobs.map(async (job) => ({
      id: job.id,
      type: job.name,
      status: await job.getState(),
      data: job.data,
      createdAt: new Date(job.timestamp),
    }))
  )

  res.json({
    success: true,
    data,
    meta: {
      page,
      limit,
    },
  })
})

// DELETE /jobs/:id - ジョブキャンセル
router.delete('/:id', async (req, res) => {
  const { id } = req.params

  const job = await emailQueue.getJob(id)

  if (!job) {
    return res.status(404).json({
      success: false,
      error: {
        code: 'JOB_NOT_FOUND',
        message: 'Job not found',
      },
    })
  }

  await job.remove()

  res.status(204).send()
})

export default router
```

## Webhook実装

外部サービスへのイベント通知を実装します。

```typescript
// prisma/schema.prisma
model Webhook {
  id        String   @id @default(cuid())
  url       String
  events    String[] // ['user.created', 'user.updated', 'post.created']
  secret    String
  active    Boolean  @default(true)
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model WebhookDelivery {
  id         String   @id @default(cuid())
  webhookId  String
  webhook    Webhook  @relation(fields: [webhookId], references: [id])
  event      String
  payload    Json
  response   Json?
  status     String   // 'pending', 'success', 'failed'
  attempts   Int      @default(0)
  createdAt  DateTime @default(now())
  deliveredAt DateTime?
}
```

```typescript
// src/services/webhook.service.ts
import axios from 'axios'
import crypto from 'crypto'
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

export class WebhookService {
  // Webhookの登録
  async createWebhook(data: {
    url: string
    events: string[]
    userId: string
  }) {
    const secret = crypto.randomBytes(32).toString('hex')

    return prisma.webhook.create({
      data: {
        url: data.url,
        events: data.events,
        secret,
        userId: data.userId,
      },
    })
  }

  // イベント発火
  async triggerEvent(event: string, payload: any) {
    // 該当イベントを購読しているWebhookを取得
    const webhooks = await prisma.webhook.findMany({
      where: {
        active: true,
        events: {
          has: event,
        },
      },
    })

    // 各Webhookに配信
    const deliveries = await Promise.all(
      webhooks.map((webhook) =>
        this.deliverWebhook(webhook, event, payload)
      )
    )

    return deliveries
  }

  // Webhook配信
  private async deliverWebhook(
    webhook: any,
    event: string,
    payload: any
  ) {
    // 配信レコード作成
    const delivery = await prisma.webhookDelivery.create({
      data: {
        webhookId: webhook.id,
        event,
        payload,
        status: 'pending',
      },
    })

    try {
      // 署名生成
      const signature = this.generateSignature(
        JSON.stringify(payload),
        webhook.secret
      )

      // Webhook送信
      const response = await axios.post(
        webhook.url,
        {
          event,
          payload,
          timestamp: new Date().toISOString(),
        },
        {
          headers: {
            'Content-Type': 'application/json',
            'X-Webhook-Signature': signature,
            'X-Webhook-Event': event,
          },
          timeout: 5000,
        }
      )

      // 成功レコード更新
      await prisma.webhookDelivery.update({
        where: { id: delivery.id },
        data: {
          status: 'success',
          response: {
            status: response.status,
            headers: response.headers,
            data: response.data,
          },
          deliveredAt: new Date(),
          attempts: 1,
        },
      })

      return { success: true, delivery }
    } catch (error: any) {
      // 失敗レコード更新
      await prisma.webhookDelivery.update({
        where: { id: delivery.id },
        data: {
          status: 'failed',
          response: {
            error: error.message,
            code: error.code,
          },
          attempts: 1,
        },
      })

      return { success: false, delivery, error: error.message }
    }
  }

  // 署名生成（HMAC-SHA256）
  private generateSignature(payload: string, secret: string): string {
    return crypto
      .createHmac('sha256', secret)
      .update(payload)
      .digest('hex')
  }

  // 署名検証
  verifySignature(payload: string, signature: string, secret: string): boolean {
    const expectedSignature = this.generateSignature(payload, secret)
    return crypto.timingSafeEqual(
      Buffer.from(signature),
      Buffer.from(expectedSignature)
    )
  }
}
```

```typescript
// src/routes/webhook.routes.ts
import { Router } from 'express'
import { WebhookService } from '../services/webhook.service'
import { PrismaClient } from '@prisma/client'

const router = Router()
const webhookService = new WebhookService()
const prisma = new PrismaClient()

// POST /webhooks - Webhook登録
router.post('/', async (req, res) => {
  const { url, events } = req.body

  // URLバリデーション
  try {
    new URL(url)
  } catch (error) {
    return res.status(400).json({
      success: false,
      error: {
        code: 'INVALID_URL',
        message: 'Invalid webhook URL',
      },
    })
  }

  // イベントバリデーション
  const validEvents = [
    'user.created',
    'user.updated',
    'user.deleted',
    'post.created',
    'post.updated',
    'post.deleted',
  ]

  const invalidEvents = events.filter((e: string) => !validEvents.includes(e))
  if (invalidEvents.length > 0) {
    return res.status(400).json({
      success: false,
      error: {
        code: 'INVALID_EVENTS',
        message: `Invalid events: ${invalidEvents.join(', ')}`,
        validEvents,
      },
    })
  }

  const webhook = await webhookService.createWebhook({
    url,
    events,
    userId: req.user!.userId,
  })

  res.status(201).json({
    success: true,
    data: webhook,
  })
})

// GET /webhooks - Webhook一覧
router.get('/', async (req, res) => {
  const webhooks = await prisma.webhook.findMany({
    where: { userId: req.user!.userId },
    include: {
      _count: {
        select: {
          deliveries: true,
        },
      },
    },
  })

  res.json({
    success: true,
    data: webhooks,
  })
})

// GET /webhooks/:id - Webhook詳細
router.get('/:id', async (req, res) => {
  const webhook = await prisma.webhook.findUnique({
    where: { id: req.params.id },
    include: {
      deliveries: {
        orderBy: { createdAt: 'desc' },
        take: 10,
      },
    },
  })

  if (!webhook) {
    return res.status(404).json({
      success: false,
      error: {
        code: 'WEBHOOK_NOT_FOUND',
        message: 'Webhook not found',
      },
    })
  }

  // 権限チェック
  if (webhook.userId !== req.user!.userId) {
    return res.status(403).json({
      success: false,
      error: {
        code: 'FORBIDDEN',
        message: 'Access denied',
      },
    })
  }

  res.json({
    success: true,
    data: webhook,
  })
})

// PATCH /webhooks/:id - Webhook更新
router.patch('/:id', async (req, res) => {
  const { url, events, active } = req.body

  const webhook = await prisma.webhook.update({
    where: { id: req.params.id },
    data: {
      ...(url && { url }),
      ...(events && { events }),
      ...(active !== undefined && { active }),
    },
  })

  res.json({
    success: true,
    data: webhook,
  })
})

// DELETE /webhooks/:id - Webhook削除
router.delete('/:id', async (req, res) => {
  await prisma.webhook.delete({
    where: { id: req.params.id },
  })

  res.status(204).send()
})

// POST /webhooks/:id/test - Webhookテスト
router.post('/:id/test', async (req, res) => {
  const webhook = await prisma.webhook.findUnique({
    where: { id: req.params.id },
  })

  if (!webhook) {
    return res.status(404).json({
      success: false,
      error: {
        code: 'WEBHOOK_NOT_FOUND',
        message: 'Webhook not found',
      },
    })
  }

  const result = await webhookService.triggerEvent('webhook.test', {
    message: 'This is a test webhook',
    webhookId: webhook.id,
  })

  res.json({
    success: true,
    data: result,
  })
})

export default router
```

```typescript
// 使用例: ユーザー作成時にWebhookを発火
router.post('/users', async (req, res) => {
  const user = await prisma.user.create({
    data: req.body,
  })

  // Webhookを非同期で発火
  webhookService.triggerEvent('user.created', {
    id: user.id,
    email: user.email,
    name: user.name,
    createdAt: user.createdAt,
  }).catch((error) => {
    console.error('Failed to trigger webhook:', error)
  })

  res.status(201).json({
    success: true,
    data: user,
  })
})
```

## APIレート制限

### 基本的なレート制限

```typescript
// src/middleware/rate-limit.ts
import rateLimit from 'express-rate-limit'
import RedisStore from 'rate-limit-redis'
import { createClient } from 'redis'

const redisClient = createClient({
  url: process.env.REDIS_URL,
})

redisClient.connect()

// 一般的なエンドポイント用
export const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15分
  max: 100, // 100リクエスト
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({
    client: redisClient,
    prefix: 'rl:api:',
  }),
  message: {
    success: false,
    error: {
      code: 'RATE_LIMIT_EXCEEDED',
      message: 'Too many requests, please try again later',
    },
  },
  keyGenerator: (req) => {
    // 認証済みユーザーはユーザーID、未認証はIPアドレス
    return req.user?.userId || req.ip
  },
})

// 認証エンドポイント用（厳しい制限）
export const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  skipSuccessfulRequests: true,
  store: new RedisStore({
    client: redisClient,
    prefix: 'rl:auth:',
  }),
})

// 検索エンドポイント用
export const searchLimiter = rateLimit({
  windowMs: 1 * 60 * 1000, // 1分
  max: 10,
  store: new RedisStore({
    client: redisClient,
    prefix: 'rl:search:',
  }),
})

// アップロードエンドポイント用
export const uploadLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1時間
  max: 20,
  store: new RedisStore({
    client: redisClient,
    prefix: 'rl:upload:',
  }),
})
```

### カスタムレート制限

```typescript
// src/middleware/custom-rate-limit.ts
import { Request, Response, NextFunction } from 'express'
import { createClient } from 'redis'

const redisClient = createClient({
  url: process.env.REDIS_URL,
})

redisClient.connect()

export interface RateLimitConfig {
  points: number // 許可するポイント数
  duration: number // 秒単位
  blockDuration?: number // ブロック期間（秒）
}

export function customRateLimit(config: RateLimitConfig) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const key = `custom_rl:${req.user?.userId || req.ip}`

    try {
      // 現在のカウントを取得
      const current = await redisClient.get(key)
      const count = current ? parseInt(current) : 0

      if (count >= config.points) {
        // レート制限超過
        const ttl = await redisClient.ttl(key)

        return res.status(429).json({
          success: false,
          error: {
            code: 'RATE_LIMIT_EXCEEDED',
            message: 'Rate limit exceeded',
            retryAfter: ttl,
          },
        })
      }

      // カウントを増やす
      if (count === 0) {
        await redisClient.set(key, '1', { EX: config.duration })
      } else {
        await redisClient.incr(key)
      }

      // レスポンスヘッダーに情報を追加
      const remaining = config.points - (count + 1)
      res.setHeader('X-RateLimit-Limit', config.points)
      res.setHeader('X-RateLimit-Remaining', remaining)
      res.setHeader('X-RateLimit-Reset', Date.now() + (config.duration * 1000))

      next()
    } catch (error) {
      console.error('Rate limit error:', error)
      next()
    }
  }
}

// 使用例
router.post(
  '/posts',
  customRateLimit({
    points: 10, // 10回まで
    duration: 60, // 60秒間
  }),
  async (req, res) => {
    // ...
  }
)
```

## リクエスト/レスポンスバリデーション

### Zodによるバリデーション

```typescript
// src/schemas/user.schema.ts
import { z } from 'zod'

export const createUserSchema = z.object({
  email: z.string().email('Invalid email address'),
  name: z.string().min(2, 'Name must be at least 2 characters'),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(
      /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
      'Password must contain at least one uppercase letter, one lowercase letter, and one number'
    ),
  role: z.enum(['user', 'admin']).optional(),
})

export const updateUserSchema = z.object({
  email: z.string().email().optional(),
  name: z.string().min(2).optional(),
  role: z.enum(['user', 'admin']).optional(),
})

export const paginationSchema = z.object({
  page: z.string().regex(/^\d+$/).transform(Number).optional(),
  limit: z.string().regex(/^\d+$/).transform(Number).optional(),
  sortBy: z.string().optional(),
  order: z.enum(['asc', 'desc']).optional(),
})

export const postFilterSchema = z.object({
  search: z.string().optional(),
  authorId: z.string().optional(),
  published: z.enum(['true', 'false']).transform((v) => v === 'true').optional(),
  tags: z.string().transform((v) => v.split(',')).optional(),
  createdAfter: z.string().datetime().optional(),
  createdBefore: z.string().datetime().optional(),
})
```

```typescript
// src/middleware/validation.ts
import { Request, Response, NextFunction } from 'express'
import { ZodSchema, ZodError } from 'zod'

export function validate(schema: ZodSchema, source: 'body' | 'query' | 'params' = 'body') {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      const data = req[source]
      const validated = schema.parse(data)

      // バリデーション済みデータで置き換え
      req[source] = validated

      next()
    } catch (error) {
      if (error instanceof ZodError) {
        const errors = error.errors.map((err) => ({
          field: err.path.join('.'),
          message: err.message,
        }))

        return res.status(400).json({
          success: false,
          error: {
            code: 'VALIDATION_ERROR',
            message: 'Validation failed',
            details: errors,
          },
        })
      }

      next(error)
    }
  }
}

// 使用例
router.post(
  '/users',
  validate(createUserSchema, 'body'),
  async (req, res) => {
    // req.bodyは型安全に使用可能
    const { email, name, password, role } = req.body
    // ...
  }
)

router.get(
  '/posts',
  validate(paginationSchema, 'query'),
  validate(postFilterSchema, 'query'),
  async (req, res) => {
    // クエリパラメータが検証済み
    const { page, limit, search, tags } = req.query
    // ...
  }
)
```

## セキュリティベストプラクティス

### SQL Injection対策

```typescript
// ❌ 危険: 生のSQLクエリ
app.get('/users', async (req, res) => {
  const name = req.query.name
  const users = await prisma.$queryRawUnsafe(
    `SELECT * FROM users WHERE name = '${name}'`
  )
  // SQLインジェクションの脆弱性
})

// ✅ 安全: パラメータ化されたクエリ
app.get('/users', async (req, res) => {
  const name = req.query.name as string
  const users = await prisma.$queryRaw`
    SELECT * FROM users WHERE name = ${name}
  `
  // Prismaが自動的にエスケープ
})

// ✅ 最も安全: ORMのクエリビルダーを使用
app.get('/users', async (req, res) => {
  const name = req.query.name as string
  const users = await prisma.user.findMany({
    where: {
      name: {
        contains: name,
        mode: 'insensitive',
      },
    },
  })
})
```

### XSS対策

```typescript
// src/middleware/sanitize.ts
import DOMPurify from 'isomorphic-dompurify'

export function sanitizeBody() {
  return (req: Request, res: Response, next: NextFunction) => {
    if (req.body) {
      req.body = sanitizeObject(req.body)
    }
    next()
  }
}

function sanitizeObject(obj: any): any {
  if (typeof obj === 'string') {
    return DOMPurify.sanitize(obj)
  }

  if (Array.isArray(obj)) {
    return obj.map(sanitizeObject)
  }

  if (typeof obj === 'object' && obj !== null) {
    const sanitized: any = {}
    for (const key in obj) {
      sanitized[key] = sanitizeObject(obj[key])
    }
    return sanitized
  }

  return obj
}

// 使用例
router.post('/posts', sanitizeBody(), async (req, res) => {
  // req.bodyはサニタイズ済み
  const { title, content } = req.body
  // ...
})
```

### CSRF対策

```typescript
// src/middleware/csrf.ts
import csrf from 'csurf'
import cookieParser from 'cookie-parser'

const csrfProtection = csrf({
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
  },
})

// CSRFトークン取得エンドポイント
app.get('/csrf-token', csrfProtection, (req, res) => {
  res.json({
    csrfToken: req.csrfToken(),
  })
})

// 保護されたエンドポイント
app.post('/posts', csrfProtection, async (req, res) => {
  // CSRFトークンが検証される
})
```

### セキュアヘッダーの設定

```typescript
// src/middleware/security.ts
import helmet from 'helmet'

export function setupSecurity(app: Express) {
  // Helmet - セキュリティヘッダーを自動設定
  app.use(helmet())

  // カスタムセキュリティヘッダー
  app.use((req, res, next) => {
    // Content Security Policy
    res.setHeader(
      'Content-Security-Policy',
      "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'"
    )

    // Strict Transport Security
    res.setHeader(
      'Strict-Transport-Security',
      'max-age=31536000; includeSubDomains'
    )

    // X-Content-Type-Options
    res.setHeader('X-Content-Type-Options', 'nosniff')

    // X-Frame-Options
    res.setHeader('X-Frame-Options', 'DENY')

    // X-XSS-Protection
    res.setHeader('X-XSS-Protection', '1; mode=block')

    // Referrer Policy
    res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin')

    next()
  })
}
```

## ベンチマーク指標 - エンドポイント最適化シナリオ

### 想定シナリオ: SNSプラットフォームのAPI最適化

#### 初期状態

| 指標 | 値 |
|-----|-----|
| 検索API平均レスポンス | 2,800ms |
| ファイルアップロード成功率 | 78% |
| Webhookbの配信成功率 | 65% |
| 1日あたりの不正リクエスト | 50,000件 |

#### 実施した改善

1. **検索の最適化**
   - PostgreSQL全文検索インデックス追加
   - Elasticsearchの導入
   - 検索結果のキャッシング

2. **ファイルアップロードの改善**
   - チャンクアップロードの実装
   - リトライメカニズムの追加
   - エラーハンドリングの強化

3. **Webhookの信頼性向上**
   - リトライロジックの実装（指数バックオフ）
   - デッドレターキューの導入
   - 配信ステータスの追跡

4. **セキュリティ強化**
   - レート制限の実装
   - リクエストバリデーションの強化
   - 不正アクセス検知

#### 改善後の結果

| 指標 | 値 | 改善率 |
|-----|-----|--------|
| 検索API平均レスポンス | 180ms | **-93.6%** |
| ファイルアップロード成功率 | 99.2% | **+27.2%** |
| Webhookの配信成功率 | 98.5% | **+51.5%** |
| 1日あたりの不正リクエスト | 500件 | **-99%** |

## 実践演習

### 演習1: 複雑な検索エンドポイントの実装

記事検索APIを実装してください。

**要件:**
- 全文検索
- タグフィルタリング
- 日付範囲フィルタリング
- 著者フィルタリング
- ソート（relevance, date, views）
- ページネーション

### 演習2: ファイルアップロードの実装

画像アップロードAPIを実装してください。

**要件:**
- 複数ファイルアップロード対応
- ファイルタイプ検証（JPEG, PNG, GIF, WebP）
- サイズ制限（5MB）
- 画像のリサイズ（Sharp使用）
- サムネイル生成
- メタデータ保存

### 演習3: Webhook システムの構築

イベント通知システムを実装してください。

**要件:**
- Webhook登録/更新/削除
- イベントフィルタリング
- 署名検証
- リトライメカニズム
- 配信ログ

## まとめ

この章では、実践的なエンドポイント設計について学習しました。

### 重要なポイント

1. **リソース関係**を適切に設計する
2. **バッチ操作**で効率を向上させる
3. **高度な検索**機能を実装する
4. **ファイル処理**を適切にハンドリングする
5. **非同期処理**でパフォーマンスを改善する
6. **Webhook**で外部連携を実現する
7. **レート制限**でAPI濫用を防止する
8. **バリデーション**でデータ整合性を保つ
9. **セキュリティ**を常に意識する

### 次のステップ

次章では、API認証・認可戦略について学習します。JWT、OAuth 2.0、API Key、ロールベースアクセス制御（RBAC）など、セキュアなAPI構築に不可欠な認証・認可メカニズムを詳しく扱います。
