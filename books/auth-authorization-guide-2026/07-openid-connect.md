---
title: "OpenID Connect"
---

# OpenID Connect

## この章で学ぶこと

OpenID Connect（OIDC）は、OAuth 2.0 の上に構築された**認証レイヤー**です。前章で学んだ OAuth 2.0 が「認可」のプロトコルであるのに対し、OIDC は「認証」を標準化します。

この章では以下のトピックを扱います。

- OIDC と OAuth 2.0 の関係と根本的な違い
- ID トークンの構造・クレーム・検証フロー
- UserInfo エンドポイントの役割と使い方
- OIDC Discovery による設定の自動取得
- 認証フロー（Authorization Code Flow + PKCE）
- 主要 OIDC プロバイダーの特徴と注意点

## OIDC と OAuth 2.0 の関係

### なぜ OIDC が必要なのか

OAuth 2.0 は「認可」のためのプロトコルであり、「認証」を目的としたものではありません。アクセストークンは「リソースへのアクセスを許可する」ことを意味しますが、「持ち主が誰であるか」は保証しません。

多くの開発者が OAuth 2.0 を認証に流用した結果、セキュリティ上の脆弱性が生じました。OIDC はこの問題を解決するために生まれた仕様です。

```
┌──────────────────────────────────┐
│         OpenID Connect           │
│  ┌──────────────────────────┐    │
│  │       OAuth 2.0          │    │
│  │  （認可フレームワーク）     │    │
│  └──────────────────────────┘    │
│  + ID Token（認証）              │
│  + UserInfo エンドポイント        │
│  + Discovery                    │
│  + Session Management           │
└──────────────────────────────────┘

OIDC = OAuth 2.0 + 認証の標準化
```

| 項目 | OAuth 2.0 | OpenID Connect |
|------|-----------|----------------|
| 目的 | 認可（Authorization） | 認証（Authentication） |
| 質問 | 「このアプリに何を許可しますか？」 | 「あなたは誰ですか？」 |
| 結果 | アクセストークン | ID トークン + アクセストークン |
| ユーザー識別 | 保証しない | 保証する |
| 仕様 | RFC 6749, 6750 | OpenID Connect Core 1.0 |

### OAuth 2.0 を認証に使う場合の問題

OAuth 2.0 を認証目的で流用すると、以下のような脆弱性が生じます。

- **トークン置換攻撃**: 攻撃者が悪意あるアプリで取得したアクセストークンを正規アプリに送信し、なりすましログインが成立します。OIDC では ID トークンの `aud` クレームで防止できます
- **アクセストークンの不透明性**: 形式が規定されておらず、ユーザー情報が含まれる保証がありません
- **認証時刻の保証がない**: 古いトークンが再利用される可能性があります（OIDC では `auth_time` で解決）
- **リプレイ攻撃**: アクセストークンにリプレイ防止の仕組みがありません（OIDC では `nonce` で解決）

## ID トークン

### ID トークンの構造

ID トークンは **JWT 形式** で、ユーザーの認証情報を含みます。アクセストークンとは異なり、クライアントアプリケーション内で消費されることを目的としています。

```json
{
  "iss": "https://accounts.google.com",
  "sub": "110169484474386276334",
  "aud": "my-client-id",
  "exp": 1700000000,
  "iat": 1699999100,
  "auth_time": 1699999000,
  "nonce": "random-nonce-value",
  "email": "alice@example.com",
  "email_verified": true,
  "name": "Alice Example",
  "picture": "https://example.com/alice.jpg"
}
```

| クレーム | 名称 | 説明 |
|---------|------|------|
| `iss` | Issuer | トークン発行者の URL |
| `sub` | Subject | ユーザーの一意識別子（IdP 内で一意） |
| `aud` | Audience | トークンの対象クライアント ID |
| `exp` | Expiration | 有効期限（UNIX タイムスタンプ） |
| `iat` | Issued At | 発行時刻（UNIX タイムスタンプ） |

:::message alert
**アクセストークンと ID トークンの混同は重大な脆弱性を生みます。** ID トークンを API アクセスに使用してはいけません。ID トークンはクライアント内で消費し、API にはアクセストークンを送信します。
:::

