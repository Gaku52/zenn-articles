---
title: "Docker Compose応用"
---

## この章で学ぶこと

前章までで Docker Compose の基本的な使い方を学びました。この章では、実際の開発・運用で必要になる応用機能を扱います。profiles による選択的なサービス起動、extends/merge による設定の再利用、ヘルスチェックによる起動順序の保証、リソース制限、そして複数の compose.yaml を使い分ける方法を身につけましょう。

## profiles -- サービスの選択的起動

### profiles とは

`profiles` は、サービスを用途別にグルーピングして必要なときだけ起動する仕組みです。`profiles` が指定されていないサービスは常に起動されます。`profiles` が指定されたサービスは、明示的にプロファイルを有効化しないと起動されません。

```yaml
# compose.yaml
services:
  # profiles なし → 常に起動
  app:
    build: .
    ports:
      - "3000:3000"

  db:
    image: postgres:16-alpine

  # debug プロファイル -- 開発時のみ起動
  pgadmin:
    image: dpage/pgadmin4:latest
    profiles: ["debug"]
    ports:
      - "5050:80"

  redis-commander:
    image: rediscommander/redis-commander:latest
    profiles: ["debug"]
    ports:
      - "8081:8081"

  # test プロファイル -- テスト実行時のみ
  test-runner:
    build:
      context: .
      target: test
    profiles: ["test"]
    command: npm test
```

### profiles の起動方法

```bash
# デフォルトサービスのみ起動（app, db）
docker compose up -d

# debug プロファイルを追加起動
docker compose --profile debug up -d

# 複数プロファイルを同時に有効化
docker compose --profile debug --profile monitoring up -d

# 環境変数で指定する方法
COMPOSE_PROFILES=debug,monitoring docker compose up -d

# テストの実行（ワンショット）
docker compose --profile test run --rm test-runner

# 特定プロファイルのサービスだけ停止
docker compose --profile debug stop
```

普段の開発では `docker compose up -d` だけでシンプルに動かし、DB を GUI で確認したいときだけ `--profile debug` を追加する、という運用ができます。

## extends と merge -- 設定の再利用

### extends による継承

`extends` を使うと、別ファイルのサービス設定を継承して新しいサービスを定義できます。共通部分をベースに切り出し、差分だけ記述する方法です。

```yaml
# common.yaml（共通定義）
services:
  base-app:
    build: .
    environment:
      TZ: Asia/Tokyo
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

```yaml
# compose.yaml
services:
  web:
    extends:
      file: common.yaml
      service: base-app
    ports:
      - "3000:3000"
    command: node dist/web.js

  worker:
    extends:
      file: common.yaml
      service: base-app
    command: node dist/worker.js
```

`extends` で継承した設定は、サービス側の定義で上書きできます。

### YAML アンカーによる DRY 化

同一ファイル内で設定を共有したい場合は、YAML アンカー（`&` と `*`）と Extension Fields（`x-` プレフィックス）を組み合わせます。

```yaml
x-common-env: &common-env
  TZ: Asia/Tokyo
  NODE_ENV: production
  DATABASE_URL: postgresql://postgres:${DB_PASSWORD}@db:5432/myapp

x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

services:
  web:
    build: .
    environment:
      <<: *common-env
      SERVER_TYPE: web
    logging: *default-logging

  worker:
    build: .
    environment:
      <<: *common-env
      SERVER_TYPE: worker
    logging: *default-logging
```

`x-` で始まるフィールドは Compose に無視されるため、アンカーの定義場所として安全に使えます。`<<: *common-env` はマージキーと呼ばれ、アンカーの内容を展開した上で、同じキーがあればサービス側の値が優先されます。

## ヘルスチェック -- 確実な起動順序の保証

### なぜヘルスチェックが必要か

`depends_on` だけでは「コンテナが起動した」ことしか保証しません。データベースはコンテナ起動から接続受付まで数秒かかるため、アプリが接続エラーでクラッシュする可能性があります。`healthcheck` と `condition: service_healthy` を組み合わせることで、サービスが本当に利用可能になるまで待機できます。

### 主要サービスの healthcheck 設定例

```yaml
services:
  # PostgreSQL
  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d myapp"]
      interval: 5s       # チェック間隔
      timeout: 5s        # タイムアウト
      retries: 5         # リトライ回数
      start_period: 30s  # 起動猶予（この間の失敗はカウントしない）

  # Redis
  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  # HTTP サービス
  api:
    build: .
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 15s
```

`start_period` はサービスの初期化に必要な時間を見積もって設定します。テスト環境で `docker compose up` してからサービスが応答するまでの時間を計測し、その 1.5〜2 倍を設定するのが目安です。

### depends_on の 3 つの条件

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
      migration:
        condition: service_completed_successfully
```

| 条件 | 動作 | 使いどころ |
|------|------|-----------|
| `service_started` | コンテナが起動したら次へ進む | Redis など起動が速いサービス |
| `service_healthy` | healthcheck が通るまで待機 | DB など初期化に時間がかかるサービス |
| `service_completed_successfully` | 終了コード 0 で完了するまで待機 | マイグレーション、シード処理 |

### 複雑な起動チェーンの例

DB 起動 → マイグレーション → アプリ起動という順序を保証する例です。

