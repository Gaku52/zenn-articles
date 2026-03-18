---
title: "多要素認証（MFA）"
---

パスワードだけでアカウントを守る時代は終わりました。どれほど強固なパスワードポリシーを設定しても、フィッシングやクレデンシャルスタッフィングによって突破されるリスクは避けられません。この章では、多要素認証（MFA）の各方式を理解し、TOTP の実装からリカバリーコード設計、そして MFA の UX まで実践的に学んでいきます。

## 認証の 3 要素

MFA を正しく理解するには、まず「認証要素」の分類を押さえる必要があります。

| カテゴリ | 英語名 | 具体例 |
|---------|--------|--------|
| 知識要素 | Something You Know | パスワード、PIN、秘密の質問 |
| 所有要素 | Something You Have | スマートフォン、ハードウェアキー、ICカード |
| 生体要素 | Something You Are | 指紋、顔認証、虹彩、声紋 |

MFA の原則は **異なるカテゴリの要素を組み合わせる** ことです。同一カテゴリの複数要素を使っても MFA にはなりません。

```
# これは MFA ではない（両方とも知識要素）
パスワード + 秘密の質問  => 単一要素

# これが MFA（知識 + 所有）
パスワード + TOTP        => 多要素認証

# Passkey は 1 操作で多要素を実現（所有 + 生体）
Passkey（デバイス所有 + 指紋/顔認証） => 多要素認証
```

## MFA 方式の比較

主要な MFA 方式を、セキュリティ・UX・コストの観点から比較します。

| 方式 | セキュリティ | UX | フィッシング耐性 | コスト | オフライン |
|------|------------|-----|----------------|--------|-----------|
| SMS OTP | 低い | 良い | なし | 通信費 | 不可 |
| TOTP | 中程度 | 良い | なし | 無料 | 可能 |
| Push 通知 | 中程度 | 最良 | 限定的 | 有料 | 不可 |
| WebAuthn | 最強 | 最良 | あり | キー代 | 可能 |
| Passkeys | 最強 | 最良 | あり | 無料 | 可能 |

推奨の優先順位は以下のとおりです。

1. **Passkeys / WebAuthn** -- 最も安全で UX も最良
2. **TOTP** -- 広く普及しており無料で導入可能
3. **Push 通知** -- UX に優れるがサービス依存
4. **SMS OTP** -- 最後の手段として提供

### SMS OTP のリスク

SMS OTP は最も馴染みのある方式ですが、以下のリスクがあります。

- **SIM スワップ攻撃**: 攻撃者が携帯ショップで SIM を再発行し、SMS を横取りする
- **SS7 プロトコルの脆弱性**: 1975 年設計の電話網制御プロトコルに認証の仕組みがなく、SMS の傍受が可能
- **リアルタイムフィッシング**: 偽サイトに入力させたコードを即座に本物のサイトで使う

NIST SP 800-63B では SMS を「制限付きの認証器」として位置づけており、可能であれば TOTP や WebAuthn への移行が推奨されています。

## TOTP の仕組み

TOTP（Time-based One-Time Password）は RFC 6238 で定義されたアルゴリズムです。サーバーとクライアントが共有する秘密鍵と現在時刻から、30 秒ごとに変わる 6 桁のコードを生成します。

```
TOTP 生成の流れ:

1. タイムステップを計算
   T = floor(現在のUNIX秒 / 30)

2. HMAC-SHA1 を計算
   hmac = HMAC-SHA1(秘密鍵, T)

3. Dynamic Truncation で 6 桁に変換
   offset = hmac の最終バイトの下位 4 ビット
   4 バイトを切り出して 31 ビット整数に変換
   otp = 整数 % 1000000
```

### speakeasy を使った TOTP 実装

実際のアプリケーションでは、`speakeasy` ライブラリを使って TOTP を実装します。

```bash
npm install speakeasy qrcode
```

**セットアップ（秘密鍵の生成と QR コード表示）**

