# Clickフレームワーク

Clickは、Pythonで最も人気のあるCLIフレームワークの一つです。[Flask](https://flask.palletsprojects.com/)の開発者としても知られるArmin Ronacherによって開発され、Ansible、pipx、AWS SAM CLIなど多くの著名プロジェクトで採用されています。

本章では、Clickを使った引数パース、コマンド定義、オプション設定の基本から、高度な機能まで詳しく学びます。

## Clickの特徴

### デコレータベースの直感的なAPI

Clickは、Pythonのデコレータを活用した直感的なAPIを提供します。関数にデコレータを追加するだけで、簡単にCLIコマンドを作成できます。

### 自動ヘルプ生成

コマンドやオプションの説明を docstring やヘルプテキストとして記述すると、自動的に`--help`オプションが生成されます。

### 型変換とバリデーション

オプションや引数の型を指定することで、自動的に値の変換とバリデーションが行われます。

### ネストされたコマンド

Git のような複雑なコマンド階層（`git commit`、`git push` など）を簡単に実装できます。

## インストール

```bash
pip install click
```

Click は[PyPI](https://pypi.org/project/click/)で公開されており、依存関係が少なく軽量です。

## 基本的な使い方

### シンプルなCLI

```python
import click

@click.command()
@click.argument('name')
@click.option('--template', '-t', default='default', help='Template to use')
@click.option('--verbose', '-v', is_flag=True, help='Verbose output')
def create(name, template, verbose):
    """Create a new project"""
    click.echo(f'Creating project: {name}')
    click.echo(f'Template: {template}')
    if verbose:
        click.echo('Verbose mode enabled')

if __name__ == '__main__':
    create()
```

## コマンドグループ

```python
import click

@click.group()
def cli():
    """My CLI Tool"""
    pass

@cli.command()
@click.argument('name')
@click.option('--template', '-t', default='default')
def create(name, template):
    """Create a new project"""
    click.echo(f'Creating: {name} with {template}')

@cli.command()
@click.option('--all', '-a', is_flag=True)
def list(all):
    """List projects"""
    click.echo('Listing projects...')

if __name__ == '__main__':
    cli()
```

## オプションの種類

### 真偽値フラグ

```python
@click.command()
@click.option('--verbose', '-v', is_flag=True)
@click.option('--typescript/--no-typescript', default=True)
def create(verbose, typescript):
    click.echo(f'Verbose: {verbose}')
    click.echo(f'TypeScript: {typescript}')
```

### 複数値

```python
@click.command()
@click.option('--feature', '-f', multiple=True)
def create(feature):
    click.echo(f'Features: {", ".join(feature)}')

# 使用例: mycli create --feature eslint --feature prettier
```

### 選択肢

```python
@click.command()
@click.option('--env', type=click.Choice(['dev', 'staging', 'prod']))
def deploy(env):
    click.echo(f'Deploying to: {env}')
```

## バリデーション

```python
def validate_email(ctx, param, value):
    import re
    if not re.match(r'^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$', value):
        raise click.BadParameter('Invalid email address')
    return value

@click.command()
@click.option('--email', callback=validate_email)
def register(email):
    click.echo(f'Email: {email}')
```

## プロンプト

```python
@click.command()
@click.option('--name', prompt='Project name')
@click.option('--password', prompt=True, hide_input=True, confirmation_prompt=True)
def create(name, password):
    click.echo(f'Creating project: {name}')
```

## ファイル操作

```python
@click.command()
@click.argument('input', type=click.File('r'))
@click.argument('output', type=click.File('w'))
def convert(input, output):
    content = input.read()
    output.write(content.upper())
```

## コンテキスト

```python
@click.group()
@click.option('--verbose', '-v', is_flag=True)
@click.pass_context
def cli(ctx, verbose):
    ctx.ensure_object(dict)
    ctx.obj['VERBOSE'] = verbose

@cli.command()
@click.pass_context
def create(ctx):
    if ctx.obj['VERBOSE']:
        click.echo('Verbose mode')
    click.echo('Creating...')
```

## カラー出力

```python
click.echo(click.style('Success!', fg='green', bold=True))
click.echo(click.style('Error!', fg='red'))
click.echo(click.style('Warning', fg='yellow'))

# ショートカット
click.secho('Success!', fg='green')
```

## 高度な機能

### カスタムパラメータ型

```python
import click

class EmailType(click.ParamType):
    name = "email"

    def convert(self, value, param, ctx):
        import re
        if not re.match(r'^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$', value):
            self.fail(f'{value} is not a valid email address', param, ctx)
        return value

EMAIL = EmailType()

@click.command()
@click.option('--email', type=EMAIL, required=True)
def register(email):
    """Register a new user"""
    click.echo(f'Registered: {email}')
```

### 進捗バー

```python
import click
import time

@click.command()
@click.option('--count', default=100, help='Number of items to process')
def process(count):
    """Process items with progress bar"""
    with click.progressbar(range(count), label='Processing') as bar:
        for item in bar:
            time.sleep(0.01)  # 処理のシミュレーション
    click.echo('Done!')
```

### 設定ファイルの読み込み

```python
import click
import os
import json

def read_config(ctx, param, value):
    """設定ファイルを読み込んでデフォルト値を設定"""
    if value and os.path.exists(value):
        with open(value) as f:
            config = json.load(f)
            ctx.default_map = config
    return value

@click.command()
@click.option('--config', type=click.Path(), callback=read_config,
              is_eager=True, expose_value=False, help='Config file path')
@click.option('--host', default='localhost', help='Server host')
@click.option('--port', default=8000, type=int, help='Server port')
def serve(host, port):
    """Start development server"""
    click.echo(f'Starting server on {host}:{port}')
```

### エラーハンドリング

```python
import click
import sys

class CustomError(click.ClickException):
    def show(self):
        click.secho(f'Error: {self.format_message()}', fg='red', err=True)

@click.command()
@click.argument('filename', type=click.Path(exists=True))
def process_file(filename):
    """Process a file"""
    try:
        with open(filename) as f:
            content = f.read()
            if not content:
                raise CustomError('File is empty')
        click.echo('Processing successful')
    except IOError as e:
        raise click.ClickException(f'Cannot read file: {e}')
```

## 実践的なCLI例

### 例1: プロジェクト管理ツール

```python
import click
import os
import json
from pathlib import Path

@click.group()
@click.version_option(version='1.0.0')
def cli():
    """Project management CLI tool"""
    pass

@cli.command()
@click.argument('name')
@click.option('--template', '-t',
              type=click.Choice(['python', 'nodejs', 'go']),
              default='python',
              help='Project template')
@click.option('--git/--no-git', default=True, help='Initialize git repository')
@click.option('--venv/--no-venv', default=True, help='Create virtual environment')
def create(name, template, git, venv):
    """Create a new project"""
    project_path = Path(name)

    if project_path.exists():
        raise click.ClickException(f'Directory {name} already exists')

    # プロジェクトディレクトリ作成
    project_path.mkdir()
    click.secho(f'Created directory: {name}', fg='green')

    # テンプレート適用
    if template == 'python':
        create_python_project(project_path, venv)
    elif template == 'nodejs':
        create_nodejs_project(project_path)
    elif template == 'go':
        create_go_project(project_path)

    # Git 初期化
    if git:
        import subprocess
        subprocess.run(['git', 'init'], cwd=project_path, check=True)
        click.secho('Initialized git repository', fg='green')

    click.secho(f'\nProject {name} created successfully!', fg='green', bold=True)

def create_python_project(path, venv):
    """Create Python project structure"""
    # ディレクトリ構造
    (path / 'src').mkdir()
    (path / 'tests').mkdir()

    # ファイル作成
    (path / 'README.md').write_text(f'# {path.name}\n')
    (path / 'requirements.txt').write_text('')
    (path / '.gitignore').write_text('__pycache__/\n*.pyc\nvenv/\n.env\n')

    if venv:
        import subprocess
        subprocess.run(['python', '-m', 'venv', 'venv'], cwd=path, check=True)
        click.secho('Created virtual environment', fg='green')

def create_nodejs_project(path):
    """Create Node.js project structure"""
    (path / 'src').mkdir()
    (path / 'tests').mkdir()

    package_json = {
        "name": path.name,
        "version": "1.0.0",
        "main": "src/index.js",
        "scripts": {
            "test": "jest"
        }
    }
    (path / 'package.json').write_text(json.dumps(package_json, indent=2))
    (path / '.gitignore').write_text('node_modules/\n.env\n')

def create_go_project(path):
    """Create Go project structure"""
    (path / 'cmd').mkdir()
    (path / 'pkg').mkdir()

    (path / 'go.mod').write_text(f'module {path.name}\n\ngo 1.19\n')
    (path / '.gitignore').write_text('*.exe\n*.test\n.env\n')

@cli.command()
@click.option('--all', '-a', is_flag=True, help='List all projects')
@click.option('--format', '-f',
              type=click.Choice(['table', 'json', 'plain']),
              default='table',
              help='Output format')
def list(all, format):
    """List projects in current directory"""
    projects = []
    for item in Path('.').iterdir():
        if item.is_dir() and not item.name.startswith('.'):
            # プロジェクトタイプを判定
            project_type = 'unknown'
            if (item / 'package.json').exists():
                project_type = 'nodejs'
            elif (item / 'requirements.txt').exists():
                project_type = 'python'
            elif (item / 'go.mod').exists():
                project_type = 'go'

            projects.append({
                'name': item.name,
                'type': project_type
            })

    if format == 'json':
        click.echo(json.dumps(projects, indent=2))
    elif format == 'table':
        click.echo('Name\t\tType')
        click.echo('-' * 40)
        for p in projects:
            click.echo(f'{p["name"]}\t\t{p["type"]}')
    else:
        for p in projects:
            click.echo(p['name'])

@cli.command()
@click.argument('project_name')
@click.confirmation_option(prompt='Are you sure you want to delete this project?')
def delete(project_name):
    """Delete a project"""
    import shutil
    project_path = Path(project_name)

    if not project_path.exists():
        raise click.ClickException(f'Project {project_name} not found')

    shutil.rmtree(project_path)
    click.secho(f'Deleted project: {project_name}', fg='red')

if __name__ == '__main__':
    cli()
```

### 例2: データベース管理ツール

```python
import click
import json
from datetime import datetime

@click.group()
@click.pass_context
def db(ctx):
    """Database management tool"""
    ctx.ensure_object(dict)
    ctx.obj['DB_CONFIG'] = load_db_config()

def load_db_config():
    """Load database configuration"""
    # 実際は設定ファイルから読み込む
    return {
        'host': 'localhost',
        'port': 5432,
        'database': 'myapp'
    }

@db.command()
@click.option('--sql', '-s', help='SQL file to execute')
@click.option('--query', '-q', help='SQL query to execute')
@click.pass_context
def execute(ctx, sql, query):
    """Execute SQL query or file"""
    config = ctx.obj['DB_CONFIG']

    if sql:
        with open(sql) as f:
            query = f.read()

    if not query:
        raise click.UsageError('Either --sql or --query must be provided')

    click.echo(f'Executing on {config["host"]}:{config["port"]}/{config["database"]}')
    click.echo(f'Query: {query}')
    # 実際のDB実行処理

@db.command()
@click.option('--output', '-o', type=click.File('w'), default='-',
              help='Output file (default: stdout)')
@click.option('--format', '-f',
              type=click.Choice(['sql', 'json']),
              default='sql',
              help='Backup format')
@click.pass_context
def backup(ctx, output, format):
    """Backup database"""
    config = ctx.obj['DB_CONFIG']

    click.echo(f'Backing up {config["database"]}...', err=True)

    # バックアップデータの生成（例）
    backup_data = {
        'timestamp': datetime.now().isoformat(),
        'database': config['database'],
        'tables': ['users', 'posts', 'comments']
    }

    if format == 'json':
        output.write(json.dumps(backup_data, indent=2))
    else:
        output.write(f"-- Backup of {config['database']}\n")
        output.write(f"-- Date: {backup_data['timestamp']}\n")

    click.secho('Backup completed!', fg='green', err=True)

@db.command()
@click.argument('backup_file', type=click.File('r'))
@click.option('--force', '-f', is_flag=True, help='Force restore without confirmation')
@click.pass_context
def restore(ctx, backup_file, force):
    """Restore database from backup"""
    config = ctx.obj['DB_CONFIG']

    if not force:
        click.confirm(
            f'This will overwrite {config["database"]}. Continue?',
            abort=True
        )

    backup_data = backup_file.read()
    click.echo(f'Restoring to {config["database"]}...')

    with click.progressbar(length=100, label='Restoring') as bar:
        # 復元処理のシミュレーション
        import time
        for i in range(100):
            time.sleep(0.01)
            bar.update(1)

    click.secho('Restore completed!', fg='green')

if __name__ == '__main__':
    db()
```

### 例3: API クライアントツール

```python
import click
import json
import sys

@click.group()
@click.option('--base-url', envvar='API_BASE_URL',
              default='https://api.example.com',
              help='API base URL')
@click.option('--api-key', envvar='API_KEY',
              help='API key for authentication')
@click.option('--verbose', '-v', is_flag=True, help='Verbose output')
@click.pass_context
def api(ctx, base_url, api_key, verbose):
    """API client tool"""
    ctx.ensure_object(dict)
    ctx.obj['BASE_URL'] = base_url
    ctx.obj['API_KEY'] = api_key
    ctx.obj['VERBOSE'] = verbose

@api.command()
@click.argument('endpoint')
@click.option('--method', '-m',
              type=click.Choice(['GET', 'POST', 'PUT', 'DELETE']),
              default='GET',
              help='HTTP method')
@click.option('--data', '-d', help='Request data (JSON)')
@click.option('--header', '-H', multiple=True, help='Additional headers')
@click.option('--output', '-o', type=click.File('w'), help='Output file')
@click.pass_context
def request(ctx, endpoint, method, data, header, output):
    """Make API request"""
    base_url = ctx.obj['BASE_URL']
    api_key = ctx.obj['API_KEY']
    verbose = ctx.obj['VERBOSE']

    url = f'{base_url}/{endpoint.lstrip("/")}'

    # ヘッダー構築
    headers = {'Authorization': f'Bearer {api_key}'} if api_key else {}
    for h in header:
        key, value = h.split(':', 1)
        headers[key.strip()] = value.strip()

    if verbose:
        click.echo(f'URL: {url}', err=True)
        click.echo(f'Method: {method}', err=True)
        click.echo(f'Headers: {json.dumps(headers, indent=2)}', err=True)

    # リクエスト実行（例）
    response_data = {
        'status': 200,
        'data': {'message': 'Success'}
    }

    # 出力
    output_text = json.dumps(response_data, indent=2)
    if output:
        output.write(output_text)
        click.secho(f'Response saved to {output.name}', fg='green', err=True)
    else:
        click.echo(output_text)

@api.command()
@click.option('--format', '-f',
              type=click.Choice(['table', 'json']),
              default='table',
              help='Output format')
@click.pass_context
def endpoints(ctx, format):
    """List available API endpoints"""
    # エンドポイント一覧（例）
    endpoints = [
        {'path': '/users', 'methods': ['GET', 'POST']},
        {'path': '/users/{id}', 'methods': ['GET', 'PUT', 'DELETE']},
        {'path': '/posts', 'methods': ['GET', 'POST']},
    ]

    if format == 'json':
        click.echo(json.dumps(endpoints, indent=2))
    else:
        click.echo('Endpoint\t\tMethods')
        click.echo('-' * 50)
        for ep in endpoints:
            methods = ', '.join(ep['methods'])
            click.echo(f'{ep["path"]}\t\t{methods}')

if __name__ == '__main__':
    api()
```

## Click vs Typer 比較

| 特徴 | Click | Typer |
|------|-------|-------|
| **API スタイル** | デコレータベース | 型ヒントベース |
| **Python バージョン** | 3.6+ | 3.6+ |
| **学習曲線** | 中程度 | 低（型ヒントに慣れている場合） |
| **自動補完** | サポート | 組み込み |
| **型安全性** | 実行時チェック | 型ヒントによる静的チェック |
| **ドキュメント生成** | 手動 | 自動（型から推論） |
| **依存関係** | 少ない | Click に依存 |
| **採用事例** | Ansible, AWS SAM CLI | FastAPI CLI（開発中） |
| **パフォーマンス** | 高速 | Click とほぼ同等 |
| **エコシステム** | 成熟 | 成長中 |

### どちらを選ぶべきか

**Click を選ぶ場合:**
- 安定性と実績を重視する
- 細かい制御が必要
- 軽量な依存関係を保ちたい
- 既存のClickプロジェクトとの互換性が必要

**Typer を選ぶ場合:**
- モダンなPython開発（型ヒント重視）
- 開発体験を重視する
- 自動補完機能が必要
- FastAPIなど他のモダンフレームワークと統合する

## 実装チェックリスト

Clickで堅牢なCLIツールを開発する際のチェックリストです。

### 基本機能
- [ ] 適切なコマンド名とサブコマンド構造
- [ ] わかりやすいヘルプメッセージ
- [ ] バージョン情報の表示（`--version`）
- [ ] エラーメッセージが明確で具体的

### オプション設計
- [ ] 短いオプション（`-v`）と長いオプション（`--verbose`）の両方を提供
- [ ] デフォルト値が適切に設定されている
- [ ] 必須オプションと任意オプションが明確
- [ ] 環境変数からの読み込みサポート

### 入力バリデーション
- [ ] 引数とオプションの型チェック
- [ ] カスタムバリデーション関数の実装
- [ ] ファイルパスの存在確認
- [ ] 相互排他的なオプションの処理

### ユーザー体験
- [ ] カラー出力で重要な情報を強調
- [ ] 進捗バーで長時間の処理を可視化
- [ ] 確認プロンプトで破壊的操作を保護
- [ ] 詳細モード（`--verbose`）の実装

### エラーハンドリング
- [ ] 適切な終了コード（0=成功、1=エラー）
- [ ] わかりやすいエラーメッセージ
- [ ] スタックトレースの制御（デバッグモード）
- [ ] 例外の適切なキャッチと処理

### テスト
- [ ] ClickのTestRunnerを使った単体テスト
- [ ] 異常系のテスト
- [ ] 環境変数のテスト
- [ ] ファイル入出力のテスト

### ドキュメント
- [ ] README にインストール方法を記載
- [ ] 使用例を豊富に用意
- [ ] すべてのコマンドとオプションを文書化
- [ ] トラブルシューティングガイド

### 配布
- [ ] setup.py または pyproject.toml の設定
- [ ] entry_points の正しい設定
- [ ] 依存関係の明示
- [ ] PyPI への公開準備

## まとめ

Clickは、成熟した安定性の高いCLIフレームワークです。デコレータベースの直感的なAPIにより、シンプルなツールから複雑なマルチコマンドツールまで幅広く対応できます。

本章で学んだ内容：
- Clickの基本的な使い方（コマンド、オプション、引数）
- コマンドグループによる階層的なCLI設計
- カスタムバリデーションとパラメータ型
- 実践的な3つのCLI実装例（プロジェクト管理、DB管理、API クライアント）
- ClickとTyperの比較と使い分け
- 実装チェックリスト

次章では、より modern な Typer フレームワークを学び、型ヒントを活用したCLI開発を習得します。
