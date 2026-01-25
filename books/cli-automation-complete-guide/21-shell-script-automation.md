# Shellスクリプト自動化

本章では、Bashスクリプトを使った自動化の基礎と実践的なパターンを学びます。Shellスクリプトは、Linuxサーバー管理、デプロイメント、バッチ処理など、幅広い自動化タスクに活用されています。

## Bash基礎

### 基本構文と変数

```bash
#!/bin/bash

# 変数定義（スペース厳守）
NAME="myapp"
VERSION="1.0.0"
DEPLOY_DIR="/var/www/${NAME}"

# 環境変数のデフォルト値
ENV=${ENV:-development}
DEBUG=${DEBUG:-false}

echo "Application: $NAME v$VERSION"
echo "Environment: $ENV"
echo "Deploy directory: $DEPLOY_DIR"

# コマンド実行結果を変数に格納
CURRENT_DATE=$(date +%Y-%m-%d)
FILE_COUNT=$(ls -1 | wc -l)

echo "Current date: $CURRENT_DATE"
echo "File count: $FILE_COUNT"
```

### 条件分岐

```bash
#!/bin/bash

set -euo pipefail

ENV=${1:-development}

# if-else文
if [ "$ENV" = "production" ]; then
    echo "Production deployment"
    BRANCH="main"
    REPLICAS=3
elif [ "$ENV" = "staging" ]; then
    echo "Staging deployment"
    BRANCH="develop"
    REPLICAS=2
else
    echo "Development deployment"
    BRANCH="develop"
    REPLICAS=1
fi

# ファイル存在チェック
CONFIG_FILE="config/${ENV}.conf"
if [ ! -f "$CONFIG_FILE" ]; then
    echo "Error: Config file not found: $CONFIG_FILE" >&2
    exit 1
fi

# 数値比較
if [ $REPLICAS -gt 1 ]; then
    echo "Multi-replica deployment: $REPLICAS replicas"
fi

# 文字列の空チェック
if [ -z "$BRANCH" ]; then
    echo "Error: Branch is not specified" >&2
    exit 1
fi
```

### ループ処理

```bash
#!/bin/bash

set -euo pipefail

# for文（配列）
ENVIRONMENTS=("development" "staging" "production")

for env in "${ENVIRONMENTS[@]}"; do
    echo "Processing environment: $env"
    cp "config/template.conf" "config/${env}.conf"
done

# for文（範囲）
for i in {1..5}; do
    echo "Iteration: $i"
    sleep 1
done

# while文
COUNTER=0
while [ $COUNTER -lt 3 ]; do
    echo "Counter: $COUNTER"
    COUNTER=$((COUNTER + 1))
done

# ファイル処理
while IFS= read -r line; do
    echo "Processing: $line"
done < input.txt
```

## エラーハンドリング

### set オプション

```bash
#!/bin/bash

# エラー時に即座に終了
set -e

# 未定義変数の使用時にエラー
set -u

# パイプライン内のエラーを検知
set -o pipefail

# 上記をまとめて設定
set -euo pipefail

# デバッグモード（全コマンドを表示）
# set -x
```

### trap によるクリーンアップ

```bash
#!/bin/bash

set -euo pipefail

# 一時ファイル
TEMP_DIR=$(mktemp -d)
LOCK_FILE="/var/lock/deploy.lock"

# クリーンアップ関数
cleanup() {
    local exit_code=$?
    echo "Cleaning up..."
    rm -rf "$TEMP_DIR"
    rm -f "$LOCK_FILE"
    exit $exit_code
}

# EXIT、INT、TERMシグナルでクリーンアップ実行
trap cleanup EXIT INT TERM

# エラー時の処理
error_handler() {
    local line_num=$1
    echo "Error occurred at line $line_num" >&2
    # Slack通知など
}

trap 'error_handler $LINENO' ERR

# メイン処理
echo "Starting deployment..."
touch "$LOCK_FILE"

# 処理実行
cp config.yaml "$TEMP_DIR/"

echo "Deployment completed"
```

## 引数解析

### 位置引数

```bash
#!/bin/bash

set -euo pipefail

# 引数チェック
if [ $# -lt 2 ]; then
    echo "Usage: $0 <environment> <version>" >&2
    exit 1
fi

ENV=$1
VERSION=$2
DRY_RUN=${3:-false}

echo "Environment: $ENV"
echo "Version: $VERSION"
echo "Dry run: $DRY_RUN"
```

### getoptsによるオプション解析

