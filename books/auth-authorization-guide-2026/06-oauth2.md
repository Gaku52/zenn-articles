---
title: "OAuth 2.0"
---

# OAuth 2.0

## この章で学ぶこと

OAuth 2.0 は、**第三者アプリケーションにユーザーのリソースへの限定的なアクセスを許可する**ための標準プロトコルです。たとえば「GitHub のリポジトリ一覧を MyApp に表示したい」というとき、ユーザーは GitHub のパスワードを MyApp に渡す必要はありません。OAuth 2.0 を使えば、必要な権限だけを安全に委譲できます。

この章では以下の内容を扱います。

- OAuth 2.0 の役割と 4 つのアクター
- Authorization Code Flow（+ PKCE）
- Client Credentials Flow
- スコープによる権限制御
- アクセストークンの扱い方

## OAuth 2.0 の役割と 4 つのアクター

OAuth 2.0 は**認可（Authorization）**のプロトコルです。「認証（あなたは誰か？）」ではなく、「認可（何を許可するか？）」を扱います。認証の仕組みは次章の OpenID Connect が担当します。

### 4 つのアクター

OAuth 2.0 には 4 つの登場人物がいます。

| アクター | 役割 | 具体例 |
|---|---|---|
| **Resource Owner**（リソースオーナー） | リソースの所有者。アクセスを許可する人 | ユーザー本人 |
| **Client**（クライアント） | リソースへのアクセスを要求するアプリ | MyApp（Web アプリ・モバイルアプリ） |
| **Authorization Server**（認可サーバー） | アクセス許可（トークン）を発行するサーバー | GitHub の OAuth サーバー |
| **Resource Server**（リソースサーバー） | 保護されたリソースを持つサーバー | GitHub API |

ポイントは、**ユーザーのパスワードが MyApp に渡らない**ことです。MyApp が受け取るのはアクセストークンだけであり、しかもそのトークンには限定的な権限（スコープ）しか付与されません。

### クライアントの分類

OAuth 2.0 ではクライアントを 2 種類に分類します。この分類はフローの選択に直結します。

| 種類 | 特徴 | 例 |
|---|---|---|
| **Confidential Client** | `client_secret` を安全に保持できる | サーバーサイド Web アプリ |
| **Public Client** | `client_secret` を安全に保持できない | SPA、モバイルアプリ |

SPA やモバイルアプリのソースコードはユーザーの手元にあるため、`client_secret` を埋め込んでも容易に取得されてしまいます。このため Public Client には PKCE という追加の保護が必要です。

## Authorization Code Flow

Authorization Code Flow は**最も安全で最も一般的なフロー**です。サーバーサイド Web アプリで標準的に使われます。

### フローの全体像

```
ユーザー      クライアント    認可サーバー     リソースサーバー
  │             │             │               │
  │ ログイン     │             │               │
  │────────────>│             │               │
  │             │ (1) 認可リクエスト            │
  │             │────────────>│               │
  │ (2) ログイン + 同意画面   │               │
  │<─────────────────────────│               │
  │ (3) 承認    │             │               │
  │─────────────────────────>│               │
  │ (4) 認可コード            │               │
  │────────────>│             │               │
  │             │ (5) code → token 交換       │
  │             │────────────>│               │
  │             │ (6) access_token            │
  │             │<────────────│               │
  │             │ (7) Bearer token            │
  │             │────────────────────────────>│
  │             │ (8) リソース                 │
  │             │<────────────────────────────│
  │<────────────│             │               │
```

### 各ステップの詳細

**(1) 認可リクエスト**

クライアントはユーザーのブラウザを認可サーバーにリダイレクトします。

```
GET /authorize?
  response_type=code
  &client_id=my-app-id
  &redirect_uri=https://myapp.com/callback
  &scope=read:user repo
  &state=xyzabc123
```

主要なパラメータは以下のとおりです。

| パラメータ | 説明 |
|---|---|
| `response_type=code` | 認可コードを要求することを指定します |
| `client_id` | アプリケーションの識別子です |
| `redirect_uri` | 認可後のコールバック先です。事前登録した URI と完全一致が必須です |
| `scope` | 要求する権限です（後述） |
| `state` | CSRF 防御用のランダム値です。コールバック時に一致を確認します |

**(2)-(3) ユーザーの認証と同意**

認可サーバーはログイン画面を表示し、ユーザーが認証します。その後、「MyApp にリポジトリへのアクセスを許可しますか？」という同意画面が表示されます。

**(4) 認可コードの発行**

ユーザーが許可すると、認可サーバーは短寿命の**認可コード（authorization code）**を発行し、`redirect_uri` にリダイレクトします。

```
HTTP/1.1 302 Found
Location: https://myapp.com/callback
  ?code=SplxlOBeZQQYbYS6WxSbIA
  &state=xyzabc123
```

認可コードには以下の特徴があります。

- 有効期限が非常に短い（推奨: 30 秒 ~ 1 分）
- 一度だけ使用可能（使用済みになると無効化）
- これ単体ではリソースにアクセスできない

