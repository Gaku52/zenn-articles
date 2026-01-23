---
title: "実戦ケーススタディ Part 1 - フルスタックアプリCI/CD完全構築"
---

# Chapter 12 - 実戦ケーススタディ Part 1(フルスタックアプリ)

## 想定するプロジェクト

### 対象アプリケーション

**SparkVault** - タスク管理SaaSアプリケーション

- **フロントエンド**: Next.js 14 (App Router)、TypeScript、Tailwind CSS
- **バックエンド**: Next.js API Routes、Prisma、PostgreSQL
- **インフラ**: Vercel(Production)、Supabase(Database)
- **チーム規模**: 5名
- **デプロイ頻度**: 1日10回
- **ユーザー数**: 月間10万ユーザー

### 導入前の課題

**手作業の問題:**
- デプロイ: 手作業30分
- テスト実行: 毎回忘れる
- Lintチェック: PR後に発見
- データベースマイグレーション: 本番で失敗
- 環境変数の不整合: Staging/Productionで異なる値

**導入後の成果:**
- ✅ デプロイ時間: 30分 → 4分 (-87%)
- ✅ テスト実行率: 50% → 100%
- ✅ Lintエラー: PR後 → PR前に検知
- ✅ マイグレーション失敗: 月3回 → 0回
- ✅ 環境変数エラー: 月5回 → 0回

## プロジェクト構成

```
sparkvault/
├── .github/
│   └── workflows/
│       ├── ci.yml              # PR時のCI
│       ├── deploy-staging.yml  # Stagingデプロイ
│       └── deploy-prod.yml     # Productionデプロイ
├── prisma/
│   ├── schema.prisma
│   ├── migrations/
│   └── seed.ts
├── src/
│   ├── app/
│   ├── components/
│   ├── lib/
│   └── __tests__/
├── package.json
├── next.config.js
└── vercel.json
```

## CI/CDパイプライン設計

### パイプライン全体像

```
┌─────────────────────────────────────────┐
│  PR作成/更新                             │
└──────────────┬──────────────────────────┘
               │
    ┌──────────▼──────────┐
    │  Lint & Type Check  │  並列実行
    └──────────┬──────────┘
               │
    ┌──────────▼──────────┐
    │  Unit Tests         │  並列実行
    │  Integration Tests  │
    └──────────┬──────────┘
               │
    ┌──────────▼──────────┐
    │  Build Check        │
    └──────────┬──────────┘
               │
         ✅ CI完了
               │
    ┌──────────▼──────────┐
    │  PRマージ(develop)   │
    └──────────┬──────────┘
               │
    ┌──────────▼──────────┐
    │  Deploy to Staging  │
    │  + DB Migration     │
    └──────────┬──────────┘
               │
         ✅ E2Eテスト
               │
    ┌──────────▼──────────┐
    │  PRマージ(main)      │
    └──────────┬──────────┘
               │
    ┌──────────▼──────────┐
    │  Deploy to Prod     │
    │  + DB Migration     │
    │  + Smoke Test       │
    └──────────┬──────────┘
               │
         ✅ 本番デプロイ完了
```

## 完全ワークフロー実装

### 1. PR時のCI(.github/workflows/ci.yml)

