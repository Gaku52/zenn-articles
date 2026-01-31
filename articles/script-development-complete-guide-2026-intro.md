---
title: "【2026年版】スクリプト開発完全ガイドを公開しました【Shell・Python・Node.js自動化の実践】"
emoji: "🔧"
type: "tech"
topics: ["bash", "python", "nodejs", "automation", "devops"]
published: false
---

# スクリプト開発完全ガイド 2026 を公開しました

## こんな状況、ありませんか？

金曜の夕方、17時30分。

デプロイ作業の手順書を開いて、一つずつコマンドをコピペ。

```bash
$ cd /tmp/build
$ git clone https://...
$ cd repo
$ npm install
```

途中でタイポして、やり直し。
あるコマンドが失敗しても、次のコマンドを実行してしまい、後で気づく。

「これ、毎週やってるんだけど...自動化できないの？」

---

月曜の朝。
先輩が書いたログ集計スクリプトをメンテしようとするが、

```bash
cat log.txt | grep ERROR | awk '{print $3}' | sort | uniq -c | ...
```

「これ、何してるんだ...？」

コメントなし。変数名は `a`、`b`、`tmp`。
半年前の自分が書いたスクリプトすら、もう理解できない。

---

こんな経験、ありませんか？

開発現場では日々多くの定型作業が発生しますが、適切な自動化スクリプトがあれば、それらの作業時間を大幅に削減できる可能性があります。

**よく聞く課題：**

- 手作業による繰り返し作業で時間を消費
- スクリプトのエラーハンドリングが不十分
- Shell、Python、Node.jsの使い分けがわからない
- 保守性の低いワンライナーが量産される

そこで、**Shell、Python、Node.jsを使った効率的なスクリプト開発のベストプラクティスを体系的にまとめた完全ガイド**を執筆しました。

https://zenn.dev/gaku/books/script-development-complete-guide-2026

## なぜスクリプト開発で挫折するのか

### 1. 「とりあえず動けばいい」で保守性を犠牲にしている

締め切り間近。
「とりあえず動けばいいから...」と書いたスクリプト。

3ヶ月後、そのスクリプトが突然エラーを吐き始める。
エラーメッセージは出ない。ログもない。
どこで失敗したのかすら、わからない。

スクリプトは簡単に書けるため、保守性やエラーハンドリングが軽視されがちです。

**よくある失敗:**
- エラーチェックがない
- ハードコードされた値が散在
- コメントがなく半年後に理解できない
- 環境依存で他の人が実行できない

「動くこと」と「保守できること」は、別の話です。

### 2. 言語の選択基準が不明確

「JSON処理、Shellでできるよね？」

```bash
response=$(curl -s https://api.example.com/users)
users=$(echo "$response" | grep -o '"name":"[^"]*"' | sed 's/"name":"//;s/"//')
```

書けることと、書くべきかどうかは別問題。

Shell、Python、Node.jsそれぞれに得意・不得意があり、適切に使い分ける必要があります。

```
Shell、Python、Node.js... どれを使えばいいの？
```

**選択を誤ると:**
- Shellで複雑なデータ処理をして可読性が悪化
- Pythonで簡単なファイル操作をして環境構築が面倒に
- Node.jsで同期処理を書いて非効率に

### 3. エラーハンドリングが不十分

スクリプトの実行は自動化されることが多いため、エラーが適切にハンドリングされないと、問題に気づかないまま処理が進んでしまいます。

深夜2時、デプロイスクリプトがエラーで止まっている。
でも、誰も気づいていない。
エラー通知も、ログもない。

朝になって初めて、「あれ、デプロイされてない？」

**考えられる問題:**
- ファイルが存在しない場合の処理
- ネットワークエラー時の挙動
- 権限エラーの検知
- 部分的な失敗の扱い

## よくある3つの間違い

本書で扱う内容から、特によくある間違いを3つ紹介します。

### 間違い1: エラーチェックがない危険なスクリプト

**❌ 悪い例:**

```bash
#!/bin/bash

# エラーチェックなしで実行
cd /tmp/build
rm -rf *
git clone https://github.com/user/repo.git
cd repo
npm install
npm run build
```

**何が起こるか？**

```
$ cd /tmp/build
bash: cd: /tmp/build: No such file or directory
$ rm -rf *
# ← 今いるディレクトリで実行される！
```

想像してみてください。
`cd`が失敗して、現在のディレクトリが`/`や`$HOME`だったら？
`rm -rf *`が実行されたら...

