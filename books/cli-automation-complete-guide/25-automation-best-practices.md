# 自動化ベストプラクティス

本章では、スクリプト自動化のベストプラクティスを学びます。

## べき等性の確保

```bash
# 推奨されるパターン: 何度実行しても同じ結果
mkdir -p /var/www/app  # ディレクトリが既に存在してもエラーにならない
```

## ログ出力

```bash
LOG_FILE="/var/log/deploy_$(date +%Y%m%d).log"

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

log "INFO" "Starting deployment..."
```

## エラー通知

```bash
deploy() {
    if ! npm run build; then
        curl -X POST "$SLACK_WEBHOOK" \
            -d '{"text": "Deployment failed!"}'
        exit 1
    fi
}
```

## ドライランモード

```bash
DRY_RUN=${DRY_RUN:-false}

execute() {
    if [ "$DRY_RUN" = "true" ]; then
        echo "[DRY RUN] Would execute: $*"
    else
        "$@"
    fi
}

execute rm -rf /tmp/cache
```

## まとめ

本章では、自動化スクリプトのベストプラクティスを学びました。

**重要ポイント**:
- べき等性を確保し、何度実行しても安全に
- ログ出力で実行履歴を記録
- エラー時は適切に通知
- ドライランモードで安全性を確保

これらを実践することで、信頼性の高い自動化システムを構築できます。

本書を通じて、CLI開発とスクリプト自動化の全体像を学びました。これらの知識を活かし、効率的で保守性の高いツールを開発してください。
