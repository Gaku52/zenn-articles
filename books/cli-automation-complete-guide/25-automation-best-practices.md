---
title: "自動化ベストプラクティス"
---

# 自動化ベストプラクティス

本章では、スクリプト自動化における重要なベストプラクティスを学びます。これらの原則を実践することで、信頼性が高く、保守しやすい自動化システムを構築できます。

## 冪等性の確保

冪等性とは、同じ操作を何度実行しても同じ結果になる性質です。自動化スクリプトにおいて冪等性は極めて重要です。

### Shellスクリプトでの冪等性

```bash
#!/bin/bash

set -euo pipefail

# 非冪等な例（推奨されない）
mkdir /var/www/app  # 2回目の実行でエラー

# 冪等な例（推奨される）
mkdir -p /var/www/app  # 既に存在してもエラーにならない

# ファイルコピーの冪等性
if [ ! -f "/etc/myapp/config.conf" ]; then
    cp config.conf /etc/myapp/config.conf
fi

# rsyncによる冪等なコピー
rsync -av --delete src/ dest/

# シンボリックリンクの冪等作成
ln -sfn /opt/myapp/current /var/www/myapp

# システムサービスの冪等起動
systemctl is-active --quiet myapp || systemctl start myapp

# 設定ファイルの冪等更新
if ! grep -q "^export PATH=" ~/.bashrc; then
    echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
fi
```

### Pythonでの冪等性

```python
#!/usr/bin/env python3

from pathlib import Path
import json

def ensure_directory(path: Path) -> None:
    """ディレクトリの冪等作成"""
    path.mkdir(parents=True, exist_ok=True)

def update_config(config_file: Path, updates: dict) -> None:
    """設定ファイルの冪等更新"""
    # 既存設定読み込み
    if config_file.exists():
        with config_file.open('r') as f:
            config = json.load(f)
    else:
        config = {}

    # 設定更新
    config.update(updates)

    # 書き込み
    with config_file.open('w') as f:
        json.dump(config, f, indent=2)

# 使用例
ensure_directory(Path('/var/www/app'))
update_config(
    Path('/etc/myapp/config.json'),
    {'version': '1.0.0', 'debug': False}
)
```

### Node.jsでの冪等性

```typescript
import * as fs from 'fs/promises'
import * as path from 'path'

/**
 * ディレクトリの冪等作成
 */
async function ensureDirectory(dirPath: string): Promise<void> {
    try {
        await fs.mkdir(dirPath, { recursive: true })
    } catch (error) {
        // 既に存在する場合は無視
        if (error.code !== 'EEXIST') {
            throw error
        }
    }
}

/**
 * 設定ファイルの冪等更新
 */
async function updateConfig(
    configFile: string,
    updates: Record<string, any>
): Promise<void> {
    let config: Record<string, any> = {}

    // 既存設定読み込み
    try {
        const content = await fs.readFile(configFile, 'utf-8')
        config = JSON.parse(content)
    } catch (error) {
        // ファイルが存在しない場合は新規作成
    }

    // 設定更新
    config = { ...config, ...updates }

    // 書き込み
    await fs.writeFile(
        configFile,
        JSON.stringify(config, null, 2),
        'utf-8'
    )
}

// 使用例
await ensureDirectory('/var/www/app')
await updateConfig('/etc/myapp/config.json', {
    version: '1.0.0',
    debug: false,
})
```

## ドライランモード実装

ドライランモードは、実際の変更を行わずに処理内容を確認できる機能です。

### Shellスクリプト

```bash
#!/bin/bash

set -euo pipefail

# ドライランモードのフラグ
DRY_RUN=${DRY_RUN:-false}

# ログ関数
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*"
}

# 実行関数（ドライラン対応）
execute() {
    if [ "$DRY_RUN" = "true" ]; then
        log "[DRY RUN] Would execute: $*"
    else
        log "Executing: $*"
        "$@"
    fi
}

# 使用例
log "Starting deployment..."

execute mkdir -p /var/www/app
execute rsync -av dist/ /var/www/app/
execute systemctl restart myapp

log "Deployment completed"

# 実行方法:
# 通常実行: ./deploy.sh
# ドライラン: DRY_RUN=true ./deploy.sh
```

