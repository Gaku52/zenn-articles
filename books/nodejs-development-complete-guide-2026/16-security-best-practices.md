---
title: "セキュリティベストプラクティス - 安全なアプリケーション構築"
---

# セキュリティベストプラクティス - 安全なアプリケーション構築

Node.jsアプリケーションにおける主要なセキュリティリスクと、それらを防ぐための実践的な対策を学びます。

## OWASP Top 10 への対策

### 1. インジェクション攻撃の防止

```typescript
// ❌ Bad: SQLインジェクションの危険性
app.get('/users', async (req, res) => {
  const query = `SELECT * FROM users WHERE name = '${req.query.name}'`;
  const users = await db.query(query);
  res.json(users);
});

// ✅ Good: Prisma のパラメータ化されたクエリ
app.get('/users', async (req, res) => {
  const users = await prisma.user.findMany({
    where: {
      name: req.query.name,
    },
  });
  res.json(users);
});

// ✅ Good: プレースホルダーを使用
app.get('/users', async (req, res) => {
  const users = await prisma.$queryRaw`
    SELECT * FROM users WHERE name = ${req.query.name}
  `;
  res.json(users);
});
```

### 2. XSS (Cross-Site Scripting) 対策

```typescript
import helmet from 'helmet';
import { escape } from 'html-escaper';

// helmet でセキュリティヘッダーを設定
app.use(
  helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", "'unsafe-inline'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        imgSrc: ["'self'", 'data:', 'https:'],
      },
    },
  })
);

// ユーザー入力をエスケープ
app.post('/comments', async (req, res) => {
  const sanitizedContent = escape(req.body.content);

  const comment = await prisma.comment.create({
    data: {
      content: sanitizedContent,
      userId: req.user.id,
    },
  });

  res.json(comment);
});

// テンプレートエンジンで自動エスケープ（EJS）
app.set('view engine', 'ejs');
// <%= user.name %> は自動エスケープ
// <%- user.name %> はエスケープなし（危険）
```

### 3. CSRF (Cross-Site Request Forgery) 対策

```typescript
import csrf from 'csurf';
import cookieParser from 'cookie-parser';

app.use(cookieParser());
app.use(csrf({ cookie: true }));

// CSRFトークンをフォームに埋め込む
app.get('/form', (req, res) => {
  res.render('form', { csrfToken: req.csrfToken() });
});

// POSTリクエストでトークンを検証
app.post('/submit', (req, res) => {
  // csrfミドルウェアが自動検証
  res.json({ message: 'Success' });
});

// SPA向け: トークンをヘッダーで送信
app.get('/api/csrf-token', (req, res) => {
  res.json({ token: req.csrfToken() });
});
```

### 4. 認証とセッション管理

```typescript
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';

// パスワードのハッシュ化
async function hashPassword(password: string): Promise<string> {
  const salt = await bcrypt.genSalt(10);
  return bcrypt.hash(password, salt);
}

// パスワードの検証
async function verifyPassword(
  password: string,
  hash: string
): Promise<boolean> {
  return bcrypt.compare(password, hash);
}

// ユーザー登録
app.post('/auth/register', async (req, res) => {
  const { email, password } = req.body;

  // パスワードの強度チェック
  if (password.length < 8) {
    return res.status(400).json({ error: 'Password too short' });
  }

  const hashedPassword = await hashPassword(password);

  const user = await prisma.user.create({
    data: {
      email,
      password: hashedPassword,
    },
  });

  res.json({ id: user.id, email: user.email });
});

// ログイン
app.post('/auth/login', async (req, res) => {
  const { email, password } = req.body;

  const user = await prisma.user.findUnique({
    where: { email },
  });

  if (!user || !(await verifyPassword(password, user.password))) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // JWTトークン生成
  const token = jwt.sign(
    { userId: user.id, email: user.email },
    process.env.JWT_SECRET!,
    { expiresIn: '7d' }
  );

  res.json({ token });
});

// JWT検証ミドルウェア
function authenticate(req: Request, res: Response, next: NextFunction) {
  const token = req.headers.authorization?.replace('Bearer ', '');

  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
}
```

