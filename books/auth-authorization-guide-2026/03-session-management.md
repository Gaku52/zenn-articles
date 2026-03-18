---
title: "セッション管理"
---

## この章で学ぶこと

HTTP はステートレスなプロトコルです。サーバーは個々のリクエストを独立して処理するため、「同じユーザーからの連続したリクエスト」を識別できません。この問題を解決するのが **Cookie** と **セッション** の組み合わせです。

この章では、以下のトピックを扱います。

- Cookie の仕組みと送受信フロー
- セッション ID の安全な生成方法
- セッションストア（Redis / データベース）の選び方
- Cookie のセキュリティ属性（HttpOnly / Secure / SameSite）
- セッション固定攻撃とその対策

---

## Cookie の仕組み

Cookie は、サーバーがレスポンスヘッダー `Set-Cookie` でブラウザに値を渡し、ブラウザがそれ以降のリクエストで自動的に `Cookie` ヘッダーとして返す仕組みです。

```
サーバー → ブラウザ（レスポンス）:
  Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=3600

ブラウザ → サーバー（次回以降のリクエスト）:
  Cookie: session_id=abc123
```

この流れを図にすると、以下のようになります。

```
┌──────────┐                           ┌──────────┐
│ ブラウザ  │  POST /login              │ サーバー  │
│          │ ─────────────────────────> │          │
│          │  Set-Cookie: sid=abc123    │ セッション │
│          │ <───────────────────────── │ 作成      │
│          │                           │          │
│          │  GET /dashboard            │          │
│          │  Cookie: sid=abc123        │          │
│          │ ─────────────────────────> │          │
│          │  200 OK (認証済み)          │ 「ユーザー │
│          │ <───────────────────────── │  Aだ」    │
└──────────┘                           └──────────┘
```

ポイントは **ブラウザが Cookie を自動送信する** ことです。開発者がフロントエンドで毎回トークンを付与するコードを書く必要はありません。Cookie の送信条件は、ドメイン・パス・SameSite 属性などで制御されます。

---

## セッション ID の生成

セッション管理の要は「セッション ID」です。このランダムな文字列を Cookie に保存し、サーバー側のストアに保存されたユーザー情報と紐づけます。

### セッション ID に求められる要件

OWASP のガイドラインでは、セッション ID には以下の要件が求められます。

| 要件 | 内容 |
|------|------|
| エントロピー | 最低 128 ビット、推奨 256 ビット |
| 予測不可能性 | 暗号論的擬似乱数生成器（CSPRNG）を使用する |
| 一意性 | 衝突が事実上発生しないこと |

### 安全な生成方法

Node.js では `crypto.randomBytes()` を使います。

```typescript
import crypto from "crypto";

// 256 ビット（32 バイト）のランダム値を 16 進数文字列に変換
const sessionId = crypto.randomBytes(32).toString("hex");
// 例: "a3f1b9c8d7e6f5a4b3c2d1e0f9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f1a0"
```

### 危険な生成方法

以下のような方法は絶対に避けてください。

```typescript
// Math.random() は予測可能（V8 の xorshift128+ は逆算可能）
const bad1 = Math.random().toString(36).substr(2);

// タイムスタンプは推測可能
const bad2 = Date.now().toString(36);

// ユーザー情報を含めると情報漏洩 + 推測可能
const bad3 = `user_${userId}_${timestamp}`;
```

`Math.random()` はエントロピーが約 52 ビットしかなく、暗号論的に安全ではありません。必ず `crypto.randomBytes()` や同等の CSPRNG を使いましょう。

---

## セッションストア

セッション ID をキーとして、ユーザー情報を保存する場所が **セッションストア** です。主な選択肢は Redis とデータベースの 2 つです。

### セッションストアの比較

| ストア | 速度 | スケール | TTL 自動管理 | 永続化 | 用途 |
|--------|------|----------|-------------|--------|------|
| メモリ（Map） | 最速 | 不可 | 手動 | 不可 | 開発環境のみ |
| Redis | 高速 | 可能 | あり | 条件付き | **本番推奨** |
| PostgreSQL / MySQL | 中程度 | 可能 | 手動 | あり | 追加インフラを避けたい場合 |
| DynamoDB | 高速 | 自動 | あり | あり | AWS 環境 |

### Redis をセッションストアに使う理由

Redis はインメモリ型のデータストアで、以下の点がセッション管理に適しています。

1. **TTL の自動管理** -- `SETEX` コマンドで有効期限を設定すると、期限切れのデータは自動削除されます
2. **高速な読み書き** -- 全リクエストでセッションを参照するため、レイテンシの低さは重要です（Redis は p50 で 0.1 - 0.5 ms）
3. **スケーリング対応** -- Redis Sentinel による高可用性構成や、Redis Cluster による水平分散が可能です

### Redis セッションストアの実装例

