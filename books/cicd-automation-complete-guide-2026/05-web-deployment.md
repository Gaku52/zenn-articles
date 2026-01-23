---
title: "Web自動デプロイ - Vercel/Netlify/AWS完全ガイド"
---

# Chapter 05 - Web自動デプロイ(Vercel/Netlify/AWS)

## Web デプロイメント概要

Webアプリケーションの自動デプロイは、開発速度とサービス品質を劇的に向上させます。本章では、主要なプラットフォーム(Vercel、Netlify、AWS)への自動デプロイを実装します。

### 想定される効果: Web自動デプロイ導入効果

あるSaaSプロダクト(月間50万PV、Next.js製)での想定シナリオ:

**最適化前:**
- デプロイ作業: 手動30分
- デプロイ頻度: 週1回
- 環境差異バグ: 月3件
- ロールバック時間: 平均20分

**最適化後:**
- ✅ デプロイ作業: 30分 → 3分 (-90%)
- ✅ デプロイ頻度: 週1回 → 日5回 (+500%)
- ✅ 環境差異バグ: 月3件 → 0件 (-100%)
- ✅ ロールバック時間: 20分 → 30秒 (-97.5%)
- ✅ プレビューURL: 全PR自動生成

## Vercelへの自動デプロイ

### Vercel CLI を使用した基本デプロイ

```yaml
# .github/workflows/vercel-deploy.yml
name: Deploy to Vercel

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  deploy-preview:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    environment:
      name: preview
      url: ${{ steps.deploy.outputs.preview-url }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: 依存関係インストール
        run: npm ci

      - name: Vercel CLIインストール
        run: npm i -g vercel@latest

      - name: プレビューデプロイ
        id: deploy
        env:
          VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}
          VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
          VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}
        run: |
          PREVIEW_URL=$(vercel deploy --token=$VERCEL_TOKEN)
          echo "preview-url=$PREVIEW_URL" >> $GITHUB_OUTPUT
          echo "✅ Preview deployed to: $PREVIEW_URL"

      - name: PRにプレビューURLをコメント
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## 🚀 Preview Deployment\n\nYour preview is ready!\n\n👉 ${{ steps.deploy.outputs.preview-url }}`
            })

  deploy-production:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://your-app.vercel.app

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: 本番デプロイ
        env:
          VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}
          VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
          VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}
        run: |
          npm i -g vercel@latest
          vercel --prod --token=$VERCEL_TOKEN

      - name: Slack通知
        if: success()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -H 'Content-Type: application/json' \
            -d '{"text":"✅ Production deployed successfully!"}'
```

**想定時間(Next.jsアプリ):**
- プレビューデプロイ: 2-3分
- 本番デプロイ: 3-4分
- ロールバック: 30秒

### Vercel Action を使用した高度なデプロイ

```yaml
# .github/workflows/vercel-advanced.yml
name: Advanced Vercel Deploy

