---
title: "Dockerネットワーク"
---

## この章で学ぶこと

Dockerコンテナは、それぞれが独立したネットワーク空間を持っています。この章では、コンテナ同士やホスト・外部との通信を制御する**Dockerネットワーキング**の仕組みを学びます。

- 4つのネットワークドライバー（bridge / host / none / overlay）の違いと使い分け
- ユーザー定義ネットワークの作成とコンテナの接続
- コンテナ間通信とDNSによるサービスディスカバリ
- ポートマッピングによる外部公開

---

## ネットワークドライバーの全体像

Dockerは複数のネットワークドライバーを提供しています。まずは全体像を把握しましょう。

| ドライバー | スコープ | 主な用途 | IPアドレス |
|-----------|---------|---------|-----------|
| **bridge** | 単一ホスト | 開発環境・一般用途（デフォルト） | 自動割当（172.17.x.x） |
| **host** | 単一ホスト | 最大パフォーマンスが必要な場面 | ホストと共有 |
| **none** | - | ネットワークを完全に無効化 | なし |
| **overlay** | マルチホスト | Docker Swarm / 本番クラスタ | 自動割当（10.0.x.x） |

どのドライバーを使うか迷ったら、以下のフローで判断できます。

```
ネットワークが必要か？
  ├── No → none
  └── Yes → マルチホスト通信が必要か？
              ├── Yes → overlay（Swarm / K8s）
              └── No → 最大パフォーマンスが必要か？
                         ├── Yes → host（Linuxのみ完全対応）
                         └── No → bridge（推奨）
```

---

## bridgeネットワーク

### デフォルトbridgeとユーザー定義bridgeの違い

Dockerインストール時に自動作成される `docker0`（デフォルトbridge）と、自分で作成するユーザー定義bridgeには大きな違いがあります。

| 特性 | デフォルトbridge | ユーザー定義bridge |
|-----|-----------------|------------------|
| DNS解決 | 不可（IPアドレスのみ） | コンテナ名で解決可能 |
| ネットワーク分離 | 分離なし | ネットワーク単位で分離 |
| ライブ接続/切断 | 不可 | 可能 |
| カスタムサブネット | 不可 | 可能 |

**実務では必ずユーザー定義bridgeを使いましょう。** デフォルトbridgeではコンテナ名でのDNS解決ができないため、不便でセキュリティ上も望ましくありません。

### ユーザー定義bridgeの作成と接続

```bash
# ネットワークを作成します
docker network create \
  --driver bridge \
  --subnet 192.168.100.0/24 \
  --gateway 192.168.100.1 \
  my-app-network

# コンテナをネットワークに接続して起動します
docker run -d --name web-server --network my-app-network nginx:alpine
docker run -d --name api-server --network my-app-network node:20-alpine sleep infinity

# web-server から api-server にコンテナ名で通信できます
docker exec web-server ping -c 3 api-server

# 既存コンテナをあとからネットワークに接続・切断できます
docker network connect my-app-network existing-container
docker network disconnect my-app-network existing-container
```

### Docker Composeでのネットワーク定義

```yaml
services:
  frontend:
    image: nginx:alpine
    networks:
      - frontend-net
    ports:
      - "80:80"

  api:
    build: ./api
    networks:
      - frontend-net
      - backend-net
    environment:
      DB_HOST: database  # サービス名がDNS名になります

  database:
    image: postgres:16-alpine
    networks:
      - backend-net

networks:
  frontend-net:
    driver: bridge
  backend-net:
    driver: bridge
    internal: true  # 外部への通信を遮断します
```

この構成では `frontend` から `database` への直接通信はできません。`api` だけが両方のネットワークに参加しているため、層ごとのアクセス制御が実現できます。

```
外部（インターネット）
    │ :80
    ▼
 frontend ── frontend-net
                 │
               api ── backend-net（internal: true）
                          │
                      database（外部アクセス不可）
```

---

## hostネットワーク

`host` ドライバーを使うと、コンテナはホストマシンのネットワーク名前空間を直接共有します。NAT変換が不要なため、最もパフォーマンスが高い方式です。

```bash
# hostネットワークでNginxを起動します
docker run -d --network host --name web nginx:alpine

# ポートマッピング（-p）は不要です
curl http://localhost:80
```

Docker Composeでは `network_mode: host` を指定します。

