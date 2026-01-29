---
title: "メンテナンスと運用スクリプト"
---

# メンテナンスと運用スクリプト

システムの安定稼働には、定期的なメンテナンスと監視が欠かせません。本章では、バックアップ、ログ管理、モニタリングなどの運用スクリプトを学びます。

## バックアップスクリプト

### 包括的なバックアップ

```bash
#!/usr/bin/env bash
# comprehensive-backup.sh

set -euo pipefail

readonly BACKUP_ROOT="${BACKUP_ROOT:-/var/backups}"
readonly TIMESTAMP="$(date +%Y%m%d_%H%M%S)"
readonly BACKUP_DIR="${BACKUP_ROOT}/${TIMESTAMP}"
readonly RETENTION_DAYS="${RETENTION_DAYS:-30}"

backup_databases() {
    echo "Backing up databases"

    mkdir -p "${BACKUP_DIR}/database"

    # PostgreSQL
    if command -v pg_dumpall &>/dev/null; then
        pg_dumpall -U postgres | gzip > "${BACKUP_DIR}/database/postgres.sql.gz"
    fi

    # MySQL
    if command -v mysqldump &>/dev/null; then
        mysqldump --all-databases | gzip > "${BACKUP_DIR}/database/mysql.sql.gz"
    fi
}

backup_files() {
    echo "Backing up files"

    mkdir -p "${BACKUP_DIR}/files"

    local directories=("/var/www" "/home" "/etc")

    for dir in "${directories[@]}"; do
        if [[ -d "${dir}" ]]; then
            local backup_name
            backup_name="$(echo "${dir}" | tr '/' '_' | sed 's/^_//')"

            tar -czf "${BACKUP_DIR}/files/${backup_name}.tar.gz" \
                --exclude='node_modules' \
                --exclude='.git' \
                --exclude='*.log' \
                "${dir}"
        fi
    done
}

encrypt_backup() {
    local backup_archive="${BACKUP_ROOT}/backup_${TIMESTAMP}.tar.gz"

    echo "Creating encrypted backup archive"

    tar -czf "${backup_archive}" -C "${BACKUP_ROOT}" "${TIMESTAMP}"

    if [[ -n "${GPG_RECIPIENT:-}" ]]; then
        gpg --encrypt --recipient "${GPG_RECIPIENT}" \
            --output "${backup_archive}.gpg" \
            "${backup_archive}"

        rm "${backup_archive}"
    fi
}

upload_to_s3() {
    local backup_file="${1}"

    if command -v aws &>/dev/null && [[ -n "${S3_BUCKET:-}" ]]; then
        echo "Uploading to S3"
        aws s3 cp "${backup_file}" \
            "s3://${S3_BUCKET}/backups/$(basename "${backup_file}")"
    fi
}

cleanup_old_backups() {
    echo "Cleaning up backups older than ${RETENTION_DAYS} days"

    find "${BACKUP_ROOT}" \
        -maxdepth 1 \
        -type d \
        -name "[0-9]*" \
        -mtime "+${RETENTION_DAYS}" \
        -exec rm -rf {} \;
}

main() {
    echo "=== Backup Started ==="

    backup_databases
    backup_files
    encrypt_backup
    cleanup_old_backups

    echo "=== Backup Complete ==="
}

main
```

## ログ管理

### ログローテーション

```bash
#!/usr/bin/env bash
# log-rotate.sh

set -euo pipefail

readonly LOG_DIR="${LOG_DIR:-/var/log/app}"
readonly MAX_SIZE="${MAX_SIZE:-100M}"
readonly MAX_AGE="${MAX_AGE:-30}"
readonly MAX_BACKUPS="${MAX_BACKUPS:-10}"

rotate_log() {
    local log_file="${1}"
    local max_size_bytes=$((${MAX_SIZE%M} * 1024 * 1024))

    if [[ ! -f "${log_file}" ]]; then
        return 0
    fi

    local file_size
    file_size="$(stat -f%z "${log_file}" 2>/dev/null || stat -c%s "${log_file}" 2>/dev/null)"

    if ((file_size < max_size_bytes)); then
        return 0
    fi

    echo "Rotating ${log_file}"

    # 既存のバックアップをシフト
    for ((i = MAX_BACKUPS; i >= 1; i--)); do
        if [[ -f "${log_file}.${i}" ]]; then
            if ((i == MAX_BACKUPS)); then
                rm "${log_file}.${i}"
            else
                mv "${log_file}.${i}" "${log_file}.$((i + 1))"
            fi
        fi
    done

    # ローテーション
    cp "${log_file}" "${log_file}.1"
    : > "${log_file}"
    gzip "${log_file}.1"
}

cleanup_old_logs() {
    echo "Cleaning up logs older than ${MAX_AGE} days"

    find "${LOG_DIR}" \
        -name "*.log.*" \
        -type f \
        -mtime "+${MAX_AGE}" \
        -delete \
        -print
}

main() {
    echo "=== Log Rotation Started ==="

    find "${LOG_DIR}" -name "*.log" -type f | while read -r log_file; do
        rotate_log "${log_file}"
    done

    cleanup_old_logs

    echo "=== Log Rotation Complete ==="
}

main
```

