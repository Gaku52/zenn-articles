---
title: "自動化とデプロイメント"
---

# 自動化とデプロイメント

スクリプトの最も重要な用途の一つが、デプロイメントの自動化です。本章では、実践的なデプロイメントスクリプトの設計と実装方法を学びます。

## デプロイメントスクリプトの基本設計

### 設計原則

デプロイメントスクリプトを設計する際の重要な原則：

1. **べき等性**: 何度実行しても同じ結果になる
2. **ロールバック可能**: 問題発生時に元に戻せる
3. **検証**: 各ステップの成功を確認
4. **ログ記録**: すべての操作を記録
5. **通知**: 結果を関係者に通知

### デプロイメントフレームワーク

```bash
#!/usr/bin/env bash
# deploy-framework.sh

set -euo pipefail

readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly TIMESTAMP="$(date +%Y%m%d_%H%M%S)"
readonly LOG_FILE="/var/log/deploy/${TIMESTAMP}.log"

# ログ関数
log() {
    local level="${1}"
    shift
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] [${level}] $*" | tee -a "${LOG_FILE}"
}

# デプロイメント状態の管理
STATE_FILE="${HOME}/.deploy/state.json"

save_state() {
    local key="${1}"
    local value="${2}"

    mkdir -p "$(dirname "${STATE_FILE}")"

    if [[ ! -f "${STATE_FILE}" ]]; then
        echo "{}" > "${STATE_FILE}"
    fi

    # jqを使用してJSON更新
    local temp_file
    temp_file="$(mktemp)"

    jq --arg key "${key}" --arg value "${value}" \
       '.[$key] = $value' "${STATE_FILE}" > "${temp_file}"

    mv "${temp_file}" "${STATE_FILE}"
}

load_state() {
    local key="${1}"

    if [[ ! -f "${STATE_FILE}" ]]; then
        return 1
    fi

    jq -r --arg key "${key}" '.[$key] // empty' "${STATE_FILE}"
}

# べき等性の実装
ensure_idempotent() {
    local operation_id="${1}"
    shift
    local operation=("$@")

    local state_key="operation_${operation_id}"
    local last_run
    last_run="$(load_state "${state_key}")"

    if [[ -n "${last_run}" ]]; then
        log "INFO" "Operation ${operation_id} already completed at ${last_run}"
        return 0
    fi

    log "INFO" "Executing operation: ${operation_id}"
    "${operation[@]}" || return 1

    save_state "${state_key}" "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
    log "INFO" "Operation ${operation_id} completed"
}
```

## Zero-Downtime デプロイメント

### 基本実装

