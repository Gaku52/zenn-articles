---
title: "ベストプラクティス集"
---

# Dockerfile ベストプラクティスまとめ

本書の最終章として、これまで学んできた Docker の知識を体系的に整理します。Dockerfile を書く際に押さえるべき原則、言語ごとのテンプレート、チェックリスト、そしてよくあるアンチパターンをまとめています。この章を辞書的にお使いいただき、日々の開発で参照してください。

---

## Dockerfile の基本原則

本番品質の Dockerfile を構築するために、以下の 5 つの原則を常に意識してください。

### 1. レイヤーキャッシュを最大限に活用する

Dockerfile の命令は「変更頻度の低いもの」を上に、「変更頻度の高いもの」を下に配置します。こうすることで、ソースコードを変更してもキャッシュが最大限に効き、ビルド時間を大幅に短縮できます。

```dockerfile
# 変更頻度: 最低（ベースイメージ）
FROM node:20-alpine
WORKDIR /app

# 変更頻度: 低（システム依存関係）
RUN apk add --no-cache curl

# 変更頻度: 中低（パッケージ定義ファイル）
COPY package.json package-lock.json ./
RUN npm ci --only=production

# 変更頻度: 最高（ソースコード）
COPY src/ ./src/
RUN npm run build
```

### 2. マルチステージビルドでイメージを最小化する

ビルドに必要なツール（コンパイラ、devDependencies など）を最終イメージに含めないことで、サイズとセキュリティの両方を改善できます。

### 3. non-root ユーザーで実行する

コンテナ内のプロセスを root で実行することは、セキュリティ上の大きなリスクです。必ず専用ユーザーを作成して切り替えてください。

### 4. .dockerignore でビルドコンテキストを最小化する

`.git`、`node_modules`、`.env` などの不要なファイルを除外し、ビルドコンテキストの転送量を削減します。

### 5. 具体的なバージョンタグを指定する

`FROM node:latest` ではなく `FROM node:20-alpine` のように、明示的なバージョンを指定します。これにより、ビルドの再現性が保証されます。

---

## 言語別 Dockerfile テンプレート

### Node.js（TypeScript API サーバー）

```dockerfile
# syntax=docker/dockerfile:1

# === 依存関係インストール ===
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

# === ビルド ===
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json tsconfig.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci
COPY src/ ./src/
RUN npm run build

# === 本番 ===
FROM node:20-alpine
RUN addgroup -S app && adduser -S app -G app
RUN apk add --no-cache dumb-init

WORKDIR /app

COPY --from=deps --chown=app:app /app/node_modules ./node_modules
COPY --from=builder --chown=app:app /app/dist ./dist
COPY --chown=app:app package.json ./

USER app
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider \
        http://localhost:3000/health || exit 1

ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/server.js"]
```

**Node.js のポイント:**

- `npm ci` は `package-lock.json` に完全一致するバージョンをインストールします。CI/CD 環境では必ずこちらを使います
- `dumb-init` は PID 1 問題を解決します。Node.js プロセスが PID 1 で動作すると、SIGTERM などのシグナルを正しく処理できません
- Alpine の musl libc で互換性問題が発生する場合（`sharp`、`bcrypt` など）は `node:20-slim` を検討してください

### Go（API サーバー）

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.22-alpine AS builder

RUN apk add --no-cache ca-certificates tzdata
RUN adduser -D -g '' appuser

WORKDIR /app

COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download && go mod verify

COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=linux \
    go build -ldflags="-w -s" -o /server ./cmd/server

# === 本番（scratch: 最小イメージ）===
FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=builder /etc/passwd /etc/passwd
COPY --from=builder /server /server

USER appuser
EXPOSE 8080
ENTRYPOINT ["/server"]
```

**Go のポイント:**

- `CGO_ENABLED=0` で静的バイナリを生成すると、`scratch`（空のイメージ）をベースにできます。最終イメージは 5-20MB 程度になります
- `-ldflags="-w -s"` でデバッグ情報とシンボルテーブルを除去し、バイナリサイズを削減します
- `scratch` には何もないため、HTTPS 通信に必要な CA 証明書とタイムゾーンデータを手動でコピーする必要があります

### Python（FastAPI サーバー）

```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.12-slim AS builder

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --prefix=/install -r requirements.txt

