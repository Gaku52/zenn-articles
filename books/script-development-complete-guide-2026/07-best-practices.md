---
title: "ベストプラクティスとパターン集"
---

# ベストプラクティスとパターン集

本章では、これまで学んだ内容を踏まえ、スクリプト開発のベストプラクティスと実践的なパターンをまとめます。

## スクリプト開発の原則

### 1. 読みやすさを重視する

**コードは書く時間よりも読む時間の方が長い**ため、可読性を最優先にしましょう。

```bash
# ❌ 悪い例
f=/var/log/app.log;s=$(wc -l<$f);echo "$s lines"

# ✅ 良い例
readonly LOG_FILE="/var/log/app.log"
line_count="$(wc -l < "${LOG_FILE}")"
echo "Log file contains ${line_count} lines"
```

### 2. エラーハンドリングを徹底する

すべてのスクリプトに適切なエラーハンドリングを実装しましょう。

```bash
# Bashの基本設定
set -euo pipefail

# エラーハンドラ
trap 'echo "Error on line $LINENO" >&2' ERR
trap 'cleanup' EXIT
```

```python
# Pythonの例外処理
try:
    result = process_data(input_file)
except FileNotFoundError as e:
    logger.error(f"File not found: {e}")
    sys.exit(1)
except Exception as e:
    logger.error(f"Unexpected error: {e}")
    sys.exit(1)
```

### 3. べき等性を確保する

何度実行しても同じ結果になるようにスクリプトを設計しましょう。

```bash
# べき等な操作の例
ensure_directory() {
    local dir_path="${1}"

    # ディレクトリが既に存在する場合は何もしない
    if [[ ! -d "${dir_path}" ]]; then
        mkdir -p "${dir_path}"
    fi
}

# シンボリックリンクの作成
ln -nfs /source /target  # -n オプションでべき等性を確保
```

### 4. ログ出力を適切に行う

実行状況を追跡できるよう、適切なログ出力を実装しましょう。

```bash
# 構造化ログ
log() {
    local level="${1}"
    shift
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] [${level}] $*" | tee -a "${LOG_FILE}"
}

log "INFO" "Processing started"
log "ERROR" "Failed to connect to database"
```

```python
# Pythonの構造化ログ
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

logger.info("Processing started", extra={
    'file': input_file,
    'rows': row_count
})
```

### 5. テスタビリティを考慮する

テストしやすい構造でスクリプトを設計しましょう。

```bash
# テスト可能な関数設計
calculate_total() {
    local price="${1}"
    local quantity="${2}"
    echo $((price * quantity))
}

# テスト
test_calculate_total() {
    local result
    result="$(calculate_total 100 5)"

    if [[ "${result}" == "500" ]]; then
        echo "✓ Test passed"
    else
        echo "✗ Test failed: expected 500, got ${result}"
        return 1
    fi
}
```

## セキュリティベストプラクティス

### 1. 機密情報の管理

```bash
# ❌ 悪い例: ハードコードされた機密情報
DB_PASSWORD="secret123"

# ✅ 良い例: 環境変数から取得
DB_PASSWORD="${DB_PASSWORD:?Database password not set}"

# さらに良い例: ファイルから読み込み
if [[ -f "${HOME}/.db_credentials" ]]; then
    source "${HOME}/.db_credentials"
fi
```

### 2. 入力値の検証

```bash
# 入力値の検証
validate_email() {
    local email="${1}"

    if [[ ! "${email}" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
        echo "Invalid email format: ${email}" >&2
        return 1
    fi
}

# 使用前に必ず検証
validate_email "${USER_EMAIL}" || exit 1
```

### 3. 安全なファイル操作

```bash
# 一時ファイルの安全な作成
temp_file="$(mktemp)"
trap "rm -f '${temp_file}'" EXIT

# パーミッションの設定
chmod 600 "${temp_file}"

# アトミックな書き込み
echo "data" > "${temp_file}"
mv "${temp_file}" "${target_file}"
```

## パフォーマンス最適化

### 1. 不要な処理を避ける

