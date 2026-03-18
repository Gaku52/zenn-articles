---
title: "開発ワークフロー"
---

## この章で学ぶこと

Docker を使った日常の開発ワークフローを最適化する方法を学びます。ホットリロードによるコード変更の即時反映、Dev Containers によるチーム統一の開発環境、ローカル DB/Redis 構築、そして開発と本番で Compose ファイルを分離する実践パターンを扱います。

---

## ホットリロード設定

開発中にコードを変更するたびにコンテナを再ビルドしていては生産性が下がります。ホットリロードを正しく設定すれば、ファイル保存の瞬間に変更が反映されます。

### Node.js（Vite / Next.js）の場合

```yaml
# compose.yaml
services:
  app:
    build:
      context: .
      target: dev
    ports:
      - "3000:3000"
      - "24678:24678"   # Vite HMR 用 WebSocket ポート
    volumes:
      - .:/app
      - node_modules:/app/node_modules  # パフォーマンスのため分離
    environment:
      WATCHPACK_POLLING: "true"         # Next.js のファイル監視
      CHOKIDAR_USEPOLLING: "true"       # Chokidar の polling
    command: pnpm dev --host

volumes:
  node_modules:
```

ポイントは 2 つあります。

1. **ソースコードはバインドマウント**で共有し、ホスト側の変更をコンテナに即時反映させます
2. **`node_modules` は名前付きボリュームで分離**します。macOS ではバインドマウント経由の I/O が遅く、分離するだけで `npm install` が数倍速くなります

### 開発用 Dockerfile

```dockerfile
FROM node:20-slim AS dev
WORKDIR /app
RUN corepack enable
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
# ソースはバインドマウントで注入するため COPY 不要
EXPOSE 3000
CMD ["pnpm", "dev"]
```

本番用の `COPY . .` を開発ステージに入れるとバインドマウントと競合します。開発ステージではソースのコピーを省略してください。

### Python（FastAPI）の場合

```yaml
services:
  api:
    build:
      context: .
      target: dev
    ports:
      - "8000:8000"
    volumes:
      - .:/app
    environment:
      PYTHONDONTWRITEBYTECODE: "1"
      PYTHONUNBUFFERED: "1"
    command: uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

`--reload` フラグだけで、ファイル変更時に uvicorn が自動的にプロセスを再起動してくれます。

### Docker Compose Watch（V2.22+）

`develop.watch` セクションを使ったファイル同期です。バインドマウントの代替として、特に macOS で効果的です。

```yaml
services:
  app:
    build: .
    develop:
      watch:
        - action: sync           # ソース変更 → コンテナにコピー
          path: ./src
          target: /app/src
        - action: rebuild        # package.json 変更 → 再ビルド
          path: ./package.json
        - action: sync+restart   # 設定変更 → コピー後に再起動
          path: ./config
          target: /app/config
```

| アクション | 動作 | 使いどころ |
|-----------|------|-----------|
| `sync` | ファイルをコンテナにコピー | ソースコード |
| `rebuild` | イメージを再ビルド | 依存関係の変更 |
| `sync+restart` | コピー後にプロセス再起動 | 設定ファイル |

`docker compose watch` コマンドで起動します。

---

## Dev Containers で統一された開発環境

Dev Containers を使うと、VS Code がコンテナ内で直接動作します。チーム全員が同じ OS、同じツールバージョン、同じ拡張機能で開発できるため「自分の環境では動くのに」という問題を根本的に解消できます。

### devcontainer.json の設定

```jsonc
// .devcontainer/devcontainer.json
{
  "name": "My Project",
  "dockerComposeFile": ["../compose.yaml", "compose.devcontainer.yaml"],
  "service": "app",
  "workspaceFolder": "/app",
  "features": {
    "ghcr.io/devcontainers/features/git:1": {},
    "ghcr.io/devcontainers/features/github-cli:1": {}
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode"
      ],
      "settings": {
        "editor.formatOnSave": true,
        "editor.defaultFormatter": "esbenp.prettier-vscode"
      }
    }
  },
  "forwardPorts": [3000, 5432, 6379],
  "postCreateCommand": "pnpm install && npx prisma generate",
  "remoteUser": "node"
}
```

### Dev Container 用の override ファイル

```yaml
# .devcontainer/compose.devcontainer.yaml
services:
  app:
    build:
      context: ..
      target: dev
    volumes:
      - ..:/app:cached
      - node_modules:/app/node_modules
    command: sleep infinity   # VS Code がシェルを使うためプロセスを維持

volumes:
  node_modules:
```

`command: sleep infinity` がポイントです。Dev Containers ではコンテナ内のターミナルから手動でコマンドを実行するため、アプリを自動起動する必要はありません。

Dev Containers はチーム全員が VS Code を使っている場合に最適です。エディタが混在するチームではバインドマウント方式の方が柔軟です。
---

## ローカル DB / Redis の構築

開発環境で DB やキャッシュをローカルに立てれば、外部サービスへの依存がなくなり、オフラインでも開発できます。

### PostgreSQL + Redis の構成

```yaml
services:
  app:
    build:
      context: .
      target: dev
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://postgres:postgres@db:5432/myapp_dev
      REDIS_URL: redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: myapp_dev
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --maxmemory 128mb --maxmemory-policy allkeys-lru