### ID トークンの検証

ID トークンの検証は OIDC セキュリティの要です。以下のステップを漏れなく実行してください。

1. **JWT の形式検証** -- `alg: "none"` は絶対に受け入れない
2. **署名の検証** -- IdP の公開鍵（JWKS エンドポイント）で検証
3. **`iss` の検証** -- 期待する IdP の URL と完全一致するか
4. **`aud` の検証** -- 自分の `client_id` が含まれているか
5. **`exp` の検証** -- 現在時刻が有効期限内か（クロックスキュー考慮）
6. **`nonce` の検証** -- セッションに保存した値と一致するか（リプレイ攻撃防止）
7. **`auth_time` の検証** -- `max_age` を指定した場合、認証時刻が範囲内か

```typescript
import { jwtVerify, createRemoteJWKSet } from 'jose';

const GOOGLE_JWKS = createRemoteJWKSet(
  new URL('https://www.googleapis.com/oauth2/v3/certs')
);

async function verifyIdToken(idToken: string, expectedNonce: string) {
  const { payload } = await jwtVerify(idToken, GOOGLE_JWKS, {
    issuer: 'https://accounts.google.com',
    audience: process.env.GOOGLE_CLIENT_ID!,
    algorithms: ['RS256'],
    clockTolerance: 5,
  });

  if (payload.nonce !== expectedNonce) {
    throw new Error('Invalid nonce - possible replay attack');
  }

  return {
    sub: payload.sub!,
    email: payload.email as string,
    name: payload.name as string,
    emailVerified: payload.email_verified as boolean,
  };
}
```

### JWKS（JSON Web Key Set）

IdP は署名検証用の公開鍵を JWKS エンドポイントで公開しています。JWT ヘッダーの `kid` と JWKS の `kid` を照合して正しい鍵を選択します。鍵は定期的にローテーションされるため、24 時間程度のキャッシュを設定し、`kid` が見つからない場合に再取得する方式が推奨されます。

## UserInfo エンドポイント

UserInfo エンドポイントは、アクセストークンを使ってユーザーの詳細なプロフィール情報を取得する API です。

- **ID トークン**: 認証時に最小限の情報を含む。認証時点のスナップショット
- **UserInfo**: `scope` で要求した詳細情報を返す。最新の情報を取得できる

```typescript
async function getUserInfo(accessToken: string) {
  const res = await fetch(
    'https://openidconnect.googleapis.com/v1/userinfo',
    { headers: { Authorization: `Bearer ${accessToken}` } }
  );
  if (!res.ok) throw new Error(`UserInfo failed: ${res.status}`);
  return res.json();
}
```

### 標準スコープとクレームの対応

OIDC では、リクエスト時の `scope` パラメータによって返されるクレームが決まります。

| スコープ | 返されるクレーム |
|---------|----------------|
| `openid` | `sub`（必須スコープ） |
| `profile` | `name`, `family_name`, `given_name`, `picture`, `locale` 等 |
| `email` | `email`, `email_verified` |
| `address` | `address`（構造化住所） |
| `phone` | `phone_number`, `phone_number_verified` |

一般的な Web アプリケーションでは `scope: "openid email profile"` を指定すれば、ユーザー ID、メールアドレス、表示名、アイコンが取得できます。

:::message
ID トークンの情報は認証時点のスナップショットです。ユーザーが IdP 側でプロフィールを変更した場合、最新情報の取得には UserInfo エンドポイントを呼び出す必要があります。
:::

## OIDC Discovery

OIDC Discovery は、IdP の設定情報を**標準化された形式で自動取得**する仕組みです。エンドポイント URL をハードコードする必要がなくなり、IdP の変更にも自動追従できます。

```
GET https://accounts.google.com/.well-known/openid-configuration
```

```json
{
  "issuer": "https://accounts.google.com",
  "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
  "token_endpoint": "https://oauth2.googleapis.com/token",
  "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
  "scopes_supported": ["openid", "email", "profile"],
  "response_types_supported": ["code", "token", "id_token"],
  "id_token_signing_alg_values_supported": ["RS256"]
}
```