**問題点:**
- `cd`が失敗しても次の処理が実行される
- `/tmp/build`への移動失敗で予期せぬ場所で`rm -rf *`が実行される危険性
- `git clone`失敗時も処理が続行
- エラーが発生しても気づかない

**✅ 正しい例: エラーハンドリング付き**

```bash
#!/bin/bash

# エラー時に即座に終了
set -euo pipefail

# ログ関数
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" >&2
}

error() {
    log "ERROR: $*"
    exit 1
}

# 環境変数のチェック
: "${BUILD_DIR:?BUILD_DIR is not set}"
: "${REPO_URL:?REPO_URL is not set}"

# ビルドディレクトリの作成
log "Creating build directory: ${BUILD_DIR}"
mkdir -p "${BUILD_DIR}" || error "Failed to create build directory"

# ディレクトリ移動
log "Changing to build directory"
cd "${BUILD_DIR}" || error "Failed to change directory to ${BUILD_DIR}"

# 既存のファイルをクリーンアップ
if [ -d "repo" ]; then
    log "Removing existing repo directory"
    rm -rf repo || error "Failed to remove existing repo"
fi

# リポジトリのクローン
log "Cloning repository: ${REPO_URL}"
git clone "${REPO_URL}" repo || error "Failed to clone repository"

# リポジトリディレクトリへ移動
cd repo || error "Failed to change to repo directory"

# Node.jsのバージョンチェック
if ! command -v node &> /dev/null; then
    error "Node.js is not installed"
fi

log "Node.js version: $(node --version)"

# 依存関係のインストール
log "Installing dependencies"
npm ci || error "Failed to install dependencies"

# ビルド実行
log "Building project"
npm run build || error "Failed to build project"

log "Build completed successfully"
```

**ビフォー・アフター:**

| 項目 | 改善前 | 改善後 |
|------|--------|--------|
| **エラー検知** | なし | `set -euo pipefail`で即座に停止 |
| **ログ** | なし | タイムスタンプ付きで進捗を記録 |
| **安全性** | `cd`失敗で危険 | 移動失敗時は即座にエラー終了 |
| **デバッグ** | どこで失敗したか不明 | 明確なエラーメッセージ |
| **再現性** | 環境依存 | 環境変数で設定を外部化 |

**期待できる効果:**
- エラー発生時に即座に停止し、問題の拡大を防ぐ
- ログで処理の進行状況を把握でき、トラブルシューティングが容易に
- 環境変数の検証により、実行前に設定ミスを検知
- わかりやすいエラーメッセージで、問題の特定が迅速に

### 間違い2: 言語の選択ミス

**❌ 悪い例: Shellで複雑なJSON処理**

```bash
#!/bin/bash

# ShellでJSONをパースしようとする
response=$(curl -s https://api.example.com/users)

# 複雑で読みづらいJSONパース
users=$(echo "$response" | grep -o '"name":"[^"]*"' | sed 's/"name":"//;s/"//')

for user in $users; do
    echo "User: $user"
done
```

**何が起こるか？**

```json
{
  "users": [
    {"name": "Alice Smith", "email": "alice@example.com"},
    {"name": "Bob", "email": "bob@example.com"}
  ]
}
```

このJSONに対して、上のスクリプトは正しく動作しません。
スペースを含む名前、ネストされたJSON、配列...すべてが問題になります。

**問題点:**
- JSONパースが不正確で脆弱
- エスケープ処理が不十分
- ネストされたJSONに対応できない
- 保守性が低い

**✅ 正しい例: Pythonで JSON処理**

```python
#!/usr/bin/env python3

import json
import sys
from typing import List, Dict
from urllib.request import urlopen
from urllib.error import URLError

def fetch_users(api_url: str) -> List[Dict]:
    """APIからユーザー情報を取得"""
    try:
        with urlopen(api_url, timeout=10) as response:
            data = json.loads(response.read().decode('utf-8'))
            return data.get('users', [])
    except URLError as e:
        print(f"Error fetching data: {e}", file=sys.stderr)
        sys.exit(1)
    except json.JSONDecodeError as e:
        print(f"Error parsing JSON: {e}", file=sys.stderr)
        sys.exit(1)

def process_users(users: List[Dict]) -> None:
    """ユーザー情報を処理"""
    for user in users:
        name = user.get('name', 'Unknown')
        email = user.get('email', 'N/A')
        age = user.get('age', 0)

        print(f"User: {name}")
        print(f"  Email: {email}")
        print(f"  Age: {age}")
        print()

def main():
    api_url = "https://api.example.com/users"

    print("Fetching users...", file=sys.stderr)
    users = fetch_users(api_url)

    if not users:
        print("No users found", file=sys.stderr)
        return

    print(f"Found {len(users)} users", file=sys.stderr)
    process_users(users)

if __name__ == "__main__":
    main()
```

