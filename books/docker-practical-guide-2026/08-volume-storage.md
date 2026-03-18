---
title: "ボリュームとストレージ"
---

## この章で学ぶこと

Docker コンテナは「使い捨て」が前提です。コンテナを削除すると、中で作成・変更されたデータはすべて消えます。データベースのレコードやアップロードファイルなど、コンテナのライフサイクルを超えて保持したいデータには**ボリューム**が必要です。

この章では、3 つのマウント方式（Named Volume / Bind Mount / tmpfs）の使い分け、バックアップとリストア、ボリュームドライバーについて解説します。

## コンテナのデータが消える仕組み

Docker イメージは読み取り専用のレイヤーで構成されています。コンテナ起動時、その上に薄い「書き込み可能レイヤー」が追加されます。コンテナ内での書き込みはすべてこのレイヤーに記録され、コンテナ削除で一緒に消えます。

```
┌──────────────────────────────────────┐
│  Container（書き込み可能レイヤー）      │ ← 削除で消失
├──────────────────────────────────────┤
│  Layer 3: COPY app /app              │
│  Layer 2: RUN npm install            │ ← 読み取り専用
│  Layer 1: node:20-alpine             │
└──────────────────────────────────────┘
```

## ボリュームの 3 つの種類

Docker はデータをコンテナの外に保存するために 3 つのマウント方式を提供しています。

| 特性 | Named Volume | Bind Mount | tmpfs |
|------|-------------|------------|-------|
| 保存先 | Docker 管理領域 | ホスト上の任意パス | メモリ（RAM） |
| Docker CLI で管理 | 可能 | 不可 | 不可 |
| コンテナ間共有 | 容易 | 可能 | 不可 |
| 永続化 | コンテナ削除後も残る | コンテナ削除後も残る | コンテナ停止で消失 |
| パフォーマンス | ドライバー依存 | ネイティブ | 最高速 |
| 本番推奨度 | 高い | 低い（開発向き） | 特殊用途 |

### Named Volume（名前付きボリューム）

Docker エンジンが管理するボリュームです。`/var/lib/docker/volumes/` 配下に保存され、本番環境で最も推奨されます。

```bash
# ボリュームを作成します
docker volume create my-data

# ボリュームをマウントしてコンテナを起動します（-v 構文）
docker run -d \
  --name postgres-db \
  -v my-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:16-alpine

# --mount 構文（より明示的で推奨）
docker run -d \
  --name postgres-db \
  --mount type=volume,source=my-data,target=/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:16-alpine
```

`--mount` 構文は、存在しないパスを指定するとエラーを返してくれるため安全です。`-v` 構文は空のディレクトリを自動作成してしまうため、意図しない動作の原因になることがあります。

コンテナを削除してもボリュームは残ります。

```bash
docker rm -f postgres-db
docker volume ls   # my-data は健在です
```

Docker Compose では、トップレベルの `volumes` キーで宣言します。

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
volumes:
  pgdata:
  redis-data:
```

### Bind Mount（バインドマウント）

ホストの任意のディレクトリをコンテナにマウントする方式です。開発時のソースコード同期に多用されます。

```bash
docker run -d \
  --name dev-server \
  -v $(pwd)/src:/app/src \
  -v $(pwd)/config.yaml:/app/config.yaml:ro \
  my-app:dev
```

末尾の `:ro` は読み取り専用です。設定ファイルなどコンテナから書き換えてほしくないものに付けましょう。

Docker Compose での典型的な開発構成です。

```yaml
services:
  app:
    build: .
    volumes:
      - ./src:/app/src              # ソースコード同期
      - ./config:/app/config:ro     # 設定ファイル（読取専用）
      - node_modules:/app/node_modules  # 依存は Volume で分離
volumes:
  node_modules:
```

macOS では Bind Mount のファイル I/O が遅いため、`node_modules` のような大量のファイルは Named Volume に分離すると `npm install` が 8 倍ほど高速化されます。

### tmpfs マウント

メモリ上にのみ存在する一時ファイルシステムです。ディスクに書き込まれないため、機密データの一時保存やテスト用 DB の高速化に適しています。

```bash
docker run -d \
  --name secure-app \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \
  my-app
```

```yaml
services:
  # テスト用 DB（tmpfs でディスク I/O なし → 高速）
  db-test:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: test
    tmpfs:
      - /var/lib/postgresql/data:size=512m
```

### どの方式を選ぶか

```
データを永続化する必要がある？
├── Yes → ホストの特定パスを指定する必要がある？
│          ├── Yes → Bind Mount（開発時のソースコード同期など）
│          └── No  → Named Volume（DB、アプリデータなど）
└── No  → セキュリティ / パフォーマンスが重要？
           ├── Yes → tmpfs（一時ファイル、シークレットなど）
           └── No  → コンテナの書き込みレイヤーで十分
```

## データ永続化の実践

主要なデータベースでは、データディレクトリに Named Volume を、設定や初期化スクリプトには Bind Mount（読取専用）を使うのが定石です。

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data          # データ → Named Volume
      - ./initdb:/docker-entrypoint-initdb.d:ro  # 初期化 → Bind Mount
    shm_size: "256m"
volumes:
  pgdata:
```

複数のコンテナから同じボリュームをマウントすることもできます。読み取り専用のコンテナには `:ro` を付けましょう。

```yaml
services:
  writer:
    volumes:
      - shared-data:/data        # 書き込み可
  reader:
    volumes:
      - shared-data:/data:ro     # 読み取り専用
volumes:
  shared-data:
```

ただしデータベースのボリュームは、原則として 1 つのコンテナからのみアクセスしてください。

## バックアップとリストア

