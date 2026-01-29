---
title: "Shell スクリプトの基礎と実践"
---

# Shell スクリプトの基礎と実践

Shell スクリプトは、システム管理、自動化、デプロイメントなど、あらゆる場面で活用される基本的かつ強力なツールです。本章では、Bashを中心に、堅牢で保守性の高いShellスクリプトの書き方を学びます。

## 基本構造とベストプラクティス

### Shebangと基本設定

すべてのスクリプトは、適切なShebangとエラーハンドリング設定から始めましょう。

```bash
#!/usr/bin/env bash
# スクリプトの説明: データベースバックアップスクリプト
# 作成者: DevOps Team
# 最終更新: 2026-01-29
# 使用方法: ./backup.sh [database_name]

set -euo pipefail  # 厳格なエラーハンドリング

# グローバル変数
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "${BASH_SOURCE[0]}")"
readonly TIMESTAMP="$(date +%Y%m%d_%H%M%S)"

# メイン処理
main() {
    echo "Starting backup at ${TIMESTAMP}"
    # 処理内容
}

# スクリプトが直接実行された場合のみmainを呼び出す
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi
```

**重要なポイント**:
- `#!/usr/bin/env bash`: 環境に応じたBashを使用
- `set -euo pipefail`: エラー時の即座終了、未定義変数の検出、パイプラインエラーの検出
- `readonly`: 変更不可の定数として定義
- コメントによるドキュメンテーション

### 変数とデータ型

```bash
# 定数の定義（変更不可）
readonly DATABASE_NAME="mydb"
readonly BACKUP_DIR="/var/backups"
readonly MAX_RETRIES=3

# 環境変数のデフォルト値
LOG_LEVEL="${LOG_LEVEL:-INFO}"
TIMEOUT="${TIMEOUT:-30}"

# 配列の使用
declare -a backup_targets=("db1" "db2" "db3")

# 連想配列の使用
declare -A config=(
    ["host"]="localhost"
    ["port"]="5432"
    ["user"]="admin"
)

# 変数の使用
echo "Database: ${DATABASE_NAME}"
echo "Config: ${config[host]}:${config[port]}"

# 配列のループ
for target in "${backup_targets[@]}"; do
    echo "Processing: ${target}"
done
```

### 文字列操作

```bash
text="Hello, World!"

# 長さの取得
echo "${#text}"  # 13

# 部分文字列
echo "${text:0:5}"  # Hello
echo "${text:7}"    # World!

# 置換
echo "${text/World/Bash}"      # Hello, Bash! (最初の一致)
echo "${text//o/0}"            # Hell0, W0rld! (すべて置換)

# 大文字・小文字変換
echo "${text^^}"  # HELLO, WORLD!
echo "${text,,}"  # hello, world!

# パス操作
filepath="/var/log/app.log"
echo "$(dirname "${filepath}")"    # /var/log
echo "$(basename "${filepath}")"   # app.log
echo "${filepath##*.}"             # log (拡張子)
echo "${filepath%.*}"              # /var/log/app (拡張子なし)
```

## エラーハンドリング

### 包括的なエラーハンドラ

```bash
#!/usr/bin/env bash

set -euo pipefail

# エラーハンドラの定義
error_handler() {
    local line_number="${1}"
    local command="${2}"
    local exit_code="${3}"

    echo "Error occurred in ${SCRIPT_NAME}" >&2
    echo "  Line: ${line_number}" >&2
    echo "  Command: ${command}" >&2
    echo "  Exit code: ${exit_code}" >&2
}

# trapの設定
trap 'error_handler ${LINENO} "${BASH_COMMAND}" $?' ERR

# クリーンアップ処理
cleanup() {
    local exit_code=$?
    echo "Cleaning up..."
    rm -f "${TEMP_FILE:-}"
    exit "${exit_code}"
}

trap cleanup EXIT ERR INT TERM
```

### リトライロジック

```bash
retry_command() {
    local -r max_attempts="${1}"
    local -r delay="${2}"
    shift 2
    local -r command=("$@")
    local attempt=1

    until "${command[@]}"; do
        if ((attempt >= max_attempts)); then
            echo "Command failed after ${max_attempts} attempts" >&2
            return 1
        fi

        echo "Attempt ${attempt} failed. Retrying in ${delay}s..." >&2
        sleep "${delay}"
        ((attempt++))
    done

    echo "Command succeeded on attempt ${attempt}"
    return 0
}

# 使用例
retry_command 3 5 curl -f "https://api.example.com/health" || {
    echo "Health check failed"
    exit 1
}
```

