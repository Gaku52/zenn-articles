---
title: "Python スクリプト開発の実践"
---

# Python スクリプト開発の実践

Pythonは、その豊富なライブラリとシンプルな文法により、スクリプト開発に最適な言語です。本章では、実践的なPythonスクリプトの書き方、データ処理、API連携などを学びます。

## 基本構造とベストプラクティス

### スクリプトの基本テンプレート

```python
#!/usr/bin/env python3
"""
データ処理スクリプト

このスクリプトは、CSVファイルを読み込み、データを処理して出力します。

使用例:
    python process_data.py input.csv -o output.csv --verbose
"""

import sys
import logging
from pathlib import Path
from typing import Optional

# ログ設定
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


def main() -> int:
    """メイン処理"""
    try:
        # 処理内容
        logger.info("Processing started")

        # 処理が成功した場合は0を返す
        return 0

    except Exception as e:
        logger.error(f"Error occurred: {e}")
        return 1


if __name__ == "__main__":
    sys.exit(main())
```

### 型ヒントの活用

型ヒントを使用することで、コードの可読性と保守性が向上します。

```python
from typing import List, Dict, Optional, Union, Tuple
from pathlib import Path

def process_file(
    file_path: Path,
    output_dir: Optional[Path] = None,
    verbose: bool = False
) -> Dict[str, Union[int, str]]:
    """
    ファイルを処理する

    Args:
        file_path: 入力ファイルのパス
        output_dir: 出力ディレクトリ（省略可）
        verbose: 詳細出力を有効にする

    Returns:
        処理結果を含む辞書

    Raises:
        FileNotFoundError: ファイルが存在しない場合
        ValueError: 無効なデータ形式の場合
    """
    if not file_path.exists():
        raise FileNotFoundError(f"File not found: {file_path}")

    result: Dict[str, Union[int, str]] = {
        "processed": 0,
        "status": "success"
    }

    return result
```

## argparseによる引数処理

### 基本的な引数処理

```python
import argparse
from pathlib import Path

def create_parser() -> argparse.ArgumentParser:
    """コマンドライン引数パーサーを作成"""
    parser = argparse.ArgumentParser(
        description='データ処理スクリプト',
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog='''
使用例:
  %(prog)s input.csv -o output.csv
  %(prog)s data/*.csv --verbose
  %(prog)s input.csv --filter "age > 30"
        '''
    )

    # 位置引数
    parser.add_argument(
        'input',
        type=Path,
        help='入力ファイルパス'
    )

    # オプション引数
    parser.add_argument(
        '-o', '--output',
        type=Path,
        help='出力ファイルパス'
    )

    parser.add_argument(
        '-v', '--verbose',
        action='store_true',
        help='詳細な出力を有効にする'
    )

    parser.add_argument(
        '--filter',
        type=str,
        help='フィルタ条件'
    )

    parser.add_argument(
        '--format',
        choices=['csv', 'json', 'excel'],
        default='csv',
        help='出力フォーマット'
    )

    return parser


def main() -> int:
    parser = create_parser()
    args = parser.parse_args()

    # バリデーション
    if not args.input.exists():
        parser.error(f"Input file does not exist: {args.input}")

    # ログレベルの設定
    if args.verbose:
        logging.getLogger().setLevel(logging.DEBUG)

    logger.info(f"Processing {args.input}")

    return 0


if __name__ == "__main__":
    sys.exit(main())
```

### サブコマンドの実装

```python
def create_parser_with_subcommands() -> argparse.ArgumentParser:
    """サブコマンド付きパーサーを作成"""
    parser = argparse.ArgumentParser(
        description='データ管理ツール'
    )

    # グローバルオプション
    parser.add_argument(
        '-v', '--verbose',
        action='store_true',
        help='詳細出力'
    )

    # サブコマンド
    subparsers = parser.add_subparsers(
        dest='command',
        required=True,
        help='利用可能なコマンド'
    )

    # import サブコマンド
    import_parser = subparsers.add_parser(
        'import',
        help='データをインポート'
    )
    import_parser.add_argument('file', type=Path)
    import_parser.add_argument('--format', choices=['csv', 'json'])

    # export サブコマンド
    export_parser = subparsers.add_parser(
        'export',
        help='データをエクスポート'
    )
    export_parser.add_argument('output', type=Path)
    export_parser.add_argument('--filter', type=str)

    # analyze サブコマンド
    analyze_parser = subparsers.add_parser(
        'analyze',
        help='データを分析'
    )
    analyze_parser.add_argument('--metric', choices=['summary', 'detail'])

    return parser


def cmd_import(args: argparse.Namespace) -> int:
    """importコマンドの実装"""
    logger.info(f"Importing from {args.file}")
    # インポート処理
    return 0


def cmd_export(args: argparse.Namespace) -> int:
    """exportコマンドの実装"""
    logger.info(f"Exporting to {args.output}")
    # エクスポート処理
    return 0


def cmd_analyze(args: argparse.Namespace) -> int:
    """analyzeコマンドの実装"""
    logger.info(f"Analyzing with metric: {args.metric}")
    # 分析処理
    return 0


def main() -> int:
    parser = create_parser_with_subcommands()
    args = parser.parse_args()

    # サブコマンドのディスパッチ
    commands = {
        'import': cmd_import,
        'export': cmd_export,
        'analyze': cmd_analyze
    }

    return commands[args.command](args)
```

