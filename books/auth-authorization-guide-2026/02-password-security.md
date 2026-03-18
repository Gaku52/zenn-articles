---
title: "パスワードセキュリティ"
---

## はじめに

パスワードは最も広く使われている認証手段ですが、同時に最も攻撃されやすい認証手段でもあります。この章では、パスワードを安全に扱うための技術と考え方を体系的に学びます。

具体的には、以下のトピックを扱います。

- パスワードハッシュの仕組みと推奨アルゴリズム（bcrypt / Argon2）
- ソルトとストレッチングによる防御強化
- モダンなパスワードポリシーの設計
- Have I Been Pwned を活用した漏洩チェック

## なぜパスワードを「ハッシュ化」するのか

パスワードをデータベースにそのまま（平文で）保存するのは、最も危険な実装です。DB が漏洩した瞬間、全ユーザーのパスワードが攻撃者に知られてしまいます。

そこで使われるのが **ハッシュ関数** です。ハッシュ関数は入力を固定長の値に変換する「一方向関数」で、ハッシュ値から元のパスワードを復元することはできません。

```
平文保存のリスク:
  DB漏洩 → 全ユーザーのパスワードが即座に判明
  内部犯行 → 開発者がパスワードを閲覧可能
  ログ混入 → ログファイルにパスワードが記録される

ハッシュ化すると:
  "password123" → bcrypt → "$2b$12$LJ3m4ys..."
  ハッシュ値から元のパスワードを復元することは不可能
```

### ハッシュと暗号化の違い

よくある誤解として「パスワードを暗号化して保存する」というものがあります。しかし、暗号化は鍵があれば復号できてしまうため、パスワード保存には不適切です。

| 特徴 | ハッシュ（一方向） | 暗号化（双方向） |
|------|-----------------|----------------|
| 復元 | 不可能 | 鍵があれば可能 |
| 用途 | パスワード保存 | データの保護 |
| 例 | bcrypt, Argon2 | AES, ChaCha20 |

パスワードの保存には、必ずハッシュ関数を使用してください。

## ソルト -- レインボーテーブル攻撃への対策

### ソルトとは

ソルトとは、パスワードをハッシュ化する際に付加するランダムな値です。ソルトを加えることで、同じパスワードでもユーザーごとに異なるハッシュ値が生成されます。

```
ソルトなしの場合:
  "password" → SHA-256 → "5e884..."（全ユーザーで同じハッシュ）
  → レインボーテーブル（事前計算済みの逆引き表）で一括解読可能

ソルト付きの場合:
  "password" + "a3f8e2..." → "8b2c1..."（ユーザーAのハッシュ）
  "password" + "7d4b9c..." → "f1e3a..."（ユーザーBのハッシュ）
  → 同じパスワードでも異なるハッシュ値になる
```

### ソルトの要件

安全なソルトには、以下の条件が求められます。

1. **暗号的に安全な乱数生成器** で生成すること（`crypto.randomBytes(16)` 以上）
2. **ユーザーごとに一意** であること
3. **16バイト（128ビット）以上** の長さがあること
4. ハッシュ値と共に保存すること

bcrypt や Argon2 はソルトを自動的に生成し、ハッシュ値に埋め込みます。手動でソルトを管理する必要はありません。

## ストレッチング -- 「遅さ」が武器になる

### なぜ高速なハッシュ関数ではダメなのか

MD5 や SHA-256 は非常に高速に計算できるため、攻撃者は GPU を使って毎秒数十億回のハッシュ計算を行えます。つまり、短いパスワードは数秒で解読されてしまいます。

```
攻撃速度の比較（GPUクラスター想定）:

  アルゴリズム       試行速度/秒      8文字ランダム解読
  MD5              ~300億          数秒
  SHA-256          ~30億           数分
  bcrypt (cost=12) ~10万           数十年
  Argon2id (64MB)  ~1,000          数億年
```

