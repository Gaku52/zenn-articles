# CI/CD統合

本章では、CLIツールとスクリプトをCI/CDパイプラインに統合する方法を学びます。CI/CD統合により、開発からデプロイまでの自動化を実現し、開発効率と品質を大幅に向上できます。

## GitHub Actions統合

GitHub Actionsは、GitHubが提供する強力なCI/CDプラットフォームです。

### 基本的なワークフロー

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build
```

### CLIツール実行

```yaml
# .github/workflows/cli-automation.yml
name: CLI Automation

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - development
          - staging
          - production

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install CLI dependencies
        run: npm install -g @myorg/deploy-cli

      - name: Run deployment CLI
        env:
          DEPLOY_ENV: ${{ github.event.inputs.environment || 'development' }}
          API_KEY: ${{ secrets.API_KEY }}
        run: |
          deploy-cli deploy \
            --env $DEPLOY_ENV \
            --version ${{ github.sha }}

      - name: Notify deployment
        if: success()
        run: |
          echo "Deployment to $DEPLOY_ENV completed successfully"
```

### マトリックスビルド

```yaml
# .github/workflows/matrix.yml
name: Matrix Build

on:
  push:
    branches: [main]

jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node-version: [16, 18, 20]

    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Test CLI
        run: |
          npm link
          mycli --version
          mycli test-command
```

### スクリプト実行

```yaml
# .github/workflows/scripts.yml
name: Run Scripts

on:
  schedule:
    # 毎日午前2時（UTC）に実行
    - cron: '0 2 * * *'
  workflow_dispatch:

jobs:
  backup:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run backup script
        env:
          DB_HOST: ${{ secrets.DB_HOST }}
          DB_USER: ${{ secrets.DB_USER }}
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: python scripts/backup.py

      - name: Upload backup artifact
        uses: actions/upload-artifact@v4
        with:
          name: backup-${{ github.run_id }}
          path: backups/
          retention-days: 7
```

### 環境別デプロイメント

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches:
      - main
      - develop
      - 'release/**'

jobs:
  determine-environment:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.set-env.outputs.environment }}

    steps:
      - name: Determine environment
        id: set-env
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "environment=production" >> $GITHUB_OUTPUT
          elif [[ "${{ github.ref }}" == "refs/heads/develop" ]]; then
            echo "environment=staging" >> $GITHUB_OUTPUT
          else
            echo "environment=development" >> $GITHUB_OUTPUT
          fi

  deploy:
    needs: determine-environment
    runs-on: ubuntu-latest
    environment: ${{ needs.determine-environment.outputs.environment }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Build
        env:
          NODE_ENV: ${{ needs.determine-environment.outputs.environment }}
        run: npm run build

      - name: Deploy
        env:
          DEPLOY_ENV: ${{ needs.determine-environment.outputs.environment }}
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
        run: |
          mkdir -p ~/.ssh
          echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ./scripts/deploy.sh $DEPLOY_ENV
```

## GitLab CI統合

GitLab CIは、GitLabに統合されたCI/CDツールです。

### 基本的なパイプライン

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

variables:
  NODE_VERSION: "18"

# テンプレート定義
.node_template: &node_template
  image: node:${NODE_VERSION}
  before_script:
    - npm ci
  cache:
    paths:
      - node_modules/

# テスト
test:lint:
  <<: *node_template
  stage: test
  script:
    - npm run lint

test:unit:
  <<: *node_template
  stage: test
  script:
    - npm test
  coverage: '/Lines\s*:\s*(\d+\.\d+)%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

# ビルド
build:
  <<: *node_template
  stage: build
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 week

# デプロイ
deploy:staging:
  stage: deploy
  script:
    - ./scripts/deploy.sh staging
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop

deploy:production:
  stage: deploy
  script:
    - ./scripts/deploy.sh production
  environment:
    name: production
    url: https://example.com
  only:
    - main
  when: manual  # 手動承認
```

### CLIツール実行

```yaml
# .gitlab-ci.yml
stages:
  - setup
  - automation

setup:cli:
  stage: setup
  image: node:18
  script:
    - npm install -g @myorg/deploy-cli
  artifacts:
    paths:
      - /usr/local/lib/node_modules/@myorg/deploy-cli
    expire_in: 1 hour

run:automation:
  stage: automation
  dependencies:
    - setup:cli
  script:
    - deploy-cli deploy --env $CI_ENVIRONMENT_NAME
  variables:
    CI_ENVIRONMENT_NAME: staging
  environment:
    name: staging
```

### スクリプト実行とスケジューリング

```yaml
# .gitlab-ci.yml
stages:
  - backup
  - notify

# バックアップジョブ
backup:database:
  stage: backup
  image: python:3.11
  before_script:
    - pip install -r requirements.txt
  script:
    - python scripts/backup.py
  artifacts:
    paths:
      - backups/
    expire_in: 30 days
  only:
    - schedules  # スケジュール実行のみ

# 通知ジョブ
notify:success:
  stage: notify
  script:
    - |
      curl -X POST $SLACK_WEBHOOK \
        -H 'Content-Type: application/json' \
        -d "{\"text\": \"Backup completed successfully\"}"
  when: on_success
  only:
    - schedules

notify:failure:
  stage: notify
  script:
    - |
      curl -X POST $SLACK_WEBHOOK \
        -H 'Content-Type: application/json' \
        -d "{\"text\": \"Backup failed!\"}"
  when: on_failure
  only:
    - schedules