```typescript
import speakeasy from "speakeasy";
import QRCode from "qrcode";

async function setupTOTP(userId: string, email: string) {
  // 秘密鍵を生成
  const secret = speakeasy.generateSecret({
    name: `MyApp (${email})`,
    issuer: "MyApp",
    length: 20,
  });

  // 秘密鍵を暗号化して DB に仮保存
  await db.mfaSetup.create({
    data: {
      userId,
      secret: encrypt(secret.base32), // AES-256-GCM で暗号化
      verified: false,
    },
  });

  // QR コードを生成
  const qrCodeUrl = await QRCode.toDataURL(secret.otpauth_url!);

  return {
    qrCodeUrl,       // フロントエンドで <img> に表示
    manualEntry: secret.base32, // 手動入力用
  };
}
```

**セットアップの検証（ユーザーが入力したコードを確認）**

```typescript
async function verifyTOTPSetup(userId: string, token: string) {
  const setup = await db.mfaSetup.findUnique({ where: { userId } });
  if (!setup) throw new Error("MFA setup not found");

  const secret = decrypt(setup.secret);

  // コードを検証（前後 1 ステップのウィンドウ）
  const isValid = speakeasy.totp.verify({
    secret,
    encoding: "base32",
    token,
    window: 1,
  });

  if (!isValid) {
    throw new Error("無効な認証コードです");
  }

  // MFA を有効化
  await db.$transaction([
    db.user.update({
      where: { id: userId },
      data: { mfaEnabled: true },
    }),
    db.mfaSetup.update({
      where: { userId },
      data: { verified: true },
    }),
  ]);

  // リカバリーコードを生成して返す
  const recoveryCodes = generateRecoveryCodes();
  await saveRecoveryCodes(userId, recoveryCodes);

  return { recoveryCodes };
}
```

**認証時の検証**

```typescript
async function verifyTOTPLogin(userId: string, token: string): Promise<boolean> {
  const setup = await db.mfaSetup.findUnique({
    where: { userId, verified: true },
  });
  if (!setup) return false;

  const secret = decrypt(setup.secret);

  // リプレイ攻撃防止: 使用済みコードを拒否
  const codeKey = `totp:used:${userId}:${token}`;
  const isUsed = await redis.exists(codeKey);
  if (isUsed) return false;

  const isValid = speakeasy.totp.verify({
    secret,
    encoding: "base32",
    token,
    window: 1,
  });

  if (isValid) {
    // 90 秒間（period x 3）使用済みとして記録
    await redis.setex(codeKey, 90, "1");
  }

  return isValid;
}
```

### TOTP のセキュリティ対策チェックリスト

TOTP を実装する際は、以下の対策を必ず行ってください。

- **秘密鍵の暗号化保存**: DB に平文で保存しない。AES-256-GCM 等で暗号化し、鍵は KMS で管理する
- **リプレイ攻撃防止**: 同じ OTP の再使用を Redis で追跡して拒否する
- **ブルートフォース対策**: 5 回失敗でアカウントを 15〜30 分ロックアウトする
- **タイムドリフト対応**: NTP で時刻を同期し、検証ウィンドウ（前後 1〜2 ステップ）を設定する
- **タイミングセーフ比較**: 文字列比較にタイミング攻撃の余地を残さない

## WebAuthn と Passkeys

WebAuthn は W3C と FIDO Alliance が策定した、公開鍵暗号ベースの認証規格です。秘密鍵はデバイスに保存され、サーバーには公開鍵のみが保存されます。

### なぜフィッシングに強いのか

WebAuthn の認証データには `rpIdHash`（Relying Party のドメインハッシュ）が含まれます。`example.com` で登録した鍵は、偽サイト `evil.com` からの認証要求では rpId が不一致となり、認証器が自動的に拒否します。ユーザーが偽サイトにアクセスしても、鍵が使われることは原理的にありません。

### Passkeys とは

Passkeys は WebAuthn の進化版です。従来の WebAuthn ではデバイスに紐づいた鍵が同期されませんでしたが、Passkeys は iCloud Keychain や Google Password Manager を通じてクラウド同期されます。