### Python

```python
#!/usr/bin/env python3

import argparse
import logging
from pathlib import Path
import subprocess
from typing import List

class AutomationScript:
    """自動化スクリプト（ドライランモード対応）"""

    def __init__(self, dry_run: bool = False):
        self.dry_run = dry_run
        self.logger = logging.getLogger(__name__)

    def execute(self, command: List[str]) -> None:
        """コマンド実行（ドライラン対応）"""
        cmd_str = ' '.join(command)

        if self.dry_run:
            self.logger.info(f'[DRY RUN] Would execute: {cmd_str}')
        else:
            self.logger.info(f'Executing: {cmd_str}')
            subprocess.run(command, check=True)

    def write_file(self, file_path: Path, content: str) -> None:
        """ファイル書き込み（ドライラン対応）"""
        if self.dry_run:
            self.logger.info(f'[DRY RUN] Would write to: {file_path}')
            self.logger.debug(f'Content preview:\n{content[:100]}...')
        else:
            self.logger.info(f'Writing to: {file_path}')
            file_path.write_text(content, encoding='utf-8')

    def deploy(self) -> None:
        """デプロイ処理"""
        self.logger.info('Starting deployment')

        # ディレクトリ作成
        deploy_dir = Path('/var/www/app')
        if not self.dry_run:
            deploy_dir.mkdir(parents=True, exist_ok=True)
        else:
            self.logger.info(f'[DRY RUN] Would create directory: {deploy_dir}')

        # ビルド
        self.execute(['npm', 'run', 'build'])

        # デプロイ
        self.execute(['rsync', '-av', 'dist/', str(deploy_dir) + '/'])

        # サービス再起動
        self.execute(['systemctl', 'restart', 'myapp'])

        self.logger.info('Deployment completed')

def main() -> None:
    parser = argparse.ArgumentParser(description='Deployment script')
    parser.add_argument(
        '--dry-run',
        action='store_true',
        help='Dry run mode (no actual changes)',
    )
    parser.add_argument('-v', '--verbose', action='store_true')
    args = parser.parse_args()

    logging.basicConfig(
        level=logging.DEBUG if args.verbose else logging.INFO,
        format='%(levelname)s: %(message)s',
    )

    script = AutomationScript(dry_run=args.dry_run)
    script.deploy()

if __name__ == '__main__':
    main()
```

### Node.js/TypeScript

```typescript
#!/usr/bin/env ts-node

import { execSync } from 'child_process'
import * as fs from 'fs/promises'
import { Command } from 'commander'

class AutomationScript {
    constructor(private dryRun: boolean = false) {}

    /**
     * コマンド実行（ドライラン対応）
     */
    async execute(command: string): Promise<void> {
        if (this.dryRun) {
            console.log(`[DRY RUN] Would execute: ${command}`)
        } else {
            console.log(`Executing: ${command}`)
            execSync(command, { stdio: 'inherit' })
        }
    }

    /**
     * ファイル書き込み（ドライラン対応）
     */
    async writeFile(filePath: string, content: string): Promise<void> {
        if (this.dryRun) {
            console.log(`[DRY RUN] Would write to: ${filePath}`)
            console.log(`Content preview:\n${content.substring(0, 100)}...`)
        } else {
            console.log(`Writing to: ${filePath}`)
            await fs.writeFile(filePath, content, 'utf-8')
        }
    }

    /**
     * デプロイ処理
     */
    async deploy(): Promise<void> {
        console.log('Starting deployment')

        // ビルド
        await this.execute('npm run build')

        // デプロイ
        await this.execute('rsync -av dist/ /var/www/app/')

        // サービス再起動
        await this.execute('systemctl restart myapp')

        console.log('Deployment completed')
    }
}

// CLI設定
const program = new Command()

program
    .option('-d, --dry-run', 'Dry run mode (no actual changes)')
    .option('-v, --verbose', 'Verbose output')
    .parse()

const options = program.opts()

// 実行
const script = new AutomationScript(options.dryRun)
script.deploy().catch(console.error)
```