**ビフォー・アフター:**

| 項目 | Shell版（悪い例） | Python版（良い例） |
|------|------------------|-------------------|
| **JSONパース** | 正規表現で脆弱 | `json`モジュールで確実 |
| **エラー処理** | なし | ネットワーク・パースエラーを捕捉 |
| **型安全性** | なし | 型ヒントで意図を明確化 |
| **保守性** | 低い | 関数分割で見通し良く |
| **拡張性** | 困難 | 容易に機能追加可能 |

**期待できる効果:**
- 正確なJSONパースにより、データの取りこぼしや誤解釈がなくなる
- 型ヒントによる可読性向上で、半年後でもコードの意図が理解できる
- 適切なエラーハンドリングで、問題の早期発見が可能に
- 保守性の高いコード構造により、機能追加や修正が容易に

### 間違い3: ハードコードされた値と環境依存

**❌ 悪い例:**

```python
#!/usr/bin/env python3

import os

# ハードコードされたパス
log_file = "/home/user/logs/app.log"
config_file = "/etc/myapp/config.json"

# ファイル存在チェックなし
with open(config_file) as f:
    config = json.load(f)

# 環境に依存したコマンド
os.system("mysql -u root -ppassword mydb < dump.sql")
```

**何が起こるか？**

```
# 開発マシンでは動く
$ ./deploy.py
OK

# 本番サーバーでは...
$ ./deploy.py
FileNotFoundError: [Errno 2] No such file or directory: '/home/user/logs/app.log'

# 他のメンバーのマシンでは...
$ ./deploy.py
FileNotFoundError: [Errno 2] No such file or directory: '/etc/myapp/config.json'
```

「俺の環境では動くんだけど...」

**問題点:**
- 他の環境で動作しない
- パスワードが平文で書かれている
- ファイルの存在確認がない
- 設定の柔軟性がない

**✅ 正しい例: 環境変数と設定ファイル**

```python
#!/usr/bin/env python3

import os
import sys
import json
from pathlib import Path
from typing import Dict, Optional

def load_config() -> Dict:
    """設定ファイルを読み込む"""
    # 環境変数から設定ファイルのパスを取得
    config_path = os.getenv('CONFIG_PATH', './config.json')
    config_file = Path(config_path)

    if not config_file.exists():
        print(f"Error: Config file not found: {config_file}", file=sys.stderr)
        sys.exit(1)

    try:
        with config_file.open() as f:
            return json.load(f)
    except json.JSONDecodeError as e:
        print(f"Error: Invalid JSON in config file: {e}", file=sys.stderr)
        sys.exit(1)

def get_log_path(config: Dict) -> Path:
    """ログファイルのパスを取得"""
    # 環境変数 > 設定ファイル > デフォルト の優先順位
    log_path = os.getenv('LOG_PATH') or config.get('log_path') or './logs/app.log'
    log_file = Path(log_path)

    # ログディレクトリを作成
    log_file.parent.mkdir(parents=True, exist_ok=True)

    return log_file

def execute_sql_import(config: Dict) -> bool:
    """SQLファイルをインポート"""
    import subprocess

    # データベース接続情報を環境変数から取得
    db_host = os.getenv('DB_HOST', config.get('db_host', 'localhost'))
    db_user = os.getenv('DB_USER', config.get('db_user'))
    db_password = os.getenv('DB_PASSWORD')  # 環境変数からのみ取得
    db_name = os.getenv('DB_NAME', config.get('db_name'))

    if not all([db_user, db_password, db_name]):
        print("Error: Database credentials not provided", file=sys.stderr)
        return False

    sql_file = Path(config.get('sql_dump_path', './dump.sql'))
    if not sql_file.exists():
        print(f"Error: SQL dump file not found: {sql_file}", file=sys.stderr)
        return False

    # mysql コマンドを安全に実行
    cmd = [
        'mysql',
        f'--host={db_host}',
        f'--user={db_user}',
        f'--password={db_password}',
        db_name
    ]

    try:
        with sql_file.open() as f:
            result = subprocess.run(
                cmd,
                stdin=f,
                capture_output=True,
                text=True,
                timeout=300
            )

        if result.returncode != 0:
            print(f"Error: SQL import failed: {result.stderr}", file=sys.stderr)
            return False

        return True
    except subprocess.TimeoutExpired:
        print("Error: SQL import timed out", file=sys.stderr)
        return False
    except Exception as e:
        print(f"Error: {e}", file=sys.stderr)
        return False

def main():
    # 設定を読み込み
    config = load_config()

    # ログパスを取得
    log_path = get_log_path(config)
    print(f"Log file: {log_path}")

    # SQLインポートを実行
    if execute_sql_import(config):
        print("SQL import completed successfully")
    else:
        sys.exit(1)

if __name__ == "__main__":
    main()
```