# === 本番 ===
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update && \
    apt-get install -y --no-install-recommends libpq5 curl && \
    rm -rf /var/lib/apt/lists/*

COPY --from=builder /install /usr/local

RUN useradd --create-home appuser
COPY --chown=appuser:appuser . .
USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "app.main:app", \
     "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

**Python のポイント:**

- `PYTHONDONTWRITEBYTECODE=1` は `.pyc` ファイルの生成を抑制し、イメージサイズを削減します
- `PYTHONUNBUFFERED=1` はログ出力をリアルタイムにし、`docker logs` でのデバッグを容易にします
- ビルドステージでのみ `gcc` をインストールし、本番ステージにはランタイムライブラリ（`libpq5` など）だけを含めます
- `pip install --prefix=/install` で依存関係を分離ディレクトリにインストールし、本番ステージにコピーするのが定番パターンです

---

## BuildKit マウントキャッシュの活用

BuildKit の `--mount=type=cache` を使うと、パッケージマネージャのキャッシュをビルド間で再利用できます。レイヤーキャッシュとは異なり、イメージに含まれないため安全です。

| 言語 | キャッシュパス | ロックファイル |
|---|---|---|
| Node.js (npm) | `/root/.npm` | `package-lock.json` |
| Node.js (pnpm) | `/root/.local/share/pnpm/store` | `pnpm-lock.yaml` |
| Python (pip) | `/root/.cache/pip` | `requirements.txt` |
| Python (uv) | `/root/.cache/uv` | `uv.lock` |
| Go | `/go/pkg/mod`, `/root/.cache/go-build` | `go.sum` |
| Rust | `/usr/local/cargo/registry` | `Cargo.lock` |
| Java (Gradle) | `/root/.gradle` | `gradle.lockfile` |

---

## ベストプラクティス チェックリスト

Dockerfile をレビューする際や、新規プロジェクトでコンテナ化を始める際に活用してください。

### 基本設定

- [ ] `FROM` でバージョンタグを明示的に指定している（`latest` を使っていない）
- [ ] `.dockerignore` を設定して不要ファイルを除外している
- [ ] マルチステージビルドを使い、ビルドツールを本番イメージに含めていない
- [ ] 変更頻度の低い命令を Dockerfile の上部に配置している
- [ ] `COPY . .` の前に依存関係ファイルだけを先にコピーしている

### セキュリティ

- [ ] non-root ユーザーを作成し、`USER` 命令で切り替えている
- [ ] 最小ベースイメージを使用している（alpine / slim / distroless / scratch）
- [ ] シークレット（パスワード、APIキーなど）を `ENV` や `ARG` で埋め込んでいない
- [ ] `HEALTHCHECK` を定義している
- [ ] 脆弱性スキャン（Trivy / Docker Scout）を CI に組み込んでいる

### 効率性

- [ ] `RUN` 命令をまとめてレイヤー数を削減している
- [ ] パッケージキャッシュを削除している（`rm -rf /var/lib/apt/lists/*` など）
- [ ] `--no-install-recommends`（apt）または `--no-cache`（apk）を使用している
- [ ] BuildKit マウントキャッシュ（`--mount=type=cache`）を活用している
- [ ] `CMD` / `ENTRYPOINT` で exec 形式（JSON 配列形式）を使用している

### 保守性

- [ ] `LABEL` でメタデータ（バージョン、作者、リポジトリ URL）を付与している
- [ ] `EXPOSE` でポート番号をドキュメントとして記述している
- [ ] Hadolint でリントを実施し、CI で自動チェックしている

---

## よくあるアンチパターン

### 1. `apt-get update` と `apt-get install` を別レイヤーにする

```dockerfile
# NG
RUN apt-get update
RUN apt-get install -y curl
# update のキャッシュだけが残り、古いパッケージリストで
# install が実行される可能性があります

# OK
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

### 2. ビルドツールを最終イメージに残す

```dockerfile
# NG: gcc (約200MB) が不要なのに含まれています
FROM python:3.12-slim
RUN apt-get update && apt-get install -y gcc libpq-dev
RUN pip install -r requirements.txt

# OK: マルチステージで分離します
FROM python:3.12-slim AS builder
RUN apt-get update && apt-get install -y gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

FROM python:3.12-slim
COPY --from=builder /install /usr/local
```

### 3. .dockerignore なしで `COPY . .` を実行する

```dockerfile
# NG: node_modules (300MB+) もコピーされます
FROM node:20-alpine
COPY . .
RUN npm ci  # コンテナ内で再インストールするため完全に無駄です

# OK: .dockerignore で除外 + 段階的コピー
FROM node:20-alpine
COPY package.json package-lock.json ./
RUN npm ci --only=production
COPY . .
```

### 4. ENV で変更頻度の高い値を上部に設定する

`ENV APP_VERSION=1.0.0` のような頻繁に変わる値を Dockerfile の上部に置くと、それ以降の全レイヤーのキャッシュが無効になります。変更頻度の高い `ENV` / `ARG` は Dockerfile の末尾に配置してください。

### 5. 開発用依存関係を本番イメージに含める

`npm install` ではなく `npm ci --only=production` を使い、eslint や jest などの開発ツールを本番イメージから除外してください。

### 6. HEALTHCHECK を省略する

プロセスが生きていてもアプリケーションが応答不能になるケースは頻繁に起こります。必ず `HEALTHCHECK` を設定してください。

### 7. `ADD` を `COPY` の代わりに使う

`ADD` は URL 取得や tar 展開の機能を持つため、予期しない動作の原因になります。ファイルコピーには常に `COPY` を使ってください。

---

## ベースイメージの選び方

| ベースイメージ | サイズ | 特徴 | 推奨用途 |
|---|---|---|---|
| `alpine` | 約 7MB | musl libc、最小構成 | Go、Node.js、汎用 |
| `slim` (Debian) | 約 74MB | glibc、必要十分な構成 | Python、Ruby、互換性重視 |
| `distroless` | 数MB | シェルなし、最小ランタイム | 本番環境、セキュリティ重視 |
| `scratch` | 0MB | 完全に空 | Go、Rust の静的バイナリ |

Alpine は最も軽量ですが、musl libc の互換性問題に注意してください。Python のネイティブ拡張（numpy など）や Node.js のネイティブモジュールでビルドエラーが発生する場合は、slim（glibc ベース）を選択します。

---

## セキュリティの強化

CI パイプラインに Trivy（脆弱性スキャン）と Hadolint（Dockerfile リント）を組み込むことで、問題を早期に検出できます。

```yaml
# GitHub Actions での Trivy + Hadolint
- uses: aquasecurity/trivy-action@master
  with:
    image-ref: my-app:${{ github.sha }}
    exit-code: '1'
    severity: 'CRITICAL,HIGH'

- uses: hadolint/hadolint-action@v3.1.0
  with:
    dockerfile: Dockerfile
```

ビルド時に必要なシークレットは BuildKit のシークレットマウントを使います。イメージのレイヤーに残りません。

```dockerfile
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) \
    npm ci --only=production
```

実行時に必要なシークレット（DB パスワードなど）は `docker run -e` や Docker Secrets で注入します。`ENV` や `ARG` に直接書くと `docker history` で閲覧できてしまうため、避けてください。

---

## 次のステップ: Kubernetes 入門

Docker でコンテナの基礎を習得したら、次はコンテナオーケストレーションに進みましょう。Kubernetes（K8s）は、複数のコンテナを大規模に管理・運用するためのプラットフォームです。

| Docker の概念 | Kubernetes の対応概念 |
|---|---|
| `docker run` | Pod / Deployment |
| `docker-compose.yml` | マニフェスト (YAML) |
| ポートマッピング | Service |
| ボリューム | PersistentVolume |
| 環境変数 | ConfigMap / Secret |
| HEALTHCHECK | livenessProbe / readinessProbe |

学習の進め方としては、まず minikube や kind でローカルクラスタを作成し、Pod / Deployment / Service の基本リソースを YAML で定義して kubectl で適用する流れを体験します。その後、Helm チャートによるパッケージング、GitHub Actions からの自動デプロイへとステップアップしていきます。

本書で身につけた Dockerfile の最適化やマルチステージビルドの知識は、Kubernetes 環境でもそのまま活きてきます。

---

## まとめ

| カテゴリ | ベストプラクティス | 効果 |
|---|---|---|
| レイヤー設計 | 変更頻度の低い命令を上に配置する | ビルド時間の短縮 |
| マルチステージ | ビルドツールを本番イメージに含めない | イメージサイズ削減・攻撃面縮小 |
| ベースイメージ | alpine / slim / distroless を用途で選択する | サイズとセキュリティの最適化 |
| キャッシュ | BuildKit マウントキャッシュを活用する | 依存関係の再インストール時間削減 |
| セキュリティ | non-root ユーザーで実行する | 権限昇格リスクの排除 |
| セキュリティ | シークレットマウントで機密情報を渡す | イメージへの機密情報漏洩防止 |
| セキュリティ | Trivy / Hadolint を CI に組み込む | 脆弱性とベストプラクティス違反の自動検出 |
| .dockerignore | node_modules / .git / .env を除外する | ビルドコンテキストの最小化 |
| HEALTHCHECK | アプリケーション固有のヘルスチェックを設定する | 異常検知と自動復旧の実現 |
| 保守性 | LABEL でメタデータを付与する | イメージの追跡・管理の容易化 |
| Node.js | npm ci + dumb-init + Alpine | PID 1 問題の解決・再現性の確保 |
| Go | CGO_ENABLED=0 + scratch | 最小イメージ（5-20MB） |
| Python | slim + pip install --prefix で分離 | ビルドツール除去・環境変数設定 |
| 次のステップ | Kubernetes でコンテナオーケストレーションを学ぶ | 大規模運用への対応力 |