## 適切なログレベル

ログレベルを適切に使い分けることで、運用時のトラブルシューティングが容易になります。

### ログレベルの使い分け

```python
#!/usr/bin/env python3

import logging
from pathlib import Path

def setup_logging(verbose: bool = False) -> logging.Logger:
    """ロガー設定"""
    logger = logging.getLogger(__name__)

    # ハンドラー設定
    handler = logging.StreamHandler()
    formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    )
    handler.setFormatter(formatter)
    logger.addHandler(handler)

    # レベル設定
    logger.setLevel(logging.DEBUG if verbose else logging.INFO)

    return logger

def process_files(directory: Path, logger: logging.Logger) -> None:
    """ファイル処理"""
    logger.info(f'Processing directory: {directory}')

    files = list(directory.glob('*.txt'))
    logger.debug(f'Found {len(files)} files')

    for file_path in files:
        logger.debug(f'Processing file: {file_path}')

        try:
            content = file_path.read_text()
            logger.debug(f'File size: {len(content)} bytes')

            if len(content) == 0:
                logger.warning(f'Empty file: {file_path}')
                continue

            # 処理実行
            processed = content.upper()
            file_path.write_text(processed)

            logger.info(f'Processed: {file_path.name}')
        except Exception as e:
            logger.error(f'Failed to process {file_path}: {e}')
            logger.exception('Detailed error:')

# ログレベルガイドライン:
# DEBUG: 詳細なデバッグ情報（ファイルサイズ、変数値など）
# INFO: 通常の処理進行状況（処理開始、完了など）
# WARNING: 警告（空ファイル、推奨されない設定など）
# ERROR: エラー（処理失敗、例外発生など）
# CRITICAL: 致命的エラー（システム停止など）
```

## エラーリカバリー

エラー発生時の適切なリカバリー処理を実装します。

### リトライ機構

```python
#!/usr/bin/env python3

import time
import logging
from typing import Callable, TypeVar, Optional
from functools import wraps

T = TypeVar('T')

def retry(
    max_attempts: int = 3,
    delay: float = 1.0,
    backoff: float = 2.0,
    exceptions: tuple = (Exception,)
):
    """リトライデコレーター"""
    def decorator(func: Callable[..., T]) -> Callable[..., T]:
        @wraps(func)
        def wrapper(*args, **kwargs) -> T:
            current_delay = delay

            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts:
                        logging.error(f'Failed after {max_attempts} attempts')
                        raise

                    logging.warning(
                        f'Attempt {attempt}/{max_attempts} failed: {e}. '
                        f'Retrying in {current_delay}s...'
                    )
                    time.sleep(current_delay)
                    current_delay *= backoff

            raise Exception('Unreachable')

        return wrapper
    return decorator

# 使用例
@retry(max_attempts=3, delay=1.0, backoff=2.0)
def fetch_data(url: str) -> dict:
    """データ取得（リトライ付き）"""
    import requests
    response = requests.get(url, timeout=10)
    response.raise_for_status()
    return response.json()
```

### ロールバック機構