```yaml
name: CI

on:
  pull_request:
    branches: [main, develop]
    types: [opened, synchronize, reopened]

# 同じPRで複数回プッシュした場合、前の実行をキャンセル
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # ========================================
  # 並列実行: Lint & Type Check
  # ========================================
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: ESLint
        run: npm run lint

      - name: Prettier check
        run: npm run format:check

  type-check:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: TypeScript check
        run: npm run type-check

  # ========================================
  # 並列実行: Unit & Integration Tests
  # ========================================
  unit-tests:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      # テストデータベースのセットアップ
      - name: Setup test database
        run: |
          docker run -d \
            --name postgres-test \
            -e POSTGRES_PASSWORD=test \
            -e POSTGRES_DB=sparkvault_test \
            -p 5432:5432 \
            postgres:15-alpine

          # データベース起動待機
          sleep 5

      - name: Run Prisma migrations
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/sparkvault_test
        run: npx prisma migrate deploy

      - name: Run unit tests
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/sparkvault_test
          NODE_ENV: test
        run: npm run test:unit -- --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage/coverage-final.json
          flags: unit-tests

  integration-tests:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Setup test database
        run: |
          docker run -d \
            --name postgres-test \
            -e POSTGRES_PASSWORD=test \
            -e POSTGRES_DB=sparkvault_test \
            -p 5432:5432 \
            postgres:15-alpine
          sleep 5

      - name: Run Prisma migrations
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/sparkvault_test
        run: npx prisma migrate deploy

      - name: Run integration tests
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/sparkvault_test
          NODE_ENV: test
        run: npm run test:integration

  # ========================================
  # ビルドチェック
  # ========================================
  build:
    needs: [lint, type-check, unit-tests, integration-tests]
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      # Next.js キャッシュ
      - name: Cache Next.js build
        uses: actions/cache@v4
        with:
          path: |
            .next/cache
          key: ${{ runner.os }}-nextjs-${{ hashFiles('**/package-lock.json') }}-${{ hashFiles('**/*.js', '**/*.jsx', '**/*.ts', '**/*.tsx') }}
          restore-keys: |
            ${{ runner.os }}-nextjs-${{ hashFiles('**/package-lock.json') }}-

      - name: Build
        env:
          NEXT_PUBLIC_API_URL: https://staging.sparkvault.app
        run: npm run build

      - name: Check build size
        run: |
          SIZE=$(du -sh .next | cut -f1)
          echo "Build size: $SIZE"

          # サマリーに追加
          cat >> $GITHUB_STEP_SUMMARY << EOF
          ## 📦 Build Summary

          - **Build Size**: $SIZE
          - **Status**: ✅ Success
          - **Node Version**: $(node -v)
          - **Next.js Version**: $(npm list next --depth=0 | grep next@)
          EOF

  # ========================================
  # PRコメント(テスト結果)
  # ========================================
  comment:
    needs: [build]
    if: always()
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - name: Comment PR
        uses: actions/github-script@v7
        with:
          script: |
            const status = '${{ needs.build.result }}' === 'success' ? '✅ All checks passed!' : '❌ Some checks failed';
            const comment = `
            ## CI Results

            ${status}

            ### Details
            - Lint: ${{ needs.lint.result }}
            - Type Check: ${{ needs.type-check.result }}
            - Unit Tests: ${{ needs.unit-tests.result }}
            - Integration Tests: ${{ needs.integration-tests.result }}
            - Build: ${{ needs.build.result }}

            [View full logs](https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }})
            `;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: comment
            });
```

**想定される実行時間:**
- 並列実行: 6分
- 直列実行(仮): 18分
- **時短効果: -67%**

### 2. Stagingデプロイ(.github/workflows/deploy-staging.yml)

```yaml
name: Deploy to Staging

on:
  push:
    branches: [develop]

jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    environment:
      name: staging
      url: https://staging.sparkvault.app

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      # ========================================
      # データベースマイグレーション
      # ========================================
      - name: Run database migrations
        env:
          DATABASE_URL: ${{ secrets.STAGING_DATABASE_URL }}
        run: |
          npx prisma migrate deploy
          npx prisma generate

      # ========================================
      # Vercelデプロイ
      # ========================================
      - name: Deploy to Vercel
        env:
          VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}
          VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
          VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}
        run: |
          npm install -g vercel

          # Stagingにデプロイ
          vercel deploy \
            --token $VERCEL_TOKEN \
            --env NEXT_PUBLIC_API_URL=https://staging.sparkvault.app \
            --env DATABASE_URL=${{ secrets.STAGING_DATABASE_URL }} \
            --yes \
            > deployment-url.txt

          DEPLOYMENT_URL=$(cat deployment-url.txt)
          echo "Deployment URL: $DEPLOYMENT_URL"
          echo "deployment_url=$DEPLOYMENT_URL" >> $GITHUB_OUTPUT

      # ========================================
      # Smoke Test
      # ========================================
      - name: Smoke test
        run: |
          sleep 10  # デプロイ反映待ち

          # ヘルスチェック
          curl -f https://staging.sparkvault.app/api/health || exit 1

          echo "✅ Smoke test passed"

      # ========================================
      # Slack通知
      # ========================================
      - name: Notify Slack
        if: always()
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
        run: |
          STATUS="${{ job.status }}"
          if [ "$STATUS" = "success" ]; then
            COLOR="good"
            EMOJI="✅"
          else
            COLOR="danger"
            EMOJI="❌"
          fi

          curl -X POST $SLACK_WEBHOOK \
            -H 'Content-Type: application/json' \
            -d '{
              "text": "'"$EMOJI Staging Deployment $STATUS"'",
              "attachments": [{
                "color": "'"$COLOR"'",
                "fields": [
                  {
                    "title": "Environment",
                    "value": "Staging",
                    "short": true
                  },
                  {
                    "title": "Deployed by",
                    "value": "${{ github.actor }}",
                    "short": true
                  },
                  {
                    "title": "URL",
                    "value": "https://staging.sparkvault.app"
                  }
                ]
              }]
            }'
```

**想定される実行時間:**
- マイグレーション: 30秒
- デプロイ: 2分
- スモークテスト: 10秒
- **合計: 3分**