## レート制限とDDoS対策

### express-rate-limit の使用

```typescript
import rateLimit from 'express-rate-limit';

// 一般的なエンドポイント
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15分
  max: 100, // 最大100リクエスト
  message: 'Too many requests, please try again later.',
  standardHeaders: true,
  legacyHeaders: false,
});

app.use('/api/', apiLimiter);

// ログイン専用（厳しめ）
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5, // 15分で5回まで
  skipSuccessfulRequests: true, // 成功したリクエストはカウントしない
});

app.post('/auth/login', loginLimiter, async (req, res) => {
  // ログイン処理
});

// IP別の制限
const createAccountLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1時間
  max: 3, // 1時間に3アカウントまで
  keyGenerator: (req) => req.ip,
});

app.post('/auth/register', createAccountLimiter, async (req, res) => {
  // 登録処理
});
```

### Redisを使った分散レート制限

```typescript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import { createClient } from 'redis';

const redisClient = createClient({
  url: process.env.REDIS_URL,
});

await redisClient.connect();

const limiter = rateLimit({
  store: new RedisStore({
    client: redisClient,
    prefix: 'rate-limit:',
  }),
  windowMs: 15 * 60 * 1000,
  max: 100,
});

app.use(limiter);
```

## 入力検証とサニタイゼーション

### Zod による厳密な検証

```typescript
import { z } from 'zod';

// スキーマ定義
const createUserSchema = z.object({
  email: z.string().email().max(255),
  password: z
    .string()
    .min(8)
    .max(100)
    .regex(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, {
      message: 'Password must contain uppercase, lowercase, and number',
    }),
  name: z.string().min(1).max(100).trim(),
  age: z.number().int().min(18).max(150).optional(),
});

// 検証ミドルウェア
function validate(schema: z.ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      req.body = schema.parse(req.body);
      next();
    } catch (error) {
      if (error instanceof z.ZodError) {
        return res.status(400).json({
          error: 'Validation failed',
          details: error.errors,
        });
      }
      next(error);
    }
  };
}

// 使用例
app.post('/users', validate(createUserSchema), async (req, res) => {
  const user = await prisma.user.create({
    data: req.body,
  });
  res.json(user);
});
```

### ファイルアップロードの検証

```typescript
import multer from 'multer';
import path from 'path';

// ファイルサイズとMIMEタイプの制限
const upload = multer({
  storage: multer.diskStorage({
    destination: 'uploads/',
    filename: (req, file, cb) => {
      const uniqueSuffix = `${Date.now()}-${Math.round(Math.random() * 1e9)}`;
      cb(null, `${uniqueSuffix}${path.extname(file.originalname)}`);
    },
  }),
  limits: {
    fileSize: 5 * 1024 * 1024, // 5MB
  },
  fileFilter: (req, file, cb) => {
    const allowedMimes = ['image/jpeg', 'image/png', 'image/gif'];

    if (allowedMimes.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new Error('Invalid file type'));
    }
  },
});

app.post('/upload', upload.single('image'), (req, res) => {
  res.json({ filename: req.file?.filename });
});

// エラーハンドリング
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  if (err instanceof multer.MulterError) {
    if (err.code === 'LIMIT_FILE_SIZE') {
      return res.status(400).json({ error: 'File too large' });
    }
  }
  next(err);
});
```

## 環境変数とシークレット管理

### dotenv の安全な使用

