---
title: "Dockerコマンド操作"
---

この章では、Docker を使った日常的な操作を体系的に学びます。イメージの取得からコンテナの起動・管理、そしてレジストリへの配布まで、実務で必要となるコマンドを一通り押さえていきます。

## イメージ操作の基本

Docker を使うには、まず**イメージ**を手元に用意する必要があります。イメージはコンテナの「設計図」にあたるもので、アプリケーションの実行に必要なファイルや設定がすべて含まれています。

### docker pull — イメージの取得

Docker Hub などのレジストリからイメージをダウンロードするコマンドです。

```bash
# 最新バージョンを取得（:latest が暗黙的に指定されます）
docker pull nginx

# バージョンを明示的に指定
docker pull nginx:1.25.3-alpine

# 特定のプラットフォーム向けに取得（Apple Silicon Mac で amd64 が必要な場合など）
docker pull --platform linux/amd64 nginx:alpine
```

`docker pull` はレイヤー単位でダウンロードを行います。すでにローカルに存在するレイヤーはスキップされるため、2 回目以降は差分だけの高速なダウンロードになります。

```bash
# 出力例
# 1.25.3-alpine: Pulling from library/nginx
# 4abcdefgh123: Already exists    ← キャッシュ済みレイヤー
# 7hijklmno456: Pull complete     ← 新規ダウンロード
# Digest: sha256:abc123...
# Status: Downloaded newer image for nginx:1.25.3-alpine
```

:::message
**ポイント**: 本番環境では `latest` タグの使用は避けましょう。どのバージョンが動いているか分からなくなり、トラブルシューティングが困難になります。`nginx:1.25.3-alpine` のようにバージョンを固定するのがベストプラクティスです。
:::

### docker build — イメージのビルド

`Dockerfile` からオリジナルのイメージを作成するコマンドです。

```bash
# カレントディレクトリの Dockerfile からビルド
docker build -t my-app:v1.0.0 .

# ビルド引数を渡す
docker build --build-arg NODE_ENV=production -t my-app:v1.0.0 .

# 別のファイルを Dockerfile として指定
docker build -f Dockerfile.prod -t my-app:v1.0.0 .

# キャッシュを使わずにフルビルド
docker build --no-cache -t my-app:v1.0.0 .
```

`-t` オプションで「リポジトリ名:タグ」の形式で名前を付けます。`.` はビルドコンテキスト（Dockerfile から参照できるファイル群のルートディレクトリ）を指定しています。

### docker tag — タグの付与

既存のイメージに別名（タグ）を付けるコマンドです。イメージのコピーは発生せず、同じイメージを異なる名前で参照できるようになります。

```bash
# ローカルでの別名付与
docker tag my-app:v1.0.0 my-app:latest

# レジストリへの push 用にタグ付け
docker tag my-app:v1.0.0 ghcr.io/myuser/my-app:v1.0.0

# 複数のタグを付けるのが一般的です
docker tag my-app:v1.0.0 my-app:v1.0
docker tag my-app:v1.0.0 my-app:v1
```

### docker push — イメージの配布

ローカルのイメージをレジストリにアップロードするコマンドです。事前にレジストリへのログインが必要です。

```bash
# Docker Hub にログイン
docker login

# イメージをプッシュ
docker push myuser/my-app:v1.0.0

# GitHub Container Registry の場合
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin
docker push ghcr.io/myuser/my-app:v1.0.0

# ログアウト
docker logout
```

### イメージの一覧・削除

```bash
# ローカルのイメージ一覧
docker images

# 出力例
# REPOSITORY   TAG             IMAGE ID       CREATED        SIZE
# my-app       v1.0.0          a1b2c3d4e5f6   2 hours ago    150MB
# nginx        1.25.3-alpine   f6e5d4c3b2a1   3 days ago     42.6MB

# 特定のイメージを削除
docker rmi nginx:1.25.3-alpine

# タグなし（dangling）イメージを一括削除
docker image prune

# 未使用の全イメージを削除
docker image prune -a
```

---

## コンテナ操作の実践

イメージを取得したら、そこからコンテナを起動して操作していきます。

### docker run — コンテナの起動

```bash
# 基本的な起動
docker run nginx

# バックグラウンドで起動（-d: デタッチモード）
docker run -d --name my-nginx nginx:1.25.3-alpine

# ポートマッピング付きで起動（ホスト:コンテナ）
docker run -d -p 8080:80 --name web nginx:1.25.3-alpine

# 環境変数を渡して起動
docker run -d -e POSTGRES_PASSWORD=mysecret -p 5432:5432 postgres:16-alpine

# ボリュームマウント付きで起動
docker run -d -v $(pwd)/html:/usr/share/nginx/html -p 8080:80 nginx

# 対話モードで起動してシェルに入る
docker run -it --rm ubuntu:22.04 /bin/bash
```

