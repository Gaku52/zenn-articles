---
title: "Docker Compose入門"
---

## Docker Compose とは

Docker Compose は、複数のコンテナをまとめて定義・管理するためのツールです。Web アプリケーションを動かすにはアプリサーバー、データベース、キャッシュなど複数のコンテナが必要になりますが、それぞれを `docker run` で個別に起動するのは手間がかかりますし、再現性にも問題があります。

Docker Compose を使えば、**1つの YAML ファイルに全サービスの構成を記述**し、`docker compose up` の1コマンドですべてを起動できます。

```bash
# docker run を何度も打つ代わりに...
docker compose up -d
# これだけで全サービスが起動します
```

:::message
現在は `docker-compose`（ハイフンあり、V1）ではなく `docker compose`（スペース区切り、V2）を使います。V1 は 2023年6月に EOL を迎えています。
:::

---

## compose.yaml の基本構造

Docker Compose の設定ファイルは `compose.yaml`（または `compose.yml`、`docker-compose.yml`）です。トップレベルで大きく3つのセクションに分かれます。

```yaml
# compose.yaml
services:    # コンテナ（サービス）の定義 ── 必須
  web:
    image: node:20-alpine
  db:
    image: postgres:16-alpine

volumes:     # データ永続化の定義 ── 任意
  pgdata:

networks:    # コンテナ間通信の定義 ── 任意
  backend:
```

:::message
以前は `version: "3"` のようにバージョンを指定していましたが、現在は**省略が推奨**です。Docker Compose V2 が Compose Specification に基づいて自動判定します。
:::

| セクション | 役割 | 必須/任意 |
|-----------|------|----------|
| `services` | コンテナの構成を定義する | 必須 |
| `volumes` | データを永続化するボリュームを定義する | 任意 |
| `networks` | コンテナ間の通信経路を定義する | 任意 |

---

## services ── コンテナを定義する

`services` はファイルの中心となるセクションです。主要なオプションを見ていきましょう。

### イメージの指定とビルド

```yaml
services:
  # 方法1: 既存のイメージを使う
  db:
    image: postgres:16-alpine

  # 方法2: Dockerfile からビルドする
  web:
    build:
      context: .
      dockerfile: Dockerfile
    image: myapp:latest   # ビルド後のタグ名
```

`build: .` のように短縮形で書くこともできます。カレントディレクトリの `Dockerfile` が使われます。

### ポートの公開

ホストマシンからコンテナにアクセスするにはポートマッピングを設定します。

```yaml
services:
  web:
    ports:
      - "3000:3000"            # ホスト:コンテナ
      - "127.0.0.1:8080:8080"  # localhost からのみアクセス可能
```

`"ホスト側ポート:コンテナ側ポート"` の順番です。`127.0.0.1` を付けると外部からのアクセスを防げます。

### 環境変数

アプリケーションの設定値は `environment` で渡します。

```yaml
services:
  web:
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://postgres:postgres@db:5432/myapp
```

環境変数が多い場合は `.env` ファイルに分離できます。

```yaml
services:
  web:
    env_file:
      - .env
      - .env.local   # 後に指定したファイルが優先されます
```

### ボリュームマウント

コンテナ内のデータを永続化したり、ホストのファイルをコンテナに共有する際に使います。

```yaml
services:
  web:
    volumes:
      - ./src:/app/src          # バインドマウント（ホスト → コンテナ）
      - node_modules:/app/node_modules  # 名前付きボリューム
      - ./config:/app/config:ro # 読み取り専用
volumes:
  node_modules:
```

### 再起動ポリシー

```yaml
services:
  web:
    restart: unless-stopped
```

| 値 | 動作 |
|---|------|
| `no` | 再起動しません（デフォルト） |
| `always` | 常に再起動します |
| `on-failure` | 異常終了時のみ再起動します |
| `unless-stopped` | 手動停止以外は再起動します |

---

## networks ── コンテナ間通信を設計する

Docker Compose は起動時にデフォルトのネットワークを自動作成します。同じネットワーク内のコンテナは**サービス名を DNS ホスト名として**互いにアクセスできます。

```yaml
services:
  web:
    environment:
      # "db" というサービス名でデータベースに接続できます
      DATABASE_URL: postgresql://postgres:postgres@db:5432/myapp
  db:
    image: postgres:16-alpine
```

`web` コンテナから `db:5432` で PostgreSQL に接続できます。IP アドレスを覚える必要はありません。

### ネットワークの分離

セキュリティ上の理由でネットワークを分離したい場合はカスタムネットワークを定義します。

```yaml
services:
  nginx:
    networks: [frontend]
  api:
    networks: [frontend, backend]
  db:
    networks: [backend]    # nginx からは直接アクセスできません

networks:
  frontend:
  backend:
    internal: true   # 外部アクセスを遮断します
```

---

## volumes ── データを永続化する

コンテナは削除するとデータが消えます。データベースのデータなど残したい情報はボリュームに保存します。

| 種類 | 特徴 | 主な用途 |
|------|------|---------|
| 名前付きボリューム | Docker が管理。高速で安定 | DB データの永続化 |
| バインドマウント | ホストのディレクトリを共有 | ソースコードの同期（開発時） |
| tmpfs | メモリ上に作成。停止で消える | 一時ファイル・キャッシュ |

```yaml
services:
  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:    # トップレベルで宣言が必要です
```

:::message alert
`docker compose down -v` を実行するとボリュームも削除されます。本番データがある場合は `-v` を付けないよう注意してください。
:::