## データ処理

### CSV/Excelファイルの処理

```python
import pandas as pd
from pathlib import Path
from typing import List, Optional

def process_csv(
    input_path: Path,
    output_path: Path,
    columns: Optional[List[str]] = None
) -> None:
    """
    CSVファイルを処理する

    Args:
        input_path: 入力CSVファイル
        output_path: 出力CSVファイル
        columns: 処理対象のカラム
    """
    logger.info(f"Reading {input_path}")

    # CSV読み込み
    df = pd.read_csv(input_path)

    logger.info(f"Loaded {len(df)} rows")

    # カラムのフィルタリング
    if columns:
        df = df[columns]

    # データ処理
    # 例: 数値カラムの合計を計算
    if 'price' in df.columns and 'quantity' in df.columns:
        df['total'] = df['price'] * df['quantity']

    # 欠損値の処理
    df = df.dropna()

    # フィルタリング
    # 例: total > 1000 の行のみ
    if 'total' in df.columns:
        df = df[df['total'] > 1000]

    # 出力
    df.to_csv(output_path, index=False)
    logger.info(f"Wrote {len(df)} rows to {output_path}")


def process_excel(
    input_path: Path,
    sheet_name: str = 'Sheet1'
) -> pd.DataFrame:
    """
    Excelファイルを処理する

    Args:
        input_path: 入力Excelファイル
        sheet_name: シート名

    Returns:
        処理後のDataFrame
    """
    # Excelファイル読み込み
    df = pd.read_excel(input_path, sheet_name=sheet_name)

    # データ処理
    # 日付カラムの変換
    if 'date' in df.columns:
        df['date'] = pd.to_datetime(df['date'])

    # カテゴリカルデータの処理
    if 'category' in df.columns:
        df['category'] = df['category'].astype('category')

    return df
```

### JSONファイルの処理

```python
import json
from pathlib import Path
from typing import Any, Dict, List

def read_json(file_path: Path) -> Dict[str, Any]:
    """JSONファイルを読み込む"""
    with open(file_path, 'r', encoding='utf-8') as f:
        return json.load(f)


def write_json(
    file_path: Path,
    data: Dict[str, Any],
    pretty: bool = True
) -> None:
    """JSONファイルに書き込む"""
    with open(file_path, 'w', encoding='utf-8') as f:
        if pretty:
            json.dump(data, f, indent=2, ensure_ascii=False)
        else:
            json.dump(data, f, ensure_ascii=False)


def process_json_batch(
    input_dir: Path,
    pattern: str = "*.json"
) -> List[Dict[str, Any]]:
    """
    複数のJSONファイルをバッチ処理

    Args:
        input_dir: 入力ディレクトリ
        pattern: ファイルパターン

    Returns:
        処理結果のリスト
    """
    results = []

    for json_file in input_dir.glob(pattern):
        try:
            data = read_json(json_file)
            # データ処理
            processed = {
                'file': json_file.name,
                'record_count': len(data) if isinstance(data, list) else 1,
                'data': data
            }
            results.append(processed)

        except json.JSONDecodeError as e:
            logger.error(f"Invalid JSON in {json_file}: {e}")
        except Exception as e:
            logger.error(f"Error processing {json_file}: {e}")

    return results
```

## API連携

### RESTful API クライアント

