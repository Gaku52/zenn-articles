---
title: "セキュリティとシークレット管理 - 安全なCI/CD運用"
---

# Chapter 06 - セキュリティとシークレット管理

## CI/CDセキュリティの重要性

CI/CDパイプラインは、本番環境への直接的なアクセス権を持つため、セキュリティ侵害の標的になりやすい領域です。適切なシークレット管理と脆弱性対策が不可欠です。

### 想定される効果: セキュリティインシデント削減効果

あるフィンテックスタートアップ(従業員50名)での想定シナリオ:

**最適化前:**
- APIキー漏洩: 年2件
- 脆弱性検出: 本番環境で発見
- シークレット管理: .envファイルで手動管理
- アクセス権限: 全開発者が本番アクセス可能

**最適化後:**
- ✅ APIキー漏洩: 年2件 → 0件 (-100%)
- ✅ 脆弱性検出: CI段階で自動検出(本番流入0件)
- ✅ シークレット管理: GitHub Secrets + OIDC自動化
- ✅ アクセス権限: 最小権限の原則適用
- ✅ 監査ログ: 全デプロイ操作を記録

## GitHub Secrets管理

### 基本的なSecrets設定

```yaml
# .github/workflows/secure-deploy.yml
name: Secure Deployment

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # 環境保護ルール適用

    steps:
      - uses: actions/checkout@v4

      - name: 環境変数設定
        env:
          # ❌ 悪い例: シークレットを直接表示
          # API_KEY: ${{ secrets.API_KEY }}

          # ✅ 良い例: 環境変数経由で使用
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          API_KEY: ${{ secrets.API_KEY }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          # Secretsはログに***として表示される
          echo "Deploying with API_KEY: $API_KEY"  # *** と表示

          # .envファイル生成
          cat > .env.production <<EOF
          DATABASE_URL=$DATABASE_URL
          API_KEY=$API_KEY
          EOF

      - name: デプロイ
        run: npm run deploy
```

### 環境別Secrets管理

```yaml
# Settings → Environments で設定

# Development環境
jobs:
  deploy-dev:
    environment: development
    steps:
      - env:
          API_URL: ${{ secrets.DEV_API_URL }}  # Development用
        run: npm run deploy:dev

# Staging環境
  deploy-staging:
    environment: staging
    steps:
      - env:
          API_URL: ${{ secrets.STAGING_API_URL }}  # Staging用
        run: npm run deploy:staging

# Production環境(承認必須)
  deploy-prod:
    environment: production
    steps:
      - env:
          API_URL: ${{ secrets.PROD_API_URL }}  # Production用
        run: npm run deploy:prod
```

**環境保護ルール設定:**
```
Settings → Environments → production
✅ Required reviewers: CTO, テックリード
✅ Wait timer: 5分(緊急時キャンセル可能)
✅ Deployment branches: main のみ
```

## OIDC認証によるパスワードレス認証

### AWS OIDCセットアップ

```yaml
# .github/workflows/aws-oidc.yml
name: AWS Deploy with OIDC

on:
  push:
    branches: [main]

permissions:
  id-token: write  # OIDCトークン取得に必要
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: AWS認証(OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: ap-northeast-1
          # アクセスキー不要!

      - name: S3デプロイ
        run: aws s3 sync dist/ s3://my-bucket/
```

**AWS IAMロール設定(Terraform):**

```hcl
# terraform/iam-oidc.tf

# OIDCプロバイダー
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = ["sts.amazonaws.com"]

  thumbprint_list = [
    "6938fd4d98bab03faadb97b34396831e3780aea1"
  ]
}

# GitHubActions用IAMロール
resource "aws_iam_role" "github_actions" {
  name = "GitHubActionsRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.github.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
          }
          StringLike = {
            "token.actions.githubusercontent.com:sub" = "repo:your-org/your-repo:*"
          }
        }
      }
    ]
  })
}

# S3アクセス権限
resource "aws_iam_role_policy" "github_actions_s3" {
  role = aws_iam_role.github_actions.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:PutObject",
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Resource = [
          "arn:aws:s3:::my-bucket",
          "arn:aws:s3:::my-bucket/*"
        ]
      }
    ]
  })
}
```