on:
  push:
    branches: [main]
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Vercelデプロイ
        uses: amondnet/vercel-action@v25
        id: vercel-deploy
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: ${{ github.event_name == 'push' && '--prod' || '' }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
          github-comment: true
          working-directory: ./

      - name: E2Eテスト(デプロイ済み環境)
        run: |
          export PLAYWRIGHT_TEST_BASE_URL=${{ steps.vercel-deploy.outputs.preview-url }}
          npm run test:e2e
```

## Netlifyへの自動デプロイ

### Netlify CLIを使用したデプロイ

```yaml
# .github/workflows/netlify-deploy.yml
name: Deploy to Netlify

on:
  push:
    branches: [main]
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: ビルド
        run: npm run build
        env:
          NEXT_PUBLIC_API_URL: ${{ secrets.API_URL }}

      - name: Netlifyにデプロイ
        uses: nwtgck/actions-netlify@v2.1
        with:
          publish-dir: './out'
          production-branch: main
          github-token: ${{ secrets.GITHUB_TOKEN }}
          deploy-message: "Deploy from GitHub Actions"
          enable-pull-request-comment: true
          enable-commit-comment: true
          overwrites-pull-request-comment: true
        env:
          NETLIFY_AUTH_TOKEN: ${{ secrets.NETLIFY_AUTH_TOKEN }}
          NETLIFY_SITE_ID: ${{ secrets.NETLIFY_SITE_ID }}
        timeout-minutes: 10
```

### Netlify設定ファイル

```toml
# netlify.toml
[build]
  command = "npm run build"
  publish = "out"
  functions = "netlify/functions"

[build.environment]
  NODE_VERSION = "20"
  NPM_FLAGS = "--prefix=/dev/null"

[[redirects]]
  from = "/api/*"
  to = "/.netlify/functions/:splat"
  status = 200

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200

[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-Content-Type-Options = "nosniff"
    Referrer-Policy = "strict-origin-when-cross-origin"

[context.production]
  environment = { NODE_ENV = "production" }

[context.deploy-preview]
  environment = { NODE_ENV = "preview" }
```

**想定時間:**
- ビルド: 2-3分
- デプロイ: 30-60秒
- プレビュー生成: 3-4分

## AWS S3 + CloudFront デプロイ

### S3バケット設定

```yaml
# .github/workflows/aws-deploy.yml
name: Deploy to AWS S3

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: 依存関係とビルド
        run: |
          npm ci
          npm run build

      - name: AWS認証(OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: ap-northeast-1

      - name: S3にデプロイ
        run: |
          aws s3 sync out/ s3://your-bucket-name/ \
            --delete \
            --cache-control "public, max-age=31536000, immutable" \
            --exclude "*.html" \
            --exclude "service-worker.js"

          # HTMLファイルはキャッシュ無効
          aws s3 sync out/ s3://your-bucket-name/ \
            --exclude "*" \
            --include "*.html" \
            --cache-control "public, max-age=0, must-revalidate"

      - name: CloudFrontキャッシュ無効化
        run: |
          aws cloudfront create-invalidation \
            --distribution-id E1234567890ABC \
            --paths "/*"

      - name: デプロイ完了通知
        if: success()
        run: |
          echo "✅ Deployed to https://your-domain.com"
```

**想定時間:**
- S3アップロード: 1-2分
- CloudFront無効化: 30-60秒
- 合計: 2-3分

### AWS CDKによるインフラ構築

```typescript
// lib/frontend-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as cloudfront from 'aws-cdk-lib/aws-cloudfront';
import * as origins from 'aws-cdk-lib/aws-cloudfront-origins';
import * as s3deploy from 'aws-cdk-lib/aws-s3-deployment';

export class FrontendStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // S3バケット
    const websiteBucket = new s3.Bucket(this, 'WebsiteBucket', {
      versioned: true,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
    });

    // CloudFront
    const distribution = new cloudfront.Distribution(this, 'Distribution', {
      defaultBehavior: {
        origin: new origins.S3Origin(websiteBucket),
        viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
        cachePolicy: cloudfront.CachePolicy.CACHING_OPTIMIZED,
      },
      defaultRootObject: 'index.html',
      errorResponses: [
        {
          httpStatus: 404,
          responseHttpStatus: 200,
          responsePagePath: '/index.html',
        },
      ],
    });

    new cdk.CfnOutput(this, 'DistributionDomainName', {
      value: distribution.distributionDomainName,
    });
  }
}
```

## Docker + AWS ECSデプロイ

### Dockerfileの最適化

```dockerfile
# Dockerfile
FROM node:20-alpine AS base

# 依存関係インストール
FROM base AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# ビルド
FROM base AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# 本番環境
FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT 3000

CMD ["node", "server.js"]
```

### ECSデプロイワークフロー

```yaml
# .github/workflows/ecs-deploy.yml
name: Deploy to ECS

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: AWS認証
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: ap-northeast-1

      - name: ECRログイン
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2

      - name: Dockerイメージビルド・プッシュ
        env:
          ECR_REGISTRY: ${{ steps.ecr-login.outputs.registry }}
          ECR_REPOSITORY: my-app
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
                     $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

      - name: タスク定義更新
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: my-app
          image: ${{ steps.ecr-login.outputs.registry }}/my-app:${{ github.sha }}

      - name: ECSサービス更新
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: my-app-service
          cluster: production-cluster
          wait-for-service-stability: true
```

**想定時間(Next.jsアプリ):**
- Dockerビルド: 3-5分
- ECRプッシュ: 1-2分
- ECSデプロイ: 2-3分
- 合計: 6-10分

## トラブルシューティング

### 問題1: Vercelデプロイが失敗する

**症状:**
```
Error: Failed to deploy to Vercel
```

**対処法:**

```yaml
# 1. トークンの確認
- name: トークン検証
  env:
    VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}
  run: |
    if [ -z "$VERCEL_TOKEN" ]; then
      echo "❌ VERCEL_TOKEN が設定されていません"
      exit 1
    fi

# 2. プロジェクトID確認
- name: Vercel設定確認
  run: |
    vercel whoami --token=${{ secrets.VERCEL_TOKEN }}
    vercel ls --token=${{ secrets.VERCEL_TOKEN }}
```

### 問題2: CloudFrontキャッシュが更新されない

**症状:**
```
デプロイ後も古いコンテンツが表示される
```

**対処法:**

```yaml
# キャッシュ無効化を確実に実行
- name: CloudFront無効化(リトライ付き)
  uses: nick-fields/retry@v2
  with:
    timeout_minutes: 5
    max_attempts: 3
    command: |
      aws cloudfront create-invalidation \
        --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
        --paths "/*" "/_next/*"
```

### 問題3: Dockerビルドが遅い

**対処法:**

```yaml
# Docker Buildxでキャッシュ活用
- name: Docker Buildx セットアップ
  uses: docker/setup-buildx-action@v3

- name: キャッシュ付きビルド
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ${{ steps.ecr-login.outputs.registry }}/my-app:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

**想定効果:**
- 初回ビルド: 5分
- キャッシュ利用時: 1-2分 (-60-80%)

## まとめ

この章では、主要プラットフォームへのWeb自動デプロイを学びました:

✅ **Vercel**: 最速デプロイ、自動プレビューURL生成
✅ **Netlify**: 柔軟な設定、エッジファンクション対応
✅ **AWS S3/CloudFront**: フルコントロール、大規模対応
✅ **AWS ECS**: コンテナベース、マイクロサービス対応

### 重要な想定される効果まとめ

| プラットフォーム | デプロイ時間 | ロールバック | コスト(月間50万PV) |
|--------------|-----------|------------|-----------------|
| Vercel | 2-4分 | 30秒 | $20-$50 |
| Netlify | 3-5分 | 30秒 | $19-$45 |
| AWS S3+CF | 2-3分 | 1分 | $5-$15 |
| AWS ECS | 6-10分 | 2-3分 | $30-$100 |

### 次のステップ

**Chapter 06 - セキュリティとシークレット管理**では、安全なCI/CD運用のためのシークレット管理、脆弱性スキャン、OIDC認証を学びます。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