```python
import requests
from typing import Dict, Any, Optional
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

class APIClient:
    """RESTful APIクライアント"""

    def __init__(
        self,
        base_url: str,
        api_key: Optional[str] = None,
        timeout: int = 30
    ):
        self.base_url = base_url.rstrip('/')
        self.timeout = timeout
        self.session = self._create_session()

        if api_key:
            self.session.headers.update({
                'Authorization': f'Bearer {api_key}'
            })

    def _create_session(self) -> requests.Session:
        """リトライ機能付きセッションを作成"""
        session = requests.Session()

        retry_strategy = Retry(
            total=3,
            backoff_factor=1,
            status_forcelist=[429, 500, 502, 503, 504]
        )

        adapter = HTTPAdapter(max_retries=retry_strategy)
        session.mount("http://", adapter)
        session.mount("https://", adapter)

        return session

    def get(
        self,
        endpoint: str,
        params: Optional[Dict[str, Any]] = None
    ) -> Dict[str, Any]:
        """GETリクエスト"""
        url = f"{self.base_url}/{endpoint.lstrip('/')}"

        response = self.session.get(
            url,
            params=params,
            timeout=self.timeout
        )
        response.raise_for_status()

        return response.json()

    def post(
        self,
        endpoint: str,
        data: Dict[str, Any]
    ) -> Dict[str, Any]:
        """POSTリクエスト"""
        url = f"{self.base_url}/{endpoint.lstrip('/')}"

        response = self.session.post(
            url,
            json=data,
            timeout=self.timeout
        )
        response.raise_for_status()

        return response.json()


# 使用例
def fetch_users_example():
    """ユーザー情報を取得する例"""
    client = APIClient(
        base_url="https://api.example.com",
        api_key="your_api_key"
    )

    try:
        users = client.get('/users', params={'limit': 100})
        logger.info(f"Fetched {len(users)} users")

        return users

    except requests.exceptions.RequestException as e:
        logger.error(f"API request failed: {e}")
        raise
```

## データベース操作

### PostgreSQL操作

```python
import psycopg2
from psycopg2.extras import RealDictCursor
from typing import List, Dict, Any
from contextlib import contextmanager

class DatabaseClient:
    """PostgreSQLクライアント"""

    def __init__(self, connection_string: str):
        self.connection_string = connection_string

    @contextmanager
    def get_connection(self):
        """コネクションのコンテキストマネージャー"""
        conn = psycopg2.connect(self.connection_string)
        try:
            yield conn
            conn.commit()
        except Exception as e:
            conn.rollback()
            logger.error(f"Database error: {e}")
            raise
        finally:
            conn.close()

    def execute_query(
        self,
        query: str,
        params: tuple = ()
    ) -> List[Dict[str, Any]]:
        """クエリを実行して結果を取得"""
        with self.get_connection() as conn:
            with conn.cursor(cursor_factory=RealDictCursor) as cur:
                cur.execute(query, params)
                return [dict(row) for row in cur.fetchall()]

    def execute_update(
        self,
        query: str,
        params: tuple = ()
    ) -> int:
        """更新クエリを実行"""
        with self.get_connection() as conn:
            with conn.cursor() as cur:
                cur.execute(query, params)
                return cur.rowcount


# 使用例
def migrate_data_example():
    """データ移行の例"""
    db = DatabaseClient("postgresql://localhost/mydb")

    # データの取得
    old_users = db.execute_query(
        "SELECT id, name, email FROM old_users WHERE migrated = FALSE"
    )

    logger.info(f"Found {len(old_users)} users to migrate")

    # データの挿入
    for user in old_users:
        db.execute_update(
            """
            INSERT INTO new_users (id, name, email)
            VALUES (%s, %s, %s)
            ON CONFLICT (id) DO NOTHING
            """,
            (user['id'], user['name'], user['email'])
        )

    # フラグの更新
    updated = db.execute_update(
        "UPDATE old_users SET migrated = TRUE WHERE id = ANY(%s)",
        ([user['id'] for user in old_users],)
    )

    logger.info(f"Migrated {updated} users")
```

## 並行処理

### マルチスレッド処理

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
from typing import List, Callable, Any

def parallel_process(
    items: List[Any],
    process_func: Callable,
    max_workers: int = 4
) -> List[Any]:
    """
    アイテムを並列処理する

    Args:
        items: 処理対象のアイテムリスト
        process_func: 処理関数
        max_workers: 最大ワーカー数

    Returns:
        処理結果のリスト
    """
    results = []

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        # タスクを投入
        future_to_item = {
            executor.submit(process_func, item): item
            for item in items
        }

        # 完了したタスクから結果を取得
        for future in as_completed(future_to_item):
            item = future_to_item[future]
            try:
                result = future.result()
                results.append(result)
            except Exception as e:
                logger.error(f"Error processing {item}: {e}")

    return results


# 使用例
def download_files_parallel(urls: List[str]) -> None:
    """複数のファイルを並列ダウンロード"""
    def download(url: str) -> str:
        response = requests.get(url)
        filename = url.split('/')[-1]

        with open(filename, 'wb') as f:
            f.write(response.content)

        return filename

    downloaded = parallel_process(urls, download, max_workers=8)
    logger.info(f"Downloaded {len(downloaded)} files")
```

## エラーハンドリングとロギング

### カスタム例外

```python
class ScriptError(Exception):
    """スクリプトエラーの基底クラス"""
    pass