よく使うオプションをまとめます。

| オプション | 意味 | 例 |
|---|---|---|
| `-d` | バックグラウンド実行 | `docker run -d nginx` |
| `-p` | ポートマッピング | `-p 8080:80` |
| `-e` | 環境変数の設定 | `-e NODE_ENV=production` |
| `-v` | ボリュームマウント | `-v ./data:/app/data` |
| `--name` | コンテナ名の指定 | `--name my-app` |
| `--rm` | 終了時に自動削除 | `docker run --rm alpine echo hello` |
| `-it` | 対話モード（TTY 割り当て） | `docker run -it ubuntu bash` |

### docker ps — コンテナの一覧表示

```bash
# 実行中のコンテナを表示
docker ps

# 出力例
# CONTAINER ID   IMAGE                  COMMAND                  STATUS         PORTS                  NAMES
# a1b2c3d4e5f6   nginx:1.25.3-alpine    "/docker-entrypoint.…"   Up 5 minutes   0.0.0.0:8080->80/tcp   web

# 停止中を含むすべてのコンテナを表示
docker ps -a

# コンテナ ID のみ表示（スクリプトで便利）
docker ps -q

# フォーマットを指定して表示
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
```

### docker logs — ログの確認

コンテナの標準出力・標準エラー出力を確認できます。トラブルシューティングで最もよく使うコマンドの1つです。

```bash
# コンテナのログを表示
docker logs web

# 末尾の 50 行のみ表示
docker logs --tail 50 web

# リアルタイムでログを監視（tail -f のように）
docker logs -f web

# タイムスタンプ付きで表示
docker logs -t web

# 特定の時間以降のログを表示
docker logs --since 2026-03-18T10:00:00 web

# 直近 30 分のログを表示
docker logs --since 30m web
```

:::message
**ポイント**: `docker logs -f` でリアルタイム監視しているときは `Ctrl + C` で抜けられます。コンテナ自体は停止しません。
:::

### docker exec — 起動中のコンテナでコマンド実行

稼働中のコンテナの内部に入ってデバッグしたい場面で活躍します。

```bash
# コンテナ内でシェルを起動
docker exec -it web /bin/sh

# コンテナ内で単発コマンドを実行
docker exec web cat /etc/nginx/nginx.conf

# 環境変数を確認
docker exec web env

# プロセス一覧を確認
docker exec web ps aux

# root 以外のユーザーで実行
docker exec -u www-data web whoami
```

### docker inspect — 詳細情報の確認

コンテナやイメージの内部情報を JSON 形式で取得します。ネットワーク設定やマウント情報の確認に便利です。

```bash
# コンテナの全情報を表示
docker inspect web

# IP アドレスを取得
docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web

# マウント情報を取得
docker inspect --format '{{json .Mounts}}' web | python3 -m json.tool

# 環境変数の一覧を取得
docker inspect --format '{{json .Config.Env}}' web | python3 -m json.tool

# イメージの公開ポートを確認
docker inspect --format '{{json .Config.ExposedPorts}}' nginx:alpine
```

### コンテナのライフサイクル管理

```bash
# コンテナの停止
docker stop web

# コンテナの再起動
docker restart web

# 停止したコンテナの起動
docker start web

# コンテナの強制停止（SIGKILL）
docker kill web

# コンテナの削除（停止済みのもの）
docker rm web

# 停止と削除を同時に
docker rm -f web

# 停止中のコンテナを一括削除
docker container prune
```

---

## イメージ管理の実践

日々の開発で蓄積されるイメージを適切に管理する方法を見ていきます。

### ディスク使用量の確認とクリーンアップ

```bash
# Docker 全体のディスク使用量を確認
docker system df

# 出力例
# TYPE            TOTAL   ACTIVE   SIZE      RECLAIMABLE
# Images          15      3        4.2GB     3.1GB (73%)
# Containers      8       2        120MB     95MB (79%)
# Local Volumes   5       2        800MB     300MB (37%)
# Build Cache     45      0        1.5GB     1.5GB (100%)

# 未使用のリソースを一括削除
docker system prune

# ボリュームも含めて完全クリーンアップ（注意して実行してください）
docker system prune -a --volumes
```