Discovery を利用すると、複数の OIDC プロバイダーを同一のコードで扱えます。

```typescript
class OIDCProvider {
  private config: any = null;

  constructor(
    private issuerUrl: string,
    private clientId: string,
    private clientSecret: string
  ) {}

  async discover() {
    if (this.config) return this.config;
    const url = `${this.issuerUrl}/.well-known/openid-configuration`;
    const res = await fetch(url);
    this.config = await res.json();
    if (this.config.issuer !== this.issuerUrl) {
      throw new Error('Issuer mismatch');
    }
    return this.config;
  }

  async getAuthorizationUrl(redirectUri: string, state: string, nonce: string) {
    const config = await this.discover();
    const params = new URLSearchParams({
      response_type: 'code',
      client_id: this.clientId,
      redirect_uri: redirectUri,
      scope: 'openid email profile',
      state,
      nonce,
    });
    return `${config.authorization_endpoint}?${params}`;
  }
}

// 複数プロバイダーの一元管理
const providers = {
  google: new OIDCProvider('https://accounts.google.com', '...', '...'),
  microsoft: new OIDCProvider(
    'https://login.microsoftonline.com/common/v2.0', '...', '...'
  ),
};
```

設定は頻繁に変わらないため、24 時間程度のキャッシュでパフォーマンスを最適化できます。
## 認証フロー

### Authorization Code Flow + PKCE

OIDC で推奨される認証フローは **Authorization Code Flow with PKCE** です。

```
ブラウザ          サーバー           IdP
  │ ログインクリック  │                │
  │───────────────>│                │
  │                │ state/nonce/PKCE│
  │  302 Redirect  │                │
  │<───────────────│                │
  │ GET /authorize?response_type=code│
  │   &scope=openid&state=xxx       │
  │   &nonce=xxx&code_challenge=xxx │
  │────────────────────────────────>│
  │      ログイン画面 → 認証          │
  │<────────────────────────────────│
  │ 302 ?code=AUTH_CODE&state=xxx   │
  │<────────────────────────────────│
  │ GET /callback?code=...&state=...│
  │───────────────>│ state 検証      │
  │                │ POST /token     │
  │                │  code + verifier│
  │                │────────────────>│
  │                │ ID Token +      │
  │                │ Access Token    │
  │                │<────────────────│
  │                │ ID Token 検証   │
  │ Set-Cookie     │ セッション作成   │
  │<───────────────│                │
```

### 実装のポイント

**認証開始**

```typescript
import crypto from 'crypto';

// PKCE の code_verifier と code_challenge を生成
const codeVerifier = crypto.randomBytes(32).toString('base64url');
const codeChallenge = crypto
  .createHash('sha256')
  .update(codeVerifier)
  .digest('base64url');

// state, nonce をランダム生成し HttpOnly Cookie に保存
const state = crypto.randomUUID();
const nonce = crypto.randomUUID();
```

**コールバック処理**

```typescript
// state の検証（CSRF 防止）
if (state !== storedState) throw new Error('Invalid state');

// 認可コードをトークンに交換
const tokens = await exchangeCode(code, redirectUri, codeVerifier);

// ID トークンの検証（nonce を含む全ステップ）
const user = await verifyIdToken(tokens.idToken, storedNonce);

// セッションを作成し、一時的な Cookie を削除
```

**セキュリティ上の必須事項**

- `state` で CSRF を防止する
- `nonce` でリプレイ攻撃を防止する
- PKCE で認可コード横取り攻撃を防止する
- `redirect_uri` は完全一致で検証する（ワイルドカード禁止）
- `client_secret` をフロントエンドに含めない

## OIDC プロバイダー

### 主要プロバイダー比較

| プロバイダー | Discovery | PKCE | 特記事項 |
|-------------|-----------|------|----------|
| Google | 対応 | 対応 | 最も標準準拠 |
| Microsoft | 対応 | 対応 | テナント別 issuer URL |
| Apple | 対応 | 対応 | `name` は初回認可時のみ |
| GitHub | 非対応 | 対応 | OIDC 非準拠、独自 OAuth |
| LINE | 対応 | 非対応 | 日本市場で重要 |
| Auth0 | 対応 | 対応 | カスタマイズ性が高い |
| Keycloak | 対応 | 対応 | セルフホスト可能 |