```yaml
services:
  high-perf-app:
    image: my-app:latest
    network_mode: host
```

:::message alert
hostネットワークは **Linuxでのみ完全にサポート** されています。macOS / WindowsではDocker DesktopのVM内でのhost共有となるため、ホストOSから直接アクセスできません。
:::

---

## noneネットワーク

`none` ドライバーはコンテナのネットワークを完全に無効化します。ループバック（`lo`）のみ利用可能です。

```bash
docker run -d --network none --name isolated-app alpine sleep infinity

# ネットワークインターフェースを確認します（loのみ）
docker exec isolated-app ip addr
```

機密計算の隔離、外部通信不要なバッチ処理、ネットワーク障害テストなどの場面で使います。

---

## overlayネットワーク

`overlay` ドライバーは、複数のDockerホストにまたがるネットワークを構築します。Docker SwarmやKubernetesで使われます。

```
┌──── Host A ────┐    ┌──── Host B ────┐
│  Web-1  Web-2  │    │  API-1  DB-1   │
│     │     │    │    │    │     │     │
│  overlay: my-overlay ◄──► overlay     │
│  (VXLAN トンネル)  │    │  (VXLAN)     │
│       │        │    │       │        │
│     eth0       │    │     eth0       │
└───────┼────────┘    └───────┼────────┘
        └── UDP 4789 (VXLAN) ─┘
```

```bash
# Swarmを初期化します（マネージャーノード）
docker swarm init --advertise-addr 192.168.1.10

# 暗号化付きオーバーレイネットワークを作成します
docker network create \
  --driver overlay \
  --subnet 10.10.0.0/16 \
  --opt encrypted \
  --attachable \
  my-overlay

# サービスをデプロイします
docker service create \
  --name web --network my-overlay \
  --replicas 3 --publish published=80,target=80 \
  nginx:alpine
```

:::message
overlayネットワークではVXLANカプセル化のオーバーヘッドが5-10%程度あります。`--opt encrypted` を有効にするとIPsec暗号化が加わり、CPU負荷がさらに増加します。
:::

---

## ポートマッピング

コンテナ内のサービスを外部に公開するには、ポートマッピングを使います。

```bash
# ホストの8080番をコンテナの80番にマッピングします
docker run -d -p 8080:80 nginx:alpine

# localhostのみに公開します（セキュリティ推奨）
docker run -d -p 127.0.0.1:8080:80 nginx:alpine

# UDPポートを公開します
docker run -d -p 5353:53/udp dns-server

# 複数ポートを公開します
docker run -d -p 80:80 -p 443:443 nginx:alpine

# ランダムポートを割り当てます
docker run -d -p 80 nginx:alpine
docker port <container>  # 割り当てポートを確認
```

### Docker Composeでのポートマッピング

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
      - "127.0.0.1:8080:80"      # localhostのみ

  database:
    image: postgres:16-alpine
    ports:
      - "127.0.0.1:5432:5432"    # DBは必ずlocalhostのみ
    expose:
      - "5432"                    # コンテナ間通信のみ
```

:::message alert
データベースのポートを `0.0.0.0` に公開するのは危険です。`127.0.0.1` にバインドして外部アクセスを防ぎましょう。
:::

`.env` ファイルでポート番号を変数化すると、環境ごとの競合を防げます。

```yaml
services:
  app:
    ports:
      - "${APP_PORT:-3000}:3000"
  db:
    ports:
      - "127.0.0.1:${DB_PORT:-5432}:5432"
```

---

## コンテナ間通信とDNS解決

### Docker内蔵DNSの仕組み

ユーザー定義ネットワーク内では、Docker内蔵DNSサーバー（`127.0.0.11`）がコンテナ名をIPアドレスに自動変換してくれます。

```
コンテナA が "api-server" を名前解決する流れ

1. /etc/resolv.conf → nameserver 127.0.0.11
2. Docker内蔵DNS に問い合わせ
   → 見つかった → 172.20.0.3（api-serverのIP）
   → 見つからない → ホストのDNS（8.8.8.8等）に転送