### イメージの保存と読み込み

レジストリを使えない環境（オフライン環境やセキュリティ上の制約がある環境）でイメージを受け渡す方法です。

```bash
# イメージをファイルに保存
docker save -o my-app-v1.tar my-app:v1.0.0

# 圧縮して保存（サイズ削減）
docker save my-app:v1.0.0 | gzip > my-app-v1.tar.gz

# ファイルからイメージを読み込み
docker load -i my-app-v1.tar

# 圧縮ファイルから読み込み
gunzip -c my-app-v1.tar.gz | docker load
```

### イメージの履歴を確認

イメージがどのように構築されたかを確認できます。Dockerfile の各命令に対応するレイヤーが表示されます。

```bash
# レイヤー履歴の表示
docker history nginx:alpine

# 出力例
# IMAGE          CREATED       CREATED BY                                      SIZE
# a1b2c3d4       2 days ago    CMD ["nginx" "-g" "daemon off;"]                0B
# <missing>      2 days ago    EXPOSE map[80/tcp:{}]                           0B
# <missing>      2 days ago    RUN /bin/sh -c set -x && addgroup...            26.7MB
```

---

## 実践シナリオ：Web アプリをビルドしてデプロイ

ここまで学んだコマンドを組み合わせて、実際の開発フローを体験してみましょう。

```bash
# 1. アプリケーションのイメージをビルド
docker build -t my-web-app:v1.0.0 .

# 2. ローカルで動作確認
docker run -d --name test-app -p 3000:3000 my-web-app:v1.0.0

# 3. ログを確認して正常起動を確認
docker logs test-app

# 4. コンテナ内部を確認
docker exec -it test-app /bin/sh

# 5. 問題なければレジストリ用にタグ付け
docker tag my-web-app:v1.0.0 ghcr.io/myuser/my-web-app:v1.0.0

# 6. レジストリにプッシュ
docker push ghcr.io/myuser/my-web-app:v1.0.0

# 7. テスト用コンテナを片付け
docker rm -f test-app
```

---

## まとめ

この章で扱った主要コマンドを整理します。

| カテゴリ | コマンド | 用途 |
|---|---|---|
| イメージ取得 | `docker pull` | レジストリからイメージをダウンロード |
| イメージビルド | `docker build` | Dockerfile からイメージを作成 |
| タグ付け | `docker tag` | イメージに別名を付与 |
| イメージ配布 | `docker push` | レジストリにイメージをアップロード |
| イメージ一覧 | `docker images` | ローカルのイメージを一覧表示 |
| イメージ削除 | `docker rmi` | ローカルのイメージを削除 |
| コンテナ起動 | `docker run` | イメージからコンテナを起動 |
| コンテナ一覧 | `docker ps` | 実行中のコンテナを一覧表示 |
| ログ確認 | `docker logs` | コンテナのログを表示 |
| コマンド実行 | `docker exec` | 起動中のコンテナでコマンドを実行 |
| 詳細情報 | `docker inspect` | コンテナやイメージの詳細情報を表示 |
| コンテナ停止 | `docker stop` | コンテナを停止 |
| コンテナ削除 | `docker rm` | コンテナを削除 |
| ディスク確認 | `docker system df` | Docker のディスク使用量を確認 |
| 一括整理 | `docker system prune` | 未使用リソースを一括削除 |

---

## やってみよう！

以下の演習で、この章の内容を実際に手を動かして確認してみましょう。

1. **イメージの取得と確認**: `nginx:alpine` と `nginx:latest` を pull して、`docker images` でサイズの違いを比較してみましょう。Alpine ベースのイメージがどれだけ軽量か実感できます。

2. **コンテナの起動とアクセス**: `docker run -d -p 8080:80 --name my-nginx nginx:alpine` で Nginx を起動し、ブラウザで `http://localhost:8080` にアクセスしてみましょう。

3. **ログとデバッグ**: 起動した Nginx コンテナに対して `docker logs my-nginx` でログを確認し、`docker exec -it my-nginx /bin/sh` でコンテナ内部に入ってファイル構成を調べてみましょう。

4. **inspect で情報取得**: `docker inspect my-nginx` を実行し、IP アドレスやマウント情報を確認してみましょう。`--format` オプションで特定の情報だけ抜き出す練習もしてみてください。

5. **クリーンアップ**: 演習が終わったら `docker rm -f my-nginx` でコンテナを削除し、`docker system df` でディスク使用量を確認してみましょう。`docker system prune` で不要なリソースを掃除するところまでやってみてください。