```bash
#!/usr/bin/env bash
# zero-downtime-deploy.sh

set -euo pipefail

readonly APP_NAME="${APP_NAME:-myapp}"
readonly DEPLOY_USER="${DEPLOY_USER:-deploy}"
readonly DEPLOY_PATH="/var/www/${APP_NAME}"
readonly RELEASES_PATH="${DEPLOY_PATH}/releases"
readonly SHARED_PATH="${DEPLOY_PATH}/shared"
readonly CURRENT_LINK="${DEPLOY_PATH}/current"

# ディレクトリ構造
# /var/www/myapp/
# ├── current -> releases/20260129120000
# ├── releases/
# │   ├── 20260129120000/
# │   ├── 20260129110000/
# │   └── 20260129100000/
# └── shared/
#     ├── logs/
#     ├── tmp/
#     └── config/

deploy_setup() {
    log "INFO" "Setting up deploy directories"

    ssh "${DEPLOY_USER}@${TARGET_HOST}" "
        mkdir -p ${RELEASES_PATH}
        mkdir -p ${SHARED_PATH}/{logs,tmp,config}
    "
}

prepare_release() {
    local release_name="${1}"
    local release_path="${RELEASES_PATH}/${release_name}"

    log "INFO" "Preparing release: ${release_name}"

    # コードのデプロイ
    rsync -avz --delete \
        --exclude='.git' \
        --exclude='node_modules' \
        --exclude='.env' \
        ./ "${DEPLOY_USER}@${TARGET_HOST}:${release_path}/"

    # 共有ファイルへのシンボリックリンク
    ssh "${DEPLOY_USER}@${TARGET_HOST}" "
        ln -nfs ${SHARED_PATH}/logs ${release_path}/logs
        ln -nfs ${SHARED_PATH}/tmp ${release_path}/tmp
        ln -nfs ${SHARED_PATH}/config/.env ${release_path}/.env
    "
}

build_release() {
    local release_path="${1}"

    log "INFO" "Building release"

    ssh "${DEPLOY_USER}@${TARGET_HOST}" "
        cd ${release_path}
        npm ci --production
        npm run build
    "
}

run_migrations() {
    local release_path="${1}"

    log "INFO" "Running database migrations"

    ssh "${DEPLOY_USER}@${TARGET_HOST}" "
        cd ${release_path}
        npm run migrate
    " || {
        log "ERROR" "Migration failed"
        return 1
    }
}

switch_release() {
    local release_path="${1}"

    log "INFO" "Switching to new release"

    # アトミックなシンボリックリンクの更新
    ssh "${DEPLOY_USER}@${TARGET_HOST}" "
        ln -nfs ${release_path} ${CURRENT_LINK}.tmp
        mv -Tf ${CURRENT_LINK}.tmp ${CURRENT_LINK}
    "
}

reload_application() {
    log "INFO" "Reloading application"

    # Graceful reload（PM2の例）
    ssh "${DEPLOY_USER}@${TARGET_HOST}" "
        cd ${CURRENT_LINK}
        pm2 reload ecosystem.config.js --update-env
    " || {
        log "ERROR" "Application reload failed"
        return 1
    }
}

health_check() {
    local max_retries=30
    local retry_delay=2

    log "INFO" "Running health checks"

    for ((i=1; i<=max_retries; i++)); do
        if curl -sf "https://${TARGET_HOST}/health" > /dev/null; then
            log "INFO" "Health check passed"
            return 0
        fi

        log "WARNING" "Health check failed (attempt ${i}/${max_retries})"
        sleep "${retry_delay}"
    done

    log "ERROR" "Health check failed after ${max_retries} attempts"
    return 1
}

rollback() {
    log "WARNING" "Rolling back deployment"

    local previous_release
    previous_release="$(ssh "${DEPLOY_USER}@${TARGET_HOST}" "
        ls -t ${RELEASES_PATH} | sed -n 2p
    ")"

    if [[ -z "${previous_release}" ]]; then
        log "ERROR" "No previous release found for rollback"
        return 1
    fi

    switch_release "${RELEASES_PATH}/${previous_release}"
    reload_application
}

cleanup_old_releases() {
    local keep_releases="${1:-5}"

    log "INFO" "Cleaning up old releases (keeping ${keep_releases})"

    ssh "${DEPLOY_USER}@${TARGET_HOST}" "
        cd ${RELEASES_PATH}
        ls -t | tail -n +$((keep_releases + 1)) | xargs -r rm -rf
    "
}

# メインデプロイプロセス
deploy() {
    local release_name
    release_name="$(date +%Y%m%d%H%M%S)"
    local release_path="${RELEASES_PATH}/${release_name}"

    log "INFO" "=== Starting Deployment ==="
    log "INFO" "Release: ${release_name}"
    log "INFO" "Target: ${TARGET_HOST}"

    deploy_setup || return 1
    prepare_release "${release_name}" || return 1
    build_release "${release_path}" || return 1
    run_migrations "${release_path}" || {
        log "ERROR" "Deployment failed during migrations"
        return 1
    }

    switch_release "${release_path}" || return 1
    reload_application || {
        log "ERROR" "Application reload failed, rolling back"
        rollback
        return 1
    }

    if health_check; then
        log "INFO" "Deployment successful"
        cleanup_old_releases 5
        log "INFO" "=== Deployment Complete ==="
    else
        log "ERROR" "Health checks failed, rolling back"
        rollback
        return 1
    fi
}

# 使用例
TARGET_HOST="${1:?Target host required}"
deploy
```

## Blue-Green デプロイメント

### 実装例

