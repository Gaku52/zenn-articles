---
title: "本番環境で泣かないためのNode.jsエラーハンドリング実践ガイド"
emoji: "🛡️"
type: "tech"
topics: ["nodejs", "errorhandling", "backend", "production", "typescript"]
published: false
---

## はじめに

深夜2時、アラートが鳴る。本番環境でアプリがクラッシュ。ログを見ても原因がわからない...こんな悪夢を経験したことはありませんか？

開発環境では問題なく動いていたコードが、本番環境で予期せぬエラーを引き起こす。これは多くのNode.js開発者が経験する課題です。

この記事では、本番環境で安定稼働するために必要な5つのエラーハンドリングパターンを紹介します。適切な実装により、システムの信頼性向上とデバッグ時間の大幅な短縮が期待できます。

## 1. 未捕捉例外によるクラッシュを防ぐ

### 問題点

Node.jsアプリケーションでは、未捕捉の例外やPromise拒否がプロセス全体をクラッシュさせる可能性があります。

```typescript
// ❌ 未捕捉の例外でプロセスがクラッシュ
app.get('/user/:id', async (req, res) => {
  const user = await db.user.findUnique({
    where: { id: req.params.id }
  });

  // userがnullの場合、次の行でエラー
  res.json({ name: user.name }); // Cannot read property 'name' of null
});
```

### 解決策: 多層防御のエラーハンドリング

```typescript
// ✅ グローバルエラーハンドラーの設定
process.on('uncaughtException', (error: Error) => {
  console.error('Uncaught Exception:', error);
  // エラーログをロギングサービスに送信
  logger.fatal(error, 'Uncaught Exception - Process will exit');

  // グレースフルシャットダウン
  process.exit(1);
});

process.on('unhandledRejection', (reason: any, promise: Promise<any>) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
  logger.error({ reason, promise }, 'Unhandled Promise Rejection');

  // 本番環境では必要に応じてプロセスを再起動
  if (process.env.NODE_ENV === 'production') {
    process.exit(1);
  }
});

// ✅ ルートレベルでのエラーハンドリング
app.get('/user/:id', async (req, res, next) => {
  try {
    const user = await db.user.findUnique({
      where: { id: req.params.id }
    });

    if (!user) {
      return res.status(404).json({
        error: 'User not found'
      });
    }

    res.json({ name: user.name });
  } catch (error) {
    next(error); // エラーミドルウェアに委譲
  }
});

// ✅ Expressエラーハンドリングミドルウェア
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  logger.error({
    err,
    req: {
      method: req.method,
      url: req.url,
      headers: req.headers
    }
  }, 'Request failed');

  res.status(500).json({
    error: 'Internal server error',
    requestId: req.id // トレーシング用
  });
});
```

**ベストプラクティス:**
- グローバルハンドラーで最後の砦を用意
- 各ルートで適切にtry-catch
- エラーミドルウェアで統一的な処理

## 2. 構造化ロギングでデバッグを効率化

### 問題点

console.logでの非構造化ログは、本番環境でのデバッグを困難にします。

```typescript
// ❌ 非構造化ログ
console.log('Error occurred:', error.message);
console.log('User ID:', userId);
```

### 解決策: pinoなどの構造化ロガー

```typescript
import pino from 'pino';

// ✅ 構造化ロガーのセットアップ
const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => {
      return { level: label };
    }
  },
  serializers: {
    err: pino.stdSerializers.err,
    req: pino.stdSerializers.req,
    res: pino.stdSerializers.res
  }
});

// ✅ 構造化されたエラーログ
try {
  const user = await getUser(userId);
} catch (error) {
  logger.error({
    err: error,
    userId,
    operation: 'getUser',
    timestamp: new Date().toISOString()
  }, 'Failed to fetch user');
}

// ✅ コンテキスト付きロガー
const requestLogger = logger.child({
  requestId: req.id,
  userId: req.user?.id
});

requestLogger.info('Processing request');
requestLogger.error({ err: error }, 'Request failed');
```

**JSON出力例:**
```json
{
  "level": "error",
  "err": {
    "type": "DatabaseError",
    "message": "Connection timeout",
    "stack": "..."
  },
  "userId": "user_123",
  "operation": "getUser",
  "timestamp": "2026-01-24T10:30:00.000Z",
  "msg": "Failed to fetch user"
}
```

