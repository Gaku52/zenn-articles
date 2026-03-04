---
title: "第19章 クラウドセキュリティの基礎"
---

# クラウドセキュリティ基礎

> 責任共有モデルの正しい理解、IAM による最小権限アクセス制御、保存時・転送時の暗号化まで、クラウド環境のセキュリティ基盤を体系的に学ぶ

## この章で学ぶこと

1. **責任共有モデル** — クラウドプロバイダとユーザの責任分担の理解
2. **IAM (Identity and Access Management)** — 最小権限の原則に基づくアクセス制御
3. **データ暗号化** — 保存時 (at rest) と転送時 (in transit) の暗号化戦略
4. **ネットワークセキュリティ** — VPC 設計とセグメンテーションの実践
5. **セキュリティサービス** — 主要クラウドプロバイダのセキュリティサービス比較
6. **ログ・監査・コンプライアンス** — 証跡の確保と継続的な監査体制
7. **インシデント対応** — クラウド環境固有のインシデントレスポンス

---

## 1. 責任共有モデル

### サービスモデル別の責任分担

クラウドセキュリティで最も重要な概念が「責任共有モデル (Shared Responsibility Model)」である。クラウドプロバイダが「クラウド "の" セキュリティ (Security "of" the Cloud)」を担い、ユーザが「クラウド "における" セキュリティ (Security "in" the Cloud)」を担う。

```
+----------------------------------------------------------+
|              責任共有モデル                                 |
|----------------------------------------------------------|
|                  IaaS    PaaS    SaaS                     |
|                  (EC2)   (Lambda) (Office365)             |
|----------------------------------------------------------|
| データ          | User | User  | User                    |
| アプリケーション | User | User  | Provider                |
| ランタイム      | User | Provider| Provider               |
| ミドルウェア    | User | Provider| Provider               |
| OS             | User | Provider| Provider               |
| 仮想化         | Provider| Provider| Provider             |
| ネットワーク   | Provider| Provider| Provider              |
| 物理           | Provider| Provider| Provider              |
|----------------------------------------------------------|
| User = ユーザ責任  |  Provider = クラウド事業者責任         |
+----------------------------------------------------------+
```

### 各サービスモデルにおけるユーザ責任の詳細

#### IaaS (Infrastructure as a Service) の場合

IaaS ではユーザの責任範囲が最も広い。OS パッチ適用、ミドルウェアの設定、ファイアウォールルール、アプリケーションのセキュリティすべてがユーザ責任となる。

```
IaaS ユーザ責任チェックリスト:
+----------------------------------------------------------+
| [ ] OS のパッチ適用を定期的に実施している                    |
| [ ] 不要なポートを閉じ、セキュリティグループを最小化          |
| [ ] ホストベースの IDS/IPS を導入している                   |
| [ ] ログの収集と監視を設定している                           |
| [ ] データの暗号化 (EBS, S3) を有効にしている                |
| [ ] IAM ロールを使い、アクセスキーをハードコードしていない      |
| [ ] AMI / VM イメージのセキュリティスキャンを実施している       |
| [ ] バックアップと災害復旧の計画を策定している                 |
+----------------------------------------------------------+
```

#### PaaS (Platform as a Service) の場合

PaaS ではランタイム以下がプロバイダ責任となるが、アプリケーションコードとデータのセキュリティはユーザ責任である。

```
PaaS ユーザ責任チェックリスト:
+----------------------------------------------------------+
| [ ] アプリケーションコードの脆弱性スキャンを実施している       |
| [ ] 環境変数 / シークレット管理を適切に行っている             |
| [ ] API のアクセス制御 (認証・認可) を実装している            |
| [ ] 関数 / コンテナの実行権限を最小化している                 |
| [ ] デプロイパイプラインのセキュリティを確保している            |
| [ ] 依存ライブラリの脆弱性を定期的にチェックしている           |
+----------------------------------------------------------+
```

#### SaaS (Software as a Service) の場合

SaaS ではユーザ責任はデータとアクセス管理に限定されるが、それでも重大なリスクが存在する。

```
SaaS ユーザ責任チェックリスト:
+----------------------------------------------------------+
| [ ] ユーザアカウントの棚卸しを定期的に実施している            |
| [ ] MFA (多要素認証) を全ユーザに有効化している              |
| [ ] データの分類と共有設定を適切に管理している                 |
| [ ] 退職者のアカウント即時無効化プロセスがある                 |
| [ ] SaaS API のアクセストークン管理を行っている               |
| [ ] データエクスポート / バックアップの手段を確保している       |
+----------------------------------------------------------+
```

### 責任共有でよくある誤解

```
+----------------------------------------------------------+
|  誤解                        | 正しい理解                  |
|----------------------------------------------------------+
|  「クラウドだから安全」        | インフラは安全、設定は自己責任 |
|  「暗号化はクラウドが自動で」  | 有効化はユーザが明示的に行う   |
|  「アクセス制御は不要」        | IAM 設定はユーザの責任        |
|  「バックアップは自動」        | 設定・テストはユーザの責任     |
|  「コンプライアンスも自動」    | 認証取得と維持はユーザの責任   |
|  「ログは永久保存される」      | 保持期間の設定はユーザの責任   |
|  「マルチテナントは危険」      | 論理的隔離は物理隔離と同等    |
|  「プロバイダがDRも担保」      | 可用性設計はユーザの責任       |
+----------------------------------------------------------+
```

### 責任共有モデルの実例: S3 データ漏洩

実際に多発しているインシデントで責任共有モデルを理解する。

```
[インシデント例: S3 バケットの公開設定ミス]

状況:
  企業が顧客データを S3 に保存
  バケットポリシーを誤って Public に設定
  外部から全データにアクセス可能な状態が数ヶ月続いた

責任の所在:
  +----------------------------------------------------------+
  | AWS の責任                    | ユーザの責任               |
  |----------------------------------------------------------+
  | S3 サービスの可用性            | バケットポリシーの設定      |
  | S3 インフラの物理セキュリティ   | パブリックアクセスブロック   |
  | S3 の暗号化機能の提供          | 暗号化の有効化             |
  | S3 のアクセスログ機能の提供     | ログの有効化と監視          |
  +----------------------------------------------------------+

  → AWS は「S3 は正しく動作していた」と判断
  → 設定ミスはユーザの責任
  → AWS は S3 Block Public Access 機能を追加して対策を支援
```

