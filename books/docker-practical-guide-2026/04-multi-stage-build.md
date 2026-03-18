---
title: "マルチステージビルドと最適化"
---

## マルチステージビルドとは

Docker イメージを本番運用するとき、最も多い失敗は「ビルドに使ったツールがそのまま最終イメージに残ってしまう」ことです。コンパイラ、リンカ、テストフレームワーク、開発用ライブラリ......。これらが残っていると、イメージサイズが肥大化し、セキュリティリスクも増大します。

マルチステージビルドは、1 つの Dockerfile の中に複数のビルドステージを定義し、最終ステージには本当に必要なファイルだけをコピーすることで、この問題を解決します。

### シングルステージとの違い

シングルステージビルドでは、すべてが 1 つのイメージに詰め込まれます。

```
シングルステージ（従来型）
+------------------------------------------+
|  Node.js ランタイム         ~300 MB       |
|  npm / yarn                 ~50 MB        |
|  ビルドツール (gcc 等)       ~200 MB       |
|  node_modules (dev 含む)    ~400 MB       |
|  ソースコード                ~10 MB        |
|  ビルド成果物               ~5 MB          |
+------------------------------------------+
合計: ~965 MB  ← ビルドツールは実行時に不要
```

マルチステージビルドでは、ビルド環境と実行環境を分離します。

```
マルチステージビルド
Stage 1: ビルド            Stage 2: 実行
+--------------------+    +--------------------+
| Node.js + npm      |    | Node.js (Alpine)   |
| ビルドツール        |    | 本番 node_modules  |
| 全 node_modules    | -> | ビルド成果物        |
| ソースコード        |COPY| (必要なものだけ)    |
+--------------------+    +--------------------+
~965 MB (破棄)             ~150 MB (最終イメージ)
```

### 基本構文

マルチステージビルドの基本的な書き方を見てみましょう。

```dockerfile
# ステージ 1: ビルド
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# ステージ 2: 実行（最終イメージ）
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

ポイントは `FROM` が 2 回登場していることです。2 つ目の `FROM` で新しいステージが始まり、`COPY --from=builder` によってビルドステージの成果物だけを持ってきます。ビルドステージの残りのファイル（ソースコード、devDependencies など）は最終イメージに含まれません。

```bash
# ビルド（最終ステージのみが最終イメージになる）
docker build -t my-app:v1.0.0 .

# 特定のステージまでビルドしたい場合
docker build --target builder -t my-app-builder .
```

## イメージサイズの削減効果

マルチステージビルドを適用すると、どの程度のサイズ削減が見込めるのでしょうか。言語ごとに比較してみます。

| アプリ | シングルステージ | マルチステージ | 削減率 |
|---|---|---|---|
| Node.js (Express) | ~950 MB | ~150 MB | 84% |
| Go (Web API) | ~800 MB | ~12 MB | 98% |
| Rust (Web API) | ~1.5 GB | ~15 MB | 99% |
| Java (Spring Boot) | ~600 MB | ~200 MB | 67% |
| Python (FastAPI) | ~900 MB | ~180 MB | 80% |

Go や Rust のようにシングルバイナリにコンパイルできる言語では、ベースイメージに `scratch`（完全に空のイメージ）を使うことで劇的な削減が可能です。

## ビルドキャッシュ戦略

マルチステージビルドの効果を最大化するには、Docker のビルドキャッシュを正しく理解する必要があります。

### キャッシュの動作原理

Docker は Dockerfile の各命令をレイヤーとして管理しています。ビルド時に各レイヤーのキャッシュが有効かどうかを上から順に判定し、あるレイヤーでキャッシュが無効になると、それ以降のすべてのレイヤーが再ビルドされます。

- `FROM`: ベースイメージが同じかどうか
- `RUN`: コマンド文字列が完全一致するかどうか
- `COPY` / `ADD`: コピー対象ファイルの内容が変わっていないかどうか

この「カスケード無効化」の性質を踏まえると、最適化の原則は「変更頻度の低い命令を上に、高い命令を下に配置する」ことになります。

### 依存関係とソースコードの分離

キャッシュ最適化の最も基本的なテクニックです。

```dockerfile
# NG: 毎回全依存関係を再インストール
FROM node:20-alpine AS builder
WORKDIR /app
COPY . .                    # ソースコード変更 → 全キャッシュ無効
RUN npm ci                  # 毎回再実行される
RUN npm run build

