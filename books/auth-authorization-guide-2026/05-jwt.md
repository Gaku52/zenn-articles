---
title: "JWT（JSON Web Token）"
---

## この章で学ぶこと

前章まででセッション管理とCSRF対策を学びました。この章では、モダンな認証の中核技術である **JWT（JSON Web Token）** を取り上げます。

- JWTの3部構造（Header / Payload / Signature）を理解する
- 署名アルゴリズム（HS256 / RS256）の違いと選び方を把握する
- 登録済みクレームとカスタムクレームの設計を学ぶ
- 有効期限の管理と検証フローを実装できるようになる
- セッション方式とJWT方式の使い分けを判断できるようになる

---

## 1. JWTとは何か

JWT（JSON Web Token、RFC 7519）は、2者間で情報を安全にやり取りするためのコンパクトなトークン形式です。JSON形式のデータに電子署名を付与し、改ざんを検知できるようにしたものです。

最大の特徴は **ステートレス** であることです。トークン自体に必要な情報がすべて含まれているため、サーバー側でセッション情報を保持する必要がありません。

:::message
**重要**: JWTは「署名」であり「暗号化」ではありません。ペイロードはBase64URLエンコードされているだけなので、誰でもデコードして中身を読めます。機密情報をペイロードに含めてはいけません。
:::

---

## 2. JWTの構造（Header / Payload / Signature）

JWTは **ドット（`.`）で区切られた3つのパート** で構成されています。

```
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0.SflKxwRJSM...

[Header].[Payload].[Signature]
```

### 2.1 Header（ヘッダー）

トークンの種類と署名アルゴリズムを指定します。

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "key-id-1"
}
```

| フィールド | 意味 | 例 |
|-----------|------|-----|
| `alg` | 署名アルゴリズム | `"HS256"`, `"RS256"` |
| `typ` | トークンタイプ | `"JWT"` |
| `kid` | 鍵ID（複数鍵の管理用） | `"key-2026-01"` |

### 2.2 Payload（ペイロード）

**クレーム（Claims）** と呼ばれるデータが格納されます。

```json
{
  "sub": "user_123",
  "iss": "https://auth.example.com",
  "aud": "https://api.example.com",
  "exp": 1700000000,
  "iat": 1699999100,
  "role": "admin"
}
```

### 2.3 Signature（署名）

ヘッダーとペイロードを連結した文字列に対して秘密鍵で署名した結果です。

```
Signature = ALGORITHM(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  秘密鍵
)
```

ペイロードの1文字でも変更されると署名の検証に失敗します。

### 2.4 JWTの生成フロー

```
Step 1: ヘッダーをJSON → Base64URLエンコード
Step 2: ペイロードをJSON → Base64URLエンコード
Step 3: 両者を "." で連結
Step 4: 連結文字列に秘密鍵で署名 → Base64URLエンコード
Step 5: ヘッダー.ペイロード.署名 を連結 → JWT完成
```

---

## 3. 署名アルゴリズム（HS256 / RS256）

署名アルゴリズムは **対称鍵方式** と **非対称鍵方式** の2種類に分かれます。

### 3.1 HS256（HMAC-SHA256）— 対称鍵方式

署名と検証に **同じ秘密鍵** を使う方式です。

- 処理が高速です
- 仕組みがシンプルで実装が容易です
- 検証する側にも秘密鍵を渡す必要があり、漏洩時に偽造が可能になります

```typescript
import { SignJWT, jwtVerify } from 'jose';

const secret = new TextEncoder().encode('最低32バイト以上の秘密鍵を設定');

const token = await new SignJWT({ sub: 'user_123', role: 'user' })
  .setProtectedHeader({ alg: 'HS256' })
  .setIssuedAt()
  .setExpirationTime('15m')
  .sign(secret);

const { payload } = await jwtVerify(token, secret, {
  algorithms: ['HS256'],
});
```

### 3.2 RS256（RSA-SHA256）— 非対称鍵方式

**秘密鍵で署名し、公開鍵で検証する** 方式です。

- 検証側に秘密鍵を渡す必要がありません
- マイクロサービス構成に適しています
- HS256と比べて署名が遅く、鍵・署名サイズが大きくなります

```typescript
import { SignJWT, jwtVerify, generateKeyPair } from 'jose';

const { publicKey, privateKey } = await generateKeyPair('RS256');

