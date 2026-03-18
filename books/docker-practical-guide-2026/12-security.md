---
title: "コンテナセキュリティ"
---

## コンテナを安全に動かすための考え方

Docker コンテナはプロセスを隔離してくれますが、適切な対策なしでは脅威を防げません。本章では「多層防御」の考え方にもとづき、コンテナ環境を段階的に堅牢にしていきます。

取り上げる領域は次の 5 つです。

1. Rootless 実行（非 root ユーザーでの起動）
2. Read-only ファイルシステム
3. イメージスキャン（Trivy）
4. Seccomp / AppArmor による syscall 制限
5. サプライチェーンセキュリティ

---

## 1. Rootless 実行

### なぜ root で動かしてはいけないのか

Docker コンテナはデフォルトで root として実行されます。コンテナの隔離が突破された場合、ホスト上でも root 権限を奪取される恐れがあります。非 root ユーザーで動かすだけで、このリスクを大幅に軽減できます。

### Dockerfile での設定

```dockerfile
FROM node:20-alpine

RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

WORKDIR /app
COPY --chown=appuser:appgroup . .
RUN npm ci --production

USER appuser
CMD ["node", "index.js"]
```

`USER` 命令以降、`CMD` や `ENTRYPOINT` が指定したユーザーで実行されます。

### docker run 時の指定

```bash
docker run --user 1001:1001 myapp:latest
```

### Capability の制限

Docker はデフォルトで 14 個の Linux Capability を付与しますが、大半のアプリでは不要です。

```bash
docker run \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  myapp:latest
```

| オプション | 効果 |
|-----------|------|
| `--cap-drop ALL` | 全 Capability を除去します |
| `--cap-add NET_BIND_SERVICE` | 1024 番未満のポートバインドのみ許可します |
| `--security-opt no-new-privileges` | 実行中の権限昇格を防ぎます |

---

## 2. Read-only ファイルシステム

### 書き込みを禁止して改ざんを防ぐ

ファイルシステムを読み取り専用にすると、攻撃者がマルウェアを書き込んだり設定を改ざんしたりすることを防げます。

```bash
docker run \
  --read-only \
  --tmpfs /tmp:noexec,nosuid,size=64m \
  myapp:latest
```

`--tmpfs` でアプリが一時ファイルを書き込むディレクトリだけを許可します。

### アプリケーション別の書き込みディレクトリ

| アプリケーション | 書き込みが必要なパス |
|----------------|-------------------|
| Node.js | `/tmp` |
| Python | `/tmp`, `__pycache__` |
| Nginx | `/var/cache/nginx`, `/var/run` |

エラーログに `EROFS (Read-only file system)` が出たら、そのパスだけを `tmpfs` でマウントしてください。

### Docker Compose で全対策を組み合わせる

```yaml
services:
  app:
    image: myapp:latest
    user: "1001:1001"
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=64m
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    deploy:
      resources:
        limits:
          memory: 256M
          pids: 100
```

---

## 3. イメージスキャン（Trivy）

### Trivy とは

Trivy は Aqua Security が開発するオープンソースのセキュリティスキャナです。コンテナイメージの OS パッケージやライブラリに含まれる既知脆弱性 (CVE) を検出します。

### 基本的な使い方

```bash
# イメージをスキャンします
trivy image myapp:latest

# CRITICAL と HIGH のみ表示します
trivy image --severity CRITICAL,HIGH myapp:latest

# 修正版未リリースの脆弱性を除外します
trivy image --ignore-unfixed myapp:latest

# JSON 形式で出力します
trivy image --format json --output results.json myapp:latest

# Dockerfile の設定ミスを検出します
trivy config Dockerfile
```

### CI/CD への統合

GitHub Actions に組み込むことで、脆弱なイメージのデプロイを自動ブロックできます。

```yaml
# .github/workflows/security.yml
name: Security Scan
on:
  push:
    branches: [main]

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .
      - name: Run Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
          ignore-unfixed: true
```

`exit-code: '1'` により、CRITICAL/HIGH が見つかるとジョブが失敗します。

### .trivyignore で特定の CVE を除外する

```text
# .trivyignore
CVE-2024-12345  # libxml2 - XML パースは未使用
CVE-2024-67890  # 修正版未リリース（チケットで追跡中）
```

「無視する」のではなく「追跡する」ことが大切です。

### ベースイメージの選択が結果を左右する

| ベースイメージ | サイズ目安 | CVE 数の傾向 |
|--------------|----------|-------------|
| `node:20` | ~350MB | 多い |
| `node:20-alpine` | ~130MB | 少ない |
| `gcr.io/distroless/nodejs20` | ~50MB | 最小 |

本番では alpine か distroless を選ぶのがおすすめです。

---

## 4. Seccomp / AppArmor

### Seccomp（Secure Computing Mode）

Seccomp はコンテナが呼び出せるシステムコールを制限する Linux カーネルの機能です。Docker はデフォルトで危険な syscall をブロックするプロファイルを適用しています。

```bash
# デフォルトプロファイルを明示的に指定します
docker run --security-opt seccomp=default myapp:latest

# カスタムプロファイルを適用します
docker run --security-opt seccomp=my-profile.json myapp:latest
```