```typescript
import Redis from "ioredis";
import crypto from "crypto";

const redis = new Redis(process.env.REDIS_URL!);
const SESSION_PREFIX = "sess:";
const SESSION_TTL = 1800; // 30 分（スライディング有効期限）

// セッション作成
async function createSession(userId: string, role: string): Promise<string> {
  const sessionId = crypto.randomBytes(32).toString("hex");
  const data = JSON.stringify({
    userId,
    role,
    createdAt: Date.now(),
    lastAccessedAt: Date.now(),
  });
  await redis.setex(SESSION_PREFIX + sessionId, SESSION_TTL, data);
  return sessionId;
}

// セッション取得（アクセスのたびに TTL をリセット）
async function getSession(sessionId: string) {
  const key = SESSION_PREFIX + sessionId;
  const data = await redis.get(key);
  if (!data) return null;

  // スライディング有効期限: TTL をリセット
  await redis.expire(key, SESSION_TTL);
  return JSON.parse(data);
}

// セッション破棄
async function destroySession(sessionId: string) {
  await redis.del(SESSION_PREFIX + sessionId);
}
```

### データベースセッションストアの場合

Redis を導入できない環境では、RDB にセッションテーブルを作成します。

```sql
CREATE TABLE sessions (
  id            TEXT PRIMARY KEY,
  session_token TEXT UNIQUE NOT NULL,
  user_id       TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  data          JSONB,
  expires_at    TIMESTAMPTZ NOT NULL,
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  last_accessed_at TIMESTAMPTZ
);

CREATE INDEX idx_sessions_user_id ON sessions(user_id);
CREATE INDEX idx_sessions_expires_at ON sessions(expires_at);
```

Redis と違い、期限切れのレコードは自動削除されません。定期的にクリーンアップするバッチ処理が必要です。

```sql
DELETE FROM sessions WHERE expires_at < NOW();
```

### 有効期限のベストプラクティス

有効期限には **スライディング** と **絶対** の 2 種類があります。推奨はこの 2 つを組み合わせた **ハイブリッド方式** です。

| 方式 | 説明 | 例 |
|------|------|------|
| スライディング | アクセスのたびに期限をリセットする | 30 分操作なしでログアウト |
| 絶対 | セッション作成時から固定時間で失効する | 24 時間後に強制ログアウト |
| **ハイブリッド** | 両方を組み合わせ、早い方で失効させる | **スライディング 30 分 + 絶対 24 時間** |

スライディングだけだと、攻撃者がセッションを乗っ取った場合に永続的にアクセスし続けられます。絶対有効期限と組み合わせることで、被害の時間を制限できます。

---

## Cookie のセキュリティ属性

Cookie にはセキュリティを強化するための属性が複数あります。セッション Cookie には必ず以下を設定してください。

### 属性一覧

| 属性 | 推奨値 | 目的 |
|------|--------|------|
| **HttpOnly** | `true` | JavaScript（`document.cookie`）からアクセス不可にする。XSS でのトークン窃取を防止します |
| **Secure** | `true` | HTTPS 通信でのみ Cookie を送信します。中間者攻撃を防止します |
| **SameSite** | `Lax` | クロスサイトリクエストでの Cookie 送信を制御します。CSRF 防御に有効です |
| **Path** | `/` | Cookie が送信されるパス範囲を指定します |
| **Max-Age** | `86400` | Cookie の有効期間（秒）を設定します |

### SameSite 属性の詳細

SameSite は 3 つの値を取ります。

**`Strict`** -- 同一サイトからのリクエストでのみ Cookie を送信します。外部サイトからのリンク遷移でも Cookie は送られないため、Google 検索結果からアクセスすると未ログイン状態になってしまいます。

**`Lax`（推奨）** -- トップレベルナビゲーション（リンク遷移）の GET では Cookie を送信しますが、クロスサイトの POST / iframe / img 等では送信しません。UX を損なわずに CSRF を防御できるバランスの良い設定です。

**`None`** -- すべてのクロスサイトリクエストで Cookie を送信します。`Secure` 属性が必須です。サードパーティ Cookie として使用する場合にのみ指定します。

### Cookie 設定の実装例

```typescript
// Express での設定
app.use(session({
  name: "__Host-sid",
  secret: process.env.SESSION_SECRET!,
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax",
    maxAge: 24 * 60 * 60 * 1000, // 24 時間
  },
}));
```

### Cookie Prefix

Cookie 名に `__Host-` や `__Secure-` を付けることで、さらに安全性を高められます。

| プレフィックス | 要件 |
|---------------|------|
| `__Secure-` | `Secure` 属性が必須。HTTPS でのみ設定可能 |
| `__Host-`（最も厳格） | `Secure` 必須、`Domain` 指定不可、`Path=/` 必須 |

`__Host-` を使うと、サブドメインからの Cookie 上書き攻撃（Cookie Tossing）を防止できます。

```
Set-Cookie: __Host-sid=abc123; Secure; Path=/
```

---