| 項目 | 従来の WebAuthn | Passkeys |
|------|---------------|----------|
| 鍵の同期 | デバイスに紐付き | クラウド同期 |
| デバイス間共有 | 不可 | 可能 |
| バックアップ | なし | 自動 |
| デバイス紛失時 | アクセス不可 | 他のデバイスで可能 |

### 実装に使うライブラリ

WebAuthn の実装には `@simplewebauthn/server`（サーバー側）と `@simplewebauthn/browser`（クライアント側）を使います。基本的な流れは以下のとおりです。

1. **登録**: サーバーが `generateRegistrationOptions` でチャレンジを生成 → クライアントが `startRegistration` で認証器にアクセス → サーバーが `verifyRegistrationResponse` で公開鍵を保存
2. **認証**: サーバーが `generateAuthenticationOptions` でチャレンジを生成 → クライアントが `startAuthentication` で署名 → サーバーが `verifyAuthenticationResponse` で署名を検証

重要なポイントは、`rpID`（Relying Party ID）をドメインと一致させることです。これがフィッシング耐性の要となります。また、`residentKey: "preferred"` を設定することで Passkey をサポートできます。

## リカバリーコード

MFA デバイスを紛失した場合の最後の砦がリカバリーコードです。セットアップ完了時に 10 個のワンタイムコードを生成し、ユーザーに安全な場所への保管を案内します。

### 設計のポイント

- **生成数**: 8〜10 個（各コード 1 回使い切り）
- **エントロピー**: 十分な長さの暗号論的乱数を使用する
- **保存方法**: ハッシュ化して DB に保存する（平文保存は厳禁）
- **残数通知**: 残り 2 個以下でメール通知を送る

```typescript
import crypto from "crypto";

function generateRecoveryCodes(count: number = 10): string[] {
  return Array.from({ length: count }, () => {
    const bytes = crypto.randomBytes(6);
    const hex = bytes.toString("hex");
    // "a3f8-e2b1-c7d9" のような読みやすい形式
    return `${hex.slice(0, 4)}-${hex.slice(4, 8)}-${hex.slice(8, 12)}`;
  });
}

async function saveCodes(userId: string, codes: string[]): Promise<void> {
  // 既存コードを削除
  await db.recoveryCode.deleteMany({ where: { userId } });

  // ハッシュ化して保存
  const hashedCodes = codes.map((code) => ({
    userId,
    code: crypto.createHash("sha256").update(code).digest("hex"),
    used: false,
  }));

  await db.recoveryCode.createMany({ data: hashedCodes });
}

async function verifyRecoveryCode(
  userId: string,
  code: string
): Promise<boolean> {
  // レート制限: 1 時間に 10 回まで
  const rateLimitKey = `recovery:ratelimit:${userId}`;
  const attempts = await redis.incr(rateLimitKey);
  if (attempts === 1) await redis.expire(rateLimitKey, 3600);
  if (attempts > 10) {
    throw new Error("試行回数の上限に達しました。1 時間後に再試行してください。");
  }

  const hashedCode = crypto.createHash("sha256").update(code).digest("hex");

  const record = await db.recoveryCode.findFirst({
    where: { userId, code: hashedCode, used: false },
  });
  if (!record) return false;

  // 使用済みにマーク
  await db.recoveryCode.update({
    where: { id: record.id },
    data: { used: true, usedAt: new Date() },
  });

  // 残りのコード数を確認
  const remaining = await db.recoveryCode.count({
    where: { userId, used: false },
  });
  if (remaining <= 2) {
    await notifyLowRecoveryCodes(userId, remaining);
  }

  return true;
}
```

## MFA の UX 設計

MFA はセキュリティ施策ですが、UX を軽視するとユーザーの離脱につながります。以下のポイントを押さえて設計しましょう。

### セットアップフロー

MFA のセットアップは以下の 5 ステップで構成します。