class ValidationError(ScriptError):
    """検証エラー"""
    pass

class ProcessingError(ScriptError):
    """処理エラー"""
    pass


def validate_data(data: Dict[str, Any]) -> None:
    """
    データを検証する

    Raises:
        ValidationError: データが無効な場合
    """
    required_fields = ['id', 'name', 'email']

    for field in required_fields:
        if field not in data:
            raise ValidationError(f"Missing required field: {field}")

    # メールアドレスの検証
    if '@' not in data['email']:
        raise ValidationError(f"Invalid email: {data['email']}")
```

### 構造化ログ

```python
import logging
from pythonjsonlogger import jsonlogger

def setup_logging(level: str = "INFO") -> None:
    """構造化ログを設定"""
    logger = logging.getLogger()
    logger.setLevel(level)

    handler = logging.StreamHandler()

    formatter = jsonlogger.JsonFormatter(
        '%(asctime)s %(name)s %(levelname)s %(message)s'
    )
    handler.setFormatter(formatter)

    logger.addHandler(handler)


# 使用例
logger.info("Processing started", extra={
    'file': 'input.csv',
    'rows': 1000
})
```

## 実践例: データ処理パイプライン

```python
#!/usr/bin/env python3
"""
データ処理パイプライン

CSVファイルを読み込み、処理し、結果を出力する
"""

import sys
import argparse
import logging
from pathlib import Path
from typing import Optional
import pandas as pd

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


class DataPipeline:
    """データ処理パイプライン"""

    def __init__(self, input_path: Path, output_path: Path):
        self.input_path = input_path
        self.output_path = output_path
        self.df: Optional[pd.DataFrame] = None

    def load(self) -> 'DataPipeline':
        """データを読み込む"""
        logger.info(f"Loading data from {self.input_path}")
        self.df = pd.read_csv(self.input_path)
        logger.info(f"Loaded {len(self.df)} rows")
        return self

    def clean(self) -> 'DataPipeline':
        """データをクリーニング"""
        logger.info("Cleaning data")

        # 欠損値の削除
        initial_rows = len(self.df)
        self.df = self.df.dropna()
        removed_rows = initial_rows - len(self.df)

        if removed_rows > 0:
            logger.info(f"Removed {removed_rows} rows with missing values")

        return self

    def transform(self) -> 'DataPipeline':
        """データを変換"""
        logger.info("Transforming data")

        # 計算カラムの追加
        if 'price' in self.df.columns and 'quantity' in self.df.columns:
            self.df['total'] = self.df['price'] * self.df['quantity']

        return self

    def filter(self, condition: str) -> 'DataPipeline':
        """データをフィルタリング"""
        logger.info(f"Filtering data: {condition}")

        initial_rows = len(self.df)
        self.df = self.df.query(condition)
        filtered_rows = initial_rows - len(self.df)

        logger.info(f"Filtered out {filtered_rows} rows")

        return self

    def save(self) -> 'DataPipeline':
        """データを保存"""
        logger.info(f"Saving data to {self.output_path}")

        self.output_path.parent.mkdir(parents=True, exist_ok=True)
        self.df.to_csv(self.output_path, index=False)

        logger.info(f"Saved {len(self.df)} rows")

        return self


def main() -> int:
    parser = argparse.ArgumentParser(description='データ処理パイプライン')
    parser.add_argument('input', type=Path, help='入力CSVファイル')
    parser.add_argument('-o', '--output', type=Path, required=True, help='出力CSVファイル')
    parser.add_argument('--filter', type=str, help='フィルタ条件')
    parser.add_argument('-v', '--verbose', action='store_true', help='詳細出力')

    args = parser.parse_args()

    if args.verbose:
        logging.getLogger().setLevel(logging.DEBUG)

    try:
        # パイプライン実行
        pipeline = DataPipeline(args.input, args.output)
        pipeline.load().clean().transform()

        if args.filter:
            pipeline.filter(args.filter)

        pipeline.save()

        logger.info("Pipeline completed successfully")
        return 0

    except Exception as e:
        logger.error(f"Pipeline failed: {e}")
        return 1


if __name__ == "__main__":
    sys.exit(main())
```

## まとめ

本章では、Pythonを使った実践的なスクリプト開発を学びました：

- **基本構造**: 型ヒント、ログ設定、エラーハンドリング
- **引数処理**: argparse、サブコマンド
- **データ処理**: CSV/Excel/JSON、pandas
- **API連携**: requestsライブラリ、リトライ戦略
- **データベース**: PostgreSQL操作、トランザクション管理
- **並行処理**: ThreadPoolExecutor

次章では、Node.jsとTypeScriptを使った現代的なスクリプト開発について学びます。