```

### DNS解決の優先順位

1. **コンテナ名** (`--name` で指定した名前)
2. **ネットワークエイリアス** (`--network-alias` や `networks.aliases`)
3. **Composeサービス名** (docker-compose.yml の `services` キー)
4. **外部DNS** (ホストのresolv.confに従う)

### エイリアスによるサービスディスカバリ

エイリアスを使うと、1つのDNS名に複数のコンテナを紐づけてラウンドロビンで分散できます。

```yaml
services:
  cache-primary:
    image: redis:7-alpine
    networks:
      app-net:
        aliases:
          - redis
  cache-replica:
    image: redis:7-alpine
    networks:
      app-net:
        aliases:
          - redis  # 同一エイリアスでラウンドロビンDNS
networks:
  app-net:
    driver: bridge
```

```bash
# DNS解決を確認します（両方のIPが返ります）
docker run --rm --network app-net busybox nslookup redis
```

### コンテナからホストマシンへのアクセス

Docker Desktop（macOS / Windows）では `host.docker.internal` が使えます。Linuxでは `extra_hosts` で設定します。

```yaml
services:
  app:
    extra_hosts:
      - "host.docker.internal:host-gateway"
    environment:
      HOST_API: http://host.docker.internal:4000
```

---

## ネットワーク設計のベストプラクティス

実際のプロジェクトでは、層ごとにネットワークを分離しましょう。以下は典型的な3層構成の例です。

| サービス | public | app-tier | data-tier |
|---------|--------|----------|-----------|
| nginx | ○ | ○ | - |
| api | - | ○ | ○ |
| database | - | - | ○ |
| redis | - | - | ○ |

`data-tier` に `internal: true` を設定すれば、たとえnginxが侵害されてもデータベースに直接到達できません。ネットワーク分離はセキュリティの基本防御策です。

---

## トラブルシューティング

```bash
# ネットワーク一覧を確認します
docker network ls

# 特定ネットワークの詳細を確認します
docker network inspect my-app-network

# コンテナのIPアドレスを取得します
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-container

# netshootコンテナで本格的な診断を行います
docker run -it --rm --network my-app-network nicolaka/netshoot
```

### よくある問題と解決策

| 問題 | 原因 | 解決策 |
|------|------|--------|
| コンテナ間で名前解決できない | デフォルトbridgeを使用 | ユーザー定義ネットワークに変更する |
| ポートが既に使用中 | ホスト側でポート競合 | `docker ps` で確認し停止またはポート変更 |
| コンテナから外部に通信できない | DNS設定の問題 | `dns:` オプションで外部DNSを指定する |
| ポートフォワードが動かない | iptablesルールの問題 | `sudo iptables -t nat -L -n` で確認 |
| IPアドレスが変わって通信が切れる | IPハードコード | DNS名（サービス名）を使用する |

---

## まとめ

| 項目 | ポイント |
|------|---------|
| bridge | 単一ホストのデフォルト。ユーザー定義bridgeを必ず使いましょう |
| host | NATなしで最高パフォーマンス。Linuxのみ完全対応です |
| none | ネットワークを完全に無効化します。隔離が必要な場面で使います |
| overlay | マルチホスト通信を実現します。Swarm / K8sで使用します |
| ポートマッピング | `-p ホスト:コンテナ` で外部に公開。`127.0.0.1` バインド推奨です |
| DNS解決 | ユーザー定義ネットワーク内でコンテナ名が自動的にIPに変換されます |
| コンテナ間通信 | 同一ネットワーク内のコンテナは名前で相互に通信できます |
| ネットワーク分離 | `internal: true` で外部遮断。層ごとの分離がセキュリティの基本です |

## やってみよう！

1. **ユーザー定義bridgeの作成**: `docker network create` でネットワークを作り、2つのコンテナ（nginx と alpine）を接続して、コンテナ名で `ping` が通ることを確認してください

2. **Composeでのネットワーク分離**: frontend / backend の2つのネットワークを定義し、frontendからdatabaseへ直接アクセスできないことを確認してください

3. **ポートマッピングの検証**: Nginxコンテナを `127.0.0.1:8080:80` で起動し、`curl http://localhost:8080` でアクセスできることを確認してください

4. **DNS解決の確認**: `docker run --rm --network <your-network> busybox nslookup <container-name>` を実行して、内蔵DNSの動作を観察してください

5. **トラブルシューティング実践**: デフォルトbridgeで2つのコンテナを起動し、コンテナ名での `ping` が失敗することを確認してから、ユーザー定義ネットワークに切り替えて成功させてください