**想定効果:**
- アクセスキーローテーション: 不要(自動)
- セキュリティリスク: -100%(キー漏洩リスク排除)
- 設定時間: 初回30分、以降メンテナンス不要

## 脆弱性スキャン

### 依存関係の脆弱性スキャン

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [main, develop]
  pull_request:
  schedule:
    - cron: '0 0 * * 1'  # 毎週月曜0時

jobs:
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: npm audit
        run: |
          npm audit --audit-level=moderate
          npm audit fix --dry-run

      - name: Snyk スキャン
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

      - name: Trivy スキャン
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: 結果をGitHub Security タブにアップロード
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'
```

### Dockerイメージスキャン

```yaml
# .github/workflows/docker-scan.yml
name: Docker Image Scan

on:
  push:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Dockerイメージビルド
        run: docker build -t myapp:${{ github.sha }} .

      - name: Trivyでイメージスキャン
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:${{ github.sha }}'
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH'

      - name: Dockleでベストプラクティスチェック
        uses: goodwithtech/dockle-action@main
        with:
          image: 'myapp:${{ github.sha }}'
          format: 'json'
          exit-code: '1'
          exit-level: 'warn'
```

**想定検出率:**
- Critical脆弱性: 月平均2-3件検出
- High脆弱性: 月平均5-8件検出
- 本番流入: 0件(CI段階で全てブロック)

## コードセキュリティスキャン

### CodeQL による静的解析

```yaml
# .github/workflows/codeql-analysis.yml
name: "CodeQL"

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * 0'  # 毎週日曜0時

jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write

    strategy:
      fail-fast: false
      matrix:
        language: ['javascript', 'typescript']

    steps:
      - uses: actions/checkout@v4

      - name: CodeQL 初期化
        uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
          queries: security-extended,security-and-quality

      - name: 自動ビルド
        uses: github/codeql-action/autobuild@v3

      - name: CodeQL 分析
        uses: github/codeql-action/analyze@v3
        with:
          category: "/language:${{ matrix.language }}"
```

### シークレットスキャン

```yaml
# .github/workflows/secret-scan.yml
name: Secret Scan

on: [push, pull_request]

jobs:
  gitleaks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Gitleaks スキャン
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  trufflehog:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: TruffleHog スキャン
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
```

**検出例:**
```
❌ AWS Access Key: AKIAIOSFODNN7EXAMPLE
❌ GitHub Token: ghp_xxxxxxxxxxxxxxxxxxxx
❌ Private Key: -----BEGIN RSA PRIVATE KEY-----
```

## 権限管理

### 最小権限の原則

```yaml
# .github/workflows/minimal-permissions.yml
name: Minimal Permissions

on: [push]

# デフォルトで全て読み取り専用
permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    # このジョブは読み取りのみ
    steps:
      - uses: actions/checkout@v4
      - run: npm test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    # デプロイは必要な権限のみ付与
    permissions:
      contents: read
      id-token: write  # OIDC用
      deployments: write  # デプロイメント作成
    steps:
      - uses: actions/checkout@v4
      - run: npm run deploy

  comment:
    runs-on: ubuntu-latest
    # PRコメント用
    permissions:
      pull-requests: write
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: 'Deployment completed!'
            })
```

### サードパーティActionのバージョン固定

```yaml
# ❌ 悪い例: 最新版を常に使用(セキュリティリスク)
- uses: actions/checkout@v4

# ✅ 良い例: コミットSHAで固定
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # v4.1.1

# ✅ Dependabotで自動アップデート
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    reviewers:
      - "tech-lead"
    labels:
      - "dependencies"
      - "github-actions"