```yaml
services:
  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 30s

  migration:
    build: .
    depends_on:
      db:
        condition: service_healthy
    command: npx prisma migrate deploy

  app:
    build: .
    depends_on:
      migration:
        condition: service_completed_successfully
    ports:
      - "3000:3000"
```

`migration` は DB の healthcheck が通ってから実行され、`app` は migration が正常終了（終了コード 0）してから起動します。

## リソース制限 -- 安定稼働のための設定

### CPU とメモリの制限

リソース制限を設定しないと、1 つのサービスがホスト全体のリソースを消費し、他のサービスに影響します。特に本番環境では必ず設定しましょう。

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '1.0'       # CPU 上限 1コア
          memory: 512M       # メモリ上限 512MB
        reservations:
          cpus: '0.25'      # 最低保証 CPU
          memory: 128M       # 最低保証メモリ
```

`limits` はそのサービスが使える最大量です。`reservations` はホストがそのサービスのために確保する最低量で、他のサービスが暴走してもこの分は守られます。

### ロギングの制限

ログのローテーションを設定しないと、ログファイルが際限なく増えてディスクが枯渇します。

```yaml
services:
  app:
    logging:
      driver: json-file
      options:
        max-size: "10m"  # ログファイル最大サイズ
        max-file: "3"    # 保持するファイル数
```

この設定で、ログは最大 10MB x 3 ファイル = 30MB に制限されます。

## 複数 compose.yaml の使い分けと override

### override パターン

Docker Compose はベースの設定ファイルと環境固有の設定ファイルをマージして 1 つの構成を作れます。

```yaml
# compose.yaml（ベース設定）
services:
  app:
    build: .
    environment:
      NODE_ENV: production

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

```yaml
# compose.override.yaml（開発用。自動マージされる）
services:
  app:
    build:
      target: development
    environment:
      NODE_ENV: development
      DEBUG: "true"
    volumes:
      - .:/app
    ports:
      - "3000:3000"
      - "9229:9229"   # デバッガポート

  db:
    ports:
      - "5432:5432"   # 開発時のみ外部公開
    environment:
      POSTGRES_PASSWORD: postgres
```

```yaml
# compose.prod.yaml（本番用）
services:
  app:
    restart: always
    deploy:
      resources:
        limits: { cpus: '1.0', memory: 512M }
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }

  db:
    restart: always
    deploy:
      resources:
        limits: { cpus: '2.0', memory: 1G }
```

### マージのコマンド

```bash
# 開発（compose.yaml + compose.override.yaml を自動マージ）
docker compose up -d

# 本番（override を除外し、prod を適用）
docker compose -f compose.yaml -f compose.prod.yaml up -d

# マージ結果のプレビュー
docker compose -f compose.yaml -f compose.prod.yaml config
```

`compose.override.yaml` は特別なファイル名で、`docker compose up` 時に自動的にマージされます。`-f` で明示的にファイルを指定した場合、`compose.override.yaml` は自動マージされません。

### マージの規則

| 設定項目 | マージ動作 |
|----------|-----------|
| `image`, `command` | 後のファイルで上書き |
| `environment` | キー単位でマージ（同じキーは上書き） |
| `volumes`, `ports` | 追加される |
| `deploy` | ディープマージ |
| `healthcheck` | 後のファイルで完全上書き |

### COMPOSE_FILE 環境変数

`.env` ファイルで読み込むファイルを指定すると、`-f` の指定を省略できます。

```bash
# .env
COMPOSE_FILE=compose.yaml:compose.prod.yaml
```

これで `docker compose up -d` だけで本番構成が適用されます。環境ごとに `.env` を切り替えることで、チーム全体で統一した運用が可能です。

## まとめ

| 機能 | 用途 | ポイント |
|------|------|---------|
| profiles | サービスの選択的起動 | `profiles: ["debug"]` で定義し `--profile debug` で起動 |
| extends | 別ファイルからの設定継承 | 共通設定を `common.yaml` にまとめて差分だけ記述 |
| YAML アンカー | 同一ファイル内の設定共有 | `x-` Extension Fields にアンカーを定義して `<<:` で展開 |
| healthcheck | サービスの準備完了を検知 | `interval`, `timeout`, `retries`, `start_period` を適切に設定 |
| depends_on condition | 起動順序の保証 | `service_healthy` / `service_completed_successfully` を使い分け |
| リソース制限 | CPU/メモリの上限設定 | `deploy.resources.limits` と `reservations` で制限と最低保証 |
| ログローテーション | ディスク枯渇の防止 | `max-size` と `max-file` を必ず設定 |
| override | 環境ごとの設定切り替え | `compose.override.yaml` は自動マージ、`-f` で明示指定も可 |
| COMPOSE_FILE | マージ対象の自動選択 | `.env` に定義して `docker compose up` だけで環境切り替え |

## やってみよう！

1. **profiles を使った環境分離**: `debug` プロファイルを追加し、pgAdmin を `--profile debug` でのみ起動する設定を作ってみてください。
2. **healthcheck の導入**: DB に healthcheck を設定し、`condition: service_healthy` でアプリが DB 準備完了後に起動することを確認しましょう。
3. **override ファイルの作成**: ベース・開発用・本番用の 3 ファイル構成を作り、`docker compose config` でマージ結果を確認してください。
4. **YAML アンカーの活用**: 3 つ以上のサービスが同じ設定を持つ compose.yaml で `x-` とアンカーを使い、重複を排除してください。