ストレッチングとは、ハッシュ計算を意図的に遅くする技法です。bcrypt や Argon2 は、内部でハッシュ計算を何千回も繰り返すことで、1回あたりの計算時間を数百ミリ秒に引き上げています。

正規ユーザーにとっての数百ミリ秒は問題になりませんが、攻撃者にとっては試行回数が激減するため、ブルートフォース攻撃が事実上不可能になります。

## 推奨アルゴリズム: bcrypt と Argon2

### アルゴリズム比較表

| アルゴリズム | 推奨度 | 特徴 | GPU耐性 |
|------------|-------|------|---------|
| **Argon2id** | 最良 | メモリハード、GPU耐性最強 | 最高 |
| **bcrypt** | 良好 | 実績豊富、広くサポート | 高い |
| scrypt | 良好 | メモリハード | 高い |
| PBKDF2 | 条件付き | FIPS準拠が必要な場合のみ | 低い |
| SHA-256 | 不可 | 高速すぎる | なし |
| MD5 | 不可 | 高速＋衝突脆弱性 | なし |

**新規プロジェクトでは Argon2id、既存プロジェクトでは bcrypt を推奨します。**

### bcrypt の仕組み

bcrypt は Blowfish 暗号をベースにしたパスワード専用のハッシュ関数です。コスト係数（cost factor）によって計算回数を制御できます。

```
bcrypt ハッシュ値の構造:
  $2b$12$LJ3m4ys3Gk8v0f2xKb2I4OXYiDkG0...
   |  |  |  |
   |  |  |  └── ハッシュ値 + ソルト（自動埋め込み）
   |  |  └───── コスト係数（2^12 = 4,096回のイテレーション）
   |  └──────── バージョン（2b が最新）
   └─────────── アルゴリズム識別子
```

TypeScript での実装例は以下のとおりです。

```typescript
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 12; // コスト係数

// パスワードのハッシュ化
async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS);
}

// パスワードの検証
async function verifyPassword(
  password: string,
  hash: string,
): Promise<boolean> {
  return bcrypt.compare(password, hash);
}
```

bcrypt には **パスワード長の上限が72バイト** という制限があります。日本語（UTF-8）では1文字3バイトのため、約24文字が上限です。長いパスワードを扱う場合は、SHA-256 でプレハッシュする方法が有効です。

```typescript
import crypto from 'crypto';

async function hashLongPassword(password: string): Promise<string> {
  // SHA-256 でプレハッシュ（Base64で44文字、72バイト以内）
  const preHash = crypto
    .createHash('sha256')
    .update(password)
    .digest('base64');
  return bcrypt.hash(preHash, SALT_ROUNDS);
}
```

### Argon2id の仕組み

Argon2 は 2015年の Password Hashing Competition で優勝したアルゴリズムです。3つのバリアントがありますが、**Argon2id**（Argon2i と Argon2d のハイブリッド）が推奨されています。

Argon2id の最大の特徴は **メモリハード** であることです。ハッシュ計算時に大量のメモリを消費するため、GPU や ASIC による並列攻撃が非常に困難になります。

```
Argon2id のパラメータ:
  memoryCost (m): 使用メモリ量（KB）
  timeCost (t):   イテレーション回数
  parallelism (p): 並列レーン数

OWASP 推奨パラメータ:
  標準:   m=19456 (19MB), t=2, p=1
  高安全: m=65536 (64MB), t=3, p=4
```

```typescript
import argon2 from 'argon2';

// Argon2id でハッシュ化
async function hashPassword(password: string): Promise<string> {
  return argon2.hash(password, {
    type: argon2.argon2id,
    memoryCost: 65536,  // 64MB
    timeCost: 3,
    parallelism: 4,
    saltLength: 16,
    hashLength: 32,
  });
}

// パスワードの検証
async function verifyPassword(
  password: string,
  hash: string,
): Promise<boolean> {
  return argon2.verify(hash, password);
}

// ハッシュ結果例:
// "$argon2id$v=19$m=65536,t=3,p=4$c29tZXNhbHQ$..."
```