---

## depends_on ── 起動順序を制御する

複数のサービスがある場合、起動順序が重要になります。`depends_on` で依存関係を定義できます。

### 基本的な使い方

```yaml
services:
  web:
    build: .
    depends_on:
      - db
  db:
    image: postgres:16-alpine
```

ただし、この書き方では**コンテナが起動したことだけ**を確認します。PostgreSQL が接続を受け付けられる状態かどうかはチェックしません。

### ヘルスチェックとの組み合わせ（推奨）

`condition: service_healthy` を指定すると、サービスが準備完了するまで待機できます。

```yaml
services:
  web:
    build: .
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
```

### condition の種類

| condition | 説明 |
|-----------|------|
| `service_started` | コンテナが起動したら（デフォルト） |
| `service_healthy` | ヘルスチェックが通ったら |
| `service_completed_successfully` | コンテナが正常終了（exit 0）したら |

`service_completed_successfully` はマイグレーション処理で便利です。DB が準備完了してからマイグレーションを実行し、それが完了してから Web サーバーを起動する、という流れを宣言的に記述できます。

---

## 基本コマンド

```bash
# 起動と停止
docker compose up -d              # バックグラウンドで全サービス起動
docker compose up -d --build      # ビルドしてから起動
docker compose up -d web db       # 特定のサービスだけ起動
docker compose down               # 停止してコンテナを削除
docker compose down -v            # ボリュームも含めてすべて削除

# ログの確認
docker compose logs               # 全サービスのログ
docker compose logs -f web        # 特定サービスをリアルタイム追跡
docker compose logs --tail 100 web  # 直近100行だけ表示

# 状態確認とコマンド実行
docker compose ps                 # サービスの一覧と状態
docker compose exec web bash      # 起動中のコンテナでコマンド実行
docker compose run --rm web npm test  # 新しいコンテナでコマンド実行
```

---

## 環境変数の管理

プロジェクトルートに `.env` ファイルを置くと、`compose.yaml` 内の `${...}` で参照できます。

```bash
# .env
DB_PASSWORD=secure_password
APP_VERSION=1.2.0
```

```yaml
services:
  web:
    image: myapp:${APP_VERSION:-latest}    # 未設定時は latest
    environment:
      DB_PASSWORD: ${DB_PASSWORD}
      DB_PORT: ${DB_PORT:-5432}            # デフォルト値を設定
      DB_HOST: ${DB_HOST:?DB_HOST is required}  # 未設定ならエラー
```

環境変数は以下の順で上書きされます（上が優先）。

1. `docker compose run -e` で渡した値
2. シェルの環境変数（`export` した値）
3. `compose.yaml` の `environment`
4. `env_file` で指定したファイル
5. Dockerfile の `ENV` 命令

:::message
`.env` ファイルは `compose.yaml` 内の変数展開に使われ、`env_file` はコンテナの環境変数に直接設定されます。両者は役割が異なるので混同しないようにしましょう。
:::

---

## 実践例 ── Web アプリ + DB + Redis

ここまで学んだ要素を組み合わせた構成例です。

```yaml
services:
  app:
    build: .
    ports: ["3000:3000"]
    environment:
      DATABASE_URL: postgresql://postgres:postgres@db:5432/myapp
      REDIS_URL: redis://redis:6379
    volumes: [".:/app", "node_modules:/app/node_modules"]
    depends_on:
      db: { condition: service_healthy }
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: myapp
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    restart: unless-stopped

volumes:
  pgdata:
  node_modules:
```

- `app` は DB のヘルスチェックが通るまで起動を待機します
- DB データは `pgdata` ボリュームで永続化されます
- `node_modules` を名前付きボリュームに分離し、macOS でのパフォーマンスを改善しています

---

## まとめ

| 項目 | 要点 |
|------|------|
| compose.yaml の構造 | `services` / `volumes` / `networks` の3セクションが基本です |
| services | コンテナの定義。`image` / `build` / `ports` / `environment` / `volumes` を設定します |
| networks | デフォルトでサービス名による DNS 解決が有効です。分離が必要なら明示的に定義します |
| volumes | 名前付きボリュームを使い、DB データなどを永続化します |
| depends_on | `condition: service_healthy` とヘルスチェックの組み合わせが推奨です |
| 環境変数 | `.env` ファイルで管理し、compose.yaml へのハードコードは避けます |
| 起動/停止 | `docker compose up -d` で起動、`docker compose down` で停止します |
| ログ確認 | `docker compose logs -f サービス名` でリアルタイムに追跡できます |
| version キー | 省略が推奨です。Compose V2 が自動判定します |
| イメージタグ | `latest` を避け、`postgres:16-alpine` のようにバージョンを明示します |

---

## やってみよう！

以下の課題に取り組んで、Docker Compose の理解を深めましょう。

- [ ] **課題1**: Node.js（または Python）アプリと PostgreSQL の `compose.yaml` を作成し、`docker compose up -d` で起動してみましょう
- [ ] **課題2**: `docker compose logs -f` でログを確認し、DB の起動シーケンスを観察してみましょう
- [ ] **課題3**: `depends_on` に `condition: service_healthy` を追加し、ヘルスチェックなしの場合との違いを比較してみましょう
- [ ] **課題4**: `.env` ファイルにパスワードを定義し、`compose.yaml` 内で `${DB_PASSWORD}` として参照してみましょう
- [ ] **課題5**: `docker compose down` と `docker compose down -v` の違いを確認し、ボリュームの永続化を体験してみましょう
