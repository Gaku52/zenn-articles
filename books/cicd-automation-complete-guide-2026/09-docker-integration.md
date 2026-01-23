---
title: "Docker統合とコンテナビルド - 最適化の全技法"
---

# Chapter 09 - Docker統合とコンテナビルド

## Dockerコンテナ化の重要性

Dockerコンテナは、アプリケーションの一貫性、移植性、スケーラビリティを提供します。適切な最適化により、ビルド時間を大幅に短縮し、イメージサイズを削減できます。

### 想定される効果: Docker最適化効果

あるマイクロサービスアーキテクチャ(10サービス、Node.js)での想定シナリオ:

**最適化前:**
- Dockerビルド時間: 平均8分/サービス
- イメージサイズ: 1.2GB/サービス
- デプロイ時間: 12分(プル+起動)
- レジストリストレージ: 120GB

**最適化後:**
- ✅ ビルド時間: 8分 → 2分 (-75%)
- ✅ イメージサイズ: 1.2GB → 180MB (-85%)
- ✅ デプロイ時間: 12分 → 3分 (-75%)
- ✅ レジストリストレージ: 120GB → 18GB (-85%)
- ✅ 月間コスト削減: $180

## マルチステージビルド

### 基本的なマルチステージビルド

```dockerfile
# ❌ 悪い例: シングルステージ (1.2GB)
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
CMD ["node", "dist/server.js"]

# ✅ 良い例: マルチステージビルド (180MB)
# ステージ1: 依存関係インストール
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# ステージ2: ビルド
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# ステージ3: 本番環境
FROM node:20-alpine AS runner
WORKDIR /app

# 非rootユーザー作成
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 appuser

# 依存関係と成果物のみコピー
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./package.json

USER appuser
EXPOSE 3000
ENV NODE_ENV=production

CMD ["node", "dist/server.js"]
```

**想定比較:**

| 項目 | シングルステージ | マルチステージ | 削減率 |
|------|---------------|-------------|--------|
| イメージサイズ | 1.2GB | 180MB | -85% |
| ビルド時間 | 8分 | 6分 | -25% |
| 脆弱性 | 42件 | 3件 | -93% |

### Next.js アプリケーションの最適化

```dockerfile
# Dockerfile (Next.js Standalone)
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Next.js Standalone モード
ENV NEXT_TELEMETRY_DISABLED 1
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# 公開ファイル
COPY --from=builder /app/public ./public

# Standalone 出力
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000
ENV PORT 3000

CMD ["node", "server.js"]
```

**next.config.js 設定:**

```javascript
// next.config.js
module.exports = {
  output: 'standalone',  // Dockerビルド最適化
  swcMinify: true,
  compiler: {
    removeConsole: process.env.NODE_ENV === 'production',
  },
};
```

**想定結果:**
- イメージサイズ: 950MB → 150MB (-84%)
- 起動時間: 8秒 → 2秒 (-75%)

## BuildKitとキャッシュ活用

### BuildKit有効化と最適化

```dockerfile
# syntax=docker/dockerfile:1

FROM node:20-alpine AS deps
WORKDIR /app

# ❌ 悪い例: 全ファイルコピー後に依存関係インストール
# COPY . .
# RUN npm ci

# ✅ 良い例: package.jsonのみコピー→キャッシュ活用
COPY package.json package-lock.json ./

# マウントキャッシュを使用
RUN --mount=type=cache,target=/root/.npm \
    npm ci

FROM node:20-alpine AS builder
WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

# ビルドキャッシュマウント
RUN --mount=type=cache,target=/app/.next/cache \
    npm run build

FROM node:20-alpine AS runner
WORKDIR /app

COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

CMD ["node", "server.js"]
```

**GitHub Actions でのBuildKit活用:**

```yaml
# .github/workflows/docker-buildkit.yml
name: Docker BuildKit

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Docker Buildx セットアップ
        uses: docker/setup-buildx-action@v3

      - name: ビルド(キャッシュ活用)
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: false
          tags: myapp:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILDKIT_INLINE_CACHE=1
```

**想定効果:**