```bash
#!/usr/bin/env bash
# blue-green-deploy.sh

set -euo pipefail

readonly BLUE_ENV="blue"
readonly GREEN_ENV="green"
readonly LB_CONFIG="/etc/nginx/sites-enabled/app.conf"

get_active_environment() {
    if grep -q "upstream_${BLUE_ENV}" "${LB_CONFIG}"; then
        echo "${BLUE_ENV}"
    else
        echo "${GREEN_ENV}"
    fi
}

get_inactive_environment() {
    local active
    active="$(get_active_environment)"

    if [[ "${active}" == "${BLUE_ENV}" ]]; then
        echo "${GREEN_ENV}"
    else
        echo "${BLUE_ENV}"
    fi
}

deploy_to_environment() {
    local env="${1}"
    local version="${2}"

    log "INFO" "Deploying version ${version} to ${env} environment"

    # Dockerを使用した例
    docker-compose -f "docker-compose.${env}.yml" pull
    docker-compose -f "docker-compose.${env}.yml" up -d --force-recreate

    # ヘルスチェック
    wait_for_healthy "${env}"
}

wait_for_healthy() {
    local env="${1}"
    local port=$( [[ "${env}" == "${BLUE_ENV}" ]] && echo 8001 || echo 8002 )
    local url="http://localhost:${port}/health"
    local max_retries=30

    log "INFO" "Waiting for ${env} environment to be healthy"

    for ((i=1; i<=max_retries; i++)); do
        if curl -sf "${url}" > /dev/null; then
            log "INFO" "${env} environment is healthy"
            return 0
        fi

        sleep 2
    done

    log "ERROR" "${env} environment failed to become healthy"
    return 1
}

switch_traffic() {
    local target_env="${1}"

    log "INFO" "Switching traffic to ${target_env} environment"

    # Nginxの設定を更新
    sed -i.bak "s/upstream_[a-z]*/upstream_${target_env}/" "${LB_CONFIG}"

    # Nginxをリロード
    nginx -t && nginx -s reload || {
        log "ERROR" "Failed to reload Nginx, reverting"
        mv "${LB_CONFIG}.bak" "${LB_CONFIG}"
        nginx -s reload
        return 1
    }

    log "INFO" "Traffic switched to ${target_env}"
}

smoke_test() {
    local env="${1}"

    log "INFO" "Running smoke tests on ${env}"

    local endpoints=("/health" "/api/version" "/api/status")

    for endpoint in "${endpoints[@]}"; do
        if ! curl -sf "https://myapp.com${endpoint}" > /dev/null; then
            log "ERROR" "Smoke test failed: ${endpoint}"
            return 1
        fi
    done

    log "INFO" "Smoke tests passed"
}

blue_green_deploy() {
    local version="${1:?Version required}"

    local active_env inactive_env
    active_env="$(get_active_environment)"
    inactive_env="$(get_inactive_environment)"

    log "INFO" "=== Blue-Green Deployment ==="
    log "INFO" "Active: ${active_env}, Target: ${inactive_env}"
    log "INFO" "Version: ${version}"

    # 非アクティブ環境にデプロイ
    deploy_to_environment "${inactive_env}" "${version}" || return 1

    # スモークテスト
    smoke_test "${inactive_env}" || {
        log "ERROR" "Smoke tests failed, aborting deployment"
        return 1
    }

    # トラフィックを切り替え
    switch_traffic "${inactive_env}" || {
        log "ERROR" "Traffic switch failed"
        return 1
    }

    # 本番トラフィックでの検証
    sleep 10
    if ! smoke_test "${inactive_env}"; then
        log "ERROR" "Production validation failed, rolling back"
        switch_traffic "${active_env}"
        return 1
    fi

    log "INFO" "=== Deployment Complete ==="
    log "INFO" "Old environment (${active_env}) is still running for rollback"
}

blue_green_deploy "$@"
```

## 環境管理とプロビジョニング

### サーバーセットアップスクリプト

```bash
#!/usr/bin/env bash
# provision.sh - サーバープロビジョニング

set -euo pipefail

install_system_packages() {
    log "INFO" "Installing system packages"

    if command -v apt-get &>/dev/null; then
        apt-get update
        apt-get install -y \
            curl \
            git \
            build-essential \
            nginx \
            postgresql \
            redis-server
    fi
}

install_nodejs() {
    local node_version="${1:-18}"

    log "INFO" "Installing Node.js ${node_version}"

    # nvmを使用
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

    export NVM_DIR="${HOME}/.nvm"
    source "${NVM_DIR}/nvm.sh"

    nvm install "${node_version}"
    nvm use "${node_version}"
    nvm alias default "${node_version}"
}

setup_app_user() {
    local username="${1:-deploy}"

    log "INFO" "Setting up application user: ${username}"

    if ! id "${username}" &>/dev/null; then
        useradd -m -s /bin/bash "${username}"
    fi

    # SSHキーの設定
    local ssh_dir="/home/${username}/.ssh"
    mkdir -p "${ssh_dir}"
    chmod 700 "${ssh_dir}"

    if [[ -n "${SSH_PUBLIC_KEY:-}" ]]; then
        echo "${SSH_PUBLIC_KEY}" >> "${ssh_dir}/authorized_keys"
        chmod 600 "${ssh_dir}/authorized_keys}"
        chown -R "${username}:${username}" "${ssh_dir}"
    fi
}

configure_nginx() {
    local domain="${1}"
    local app_port="${2:-3000}"

    log "INFO" "Configuring Nginx for ${domain}"

    cat > "/etc/nginx/sites-available/${domain}" <<EOF
upstream app_server {
    server 127.0.0.1:${app_port};
}

server {
    listen 80;
    server_name ${domain};

    location / {
        proxy_pass http://app_server;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF

    ln -sf "/etc/nginx/sites-available/${domain}" "/etc/nginx/sites-enabled/"
    nginx -t && systemctl reload nginx
}

setup_ssl() {
    local domain="${1}"

    log "INFO" "Setting up SSL for ${domain}"

    if ! command -v certbot &>/dev/null; then
        apt-get install -y certbot python3-certbot-nginx
    fi

    # 証明書の取得
    certbot --nginx -d "${domain}" \
        --non-interactive \
        --agree-tos \
        -m "admin@${domain}"

    # 自動更新の設定
    cat > /etc/cron.d/certbot-renew <<EOF
0 3 * * * root certbot renew --quiet --post-hook "systemctl reload nginx"
EOF
}

configure_firewall() {
    log "INFO" "Configuring firewall"

    if command -v ufw &>/dev/null; then
        ufw allow ssh
        ufw allow http
        ufw allow https
        ufw --force enable
    fi
}

provision() {
    local domain="${1:?Domain required}"

    log "INFO" "=== Starting Server Provisioning ==="

    install_system_packages
    install_nodejs 18
    setup_app_user "deploy"
    configure_nginx "${domain}" 3000
    setup_ssl "${domain}"
    configure_firewall

    log "INFO" "=== Provisioning Complete ==="
}

provision "$@"
```