```bash
# ❌ 悪い例: 毎回コマンドを実行
for file in *.txt; do
    if command -v processor &>/dev/null; then
        processor "${file}"
    fi
done

# ✅ 良い例: 事前にチェック
if ! command -v processor &>/dev/null; then
    echo "processor not found" >&2
    exit 1
fi

for file in *.txt; do
    processor "${file}"
done
```

### 2. 並列処理の活用

```bash
# 並列処理による高速化
process_files_parallel() {
    local max_jobs=4

    for file in *.txt; do
        {
            process_file "${file}"
        } &

        # ジョブ数を制限
        while (($(jobs -r | wc -l) >= max_jobs)); do
            sleep 0.1
        done
    done

    wait  # すべての完了を待機
}
```

```python
# Pythonの並列処理
from concurrent.futures import ThreadPoolExecutor

def process_files_parallel(files, max_workers=4):
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        results = list(executor.map(process_file, files))
    return results
```

### 3. キャッシュの活用

```bash
# 結果のキャッシュ
readonly CACHE_DIR="${HOME}/.cache/myscript"
readonly CACHE_TTL=3600  # 1時間

get_cached_result() {
    local cache_key="${1}"
    local cache_file="${CACHE_DIR}/${cache_key}"

    # キャッシュが有効かチェック
    if [[ -f "${cache_file}" ]]; then
        local cache_age
        cache_age=$(($(date +%s) - $(stat -f%m "${cache_file}" 2>/dev/null || stat -c%Y "${cache_file}" 2>/dev/null)))

        if ((cache_age < CACHE_TTL)); then
            cat "${cache_file}"
            return 0
        fi
    fi

    return 1
}

save_to_cache() {
    local cache_key="${1}"
    local data="${2}"

    mkdir -p "${CACHE_DIR}"
    echo "${data}" > "${CACHE_DIR}/${cache_key}"
}
```

## コードの構造化

### 1. モジュール化

```bash
# lib/common.sh - 共通関数ライブラリ
#!/usr/bin/env bash

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" >&2
}

ensure_directory() {
    local dir="${1}"
    [[ -d "${dir}" ]] || mkdir -p "${dir}"
}

# メインスクリプト
#!/usr/bin/env bash

readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${SCRIPT_DIR}/lib/common.sh"

main() {
    log "Starting process"
    ensure_directory "/tmp/output"
}

main "$@"
```

### 2. 設定の外部化

```bash
# config/default.conf
APP_NAME="myapp"
LOG_LEVEL="INFO"
MAX_RETRIES=3

# スクリプト
#!/usr/bin/env bash

readonly CONFIG_FILE="${CONFIG_FILE:-config/default.conf}"

if [[ -f "${CONFIG_FILE}" ]]; then
    source "${CONFIG_FILE}"
fi

# 環境変数で上書き可能
APP_NAME="${APP_NAME:-myapp}"
LOG_LEVEL="${LOG_LEVEL:-INFO}"
```

### 3. ヘルプとドキュメンテーション

```bash
#!/usr/bin/env bash
# my-script.sh - スクリプトの説明

usage() {
    cat << EOF
Usage: $(basename "$0") [OPTIONS] <command>

スクリプトの詳細な説明

Options:
    -h, --help              このヘルプを表示
    -v, --verbose           詳細な出力
    -c, --config FILE       設定ファイルを指定
    -n, --dry-run           ドライランモード

Commands:
    process <file>          ファイルを処理
    clean                   クリーンアップ
    status                  ステータス確認

Examples:
    $(basename "$0") process input.txt
    $(basename "$0") --verbose --config custom.conf clean

EOF
    exit 0
}
```

## アンチパターンと回避方法

### 1. グローバル変数への依存

```bash
# ❌ 悪い例
process_data() {
    # グローバル変数に依存
    result="${input_file}_processed"
}

# ✅ 良い例
process_data() {
    local input_file="${1}"
    local output_file="${input_file}_processed"
    echo "${output_file}"
}
```

### 2. エラーの無視

```bash
# ❌ 悪い例
wget https://example.com/file.txt 2>/dev/null

# ✅ 良い例
if ! wget https://example.com/file.txt; then
    echo "Failed to download file" >&2
    exit 1
fi
```