```
Step 1: MFA 方式の選択
  - Passkey（推奨マーク付き）/ TOTP / SMS を提示

Step 2: セットアップ
  - Passkey: 指紋 or 顔認証で鍵を生成
  - TOTP: QR コードスキャン + 手動入力のフォールバック
  - SMS: 電話番号入力

Step 3: 検証
  - 実際にコードを入力して動作確認

Step 4: リカバリーコードの保存
  - 10 個のコードを表示
  - ダウンロード / コピーボタンを提供
  - 「保存しました」チェックボックスで確認
  - コードの一部を再入力させて保存を確認

Step 5: 完了
  - 信頼できるデバイスの登録を提案
```

### 認証時の UX

ログイン時に MFA を求める場合のベストプラクティスです。

- **信頼済みデバイス機能**: 「このデバイスを 30 日間信頼する」オプションを提供し、毎回の MFA 入力を避ける
- **複数方式の選択肢**: TOTP が使えない場合に SMS やリカバリーコードに切り替えられるようにする
- **復旧手段の確保**: MFA 強制によるアカウントロックは絶対に避ける。必ずリカバリーコードやサポート経由の本人確認フローを用意する

### 信頼済みデバイスの実装

信頼済みデバイスは以下の方針で実装します。

- ランダムな `deviceId` を生成し、ハッシュ化して DB に保存する
- 元の `deviceId` は `httpOnly` / `secure` / `sameSite: lax` の Cookie でクライアントに返す
- 有効期間は 30 日間とし、DB 側にも `expiresAt` を記録する
- ログイン時に Cookie の `deviceId` をハッシュ化して DB と照合し、一致すれば MFA をスキップする

### MFA 導入時のよくある失敗

| やりがちなこと | 問題 | 対策 |
|-------------|------|------|
| TOTP シークレットを平文で DB に保存 | DB 漏洩で全ユーザーの MFA が無効化 | AES-256-GCM + KMS で暗号化 |
| SMS OTP のみに依存 | SIM スワップで完全にバイパス可能 | TOTP / WebAuthn を主方式に |
| リカバリー手段なしで MFA を強制 | デバイス紛失でアカウント喪失 | リカバリーコード + サポート復旧 |
| 同じ OTP コードの再使用を許可 | リプレイ攻撃に脆弱 | 使用済みコードを Redis で追跡 |
| MFA の有無を認証前に漏らす | ユーザー列挙攻撃の手がかり | MFA の有無で応答を変えない |

## まとめ

この章で学んだ内容を整理します。

| 項目 | ポイント |
|------|---------|
| 認証の 3 要素 | 知識・所有・生体の異なるカテゴリを組み合わせる |
| TOTP | RFC 6238 ベース、speakeasy で実装、秘密鍵は暗号化保存 |
| SMS OTP | SIM スワップ等のリスクがあり、最後の手段として位置づける |
| WebAuthn | 公開鍵暗号ベース、フィッシング完全耐性 |
| Passkeys | WebAuthn + クラウド同期、UX とセキュリティを両立 |
| リカバリーコード | 10 個のワンタイムコード、ハッシュ化保存、残数通知 |
| MFA UX | 段階的導入、信頼済みデバイス、復旧手段の確保 |
| 推奨構成 | Passkey を主、TOTP を副、SMS をフォールバックに |

## やってみよう!

以下の課題に取り組んで、MFA の実装力を確かめましょう。

1. **TOTP セットアップ API を作る**: speakeasy を使って、秘密鍵の生成 → QR コード表示 → コード検証 → MFA 有効化の一連のエンドポイントを Express で実装してください。リプレイ攻撃防止とブルートフォース対策も忘れずに入れましょう。

2. **リカバリーコード管理を実装する**: MFA 有効化時にリカバリーコードを 10 個生成し、ハッシュ化して DB に保存する仕組みを作ってください。検証時にはレート制限と残数通知も実装しましょう。

3. **信頼済みデバイス機能を追加する**: ログイン時に「このデバイスを信頼する」チェックボックスを設け、30 日間は MFA をスキップできる仕組みを Cookie ベースで実装してください。デバイス ID はハッシュ化して保存しましょう。