```bash
#!/bin/bash

set -euo pipefail

# デフォルト値
ENV="development"
VERSION=""
DRY_RUN=false
VERBOSE=false

# ヘルプ表示
show_help() {
    cat << EOF
Usage: $0 [OPTIONS]

Options:
    -e, --env ENV        Environment (default: development)
    -v, --version VER    Version to deploy
    -d, --dry-run        Dry run mode
    -V, --verbose        Verbose output
    -h, --help           Show this help
EOF
}

# オプション解析
while [[ $# -gt 0 ]]; do
    case $1 in
        -e|--env)
            ENV="$2"
            shift 2
            ;;
        -v|--version)
            VERSION="$2"
            shift 2
            ;;
        -d|--dry-run)
            DRY_RUN=true
            shift
            ;;
        -V|--verbose)
            VERBOSE=true
            shift
            ;;
        -h|--help)
            show_help
            exit 0
            ;;
        *)
            echo "Unknown option: $1" >&2
            show_help
            exit 1
            ;;
    esac
done

# バリデーション
if [ -z "$VERSION" ]; then
    echo "Error: Version is required" >&2
    show_help
    exit 1
fi

echo "Deploying version $VERSION to $ENV"
```

## ログ出力とデバッグ

### ログ関数

```bash
#!/bin/bash

set -euo pipefail

# ログディレクトリ
LOG_DIR="/var/log/myapp"
mkdir -p "$LOG_DIR"

LOG_FILE="$LOG_DIR/deploy_$(date +%Y%m%d).log"

# ログレベル
LOG_LEVEL=${LOG_LEVEL:-INFO}

# ログ関数
log() {
    local level=$1
    shift
    local message="$*"
    local timestamp=$(date +'%Y-%m-%d %H:%M:%S')

    echo "[$timestamp] [$level] $message" | tee -a "$LOG_FILE"
}

log_info() {
    log "INFO" "$@"
}

log_warn() {
    log "WARN" "$@"
}

log_error() {
    log "ERROR" "$@" >&2
}

log_debug() {
    if [ "$LOG_LEVEL" = "DEBUG" ]; then
        log "DEBUG" "$@"
    fi
}

# 使用例
log_info "Starting deployment"
log_debug "Current directory: $(pwd)"
log_warn "Disk usage is high: $(df -h / | tail -1 | awk '{print $5}')"

if ! command -v docker &> /dev/null; then
    log_error "Docker is not installed"
    exit 1
fi
```

## 実用的な自動化スクリプト例

### バックアップスクリプト

```bash
#!/bin/bash

set -euo pipefail

# 設定
BACKUP_SOURCE="/var/www/myapp"
BACKUP_DEST="/backup/myapp"
RETENTION_DAYS=7

# ログ設定
LOG_FILE="/var/log/backup_$(date +%Y%m%d).log"

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

# バックアップディレクトリ作成
mkdir -p "$BACKUP_DEST"

# バックアップファイル名
BACKUP_FILE="$BACKUP_DEST/backup_$(date +%Y%m%d_%H%M%S).tar.gz"

log "Starting backup: $BACKUP_SOURCE"

# バックアップ実行
if tar -czf "$BACKUP_FILE" -C "$(dirname "$BACKUP_SOURCE")" "$(basename "$BACKUP_SOURCE")"; then
    BACKUP_SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
    log "Backup completed: $BACKUP_FILE ($BACKUP_SIZE)"
else
    log "ERROR: Backup failed"
    exit 1
fi

# 古いバックアップの削除
log "Removing backups older than $RETENTION_DAYS days"
find "$BACKUP_DEST" -name "backup_*.tar.gz" -mtime +$RETENTION_DAYS -delete

log "Backup process completed"
```

### デプロイメントスクリプト