### 主要プロバイダ別の責任共有モデルの違い

```
+----------------------------------------------------------+
| プロバイダ   | モデル名称             | 特徴              |
|----------------------------------------------------------+
| AWS          | Shared Responsibility  | 標準的な2分割      |
|              | Model                  |                   |
| Azure        | Shared Responsibility  | AWS とほぼ同一     |
|              | in the Cloud           |                   |
| GCP          | Shared Fate            | より積極的な       |
|              |                        | 支援を強調        |
+----------------------------------------------------------+

GCP の「Shared Fate」アプローチ:
  - 単なる責任分担ではなく「共に安全を実現する」姿勢
  - Assured Workloads: コンプライアンス準拠を自動支援
  - Security Command Center: 設定ミスの自動検出と修復提案
  - Organization Policy: ガードレールとしての組織ポリシー
```

---

## 2. IAM (Identity and Access Management)

### IAM の構成要素

```
+----------------------------------------------------------+
|                    IAM の構成要素                           |
|----------------------------------------------------------|
|                                                          |
|  [アイデンティティ]                                       |
|  +-- ユーザ (人間のオペレータ)                             |
|  +-- サービスアカウント (アプリケーション)                   |
|  +-- ロール (一時的な権限の集合)                           |
|  +-- グループ (ユーザの集合)                               |
|                                                          |
|  [ポリシー]                                               |
|  +-- アイデンティティベースポリシー (誰に何を許可)           |
|  +-- リソースベースポリシー (何に誰がアクセス可能)           |
|  +-- 権限境界 (最大権限の制限)                             |
|  +-- SCP (組織全体の制限)                                 |
|                                                          |
|  [認証方式]                                               |
|  +-- パスワード + MFA                                     |
|  +-- アクセスキー (プログラムアクセス)                      |
|  +-- 一時的セキュリティ認証情報 (STS)                      |
|  +-- SSO / SAML / OIDC 連携                              |
+----------------------------------------------------------+
```

### IAM ポリシーの評価ロジック

ポリシーがどのように評価されるかを正確に理解することが重要である。

```
IAM ポリシー評価フロー:

リクエスト受信
    |
    v
[1] 明示的 Deny があるか? ──── Yes ──→ アクセス拒否
    |
    No
    |
    v
[2] SCP で許可されているか? ── No ───→ アクセス拒否
    |
    Yes
    |
    v
[3] リソースベースポリシーで
    許可されているか? ────── Yes ──→ アクセス許可
    |
    No (or なし)
    |
    v
[4] アイデンティティベース
    ポリシーで許可? ──────── No ───→ アクセス拒否
    |
    Yes
    |
    v
[5] 権限境界で許可? ──────── No ───→ アクセス拒否
    |
    Yes
    |
    v
[6] セッションポリシーで許可? ─ No ───→ アクセス拒否
    |
    Yes
    |
    v
  アクセス許可

重要ポイント:
  - 明示的 Deny は常に最優先
  - 許可が明示されていない場合はデフォルト拒否 (暗黙的 Deny)
  - 複数ポリシーの AND 条件で評価される
```