### プロバイダー別の注意点

**Apple Sign In** -- `name` と `email` は初回認可時のみ返却されます。2回目以降は `sub` のみです。初回レスポンスを確実に DB へ保存してください。ユーザーがメールを非公開にした場合は `xxx@privaterelay.appleid.com` 形式のリレーメールが提供されます。

**GitHub** -- 標準 OIDC に準拠していません。ID トークンを返さず、ユーザー情報は `/user` API で取得します。`email` が `null` の場合は `/user/emails` API が必要です。

```typescript
// GitHub はユーザー情報を API で取得
const [userRes, emailsRes] = await Promise.all([
  fetch('https://api.github.com/user', {
    headers: { Authorization: `Bearer ${accessToken}` },
  }),
  fetch('https://api.github.com/user/emails', {
    headers: { Authorization: `Bearer ${accessToken}` },
  }),
]);
const emails = await emailsRes.json();
const primaryEmail = emails.find((e: any) => e.primary && e.verified);
```

**Microsoft / Azure AD** -- テナント別の issuer URL が存在します。マルチテナント対応では `common` エンドポイントを使いますが、ID トークンの `iss` にテナント ID が含まれるため検証ロジックに注意が必要です。

## よくあるアンチパターン

**1. ID トークンを API 認証に使用する**

```typescript
// NG: ID トークンで API アクセス
fetch('/api/data', { headers: { Authorization: `Bearer ${idToken}` } });
// OK: アクセストークンを使う
fetch('/api/data', { headers: { Authorization: `Bearer ${accessToken}` } });
```

**2. JWT の署名検証を省略する**

```typescript
// NG: 署名検証なしでペイロードを信頼
const payload = JSON.parse(atob(token.split('.')[1]));
// OK: 必ず署名を検証
const { payload } = await jwtVerify(token, JWKS, { issuer: '...', audience: '...' });
```

**3. `sub` 以外でユーザーを識別する** -- メールは変更される可能性があります。`sub` + `provider` の組み合わせを使いましょう。

**4. Implicit Flow を使用する** -- アクセストークンが URL フラグメントで公開されます。Authorization Code Flow + PKCE を使用してください。

## まとめ

| 概念 | ポイント |
|------|---------|
| OIDC | OAuth 2.0 上の認証レイヤー。認証と認可を明確に分離する |
| ID トークン | ユーザー情報を含む JWT。クライアント内で消費する |
| アクセストークン | API アクセス用。リソースサーバーに送信する |
| Discovery | プロバイダー設定の自動取得。24 時間キャッシュを推奨 |
| PKCE | 認可コード横取り攻撃を防止。すべてのクライアントで推奨 |
| `nonce` | リプレイ攻撃防止。ID トークンに含めて検証する |
| UserInfo | 詳細プロフィールの取得。アクセストークンで認証する |
| JWKS | 公開鍵の配布。鍵ローテーションへの対応が必要 |
| プロバイダー差異 | GitHub は OIDC 非準拠、Apple は初回のみ name 返却 |

## やってみよう！

以下の課題に取り組んで、OIDC の理解を深めましょう。

- [ ] Google の Discovery エンドポイント（`https://accounts.google.com/.well-known/openid-configuration`）にブラウザからアクセスし、返却される JSON の各フィールドの意味を確認してみましょう
- [ ] `jose` ライブラリを使って、Google OIDC の ID トークン検証コードを実装してみましょう。`iss`、`aud`、`nonce` の検証が正しく動作することをテストで確認してください
- [ ] Authorization Code Flow + PKCE を使った OIDC ログインを Node.js で実装し、`state`、`nonce`、`code_verifier` のセッション保存とコールバック検証を体験してください
- [ ] Google と GitHub 両対応のマルチプロバイダーログインを設計してみましょう。GitHub の OIDC 非準拠をどう抽象化するかがポイントです