// 秘密鍵で署名
const token = await new SignJWT({ sub: 'user_123', role: 'admin' })
  .setProtectedHeader({ alg: 'RS256', kid: 'key-2026-01' })
  .setIssuer('https://auth.example.com')
  .setAudience('https://api.example.com')
  .setIssuedAt()
  .setExpirationTime('15m')
  .sign(privateKey);

// 公開鍵で検証
const { payload } = await jwtVerify(token, publicKey, {
  algorithms: ['RS256'],
  issuer: 'https://auth.example.com',
  audience: 'https://api.example.com',
});
```

### 3.3 アルゴリズムの比較

| 項目 | HS256 | RS256 |
|------|-------|-------|
| 鍵の種類 | 対称鍵（共有秘密鍵） | 非対称鍵（公開鍵 + 秘密鍵） |
| 署名速度 | 非常に高速 | やや遅い |
| 署名サイズ | 32バイト | 256バイト |
| 鍵の配布 | 秘密鍵の共有が必要 | 公開鍵のみ配布 |
| 適した構成 | 単一サービス | マイクロサービス |

**選び方**: 単一バックエンドならHS256、複数サービスで検証するならRS256を選びましょう。さらに小さな署名サイズが必要ならES256（ECDSA）も有力です。

---

## 4. クレーム（Claims）

### 4.1 登録済みクレーム

| クレーム | 正式名 | 説明 | 推奨度 |
|---------|--------|------|--------|
| `iss` | Issuer | トークンの発行者 | 推奨 |
| `sub` | Subject | ユーザーの識別子 | 推奨 |
| `aud` | Audience | トークンの対象サービス | 推奨 |
| `exp` | Expiration | 有効期限（Unixタイムスタンプ） | 必須 |
| `nbf` | Not Before | 有効開始時刻 | 任意 |
| `iat` | Issued At | 発行時刻 | 推奨 |
| `jti` | JWT ID | トークンの一意な識別子 | 任意 |

### 4.2 カスタムクレーム

**含めてよいもの**: ロール（`role`）、権限リスト（`permissions`）、テナントID（`org_id`）

**含めてはいけないもの**: パスワード、クレジットカード情報、住所・電話番号などの個人機密情報

:::message alert
JWTのペイロードはBase64URLデコードするだけで誰でも読めます。機密情報は絶対に含めないでください。
:::

### 4.3 サイズの目安

- JWT全体で **4KB以下** を推奨します
- Authorizationヘッダーの一般的な上限は8KBです
- 詳細情報は別途 `/userinfo` エンドポイントで取得しましょう

---

## 5. 有効期限の管理

### 5.1 推奨される有効期限

| トークン種別 | 推奨有効期限 | 理由 |
|------------|-------------|------|
| アクセストークン | 15分〜1時間 | 漏洩時の被害を限定する |
| リフレッシュトークン | 7日〜30日 | UXとセキュリティのバランス |
| IDトークン | 5分〜15分 | 認証直後のみ使用する |

短いアクセストークン + リフレッシュトークンで新しいアクセストークンを取得するパターンが一般的です。

### 5.2 クロックスキューへの対応

サーバー間の時計のズレで検証が失敗することがあります。`clockTolerance` で許容範囲を設定しましょう。

```typescript
const { payload } = await jwtVerify(token, publicKey, {
  algorithms: ['RS256'],
  clockTolerance: 30, // 30秒の時計ズレを許容
});
```

---

## 6. JWTの検証フロー

JWTを受け取ったサービスが実行すべき検証手順です。

```
Step 1: 形式の検証 — ドットで区切られた3パートか？
Step 2: ヘッダーの検証 — algが許可されたアルゴリズムか？
Step 3: 署名の検証 — 公開鍵/秘密鍵で署名が正しいか？
Step 4: expの検証 — 有効期限が切れていないか？
Step 5: issの検証 — 発行者が信頼できるサーバーか？
Step 6: audの検証 — 自分宛てのトークンか？
Step 7: ビジネスロジック検証 — ロールや権限は十分か？
```

### 6.1 安全な検証の実装例

```typescript
import { jwtVerify, errors } from 'jose';