ボリュームのデータもディスク障害や操作ミスで失われる可能性があります。定期的なバックアップは必須です。

### バックアップ

一時コンテナから tar で圧縮バックアップします。

```bash
docker run --rm \
  -v my-data:/source:ro \
  -v $(pwd)/backups:/backup \
  alpine:3.19 \
  tar czf /backup/my-data-backup.tar.gz -C /source .
```

### リストア

新しいボリュームを作成し、バックアップから復元します。

```bash
docker volume create my-data-restored

docker run --rm \
  -v my-data-restored:/target \
  -v $(pwd)/backups:/backup:ro \
  alpine:3.19 \
  tar xzf /backup/my-data-backup.tar.gz -C /target
```

### 定期バックアップの自動化

Docker Compose でバックアップコンテナを併設する構成です。

```yaml
services:
  postgres:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
  backup:
    image: postgres:16-alpine
    volumes:
      - ./backups:/backups
    environment:
      PGPASSWORD: ${DB_PASSWORD}
    entrypoint: >
      sh -c "
        while true; do
          pg_dump -h postgres -U postgres myapp |
            gzip > /backups/myapp-$$(date +%Y%m%d-%H%M%S).sql.gz
          find /backups -name '*.sql.gz' -mtime +7 -delete
          sleep 86400
        done
      "
    depends_on:
      - postgres
volumes:
  pgdata:
```

7 日以上前のバックアップを自動削除し、バックアップの肥大化を防いでいます。

### ホスト間のボリューム移行

```bash
# 1. バックアップ作成
docker run --rm -v my-data:/source:ro -v $(pwd):/backup \
  alpine:3.19 tar czf /backup/vol-backup.tar.gz -C /source .

# 2. リモートに転送
scp vol-backup.tar.gz user@remote-host:/tmp/

# 3. リモートでリストア
ssh user@remote-host "
  docker volume create my-data &&
  docker run --rm -v my-data:/target -v /tmp:/backup:ro \
    alpine:3.19 tar xzf /backup/vol-backup.tar.gz -C /target
"
```

## ボリュームドライバー

ボリュームドライバーを変更すると、ローカルディスク以外のストレージにデータを保存できます。

### NFS ボリューム

複数の Docker ホスト間でデータを共有する場合に便利です。

```bash
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw,nfsvers=4 \
  --opt device=:/exports/data \
  nfs-data
```

Docker Compose での設定です。

```yaml
volumes:
  shared-data:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=192.168.1.100,rw,nfsvers=4"
      device: ":/exports/data"
```

### ストレージドライバーとの違い

**ボリュームドライバー**は Named Volume のバックエンド（local、NFS など）を制御します。**ストレージドライバー**はイメージレイヤーとコンテナの書き込みレイヤーの仕組み（overlay2 など）を制御します。

```bash
# 現在のストレージドライバーを確認します
docker info | grep "Storage Driver"

# ストレージの使用状況を確認します
docker system df
```

現在は `overlay2` がデフォルトで推奨されます。特別な理由がない限り変更は不要です。

## ボリュームの管理とメンテナンス

```bash
# 未使用ボリュームを一覧表示します
docker volume ls --filter "dangling=true"

# ボリュームごとのサイズを確認します
docker system df -v

# 特定のボリュームを削除します
docker volume rm <ボリューム名>

# 未使用ボリュームを一括削除します（慎重に）
docker volume prune
```

`docker volume prune` は停止中のコンテナが使っていたボリュームも削除対象になります。まず一覧で確認してから、個別に `docker volume rm` する方が安全です。

ラベルを付けておくと、環境やサービスごとに管理しやすくなります。

```bash
docker volume create --label environment=production --label service=postgres prod-pgdata
docker volume ls --filter "label=environment=production"
```

## まとめ

| 項目 | ポイント |
|------|---------|
| Named Volume | Docker 管理。本番推奨。ポータブルで移行しやすい |
| Bind Mount | ホストの任意パスをマウント。開発時のソースコード同期向き |
| tmpfs | メモリ上に存在。機密データの一時保存やテスト DB の高速化に最適 |
| `-v` vs `--mount` | `--mount` が明示的で安全。存在しないパスでエラーを返す |
| バックアップ | alpine + tar で実施。定期バックアップの自動化を推奨 |
| リストア | 新ボリューム作成 → tar で展開。scp でホスト間移行も可能 |
| ボリュームドライバー | local がデフォルト。NFS も `driver_opts` で設定可能 |
| メンテナンス | `docker system df -v` で定期確認。`prune` は慎重に |

## やってみよう！

1. **Named Volume の基本操作**: `docker volume create test-vol` でボリュームを作成し、PostgreSQL コンテナにマウントしてデータを投入してください。コンテナを削除・再作成して、データが残っていることを確認しましょう。

2. **Bind Mount で開発環境を構築**: 適当なディレクトリに `index.html` を作り、Nginx コンテナに Bind Mount してブラウザで表示してみましょう。ホスト側でファイルを編集するとリロードで即反映されます。
   ```bash
   echo "<h1>Hello Docker!</h1>" > index.html
   docker run -d -p 8080:80 \
     -v $(pwd)/index.html:/usr/share/nginx/html/index.html:ro \
     nginx:alpine
   ```

3. **バックアップとリストア**: 演習 1 で作成した PostgreSQL のボリュームをバックアップし、別名のボリュームにリストアしてください。新しいコンテナで起動し、データが復元されていることを確認しましょう。

4. **tmpfs でテスト DB を高速化**: `docker-compose.yml` に tmpfs を使ったテスト用 PostgreSQL を定義し、通常のボリュームの場合と起動・書き込み速度を比較してみましょう。