### コスト係数のチューニング

コスト係数は「ハッシュ計算に 250ms〜1秒かかる」程度に調整するのが一般的です。サーバーの性能に合わせて計測してから決めることが推奨されます。

```typescript
// ベンチマーク関数
async function benchmarkHash() {
  const password = 'benchmark-test-password';

  for (const cost of [10, 11, 12, 13, 14]) {
    const start = performance.now();
    await bcrypt.hash(password, cost);
    const ms = (performance.now() - start).toFixed(0);
    console.log(`bcrypt cost=${cost}: ${ms}ms`);
  }
}
```

## モダンなパスワードポリシー

### NIST SP 800-63B の推奨事項

2020年に改訂された NIST のガイドラインでは、従来の「複雑性ルール」が廃止され、よりユーザーフレンドリーかつ安全なポリシーが推奨されています。

**推奨されるポリシー:**

- 最小8文字（できれば15文字以上を推奨）
- 最大64文字以上を許容する
- Unicode の全文字を許容する（日本語もOK）
- 漏洩パスワードリストとの照合を行う
- パスワード強度メーターを表示する
- ペースト操作を許可する（パスワードマネージャー対応）

**廃止された古い慣習:**

- 大文字・小文字・数字・記号の組み合わせ強制
- 定期的なパスワード変更の強制
- 秘密の質問
- パスワードヒント

なぜ複雑性ルールが廃止されたのでしょうか。「大文字・数字・記号を含めること」というルールを課すと、多くのユーザーが `P@ssw0rd!` のような予測しやすいパターンに頼ってしまいます。定期変更を強制すると `password1`, `password2`... のようなインクリメントが発生します。結果的にセキュリティが下がるため、NIST はこれらのルールを非推奨としました。

### パスワード強度メーター

ユーザーに安全なパスワードを選んでもらうには、リアルタイムのフィードバックが効果的です。Dropbox が開発した **zxcvbn** ライブラリを使うと、辞書攻撃やパターン解析を考慮した現実的な強度評価が可能になります。

```typescript
import zxcvbn from 'zxcvbn';

function checkStrength(password: string, userInputs: string[] = []) {
  const result = zxcvbn(password, userInputs);
  return {
    score: result.score, // 0(最弱)〜4(最強)
    crackTime:
      result.crack_times_display
        .offline_slow_hashing_1e4_per_second,
    feedback: result.feedback,
  };
}

// "password"          → score: 0, "less than a second"
// "correct horse..."  → score: 4, "centuries"
```

## 漏洩チェック -- Have I Been Pwned

### Have I Been Pwned (HIBP) とは

[Have I Been Pwned](https://haveibeenpwned.com/) は、過去のデータ漏洩事件で流出したパスワードのデータベースを提供するサービスです。2026年3月時点で、数十億件のパスワードが登録されています。

ユーザーが設定しようとしているパスワードが過去に漏洩していないかを確認することで、クレデンシャルスタッフィング攻撃（他サービスで漏洩した認証情報の流用）を防ぐことができます。

### k-Anonymity モデルによる安全な照合

「パスワードを外部サービスに送信するのは危険では？」と思うかもしれません。HIBP の API は **k-Anonymity** モデルを採用しており、パスワードそのものをサーバーに送る必要がありません。

仕組みは以下のとおりです。

1. パスワードの SHA-1 ハッシュを計算する
2. ハッシュの **先頭5文字だけ** を API に送信する
3. API は先頭5文字が一致する全てのハッシュサフィックスを返す
4. クライアント側で残りの文字列を照合する

```typescript
// Have I Been Pwned API でパスワード漏洩をチェック
async function isBreachedPassword(
  password: string,
): Promise<boolean> {
  const hash = await sha1(password);
  const prefix = hash.substring(0, 5);
  const suffix = hash.substring(5).toUpperCase();

  // 先頭5文字のみ送信 → パスワード情報は漏れない
  const res = await fetch(
    `https://api.pwnedpasswords.com/range/${prefix}`,
    { headers: { 'Add-Padding': 'true' } },
  );
  const text = await res.text();

  return text.split('\n').some((line) => {
    const [hashSuffix] = line.split(':');
    return hashSuffix === suffix;
  });
}
```

### バリデーションへの統合

漏洩チェック、よく使われるパスワードの禁止、ユーザー情報との類似性チェックを統合した例を示します。

```typescript
import { z } from 'zod';