```

## セキュリティ監査ログ

### デプロイ操作の記録

```yaml
# .github/workflows/audit-log.yml
name: Deployment with Audit Log

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: デプロイ情報記録
        run: |
          cat > deployment-log.json <<EOF
          {
            "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
            "actor": "${{ github.actor }}",
            "commit": "${{ github.sha }}",
            "branch": "${{ github.ref }}",
            "workflow": "${{ github.workflow }}",
            "run_id": "${{ github.run_id }}"
          }
          EOF

      - name: デプロイ実行
        run: npm run deploy

      - name: 監査ログをS3に保存
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          aws s3 cp deployment-log.json \
            s3://audit-logs/deployments/$(date +%Y/%m/%d)/${{ github.run_id }}.json

      - name: Slack通知(監査用)
        run: |
          curl -X POST ${{ secrets.SLACK_AUDIT_WEBHOOK }} \
            -H 'Content-Type: application/json' \
            -d '{
              "text": "🔐 Deployment Audit",
              "blocks": [
                {
                  "type": "section",
                  "fields": [
                    {"type": "mrkdwn", "text": "*Actor:* ${{ github.actor }}"},
                    {"type": "mrkdwn", "text": "*Commit:* ${{ github.sha }}"},
                    {"type": "mrkdwn", "text": "*Time:* $(date)"}
                  ]
                }
              ]
            }'
```

## トラブルシューティング

### 問題1: Secretsが読めない

**症状:**
```
Error: API_KEY is undefined
```

**対処法:**

```yaml
# 1. Secretsの存在確認
- name: Secrets確認
  run: |
    if [ -z "${{ secrets.API_KEY }}" ]; then
      echo "❌ API_KEY が設定されていません"
      echo "Settings → Secrets and variables → Actions で設定してください"
      exit 1
    fi

# 2. 環境別Secretsの確認
- name: 環境確認
  run: |
    echo "Environment: ${{ github.environment }}"
    echo "環境に対応するSecretsが設定されているか確認してください"
```

### 問題2: OIDC認証が失敗する

**症状:**
```
Error: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

**対処法:**

```yaml
# 1. 権限の確認
permissions:
  id-token: write  # 必須!
  contents: read

# 2. IAMロールの信頼ポリシー確認
- name: OIDC デバッグ
  run: |
    echo "Repository: ${{ github.repository }}"
    echo "Ref: ${{ github.ref }}"
    # IAMロールの信頼ポリシーと一致するか確認
```

### 問題3: 脆弱性スキャンで誤検知

**対処法:**

```yaml
# .trivyignore
# 誤検知を除外
CVE-2021-12345  # False positive: not applicable to our use case

# または
- name: Trivy スキャン(許容レベル設定)
  uses: aquasecurity/trivy-action@master
  with:
    severity: 'CRITICAL'  # HIGHは警告のみ
    exit-code: '0'  # スキャン結果でFailしない
```

## まとめ

この章では、安全なCI/CD運用のためのセキュリティ対策を学びました:

✅ **Secrets管理**: GitHub Secrets、環境別設定、OIDC認証
✅ **脆弱性スキャン**: 依存関係、Dockerイメージ、コード静的解析
✅ **権限管理**: 最小権限の原則、Actionバージョン固定
✅ **監査ログ**: 全デプロイ操作の記録と追跡

### 重要な想定される効果まとめ

| 対策 | 効果 |
|------|------|
| OIDC認証導入 | キー漏洩リスク -100% |
| 脆弱性スキャン | 本番流入 0件 |
| 環境保護ルール | 誤デプロイ -100% |
| 監査ログ | 全操作追跡可能 |

### 次のステップ

**Chapter 07 - CI/CDパフォーマンス最適化**では、ビルド時間短縮、キャッシュ戦略、並列実行によるパイプライン高速化を学びます。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
