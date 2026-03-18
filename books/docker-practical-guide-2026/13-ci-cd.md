---
title: "CI/CDとDocker"
---

## この章で学ぶこと

コードをプッシュしたら自動でDockerイメージがビルドされ、テストされ、本番環境にデプロイされる。この一連の流れを**CI/CD（継続的インテグレーション / 継続的デリバリー）パイプライン**と呼びます。

この章では、GitHub Actionsを中心に以下を習得します。

- GitHub ActionsによるDockerイメージの自動ビルドとレジストリへのプッシュ
- CI上での自動テストとセキュリティスキャンの統合
- ビルド高速化のためのキャッシュ戦略
- GitOpsによるデプロイの自動化

## CI/CDパイプラインの全体像

Docker CI/CDパイプラインは、大きく4つのステージで構成されます。

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Code    │    │  Build   │    │  Test    │    │  Deploy  │
│  Push    │───▶│  Image   │───▶│  Scan   │───▶│  Release │
│          │    │          │    │  Verify  │    │          │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
     │               │               │               │
     ▼               ▼               ▼               ▼
  git push      docker build    trivy scan     docker push
  PR作成        multi-stage     unit test      kubectl apply
  tag作成       layer cache     integration    docker compose
```

前のステージが成功した場合のみ次に進む「ゲート」の仕組みにより、問題のあるイメージが本番に到達することを防ぎます。

## GitHub ActionsでのDockerビルド

### 基本的なワークフロー

Docker公式が提供するActionを組み合わせることで、簡潔にビルドパイプラインを構築できます。

```yaml
# .github/workflows/docker-build.yml
name: Docker Build and Push

on:
  push:
    branches: [main, develop]
    tags: ["v*"]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

このワークフローで使っている4つのActionの役割を整理します。

- **`setup-buildx-action`**: BuildKitベースのビルダーをセットアップします。キャッシュ機能に必須です
- **`login-action`**: コンテナレジストリにログインします。PRの場合はスキップします
- **`metadata-action`**: ブランチ名、セマンティックバージョン、Git SHAなどからタグを自動生成します
- **`build-push-action`**: ビルドとプッシュを一括で実行します

### タグ戦略

イメージのタグは「どのコードから作られたか」を追跡するための鍵です。

```
git push main
    └──▶ ghcr.io/user/app:main, :sha-abc1234, :latest

git tag v1.2.3
    └──▶ ghcr.io/user/app:1.2.3, :1.2, :1, :latest
```

**重要**: `latest`タグだけに依存してはいけません。`latest`は上書きされるため、本番で「どのバージョンが動いているか」がわからなくなります。必ずセマンティックバージョンやGit SHAのタグを併用してください。

## レジストリへのプッシュ

プッシュ先のレジストリは用途に応じて選びます。

| レジストリ | 認証方法 | 特徴 |
|-----------|---------|------|
| GHCR | `GITHUB_TOKEN`（追加設定不要） | GitHub Actionsとの連携が最もスムーズ |
| AWS ECR | OIDC連携 | AWS環境向き。長期アクセスキー不要 |
| Docker Hub | ユーザー名 + トークン | パブリックイメージの配布向き。プルレート制限あり |

GHCRを使う場合、上のワークフロー例のようにログインするだけでプッシュできます。ECRの場合は`aws-actions/configure-aws-credentials`と`aws-actions/amazon-ecr-login`を追加します。

## CI上での自動テスト

### Docker Composeによるテスト環境

Docker Composeを使えば、データベースなどの依存サービスも含めたテスト環境をCI上に再現できます。

```yaml
# docker-compose.test.yml
services:
  test:
    build:
      context: .
      target: test  # マルチステージビルドのテスト用ステージ
    volumes:
      - ./coverage:/app/coverage
    environment:
      NODE_ENV: test
      DATABASE_URL: postgres://test:test@db:5432/testdb

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: testdb
    tmpfs:
      - /var/lib/postgresql/data  # メモリ上で高速実行
```

ワークフローからは以下のように呼び出します。

```yaml
- name: Run unit tests in Docker
  run: |
    docker compose -f docker-compose.test.yml run --rm \
      --build test npm run test:ci
```

### Dockerfileのリントとセキュリティスキャン

Hadolintでベストプラクティス違反を検出し、Trivyでイメージの脆弱性をスキャンします。

```yaml
- name: Lint Dockerfile
  uses: hadolint/hadolint-action@v3.1.0
  with:
    dockerfile: Dockerfile
    failure-threshold: warning

- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
    format: "sarif"
    output: "trivy-results.sarif"
    severity: "CRITICAL,HIGH"
    exit-code: "1"  # 脆弱性が見つかったらパイプラインを失敗させる
```

SARIF形式で出力すると、GitHubのSecurity Alertsタブにスキャン結果が表示されます。

## キャッシュ戦略

Dockerビルドの最大のボトルネックは依存関係のインストールです。キャッシュを適切に設計すれば、ビルド時間を数分から数十秒に短縮できます。

### GitHub Actions Cache（推奨）

`build-push-action`に2行追加するだけで有効になります。