### 3. ハードコードされたパス

```bash
# ❌ 悪い例
LOG_FILE="/var/log/myapp/app.log"

# ✅ 良い例
readonly LOG_DIR="${LOG_DIR:-/var/log/myapp}"
readonly LOG_FILE="${LOG_DIR}/app.log"
```

## デバッグとトラブルシューティング

### 1. デバッグ出力

```bash
# デバッグモードの実装
readonly DEBUG="${DEBUG:-false}"

debug() {
    if [[ "${DEBUG}" == "true" ]]; then
        echo "[DEBUG] $*" >&2
    fi
}

# 使用例
debug "Processing file: ${file}"
debug "Variables: input=${input}, output=${output}"
```

### 2. ステップバイステップ実行

```bash
# set -x でコマンドトレース
#!/usr/bin/env bash

if [[ "${DEBUG}" == "true" ]]; then
    set -x
fi

# 処理内容
```

### 3. ログの活用

```bash
# 詳細なログ出力
exec > >(tee -a "${LOG_FILE}")
exec 2>&1

echo "=== Script started at $(date) ==="
echo "Arguments: $*"
echo "Environment: ${ENVIRONMENT}"
```

## チェックリスト

スクリプトをリリースする前の確認項目：

### 機能要件
- [ ] すべての要件を満たしている
- [ ] エッジケースを処理できる
- [ ] エラー処理が適切に実装されている

### コード品質
- [ ] コードレビューを実施した
- [ ] ShellCheck / Linter でチェックした
- [ ] テストを書いて実行した
- [ ] ドキュメントを作成した

### セキュリティ
- [ ] 機密情報がハードコードされていない
- [ ] 入力値の検証を実装した
- [ ] ファイルパーミッションが適切
- [ ] SQLインジェクション等の脆弱性がない

### 運用性
- [ ] ログ出力が適切
- [ ] エラーメッセージが分かりやすい
- [ ] ロールバック手順がある
- [ ] 監視・アラートが設定されている

## 推奨ツール

### Shell スクリプト
- **ShellCheck**: 静的解析ツール
- **bats**: テストフレームワーク
- **shellspec**: BDDスタイルのテストフレームワーク

### Python
- **black**: コードフォーマッター
- **mypy**: 型チェッカー
- **pytest**: テストフレームワーク
- **pylint**: リンター

### Node.js/TypeScript
- **ESLint**: リンター
- **Prettier**: コードフォーマッター
- **Jest**: テストフレームワーク
- **ts-node**: TypeScript実行環境

## 学習リソース

### 公式ドキュメント
- [Bash Reference Manual](https://www.gnu.org/software/bash/manual/)
- [Python Documentation](https://docs.python.org/3/)
- [Node.js Documentation](https://nodejs.org/docs/)

### スタイルガイド
- [Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html)
- [PEP 8 -- Style Guide for Python Code](https://pep8.org/)
- [Airbnb JavaScript Style Guide](https://github.com/airbnb/javascript)

### コミュニティ
- Stack Overflow
- GitHub Discussions
- Reddit (r/bash, r/python, r/node)

## まとめ

本書を通じて学んだ内容：

1. **Shell スクリプト**: 基本構文から実践的なパターンまで
2. **Python スクリプト**: データ処理、API連携、並行処理
3. **Node.js/TypeScript**: 型安全な現代的スクリプト開発
4. **自動化とデプロイメント**: CI/CD統合、環境管理
5. **メンテナンスと運用**: バックアップ、ログ管理、モニタリング
6. **ベストプラクティス**: セキュリティ、パフォーマンス、可読性

### 次のステップ

1. **実践する**: 学んだパターンをプロジェクトで活用
2. **共有する**: チームでベストプラクティスを共有
3. **改善する**: フィードバックを元に継続的に改善
4. **学び続ける**: 新しい技術やパターンをキャッチアップ

スクリプト開発は、エンジニアの生産性を大きく向上させる重要なスキルです。本書で学んだ知識を活かし、堅牢で保守性の高いスクリプトを書けるようになりましょう。

最後まで読んでいただき、ありがとうございました。