**(5)-(6) トークン交換**

クライアントのサーバーサイドで、認可コードをアクセストークンに交換します。この通信はサーバー間（バックチャネル）で行われるため、`client_secret` を安全に送信できます。

```typescript
// サーバーサイドでのトークン交換
const tokenResponse = await fetch(
  "https://github.com/login/oauth/access_token",
  {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Accept: "application/json",
    },
    body: JSON.stringify({
      client_id: process.env.GITHUB_CLIENT_ID,
      client_secret: process.env.GITHUB_CLIENT_SECRET,
      code: authorizationCode,
      redirect_uri: "https://myapp.com/callback",
    }),
  }
);

const { access_token, refresh_token, scope } =
  await tokenResponse.json();
```

**(7)-(8) API リクエスト**

取得したアクセストークンを `Authorization` ヘッダーに付けて API を呼び出します。

```typescript
const response = await fetch("https://api.github.com/user", {
  headers: {
    Authorization: `Bearer ${access_token}`,
  },
});
```

## Authorization Code Flow + PKCE

SPA やモバイルアプリは `client_secret` を安全に保持できません。そこで登場するのが **PKCE（Proof Key for Code Exchange、ピクシーと読みます）** です。

### PKCE が防ぐ攻撃

PKCE がないと、悪意あるアプリが認可コードを横取りしてトークンを不正取得できてしまいます。たとえばモバイルアプリではカスタム URL スキームの乗っ取りにより、リダイレクト時の認可コードが傍受される可能性があります。PKCE があれば、認可コードを傍受されても `code_verifier` を知らない攻撃者はトークン交換に失敗します。

### PKCE の仕組み

PKCE は「**認可リクエストを送った本人だけがトークン交換できる**」ことを保証します。

1. クライアントがランダム文字列 `code_verifier` を生成します
2. そのハッシュ `code_challenge = BASE64URL(SHA256(code_verifier))` を計算します
3. 認可リクエスト時に `code_challenge` を送信します
4. トークン交換時に `code_verifier` を送信します
5. 認可サーバーが `SHA256(code_verifier) == code_challenge` を検証します

攻撃者は認可コードを傍受できても、`code_verifier` を知らないためトークン交換に失敗します。SHA-256 は一方向関数なので、`code_challenge` から `code_verifier` を逆算することは計算上不可能です。

### PKCE 付きフローの実装

```typescript
// code_verifier と code_challenge の生成
async function generatePKCE() {
  const verifier = base64URLEncode(
    crypto.getRandomValues(new Uint8Array(32))
  );
  const data = new TextEncoder().encode(verifier);
  const hash = await crypto.subtle.digest("SHA-256", data);
  const challenge = base64URLEncode(new Uint8Array(hash));
  return { verifier, challenge };
}

// 認可リクエスト（PKCE 付き）
async function startAuthFlow() {
  const { verifier, challenge } = await generatePKCE();
  const state = crypto.randomUUID();

  sessionStorage.setItem("pkce_verifier", verifier);
  sessionStorage.setItem("oauth_state", state);

  const params = new URLSearchParams({
    response_type: "code",
    client_id: "my-spa-client-id",
    redirect_uri: "https://myapp.com/callback",
    scope: "openid profile email",
    state,
    code_challenge: challenge,
    code_challenge_method: "S256",
  });
  window.location.href =
    `https://auth.example.com/authorize?${params}`;
}

// コールバック処理（トークン交換）
async function handleCallback(code: string, state: string) {
  if (state !== sessionStorage.getItem("oauth_state")) {
    throw new Error("state が一致しません");
  }
  // client_secret は不要。代わりに code_verifier を送信
  const response = await fetch("https://auth.example.com/token", {
    method: "POST",
    headers: {
      "Content-Type": "application/x-www-form-urlencoded",
    },
    body: new URLSearchParams({
      grant_type: "authorization_code",
      code,
      redirect_uri: "https://myapp.com/callback",
      client_id: "my-spa-client-id",
      code_verifier: sessionStorage.getItem("pkce_verifier")!,
    }),
  });
  return response.json();
}
```

:::message
OAuth 2.1（策定中）では、Authorization Code Flow で **PKCE が必須** になります。Confidential Client であっても PKCE を付けることが推奨されています。新規実装では常に PKCE を使いましょう。
:::

## Client Credentials Flow

Client Credentials Flow は**ユーザーが介在しない、サーバー間通信**で使います。バッチ処理やマイクロサービス間の API 呼び出しが典型的なユースケースです。

### フローの全体像

```
クライアント        認可サーバー       リソースサーバー
  │                  │                │
  │ client_id +      │                │
  │ client_secret    │                │
  │─────────────────>│                │
  │ access_token     │                │
  │<─────────────────│                │
  │ Bearer token で API リクエスト    │
  │──────────────────────────────────>│
  │ リソース          │                │
  │<──────────────────────────────────│