```python
#!/usr/bin/env python3

import shutil
from pathlib import Path
from typing import Optional
import logging

class DeploymentManager:
    """デプロイメント管理（ロールバック対応）"""

    def __init__(self, deploy_dir: Path):
        self.deploy_dir = deploy_dir
        self.backup_dir: Optional[Path] = None
        self.logger = logging.getLogger(__name__)

    def backup(self) -> None:
        """現在のバージョンをバックアップ"""
        if not self.deploy_dir.exists():
            self.logger.info('No existing deployment to backup')
            return

        self.backup_dir = self.deploy_dir.parent / f'{self.deploy_dir.name}.backup'

        if self.backup_dir.exists():
            shutil.rmtree(self.backup_dir)

        shutil.copytree(self.deploy_dir, self.backup_dir)
        self.logger.info(f'Backup created: {self.backup_dir}')

    def deploy(self, source_dir: Path) -> None:
        """デプロイ実行"""
        try:
            self.logger.info('Starting deployment')

            # バックアップ作成
            self.backup()

            # デプロイ実行
            if self.deploy_dir.exists():
                shutil.rmtree(self.deploy_dir)

            shutil.copytree(source_dir, self.deploy_dir)

            # ヘルスチェック
            if not self.health_check():
                raise Exception('Health check failed')

            self.logger.info('Deployment successful')

            # バックアップ削除
            if self.backup_dir and self.backup_dir.exists():
                shutil.rmtree(self.backup_dir)

        except Exception as e:
            self.logger.error(f'Deployment failed: {e}')
            self.rollback()
            raise

    def rollback(self) -> None:
        """ロールバック実行"""
        if not self.backup_dir or not self.backup_dir.exists():
            self.logger.error('No backup available for rollback')
            return

        self.logger.info('Rolling back to previous version')

        if self.deploy_dir.exists():
            shutil.rmtree(self.deploy_dir)

        shutil.move(str(self.backup_dir), str(self.deploy_dir))
        self.logger.info('Rollback completed')

    def health_check(self) -> bool:
        """ヘルスチェック"""
        # 実際のヘルスチェック処理
        return True

# 使用例
manager = DeploymentManager(Path('/var/www/app'))
manager.deploy(Path('./dist'))
```

## スケジューリング

定期的な自動実行の設定方法です。

### cron

```bash
# crontab -e

# 毎日午前2時にバックアップ実行
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1

# 毎週月曜日午前3時にデータベースメンテナンス
0 3 * * 1 /usr/local/bin/db-maintenance.py

# 5分ごとにヘルスチェック
*/5 * * * * /usr/local/bin/health-check.sh

# 毎月1日午前0時にレポート生成
0 0 1 * * /usr/local/bin/generate-report.py
```

### GitHub Actions Schedule

```yaml
# .github/workflows/scheduled.yml
name: Scheduled Tasks

on:
  schedule:
    # 毎日午前2時（UTC）
    - cron: '0 2 * * *'
  workflow_dispatch:  # 手動実行も可能

jobs:
  backup:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Run backup
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: python scripts/backup.py

      - name: Notify
        if: failure()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -H 'Content-Type: application/json' \
            -d '{"text": "Backup failed!"}'
```

## 自動化スクリプト品質チェックリスト

### 設計
- [ ] 冪等性を確保している
- [ ] ドライランモードを実装している
- [ ] 適切なエラーハンドリングを実装している
- [ ] ロールバック機構を実装している

### ログ
- [ ] 適切なログレベルを使用している
- [ ] ログファイルに出力している
- [ ] タイムスタンプを含めている
- [ ] 機密情報をログに出力していない

### エラーハンドリング
- [ ] リトライ機構を実装している
- [ ] エラー時の通知を実装している
- [ ] 詳細なエラーメッセージを提供している

### セキュリティ
- [ ] 機密情報を環境変数で管理している
- [ ] ファイル権限を適切に設定している
- [ ] 入力値のバリデーションを実施している

### テスト
- [ ] ドライランモードでテストしている
- [ ] 各種エラーケースをテストしている
- [ ] ロールバック処理をテストしている

### ドキュメント
- [ ] 使用方法を記述している
- [ ] 必要な権限を記述している
- [ ] エラー時の対処方法を記述している

## まとめ

本章では、自動化スクリプトのベストプラクティスを学びました。

**重要ポイント**:
- 冪等性により何度実行しても安全に
- ドライランモードで安全性を確保
- 適切なログレベルで運用を容易に
- エラーリカバリーで信頼性を向上
- スケジューリングで定期実行を自動化

これらを実践することで、信頼性の高い自動化システムを構築できます。

本書を通じて、CLI開発とスクリプト自動化の全体像を学びました。これらの知識を活かし、効率的で保守性の高いツールを開発してください。

## 参考文献

- [The Twelve-Factor App](https://12factor.net/)
- [DevOps Best Practices](https://cloud.google.com/architecture/devops)
- [Bash Best Practices - Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html)
- [Python Logging HOWTO](https://docs.python.org/3/howto/logging.html)
