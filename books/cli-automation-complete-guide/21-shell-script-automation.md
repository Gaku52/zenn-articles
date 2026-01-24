# Shellスクリプト自動化

本章では、Bashスクリプトを使った自動化の基礎を学びます。

## 基本構文

```bash
#!/bin/bash

set -euo pipefail

NAME="myapp"
ENV=${1:-development}

echo "Deploying $NAME to $ENV..."

if [ "$ENV" = "production" ]; then
    echo "Production deployment"
else
    echo "Development deployment"
fi
```

## 関数

```bash
deploy() {
    local env=$1
    echo "Deploying to $env..."
    npm run build
    ssh user@server "cd /var/www && git pull"
}

deploy production
```

## エラーハンドリング

```bash
set -e  # エラー時に終了
set -u  # 未定義変数使用時にエラー
set -o pipefail  # パイプライン内のエラーを検知
```

## まとめ

Shellスクリプトにより、デプロイメントやバッチ処理を効率的に自動化できます。