**設定ファイル例 (config.json):**

```json
{
  "log_path": "./logs/app.log",
  "db_host": "localhost",
  "db_user": "appuser",
  "db_name": "mydb",
  "sql_dump_path": "./dump.sql"
}
```

**環境変数の使用例:**

```bash
# 開発環境
export CONFIG_PATH="./config.dev.json"
export DB_PASSWORD="dev_password"

# 本番環境
export CONFIG_PATH="/etc/myapp/config.prod.json"
export DB_PASSWORD="$(cat /run/secrets/db_password)"
export LOG_PATH="/var/log/myapp/app.log"
```

**ビフォー・アフター:**

| 項目 | 改善前 | 改善後 |
|------|--------|--------|
| **ポータビリティ** | 特定環境でのみ動作 | 環境変数で柔軟に対応 |
| **セキュリティ** | パスワードが平文 | 環境変数やシークレット管理 |
| **エラー処理** | なし | ファイル存在確認・例外処理 |
| **設定管理** | ハードコード | 環境ごとに設定ファイルを切替 |
| **チーム開発** | 個人環境依存 | 誰でも同じスクリプトを使用可能 |

**期待できる効果:**
- 環境変数で柔軟に設定変更可能になり、開発・ステージング・本番で同じスクリプトが使える
- パスワードが平文で保存されず、セキュリティリスクが低減
- ファイルの存在確認とエラーハンドリングにより、実行前に問題を検知
- チームメンバー全員が同じスクリプトを使用でき、属人化を防止

## 想定シナリオ：デプロイ作業を自動化する

例えば、毎週金曜日に行うデプロイ作業が、このような手順だとします：

```
1. GitHubからリポジトリをクローン
2. 依存関係をインストール (npm ci)
3. ユニットテストを実行 (npm test)
4. ビルド (npm run build)
5. ビルド成果物をサーバーに転送 (scp)
6. サーバーでサービスを再起動 (pm2 reload)
```

手作業で行うと、一つのミスで全体が失敗する可能性があります。

**【初期状態】手作業でのデプロイ**

```
17:30 手順書を開く
17:32 リポジトリをクローン
      → タイポして失敗、やり直し
17:35 npm ci を実行
      → ネットワークエラー、再実行
17:40 npm test を実行
      → 1つテストが失敗していたが気づかず次へ
17:42 npm run build を実行
17:50 scp でファイル転送
      → パスを間違えて失敗
17:53 もう一度 scp
17:58 SSH でサーバーにログイン
18:00 pm2 reload 実行
      → さっきのテスト失敗に気づく
18:05 全部やり直し...

所要時間: 約35分（失敗込み）
失敗率: 週に1-2回はミス
ストレス: 高い
```

**【改善後】スクリプトによる自動化**

```bash
$ ./deploy.sh production

[2026-01-30 17:30:15] Starting deployment to production
[2026-01-30 17:30:16] Cloning repository...
[2026-01-30 17:30:22] Installing dependencies...
[2026-01-30 17:31:45] Running tests...
[2026-01-30 17:32:10] Building project...
[2026-01-30 17:33:05] Uploading to server...
[2026-01-30 17:34:15] Reloading service...
[2026-01-30 17:34:20] Deployment completed successfully

所要時間: 約4分
失敗率: エラーがあれば即座に停止
ストレス: 低い
```

**期待できる改善点:**

| 項目 | 手作業 | 自動化後 |
|------|--------|----------|
| **所要時間** | 20-35分 | 4-5分 |
| **エラー検知** | 途中で気づかず進行 | エラーが発生したら即座に停止 |
| **再現性** | 人によって手順が異なる | 誰でも同じ手順で実行可能 |
| **ログ** | なし | タイムスタンプ付きで記録 |
| **金曜の夕方** | ストレスフル | 早く帰れる |

毎週のデプロイ作業が20分から5分になれば、その時間を新機能の開発やコードレビューに使えます。
年間で考えると、約130時間（20分 × 52週）の削減が見込めることになります。

## 本の内容を一部公開：言語の使い分け指針

本書では、このような実践的な内容を全8章にわたって解説していますが、ここでは特に重要な言語の使い分け指針を紹介します。