volumes:
  pgdata:
  redis_data:
```

### 設計のポイント

**ヘルスチェックと `depends_on`** -- `condition: service_healthy` を指定すると、DB が接続可能になるまでアプリの起動を待機します。これがないと「DB にまだ接続できない」エラーが頻発します。

**初期化スクリプト** -- `/docker-entrypoint-initdb.d/` にマウントした SQL は、ボリュームが空の初回起動時にだけ実行されます。テーブル作成やシードデータの投入に使えます。

**ポートの公開** -- 開発時は `ports` でホストに公開しておくと、TablePlus 等の GUI ツールから直接接続できて便利です。

### よく使う操作コマンド

```bash
docker compose exec db psql -U postgres -d myapp_dev   # DB 接続
docker compose exec redis redis-cli                     # Redis CLI
docker compose exec db pg_dump -U postgres myapp_dev > backup.sql  # バックアップ
docker compose down -v && docker compose up -d          # データ全消去してやり直し
```

---

## 開発と本番の Compose 分離

1 つの `compose.yaml` に開発と本番を混在させると、デバッグポートの本番公開のようなセキュリティ事故につながります。override 機能で環境ごとにファイルを分離しましょう。

### ファイル構成

```
project/
├── compose.yaml              # ベース（全環境共通）
├── compose.override.yaml     # 開発用（自動読み込み）
├── compose.prod.yaml         # 本番用
├── compose.ci.yaml           # CI 用
└── Dockerfile                # マルチステージビルド
```

### ベースファイル（全環境共通）

```yaml
# compose.yaml
services:
  app:
    build:
      context: .
    environment:
      DATABASE_URL: postgresql://postgres:postgres@db:5432/myapp
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: myapp
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

### 開発用 override

```yaml
# compose.override.yaml（docker compose up で自動読み込み）
services:
  app:
    build:
      target: dev
    ports:
      - "3000:3000"
      - "9229:9229"       # デバッガポート（開発時のみ）
    volumes:
      - .:/app
      - node_modules:/app/node_modules
    environment:
      NODE_ENV: development
    command: node --inspect=0.0.0.0:9229 node_modules/.bin/tsx watch src/index.ts

  db:
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  node_modules:
  pgdata:
```

### 本番用

```yaml
# compose.prod.yaml
services:
  app:
    build:
      target: production
    environment:
      NODE_ENV: production
    read_only: true
    security_opt:
      - no-new-privileges:true
    # デバッガポートは公開しない、ボリュームマウントもしない

  db:
    volumes:
      - pgdata_prod:/var/lib/postgresql/data

volumes:
  pgdata_prod:
```

### CI 用

```yaml
# compose.ci.yaml
services:
  app:
    build:
      target: test
    environment:
      NODE_ENV: test

  db:
    tmpfs:
      - /var/lib/postgresql/data   # メモリ上で高速化
```

### 使い分け

```bash
# 開発（compose.yaml + compose.override.yaml を自動読み込み）
docker compose up -d

# 本番（-f 指定で override の自動読み込みは無効になる）
docker compose -f compose.yaml -f compose.prod.yaml up -d

# CI
docker compose -f compose.yaml -f compose.ci.yaml up -d --wait
docker compose -f compose.yaml -f compose.ci.yaml exec -T app pnpm test
docker compose -f compose.yaml -f compose.ci.yaml down -v
```

### Makefile でコマンドを整理する

長いコマンドは Makefile にまとめておくと、チーム全員が `make dev` / `make test` で統一した操作ができます。

```makefile
dev:    ## 開発環境を起動
	docker compose up -d --wait
test:   ## CI 構成でテスト実行
	docker compose -f compose.yaml -f compose.ci.yaml up -d --wait
	docker compose -f compose.yaml -f compose.ci.yaml exec -T app pnpm test
	docker compose -f compose.yaml -f compose.ci.yaml down -v
clean:  ## 全削除（ボリューム含む）
	docker compose down -v --remove-orphans
```

---

## まとめ

| テーマ | ポイント |
|-------|---------|
| ホットリロード | バインドマウント + `node_modules` のボリューム分離が基本。macOS では Compose Watch も有効です |
| Dev Containers | VS Code + コンテナで全員同じ環境を実現します。`sleep infinity` でプロセスを維持します |
| ローカル DB/Redis | `healthcheck` + `depends_on` で起動順を制御します。初期化は `initdb.d` を使います |
| Compose 分離 | `compose.override.yaml` が開発用、`-f` 指定で本番・CI を切り替えます |
| デバッガポート | 開発の override でのみ公開し、本番には絶対に含めないでください |
| Makefile | チームの共通インターフェースとして `make dev` / `make test` を整備しましょう |

---

## やってみよう！

- [ ] 自分のプロジェクトに `compose.yaml` + `compose.override.yaml` の 2 ファイル構成を導入してみましょう
- [ ] `node_modules` を名前付きボリュームに分離し、`docker compose up` の速度変化を計測してみましょう
- [ ] `healthcheck` を追加して `depends_on` の `condition: service_healthy` でアプリの起動順を制御してみましょう
- [ ] Dev Containers を作成し、VS Code の「Reopen in Container」で開発環境を起動してみましょう
- [ ] Makefile に `dev` / `test` / `clean` の 3 ターゲットを定義し、チームメンバーと共有してみましょう