### 最小権限ポリシーの設計

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowS3ReadSpecificBucket",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::my-app-data",
                "arn:aws:s3:::my-app-data/*"
            ],
            "Condition": {
                "StringEquals": {
                    "aws:RequestedRegion": "ap-northeast-1"
                },
                "IpAddress": {
                    "aws:SourceIp": "10.0.0.0/8"
                }
            }
        },
        {
            "Sid": "DenyUnencryptedUploads",
            "Effect": "Deny",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::my-app-data/*",
            "Condition": {
                "StringNotEquals": {
                    "s3:x-amz-server-side-encryption": "aws:kms"
                }
            }
        }
    ]
}
```

### IAM ポリシー設計のベストプラクティス

```
+----------------------------------------------------------+
|  IAM ポリシー設計 5 原則                                    |
|----------------------------------------------------------|
|                                                          |
|  1. 最小権限の原則 (Least Privilege)                       |
|     - 必要な権限のみを付与する                              |
|     - ワイルドカード (*) の使用を避ける                      |
|     - Action と Resource の両方を限定する                   |
|                                                          |
|  2. 条件 (Condition) の積極活用                            |
|     - IP アドレス制限                                      |
|     - リージョン制限                                       |
|     - MFA 必須化                                          |
|     - タグベースのアクセス制御 (ABAC)                       |
|                                                          |
|  3. ロールベースアクセス制御 (RBAC)                         |
|     - 個別ユーザではなくグループ/ロールに権限を付与           |
|     - 職務分離 (Separation of Duties) を実装               |
|                                                          |
|  4. 一時的認証情報の優先使用                                |
|     - 長期アクセスキーではなく STS を使用                    |
|     - IAM ロールのアサムプション                            |
|     - セッションの有効期限を適切に設定                       |
|                                                          |
|  5. 定期的な棚卸しと権限削減                                |
|     - IAM Access Analyzer で未使用権限を検出                |
|     - 90 日以上未使用のアクセスキーを無効化                   |
|     - Credential Report で定期監査                         |
+----------------------------------------------------------+
```

### 高度なポリシー例: ABAC (属性ベースアクセス制御)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ABACTagBasedAccess",
            "Effect": "Allow",
            "Action": [
                "ec2:StartInstances",
                "ec2:StopInstances",
                "ec2:RebootInstances"
            ],
            "Resource": "arn:aws:ec2:*:*:instance/*",
            "Condition": {
                "StringEquals": {
                    "ec2:ResourceTag/Department": "${aws:PrincipalTag/Department}",
                    "ec2:ResourceTag/Environment": "${aws:PrincipalTag/Environment}"
                }
            }
        },
        {
            "Sid": "ABACDenyTagModification",
            "Effect": "Deny",
            "Action": [
                "ec2:CreateTags",
                "ec2:DeleteTags"
            ],
            "Resource": "*",
            "Condition": {
                "ForAnyValue:StringEquals": {
                    "aws:TagKeys": ["Department", "Environment"]
                }
            }
        }
    ]
}
```

```
ABAC の利点:
  - タグに基づいて動的に権限を制御
  - 新リソース追加時にポリシー更新が不要
  - スケーラブルな権限管理
  - 例: Department=Engineering のユーザは
        Department=Engineering のインスタンスのみ操作可能
```

### IAM ロールの活用 (EC2/Lambda)

```python
import boto3

# EC2 インスタンスプロファイル経由でアクセス
# (アクセスキーのハードコーディング不要)
s3 = boto3.client('s3')  # IAM ロールの認証情報を自動取得

# Lambda 用ロールの最小権限ポリシー (Terraform)
"""
resource "aws_iam_role" "lambda_role" {
  name = "my-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy" "lambda_policy" {
  role = aws_iam_role.lambda_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["dynamodb:GetItem", "dynamodb:PutItem"]
        Resource = "arn:aws:dynamodb:*:*:table/my-table"
      },
      {
        Effect   = "Allow"
        Action   = ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"]
        Resource = "arn:aws:logs:*:*:*"
      }
    ]
  })
}
"""
```

### クロスアカウントアクセスの設定

クロスアカウントアクセスは IAM ロールの AssumeRole で実現する。

```
設定手順:
1. Account A: 信頼ポリシーで Account B のロールを Principal に指定
   - ExternalId 条件で混乱した代理人問題を防止
2. Account B: sts:AssumeRole で Account A のロールを引き受ける
   - 一時的認証情報 (AccessKeyId, SecretAccessKey, SessionToken) を取得
   - DurationSeconds で有効期限を設定 (最大 12 時間)
```

### マルチアカウント戦略

```
+----------------------------------------------------------+
|  AWS Organizations                                       |
|                                                          |
|  +-- Management Account (請求・組織管理のみ)              |
|  |                                                       |
|  +-- Security OU                                         |
|  |   +-- Security Account (GuardDuty, Security Hub)      |
|  |   +-- Log Archive Account (CloudTrail, Config)        |
|  |                                                       |
|  +-- Workloads OU                                        |
|  |   +-- Production Account                              |
|  |   +-- Staging Account                                 |
|  |   +-- Development Account                             |
|  |                                                       |
|  +-- Sandbox OU                                          |
|      +-- Developer Sandbox Accounts                      |
|                                                          |
|  SCP (Service Control Policy) で全アカウントに制限適用     |
+----------------------------------------------------------+
```

### SCP (Service Control Policy) の実践例

SCP で全アカウントに適用すべき制限:

```
必須 SCP ルール:
  1. リージョン制限: 許可リージョン以外での操作を Deny
  2. 組織離脱の防止: LeaveOrganization を Deny
  3. セキュリティサービスの無効化防止:
     GuardDuty, Config, CloudTrail の停止・削除を Deny
  4. ルートアカウント使用の禁止: root の全操作を Deny
     (管理アカウントの OrganizationAdmin ロールは例外)
```

### AWS IAM Identity Center (旧 SSO) の設定

外部 IdP (Okta / Azure AD 等) と SAML 2.0 / SCIM で連携し、Permission Set でアカウントごとの権限を定義する。一時的認証情報を自動発行するため長期アクセスキーが不要になり、退職時のアクセス無効化も即座に可能。

---

## 3. データ暗号化

### 暗号化の分類

```
+----------------------------------------------------------+
|                暗号化の層                                  |
|----------------------------------------------------------|
|                                                          |
|  転送時の暗号化 (In Transit):                              |
|  +-- TLS 1.2/1.3 (HTTPS, gRPC over TLS)                 |
|  +-- VPN (IPsec, WireGuard)                              |
|  +-- SSH トンネル                                        |
|                                                          |
|  保存時の暗号化 (At Rest):                                |
|  +-- サーバサイド暗号化 (SSE)                              |
|  |   +-- SSE-S3 (S3 管理キー)                            |
|  |   +-- SSE-KMS (KMS 管理キー)                          |
|  |   +-- SSE-C (顧客提供キー)                             |
|  +-- クライアントサイド暗号化 (CSE)                        |
|  +-- ディスク暗号化 (EBS, RDS)                            |
|                                                          |
|  処理時の暗号化 (In Use):                                  |
|  +-- AWS Nitro Enclaves                                  |
|  +-- Confidential Computing                              |
+----------------------------------------------------------+
```

### 暗号化方式の詳細比較

```
+----------------------------------------------------------+
|  暗号化方式比較                                            |
|----------------------------------------------------------|
|  方式         | 鍵管理  | 監査  | コスト | ユースケース    |
|----------------------------------------------------------|
|  SSE-S3       | AWS     | 低    | 無料   | 基本的な暗号化  |
|  SSE-KMS      | ユーザ  | 高    | 有料   | コンプライアンス|
|  SSE-C        | ユーザ  | 最高  | 無料   | 完全な鍵制御    |
|  CSE          | ユーザ  | 最高  | -     | 最高機密データ  |
|----------------------------------------------------------|
|                                                          |
|  SSE-S3:                                                 |
|  - AWS が鍵を管理、最も簡単                                |
|  - 鍵のローテーションは自動                                 |
|  - 監査ログでの鍵使用追跡は不可                             |
|                                                          |
|  SSE-KMS:                                                |
|  - 鍵の作成・ローテーション・削除をユーザが制御              |
|  - CloudTrail で鍵の使用を追跡可能                         |
|  - 鍵ポリシーで細かいアクセス制御                           |
|  - API 呼び出しに対するレートリミットに注意                  |
|                                                          |
|  SSE-C:                                                  |
|  - ユーザが暗号化鍵を提供し、AWS が暗号化を実行             |
|  - AWS は鍵を保存しない (HTTPS 必須)                       |
|  - 鍵を紛失するとデータ復元不可能                           |
|                                                          |
|  CSE:                                                    |
|  - クライアント側で暗号化してから AWS に保存                 |
|  - AWS は暗号化済みデータのみ保持                           |
|  - 最も高いセキュリティだが実装コストも最大                   |
+----------------------------------------------------------+
```

### KMS (Key Management Service) の詳細

```
+----------------------------------------------------------+
|  KMS キー階層                                              |
|                                                          |
|  [Customer Master Key (CMK)]                             |
|       |                                                  |
|       | 暗号化                                            |
|       v                                                  |
|  [Data Encryption Key (DEK)]                             |
|       |                                                  |
|       | 暗号化                                            |
|       v                                                  |
|  [実データ]                                               |
|                                                          |
|  エンベロープ暗号化の仕組み:                                |
|  1. KMS に DEK の生成をリクエスト                          |
|  2. 平文 DEK と暗号化済み DEK が返される                    |
|  3. 平文 DEK でデータを暗号化                              |
|  4. 平文 DEK をメモリから削除                               |
|  5. 暗号化済み DEK をデータと共に保存                       |
|  6. 復号時: 暗号化済み DEK を KMS で復号 → データを復号     |
+----------------------------------------------------------+
```

```python
# KMS を使ったエンベロープ暗号化の実装例
import boto3
import base64
from cryptography.fernet import Fernet

kms = boto3.client('kms', region_name='ap-northeast-1')

# 1. データキーの生成
response = kms.generate_data_key(
    KeyId='alias/my-app-key',
    KeySpec='AES_256'
)

plaintext_key = response['Plaintext']       # 平文の DEK
encrypted_key = response['CiphertextBlob']  # 暗号化済み DEK

# 2. 平文キーでデータを暗号化
fernet_key = base64.urlsafe_b64encode(plaintext_key)
f = Fernet(fernet_key)
encrypted_data = f.encrypt(b"Secret data to protect")

# 3. 平文キーをメモリから削除
del plaintext_key
del fernet_key

# 4. 暗号化済みキーと暗号化データを保存
# (encrypted_key と encrypted_data を保存)

# ========================================
# 復号プロセス
# ========================================

# 5. 暗号化済みキーを KMS で復号
decrypt_response = kms.decrypt(
    CiphertextBlob=encrypted_key,
    KeyId='alias/my-app-key'
)
decrypted_key = decrypt_response['Plaintext']

# 6. 復号したキーでデータを復号
fernet_key = base64.urlsafe_b64encode(decrypted_key)
f = Fernet(fernet_key)
original_data = f.decrypt(encrypted_data)

# 7. キーをメモリから削除
del decrypted_key
del fernet_key
```

### KMS キーポリシーの設計

KMS キーポリシーでは、管理者と利用者の権限を分離する。

```
キーポリシー設計の要点:
  1. root アカウントに kms:* を許可 (IAM ポリシーとの連携用)
  2. KeyAdminRole: 鍵の作成・ローテーション・削除を許可
  3. AppServiceRole: Encrypt, Decrypt, GenerateDataKey* のみ許可
     - Condition で kms:ViaService を指定し、特定サービス経由のみに限定
  → 管理者は鍵を管理するが暗号化できない
  → アプリは暗号化できるが鍵を管理できない (職務分離)
```

### S3 バケットのセキュリティ設定

```
S3 セキュリティ必須設定チェックリスト:
+----------------------------------------------------------+
| 設定項目                        | 内容                     |
|----------------------------------------------------------+
| デフォルト暗号化                | SSE-KMS + BucketKey 有効 |
| パブリックアクセスブロック       | 4 項目すべて True        |
| バケットポリシー                | HTTPS のみ許可 (Deny HTTP)|
| バージョニング                  | 有効化 (誤削除・改ざん対策)|
| オブジェクトロック              | GOVERNANCE or COMPLIANCE |
+----------------------------------------------------------+
```

```json
// バケットポリシー: HTTPS のみ許可の例
{
    "Version": "2012-10-17",
    "Statement": [{
        "Sid": "DenyHTTP",
        "Effect": "Deny",
        "Principal": "*",
        "Action": "s3:*",
        "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"],
        "Condition": {"Bool": {"aws:SecureTransport": "false"}}
    }]
}
```

### RDS / データベースの暗号化

```
RDS セキュリティ設定の要点:
  - storage_encrypted = true (KMS キー指定)
  - publicly_accessible = false (プライベートサブネットのみ)
  - rds.force_ssl = 1 (SSL 接続を強制)
  - deletion_protection = true
  - backup_retention_period = 35 (自動バックアップ)
  - monitoring_interval = 60 (拡張モニタリング)
  - CloudWatch Logs へのログエクスポート
```

### シークレット管理

```
+----------------------------------------------------------+
|  シークレット管理のベストプラクティス                         |
|----------------------------------------------------------|
|                                                          |
|  NG パターン:                                             |
|  - ソースコードにハードコード                               |
|  - 環境変数にプレーンテキストで設定                          |
|  - 設定ファイルに記載して Git にコミット                     |
|  - チャットやメールでパスワードを共有                        |
|                                                          |
|  OK パターン:                                             |
|  - AWS Secrets Manager / Parameter Store                 |
|  - GCP Secret Manager                                    |
|  - Azure Key Vault                                       |
|  - HashiCorp Vault                                       |
|  - 実行時にのみシークレットを取得                           |
|  - 自動ローテーション                                     |
+----------------------------------------------------------+
```

```python
# AWS Secrets Manager からシークレットを取得する例
import boto3
import json

def get_secret(secret_name: str) -> dict:
    """Secrets Manager からシークレットを安全に取得する"""
    client = boto3.client('secretsmanager', region_name='ap-northeast-1')

    try:
        response = client.get_secret_value(SecretId=secret_name)
        secret = json.loads(response['SecretString'])
        return secret
    except client.exceptions.ResourceNotFoundException:
        raise ValueError(f"Secret {secret_name} not found")
    except client.exceptions.DecryptionFailure:
        raise ValueError(f"Cannot decrypt secret {secret_name}")

# 使用例: データベース接続
db_secret = get_secret('prod/myapp/database')
connection_string = (
    f"postgresql://{db_secret['username']}:{db_secret['password']}"
    f"@{db_secret['host']}:{db_secret['port']}/{db_secret['dbname']}"
)

# Terraform でのシークレット自動ローテーション設定
"""
resource "aws_secretsmanager_secret" "db_password" {
  name = "prod/myapp/database"

  # 自動ローテーション
  rotation_rules {
    automatically_after_days = 30
  }
}

resource "aws_secretsmanager_secret_rotation" "db_rotation" {
  secret_id           = aws_secretsmanager_secret.db_password.id
  rotation_lambda_arn = aws_lambda_function.secret_rotation.arn

  rotation_rules {
    automatically_after_days = 30
  }
}
"""
```

---

## 4. ネットワークセキュリティ (クラウド)

### VPC の設計

```
+----------------------------------------------------------+
|  VPC (10.0.0.0/16)                                       |
|                                                          |
|  +-- Public Subnet (10.0.1.0/24)                         |
|  |   +-- NAT Gateway                                    |
|  |   +-- ALB                                            |
|  |   Route: 0.0.0.0/0 → IGW                             |
|  |                                                       |
|  +-- Private Subnet (10.0.2.0/24)                        |
|  |   +-- EC2 / ECS                                      |
|  |   Route: 0.0.0.0/0 → NAT GW                          |
|  |                                                       |
|  +-- Data Subnet (10.0.3.0/24)                           |
|  |   +-- RDS / ElastiCache                               |
|  |   Route: ローカルのみ                                  |
|  |                                                       |
|  +-- VPC Endpoints (S3, DynamoDB, KMS)                   |
|      → インターネットを経由せず AWS サービスにアクセス       |
+----------------------------------------------------------+
```

### マルチ AZ 構成のセキュリティ設計

マルチ AZ では各 AZ に Public/Private/Data の 3 層サブネットを冗長配置する。セキュリティ設計ポイント: 各層で同じセキュリティグループ構成を適用、データサブネットは外部接続なし (ルートテーブルなし)、VPC Flow Logs を全サブネットで有効化。

### セキュリティグループとネットワーク ACL の使い分け

```
+----------------------------------------------------------+
|  セキュリティグループ vs ネットワーク ACL                    |
|----------------------------------------------------------|
|  項目            | セキュリティグループ | ネットワーク ACL  |
|----------------------------------------------------------|
|  レベル          | インスタンス          | サブネット       |
|  ステート        | ステートフル          | ステートレス     |
|  ルール          | 許可のみ             | 許可 + 拒否      |
|  評価            | 全ルール評価          | 番号順評価       |
|  デフォルト      | 全インバウンド拒否    | 全通信許可       |
|  用途            | アプリ単位の制御      | サブネット防御    |
+----------------------------------------------------------+

推奨構成:
  - セキュリティグループ: 主要なアクセス制御 (メインで使用)
  - ネットワーク ACL: 追加の防御層 (特定 IP の明示的拒否)
```

```
セキュリティグループのチェーン設計例:
  ALB SG:  ingress 443/tcp from 0.0.0.0/0 (HTTPS のみ)
  App SG:  ingress 8080/tcp from ALB SG (ALB からのみ)
  DB SG:   ingress 5432/tcp from App SG (アプリからのみ)
  → SG 参照で通信を層ごとに制限 (CIDR ではなく SG ID で指定)
```

### VPC Endpoints の活用

```
+----------------------------------------------------------+
|  VPC Endpoints の種類と使い分け                             |
|----------------------------------------------------------|
|                                                          |
|  Gateway Endpoint (無料):                                 |
|  +-- S3                                                  |
|  +-- DynamoDB                                            |
|  → ルートテーブルにエントリを追加                            |
|                                                          |
|  Interface Endpoint (有料):                               |
|  +-- KMS, Secrets Manager, STS, CloudWatch Logs          |
|  +-- ECR, ECS, Lambda, SNS, SQS                         |
|  → ENI が作成され、プライベート IP で通信                    |
|  → PrivateLink とも呼ばれる                               |
|                                                          |
|  利点:                                                   |
|  - データがインターネットを経由しない                        |
|  - NAT Gateway の帯域・コスト削減                          |
|  - VPC エンドポイントポリシーで追加の制御                    |
+----------------------------------------------------------+
```

VPC エンドポイントポリシーで、アクセス可能なリソースを特定バケットのみに制限することも可能。

### VPC Flow Logs の活用

```
VPC Flow Logs 設定の要点:
  - traffic_type: ALL (ACCEPT + REJECT の両方を記録)
  - 配信先: CloudWatch Logs (KMS 暗号化)
  - 保持期間: 90 日以上
  - max_aggregation_interval: 60 秒

Athena での分析クエリ例:
  - 拒否トラフィックの分析: action = 'REJECT' で集計
  - 不審な外部通信の検出: 内部IP→外部IP で非標準ポート通信を検出
```

---

## 5. セキュリティサービスの全体像

### クラウドセキュリティサービス比較

| カテゴリ | AWS | GCP | Azure |
|---------|-----|-----|-------|
| 脅威検知 | GuardDuty | Security Command Center | Defender for Cloud |
| 統合管理 | Security Hub | SCC Premium | Defender CSPM |
| 監査ログ | CloudTrail | Cloud Audit Logs | Activity Log |
| 設定監査 | Config | Cloud Asset Inventory | Policy |
| WAF | AWS WAF | Cloud Armor | Azure WAF |
| KMS | AWS KMS | Cloud KMS | Key Vault |
| シークレット | Secrets Manager | Secret Manager | Key Vault Secrets |
| DDoS 防御 | Shield / Shield Advanced | Cloud Armor | DDoS Protection |
| ネットワーク FW | Network Firewall | Cloud Firewall | Azure Firewall |
| 脆弱性管理 | Inspector | Security Scanner | Defender for VMs |
| コンテナ | Inspector / ECR Scan | Container Analysis | Defender for Containers |
| SIEM | Security Lake + OpenSearch | Chronicle | Sentinel |

### AWS セキュリティサービスの連携アーキテクチャ

```
+----------------------------------------------------------+
|  AWS セキュリティサービス連携図                              |
|                                                          |
|  [データ収集層]                                            |
|  CloudTrail ──┐                                          |
|  Config ──────┤                                          |
|  GuardDuty ───┤                                          |
|  Inspector ───┤──→ [Security Hub] ──→ [通知・対応]        |
|  Macie ───────┤         |                                |
|  IAM Analyzer ┘         |                                |
|                         v                                |
|                   [Security Lake]                         |
|                         |                                |
|                         v                                |
|                   [Athena / OpenSearch]                   |
|                   (分析・可視化)                           |
|                                                          |
|  [自動修復]                                               |
|  Security Hub                                            |
|       |                                                  |
|       v                                                  |
|  EventBridge ──→ Lambda ──→ 自動修復アクション              |
|  (ルールマッチ)    (修復)    (SG 変更, S3 非公開化等)       |
+----------------------------------------------------------+
```

### GuardDuty の脅威検知パターン

```
+----------------------------------------------------------+
|  GuardDuty が検知する主な脅威カテゴリ                       |
|----------------------------------------------------------|
|                                                          |
|  [偵察 (Recon)]                                          |
|  - ポートスキャン                                         |
|  - 異常な API 呼び出しパターン                             |
|  - DNS クエリの異常                                       |
|                                                          |
|  [不正アクセス (UnauthorizedAccess)]                      |
|  - 通常と異なるリージョンからの API 呼び出し                |
|  - 既知の悪意ある IP からのアクセス                         |
|  - ブルートフォース攻撃 (SSH, RDP)                         |
|                                                          |
|  [暗号通貨マイニング (CryptoCurrency)]                    |
|  - EC2 インスタンスでのマイニング活動検出                    |
|  - 暗号通貨関連の DNS クエリ                               |
|                                                          |
|  [データ窃取 (Exfiltration)]                              |
|  - 異常な量の S3 データダウンロード                         |
|  - 不審な外部エンドポイントへのデータ転送                    |
|                                                          |
|  [権限昇格 (PrivilegeEscalation)]                        |
|  - 通常使わない高権限 API の呼び出し                        |
|  - 管理者ポリシーの不正なアタッチ                           |
|                                                          |
|  重要度: High / Medium / Low で分類                       |
|  → High は即座に調査・対応が必要                           |
+----------------------------------------------------------+
```

### AWS Config ルールによる継続的コンプライアンス

AWS Config のマネージドルールとカスタムルール (Lambda) を組み合わせ、リソースの設定変更を継続的に評価する。カスタムルールでは Lambda 関数内で対象リソースの設定を確認し、`put_evaluations` で COMPLIANT / NON_COMPLIANT を報告する。

```
推奨マネージドルール例:
  - s3-bucket-server-side-encryption-enabled
  - rds-storage-encrypted
  - iam-root-access-key-check
  - cloudtrail-enabled
  - vpc-flow-logs-enabled
```

---

## 6. ログ・監査・コンプライアンス

### CloudTrail の設定と活用

```
+----------------------------------------------------------+
|  CloudTrail のベストプラクティス                             |
|----------------------------------------------------------|
|                                                          |
|  [必須設定]                                               |
|  - 全リージョンで有効化                                    |
|  - S3 バケットにログを集約                                 |
|  - ログファイルの検証 (整合性チェック) を有効化              |
|  - ログバケットの暗号化 (SSE-KMS)                          |
|  - ログバケットのアクセスログを有効化                        |
|  - MFA Delete をログバケットに設定                          |
|                                                          |
|  [推奨設定]                                               |
|  - CloudWatch Logs への配信                               |
|  - メトリクスフィルタとアラームの設定                        |
|  - AWS Organizations の組織トレイルとして設定               |
|  - S3 データイベント、Lambda データイベントの記録            |
|  - Athena テーブルを作成して分析可能にする                   |
+----------------------------------------------------------+
```

Athena を使った CloudTrail ログの分析クエリ例:

```sql
-- 未承認 API 呼び出しの検出
SELECT eventtime, useridentity.arn, eventname,
       errorcode, sourceipaddress
FROM cloudtrail_logs
WHERE errorcode IN ('AccessDenied', 'UnauthorizedAccess')
ORDER BY eventtime DESC LIMIT 100;
```

他に実装すべきクエリ: IAM ポリシー変更の追跡、ルートアカウント使用の検出、コンソールログイン失敗の検出。

### CloudWatch によるセキュリティモニタリング

CloudTrail ログに対してメトリクスフィルタを設定し、重要なイベントを検出する。

```
設定すべきメトリクスフィルタ:
+----------------------------------------------------------+
| 検出対象                    | フィルタパターン (要点)       |
|----------------------------------------------------------+
| ルートアカウント使用         | userIdentity.type = "Root"  |
| MFA なしコンソールログイン   | MFAUsed != "Yes"            |
| セキュリティグループ変更     | AuthorizeSecurityGroup*     |
| IAM ポリシー変更            | CreatePolicy, AttachRole*   |
| CloudTrail 設定変更         | StopLogging, DeleteTrail    |
+----------------------------------------------------------+
→ 各フィルタに CloudWatch Alarm + SNS 通知を設定
```

### コンプライアンスフレームワークとの対応

```
+----------------------------------------------------------+
|  主要コンプライアンスフレームワーク                           |
|----------------------------------------------------------|
|                                                          |
|  [CIS Benchmark]                                         |
|  - AWS CIS Benchmark v3.0                                |
|  - 9カテゴリ、約70のチェック項目                            |
|  - Security Hub で自動チェック可能                         |
|  - Level 1 (基本) / Level 2 (高度) の2段階                |
|                                                          |
|  [SOC 2]                                                 |
|  - セキュリティ、可用性、処理の整合性、機密性、プライバシー  |
|  - AWS Audit Manager で証跡収集を自動化                    |
|                                                          |
|  [PCI DSS]                                               |
|  - クレジットカード情報を扱うシステムに必須                  |
|  - ネットワーク分離、暗号化、アクセスログが重点              |
|  - AWS Config Rules で継続的な準拠チェック                  |
|                                                          |
|  [HIPAA]                                                 |
|  - 医療情報 (PHI) の保護に関する規制                       |
|  - AWS BAA (Business Associate Agreement) の締結が必要    |
|  - 暗号化とアクセスログが特に重要                           |
|                                                          |
|  [GDPR]                                                  |
|  - EU 個人データの保護                                     |
|  - データのリージョン制限が重要                             |
|  - データ削除 (忘れられる権利) への対応                     |
+----------------------------------------------------------+
```

---

## 7. インシデント対応

### クラウド環境のインシデントレスポンスプロセス

```
+----------------------------------------------------------+
|  クラウドインシデント対応フロー                              |
|                                                          |
|  [1. 検知 (Detection)]                                   |
|  +-- GuardDuty アラート                                   |
|  +-- CloudWatch アラーム                                  |
|  +-- Security Hub Findings                               |
|  +-- サードパーティ SIEM アラート                          |
|       |                                                  |
|       v                                                  |
|  [2. トリアージ (Triage)]                                 |
|  +-- 重要度の判定 (High/Medium/Low)                       |
|  +-- 影響範囲の特定                                       |
|  +-- エスカレーション判断                                  |
|       |                                                  |
|       v                                                  |
|  [3. 封じ込め (Containment)]                              |
|  +-- セキュリティグループの変更 (通信遮断)                  |
|  +-- IAM 認証情報の無効化                                  |
|  +-- インスタンスの隔離 (隔離用 SG にアタッチ)              |
|  +-- ★クラウド固有: スナップショット取得 (証拠保全)         |
|       |                                                  |
|       v                                                  |
|  [4. 根絶 (Eradication)]                                 |
|  +-- マルウェアの除去                                     |
|  +-- 脆弱性のパッチ適用                                    |
|  +-- 侵害されたリソースの再構築                             |
|       |                                                  |
|       v                                                  |
|  [5. 復旧 (Recovery)]                                    |
|  +-- クリーンなイメージからの再デプロイ                     |
|  +-- 段階的なトラフィック復旧                               |
|  +-- モニタリングの強化                                    |
|       |                                                  |
|       v                                                  |
|  [6. 教訓 (Lessons Learned)]                             |
|  +-- インシデント報告書の作成                               |
|  +-- 根本原因分析 (RCA)                                   |
|  +-- 改善策の実施                                         |
|  +-- ランブック / プレイブックの更新                        |
+----------------------------------------------------------+
```

### 自動封じ込めの要点

EC2 インスタンスが侵害された場合の封じ込め手順:

```
1. EBS スナップショット取得 (証拠保全)
2. 隔離用セキュリティグループ作成 (ingress/egress 全遮断)
3. インスタンスの SG を隔離用に変更
4. タグで状態を記録 (SecurityStatus=ISOLATED)
※ インスタンスは停止しない (メモリ上の証拠を保全)
```

### IAM 認証情報の緊急無効化

IAM ユーザの侵害が疑われる場合の対応手順:

```
1. 全アクセスキーを無効化 (update_access_key → Inactive)
2. コンソールパスワードの削除 (delete_login_profile)
3. 全ポリシーのデタッチ
4. 明示的 Deny ポリシーをアタッチ (二重防御)
   {"Effect": "Deny", "Action": "*", "Resource": "*"}
```

---

## 8. アンチパターン

### アンチパターン 1: ルートアカウントの日常使用

```
NG:
  → root ユーザで日常的にログイン
  → root のアクセスキーを作成してスクリプトで使用

OK:
  → root には MFA を設定し緊急時のみ使用
  → 日常運用は IAM ユーザ/ロール経由
  → root のアクセスキーは作成しない
  → 管理操作は Organizations の管理アカウントから
```

### アンチパターン 2: セキュリティグループの 0.0.0.0/0 許可

```hcl
# NG: 全ポートを全世界に公開
resource "aws_security_group_rule" "allow_all" {
  type        = "ingress"
  from_port   = 0
  to_port     = 65535
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}

# OK: 必要なポートと送信元のみ
resource "aws_security_group_rule" "allow_https" {
  type        = "ingress"
  from_port   = 443
  to_port     = 443
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]  # HTTPS は公開 OK
}

resource "aws_security_group_rule" "allow_ssh" {
  type        = "ingress"
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = ["10.0.0.0/8"]  # 社内のみ
}
```

### アンチパターン 3: 長期アクセスキーの利用

```
NG:
  → アプリケーションに IAM ユーザのアクセスキーをハードコード
  → .env ファイルにアクセスキーを記載して Git にコミット
  → 全開発者が同じアクセスキーを共有

OK:
  → EC2 / Lambda には IAM ロールをアタッチ
  → ローカル開発では aws-vault や SSO を使用
  → CI/CD では OIDC 連携 (GitHub Actions の場合)

# GitHub Actions での OIDC 連携例:
"""
# .github/workflows/deploy.yml
jobs:
  deploy:
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: ap-northeast-1
          # アクセスキーは不要、OIDC トークンで認証
"""
```

### アンチパターン 4: ログの未設定・未監視

```
NG:
  → CloudTrail を有効化していない
  → ログは有効だが誰も見ていない
  → ログの保持期間が短すぎる (デフォルト 90 日のみ)
  → ログバケットに暗号化やアクセス制限がない

OK:
  → CloudTrail + Config + GuardDuty を全リージョンで有効化
  → Security Hub に集約して一元的に監視
  → メトリクスフィルタとアラームで重要イベントを通知
  → ログを S3 に長期保存 (最低 1 年、法的要件に応じて延長)
  → ログバケットは KMS 暗号化 + MFA Delete
  → 定期的なログレビュープロセスを確立
```

### アンチパターン 5: 暗号化の不完全な適用

```
NG:
  → S3 は暗号化したが EBS は暗号化していない
  → 保存時は暗号化したが転送時 (HTTP) は未暗号化
  → KMS キーのローテーションが無効
  → 暗号化キーのアクセス制御が甘い

OK:
  → 全ストレージ (S3, EBS, RDS, EFS) を暗号化
  → 全通信を TLS 1.2 以上で暗号化
  → KMS キーの自動ローテーションを有効化
  → キーポリシーで使用者を限定
  → リージョン間のデータ転送も VPN / PrivateLink で暗号化
```

---

## 9. ゼロトラストアーキテクチャとクラウド

### ゼロトラストの原則

```
+----------------------------------------------------------+
|  ゼロトラスト 5 原則                                       |
|----------------------------------------------------------|
|                                                          |
|  1. すべてのリソースへのアクセスは認証・認可が必要            |
|     - ネットワーク内部であっても暗黙的に信頼しない            |
|     - 全リクエストにアイデンティティ検証を要求                |
|                                                          |
|  2. アクセスは最小権限で付与                                |
|     - Just-In-Time (JIT) アクセス                         |
|     - 必要な期間のみ権限を付与                              |
|                                                          |
|  3. アクセスはセッション単位で評価                           |
|     - デバイスの状態                                       |
|     - ユーザのコンテキスト (場所、時間)                     |
|     - リスクスコアに基づく動的判断                          |
|                                                          |
|  4. ネットワークセグメンテーション (マイクロセグメンテーション)|
|     - サービス間通信にも認証を要求                           |
|     - サービスメッシュ (Istio, App Mesh) の活用             |
|                                                          |
|  5. すべてのアクティビティを記録・監視                       |
|     - 異常検知の自動化                                     |
|     - 継続的なリスク評価                                    |
+----------------------------------------------------------+
```

### クラウドにおけるゼロトラストの実装例

```
+----------------------------------------------------------+
|  AWS でのゼロトラスト実装                                   |
|                                                          |
|  [アイデンティティ層]                                      |
|  IAM Identity Center + MFA                               |
|       |                                                  |
|  [ネットワーク層]                                          |
|  VPC + PrivateLink + VPC Endpoints                       |
|  (インターネットを経由しない通信)                            |
|       |                                                  |
|  [アプリケーション層]                                      |
|  AWS Verified Access                                     |
|  (VPN 不要でゼロトラストアクセス)                           |
|       |                                                  |
|  [データ層]                                               |
|  KMS 暗号化 + S3 アクセスポイント                          |
|  + Macie (データ分類)                                     |
|       |                                                  |
|  [監視層]                                                |
|  GuardDuty + Security Hub + CloudTrail                   |
|  (継続的な監視と脅威検知)                                   |
+----------------------------------------------------------+
```

## 10. FAQ

### Q1. クラウドのセキュリティは自社データセンターより安全か?

インフラの物理セキュリティ、DDoS 対策、パッチ適用は大手クラウドプロバイダの方が優れている。ただし設定ミスによるデータ漏洩は依然としてユーザ責任である。S3 バケットの公開設定ミスによるデータ流出は後を絶たない。結局のところ「クラウドのインフラは安全、しかしクラウドの中の設定は自己責任」という理解が正しい。

### Q2. IMDS (Instance Metadata Service) v1 と v2 の違いは?

IMDS v1 は単純な HTTP GET でインスタンスメタデータにアクセスでき、SSRF 攻撃で認証情報が窃取されるリスクがある (Capital One 事件)。IMDS v2 ではセッショントークンが必要になり、PUT メソッドでトークンを取得した上でアクセスする仕組みとなっている。全インスタンスで IMDS v2 のみを許可する設定を推奨する。

```python
# IMDS v2 の強制設定 (Terraform)
"""
resource "aws_instance" "example" {
  # ...
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"  # v2 のみ許可
    http_put_response_hop_limit = 1           # コンテナの場合は 2
  }
}
"""
```

### Q3. クラウドでの DDoS 対策はどうすべきか?

基本的な DDoS 防御は AWS Shield Standard (無料) で自動的に提供される。大規模な攻撃に対しては Shield Advanced (有料) が DDoS Response Team (DRT) と24時間365日のサポートを提供する。CloudFront + AWS WAF の組み合わせでアプリケーション層 (L7) の攻撃にも対処できる。

---

## まとめ

| 項目 | 要点 |
|------|------|
| 責任共有モデル | インフラはプロバイダ、設定・データはユーザの責任 |
| IAM | 最小権限の原則、ロールベース、MFA 必須、ABAC の活用 |
| 暗号化 | 保存時 (KMS) + 転送時 (TLS) を常に有効化、エンベロープ暗号化 |
| ネットワーク | VPC セグメンテーション + VPC Endpoints + Flow Logs |
| マルチアカウント | 環境別にアカウントを分離し SCP で統制 |
| 監視 | CloudTrail + Config + GuardDuty + Security Hub を必ず有効化 |
| シークレット管理 | Secrets Manager で一元管理、自動ローテーション |
| インシデント対応 | 検知→封じ込め→根絶→復旧→教訓のフローを自動化 |
| ゼロトラスト | ネットワーク位置ではなくアイデンティティで信頼を判断 |
| コンプライアンス | CIS Benchmark + Config Rules で継続的に準拠状態を監視 |

---

## 参考文献

1. **AWS Well-Architected Framework — Security Pillar** — https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/
2. **CIS Benchmarks for Cloud Providers** — https://www.cisecurity.org/cis-benchmarks
3. **NIST SP 800-144 — Guidelines on Security and Privacy in Public Cloud Computing** — https://csrc.nist.gov/publications/detail/sp/800-144/final
4. **CSA Cloud Controls Matrix** — https://cloudsecurityalliance.org/research/cloud-controls-matrix/
5. **AWS Security Best Practices** — https://aws.amazon.com/architecture/security-identity-compliance/
6. **NIST SP 800-207 — Zero Trust Architecture** — https://csrc.nist.gov/publications/detail/sp/800-207/final
7. **AWS IAM Best Practices** — https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
8. **AWS Incident Response Guide** — https://docs.aws.amazon.com/whitepapers/latest/aws-security-incident-response-guide/
9. **OWASP Cloud Security** — https://owasp.org/www-project-cloud-security/
10. **Google Cloud Security Best Practices** — https://cloud.google.com/security/best-practices