### Shell、Python、Node.jsの選択基準

```
┌─────────────────────────────────────────────────┐
│ タスクの種類                                      │
└─────────────────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
    ファイル操作    データ処理    Web/API関連
        │              │              │
        ▼              ▼              ▼

┌──────────┐   ┌──────────┐   ┌──────────┐
│  Shell   │   │  Python  │   │ Node.js  │
└──────────┘   └──────────┘   └──────────┘
```

### Shell を選ぶべきケース

**適したタスク:**
- シンプルなファイル操作
- UNIX コマンドの組み合わせ
- パイプラインでのデータ変換
- システム管理スクリプト

**例：ログローテーション**

```bash
#!/bin/bash

set -euo pipefail

LOG_DIR="/var/log/myapp"
ARCHIVE_DIR="${LOG_DIR}/archive"
RETENTION_DAYS=30

# アーカイブディレクトリを作成
mkdir -p "${ARCHIVE_DIR}"

# 1日以上経過したログファイルを圧縮
find "${LOG_DIR}" -name "*.log" -type f -mtime +0 -exec gzip {} \;

# 圧縮ファイルをアーカイブディレクトリに移動
find "${LOG_DIR}" -name "*.log.gz" -type f -exec mv {} "${ARCHIVE_DIR}/" \;

# 古いアーカイブを削除
find "${ARCHIVE_DIR}" -name "*.log.gz" -type f -mtime +${RETENTION_DAYS} -delete

echo "Log rotation completed"
```

### Python を選ぶべきケース

**適したタスク:**
- 複雑なデータ処理
- JSON/XML/CSVの操作
- API連携（REST、GraphQL）
- データ分析
- 機械学習の前処理

**例：CSV集計とレポート生成**

```python
#!/usr/bin/env python3

import csv
import sys
from pathlib import Path
from typing import List, Dict
from collections import defaultdict
from datetime import datetime

def read_csv(file_path: Path) -> List[Dict]:
    """CSVファイルを読み込む"""
    rows = []
    try:
        with file_path.open() as f:
            reader = csv.DictReader(f)
            rows = list(reader)
    except FileNotFoundError:
        print(f"Error: File not found: {file_path}", file=sys.stderr)
        sys.exit(1)
    except csv.Error as e:
        print(f"Error: Invalid CSV: {e}", file=sys.stderr)
        sys.exit(1)

    return rows

def aggregate_sales(rows: List[Dict]) -> Dict:
    """売上データを集計"""
    summary = defaultdict(lambda: {'count': 0, 'total': 0})

    for row in rows:
        category = row.get('category', 'Unknown')
        try:
            amount = float(row.get('amount', 0))
        except ValueError:
            continue

        summary[category]['count'] += 1
        summary[category]['total'] += amount

    return dict(summary)

def generate_report(summary: Dict, output_path: Path) -> None:
    """レポートを生成"""
    with output_path.open('w') as f:
        f.write(f"Sales Report - {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n")
        f.write("=" * 60 + "\n\n")

        total_sales = sum(data['total'] for data in summary.values())
        total_count = sum(data['count'] for data in summary.values())

        f.write(f"Total Sales: ${total_sales:,.2f}\n")
        f.write(f"Total Orders: {total_count:,}\n\n")

        f.write("Category Breakdown:\n")
        f.write("-" * 60 + "\n")

        for category, data in sorted(summary.items(), key=lambda x: x[1]['total'], reverse=True):
            f.write(f"{category:20s} | Orders: {data['count']:5d} | Total: ${data['total']:10,.2f}\n")

def main():
    if len(sys.argv) != 3:
        print(f"Usage: {sys.argv[0]} <input.csv> <output.txt>", file=sys.stderr)
        sys.exit(1)

    input_file = Path(sys.argv[1])
    output_file = Path(sys.argv[2])

    # CSVを読み込み
    rows = read_csv(input_file)
    print(f"Read {len(rows)} rows")

    # 集計
    summary = aggregate_sales(rows)
    print(f"Aggregated {len(summary)} categories")

    # レポート生成
    generate_report(summary, output_file)
    print(f"Report generated: {output_file}")

if __name__ == "__main__":
    main()
```

### Node.js を選ぶべきケース

**適したタスク:**
- フロントエンドビルドスクリプト
- npm パッケージとの連携
- 並列処理が必要なタスク
- Web スクレイピング
- TypeScript プロジェクトとの統合

**例：並列ファイル処理**

