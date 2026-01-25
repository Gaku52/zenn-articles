# Pythonスクリプト自動化

本章では、Pythonを使った自動化スクリプトの開発を学びます。Pythonは、読みやすい構文と豊富なライブラリにより、データ処理、API連携、システム管理など幅広い自動化タスクに適しています。

## Python自動化の基礎

### 基本スクリプト構成

```python
#!/usr/bin/env python3

import sys
import subprocess
from pathlib import Path

def main():
    # 引数解析
    env = sys.argv[1] if len(sys.argv) > 1 else 'development'

    print(f'Deploying to {env}...')

    # ビルド実行
    try:
        subprocess.run(['npm', 'run', 'build'], check=True)
        print('Build completed successfully')
    except subprocess.CalledProcessError as e:
        print(f'Build failed: {e}', file=sys.stderr)
        sys.exit(1)

if __name__ == '__main__':
    main()
```

### 型ヒントによる型安全性

```python
#!/usr/bin/env python3

from typing import Dict, List, Optional
from dataclasses import dataclass
import subprocess

@dataclass
class DeployConfig:
    """デプロイ設定"""
    environment: str
    branch: str
    build_command: str
    deploy_path: str

# 環境別設定
CONFIGS: Dict[str, DeployConfig] = {
    'development': DeployConfig(
        environment='development',
        branch='develop',
        build_command='npm run build:dev',
        deploy_path='/var/www/dev',
    ),
    'production': DeployConfig(
        environment='production',
        branch='main',
        build_command='npm run build:prod',
        deploy_path='/var/www/prod',
    ),
}

def deploy(config: DeployConfig) -> None:
    """デプロイ実行"""
    print(f'Deploying to {config.environment}')

    # ビルド
    subprocess.run(config.build_command.split(), check=True)

    # デプロイ
    subprocess.run([
        'rsync', '-avz', 'dist/',
        f'{config.deploy_path}/'
    ], check=True)

    print('Deployment completed')

def main() -> None:
    import sys

    env = sys.argv[1] if len(sys.argv) > 1 else 'development'
    config = CONFIGS.get(env)

    if not config:
        print(f'Unknown environment: {env}', file=sys.stderr)
        sys.exit(1)

    deploy(config)

if __name__ == '__main__':
    main()
```

## pathlibによるファイル操作

### ファイル・ディレクトリ操作

```python
from pathlib import Path
from typing import List

def get_files_recursive(directory: Path, pattern: str = '*') -> List[Path]:
    """ディレクトリ内のファイルを再帰的に取得"""
    files = []
    for path in directory.rglob(pattern):
        if path.is_file():
            files.append(path)
    return files

def process_text_files(dir_path: Path) -> None:
    """テキストファイル一括処理"""
    files = get_files_recursive(dir_path, '*.txt')

    for file_path in files:
        print(f'Processing: {file_path}')

        # ファイル読み込み
        content = file_path.read_text(encoding='utf-8')

        # 処理（例: 大文字変換）
        processed = content.upper()

        # ファイル書き込み
        file_path.write_text(processed, encoding='utf-8')

    print(f'Processed {len(files)} files')

# 使用例
data_dir = Path('./data')
process_text_files(data_dir)
```

### ディレクトリコピー

```python
from pathlib import Path
import shutil
from typing import Callable, Optional

def copy_directory(
    src: Path,
    dest: Path,
    *,
    overwrite: bool = False,
    filter_fn: Optional[Callable[[Path], bool]] = None
) -> None:
    """ディレクトリコピー"""
    # コピー先ディレクトリ作成
    dest.mkdir(parents=True, exist_ok=True)

    for item in src.iterdir():
        # フィルター適用
        if filter_fn and not filter_fn(item):
            continue

        dest_item = dest / item.name

        if item.is_dir():
            copy_directory(item, dest_item, overwrite=overwrite, filter_fn=filter_fn)
        else:
            # ファイル存在チェック
            if dest_item.exists() and not overwrite:
                print(f'Skipping existing file: {dest_item}')
                continue

            shutil.copy2(item, dest_item)
            print(f'Copied: {item} -> {dest_item}')

# 使用例
def exclude_node_modules(path: Path) -> bool:
    """node_modulesを除外"""
    return 'node_modules' not in path.parts

copy_directory(
    Path('./src'),
    Path('./backup'),
    overwrite=False,
    filter_fn=exclude_node_modules
)
```

## subprocessによるコマンド実行

### 基本的なコマンド実行