```bash
#!/bin/bash

set -euo pipefail

# 設定
APP_NAME="myapp"
DEPLOY_DIR="/var/www/${APP_NAME}"
GIT_REPO="https://github.com/user/${APP_NAME}.git"
BRANCH="main"

# ログ設定
LOG_FILE="/var/log/deploy_$(date +%Y%m%d).log"

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

# エラーハンドリング
error_handler() {
    log "ERROR: Deployment failed at line $1"
    # ロールバック処理
    if [ -d "${DEPLOY_DIR}.backup" ]; then
        log "Rolling back to previous version..."
        rm -rf "$DEPLOY_DIR"
        mv "${DEPLOY_DIR}.backup" "$DEPLOY_DIR"
        systemctl restart "$APP_NAME"
    fi
    exit 1
}

trap 'error_handler $LINENO' ERR

log "Starting deployment: $APP_NAME ($BRANCH)"

# 現在のバージョンをバックアップ
if [ -d "$DEPLOY_DIR" ]; then
    log "Backing up current version"
    rm -rf "${DEPLOY_DIR}.backup"
    cp -r "$DEPLOY_DIR" "${DEPLOY_DIR}.backup"
fi

# デプロイディレクトリ作成
mkdir -p "$DEPLOY_DIR"
cd "$DEPLOY_DIR"

# コード取得
log "Pulling latest code from $BRANCH"
if [ -d ".git" ]; then
    git fetch origin
    git reset --hard "origin/$BRANCH"
else
    git clone -b "$BRANCH" "$GIT_REPO" .
fi

# 依存関係インストール
log "Installing dependencies"
npm ci --production

# ビルド
log "Building application"
npm run build

# サービス再起動
log "Restarting service"
systemctl restart "$APP_NAME"

# ヘルスチェック
log "Performing health check"
sleep 5

if curl -f http://localhost:3000/health &> /dev/null; then
    log "Health check passed"
    # バックアップ削除
    rm -rf "${DEPLOY_DIR}.backup"
else
    log "ERROR: Health check failed"
    exit 1
fi

log "Deployment completed successfully"
```

### データベースバックアップスクリプト

```bash
#!/bin/bash

set -euo pipefail

# 設定
DB_NAME="myapp"
DB_USER="postgres"
BACKUP_DIR="/backup/database"
RETENTION_DAYS=30

# S3設定（オプション）
S3_BUCKET="s3://my-backups/database"
UPLOAD_TO_S3=${UPLOAD_TO_S3:-false}

# ログ設定
LOG_FILE="/var/log/db_backup_$(date +%Y%m%d).log"

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

# バックアップディレクトリ作成
mkdir -p "$BACKUP_DIR"

# バックアップファイル名
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_$(date +%Y%m%d_%H%M%S).sql.gz"

log "Starting database backup: $DB_NAME"

# バックアップ実行
if pg_dump -U "$DB_USER" "$DB_NAME" | gzip > "$BACKUP_FILE"; then
    BACKUP_SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
    log "Database backup completed: $BACKUP_FILE ($BACKUP_SIZE)"
else
    log "ERROR: Database backup failed"
    exit 1
fi

# S3へアップロード
if [ "$UPLOAD_TO_S3" = "true" ]; then
    log "Uploading to S3: $S3_BUCKET"
    if aws s3 cp "$BACKUP_FILE" "$S3_BUCKET/"; then
        log "Upload to S3 completed"
    else
        log "ERROR: S3 upload failed"
    fi
fi

# 古いバックアップの削除
log "Removing backups older than $RETENTION_DAYS days"
find "$BACKUP_DIR" -name "${DB_NAME}_*.sql.gz" -mtime +$RETENTION_DAYS -delete

log "Database backup process completed"
```

## Shellスクリプトチェックリスト

### 基本
- [ ] シバン（#!/bin/bash）を記述している
- [ ] set -euo pipefailを設定している
- [ ] 変数に適切な命名を使用している
- [ ] コメントで処理内容を説明している

### エラーハンドリング
- [ ] trapでクリーンアップ処理を実装している
- [ ] エラー時のログ出力を実装している
- [ ] 適切な終了コードを返している

### ログ
- [ ] ログファイルに出力している
- [ ] タイムスタンプを含めている
- [ ] ログレベルを使い分けている

### セキュリティ
- [ ] 機密情報をハードコードしていない
- [ ] 引数のサニタイズを行っている
- [ ] ファイルパスのバリデーションを実施している

### 保守性
- [ ] 設定を変数で定義している
- [ ] 関数で処理を分割している
- [ ] ヘルプメッセージを提供している

## まとめ

本章では、Bashスクリプトによる自動化の基礎と実践的なパターンを学びました。

**重要ポイント**:
- set -euo pipefailで堅牢なスクリプトを作成
- trapによる適切なクリーンアップ処理
- ログ出力で運用時のトラブルシューティングを容易に
- 実用的なバックアップ・デプロイスクリプトの実装

Shellスクリプトは、Linuxサーバー管理において欠かせないツールです。これらのベストプラクティスを活用し、信頼性の高い自動化を実現してください。

## 参考文献

- [Bash Reference Manual - GNU](https://www.gnu.org/software/bash/manual/)
- [Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html)
- [ShellCheck - Shell script analysis tool](https://www.shellcheck.net/)
- [Advanced Bash-Scripting Guide](https://tldp.org/LDP/abs/html/)