```typescript
#!/usr/bin/env node

import { promises as fs } from 'fs';
import { join } from 'path';
import pLimit from 'p-limit';

interface ProcessResult {
  file: string;
  success: boolean;
  error?: string;
}

async function processFile(filePath: string): Promise<ProcessResult> {
  try {
    const content = await fs.readFile(filePath, 'utf-8');

    // 何らかの処理（例：画像最適化、コード変換など）
    const processed = content.toUpperCase(); // 簡単な例

    await fs.writeFile(filePath, processed);

    return { file: filePath, success: true };
  } catch (error) {
    return {
      file: filePath,
      success: false,
      error: error instanceof Error ? error.message : 'Unknown error'
    };
  }
}

async function processDirectory(dirPath: string, concurrency: number = 5): Promise<ProcessResult[]> {
  // ファイル一覧を取得
  const files = await fs.readdir(dirPath);
  const filePaths = files.map(file => join(dirPath, file));

  // 並列処理の制限を設定
  const limit = pLimit(concurrency);

  // 並列処理を実行
  const results = await Promise.all(
    filePaths.map(file => limit(() => processFile(file)))
  );

  return results;
}

async function main() {
  const args = process.argv.slice(2);

  if (args.length === 0) {
    console.error('Usage: process-files <directory>');
    process.exit(1);
  }

  const directory = args[0];
  const concurrency = parseInt(args[1] || '5', 10);

  try {
    const stats = await fs.stat(directory);

    if (!stats.isDirectory()) {
      console.error(`Error: ${directory} is not a directory`);
      process.exit(1);
    }

    console.log(`Processing files in ${directory} (concurrency: ${concurrency})`);

    const startTime = Date.now();
    const results = await processDirectory(directory, concurrency);
    const duration = Date.now() - startTime;

    // 結果をサマリー
    const successful = results.filter(r => r.success).length;
    const failed = results.filter(r => !r.success).length;

    console.log('\nResults:');
    console.log(`  Total: ${results.length}`);
    console.log(`  Successful: ${successful}`);
    console.log(`  Failed: ${failed}`);
    console.log(`  Duration: ${duration}ms`);

    if (failed > 0) {
      console.log('\nFailed files:');
      results.filter(r => !r.success).forEach(r => {
        console.log(`  - ${r.file}: ${r.error}`);
      });
      process.exit(1);
    }
  } catch (error) {
    console.error(`Error: ${error instanceof Error ? error.message : 'Unknown error'}`);
    process.exit(1);
  }
}

main();
```

## 自動化の実践例

### デプロイメントスクリプト

```bash
#!/bin/bash

set -euo pipefail

# 設定
PROJECT_NAME="myapp"
DEPLOY_ENV="${1:-staging}"
VERSION="${2:-latest}"

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*"
}

error() {
    log "ERROR: $*" >&2
    exit 1
}

# 環境変数のチェック
case "${DEPLOY_ENV}" in
    staging)
        SERVER="staging.example.com"
        PORT="2222"
        ;;
    production)
        SERVER="prod.example.com"
        PORT="22"
        ;;
    *)
        error "Unknown environment: ${DEPLOY_ENV}"
        ;;
esac

log "Deploying ${PROJECT_NAME} v${VERSION} to ${DEPLOY_ENV}"

# ビルド
log "Building project..."
npm run build || error "Build failed"

# テスト
log "Running tests..."
npm test || error "Tests failed"

# アーカイブ作成
log "Creating deployment archive..."
tar -czf "${PROJECT_NAME}-${VERSION}.tar.gz" dist/ package.json || error "Failed to create archive"

# サーバーへ転送
log "Uploading to ${SERVER}..."
scp -P "${PORT}" "${PROJECT_NAME}-${VERSION}.tar.gz" "deploy@${SERVER}:/tmp/" || error "Upload failed"

# デプロイ実行
log "Deploying on server..."
ssh -p "${PORT}" "deploy@${SERVER}" << EOF
    set -e
    cd /var/www/${PROJECT_NAME}
    tar -xzf /tmp/${PROJECT_NAME}-${VERSION}.tar.gz
    npm install --production
    pm2 reload ${PROJECT_NAME}
    rm /tmp/${PROJECT_NAME}-${VERSION}.tar.gz
EOF

log "Deployment completed successfully"
```

**期待される効果:**
- 手作業によるミスの削減（タイポ、手順の飛ばしなど）
- デプロイ時間の短縮（手作業20分から自動化5分へ）
- 一貫性のある手順（誰が実行しても同じ結果）
- ロールバックの容易化（バージョン管理による）

## 本書で学べる全内容