```python
import subprocess
from typing import Tuple

def run_command(
    command: List[str],
    *,
    capture_output: bool = False,
    check: bool = True
) -> Tuple[int, str, str]:
    """コマンド実行"""
    try:
        if capture_output:
            result = subprocess.run(
                command,
                capture_output=True,
                text=True,
                check=check
            )
            return result.returncode, result.stdout, result.stderr
        else:
            result = subprocess.run(command, check=check)
            return result.returncode, '', ''
    except subprocess.CalledProcessError as e:
        print(f'Command failed: {e}', file=sys.stderr)
        raise

# 使用例
returncode, stdout, stderr = run_command(
    ['ls', '-la'],
    capture_output=True
)
print(stdout)
```

### パイプライン処理

```python
import subprocess
from pathlib import Path

def run_pipeline() -> None:
    """コマンドパイプライン実行"""
    # 最初のコマンド
    grep_process = subprocess.Popen(
        ['grep', '-r', 'TODO', './src'],
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        text=True
    )

    # 2番目のコマンド
    wc_process = subprocess.Popen(
        ['wc', '-l'],
        stdin=grep_process.stdout,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        text=True
    )

    # パイプをクローズ
    if grep_process.stdout:
        grep_process.stdout.close()

    # 結果取得
    stdout, stderr = wc_process.communicate()

    if stderr:
        print(f'Errors: {stderr}', file=sys.stderr)

    # ファイル出力
    Path('todo-count.txt').write_text(stdout, encoding='utf-8')
    print('Pipeline completed')

run_pipeline()
```

## argparseによる引数解析

### 基本的な引数解析

```python
import argparse
from pathlib import Path

def parse_args() -> argparse.Namespace:
    """コマンドライン引数解析"""
    parser = argparse.ArgumentParser(
        description='Deployment script',
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Examples:
  %(prog)s -e production -v 1.0.0
  %(prog)s --env staging --dry-run
        """,
    )

    parser.add_argument(
        '-e', '--env',
        choices=['development', 'staging', 'production'],
        default='development',
        help='Target environment (default: development)',
    )

    parser.add_argument(
        '-v', '--version',
        required=True,
        help='Version to deploy',
    )

    parser.add_argument(
        '-d', '--dry-run',
        action='store_true',
        help='Dry run mode',
    )

    parser.add_argument(
        '--verbose',
        action='store_true',
        help='Verbose output',
    )

    parser.add_argument(
        '--config',
        type=Path,
        default=Path('config.yaml'),
        help='Config file path',
    )

    return parser.parse_args()

# 使用例
def main() -> None:
    args = parse_args()

    print(f'Environment: {args.env}')
    print(f'Version: {args.version}')
    print(f'Dry run: {args.dry_run}')
    print(f'Config: {args.config}')

if __name__ == '__main__':
    main()
```

## loggingによるログ出力

### ログ設定

```python
import logging
from pathlib import Path
from datetime import datetime

def setup_logger(name: str, log_dir: Path) -> logging.Logger:
    """ロガー設定"""
    # ログディレクトリ作成
    log_dir.mkdir(parents=True, exist_ok=True)

    # ログファイル名
    log_file = log_dir / f'{name}_{datetime.now():%Y%m%d}.log'

    # ロガー作成
    logger = logging.getLogger(name)
    logger.setLevel(logging.DEBUG)

    # コンソールハンドラー
    console_handler = logging.StreamHandler()
    console_handler.setLevel(logging.INFO)
    console_formatter = logging.Formatter(
        '%(levelname)s: %(message)s'
    )
    console_handler.setFormatter(console_formatter)

    # ファイルハンドラー
    file_handler = logging.FileHandler(log_file, encoding='utf-8')
    file_handler.setLevel(logging.DEBUG)
    file_formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        datefmt='%Y-%m-%d %H:%M:%S',
    )
    file_handler.setFormatter(file_formatter)

    # ハンドラー追加
    logger.addHandler(console_handler)
    logger.addHandler(file_handler)

    return logger

# 使用例
logger = setup_logger('deploy', Path('/var/log/myapp'))

logger.debug('Debug message')
logger.info('Starting deployment')
logger.warning('Disk usage is high')
logger.error('Deployment failed')
```

### 構造化ログ

```python
import logging
import json
from datetime import datetime
from typing import Any, Dict

class JsonFormatter(logging.Formatter):
    """JSON形式ログフォーマッター"""

    def format(self, record: logging.LogRecord) -> str:
        log_data: Dict[str, Any] = {
            'timestamp': datetime.utcnow().isoformat(),
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
        }

        # 追加フィールド
        if hasattr(record, 'extra_data'):
            log_data.update(record.extra_data)

        # 例外情報
        if record.exc_info:
            log_data['exception'] = self.formatException(record.exc_info)

        return json.dumps(log_data, ensure_ascii=False)

# 使用例
logger = logging.getLogger('app')
handler = logging.StreamHandler()
handler.setFormatter(JsonFormatter())
logger.addHandler(handler)
logger.setLevel(logging.INFO)

# 追加データ付きログ
extra = {'user_id': 123, 'action': 'deploy'}
logger.info('Deployment started', extra={'extra_data': extra})
```