### ログ分析

```bash
#!/usr/bin/env bash
# analyze-logs.sh

analyze_error_logs() {
    local log_file="${1}"

    echo "=== Error Log Analysis ==="

    echo "Error Summary:"
    grep -i "error\|exception" "${log_file}" | \
        awk '{print $NF}' | \
        sort | uniq -c | sort -rn | head -20

    echo ""
    echo "Error Timeline:"
    grep -i "error" "${log_file}" | \
        awk '{print $1, $2}' | \
        cut -d':' -f1 | uniq -c
}

analyze_access_logs() {
    local log_file="${1}"

    echo "=== Access Log Analysis ==="

    echo "Top 10 IPs:"
    awk '{print $1}' "${log_file}" | sort | uniq -c | sort -rn | head -10

    echo ""
    echo "Top 10 URLs:"
    awk '{print $7}' "${log_file}" | sort | uniq -c | sort -rn | head -10

    echo ""
    echo "Status Code Distribution:"
    awk '{print $9}' "${log_file}" | sort | uniq -c | sort
}
```

## クリーンアップスクリプト

### ディスククリーンアップ

```bash
#!/usr/bin/env bash
# disk-cleanup.sh

set -euo pipefail

readonly DRY_RUN="${DRY_RUN:-false}"

cleanup_old_logs() {
    local log_dir="${1:-/var/log}"
    local days_old="${2:-30}"

    echo "Cleaning up logs older than ${days_old} days"

    if [[ "${DRY_RUN}" == "true" ]]; then
        find "${log_dir}" -type f -name "*.log*" -mtime "+${days_old}" -print
    else
        find "${log_dir}" -type f -name "*.log*" -mtime "+${days_old}" -delete -print
    fi
}

cleanup_docker() {
    if ! command -v docker &>/dev/null; then
        return 0
    fi

    echo "Cleaning up Docker resources"

    if [[ "${DRY_RUN}" == "true" ]]; then
        docker system df
        return 0
    fi

    docker image prune -a --filter "until=168h" -f
    docker container prune -f
    docker volume prune -f
    docker network prune -f
}

cleanup_package_managers() {
    echo "Cleaning up package manager caches"

    if command -v npm &>/dev/null; then
        npm cache clean --force
    fi

    if command -v yarn &>/dev/null; then
        yarn cache clean
    fi
}

main() {
    echo "=== Disk Cleanup Started ==="
    echo "Dry run: ${DRY_RUN}"

    df -h /

    cleanup_old_logs "/var/log" 30
    cleanup_docker
    cleanup_package_managers

    echo ""
    df -h /

    echo "=== Disk Cleanup Complete ==="
}

main
```

## モニタリング

### システムモニタリング

```bash
#!/usr/bin/env bash
# system-monitor.sh

set -euo pipefail

readonly CHECK_INTERVAL="${CHECK_INTERVAL:-60}"
readonly CPU_THRESHOLD="${CPU_THRESHOLD:-80}"
readonly MEMORY_THRESHOLD="${MEMORY_THRESHOLD:-85}"
readonly DISK_THRESHOLD="${DISK_THRESHOLD:-90}"

get_cpu_usage() {
    top -bn1 | grep "Cpu(s)" | awk '{print int($2)}'
}

get_memory_usage() {
    free | grep Mem | awk '{print int($3/$2 * 100)}'
}

get_disk_usage() {
    df -h / | tail -1 | awk '{print int($5)}'
}

send_alert() {
    local alert_type="${1}"
    local message="${2}"

    echo "[ALERT] ${alert_type}: ${message}"

    if [[ -n "${SLACK_WEBHOOK:-}" ]]; then
        curl -X POST "${SLACK_WEBHOOK}" \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"${alert_type}: ${message}\"}"
    fi
}

check_resources() {
    local cpu memory disk

    cpu="$(get_cpu_usage)"
    memory="$(get_memory_usage)"
    disk="$(get_disk_usage)"

    echo "$(date +'%Y-%m-%d %H:%M:%S') - CPU: ${cpu}%, Memory: ${memory}%, Disk: ${disk}%"

    if ((cpu >= CPU_THRESHOLD)); then
        send_alert "HIGH_CPU" "CPU usage is ${cpu}%"
    fi

    if ((memory >= MEMORY_THRESHOLD)); then
        send_alert "HIGH_MEMORY" "Memory usage is ${memory}%"
    fi

    if ((disk >= DISK_THRESHOLD)); then
        send_alert "HIGH_DISK" "Disk usage is ${disk}%"
    fi
}

monitor_loop() {
    while true; do
        check_resources
        sleep "${CHECK_INTERVAL}"
    done
}

monitor_loop
```