### 入力バリデーション

```bash
validate_input() {
    local value="${1:?Value required}"
    local validation_type="${2:?Validation type required}"

    case "${validation_type}" in
        email)
            if [[ ! "${value}" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
                echo "Invalid email: ${value}" >&2
                return 1
            fi
            ;;

        url)
            if [[ ! "${value}" =~ ^https?:// ]]; then
                echo "Invalid URL: ${value}" >&2
                return 1
            fi
            ;;

        ip)
            if [[ ! "${value}" =~ ^([0-9]{1,3}\.){3}[0-9]{1,3}$ ]]; then
                echo "Invalid IP: ${value}" >&2
                return 1
            fi
            ;;

        port)
            if [[ ! "${value}" =~ ^[0-9]+$ ]] || ((value < 1 || value > 65535)); then
                echo "Invalid port: ${value}" >&2
                return 1
            fi
            ;;

        *)
            echo "Unknown validation type: ${validation_type}" >&2
            return 2
            ;;
    esac

    return 0
}

# 使用例
validate_input "user@example.com" "email" || exit 1
validate_input "8080" "port" || exit 1
```

## コマンドライン引数の処理

### getopts による引数解析

```bash
#!/usr/bin/env bash

usage() {
    cat << EOF
Usage: ${SCRIPT_NAME} [OPTIONS]

Options:
    -h, --help              Show this help message
    -v, --verbose           Enable verbose output
    -f, --file FILE         Input file path
    -o, --output DIR        Output directory
    -n, --dry-run           Dry run mode

Examples:
    ${SCRIPT_NAME} -f input.txt -o ./output
    ${SCRIPT_NAME} --file data.csv --verbose

EOF
    exit 0
}

# デフォルト値
VERBOSE=false
DRY_RUN=false
INPUT_FILE=""
OUTPUT_DIR="./output"

# 引数解析
parse_arguments() {
    while [[ $# -gt 0 ]]; do
        case "${1}" in
            -h|--help)
                usage
                ;;
            -v|--verbose)
                VERBOSE=true
                shift
                ;;
            -f|--file)
                INPUT_FILE="${2:?Missing file argument}"
                shift 2
                ;;
            -o|--output)
                OUTPUT_DIR="${2:?Missing output directory}"
                shift 2
                ;;
            -n|--dry-run)
                DRY_RUN=true
                shift
                ;;
            -*)
                echo "Unknown option: ${1}" >&2
                usage
                ;;
            *)
                break
                ;;
        esac
    done

    # 必須パラメータの検証
    if [[ -z "${INPUT_FILE}" ]]; then
        echo "Error: Input file is required" >&2
        usage
    fi
}

parse_arguments "$@"

# デバッグ出力
if [[ "${VERBOSE}" == true ]]; then
    echo "Configuration:"
    echo "  Input file: ${INPUT_FILE}"
    echo "  Output dir: ${OUTPUT_DIR}"
    echo "  Dry run: ${DRY_RUN}"
fi
```

### サブコマンドパターン

```bash
#!/usr/bin/env bash
# deploy-tool.sh - サブコマンド型CLIツール

set -euo pipefail

readonly SCRIPT_NAME="$(basename "${0}")"

# サブコマンド: deploy
cmd_deploy() {
    local environment="${1:?Environment required}"
    local version="${2:-latest}"

    echo "Deploying version ${version} to ${environment}..."

    case "${environment}" in
        production)
            # 本番環境へのデプロイ処理
            ;;
        staging)
            # ステージング環境へのデプロイ処理
            ;;
        *)
            echo "Unknown environment: ${environment}" >&2
            return 1
            ;;
    esac
}

# サブコマンド: rollback
cmd_rollback() {
    local environment="${1:?Environment required}"
    local version="${2:?Version required}"

    echo "Rolling back ${environment} to version ${version}..."
}

# サブコマンド: status
cmd_status() {
    local environment="${1:-all}"
    echo "Status for ${environment}:"
}

# メインディスパッチャー
main() {
    if [[ $# -eq 0 ]]; then
        cat << EOF
Usage: ${SCRIPT_NAME} <command> [arguments]

Commands:
  deploy <env> [version]   Deploy to environment
  rollback <env> <version> Rollback to version
  status [env]             Show deployment status

EOF
        exit 1
    fi

    local subcommand="${1}"
    shift

    case "${subcommand}" in
        deploy)
            cmd_deploy "$@"
            ;;
        rollback)
            cmd_rollback "$@"
            ;;
        status)
            cmd_status "$@"
            ;;
        *)
            echo "Unknown command: ${subcommand}" >&2
            exit 1
            ;;
    esac
}

main "$@"
```