この記事で紹介した内容は、本書の一部に過ぎません。全8章で、以下の内容を網羅しています。

### Chapter 1: Shell スクリプト基礎

- Bash の基本構文
- 変数とパラメータ展開
- 条件分岐とループ
- 関数の定義
- エラーハンドリング（set -euo pipefail）
- デバッグ手法

### Chapter 2: Python スクリプト実践

- スクリプト設計のベストプラクティス
- 引数パース（argparse）
- ファイル操作とパス処理
- JSON/YAML/CSV処理
- ログ出力（logging）
- 例外処理

### Chapter 3: Node.js スクリプト開発

- TypeScript スクリプト
- 並列処理（Promise、async/await）
- ストリーム処理
- CLI ツール開発
- npm パッケージとの連携
- エラーハンドリング

### Chapter 4: 自動化・デプロイメント

- CI/CD 統合
- デプロイスクリプト
- ビルド自動化
- テスト自動化
- 環境構築スクリプト

### Chapter 5: メンテナンス・運用スクリプト

- ログローテーション
- バックアップスクリプト
- ヘルスチェック
- アラート通知
- データクリーンアップ

### Chapter 6: CI/CD統合

- GitHub Actions
- GitLab CI
- Jenkins
- スクリプトの再利用性
- シークレット管理

### Chapter 7: ベストプラクティス

- コーディング規約
- ドキュメント化
- テスト手法
- バージョン管理
- セキュリティ考慮事項

### Chapter 8: 実践事例

- 想定されるプロジェクトでの活用例
- トラブルシューティング
- パフォーマンス最適化
- 段階的な改善

## こんな方におすすめ

- **開発者**（手作業を自動化したい方）
- **DevOpsエンジニア**（CI/CDを構築したい方）
- **インフラエンジニア**（運用スクリプトを改善したい方）
- **スクリプト初心者**（基礎から学びたい方）
- **チームリーダー**（チームの生産性を上げたい方）

## 価格

**500円**

一般的な技術書（3,000円から5,000円）の1/6から1/10の価格で、実践的なスクリプト開発ガイドが手に入ります。

## サンプル

導入部分とShellスクリプト基礎の章は無料で読めます。ぜひご覧ください。

https://zenn.dev/gaku/books/script-development-complete-guide-2026

## 本書が必要な5つのサイン

### サイン1: 毎日同じ作業を繰り返している

**こんな状況:**

毎朝10時、ログの確認作業。

```bash
$ ssh server1
$ tail -100 /var/log/app.log | grep ERROR
$ exit
$ ssh server2
$ tail -100 /var/log/app.log | grep ERROR
$ exit
# 10台のサーバーで同じことを繰り返す...
```

30分かかる。毎日。

デプロイのたびに手動でコマンドを実行。
データのエクスポート/インポートを手作業で実施。

**本書の解決策:**
- タスクの自動化パターン
- スケジュール実行の設定
- エラー時の通知設定

**期待できる成果:**
毎日30分のログ確認作業が、スクリプト実行1分に短縮。
月間で約10時間、年間で約120時間の作業時間削減が見込めます。

### サイン2: スクリプトのエラーハンドリングが不十分

**こんな状況:**

深夜のバッチ処理。
途中でエラーが発生していたが、誰も気づかない。

```
02:00 バッチ開始
02:15 データ取得失敗（エラーチェックなし）
02:16 空のデータで処理続行
02:30 レポート生成（空のデータで）
02:31 メール送信「処理完了しました」

翌朝...
「あれ、データが空？」
```

エラーが発生しても気づかない。
途中で失敗しても処理が続行される。
エラーメッセージがわかりづらい。

**本書の解決策:**
- 包括的なエラーハンドリング
- ログ出力のベストプラクティス
- リトライロジックの実装

**期待できる成果:**
問題の早期発見により、障害対応時間が大幅に短縮。
エラー通知により、深夜の問題も翌朝には解決済みの状態に。

### サイン3: 言語の使い分けができていない

**こんな状況:**

すべてShellで書いて可読性が低い。

```bash
# 100行のShellスクリプト
# ネストした条件分岐
# 複雑な正規表現
# 誰も理解できない...
```

シンプルな処理にPythonを使って環境構築が面倒に。
用途に合わない言語で実装している。

**本書の解決策:**
- タスク別の言語選択基準
- それぞれの得意分野の理解
- ハイブリッドな活用方法

**期待できる成果:**
最適な言語選択により、開発効率が向上。
保守性が上がり、引き継ぎやメンテナンスが容易に。

### サイン4: 保守性の低いスクリプトが量産されている