async function verifyAccessToken(token: string) {
  try {
    const { payload } = await jwtVerify(token, publicKey, {
      algorithms: ['RS256'],       // 許可アルゴリズムを明示
      issuer: 'https://auth.example.com',
      audience: 'https://api.example.com',
      clockTolerance: 30,
    });

    if (!payload.sub) throw new Error('subクレームがありません');
    return { userId: payload.sub, role: payload.role as string };
  } catch (error) {
    if (error instanceof errors.JWTExpired) {
      throw new Error('トークンの有効期限が切れています');
    }
    if (error instanceof errors.JWSSignatureVerificationFailed) {
      throw new Error('署名の検証に失敗しました');
    }
    throw new Error('無効なトークンです');
  }
}
```

### 6.2 注意すべき攻撃

**alg: "none" 攻撃**: ヘッダーの `alg` を `"none"` に書き換え署名検証をバイパスする攻撃です。`algorithms` パラメータで許可するアルゴリズムを明示すれば防げます。

**アルゴリズム混乱攻撃**: RS256の公開鍵をHS256の秘密鍵として使いトークンを偽造する攻撃です。検証時にアルゴリズムを固定しましょう。

```typescript
// NG: アルゴリズム未指定
const { payload } = await jwtVerify(token, publicKey);

// OK: アルゴリズムを明示
const { payload } = await jwtVerify(token, publicKey, {
  algorithms: ['RS256'],
});
```

---

## 7. セッション vs JWT 比較

| 観点 | セッション方式 | JWT方式 |
|------|--------------|---------|
| 状態管理 | サーバー側（ステートフル） | トークン内（ステートレス） |
| スケーラビリティ | セッションストアの共有が必要 | サーバー間の状態共有が不要 |
| 即時失効 | セッション削除で即座に失効 | 有効期限まで失効が困難 |
| トークンサイズ | セッションIDのみ（小さい） | クレームを含む（大きい） |
| CSRF対策 | 必要（Cookieベース） | Authorizationヘッダーなら不要 |
| モバイル対応 | Cookie管理が煩雑 | ヘッダー送信で容易 |
| マイクロサービス | セッション共有が課題 | 公開鍵配布で対応可能 |

**セッション方式が向くケース**: 従来型Webアプリ、即時ログアウトが必要なシステム

**JWT方式が向くケース**: SPA + API構成、マイクロサービス、モバイルアプリ

:::message
JWTは万能ではありません。「即時失効が必要」「セッション状態を管理したい」というケースでは、サーバーサイドセッションのほうが適しています。
:::

---

## まとめ

| 項目 | ポイント |
|------|---------|
| JWTの構造 | Header・Payload・Signatureの3部構成 |
| HS256 | 対称鍵方式。高速だが鍵共有が必要。単一サービス向け |
| RS256 | 非対称鍵方式。公開鍵で検証。マイクロサービス向け |
| クレーム設計 | `sub`, `exp`, `iss`, `aud` を必ず設定。機密情報は含めない |
| 有効期限 | アクセストークンは15分〜1時間。リフレッシュトークンと組み合わせる |
| 検証フロー | アルゴリズム・署名・有効期限・発行者・対象者を順に検証する |
| セッションとの違い | JWTはステートレスでスケーラブル。即時失効にはセッションが有利 |

---

## やってみよう！

**課題1: JWTの手動デコード（基礎）**

以下のJWTをBase64URLデコードして、ヘッダーとペイロードの中身を確認してください。

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyXzQ1NiIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTcwMDAwMDAwMCwiZXhwIjoxNzAwMDAwOTAwfQ.SIGNATURE
```

```typescript
const parts = jwt.split('.');
const header = JSON.parse(Buffer.from(parts[0], 'base64url').toString());
const payload = JSON.parse(Buffer.from(parts[1], 'base64url').toString());
console.log('Header:', header);
console.log('Payload:', payload);
console.log('有効期限:', new Date(payload.exp * 1000).toISOString());
```

**課題2: JWT検証ミドルウェアの実装（応用）**

Express + `jose` ライブラリで、以下を満たす認証ミドルウェアを作成してください。

- RS256アルゴリズムで検証する
- `algorithms`, `issuer`, `audience` を必ず指定する
- 有効期限切れ・署名エラー・形式エラーで異なるエラーメッセージを返す

**課題3: 設計判断（発展）**

次の3つのシステムについて、セッション方式とJWT方式のどちらが適切か理由とともに考えてください。

1. 社内管理画面（利用者50名、即時アカウント停止が必要）
2. モバイルアプリ向けAPI（ユーザー100万人、マイクロサービス構成）
3. ECサイト（決済機能あり、PCI DSS準拠が必要）