## ファイル操作

### 安全なファイル処理

```bash
# 一時ファイルの安全な作成
create_temp_file() {
    local template="${1:-tmp.XXXXXXXXXX}"
    local temp_file

    temp_file="$(mktemp "/tmp/${template}")" || {
        echo "Failed to create temp file" >&2
        return 1
    }

    # クリーンアップの登録
    trap "rm -f '${temp_file}'" EXIT

    echo "${temp_file}"
}

# アトミックなファイル書き込み
atomic_write() {
    local target_file="${1:?Target file required}"
    local content="${2:?Content required}"
    local temp_file

    temp_file="$(mktemp "${target_file}.XXXXXXXXXX")" || return 1

    # 一時ファイルに書き込み
    printf '%s\n' "${content}" > "${temp_file}" || {
        rm -f "${temp_file}"
        return 1
    }

    # アトミックに置換
    mv "${temp_file}" "${target_file}" || {
        rm -f "${temp_file}"
        return 1
    }

    echo "File written atomically: ${target_file}"
}

# ファイルロックの取得
acquire_lock() {
    local lock_file="${1:?Lock file required}"
    local timeout="${2:-30}"
    local waited=0

    while ! mkdir "${lock_file}" 2>/dev/null; do
        if ((waited >= timeout)); then
            echo "Failed to acquire lock after ${timeout}s" >&2
            return 1
        fi

        sleep 1
        ((waited++))
    done

    trap "rm -rf '${lock_file}'" EXIT
    echo "Lock acquired: ${lock_file}"
}
```

### ディレクトリ操作

```bash
# ディレクトリの安全な作成
ensure_directory() {
    local dir_path="${1:?Directory path required}"
    local mode="${2:-0755}"

    if [[ ! -d "${dir_path}" ]]; then
        mkdir -p "${dir_path}" || {
            echo "Failed to create directory: ${dir_path}" >&2
            return 1
        }
        chmod "${mode}" "${dir_path}"
    fi
}

# 古いファイルの削除
cleanup_old_files() {
    local dir_path="${1:?Directory path required}"
    local days_old="${2:-7}"
    local pattern="${3:-*}"

    echo "Cleaning up files older than ${days_old} days in ${dir_path}"

    find "${dir_path}" \
        -type f \
        -name "${pattern}" \
        -mtime "+${days_old}" \
        -delete \
        -print
}
```

## プロセス管理

### バックグラウンドジョブ管理

```bash
# 並列処理の実行
parallel_execute() {
    local -r max_jobs="${1:?Max jobs required}"
    shift
    local -a tasks=("$@")
    local -a pids=()

    for task in "${tasks[@]}"; do
        # ジョブ数の制限
        while ((${#pids[@]} >= max_jobs)); do
            for i in "${!pids[@]}"; do
                if ! kill -0 "${pids[i]}" 2>/dev/null; then
                    wait "${pids[i]}"
                    unset 'pids[i]'
                fi
            done
            pids=("${pids[@]}")
            sleep 0.1
        done

        # タスクの起動
        eval "${task}" &
        pids+=($!)
    done

    # 全ジョブの完了を待機
    for pid in "${pids[@]}"; do
        wait "${pid}"
    done

    echo "All parallel tasks completed"
}
```

### グレースフルシャットダウン

```bash
graceful_shutdown() {
    local pid="${1:?PID required}"
    local timeout="${2:-30}"

    echo "Sending SIGTERM to process ${pid}"
    kill -TERM "${pid}" 2>/dev/null || return 1

    local waited=0
    while kill -0 "${pid}" 2>/dev/null; do
        if ((waited >= timeout)); then
            echo "Process did not terminate. Sending SIGKILL..."
            kill -KILL "${pid}" 2>/dev/null
            return 2
        fi

        sleep 1
        ((waited++))
    done

    echo "Process ${pid} terminated gracefully"
    return 0
}
```