const passwordSchema = z
  .string()
  .min(8, 'パスワードは8文字以上必要です')
  .max(128, 'パスワードは128文字以下にしてください')
  .refine(
    (pw) => !isCommonPassword(pw),
    'よく使われるパスワードのため安全ではありません',
  )
  .refine(
    async (pw) => !(await isBreachedPassword(pw)),
    '過去のデータ漏洩で確認されたパスワードです',
  );
```

## アンチパターン -- やってはいけないこと

最後に、パスワード管理でよくある危険な実装パターンを挙げておきます。

| アンチパターン | なぜ危険か |
|--------------|----------|
| 平文保存 | DB漏洩で全パスワードが即座に判明します |
| AES等による暗号化保存 | 鍵が漏洩すると全パスワードが復号されます |
| MD5 / SHA-256（ソルトなし） | レインボーテーブルで解読可能です |
| 独自ハッシュアルゴリズム | 検証されていない実装には脆弱性があります |
| パスワードの最大長を16文字に制限 | パスフレーズの利用を妨げます |
| エラーメッセージで存在を漏洩 | 「パスワードが違います」はユーザー列挙攻撃の手がかりになります |
| パスワード変更時に旧セッションを残す | 攻撃者のセッションが有効なまま残ります |

エラーメッセージは常に **「メールアドレスまたはパスワードが間違っています」** のように曖昧にしてください。

## まとめ

| 項目 | ベストプラクティス |
|------|-----------------|
| ハッシュアルゴリズム | 新規は Argon2id、既存は bcrypt |
| ソルト | bcrypt / Argon2 が自動生成（手動管理不要） |
| ストレッチング | 250ms〜1秒のハッシュ計算時間に調整 |
| パスワード長 | 最小8文字、最大64文字以上を許容 |
| 複雑性ルール | 廃止（NIST SP 800-63B） |
| 定期変更 | 不要（漏洩時のみ変更） |
| 漏洩チェック | Have I Been Pwned API（k-Anonymity） |
| 強度メーター | zxcvbn 等でリアルタイムフィードバック |
| エラーメッセージ | 「メールまたはパスワードが違います」（曖昧に） |
| セッション管理 | パスワード変更時に全セッション無効化 |

## やってみよう！

以下の課題に取り組んで、この章の内容を実践してみましょう。

### 課題1: ハッシュ速度の体感（初級）

MD5、SHA-256、bcrypt（cost=12）で同じパスワードをそれぞれ1,000回ハッシュ化し、所要時間を比較してください。「遅い」ことがなぜセキュリティ上の利点になるのかを体感できます。

### 課題2: 漏洩チェックの実装（中級）

Have I Been Pwned API の k-Anonymity モデルを使って、入力されたパスワードが過去に漏洩しているか確認するCLIツールを作ってみましょう。Node.js の `crypto` モジュールと `fetch` API を使えば数十行で実装できます。

### 課題3: パスワードバリデーターの構築（上級）

以下の要件を満たすパスワードバリデーション関数を実装してください。

- 最小8文字、最大128文字のチェック
- よく使われるパスワード上位1万件との照合
- HIBP API による漏洩チェック
- zxcvbn によるスコアが 3 以上であることの確認
- ユーザーのメールアドレスとの類似性チェック