| ビルド状況 | キャッシュなし | キャッシュあり | 削減率 |
|----------|-------------|-------------|--------|
| 初回ビルド | 8分 | 8分 | 0% |
| 依存関係変更なし | 8分 | 2分 | -75% |
| ソースコードのみ変更 | 8分 | 1分30秒 | -81% |

## レイヤーキャッシュの最適化

### .dockerignore の活用

```dockerfile
# .dockerignore
node_modules
npm-debug.log
.next
.git
.gitignore
README.md
.env*.local
.vscode
.idea
*.md
coverage
.turbo
dist
build
.DS_Store
```

**効果:**
- ビルドコンテキストサイズ: 500MB → 50MB (-90%)
- ビルド時間: 6分 → 4分 (-33%)

### レイヤー順序の最適化

```dockerfile
# ❌ 悪い例: 変更頻度が高いファイルを先にCOPY
FROM node:20-alpine
WORKDIR /app
COPY . .                      # 毎回キャッシュ無効化
RUN npm ci
RUN npm run build

# ✅ 良い例: 変更頻度が低いファイルを先にCOPY
FROM node:20-alpine
WORKDIR /app

# 1. package.json(変更頻度: 低)
COPY package.json package-lock.json ./
RUN npm ci

# 2. 設定ファイル(変更頻度: 中)
COPY tsconfig.json next.config.js ./

# 3. ソースコード(変更頻度: 高)
COPY src ./src
COPY public ./public

RUN npm run build
```

## セキュリティ強化

### 脆弱性スキャン統合

```yaml
# .github/workflows/docker-security.yml
name: Docker Security Scan

on: [push, pull_request]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Dockerイメージビルド
        run: docker build -t myapp:${{ github.sha }} .

      - name: Trivy 脆弱性スキャン
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

      - name: SARIF結果をGitHub Securityにアップロード
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

      - name: Dockle ベストプラクティスチェック
        uses: goodwithtech/dockle-action@main
        with:
          image: 'myapp:${{ github.sha }}'
          format: 'json'
          exit-code: '1'
          exit-level: 'warn'

      - name: Grype スキャン
        uses: anchore/scan-action@v3
        with:
          image: 'myapp:${{ github.sha }}'
          fail-build: true
          severity-cutoff: high
```

### セキュアなDockerfile

```dockerfile
# セキュリティベストプラクティス
FROM node:20-alpine AS runner

# 1. 非rootユーザーで実行
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 appuser

# 2. 必要最小限のパッケージのみ
RUN apk add --no-cache tini

# 3. 読み取り専用ファイルシステム(可能な場合)
WORKDIR /app

COPY --from=builder --chown=appuser:nodejs /app/dist ./dist
COPY --from=builder --chown=appuser:nodejs /app/node_modules ./node_modules

# 4. 非rootユーザーに切り替え
USER appuser

# 5. ヘルスチェック
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js

# 6. tini でプロセス管理
ENTRYPOINT ["/sbin/tini", "--"]

EXPOSE 3000
CMD ["node", "dist/server.js"]
```

## コンテナレジストリ統合

### GitHub Container Registry