### ヘルスチェック

```bash
#!/usr/bin/env bash
# health-check.sh

set -euo pipefail

check_service() {
    local service_name="${1}"

    if systemctl is-active --quiet "${service_name}"; then
        echo "✓ ${service_name} is running"
        return 0
    else
        echo "✗ ${service_name} is not running"
        return 1
    fi
}

check_port() {
    local host="${1}"
    local port="${2}"

    if timeout 5 bash -c "cat < /dev/null > /dev/tcp/${host}/${port}" 2>/dev/null; then
        echo "✓ Port ${port} is reachable"
        return 0
    else
        echo "✗ Port ${port} is not reachable"
        return 1
    fi
}

check_http_endpoint() {
    local url="${1}"
    local expected_status="${2:-200}"

    local status_code
    status_code="$(curl -s -o /dev/null -w '%{http_code}' "${url}")"

    if [[ "${status_code}" == "${expected_status}" ]]; then
        echo "✓ ${url} returned ${status_code}"
        return 0
    else
        echo "✗ ${url} returned ${status_code}"
        return 1
    fi
}

check_disk_space() {
    local threshold="${1:-90}"

    local usage
    usage="$(df -h / | tail -1 | awk '{print int($5)}')"

    if ((usage < threshold)); then
        echo "✓ Disk usage is ${usage}%"
        return 0
    else
        echo "✗ Disk usage is ${usage}%"
        return 1
    fi
}

main() {
    local failed_checks=0

    echo "=== System Health Check ==="

    check_service "nginx" || ((failed_checks++))
    check_service "postgresql" || ((failed_checks++))
    check_port "localhost" "80" || ((failed_checks++))
    check_http_endpoint "http://localhost/health" || ((failed_checks++))
    check_disk_space 90 || ((failed_checks++))

    echo "==========================="

    if ((failed_checks > 0)); then
        echo "Health check failed: ${failed_checks} issue(s)"
        return 1
    else
        echo "All health checks passed"
        return 0
    fi
}

main
```

## データベースメンテナンス

### PostgreSQLメンテナンス

```bash
#!/usr/bin/env bash
# postgres-maintenance.sh

set -euo pipefail

readonly DB_NAME="${1:?Database name required}"

perform_vacuum() {
    echo "Performing VACUUM ANALYZE"

    psql "${DB_NAME}" <<EOF
VACUUM (VERBOSE, ANALYZE);

SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;
EOF
}

rebuild_indexes() {
    echo "Rebuilding indexes"

    psql "${DB_NAME}" <<EOF
REINDEX DATABASE ${DB_NAME};
EOF
}

check_database_size() {
    echo "Database size information:"

    psql "${DB_NAME}" <<EOF
SELECT
    pg_database.datname,
    pg_size_pretty(pg_database_size(pg_database.datname)) AS size
FROM pg_database
ORDER BY pg_database_size(pg_database.datname) DESC;
EOF
}

main() {
    echo "=== PostgreSQL Maintenance Started ==="

    perform_vacuum
    rebuild_indexes
    check_database_size

    echo "=== PostgreSQL Maintenance Complete ==="
}

main
```

## まとめ

本章では、運用に必要なメンテナンススクリプトを学びました：

- **バックアップ**: データベース、ファイル、暗号化
- **ログ管理**: ローテーション、分析、クリーンアップ
- **モニタリング**: リソース監視、ヘルスチェック
- **データベースメンテナンス**: VACUUM、インデックス再構築

次章では、CI/CD統合について学びます。