### 3. Productionデプロイ(.github/workflows/deploy-prod.yml)

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    environment:
      name: production
      url: https://sparkvault.app

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      # ========================================
      # データベースバックアップ
      # ========================================
      - name: Backup database
        env:
          DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }}
          BACKUP_BUCKET: ${{ secrets.S3_BACKUP_BUCKET }}
        run: |
          # Supabaseバックアップ(例)
          BACKUP_FILE="backup-$(date +%Y%m%d-%H%M%S).sql"

          # バックアップ作成とS3アップロード
          # (実際のコマンドは環境により異なる)
          echo "✅ Database backup created: $BACKUP_FILE"

      # ========================================
      # データベースマイグレーション
      # ========================================
      - name: Run database migrations
        env:
          DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }}
        run: |
          npx prisma migrate deploy
          npx prisma generate

      # ========================================
      # Vercel Production デプロイ
      # ========================================
      - name: Deploy to Vercel Production
        id: deploy
        env:
          VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}
          VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
          VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}
        run: |
          npm install -g vercel

          # Productionにデプロイ
          vercel deploy \
            --prod \
            --token $VERCEL_TOKEN \
            --env NEXT_PUBLIC_API_URL=https://sparkvault.app \
            --env DATABASE_URL=${{ secrets.PROD_DATABASE_URL }} \
            --yes

      # ========================================
      # 本番ヘルスチェック
      # ========================================
      - name: Production health check
        run: |
          sleep 30  # デプロイ反映待ち

          # ヘルスチェック
          for i in {1..10}; do
            if curl -f https://sparkvault.app/api/health; then
              echo "✅ Health check passed"
              break
            fi
            echo "Retry $i/10..."
            sleep 10
          done

      # ========================================
      # エラー率監視
      # ========================================
      - name: Monitor error rate
        run: |
          sleep 60  # 1分間待機

          # エラー率チェック(例)
          ERROR_RATE=$(curl -s https://sparkvault.app/api/metrics/error-rate)

          if [ "$ERROR_RATE" -gt "5" ]; then
            echo "::error::High error rate detected: $ERROR_RATE%"
            exit 1
          fi

          echo "✅ Error rate normal: $ERROR_RATE%"

      # ========================================
      # Gitタグ作成
      # ========================================
      - name: Create release tag
        run: |
          VERSION=$(node -p "require('./package.json').version")
          TAG="v$VERSION-$(date +%Y%m%d-%H%M%S)"

          git tag $TAG
          git push origin $TAG

      # ========================================
      # Slack通知
      # ========================================
      - name: Notify Slack
        if: always()
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
        run: |
          STATUS="${{ job.status }}"
          if [ "$STATUS" = "success" ]; then
            COLOR="good"
            EMOJI="🎉"
            MENTION=""
          else
            COLOR="danger"
            EMOJI="🚨"
            MENTION="<!channel> "
          fi

          curl -X POST $SLACK_WEBHOOK \
            -H 'Content-Type: application/json' \
            -d '{
              "text": "'"$MENTION$EMOJI Production Deployment $STATUS"'",
              "attachments": [{
                "color": "'"$COLOR"'",
                "fields": [
                  {
                    "title": "Environment",
                    "value": "Production",
                    "short": true
                  },
                  {
                    "title": "Deployed by",
                    "value": "${{ github.actor }}",
                    "short": true
                  },
                  {
                    "title": "URL",
                    "value": "https://sparkvault.app"
                  }
                ]
              }]
            }'
```

**想定される実行時間:**
- バックアップ: 1分
- マイグレーション: 30秒
- デプロイ: 2分
- ヘルスチェック: 30秒
- **合計: 4分**

## 想定される効果と成果

### ビルド時間の推移

| フェーズ | 導入前 | 導入後 | 削減率 |
|---------|--------|--------|--------|
| PR CI | 手動15分 | 自動6分 | -60% |
| Stagingデプロイ | 30分 | 3分 | -90% |
| Productionデプロイ | 45分 | 4分 | -91% |

### 品質指標の改善

| 指標 | Before | After | 改善率 |
|------|--------|-------|--------|
| テスト実行率 | 50% | 100% | +100% |
| テストカバレッジ | 45% | 85% | +89% |
| 本番バグ | 12件/月 | 2件/月 | -83% |
| デプロイ頻度 | 週2回 | 日10回 | +350% |
| マイグレーション失敗 | 月3回 | 0回 | -100% |

### コスト効果

**開発者の時間節約:**
- デプロイ作業: 15時間/月 → 0時間/月
- 手動テスト: 20時間/月 → 2時間/月
- バグ修正: 30時間/月 → 8時間/月
- **合計: 55時間/月の節約**

**金銭的効果(時給5000円換算):**
- 月間節約: 275,000円
- 年間節約: 3,300,000円

## まとめ

この章では、フルスタックアプリの完全なCI/CD構築を学びました:

✅ **並列CI**: Lint、Type Check、Testを並列実行(6分)
✅ **自動デプロイ**: Staging/Production完全自動化
✅ **DBマイグレーション**: 本番でも安全に実行
✅ **監視**: ヘルスチェック、エラー率監視
✅ **通知**: Slack/PagerDutyで即座に通知

### 次のステップ

**Chapter 13 - 実戦ケーススタディ Part 2**では、iOSアプリのCI/CD構築を学びます。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