カスタムプロファイルではホワイトリスト方式で許可する syscall だけを列挙します。

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {
      "names": [
        "accept", "bind", "close", "connect",
        "epoll_create", "epoll_ctl", "epoll_wait",
        "exit", "exit_group", "fstat",
        "listen", "mmap", "open", "openat",
        "read", "write", "socket"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

### AppArmor

AppArmor はプロセスがアクセスできるファイルやネットワークをプロファイル単位で制限します。Docker は `docker-default` プロファイルを自動適用します。

```bash
docker run --security-opt apparmor=docker-default myapp:latest
```

### 両者の使い分け

| 項目 | Seccomp | AppArmor |
|------|---------|----------|
| 制限対象 | システムコール | ファイル・ネットワーク |
| 粒度 | syscall 単位 | パス・操作単位 |
| Docker デフォルト | あり | あり (Ubuntu) |

まずは Docker のデフォルトプロファイルを活用し、必要に応じてカスタムに進むのがよいでしょう。

---

## 5. サプライチェーンセキュリティ

### サプライチェーン攻撃とは

ベースイメージやライブラリが改ざんされたり、脆弱性を含むケースが増えています。2024 年の xz-utils 事件のように、信頼されていたパッケージに悪意あるコードが挿入された事例もあります。

### lockfile で依存関係を固定する

```bash
npm ci                           # lockfile に厳密に従います
pnpm install --frozen-lockfile   # 不一致時はエラーにします
```

lockfile は必ず Git にコミットしてください。

### イメージ署名（cosign）

cosign はコンテナイメージにデジタル署名を付与するツールです。署名済みイメージだけをデプロイする仕組みで改ざんを防ぎます。

```bash
# 署名します（Keyless 方式、CI/CD 向け）
cosign sign --yes ghcr.io/myorg/myapp@sha256:abc123...

# 検証します
cosign verify \
  --certificate-identity-regexp="https://github.com/myorg/.*" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  ghcr.io/myorg/myapp:1.0.0
```

### SBOM（ソフトウェア部品表）

SBOM はイメージに含まれる全パッケージとバージョンの一覧です。新しい CVE が公開されたとき、影響範囲を即座に特定できます。

```bash
# SBOM を生成します
trivy image --format spdx-json --output sbom.json myapp:latest

# SBOM から脆弱性を再スキャンします
trivy sbom sbom.json
```

### ダイジェスト参照で改ざんを防ぐ

タグ (`latest`, `v1.0.0`) はレジストリ上で別のイメージに差し替えられます。ダイジェスト参照なら改ざん不可能です。

```yaml
# NG: タグは上書き可能です
image: myapp:latest

# OK: ダイジェストは内容のハッシュです
image: myapp@sha256:a1b2c3d4e5f6...
```

---

## まとめ

| 対策 | 何を防ぐか | 設定方法 |
|------|----------|---------|
| 非 root 実行 | コンテナ脱出時の権限奪取 | `USER appuser` / `--user 1001:1001` |
| Capability 制限 | 不要な特権の悪用 | `--cap-drop ALL --cap-add ...` |
| Read-only FS | マルウェア書き込み・改ざん | `--read-only --tmpfs /tmp` |
| Trivy スキャン | 既知脆弱性のデプロイ | `trivy image --severity CRITICAL,HIGH` |
| Seccomp | 危険な syscall の実行 | `--security-opt seccomp=default` |
| AppArmor | 不正なファイルアクセス | `--security-opt apparmor=docker-default` |
| 依存関係固定 | 改ざんパッケージの混入 | `npm ci` / `--frozen-lockfile` |
| cosign 署名 | 改ざんイメージのデプロイ | `cosign sign` / `cosign verify` |
| SBOM | 脆弱性の影響範囲の把握 | `trivy image --format spdx-json` |
| ダイジェスト参照 | タグ差し替え攻撃 | `image: myapp@sha256:...` |

---

## やってみよう！

以下の 3 つの課題で、コンテナセキュリティの効果を体感してみましょう。

### 課題 1: 非 root + Read-only で Nginx を動かす

```bash
docker run -d \
  --name secure-nginx \
  --read-only \
  --tmpfs /var/cache/nginx \
  --tmpfs /var/run \
  --tmpfs /tmp \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  -p 8080:80 \
  nginx:alpine

# 動作確認
curl http://localhost:8080

# 書き込みが失敗することを確認します
docker exec secure-nginx sh -c "echo test > /etc/nginx/nginx.conf"
```

### 課題 2: Trivy でイメージをスキャンする

```bash
brew install trivy
trivy image --severity CRITICAL,HIGH nginx:alpine
trivy config Dockerfile
```

普段使っているイメージでも試してみてください。CRITICAL が 0 件になるベースイメージを探してみましょう。

### 課題 3: Docker Compose で全対策を適用する

```yaml
services:
  app:
    build: .
    user: "1001:1001"
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=64m
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    ports:
      - "3000:3000"
```

起動後、`docker exec` で `whoami`（root でないこと）や書き込み失敗を確認してみてください。