**こんな状況:**

半年前の自分が書いたスクリプト。

```python
def f(a, b, c):
    # コメントなし
    x = a + b
    y = x * c
    return y if x > 10 else 0
```

「これ、何してるんだっけ...？」

コメントがなく意図がわからない。
ハードコードされた値が散在。
誰も触りたくない「魔法のスクリプト」に。

**本書の解決策:**
- 保守性を高める設計パターン
- 設定の外部化
- ドキュメント化の手法

**期待できる成果:**
半年後、1年後でも理解できるコード。
チームメンバーへの引き継ぎが容易に。
機能追加や修正の工数が大幅に削減。

### サイン5: CI/CDに組み込む方法がわからない

**こんな状況:**

ローカルでは動くスクリプト。
でも、CI/CDで実行すると...

```
Error: /Users/username/scripts/deploy.sh: No such file or directory
Error: Permission denied
Error: DB_PASSWORD is not set
```

スクリプトがローカルでしか動かない。
環境依存で他の人が実行できない。
CI/CDでの実行がうまくいかない。

**本書の解決策:**
- CI/CD統合のベストプラクティス
- 環境変数の活用
- コンテナ対応

**期待できる成果:**
ローカル、CI/CD、本番環境で同じスクリプトが動作。
チーム全体での活用が可能に。
継続的なデリバリーが実現。

## 本書で得られる3つの具体的なスキル

### スキル1: 保守性の高いスクリプトを書ける

**習得できること:**
- エラーハンドリングの実装
- ログ出力の設計
- 設定の外部化
- ドキュメント化

**実務での価値:**

「去年書いたスクリプト、今見てもすぐ理解できる」
「新しいメンバーに引き継ぐのが楽になった」
「トラブル時の調査時間が1/3に短縮された」

後から見ても理解できるコード。
トラブル時の調査が容易。
チームでの共有が可能。

### スキル2: 適切な言語を選択できる

**習得できること:**
- Shell、Python、Node.jsの使い分け
- それぞれの得意分野の理解
- ハイブリッドな活用

**実務での価値:**

「このタスク、Shellでやるべきか、Pythonでやるべきか」
迷わず判断できるようになります。

開発効率の向上。
パフォーマンスの最適化。
学習コストの削減。

### スキル3: CI/CDに統合できる

**習得できること:**
- GitHub Actions、GitLab CIでの実行
- 環境変数とシークレット管理
- テスト自動化
- デプロイ自動化

**実務での価値:**

「プルリクエストを作ったら、自動でテストが走る」
「mainにマージしたら、自動でデプロイされる」

継続的な品質向上。
リリースサイクルの高速化。
チーム全体の生産性向上。

## 想定される効果

本書の内容を実践することで、以下のような効果が期待できます：

### 作業効率

**想定されるシナリオ:**
- 繰り返し作業の自動化により、週5時間の作業時間が削減される可能性
- デプロイ作業が20分から5分に短縮され、その時間を開発に使える
- ヒューマンエラーの防止により、トラブル対応時間が減少

### コード品質

**想定されるシナリオ:**
- 保守性の向上により、半年後でもコードの意図が理解できる
- エラーハンドリングの改善により、問題の早期発見が可能に
- ドキュメントの充実により、新しいメンバーへの引き継ぎが容易に

### チーム生産性

**想定されるシナリオ:**
- ナレッジの共有により、「あの人しかわからない」状況が解消
- 一貫した手順により、誰でも同じ作業が実行可能に
- オンボーディングの効率化により、新メンバーの立ち上がりが早く

## さいごに

スクリプト開発は、開発者の生産性を大きく左右する重要なスキルです。

金曜の夕方、早く帰りたいのにデプロイ作業。
月曜の朝、誰が書いたかわからないスクリプトのメンテ。
深夜、エラーで止まっていたバッチ処理...

こんな経験、ありませんか？

この本は、私自身が実務で培ったスクリプト開発のノウハウ、失敗から学んだ教訓、ベストプラクティスを全て詰め込みました。

- **実践的な内容**: すぐに使える実装例が豊富
- **体系的な知識**: 基礎から応用まで段階的に学べる
- **言語の使い分け**: Shell、Python、Node.jsの選択基準
- **CI/CD統合**: 自動化の完全実装

この本が、皆さんの開発をより効率的で、より楽しいものにする一助となれば幸いです。

ご質問やフィードバックがあれば、ぜひコメント欄でお聞かせください。

---

**関連リンク**

- [本書の詳細・購入はこちら](https://zenn.dev/gaku/books/script-development-complete-guide-2026)
