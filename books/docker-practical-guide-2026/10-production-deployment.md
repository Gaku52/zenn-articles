---
title: "本番デプロイ"
---

本章では、Dockerコンテナを本番環境へデプロイする際に欠かせない実践テクニックを学びます。開発環境で動くDockerfileと本番用Dockerfileには大きな違いがあります。セキュリティ、信頼性、可観測性の3つの柱を軸に、プロダクションレディなコンテナ運用を身につけましょう。

## 本番用Dockerfileの設計

本番用では「最小構成・非root・マルチステージ」が三原則です。

### マルチステージビルドで最小イメージを作る

```dockerfile
# === ビルドステージ ===
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build && npm prune --production

# === 本番ステージ ===
FROM node:20-alpine AS production

RUN apk update && apk upgrade --no-cache && \
    apk add --no-cache tini

# 非rootユーザーを作成します
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

WORKDIR /app

# ビルド成果物だけをコピーします
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/package.json ./

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

USER appuser
EXPOSE 3000
ENTRYPOINT ["tini", "--"]
CMD ["node", "dist/server.js"]
```

ポイントは次の3つです。

- **ビルドステージと本番ステージの分離**: ビルドツールやソースコードが本番イメージに含まれません
- **`npm prune --production`**: 開発用パッケージを除去してイメージサイズを削減します
- **`tini`の導入**: PID 1問題を解決し、シグナルの伝播を正しく行います

### 非rootユーザーの重要性

コンテナはデフォルトでroot権限で実行されます。コンテナエスケープの脆弱性を突かれた場合、ホストのroot権限を奪取されるリスクがあります。`USER`命令で専用ユーザーに切り替え、被害を最小限に抑えましょう。

```dockerfile
# Python の場合
FROM python:3.12-slim AS production
RUN groupadd -g 1001 appgroup && \
    useradd -u 1001 -g appgroup -s /bin/false -m appuser
WORKDIR /app
COPY --from=builder --chown=appuser:appgroup /opt/venv /opt/venv
COPY --chown=appuser:appgroup . .
USER appuser
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "app:create_app()"]
```

## 環境変数管理

本番環境では設定値を環境変数で管理するのが基本です。ただし「機密性」に応じた使い分けが重要です。

### 非機密情報は直接指定する

```yaml
services:
  api:
    image: my-api:latest
    environment:
      NODE_ENV: production
      LOG_LEVEL: info
      PORT: "3000"
      TZ: Asia/Tokyo
```

### 環境固有の設定は `.env` ファイルで管理する

```yaml
services:
  api:
    env_file:
      - .env.production   # .gitignore に必ず追加してください
```

```bash
# .env.production
DATABASE_HOST=db.internal.example.com
REDIS_HOST=redis.internal.example.com
ALLOWED_ORIGINS=https://example.com
```

## Secrets（機密情報）管理

パスワードやAPIキーは環境変数への直接記述を避け、Docker Secretsを活用します。

```yaml
services:
  api:
    image: my-api:latest
    secrets:
      - db_password
      - api_key
      - jwt_secret

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    external: true
  jwt_secret:
    environment: JWT_SECRET    # Compose v2.17+
```

### アプリケーション側でシークレットを読み取る

Docker Secretsは `/run/secrets/` にファイルとしてマウントされます。

```javascript
const fs = require("fs");
const path = require("path");

function readSecret(secretName) {
  const secretPath = path.join("/run/secrets", secretName);
  try {
    return fs.readFileSync(secretPath, "utf8").trim();
  } catch (err) {
    return process.env[secretName.toUpperCase()];
  }
}

const dbPassword = readSecret("db_password");
const jwtSecret = readSecret("jwt_secret");
```

**絶対にやってはいけないこと**: `docker-compose.yml`にパスワードを直書きすることです。Git履歴に機密情報が残り、完全な削除が困難になります。

## ヘルスチェック

ヘルスチェックは、コンテナが正常に動作しているかをDockerが定期的に確認する仕組みです。

### パラメータの意味

| パラメータ | 説明 | 推奨値 |
|-----------|------|--------|
| `interval` | チェック間隔 | 10〜30秒 |
| `timeout` | タイムアウト | 3〜10秒 |
| `retries` | 失敗許容回数 | 3〜5回 |
| `start_period` | 起動猶予期間 | アプリ起動時間に合わせる |

### Liveness と Readiness の2段階設計

```javascript
const express = require("express");
const app = express();

// Liveness: プロセスの死活だけを確認します
app.get("/health", (req, res) => {
  res.status(200).json({ status: "ok", uptime: process.uptime() });
});

// Readiness: 依存サービスとの接続も確認します
app.get("/ready", async (req, res) => {
  const checks = {};
  let isReady = true;

  try {
    await db.query("SELECT 1");
    checks.database = "ok";
  } catch (err) {
    checks.database = "error";
    isReady = false;
  }

  try {
    await redis.ping();
    checks.redis = "ok";
  } catch (err) {
    checks.redis = "error";
    isReady = false;
  }

  res.status(isReady ? 200 : 503).json({ status: isReady ? "ready" : "not_ready", checks });
});
```

### 依存関係の制御

```yaml
services:
  postgres:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  api:
    depends_on:
      postgres:
        condition: service_healthy   # DBが健全になるまで待機します
```

## グレースフルシャットダウン

`docker stop`を実行するとSIGTERMシグナルが送られます。これを受け取り、処理中のリクエストを完了してから終了する仕組みがグレースフルシャットダウンです。