## デバッグとテスト

### デバッグテクニック

```bash
#!/usr/bin/env bash

# デバッグモード
DEBUG="${DEBUG:-false}"

debug() {
    if [[ "${DEBUG}" == true ]]; then
        echo "[DEBUG] $*" >&2
    fi
}

# 実行時間計測
time_function() {
    local func_name="${1:?Function name required}"
    shift
    local start_time end_time duration

    start_time="$(date +%s%N)"
    debug "Starting ${func_name}"

    "${func_name}" "$@"
    local exit_code=$?

    end_time="$(date +%s%N)"
    duration=$(((end_time - start_time) / 1000000))  # ミリ秒

    debug "${func_name} completed in ${duration}ms (exit code: ${exit_code})"

    return "${exit_code}"
}
```

### シンプルなテストフレームワーク

```bash
#!/usr/bin/env bash
# test_functions.sh

assert_equals() {
    local expected="${1}"
    local actual="${2}"
    local message="${3:-}"

    if [[ "${expected}" == "${actual}" ]]; then
        echo "✓ PASS${message:+: ${message}}"
        return 0
    else
        echo "✗ FAIL${message:+: ${message}}" >&2
        echo "  Expected: ${expected}" >&2
        echo "  Actual:   ${actual}" >&2
        return 1
    fi
}

# テスト実行
run_tests() {
    local tests_passed=0
    local tests_failed=0

    echo "Running tests..."

    assert_equals "5" "$(add 2 3)" "add(2, 3) should return 5" && ((tests_passed++)) || ((tests_failed++))

    echo "Tests passed: ${tests_passed}"
    echo "Tests failed: ${tests_failed}"

    return "${tests_failed}"
}
```

## 実践例: データベースバックアップスクリプト

```bash
#!/usr/bin/env bash
# db-backup.sh - PostgreSQLバックアップスクリプト

set -euo pipefail

readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly TIMESTAMP="$(date +%Y%m%d_%H%M%S)"

# 設定
readonly DB_HOST="${DB_HOST:-localhost}"
readonly DB_PORT="${DB_PORT:-5432}"
readonly DB_USER="${DB_USER:-postgres}"
readonly DB_NAME="${1:?Database name required}"
readonly BACKUP_DIR="${BACKUP_DIR:-/var/backups/postgres}"
readonly RETENTION_DAYS="${RETENTION_DAYS:-7}"

# ログ関数
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*"
}

# バックアップ実行
perform_backup() {
    local backup_file="${BACKUP_DIR}/${DB_NAME}_${TIMESTAMP}.sql.gz"

    log "Starting backup of database: ${DB_NAME}"

    mkdir -p "${BACKUP_DIR}"

    PGPASSWORD="${DB_PASSWORD:-}" pg_dump \
        -h "${DB_HOST}" \
        -p "${DB_PORT}" \
        -U "${DB_USER}" \
        "${DB_NAME}" | gzip > "${backup_file}" || {
        log "ERROR: Backup failed"
        return 1
    }

    log "Backup completed: ${backup_file}"
}

# 古いバックアップの削除
cleanup_old_backups() {
    log "Cleaning up backups older than ${RETENTION_DAYS} days"

    find "${BACKUP_DIR}" \
        -name "${DB_NAME}_*.sql.gz" \
        -type f \
        -mtime "+${RETENTION_DAYS}" \
        -delete \
        -print
}

# メイン処理
main() {
    log "=== Database Backup Started ==="

    perform_backup || exit 1
    cleanup_old_backups

    log "=== Database Backup Completed ==="
}

main "$@"
```

## まとめ

本章では、Shell スクリプトの基礎から実践的なパターンまでを学びました：

- **基本構造**: Shebang、エラーハンドリング、変数管理
- **エラーハンドリング**: trap、リトライロジック、バリデーション
- **引数処理**: getopts、サブコマンドパターン
- **ファイル操作**: 安全な一時ファイル、アトミック書き込み
- **プロセス管理**: 並列処理、グレースフルシャットダウン
- **デバッグとテスト**: デバッグ出力、テストフレームワーク

次章では、Pythonを使ったより高度なスクリプト開発について学びます。