```

ユーザーの同意画面がない点が Authorization Code Flow との最大の違いです。リフレッシュトークンは発行されません。

### 実装例

```typescript
// Client Credentials Flow の基本実装
async function getServiceToken(): Promise<string> {
  const response = await fetch("https://auth.example.com/token", {
    method: "POST",
    headers: {
      "Content-Type": "application/x-www-form-urlencoded",
      Authorization: `Basic ${btoa(
        `${CLIENT_ID}:${CLIENT_SECRET}`
      )}`,
    },
    body: new URLSearchParams({
      grant_type: "client_credentials",
      scope: "read:data write:data",
    }),
  });

  const { access_token } = await response.json();
  return access_token;
}
```

本番環境ではトークンのキャッシュと有効期限の管理が重要です。トークンが期限切れになる少し前（例: 60 秒前）に新しいトークンを取得するようにしましょう。

## スコープによる権限制御

スコープは**アクセストークンに付与される権限の範囲**を定義します。最小権限の原則に従い、アプリケーションが必要とする最小限のスコープだけを要求することが重要です。

### スコープの設計パターン

```
GitHub の例:
  repo           → リポジトリ全般（読取・書込・削除）
  read:user      → ユーザープロフィールの読取
  user:email     → メールアドレスの読取
  admin:org      → 組織の管理

Google の例:
  https://www.googleapis.com/auth/calendar
  https://www.googleapis.com/auth/calendar.readonly
  https://www.googleapis.com/auth/gmail.send

resource:action 形式の例:
  users:read     → ユーザー一覧の閲覧
  users:write    → ユーザー情報の更新
  posts:read     → 記事の閲覧
  posts:publish  → 記事の公開
```

### 同意画面とスコープ

ユーザーにはスコープの内容がわかりやすく表示されます。過剰なスコープを要求すると、ユーザーが不信感を持ちアプリの利用を避ける原因になります。

```
┌─────────────────────────────────────┐
│  MyApp がアクセスを要求しています      │
│                                      │
│  ・プロフィール情報の閲覧              │
│  ・メールアドレスの読取                │
│  ・リポジトリの読取                    │
│                                      │
│  [許可する]  [拒否する]                │
└─────────────────────────────────────┘
```

## アクセストークン

アクセストークンは、リソースサーバーへの API リクエスト時に使用する**一時的な認証情報**です。

### アクセストークンの特徴

| 項目 | 内容 |
|---|---|
| 形式 | JWT（JSON Web Token）やランダム文字列など、認可サーバーの実装次第 |
| 有効期限 | 短め（一般的に 15 分 ~ 1 時間） |
| 送信方法 | `Authorization: Bearer <token>` ヘッダー |
| 保存場所（SPA） | メモリ上が推奨。`localStorage` はXSS に弱いため非推奨 |
| リフレッシュ | リフレッシュトークンを使って新しいアクセストークンを取得 |

### トークンレスポンスの例

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "8xLOxBtZp8",
  "scope": "read:user repo"
}
```

レスポンスヘッダーには `Cache-Control: no-store` が設定され、トークンがキャッシュされないようになっています。

### フローの選択指針

アプリケーションの種類によって適切なフローが異なります。

| アプリケーション | 推奨フロー |
|---|---|
| Web アプリ（サーバーあり） | Authorization Code Flow |
| SPA（サーバーなし） | Authorization Code Flow + PKCE |
| モバイルアプリ | Authorization Code Flow + PKCE |
| サーバー間通信 | Client Credentials Flow |

## まとめ

| 項目 | 内容 |
|---|---|
| OAuth 2.0 の役割 | 第三者アプリへの**認可の委譲**。パスワードを渡さずに権限を付与します |
| 4 つのアクター | Resource Owner / Client / Authorization Server / Resource Server |
| Authorization Code Flow | 認可コードを経由してトークンを取得する最も安全なフローです |
| PKCE | SPA・モバイルで必須。`code_verifier` のハッシュ検証で認可コード横取りを防ぎます |
| Client Credentials Flow | サーバー間通信用。ユーザー不在で `client_id` + `client_secret` のみで認証します |
| スコープ | アクセストークンの権限範囲を制限します。最小権限の原則が重要です |
| アクセストークン | 短寿命の一時的な認証情報。`Bearer` ヘッダーで送信します |

## やってみよう！

1. **GitHub OAuth App を登録してみましょう。** [GitHub Developer Settings](https://github.com/settings/developers) で OAuth App を作成し、`client_id` と `client_secret` を取得してください。`redirect_uri` には `http://localhost:3000/callback` を設定します。

2. **Authorization Code Flow を実装してみましょう。** Node.js（Express）で簡単なサーバーを立て、GitHub OAuth を使ったログインフローを実装してください。認可リクエスト、コールバック、トークン交換の 3 つのエンドポイントが必要です。

3. **PKCE を追加してみましょう。** 上記の実装に PKCE を追加してください。`code_verifier` の生成、`code_challenge` の計算、トークン交換時の `code_verifier` 送信を実装します。

4. **スコープを変えて動作を確認しましょう。** `scope` パラメータを `read:user` だけにした場合と `repo` を追加した場合で、取得できるデータがどう変わるか試してみてください。