**ベストプラクティス:**
- JSON形式で構造化されたログ
- エラー、リクエストID、ユーザーIDなどのコンテキスト情報を含める
- ログレベルを適切に使い分ける（error, warn, info, debug）

## 3. カスタムエラークラスで詳細な情報を保持

### 問題点

標準のErrorクラスだけでは、エラーの種類や詳細情報を十分に表現できません。

```typescript
// ❌ エラーの種類が判別しづらい
throw new Error('User not found');
throw new Error('Database connection failed');
```

### 解決策: カスタムエラークラスの定義

```typescript
// ✅ カスタムエラークラス
export class AppError extends Error {
  constructor(
    public statusCode: number,
    public message: string,
    public isOperational = true,
    public errorCode?: string
  ) {
    super(message);
    Object.setPrototypeOf(this, AppError.prototype);
    Error.captureStackTrace(this, this.constructor);
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id?: string) {
    super(
      404,
      id ? `${resource} with id ${id} not found` : `${resource} not found`,
      true,
      'NOT_FOUND'
    );
  }
}

export class ValidationError extends AppError {
  constructor(
    message: string,
    public details?: Record<string, string[]>
  ) {
    super(400, message, true, 'VALIDATION_ERROR');
  }
}

export class DatabaseError extends AppError {
  constructor(message: string, public originalError?: Error) {
    super(500, message, false, 'DATABASE_ERROR');
  }
}

// ✅ カスタムエラーの使用
async function getUser(userId: string) {
  const user = await db.user.findUnique({ where: { id: userId } });

  if (!user) {
    throw new NotFoundError('User', userId);
  }

  return user;
}

// ✅ エラーハンドラーでの型判定
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  if (err instanceof AppError) {
    logger.error({
      err,
      statusCode: err.statusCode,
      errorCode: err.errorCode,
      isOperational: err.isOperational
    }, 'Application error');

    return res.status(err.statusCode).json({
      error: err.message,
      code: err.errorCode,
      ...(err instanceof ValidationError && { details: err.details })
    });
  }

  // 予期しないエラー
  logger.fatal({ err }, 'Unexpected error');
  res.status(500).json({ error: 'Internal server error' });
});
```

**ベストプラクティス:**
- エラーの種類ごとにクラスを定義
- HTTPステータスコード、エラーコードを含める
- isOperationalフラグで予期されたエラーかを判別

## 4. リトライ戦略で一時的な障害に対応

### 問題点

外部APIやデータベースの一時的な障害で、本来成功するはずの処理が失敗することがあります。

```typescript
// ❌ リトライなし - 一時的な障害で失敗
async function fetchExternalData(url: string) {
  const response = await fetch(url);
  return response.json();
}
```

### 解決策: エクスポネンシャルバックオフでリトライ

```typescript
// ✅ リトライ機能付き
async function fetchWithRetry<T>(
  fn: () => Promise<T>,
  options: {
    maxRetries?: number;
    initialDelay?: number;
    maxDelay?: number;
    backoffMultiplier?: number;
  } = {}
): Promise<T> {
  const {
    maxRetries = 3,
    initialDelay = 1000,
    maxDelay = 10000,
    backoffMultiplier = 2
  } = options;

  let lastError: Error;
  let delay = initialDelay;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;

      if (attempt === maxRetries) {
        break;
      }

      logger.warn({
        attempt: attempt + 1,
        maxRetries,
        delay,
        error: lastError.message
      }, 'Retry attempt failed, waiting before next retry');

      await sleep(delay);
      delay = Math.min(delay * backoffMultiplier, maxDelay);
    }
  }

  throw new Error(`Failed after ${maxRetries} retries: ${lastError!.message}`);
}

function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// ✅ 使用例
async function fetchExternalData(url: string) {
  return fetchWithRetry(
    async () => {
      const response = await fetch(url);
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }
      return response.json();
    },
    {
      maxRetries: 3,
      initialDelay: 1000
    }
  );
}
```

**リトライ戦略の例:**
| 試行 | 待機時間 |
|------|---------|
| 1回目 | 1秒 |
| 2回目 | 2秒 |
| 3回目 | 4秒 |
| 4回目 | 8秒 |