```typescript
import dotenv from 'dotenv';

// 開発環境
dotenv.config();

// 必須の環境変数を検証
const requiredEnvVars = [
  'DATABASE_URL',
  'JWT_SECRET',
  'REDIS_URL',
  'NODE_ENV',
];

for (const envVar of requiredEnvVars) {
  if (!process.env[envVar]) {
    throw new Error(`Missing required environment variable: ${envVar}`);
  }
}

// 型安全な環境変数アクセス
interface Env {
  DATABASE_URL: string;
  JWT_SECRET: string;
  REDIS_URL: string;
  NODE_ENV: 'development' | 'production' | 'test';
  PORT: string;
}

function getEnv(): Env {
  return {
    DATABASE_URL: process.env.DATABASE_URL!,
    JWT_SECRET: process.env.JWT_SECRET!,
    REDIS_URL: process.env.REDIS_URL!,
    NODE_ENV: process.env.NODE_ENV as Env['NODE_ENV'],
    PORT: process.env.PORT || '3000',
  };
}

export const env = getEnv();
```

### .env のセキュリティ

```bash
# .gitignore に必ず追加
.env
.env.local
.env.*.local

# .env.example を用意（値は入れない）
DATABASE_URL=
JWT_SECRET=
REDIS_URL=
```

## セキュアなHTTPヘッダー

### helmet による包括的な設定

```typescript
import helmet from 'helmet';

app.use(
  helmet({
    // Content Security Policy
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", "'unsafe-inline'", 'cdn.example.com'],
        styleSrc: ["'self'", "'unsafe-inline'"],
        imgSrc: ["'self'", 'data:', 'https:'],
        connectSrc: ["'self'", 'api.example.com'],
        fontSrc: ["'self'", 'fonts.gstatic.com'],
        objectSrc: ["'none'"],
        upgradeInsecureRequests: [],
      },
    },

    // Strict Transport Security
    hsts: {
      maxAge: 31536000, // 1年
      includeSubDomains: true,
      preload: true,
    },

    // X-Frame-Options
    frameguard: {
      action: 'deny',
    },

    // X-Content-Type-Options
    noSniff: true,

    // Referrer-Policy
    referrerPolicy: {
      policy: 'strict-origin-when-cross-origin',
    },
  })
);
```

### CORS の適切な設定

```typescript
import cors from 'cors';

// 開発環境
if (process.env.NODE_ENV === 'development') {
  app.use(
    cors({
      origin: 'http://localhost:3000',
      credentials: true,
    })
  );
}

// 本番環境
if (process.env.NODE_ENV === 'production') {
  app.use(
    cors({
      origin: (origin, callback) => {
        const allowedOrigins = [
          'https://example.com',
          'https://www.example.com',
        ];

        if (!origin || allowedOrigins.includes(origin)) {
          callback(null, true);
        } else {
          callback(new Error('Not allowed by CORS'));
        }
      },
      credentials: true,
      maxAge: 86400, // 24時間
    })
  );
}
```

## 依存関係のセキュリティ

### npm audit の定期実行

```bash
# 脆弱性をチェック
npm audit

# 自動修正
npm audit fix

# 強制修正（破壊的変更の可能性あり）
npm audit fix --force

# package-lock.json を更新
npm audit fix --package-lock-only
```

### Snyk によるセキュリティスキャン

```bash
# インストール
npm install -g snyk

# 認証
snyk auth

# テスト
snyk test

# 継続的監視
snyk monitor

# 自動修正
snyk fix
```

### GitHub Dependabot の活用

`.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: 'npm'
    directory: '/'
    schedule:
      interval: 'weekly'
    open-pull-requests-limit: 10
    reviewers:
      - 'Gaku52'
    labels:
      - 'dependencies'
      - 'security'
```

## ログインセキュリティ

### ブルートフォース攻撃対策