```

## 環境変数・シークレット管理

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy with Secrets

on:
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      # リポジトリシークレット使用
      - name: Deploy with secrets
        env:
          API_KEY: ${{ secrets.API_KEY }}
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: ./scripts/deploy.sh

      # 環境変数ファイル作成
      - name: Create .env file
        run: |
          cat << EOF > .env
          NODE_ENV=production
          API_URL=${{ secrets.API_URL }}
          API_KEY=${{ secrets.API_KEY }}
          EOF

      - name: Deploy
        run: ./scripts/deploy.sh
```

### GitLab CI

```yaml
# .gitlab-ci.yml
deploy:
  stage: deploy
  script:
    - |
      cat << EOF > .env
      NODE_ENV=production
      API_URL=$API_URL
      API_KEY=$API_KEY
      EOF
    - ./scripts/deploy.sh
  variables:
    # プロジェクト変数として定義
    NODE_ENV: production
```

## アーティファクトの保存

### GitHub Actions

```yaml
# .github/workflows/artifacts.yml
name: Build and Archive

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Build
        run: |
          npm ci
          npm run build

      # ビルド成果物の保存
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist-${{ github.sha }}
          path: |
            dist/
            !dist/**/*.map
          retention-days: 14

      # テストレポートの保存
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: test-results/

      # カバレッジレポートの保存
      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/

  deploy:
    needs: build
    runs-on: ubuntu-latest

    steps:
      # アーティファクトのダウンロード
      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist-${{ github.sha }}
          path: dist/

      - name: Deploy
        run: |
          ls -la dist/
          # デプロイ処理
```

### GitLab CI

```yaml
# .gitlab-ci.yml
build:
  stage: build
  script:
    - npm ci
    - npm run build
  artifacts:
    name: "$CI_JOB_NAME-$CI_COMMIT_REF_SLUG"
    paths:
      - dist/
    exclude:
      - dist/**/*.map
    expire_in: 2 weeks

test:
  stage: test
  script:
    - npm test
  artifacts:
    when: always
    reports:
      junit: test-results/junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

deploy:
  stage: deploy
  dependencies:
    - build
  script:
    - ls -la dist/
    - ./scripts/deploy.sh
```

## 実践的なCI/CD設定例

### モノレポ対応

```yaml
# .github/workflows/monorepo.yml
name: Monorepo CI/CD

on:
  push:
    branches: [main]

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      cli: ${{ steps.filter.outputs.cli }}
      api: ${{ steps.filter.outputs.api }}
      web: ${{ steps.filter.outputs.web }}

    steps:
      - uses: actions/checkout@v4

      - uses: dorny/paths-filter@v2
        id: filter
        with:
          filters: |
            cli:
              - 'packages/cli/**'
            api:
              - 'packages/api/**'
            web:
              - 'packages/web/**'

  test-cli:
    needs: detect-changes
    if: needs.detect-changes.outputs.cli == 'true'
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Test CLI
        working-directory: packages/cli
        run: |
          npm ci
          npm test

  test-api:
    needs: detect-changes
    if: needs.detect-changes.outputs.api == 'true'
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Test API
        working-directory: packages/api
        run: |
          npm ci
          npm test

  test-web:
    needs: detect-changes
    if: needs.detect-changes.outputs.web == 'true'
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Test Web
        working-directory: packages/web
        run: |
          npm ci
          npm test
```

### Docker統合

```yaml
# .github/workflows/docker.yml
name: Docker Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            myorg/myapp:latest
            myorg/myapp:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Deploy to server
        env:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
        run: |
          mkdir -p ~/.ssh
          echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa

          ssh user@server << 'EOF'
            docker pull myorg/myapp:${{ github.sha }}
            docker stop myapp || true
            docker rm myapp || true
            docker run -d \
              --name myapp \
              -p 3000:3000 \
              myorg/myapp:${{ github.sha }}
          EOF
```

## CI/CDチェックリスト

### ワークフロー設計
- [ ] 適切なトリガー（push、PR、schedule）を設定している
- [ ] 必要最小限のジョブに分割している
- [ ] 並列実行を活用している
- [ ] キャッシュを適切に使用している

### セキュリティ
- [ ] シークレットを環境変数で管理している
- [ ] SSH鍵の権限を適切に設定している
- [ ] API キーをハードコードしていない
- [ ] アーティファクトの保存期間を設定している

### テスト
- [ ] リンター、テストを実行している
- [ ] テスト結果をアーティファクトに保存している
- [ ] カバレッジレポートを生成している

### デプロイ
- [ ] 環境別のデプロイ設定を分離している
- [ ] 本番デプロイに手動承認を設定している
- [ ] デプロイ後のヘルスチェックを実施している
- [ ] ロールバック手順を用意している

### 通知
- [ ] デプロイ成功・失敗を通知している
- [ ] Slack、メールなど適切な通知手段を使用している

## まとめ

本章では、CLIツールとスクリプトのCI/CD統合を学びました。

**重要ポイント**:
- GitHub Actions、GitLab CIによる自動化
- 環境変数・シークレットの安全な管理
- アーティファクトの適切な保存と活用
- マトリックスビルド、モノレポ対応
- Docker統合による一貫したデプロイメント

CI/CD統合により、開発からデプロイまでの自動化を実現し、開発効率と品質を大幅に向上できます。

## 参考文献

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [Docker Documentation](https://docs.docker.com/)
- [Workflow syntax for GitHub Actions](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