# OK: 依存関係ファイルを先にコピー
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./  # 依存関係が変わった時だけ無効
RUN npm ci                               # キャッシュが効く
COPY . .                                 # ソースコードだけ再コピー
RUN npm run build
```

ソースコードを変更しても `package.json` が変わっていなければ、`npm ci` のレイヤーはキャッシュから再利用されます。依存関係のインストールは通常 1〜3 分かかるため、この最適化は非常に効果的です。

### BuildKit のマウントキャッシュ

BuildKit には `--mount=type=cache` という強力なキャッシュ機構があります。パッケージマネージャーのキャッシュディレクトリをビルド間で永続化し、再利用できます。

```dockerfile
# Node.js の npm キャッシュ
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci
COPY . .
RUN npm run build
```

```dockerfile
# Go のモジュール + ビルドキャッシュ
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -o /server .
```

このキャッシュはイメージのレイヤーには含まれず、ビルドホスト上に永続化されるため、イメージサイズには影響しません。

## ベースイメージの選び方 ── distroless と Alpine

最終ステージのベースイメージ選択は、イメージサイズとセキュリティに直結します。

### 主要なベースイメージの比較

| ベースイメージ | サイズ | シェル | パッケージMgr | C ライブラリ | 攻撃対象面 |
|---|---|---|---|---|---|
| `scratch` | 0 MB | なし | なし | なし | 最小 |
| `distroless` | 数〜120 MB | なし | なし | glibc | 極小 |
| `alpine` | ~7 MB | ash | apk | musl | 小 |
| `debian-slim` | ~74 MB | bash | apt | glibc | 中 |
| フルイメージ | 300+ MB | bash | apt | glibc | 大 |

### scratch

完全に空のベースイメージです。Go や Rust のように静的リンクされたシングルバイナリを動かす場合に最適です。シェルがないため `docker exec` でのデバッグはできません。

```dockerfile
FROM golang:1.22-alpine AS builder
RUN apk add --no-cache ca-certificates
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-w -s" -o /server .

FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /server /server
ENTRYPOINT ["/server"]
```

scratch を使うときは CA 証明書のコピーを忘れないでください。これがないと HTTPS 通信でエラーになります。

### distroless

Google が提供する最小限のランタイムイメージです。Node.js、Java、Python など動的リンクが必要な言語に向いています。シェルがないため攻撃対象面が非常に小さく、デバッグが必要な場合は `:debug` タグのバリアントを使います。

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["dist/server.js"]
```

### Alpine

わずか 7 MB の軽量 Linux ディストリビューションです。パッケージマネージャー（apk）やシェル（ash）があるため、追加のツールインストールやデバッグが可能です。ただし C ライブラリが musl のため、glibc 前提のバイナリとの互換性問題が起きることがあります。

Python のネイティブ拡張（numpy など）や Node.js のネイティブモジュールで問題が発生する場合は、`debian-slim` に切り替えてください。

## 言語別の最適化パターン

### Node.js (TypeScript)

Node.js では 4 ステージ構成が推奨されます。依存関係を `deps`（全依存）と `prod-deps`（本番のみ）に分離することで、キャッシュ効率を最大化できます。

```dockerfile
# ステージ 1: 全依存関係
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# ステージ 2: ビルド
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# ステージ 3: 本番依存関係のみ
FROM node:20-alpine AS prod-deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production && npm cache clean --force

# ステージ 4: 実行
FROM node:20-alpine
RUN addgroup -S app && adduser -S app -G app
RUN apk add --no-cache dumb-init
WORKDIR /app
COPY --from=prod-deps --chown=app:app /app/node_modules ./node_modules
COPY --from=builder --chown=app:app /app/dist ./dist
COPY --chown=app:app package.json ./
USER app
EXPOSE 3000
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/server.js"]
```

`dumb-init` は PID 1 問題（シグナルが正しく転送されない問題）を解決するための軽量 init プロセスです。本番運用では入れておくことをおすすめします。

### Go

Go はシングルバイナリにコンパイルできるため、scratch ベースで 10〜15 MB 程度のイメージが実現できます。

```dockerfile
FROM golang:1.22-alpine AS builder
RUN apk add --no-cache ca-certificates tzdata
RUN adduser -D -g '' appuser
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download && go mod verify
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-w -s" -o /server ./cmd/server

FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=builder /etc/passwd /etc/passwd
COPY --from=builder /server /server
USER appuser
EXPOSE 8080
ENTRYPOINT ["/server"]
```

`-ldflags="-w -s"` はデバッグ情報を削除してバイナリサイズを縮小するオプションです。`CGO_ENABLED=0` で C 言語への依存を排除し、静的リンクを保証します。

### Python (FastAPI)

Python ではビルドステージで `--prefix` オプションを使い、インストール先を分離するパターンが有効です。