### シャットダウンの流れ

```
docker stop → SIGTERM送信 → アプリが受信
  1. 新規リクエスト受付停止
  2. 処理中リクエスト完了
  3. DB接続クローズ
  4. exit(0) で正常終了
  → stop_grace_period 経過後 → SIGKILL（強制終了）
```

### Node.js での実装

```javascript
const http = require("http");
const server = http.createServer((req, res) => {
  res.writeHead(200);
  res.end("OK");
});
server.listen(3000);

process.on("SIGTERM", () => {
  console.log("SIGTERM received. Shutting down gracefully...");
  server.close(() => {
    console.log("HTTP server closed");
    process.exit(0);
  });
  // 安全策：一定時間後に強制終了します
  setTimeout(() => {
    console.error("Forced shutdown after timeout");
    process.exit(1);
  }, 10000);
});
```

### exec形式のCMDが必須

```dockerfile
# NG: shell形式（シグナルが /bin/sh に届き、nodeに伝わりません）
CMD node server.js

# OK: exec形式（nodeがPID 1になり、シグナルを直接受信します）
CMD ["node", "server.js"]
```

`stop_grace_period`でSIGTERMからSIGKILLまでの猶予時間を調整できます。

```yaml
services:
  api:
    stop_grace_period: 30s
```

## ログ戦略

ログ設計は障害対応の速度に直結します。

### 3つの原則

**1. stdout/stderr に出力する**

コンテナ内のファイルにログを書いてはいけません。再起動で消失し、ディスクも圧迫します。

**2. JSON構造化ログを採用する**

```javascript
console.log(JSON.stringify({
  timestamp: new Date().toISOString(),
  level: "info",
  message: "Request handled",
  method: "GET",
  path: "/api/users",
  status: 200,
  duration_ms: 45,
  request_id: "abc-123"
}));
```

**3. ログローテーションを設定する**

```yaml
services:
  api:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"
        compress: "true"
```

### ログ設計の比較

| 項目 | 推奨 | 非推奨 | 理由 |
|------|------|--------|------|
| 出力先 | stdout / stderr | コンテナ内ファイル | Dockerログドライバーが処理します |
| 形式 | JSON構造化 | プレーンテキスト | 解析・フィルタリングが容易です |
| レベル | 環境変数で制御 | ハードコード | 環境ごとに切り替えられます |
| 相関ID | request_id を含める | IDなし | 分散トレーシングに不可欠です |
| 機密情報 | マスクまたは除外 | そのまま出力 | 漏洩防止のためです |

## 本番用docker-composeのセキュリティ設定

各セクションの知識を統合し、セキュリティを強化した設定を紹介します。

```yaml
services:
  api:
    image: registry.example.com/api:${VERSION:-latest}
    restart: unless-stopped
    read_only: true                   # ルートFS読み取り専用
    tmpfs:
      - /tmp:size=100m,noexec
    security_opt:
      - no-new-privileges:true        # 権限昇格を禁止
    cap_drop:
      - ALL                           # 全Capabilityを削除
    cap_add:
      - NET_BIND_SERVICE              # 必要なもののみ追加
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
    stop_grace_period: 30s
    networks:
      - frontend
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true                    # 外部インターネットへの通信を遮断
```

`read_only: true`を設定するとコンテナ内への書き込みが禁止されます。一時ファイルが必要な場合は`tmpfs`マウントを追加してください。`cap_drop: ALL`ですべてのLinux Capabilityを削除し、必要最小限のものだけを`cap_add`で戻すのがベストプラクティスです。

## まとめ

| テーマ | ポイント | 具体的な設定 |
|--------|---------|-------------|
| 本番用Dockerfile | マルチステージビルドで最小構成にする | `FROM ... AS builder` + 本番ステージ |
| 非rootユーザー | 専用ユーザーを作成して権限を最小化する | `USER appuser` |
| 環境変数管理 | 機密性に応じて使い分ける | `environment` / `env_file` |
| Secrets管理 | 機密情報はDocker Secretsで管理する | `secrets` + `/run/secrets/` |
| ヘルスチェック | Liveness/Readinessの2段階で監視する | `HEALTHCHECK` + `/health` |
| グレースフルシャットダウン | SIGTERMを受けて安全に終了する | exec形式CMD + `stop_grace_period` |
| ログ戦略 | stdout/stderrにJSON構造化ログを出力する | `logging` + ローテーション |
| セキュリティ強化 | 読取専用FS + Capability制限をかける | `read_only` + `cap_drop: ALL` |

## やってみよう！

以下の演習に取り組んで、本番デプロイの理解を深めましょう。

1. **本番用Dockerfileの作成**: 自分のアプリケーションでマルチステージビルドのDockerfileを書いてみましょう。非rootユーザーの設定とヘルスチェックの定義も含めてください。

2. **グレースフルシャットダウンの確認**: アプリケーションにSIGTERMハンドラーを実装し、`docker stop`で処理中のリクエストが正しく完了するかテストしてみましょう。

3. **ログ設計の実践**: アプリケーションのログをJSON構造化ログに変更し、`request_id`を含めるようにしてみましょう。`docker logs`コマンドで出力を確認してください。

4. **セキュリティチェックリスト**: 本章で紹介した設定（`read_only`、`cap_drop`、`no-new-privileges`、Secrets管理）をすべて適用した`docker-compose.prod.yml`を作成してみましょう。