**ベストプラクティス:**
- エクスポネンシャルバックオフで負荷を分散
- 最大リトライ回数と最大待機時間を設定
- リトライログで問題を早期発見

## 5. サーキットブレーカーパターンで障害の波及を防ぐ

### 問題点

外部サービスがダウンした際、全リクエストが タイムアウトまで待機し、システム全体が停止する可能性があります。

### 解決策: サーキットブレーカーの実装

```typescript
import CircuitBreaker from 'opossum';

// ✅ サーキットブレーカーの設定
const options = {
  timeout: 3000, // 3秒でタイムアウト
  errorThresholdPercentage: 50, // エラー率50%でOPEN
  resetTimeout: 30000 // 30秒後にHALF_OPENに移行
};

// 外部API呼び出しをサーキットブレーカーでラップ
const fetchUserDataBreaker = new CircuitBreaker(
  async (userId: string) => {
    const response = await fetch(`https://api.example.com/users/${userId}`);
    if (!response.ok) {
      throw new Error(`API returned ${response.status}`);
    }
    return response.json();
  },
  options
);

// イベントリスナーでモニタリング
fetchUserDataBreaker.on('open', () => {
  logger.warn('Circuit breaker opened - too many failures');
});

fetchUserDataBreaker.on('halfOpen', () => {
  logger.info('Circuit breaker half-open - testing if service recovered');
});

fetchUserDataBreaker.on('close', () => {
  logger.info('Circuit breaker closed - service healthy');
});

// ✅ フォールバック処理
fetchUserDataBreaker.fallback((userId: string) => {
  logger.warn({ userId }, 'Using fallback data due to circuit breaker');
  return {
    id: userId,
    name: 'Unknown',
    isFallback: true
  };
});

// ✅ 使用例
app.get('/user/:id', async (req, res) => {
  try {
    const userData = await fetchUserDataBreaker.fire(req.params.id);
    res.json(userData);
  } catch (error) {
    logger.error({ err: error }, 'Failed to fetch user data');
    res.status(503).json({ error: 'Service temporarily unavailable' });
  }
});
```

**サーキットブレーカーの状態遷移:**
```
CLOSED (正常) → エラー率が閾値超過 → OPEN (遮断)
       ↑                                    ↓
       └── 成功 ←── HALF_OPEN (試行) ←── 一定時間経過
```

**ベストプラクティス:**
- 外部依存サービスにサーキットブレーカーを適用
- フォールバック処理で部分的なサービス提供を継続
- メトリクスをモニタリングし、障害を早期検知

## まとめ

本番環境で安定稼働するためのエラーハンドリング手法を5つ紹介しました。

| 手法 | 効果 | 実装難易度 |
|------|------|----------|
| 多層防御ハンドリング | クラッシュ防止 | 低 |
| 構造化ロギング | デバッグ効率化 | 低 |
| カスタムエラークラス | 詳細な情報保持 | 中 |
| リトライ戦略 | 一時的障害対応 | 中 |
| サーキットブレーカー | 障害波及防止 | 高 |

これらの手法を組み合わせることで、信頼性の高いNode.jsアプリケーションを構築できます。

### より深く学びたい方へ

この記事では、エラーハンドリングの基本的なパターンを紹介しましたが、実務ではさらに以下のような内容も重要です:

- フレームワーク別（Express/NestJS/Fastify）のエラーハンドリングパターン
- 分散トレーシングとエラー追跡
- エラー監視ツール（Sentry、Datadog）との統合
- グレースフルシャットダウンの実装
- ヘルスチェックとレディネスプローブ

これらの内容を体系的に学びたい方は、拙著「[Node.js開発完全ガイド 2026](https://zenn.dev/gaku52/books/nodejs-development-complete-guide-2026)」をご参照ください。エラーハンドリングからデプロイまで、20万文字超のボリュームで詳しく解説しています。

---

**関連記事**
- [Node.jsパフォーマンス最適化の実践 - レスポンス時間を最大67%改善する5つの手法](https://zenn.dev/gaku52/articles/nodejs-performance-optimization-5-methods)
- [Node.js初心者が必ず躓く非同期処理の罠5選と解決策](https://zenn.dev/gaku52/articles/nodejs-async-processing-pitfalls)