## 実用例

### データ処理スクリプト

```python
#!/usr/bin/env python3

import argparse
import logging
from pathlib import Path
from typing import List, Dict, Any
import csv
import json

def setup_logging(verbose: bool) -> logging.Logger:
    """ログ設定"""
    logger = logging.getLogger(__name__)
    handler = logging.StreamHandler()
    formatter = logging.Formatter('%(levelname)s: %(message)s')
    handler.setFormatter(formatter)
    logger.addHandler(handler)
    logger.setLevel(logging.DEBUG if verbose else logging.INFO)
    return logger

def read_csv(file_path: Path) -> List[Dict[str, Any]]:
    """CSV読み込み"""
    data = []
    with file_path.open('r', encoding='utf-8') as f:
        reader = csv.DictReader(f)
        for row in reader:
            data.append(row)
    return data

def process_data(data: List[Dict[str, Any]]) -> List[Dict[str, Any]]:
    """データ処理"""
    processed = []

    for row in data:
        # 数値変換
        price = float(row.get('price', 0))
        quantity = int(row.get('quantity', 0))

        # 計算
        total = price * quantity

        # 条件フィルター
        if total > 1000:
            processed.append({
                'name': row.get('name'),
                'price': price,
                'quantity': quantity,
                'total': total,
            })

    return processed

def write_json(data: List[Dict[str, Any]], file_path: Path) -> None:
    """JSON出力"""
    with file_path.open('w', encoding='utf-8') as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

def main() -> None:
    parser = argparse.ArgumentParser(description='Data processing script')
    parser.add_argument('input', type=Path, help='Input CSV file')
    parser.add_argument('output', type=Path, help='Output JSON file')
    parser.add_argument('-v', '--verbose', action='store_true', help='Verbose output')
    args = parser.parse_args()

    logger = setup_logging(args.verbose)

    try:
        logger.info(f'Reading data from {args.input}')
        data = read_csv(args.input)
        logger.debug(f'Read {len(data)} rows')

        logger.info('Processing data')
        processed = process_data(data)
        logger.debug(f'Processed {len(processed)} rows')

        logger.info(f'Writing output to {args.output}')
        write_json(processed, args.output)

        logger.info('Processing completed successfully')
    except Exception as e:
        logger.error(f'Processing failed: {e}')
        raise

if __name__ == '__main__':
    main()
```

### API連携スクリプト

```python
#!/usr/bin/env python3

import argparse
import logging
from pathlib import Path
from typing import List, Dict, Any, Optional
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry
import json
import time

class ApiClient:
    """API クライアント"""

    def __init__(self, base_url: str, api_key: str, timeout: int = 10):
        self.base_url = base_url
        self.api_key = api_key
        self.timeout = timeout
        self.session = self._create_session()

    def _create_session(self) -> requests.Session:
        """セッション作成（リトライ設定）"""
        session = requests.Session()

        retry = Retry(
            total=3,
            backoff_factor=1,
            status_forcelist=[429, 500, 502, 503, 504],
        )

        adapter = HTTPAdapter(max_retries=retry)
        session.mount('http://', adapter)
        session.mount('https://', adapter)

        return session

    def request(
        self,
        method: str,
        endpoint: str,
        **kwargs
    ) -> requests.Response:
        """APIリクエスト"""
        url = f'{self.base_url}{endpoint}'
        headers = {
            'Authorization': f'Bearer {self.api_key}',
            'Content-Type': 'application/json',
        }

        response = self.session.request(
            method,
            url,
            headers=headers,
            timeout=self.timeout,
            **kwargs
        )

        response.raise_for_status()
        return response

    def get(self, endpoint: str, **kwargs) -> Dict[str, Any]:
        """GET リクエスト"""
        response = self.request('GET', endpoint, **kwargs)
        return response.json()

    def post(self, endpoint: str, data: Dict[str, Any], **kwargs) -> Dict[str, Any]:
        """POST リクエスト"""
        response = self.request('POST', endpoint, json=data, **kwargs)
        return response.json()

def sync_data(
    client: ApiClient,
    output_file: Path,
    logger: logging.Logger
) -> None:
    """データ同期"""
    try:
        logger.info('Fetching data from API')
        data = client.get('/api/items')

        if 'items' not in data:
            raise ValueError('Invalid API response')

        items = data['items']
        logger.info(f'Fetched {len(items)} items')

        # ローカルファイルに保存
        with output_file.open('w', encoding='utf-8') as f:
            json.dump(items, f, ensure_ascii=False, indent=2)

        logger.info(f'Data saved to {output_file}')
    except requests.RequestException as e:
        logger.error(f'API request failed: {e}')
        raise
    except Exception as e:
        logger.error(f'Sync failed: {e}')
        raise

def main() -> None:
    parser = argparse.ArgumentParser(description='API sync script')
    parser.add_argument('--output', type=Path, default=Path('data.json'))
    parser.add_argument('--api-key', required=True, help='API key')
    parser.add_argument('-v', '--verbose', action='store_true')
    args = parser.parse_args()

    logging.basicConfig(
        level=logging.DEBUG if args.verbose else logging.INFO,
        format='%(levelname)s: %(message)s',
    )
    logger = logging.getLogger(__name__)

    client = ApiClient(
        base_url='https://api.example.com',
        api_key=args.api_key,
    )

    sync_data(client, args.output, logger)

if __name__ == '__main__':
    main()
```