```dockerfile
FROM python:3.12-slim AS builder
ENV PYTHONDONTWRITEBYTECODE=1 PIP_NO_CACHE_DIR=1
WORKDIR /app
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
RUN apt-get update && \
    apt-get install -y --no-install-recommends libpq5 && \
    rm -rf /var/lib/apt/lists/*
COPY --from=builder /install /usr/local
RUN useradd --create-home appuser
COPY --chown=appuser:appuser . .
USER appuser
EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", \
     "--worker-class", "uvicorn.workers.UvicornWorker", "app.main:app"]
```

ビルドステージには `gcc` が必要ですが、実行ステージにはランタイムライブラリ（`libpq5`）だけを入れます。`gcc` を持ち込まないだけで数百 MB の削減になります。

### Java (Spring Boot)

Spring Boot 3.x のレイヤード JAR を活用すると、依存関係とアプリケーションコードを Docker レイヤーとして分離でき、デプロイ時のキャッシュ効率が向上します。

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY gradlew build.gradle.kts settings.gradle.kts ./
COPY gradle ./gradle
RUN ./gradlew dependencies --no-daemon
COPY src ./src
RUN ./gradlew bootJar --no-daemon
RUN java -Djarmode=layertools -jar build/libs/*.jar extract --destination extracted

FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
COPY --from=builder /app/extracted/dependencies/ ./
COPY --from=builder /app/extracted/spring-boot-loader/ ./
COPY --from=builder /app/extracted/snapshot-dependencies/ ./
COPY --from=builder /app/extracted/application/ ./
USER app
EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

ビルドステージでは JDK、実行ステージでは JRE を使う点もポイントです。

## CI/CD でのキャッシュ活用

CI 環境はステートレスなため、何もしないとビルドのたびにキャッシュが失われます。GitHub Actions では `docker/build-push-action` の `cache-from` / `cache-to` オプションで対処できます。

```yaml
- uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

`type=gha` は GitHub の Cache API を利用する方式で、設定が最もシンプルです。他にも `type=registry`（レジストリにキャッシュを保存、異なるランナー間で共有可能）や `type=local`（自前 CI サーバー向け）があります。

## よくあるアンチパターン

マルチステージビルドを使っていても、以下のミスをすると効果が半減します。

**成果物を丸ごとコピーしてしまう** -- `COPY --from=builder /app .` とすると、ソースコードや devDependencies まで含まれます。`COPY --from=builder /app/dist ./dist` のように必要なディレクトリだけを明示してください。

**キャッシュ効率を無視した COPY 順序** -- `COPY . .` の後に `RUN npm ci` と書くと、ソースコード変更のたびに依存関係を再インストールしてしまいます。`package.json` を先にコピーし、`npm ci` の後に `COPY . .` としましょう。

**scratch で CA 証明書を忘れる** -- scratch には何も入っていないため、HTTPS 通信に必要な CA 証明書をビルドステージからコピーし忘れると、外部 API 呼び出しが全て失敗します。

## まとめ

| 項目 | ポイント |
|---|---|
| マルチステージビルド | ビルド環境と実行環境を分離し、最終イメージを最小化します |
| COPY --from | ビルドステージから必要なファイルだけを実行ステージにコピーします |
| キャッシュ最適化 | 変更頻度の低い命令を上に配置し、依存関係とソースコードを分離します |
| BuildKit マウントキャッシュ | `--mount=type=cache` でパッケージマネージャーのキャッシュを再利用します |
| ベースイメージ選択 | scratch（Go/Rust）、distroless（Node.js/Java）、Alpine を用途に応じて選びます |
| Node.js | 4 ステージ構成で devDependencies を排除し、150 MB 程度に削減します |
| Go | scratch + 静的リンクで 10〜15 MB のイメージを構築します |
| Python | `--prefix` で依存関係を分離し、gcc をイメージに持ち込みません |
| Java | レイヤード JAR で依存関係とコードを分離し、デプロイを高速化します |
| CI/CD キャッシュ | `type=gha` やレジストリキャッシュで CI のビルド時間を短縮します |

## やってみよう！

以下の手順で、マルチステージビルドの効果を体感してみましょう。

1. **サイズ比較**: 既存プロジェクトの Dockerfile をシングルステージのまま `docker build` し、`docker images` でサイズを確認します。マルチステージに書き換えて再ビルドし、削減量を比較してみましょう。
2. **キャッシュ検証**: `package.json` を変えずにソースコードだけ変更して再ビルドし、`--progress=plain` で `npm ci` レイヤーがキャッシュされていることを確認しましょう。
3. **ベースイメージ切り替え**: 最終ステージを `node:20` → `node:20-alpine` → `gcr.io/distroless/nodejs20-debian12` の順に切り替え、サイズとデバッグのしやすさを比較してみましょう。
4. **dive で分析**: `docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive:latest <イメージ名>` でレイヤーごとのサイズと不要ファイルの有無を確認してみましょう。