## セッション固定攻撃と対策

### 攻撃の仕組み

セッション固定攻撃（Session Fixation）は、攻撃者が事前に取得したセッション ID を被害者に使わせる攻撃です。

```
攻撃者                         被害者                       サーバー
  │                             │                           │
  │ (1) セッション ID を取得     │                           │
  │ ─────────────────────────────────────────────────────>  │
  │ session_id = "known_id"     │                           │
  │ <─────────────────────────────────────────────────────  │
  │                             │                           │
  │ (2) リンクを送信             │                           │
  │ ──────────────────────────> │                           │
  │                             │ (3) ログイン               │
  │                             │ (session_id=known_id)     │
  │                             │ ──────────────────────>   │
  │                             │                           │
  │ (4) 同じ session_id で      │                           │
  │     アクセス                 │                           │
  │ ─────────────────────────────────────────────────────>  │
  │ ログイン済み状態で侵入!      │                           │
```

### 対策: セッション ID のローテーション

最も重要な対策は、**ログイン成功時にセッション ID を再生成する** ことです。

```typescript
async function handleLogin(req: Request, res: Response) {
  const user = await authenticateUser(req.body.email, req.body.password);
  if (!user) {
    return res.status(401).json({ error: "Invalid credentials" });
  }

  // (1) 既存セッションがあれば破棄する
  const oldSessionId = req.cookies["__Host-sid"];
  if (oldSessionId) {
    await destroySession(oldSessionId);
  }

  // (2) 新しいセッション ID を生成する
  const newSessionId = await createSession(user.id, user.role);

  // (3) 新しい Cookie を設定する
  res.cookie("__Host-sid", newSessionId, {
    httpOnly: true,
    secure: true,
    sameSite: "lax",
    path: "/",
    maxAge: 24 * 60 * 60 * 1000,
  });

  return res.json({ user: { id: user.id, email: user.email } });
}
```

### その他の対策

セッション固定攻撃を包括的に防ぐには、以下も合わせて実施します。

1. **URL パラメータからのセッション ID 受け入れを拒否する** -- セッション ID は Cookie のみで管理します
2. **ログイン前のセッションデータを新セッションにコピーしない** -- ログイン前後で完全にセッションを分離します
3. **権限変更時にもセッション ID を再生成する** -- ロールの昇格時にも ID をローテーションします

---

## ログアウトの実装

ログアウト時は、Cookie の削除だけでは不十分です。**サーバー側のセッションデータも必ず削除** してください。

```typescript
app.post("/api/auth/logout", async (req, res) => {
  const sessionId = req.cookies["__Host-sid"];

  if (sessionId) {
    // (1) サーバー側のセッションデータを削除する
    await destroySession(sessionId);
  }

  // (2) Cookie を無効化する
  res.cookie("__Host-sid", "", {
    httpOnly: true,
    secure: true,
    sameSite: "lax",
    path: "/",
    maxAge: 0, // 即座に失効
  });

  // (3) キャッシュを無効化する
  res.set("Clear-Site-Data", '"cache", "cookies", "storage"');

  return res.json({ success: true });
});
```

Cookie のみを削除して、サーバー側のセッションを残してしまうと、攻撃者がそのセッション ID を知っている場合にアクセスを継続できてしまいます。これはよくあるアンチパターンです。

---

## まとめ

| 項目 | ベストプラクティス |
|------|--------------------|
| Cookie 属性 | `HttpOnly` + `Secure` + `SameSite=Lax` + `__Host-` プレフィックス |
| セッション ID 生成 | `crypto.randomBytes(32)` で 256 ビットのランダム値を使用する |
| セッションストア | 本番は Redis、開発はメモリ、Redis 不可なら DB |
| 有効期限 | スライディング（30 分）+ 絶対（24 時間）のハイブリッド |
| セッション固定攻撃対策 | ログイン成功時にセッション ID を再生成する |
| ログアウト | サーバー側のセッション削除 + Cookie 無効化 + `Clear-Site-Data` |
| SameSite | `Lax` を基本とし、必要に応じて CSRF トークンも併用する |

---

## やってみよう!

以下のステップで、セッション管理を自分の手で実装してみましょう。

- [ ] Express または Hono で `/login` エンドポイントを作り、`crypto.randomBytes(32)` でセッション ID を生成してみる
- [ ] Cookie に `HttpOnly`、`Secure`、`SameSite=Lax` を設定し、ブラウザの開発者ツール（Application > Cookies）で属性を確認する
- [ ] Redis（またはメモリの Map）にセッションデータを保存し、別のリクエストで Cookie から取得できることを確認する
- [ ] ログイン成功時にセッション ID を再生成（ローテーション）するコードを書き、旧 ID ではアクセスできなくなることを検証する
- [ ] ログアウト時にサーバー側のセッション削除と Cookie 無効化の両方を行い、ログアウト後に同じ Cookie でアクセスしても 401 が返ることを確認する