### ファイル監視・自動処理

```python
#!/usr/bin/env python3

import time
import logging
from pathlib import Path
from typing import Set
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler, FileSystemEvent

class FileProcessor(FileSystemEventHandler):
    """ファイル処理ハンドラー"""

    def __init__(self, logger: logging.Logger):
        self.logger = logger
        self.processed: Set[Path] = set()

    def on_created(self, event: FileSystemEvent) -> None:
        """ファイル作成時の処理"""
        if event.is_directory:
            return

        file_path = Path(event.src_path)

        # 対象ファイルのみ処理
        if file_path.suffix != '.txt':
            return

        # 重複処理防止
        if file_path in self.processed:
            return

        self.logger.info(f'Processing new file: {file_path}')
        self.process_file(file_path)
        self.processed.add(file_path)

    def process_file(self, file_path: Path) -> None:
        """ファイル処理"""
        try:
            # ファイル読み込み
            content = file_path.read_text(encoding='utf-8')

            # 処理
            processed = content.upper()

            # 出力
            output_path = file_path.with_suffix('.processed.txt')
            output_path.write_text(processed, encoding='utf-8')

            self.logger.info(f'Processed: {output_path}')
        except Exception as e:
            self.logger.error(f'Failed to process {file_path}: {e}')

def main() -> None:
    import argparse

    parser = argparse.ArgumentParser(description='File watcher')
    parser.add_argument('directory', type=Path, help='Directory to watch')
    args = parser.parse_args()

    logging.basicConfig(
        level=logging.INFO,
        format='%(asctime)s - %(levelname)s - %(message)s',
    )
    logger = logging.getLogger(__name__)

    if not args.directory.is_dir():
        logger.error(f'Not a directory: {args.directory}')
        return

    event_handler = FileProcessor(logger)
    observer = Observer()
    observer.schedule(event_handler, str(args.directory), recursive=False)
    observer.start()

    logger.info(f'Watching directory: {args.directory}')

    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()
        logger.info('Stopping file watcher')

    observer.join()

if __name__ == '__main__':
    main()
```

## Pythonスクリプトチェックリスト

### 基本
- [ ] 型ヒントを使用している
- [ ] シバン（#!/usr/bin/env python3）を記述している
- [ ] if __name__ == '__main__'で実行部分を保護している

### ファイル操作
- [ ] pathlibを使用している
- [ ] with文でファイルを開いている
- [ ] エンコーディングを明示している

### コマンド実行
- [ ] subprocess.runを使用している
- [ ] check=Trueでエラーチェックしている
- [ ] 出力のキャプチャを適切に行っている

### 引数解析
- [ ] argparseを使用している
- [ ] ヘルプメッセージを記述している
- [ ] デフォルト値を設定している

### ログ出力
- [ ] loggingモジュールを使用している
- [ ] 適切なログレベルを設定している
- [ ] ログファイルに出力している

### エラーハンドリング
- [ ] try-exceptで例外を捕捉している
- [ ] 適切なエラーメッセージを出力している
- [ ] 終了コードを設定している

## まとめ

本章では、Pythonによるスクリプト自動化を学びました。

**重要ポイント**:
- pathlibによる直感的なファイル操作
- subprocessによる外部コマンド実行
- argparseによる柔軟な引数解析
- loggingによる体系的なログ管理
- 実用的なデータ処理・API連携・ファイル監視の実装例

Pythonの豊富なライブラリと読みやすい構文を活用し、保守性の高い自動化スクリプトを開発してください。

## 参考文献

- [Python Documentation](https://docs.python.org/3/)
- [The Python Standard Library](https://docs.python.org/3/library/)
- [Real Python Tutorials](https://realpython.com/)
- [pathlib - Object-oriented filesystem paths](https://docs.python.org/3/library/pathlib.html)