```yaml
# .github/workflows/ghcr-push.yml
name: Push to GHCR

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Docker Buildx セットアップ
        uses: docker/setup-buildx-action@v3

      - name: GHCR ログイン
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: メタデータ抽出
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha

      - name: ビルドとプッシュ
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Amazon ECR

```yaml
# .github/workflows/ecr-push.yml
name: Push to ECR

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

      - name: AWS認証(OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: ap-northeast-1

      - name: ECRログイン
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2

      - name: Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: ビルドとプッシュ
        uses: docker/build-push-action@v5
        env:
          ECR_REGISTRY: ${{ steps.ecr-login.outputs.registry }}
          ECR_REPOSITORY: myapp
          IMAGE_TAG: ${{ github.sha }}
        with:
          context: .
          push: true
          tags: |
            ${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}
            ${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: イメージスキャン
        run: |
          aws ecr start-image-scan \
            --repository-name myapp \
            --image-id imageTag=${{ github.sha }}

          # スキャン結果待機
          aws ecr wait image-scan-complete \
            --repository-name myapp \
            --image-id imageTag=${{ github.sha }}

          # 結果確認
          aws ecr describe-image-scan-findings \
            --repository-name myapp \
            --image-id imageTag=${{ github.sha }}
```

## Docker Compose による統合テスト

### テスト環境構築

```yaml
# docker-compose.test.yml
version: '3.9'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      NODE_ENV: test
      DATABASE_URL: postgresql://test:test@db:5432/testdb
      REDIS_URL: redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    ports:
      - "3000:3000"

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: testdb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

```yaml
# .github/workflows/docker-integration-test.yml
name: Docker Integration Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: 統合テスト環境起動
        run: docker compose -f docker-compose.test.yml up -d

      - name: ヘルスチェック待機
        run: |
          timeout 60 bash -c 'until docker compose -f docker-compose.test.yml exec -T app wget -q -O- http://localhost:3000/health; do sleep 2; done'

      - name: 統合テスト実行
        run: docker compose -f docker-compose.test.yml exec -T app npm run test:integration

      - name: E2Eテスト実行
        run: docker compose -f docker-compose.test.yml exec -T app npm run test:e2e

      - name: ログ出力(失敗時)
        if: failure()
        run: docker compose -f docker-compose.test.yml logs

      - name: クリーンアップ
        if: always()
        run: docker compose -f docker-compose.test.yml down -v
```

## トラブルシューティング

### 問題1: ビルドが遅い

**症状:**
```
Dockerビルドに8分以上かかる
```

**対処法:**

```yaml
# 1. BuildKit有効化
- name: BuildKit有効化
  run: |
    export DOCKER_BUILDKIT=1
    docker build .

# 2. マルチステージビルド導入
# 3. .dockerignore設定
# 4. キャッシュマウント使用

- name: ビルド時間計測
  run: |
    time docker build \
      --build-arg BUILDKIT_INLINE_CACHE=1 \
      --cache-from myapp:cache \
      -t myapp:latest .
```

### 問題2: イメージサイズが大きい

**対処法:**

```bash
# イメージレイヤー分析
docker history myapp:latest

# dive でレイヤー詳細確認
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive:latest myapp:latest

# 不要ファイル削除
RUN npm ci && \
    npm cache clean --force && \
    rm -rf /tmp/*
```

### 問題3: 脆弱性が多い

**対処法:**

```dockerfile
# 1. Alpine ベースイメージ使用
FROM node:20-alpine

# 2. 定期的なベースイメージ更新
RUN apk update && apk upgrade

# 3. マルチステージで不要ツール削除
# ビルドツールは builder ステージのみ
```

## まとめ

この章では、Dockerコンテナの最適化手法を学びました:

✅ **マルチステージビルド**: イメージサイズ85%削減
✅ **BuildKitキャッシュ**: ビルド時間75%短縮
✅ **セキュリティ強化**: 脆弱性93%削減
✅ **レジストリ統合**: GHCR/ECR自動プッシュ
✅ **統合テスト**: Docker Composeによる環境構築

### 重要な想定される効果まとめ

| 最適化項目 | Before | After | 削減率 |
|----------|--------|-------|--------|
| イメージサイズ | 1.2GB | 180MB | -85% |
| ビルド時間 | 8分 | 2分 | -75% |
| デプロイ時間 | 12分 | 3分 | -75% |
| 脆弱性 | 42件 | 3件 | -93% |
| 月間コスト | $200 | $20 | -90% |

### Dockerfile最適化チェックリスト

- ✅ マルチステージビルド使用
- ✅ Alpine ベースイメージ
- ✅ .dockerignore 設定
- ✅ レイヤーキャッシュ最適化
- ✅ 非rootユーザー実行
- ✅ BuildKit キャッシュマウント
- ✅ セキュリティスキャン統合
- ✅ ヘルスチェック設定

### 全9章完了!

おめでとうございます! これで「CI/CD Automation Complete Guide 2026」の全章が完了しました。

本書で学んだ内容:
1. GitHub Actions 基礎とワークフロー
2. ワークフロー設計のベストプラクティス
3. 自動テストとカバレッジ
4. Fastlane完全ガイド(iOS自動化)
5. Web自動デプロイ(Vercel/Netlify/AWS)
6. セキュリティとシークレット管理
7. CI/CDパフォーマンス最適化
8. モノレポCI/CD戦略
9. Docker統合とコンテナビルド

これらの知識を活用し、世界クラスのCI/CDパイプラインを構築してください!

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
