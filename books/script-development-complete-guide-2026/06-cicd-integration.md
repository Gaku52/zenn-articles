---
title: "CI/CD統合の実践"
---

# CI/CD統合の実践

CI/CDパイプラインとスクリプトを連携させることで、開発から本番環境へのデプロイメントを自動化できます。本章では、GitHub ActionsとGitLab CIとの統合方法を学びます。

## GitHub Actions 統合

### 基本ワークフロー

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  NODE_VERSION: '18'

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build application
        run: npm run build

      - name: Deploy to production
        env:
          DEPLOY_HOST: ${{ secrets.DEPLOY_HOST }}
          DEPLOY_USER: ${{ secrets.DEPLOY_USER }}
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
        run: |
          ./scripts/deploy.sh production

      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployment ${{ job.status }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### デプロイスクリプト

```bash
#!/usr/bin/env bash
# scripts/deploy.sh - GitHub Actions用デプロイスクリプト

set -euo pipefail

readonly ENVIRONMENT="${1:?Environment required}"

# CI環境の検出
is_ci() {
    [[ "${CI:-false}" == "true" ]]
}

# GitHub環境変数の検証
validate_ci_environment() {
    local required_vars=(
        "GITHUB_REPOSITORY"
        "GITHUB_SHA"
        "GITHUB_REF"
    )

    for var in "${required_vars[@]}"; do
        if [[ -z "${!var:-}" ]]; then
            echo "Error: Required environment variable ${var} is not set" >&2
            return 1
        fi
    done
}

# SSHキーのセットアップ
setup_ssh() {
    if [[ -z "${SSH_PRIVATE_KEY:-}" ]]; then
        return 0
    fi

    mkdir -p ~/.ssh
    echo "${SSH_PRIVATE_KEY}" > ~/.ssh/id_rsa
    chmod 600 ~/.ssh/id_rsa

    # ホストキーの検証をスキップ（CI環境のみ）
    if is_ci; then
        cat >> ~/.ssh/config <<EOF
Host *
    StrictHostKeyChecking no
    UserKnownHostsFile=/dev/null
EOF
    fi
}

# デプロイメントステータスの作成
create_deployment_status() {
    local state="${1}"  # success, failure
    local description="${2}"

    if ! is_ci; then
        return 0
    fi

    echo "Deployment ${state}: ${description}"

    # GitHub APIを使用した通知（オプション）
    if [[ -n "${GITHUB_TOKEN:-}" ]]; then
        curl -X POST \
            -H "Authorization: token ${GITHUB_TOKEN}" \
            -H "Accept: application/vnd.github.v3+json" \
            "https://api.github.com/repos/${GITHUB_REPOSITORY}/deployments" \
            -d "{
                \"ref\": \"${GITHUB_SHA}\",
                \"environment\": \"${ENVIRONMENT}\",
                \"description\": \"${description}\"
            }"
    fi
}

# メインデプロイ処理
deploy() {
    if is_ci; then
        validate_ci_environment || return 1
        setup_ssh
    fi

    echo "Deploying to ${ENVIRONMENT}"

    create_deployment_status "pending" "Deployment in progress"

    # デプロイ実行
    if ./scripts/deploy-to-server.sh "${ENVIRONMENT}"; then
        create_deployment_status "success" "Deployment completed"
        return 0
    else
        create_deployment_status "failure" "Deployment failed"
        return 1
    fi
}

deploy
```

### 再利用可能なワークフロー

```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy Workflow

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      node-version:
        required: false
        type: string
        default: '18'
    secrets:
      deploy-host:
        required: true
      deploy-user:
        required: true
      ssh-key:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'

      - name: Install and build
        run: |
          npm ci
          npm run build

      - name: Deploy
        env:
          DEPLOY_HOST: ${{ secrets.deploy-host }}
          DEPLOY_USER: ${{ secrets.deploy-user }}
          SSH_PRIVATE_KEY: ${{ secrets.ssh-key }}
        run: ./scripts/deploy.sh ${{ inputs.environment }}
```

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy to Staging

on:
  push:
    branches: [develop]

jobs:
  deploy:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
    secrets:
      deploy-host: ${{ secrets.STAGING_HOST }}
      deploy-user: ${{ secrets.STAGING_USER }}
      ssh-key: ${{ secrets.STAGING_SSH_KEY }}
```

## GitLab CI 統合

### 基本パイプライン

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  NODE_VERSION: "18"

# キャッシュ設定
cache:
  paths:
    - node_modules/
    - .npm/

before_script:
  - apt-get update -qq
  - apt-get install -y -qq openssh-client

build:
  stage: build
  image: node:${NODE_VERSION}
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour

test:
  stage: test
  image: node:${NODE_VERSION}
  script:
    - npm ci
    - npm run test
    - npm run lint
  coverage: '/Coverage: \d+\.\d+%/'

deploy:staging:
  stage: deploy
  image: node:${NODE_VERSION}
  script:
    - ./scripts/gitlab-deploy.sh staging
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop

deploy:production:
  stage: deploy
  image: node:${NODE_VERSION}
  script:
    - ./scripts/gitlab-deploy.sh production
  environment:
    name: production
    url: https://example.com
    on_stop: stop_production
  only:
    - main
  when: manual

stop_production:
  stage: deploy
  script:
    - ./scripts/stop-environment.sh production
  when: manual
  environment:
    name: production
    action: stop
```

### GitLab CI用デプロイスクリプト

```bash
#!/usr/bin/env bash
# scripts/gitlab-deploy.sh

set -euo pipefail

readonly ENVIRONMENT="${1:?Environment required}"

# GitLab環境変数の検証
validate_gitlab_environment() {
    local required_vars=(
        "CI_PROJECT_NAME"
        "CI_COMMIT_SHA"
        "CI_COMMIT_REF_NAME"
    )

    for var in "${required_vars[@]}"; do
        if [[ -z "${!var:-}" ]]; then
            echo "Error: Required GitLab variable ${var} is not set" >&2
            return 1
        fi
    done
}

# GitLab環境URLの設定
set_environment_url() {
    local url="${1}"

    # environment_url.txt に書き込むとGitLabが読み取る
    echo "${url}" > environment_url.txt
}

# GitLabへのメトリクス送信
send_gitlab_metrics() {
    local metric_name="${1}"
    local value="${2}"

    # metrics.txt に書き込むとGitLabが読み取る
    echo "${metric_name} ${value}" >> metrics.txt
}

# SSHセットアップ
setup_ssh() {
    mkdir -p ~/.ssh
    chmod 700 ~/.ssh

    if [[ -n "${SSH_PRIVATE_KEY:-}" ]]; then
        echo "${SSH_PRIVATE_KEY}" | tr -d '\r' > ~/.ssh/id_rsa
        chmod 600 ~/.ssh/id_rsa
    fi

    # SSH設定
    cat >> ~/.ssh/config <<EOF
Host *
    StrictHostKeyChecking no
    UserKnownHostsFile=/dev/null
EOF
}

# デプロイ実行
deploy() {
    validate_gitlab_environment || return 1
    setup_ssh

    echo "Deploying ${CI_PROJECT_NAME} to ${ENVIRONMENT}"
    echo "Commit: ${CI_COMMIT_SHA:0:8}"
    echo "Branch: ${CI_COMMIT_REF_NAME}"

    local start_time
    start_time="$(date +%s)"

    # デプロイ実行
    if ./scripts/deploy-to-server.sh "${ENVIRONMENT}"; then
        local end_time duration
        end_time="$(date +%s)"
        duration=$((end_time - start_time))

        # 環境URLの設定
        case "${ENVIRONMENT}" in
            production)
                set_environment_url "https://example.com"
                ;;
            staging)
                set_environment_url "https://staging.example.com"
                ;;
        esac

        # メトリクスの送信
        send_gitlab_metrics "deployment_duration" "${duration}"

        echo "Deployment completed in ${duration}s"
        return 0
    else
        echo "Deployment failed"
        return 1
    fi
}

deploy
```

## Docker統合

### Dockerビルドとデプロイ

```yaml
# .github/workflows/docker.yml
name: Docker Build and Deploy

on:
  push:
    branches: [main]

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
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Deploy to server
        env:
          DEPLOY_HOST: ${{ secrets.DEPLOY_HOST }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          ./scripts/deploy-docker.sh
```

### Dockerデプロイスクリプト

```bash
#!/usr/bin/env bash
# scripts/deploy-docker.sh

set -euo pipefail

readonly IMAGE_NAME="${REGISTRY}/${IMAGE_NAME}"
readonly IMAGE_TAG="${GITHUB_SHA}"

deploy_docker() {
    echo "Deploying Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"

    # SSHでリモートサーバーに接続してデプロイ
    ssh "${DEPLOY_USER}@${DEPLOY_HOST}" bash <<EOF
set -euo pipefail

# イメージのプル
docker pull ${IMAGE_NAME}:${IMAGE_TAG}

# 既存コンテナの停止
docker-compose down

# 新しいイメージで起動
export IMAGE_TAG=${IMAGE_TAG}
docker-compose up -d

# ヘルスチェック
for i in {1..30}; do
    if curl -sf http://localhost/health > /dev/null; then
        echo "Health check passed"
        exit 0
    fi
    sleep 2
done

echo "Health check failed"
exit 1
EOF
}

deploy_docker
```

## テスト自動化

### テストワークフロー

```yaml
# .github/workflows/test.yml
name: Test

on:
  pull_request:
  push:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [16, 18, 20]

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests
        run: npm test -- --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
```

## セキュリティチェック

### セキュリティスキャン

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 0 * * 0'  # 毎週日曜日

jobs:
  dependency-check:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Run npm audit
        run: npm audit --production

      - name: Check for outdated dependencies
        run: npm outdated || true

  secret-scan:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Secret Scanning
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
```

## 通知とレポート

### Slack通知

```yaml
# .github/workflows/notify.yml
name: Notify

on:
  workflow_run:
    workflows: ["Deploy"]
    types: [completed]

jobs:
  notify:
    runs-on: ubuntu-latest

    steps:
      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployment ${{ github.event.workflow_run.conclusion }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Deployment Result*: ${{ github.event.workflow_run.conclusion }}\n*Repository*: ${{ github.repository }}\n*Commit*: ${{ github.sha }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

## まとめ

本章では、CI/CD統合の実践を学びました：

- **GitHub Actions**: ワークフロー設計、デプロイスクリプト
- **GitLab CI**: パイプライン設定、環境変数活用
- **Docker統合**: イメージビルド、コンテナデプロイ
- **自動化**: テスト、セキュリティスキャン、通知

次章では、スクリプト開発のベストプラクティスをまとめます。