```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

`mode=max`を指定すると、最終ステージだけでなく全ステージのレイヤーキャッシュが保存されます。

### キャッシュ方式の比較

| 方式 | 速度 | 容量制限 | CI間共有 | 設定の簡便さ |
|------|------|---------|---------|-------------|
| GHA Cache | 高速 | 10GB | 同一リポジトリ | 最も簡単 |
| Registry Cache | 中速 | 無制限 | 全環境 | 中程度 |
| Local Cache | 最速 | ディスク依存 | 不可 | 簡単 |

### Dockerfileのキャッシュ最適化

CI側の設定だけでなく、Dockerfile自体もキャッシュを意識して書くことが重要です。**変更頻度の低いものを上に、高いものを下に**配置します。

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app

# 1. ロックファイルだけ先にコピー（変更頻度: 低）
COPY package.json package-lock.json ./

# 2. 依存関係インストール（最も遅いステップ）
#    → ロックファイルが変わらない限りキャッシュが効く
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# 3. ソースコードをコピー（変更頻度: 高）
COPY src/ ./src/

# 4. ビルド
RUN npm run build
```

`--mount=type=cache`はBuildKitのキャッシュマウント機能です。パッケージマネージャーのダウンロードキャッシュをビルド間で共有し、再ダウンロードを防ぎます。

## GitOps によるデプロイ自動化

GitOpsは「Gitリポジトリの状態 = インフラの望ましい状態」とする運用手法です。

### デプロイパイプラインの設計

```
git tag v1.2.3 && git push --tags
    │
    ▼
┌──────────┐    ┌────────────┐    ┌────────────────┐
│  Build   │───▶│  Security  │───▶│  Staging       │
│  & Push  │    │  Scan      │    │  Deploy (自動)  │
└──────────┘    └────────────┘    └───────┬────────┘
                                          │
                                     Smoke Test
                                          │
                                 ┌────────▼────────┐
                                 │  Production     │
                                 │  Deploy (手動承認)│
                                 └─────────────────┘
```

ステージングは自動デプロイ、本番は手動承認を挟むのが一般的です。GitHub Actionsの`environment`機能を使うと、承認フローを組み込めます。

### デプロイワークフローの要点

```yaml
# ステージングデプロイ（自動）
deploy-staging:
  needs: [build, security-scan]
  environment:
    name: staging
    url: https://staging.example.com
  steps:
    - name: Deploy to staging
      run: |
        ssh deploy@staging.example.com << EOF
          cd /opt/app
          export VERSION=${{ needs.build.outputs.version }}
          docker compose pull
          docker compose up -d --remove-orphans
        EOF
    - name: Smoke test
      run: curl -f https://staging.example.com/health || exit 1

# 本番デプロイ（手動承認後）
deploy-production:
  needs: [deploy-staging]
  environment:
    name: production  # ← ここで手動承認が求められる
    url: https://www.example.com
  steps:
    - name: Deploy to production
      run: |
        ssh deploy@prod.example.com << EOF
          cd /opt/app
          export VERSION=${{ needs.build.outputs.version }}
          docker compose pull
          docker compose up -d --remove-orphans
        EOF
```

`environment: production`を設定し、GitHubのSettings > Environments > productionで「Required reviewers」を追加すると、本番デプロイ前に指定メンバーの承認が必要になります。

### ロールバック

イミュータブルなタグでデプロイしていれば、ロールバックは前のバージョンのタグを指定するだけです。

```yaml
# .github/workflows/rollback.yml
on:
  workflow_dispatch:
    inputs:
      version:
        description: "ロールバック先のバージョン (例: 1.2.2)"
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Rollback
        run: |
          ssh deploy@prod.example.com << EOF
            cd /opt/app
            export VERSION=${{ inputs.version }}
            docker compose pull
            docker compose up -d --remove-orphans
          EOF
```

`workflow_dispatch`を使えば、GitHub UIからワンクリックでロールバックを実行できます。

## アンチパターン

CI/CDパイプラインで避けるべき典型的なミスを紹介します。

**latestタグだけでデプロイする**: どのバージョンが本番で動いているか追跡できなくなります。必ず明示的なバージョンタグを使ってください。

**シークレットをワークフローにハードコードする**: `docker login -u user -p password`のようにパスワードを直書きすると、リポジトリからシークレットが漏洩します。必ずGitHub Secretsを使ってください。

**テストなしでデプロイする**: ビルド成功だけでは安全ではありません。テスト、スキャン、ステージング検証を必ず挟んでください。

**ビルドとデプロイを1つのジョブに詰め込む**: ステージごとにジョブを分離し、ゲートを設けることで、テスト失敗時のデプロイを確実に防げます。

## まとめ

| 項目 | ポイント |
|------|---------|
| ワークフロー構成 | Docker公式Actionで統一。Buildx + metadata-action + build-push-actionが基本セット |
| レジストリ | GHCRがGitHub Actionsとの連携で最もスムーズ。ECRはAWS環境向き |
| タグ戦略 | セマンティックバージョン + Git SHAを併用。latestだけに依存しない |
| 自動テスト | Docker Composeでテスト環境を再現。Hadolint + Trivyでセキュリティも確保 |
| キャッシュ | GHA Cacheが推奨。Dockerfileのレイヤー順序最適化と組み合わせる |
| GitOps | ステージングは自動デプロイ、本番は手動承認のゲート付きパイプライン |
| ロールバック | イミュータブルタグで任意のバージョンに即座に戻せる設計にする |

## やってみよう！

- [ ] GitHub Actionsで自分のリポジトリにDockerビルドワークフローを作成し、GHCRにプッシュしてみましょう
- [ ] `docker/metadata-action`を使って、ブランチ名・Git SHA・セマンティックバージョンのタグを自動生成してみましょう
- [ ] `cache-from: type=gha`を設定し、2回目のビルドがどれだけ高速化されるか計測してみましょう
- [ ] Trivyを組み込み、自分のイメージに脆弱性がないかスキャンしてみましょう
- [ ] ステージング環境用のデプロイワークフローを作成し、`environment`で承認フローを設定してみましょう