## タスクスケジューリング

### Cron設定の自動化

```bash
#!/usr/bin/env bash
# setup-cron.sh

setup_backup_job() {
    local cron_file="/etc/cron.d/backup"

    cat > "${cron_file}" <<EOF
# Database backup (daily at 2:00 AM)
0 2 * * * ${DEPLOY_USER} /usr/local/bin/backup-db.sh >> /var/log/backup.log 2>&1

# Log cleanup (weekly on Sunday at 0:00)
0 0 * * 0 ${DEPLOY_USER} /usr/local/bin/cleanup-logs.sh >> /var/log/cleanup.log 2>&1

# Health check (every 5 minutes)
*/5 * * * * ${DEPLOY_USER} /usr/local/bin/health-check.sh >> /var/log/health.log 2>&1
EOF

    chmod 644 "${cron_file}"
    log "INFO" "Cron jobs configured"
}
```

### Systemd タイマーの使用

```bash
#!/usr/bin/env bash
# setup-systemd-timer.sh

create_backup_timer() {
    # サービスファイル
    cat > /etc/systemd/system/backup.service <<EOF
[Unit]
Description=Database Backup Service
After=postgresql.service

[Service]
Type=oneshot
User=${DEPLOY_USER}
ExecStart=/usr/local/bin/backup-db.sh
StandardOutput=journal
StandardError=journal
EOF

    # タイマーファイル
    cat > /etc/systemd/system/backup.timer <<EOF
[Unit]
Description=Database Backup Timer
Requires=backup.service

[Timer]
OnCalendar=daily
OnCalendar=02:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

    systemctl daemon-reload
    systemctl enable backup.timer
    systemctl start backup.timer

    log "INFO" "Systemd timer configured"
}
```

## 通知システム

### マルチチャネル通知

```bash
#!/usr/bin/env bash
# notify.sh

send_notification() {
    local message="${1}"
    local level="${2:-info}"
    local channels="${3:-slack}"

    IFS=',' read -ra channel_array <<< "${channels}"

    for channel in "${channel_array[@]}"; do
        case "${channel}" in
            slack) notify_slack "${message}" "${level}" ;;
            discord) notify_discord "${message}" ;;
            teams) notify_teams "${message}" "${level}" ;;
            email) notify_email "${message}" ;;
        esac
    done
}

notify_slack() {
    local message="${1}"
    local level="${2}"

    if [[ -z "${SLACK_WEBHOOK:-}" ]]; then
        return 0
    fi

    local color="good"
    case "${level}" in
        error) color="danger" ;;
        warning) color="warning" ;;
    esac

    curl -X POST "${SLACK_WEBHOOK}" \
        -H 'Content-Type: application/json' \
        -d "{
            \"text\": \"${message}\",
            \"attachments\": [{
                \"color\": \"${color}\"
            }]
        }"
}

# 使用例
send_notification "Deployment completed successfully" "success" "slack,teams"
```

## まとめ

本章では、実践的なデプロイメント自動化を学びました：

- **デプロイメント戦略**: Zero-Downtime、Blue-Green
- **環境管理**: プロビジョニング、設定管理
- **タスクスケジューリング**: Cron、Systemd
- **通知システム**: マルチチャネル通知

次章では、メンテナンスと運用スクリプトについて学びます。