```typescript
import rateLimit from 'express-rate-limit';

// ログイン試行回数の制限
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  skipSuccessfulRequests: true,
  handler: (req, res) => {
    res.status(429).json({
      error: 'Too many login attempts, please try again later',
    });
  },
});

// アカウントロック機能
interface LoginAttempt {
  count: number;
  lockedUntil?: Date;
}

const loginAttempts = new Map<string, LoginAttempt>();

app.post('/auth/login', loginLimiter, async (req, res) => {
  const { email, password } = req.body;

  // アカウントロックチェック
  const attempt = loginAttempts.get(email);
  if (attempt?.lockedUntil && attempt.lockedUntil > new Date()) {
    return res.status(429).json({
      error: 'Account temporarily locked',
      retryAfter: attempt.lockedUntil,
    });
  }

  const user = await prisma.user.findUnique({ where: { email } });

  if (!user || !(await verifyPassword(password, user.password))) {
    // 失敗カウント増加
    const currentAttempt = loginAttempts.get(email) || { count: 0 };
    currentAttempt.count++;

    // 5回失敗で30分ロック
    if (currentAttempt.count >= 5) {
      currentAttempt.lockedUntil = new Date(Date.now() + 30 * 60 * 1000);
    }

    loginAttempts.set(email, currentAttempt);

    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // ログイン成功: カウントをリセット
  loginAttempts.delete(email);

  const token = jwt.sign({ userId: user.id }, process.env.JWT_SECRET!);
  res.json({ token });
});
```

### 2要素認証 (2FA)

```typescript
import speakeasy from 'speakeasy';
import QRCode from 'qrcode';

// 2FAシークレット生成
app.post('/auth/2fa/setup', authenticate, async (req, res) => {
  const secret = speakeasy.generateSecret({
    name: `MyApp (${req.user.email})`,
  });

  // QRコードを生成
  const qrCodeUrl = await QRCode.toDataURL(secret.otpauth_url!);

  // シークレットをDBに保存
  await prisma.user.update({
    where: { id: req.user.userId },
    data: { twoFactorSecret: secret.base32 },
  });

  res.json({
    secret: secret.base32,
    qrCode: qrCodeUrl,
  });
});

// 2FA検証
app.post('/auth/2fa/verify', authenticate, async (req, res) => {
  const { token } = req.body;

  const user = await prisma.user.findUnique({
    where: { id: req.user.userId },
  });

  const verified = speakeasy.totp.verify({
    secret: user!.twoFactorSecret!,
    encoding: 'base32',
    token,
  });

  if (verified) {
    await prisma.user.update({
      where: { id: req.user.userId },
      data: { twoFactorEnabled: true },
    });

    res.json({ message: '2FA enabled' });
  } else {
    res.status(400).json({ error: 'Invalid token' });
  }
});
```

## セキュアなセッション管理

### Redis セッションストア

```typescript
import session from 'express-session';
import RedisStore from 'connect-redis';
import { createClient } from 'redis';

const redisClient = createClient({
  url: process.env.REDIS_URL,
});

await redisClient.connect();

app.use(
  session({
    store: new RedisStore({ client: redisClient }),
    secret: process.env.SESSION_SECRET!,
    resave: false,
    saveUninitialized: false,
    cookie: {
      secure: process.env.NODE_ENV === 'production', // HTTPS のみ
      httpOnly: true, // JavaScript からアクセス不可
      maxAge: 1000 * 60 * 60 * 24 * 7, // 7日
      sameSite: 'strict', // CSRF 対策
    },
  })
);
```

## まとめ

セキュリティベストプラクティス:

- ✅ Prismaでパラメータ化されたクエリを使用
- ✅ helmet でセキュリティヘッダーを設定
- ✅ CSRF トークンで保護
- ✅ bcrypt でパスワードをハッシュ化
- ✅ JWT で認証を実装
- ✅ レート制限でブルートフォース攻撃を防止
- ✅ Zod で入力を厳密に検証
- ✅ 環境変数を安全に管理
- ✅ npm audit/Snyk で依存関係を監視
- ✅ 2FAでアカウントを強化
- ❌ パスワードを平文で保存しない
- ❌ 機密情報をログやエラーメッセージに含めない
- ❌ デバッグモードを本番環境で有効にしない

次の章では、テストの自動化について学びます。
