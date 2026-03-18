---
title: "Dockerfileの書き方"
---

## Dockerfile とは

Dockerfile は、Docker イメージを構築するための設計書です。テキストファイルに「どのベースイメージを使うか」「どのファイルをコピーするか」「どのコマンドを実行するか」を記述することで、誰でも同じ環境を再現できます。

Infrastructure as Code（IaC）の考え方に基づき、インフラの構成をコードとしてバージョン管理できる点が最大のメリットです。

```dockerfile
# もっともシンプルな Dockerfile の例
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

このファイルをビルドして実行するには、次のコマンドを使います。

```bash
# イメージのビルド
docker build -t my-app:v1.0 .

# コンテナの起動
docker run -d -p 3000:3000 my-app:v1.0
```

---

## 主要な命令を覚えよう

Dockerfile には多くの命令がありますが、まずは以下の 9 つを押さえれば実用的な Dockerfile を書けます。

### FROM -- ベースイメージの指定

すべての Dockerfile は `FROM` から始まります。どの OS やランタイムをベースにするかを決める命令です。

```dockerfile
# 公式の Node.js イメージ（Alpine ベースで軽量）
FROM node:20-alpine

# Python の slim バリアント
FROM python:3.12-slim

# Go の静的バイナリ用に空のベース
FROM scratch
```

ベースイメージの選び方は、用途によって異なります。

| バリアント | サイズ目安 | 特徴 | 用途 |
|---|---|---|---|
| `scratch` | 0 MB | 完全に空 | Go/Rust の静的バイナリ |
| `alpine` | 約 7 MB | musl libc、apk | 軽量な汎用イメージ |
| `slim` | 約 74 MB | glibc、apt | 互換性重視 |
| 通常版 | 約 200 MB+ | 開発ツール豊富 | 開発・デバッグ用 |

本番環境では `alpine` や `slim` を選ぶのが一般的です。

### RUN -- ビルド時にコマンドを実行

パッケージのインストールやファイルの生成など、ビルド時に実行したいコマンドを記述します。

```dockerfile
# パッケージのインストール（推奨パターン）
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        ca-certificates && \
    rm -rf /var/lib/apt/lists/*
```

**ポイント:** 複数のコマンドは `&&` で繋いで 1 つの `RUN` にまとめます。理由は後述する「レイヤーキャッシュ」に関係します。

### COPY -- ファイルをイメージにコピー

ローカルのファイルやディレクトリをイメージ内にコピーします。

```dockerfile
# 特定のファイルをコピー
COPY package.json /app/

# ディレクトリごとコピー
COPY src/ /app/src/

# オーナーを指定してコピー
COPY --chown=node:node . /app/
```

似た命令に `ADD` がありますが、`ADD` は tar ファイルの自動展開や URL からのダウンロードといった暗黙的な動作があるため、通常は `COPY` を使います。`ADD` は tar の展開が必要な場合のみ使いましょう。

### WORKDIR -- 作業ディレクトリの設定

以降の `RUN`、`COPY`、`CMD` などが実行される作業ディレクトリを指定します。

```dockerfile
WORKDIR /app

# 存在しないディレクトリは自動作成されます
WORKDIR /app/src/components
```

`RUN cd /app` と書いても次の `RUN` 命令では元に戻ってしまいます。必ず `WORKDIR` を使ってください。

### CMD -- コンテナ起動時のデフォルトコマンド

コンテナが起動したときに実行されるコマンドを指定します。`docker run` の引数で上書きできます。

```dockerfile
# exec 形式（推奨）
CMD ["node", "server.js"]

# docker run my-app            => node server.js
# docker run my-app bash       => bash（CMD が上書きされる）
```

必ず **exec 形式**（JSON 配列）で書きましょう。シェル形式 `CMD node server.js` だと `/bin/sh -c` 経由で実行されるため、SIGTERM などのシグナルがアプリに届かず、グレースフルシャットダウンに失敗する場合があります。

### ENTRYPOINT -- 固定のメインコマンド

`CMD` との違いは、`docker run` の引数で簡単に上書きできない点です。`CMD` と組み合わせるのが一般的なパターンです。

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]

# docker run my-app            => python app.py
# docker run my-app test.py    => python test.py（CMD 部分だけ上書き）
```

| | CMD | ENTRYPOINT |
|---|---|---|
| 役割 | デフォルト引数 | 固定のメインコマンド |
| 上書き方法 | `docker run` の引数 | `--entrypoint` フラグ |
| 典型的な用途 | アプリ起動コマンド | CLI ツール、ラッパースクリプト |

### EXPOSE -- ポートの宣言

コンテナが使用するポートをドキュメントとして宣言します。実際のポート公開は `docker run -p` で行うため、`EXPOSE` だけではホストからアクセスできません。

```dockerfile
EXPOSE 3000
EXPOSE 8080/tcp
EXPOSE 53/udp
```

書かなくても動作しますが、どのポートを使うかを明示するために記載することを推奨します。

### ENV -- 環境変数の設定

ビルド時と実行時の両方で有効な環境変数を定義します。

```dockerfile
ENV NODE_ENV=production
ENV APP_PORT=3000

# 後の命令で参照可能
ENV APP_HOME=/app
WORKDIR $APP_HOME
```

`docker run -e NODE_ENV=development` のように実行時に上書きすることもできます。

**注意:** パスワードなどの機密情報を `ENV` に書いてはいけません。`docker inspect` で誰でも確認できてしまいます。

### ARG -- ビルド時のみ有効な変数

`ENV` と似ていますが、ビルド時にしか使えません。最終イメージには残りません。

```dockerfile
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine

# FROM の後に再宣言が必要（スコープがリセットされるため）
ARG APP_ENV=production
RUN echo "Building for ${APP_ENV}"
```

```bash
# ビルド時に値を渡す
docker build --build-arg APP_ENV=staging -t my-app .
```

| | ENV | ARG |
|---|---|---|
| 有効期間 | ビルド時 + 実行時 | ビルド時のみ |
| 上書き方法 | `docker run -e` | `docker build --build-arg` |
| イメージに残るか | 残る | 残らない |
| 機密情報 | 不可（inspect で見える） | 不可（history で見える場合あり） |

機密情報にはどちらも使わず、BuildKit の `--mount=type=secret` を利用してください。

---

## .dockerignore でビルドを最適化する

`docker build` を実行すると、指定したディレクトリ（ビルドコンテキスト）の中身がすべて Docker デーモンに送信されます。`node_modules` や `.git` が含まれると、ビルドが遅くなるだけでなく、不要なファイルがイメージに紛れ込むリスクもあります。

`.dockerignore` ファイルをプロジェクトルートに置くことで、不要なファイルを除外できます。

```bash
# .dockerignore

# バージョン管理
.git
.gitignore

# 依存関係（コンテナ内で再インストールするため）
node_modules
__pycache__

# 環境設定・機密情報
.env
.env.*
*.pem
*.key

# ビルド成果物
dist
build
coverage

# IDE / エディタ
.vscode
.idea

# Docker 関連ファイル自体
Dockerfile
docker-compose*.yml
.dockerignore

# OS 生成ファイル
.DS_Store
Thumbs.db
```

`.dockerignore` がない場合と比較すると、ビルドコンテキストのサイズが数百 MB から数 KB に減ることも珍しくありません。

---

## レイヤーキャッシュの仕組み

Dockerfile を効率的に書くうえで、レイヤーキャッシュの理解は欠かせません。

### レイヤーとは

Dockerfile の `FROM`、`RUN`、`COPY`、`ADD` はそれぞれ新しい **レイヤー** を生成します。一方、`WORKDIR`、`ENV`、`CMD`、`EXPOSE` などはメタデータの変更のみで、レイヤーは生成しません。

```
FROM node:20-alpine       => ベースレイヤー
WORKDIR /app              => メタデータのみ（レイヤーなし）
COPY package.json .       => Layer A
RUN npm ci                => Layer B
COPY . .                  => Layer C
EXPOSE 3000               => メタデータのみ（レイヤーなし）
CMD ["node", "server.js"] => メタデータのみ（レイヤーなし）
```

### キャッシュが効く仕組み

Docker は 2 回目以降のビルドで、各レイヤーに変更がなければキャッシュを再利用します。ただし、**あるレイヤーのキャッシュが無効になると、それ以降のすべてのレイヤーも再ビルドされます**。

```
2 回目のビルド（ソースコードのみ変更した場合）:

FROM node:20-alpine       => キャッシュ利用
COPY package.json .       => キャッシュ利用（変更なし）
RUN npm ci                => キャッシュ利用（変更なし）
COPY . .                  => 再実行（ソースが変わったため）
```

この仕組みを活かすために、**変更頻度の低いものを先に、高いものを後に** 配置します。

### よくある失敗パターン

```dockerfile
# NG: すべてのファイルを先にコピーしてからインストール
FROM node:20-alpine
WORKDIR /app
COPY . .                       # ソース変更で全レイヤーが無効に
RUN npm ci --only=production   # 毎回 npm install が走ってしまう
CMD ["node", "server.js"]

# OK: 依存ファイルだけ先にコピー
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./   # 変更頻度が低い
RUN npm ci --only=production             # キャッシュが効く
COPY . .                                 # ソース変更はここだけ再ビルド
CMD ["node", "server.js"]
```

### RUN 命令の分割に注意

```dockerfile
# NG: コマンドごとに RUN を分ける
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN rm -rf /var/lib/apt/lists/*

# OK: 1 つの RUN にまとめる
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        git && \
    rm -rf /var/lib/apt/lists/*
```

分割すると不要なレイヤーが増えるだけでなく、`apt-get update` のキャッシュが古くなり、後続の `install` が失敗する原因にもなります。

---

## 実践的な Dockerfile の例

ここまでの知識を使って、Node.js アプリケーション用の本番向け Dockerfile を書いてみましょう。

```dockerfile
FROM node:20-alpine

# セキュリティ: non-root ユーザーを作成
RUN addgroup -S app && adduser -S app -G app

WORKDIR /app

# 依存関係を先にインストール（キャッシュ効率化）
COPY package.json package-lock.json ./
RUN npm ci --only=production && npm cache clean --force

# アプリケーションコードをコピー
COPY --chown=app:app . .

# non-root ユーザーに切り替え
USER app

EXPOSE 3000

# exec 形式でコマンドを指定
CMD ["node", "server.js"]
```

このように、セキュリティ（non-root ユーザー）とビルド効率（レイヤー順序）の両方を意識した書き方が重要です。

---

## まとめ

| 命令 | 役割 | 覚えておくポイント |
|---|---|---|
| `FROM` | ベースイメージの指定 | Alpine や slim で軽量化する |
| `RUN` | ビルド時のコマンド実行 | 複数コマンドは `&&` で 1 つにまとめる |
| `COPY` | ファイルのコピー | `ADD` より `COPY` を優先する |
| `WORKDIR` | 作業ディレクトリの設定 | `RUN cd` ではなく必ずこちらを使う |
| `CMD` | デフォルトの起動コマンド | exec 形式 `["cmd","arg"]` で書く |
| `ENTRYPOINT` | 固定のメインコマンド | CMD と組み合わせて柔軟に使う |
| `EXPOSE` | ポートの宣言 | ドキュメント用。実際の公開は `-p` で行う |
| `ENV` | 環境変数（ビルド+実行時） | 機密情報は絶対に書かない |
| `ARG` | ビルド引数（ビルド時のみ） | `FROM` の前後でスコープが変わる |
| `.dockerignore` | ビルドコンテキストの除外 | `node_modules` や `.git` を必ず除外する |
| レイヤーキャッシュ | ビルドの高速化 | 変更頻度の低いものを上に配置する |

---

## やってみよう！

以下の課題に取り組んで、Dockerfile の書き方を実践してみましょう。

1. **基本の Dockerfile を書く** -- 好きな言語のアプリケーション（Hello World レベルで OK）の Dockerfile を作成し、`docker build` と `docker run` でコンテナを起動してみましょう。

2. **キャッシュの効果を体感する** -- 依存ファイル（`package.json` や `requirements.txt`）を先にコピーする方法と、全ファイルを一括コピーする方法の 2 パターンで Dockerfile を書き、ソースコードだけ変更して再ビルドしたときの速度差を比較してみましょう。

3. **`.dockerignore` を作成する** -- 自分のプロジェクトで `.dockerignore` がない状態と、適切に設定した状態で `docker build` を実行し、ビルドコンテキストのサイズがどれだけ変わるか確認してみましょう。

4. **non-root ユーザーで実行する** -- 作成した Dockerfile に `USER` 命令を追加し、コンテナ内で `whoami` を実行して root ではないことを確認してみましょう。
