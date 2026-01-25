# Typer - モダンなPython CLI

Typerは、Pythonの型ヒントを最大限に活用したモダンなCLIフレームワークです。FastAPIと同じ開発者によって作成され、型安全性と開発者体験を重視した設計が特徴です。本章では、Typerの基本から実践的な使い方まで学びます。

## Typerの特徴と利点

Typerは以下の特徴を持つ、次世代のCLIフレームワークです。

### 型ヒントによる自動処理

Python 3.6以降の型ヒントを活用し、引数の型チェック、バリデーション、ヘルプメッセージの自動生成を行います。型安全性が高く、IDEの補完機能も効果的に機能します。

### 最小限のコード

従来のCLIフレームワークと比較して、必要なコード量が大幅に削減されます。型ヒントから自動的に多くの処理が推論されるため、開発者はビジネスロジックに集中できます。

### Richとの統合

デフォルトでRichライブラリと統合されており、美しい色付き出力、プログレスバー、テーブル表示などを簡単に実装できます。

## 基本的な使い方

最もシンプルなTyperアプリケーションから始めましょう。

```python
# simple_cli.py
import typer

def main(name: str, verbose: bool = False):
    """
    Create a new project

    Args:
        name: Project name
        verbose: Enable verbose output
    """
    typer.echo(f'Creating project: {name}')
    if verbose:
        typer.echo('Verbose mode enabled')
        typer.echo('Setting up directory structure...')

if __name__ == "__main__":
    typer.run(main)
```

このコードは、以下のようなCLIインターフェースを自動生成します。

```bash
python simple_cli.py --help
# Usage: simple_cli.py [OPTIONS] NAME
#
# Create a new project
#
# Arguments:
#   NAME  [required]
#
# Options:
#   --verbose / --no-verbose  [default: no-verbose]
#   --help                    Show this message and exit.

python simple_cli.py myproject --verbose
# Creating project: myproject
# Verbose mode enabled
# Setting up directory structure...
```

型ヒントから引数の型が自動的に判定され、適切なバリデーションが適用されます。

## 複数コマンドの実装

複数のサブコマンドを持つCLIツールを作成するには、`typer.Typer()`アプリケーションインスタンスを使用します。

```python
# multi_command_cli.py
import typer
from typing import Optional

app = typer.Typer(
    name="projectctl",
    help="Project management CLI tool",
    add_completion=True
)

@app.command()
def create(
    name: str,
    template: str = typer.Option('default', help='Project template'),
    force: bool = typer.Option(False, '--force', '-f', help='Force overwrite')
):
    """
    Create a new project
    """
    if force:
        typer.secho(f'Force creating: {name}', fg=typer.colors.YELLOW)
    else:
        typer.secho(f'Creating: {name}', fg=typer.colors.GREEN)

    typer.echo(f'Template: {template}')

@app.command('list')
def list_projects(
    all: bool = typer.Option(False, '--all', '-a', help='Show all projects'),
    format: str = typer.Option('table', help='Output format: table, json, yaml')
):
    """
    List all projects
    """
    typer.echo(f'Listing projects (all={all}, format={format})')

@app.command()
def delete(
    name: str,
    confirm: bool = typer.Option(False, '--yes', '-y', help='Skip confirmation')
):
    """
    Delete a project
    """
    if not confirm:
        confirm = typer.confirm(f'Delete project "{name}"?')

    if confirm:
        typer.secho(f'Deleted: {name}', fg=typer.colors.RED)
    else:
        typer.echo('Cancelled')

if __name__ == "__main__":
    app()
```

コマンドの使用例は以下の通りです。

```bash
python multi_command_cli.py create myapp --template react
python multi_command_cli.py list --all --format json
python multi_command_cli.py delete myapp --yes
```

## Enumによる選択肢の制限

Pythonの`Enum`を使用することで、引数の値を特定の選択肢に制限できます。

```python
from enum import Enum
import typer

class Environment(str, Enum):
    development = "dev"
    staging = "staging"
    production = "prod"

class LogLevel(str, Enum):
    debug = "debug"
    info = "info"
    warning = "warning"
    error = "error"

app = typer.Typer()

@app.command()
def deploy(
    env: Environment = typer.Argument(..., help="Target environment"),
    log_level: LogLevel = typer.Option(LogLevel.info, help="Log level")
):
    """
    Deploy application to specified environment
    """
    typer.echo(f'Deploying to: {env.value}')
    typer.echo(f'Log level: {log_level.value}')

    if env == Environment.production:
        typer.secho('WARNING: Production deployment!', fg=typer.colors.RED, bold=True)

if __name__ == "__main__":
    app()
```

Enumを使用すると、ヘルプメッセージに選択肢が自動的に表示され、無効な値が入力された場合はエラーメッセージが表示されます。

```bash
python deploy_cli.py --help
# Environment: [dev|staging|prod]  [required]

python deploy_cli.py invalid
# Error: Invalid value for 'ENV': 'invalid' is not one of 'dev', 'staging', 'prod'.
```

## インタラクティブなプロンプト

ユーザーからの入力を対話的に受け取るプロンプト機能も簡単に実装できます。

```python
import typer
from typing import Optional

app = typer.Typer()

@app.command()
def configure(
    name: str = typer.Option(..., prompt=True, help="Your name"),
    email: str = typer.Option(..., prompt=True, help="Your email"),
    password: str = typer.Option(
        ...,
        prompt=True,
        confirmation_prompt=True,
        hide_input=True,
        help="Your password"
    ),
    api_key: Optional[str] = typer.Option(
        None,
        prompt="API Key (optional)",
        help="API key for external service"
    )
):
    """
    Configure user settings interactively
    """
    typer.echo(f'Configuration saved:')
    typer.echo(f'Name: {name}')
    typer.echo(f'Email: {email}')
    typer.echo(f'Password: {"*" * len(password)}')
    if api_key:
        typer.echo(f'API Key: {api_key[:4]}...')

if __name__ == "__main__":
    app()
```

実行すると、各オプションについて対話的にプロンプトが表示されます。

```bash
python configure_cli.py
# Name: John Doe
# Email: john@example.com
# Password:
# Repeat for confirmation:
# API Key (optional):
# Configuration saved:
# Name: John Doe
# Email: john@example.com
# Password: ********
```

## Richとの統合による美しい出力

TyperはRichライブラリと統合されており、美しい出力を簡単に実装できます。

```python
import typer
from rich.console import Console
from rich.progress import Progress, SpinnerColumn, TextColumn, BarColumn
from rich.table import Table
import time

app = typer.Typer()
console = Console()

@app.command()
def create(name: str):
    """
    Create a new project with progress visualization
    """
    console.print(f'[bold cyan]Creating project: {name}[/bold cyan]')

    with Progress(
        SpinnerColumn(),
        TextColumn("[progress.description]{task.description}"),
        BarColumn(),
        TextColumn("[progress.percentage]{task.percentage:>3.0f}%"),
        console=console
    ) as progress:
        task1 = progress.add_task("[cyan]Initializing...", total=100)
        task2 = progress.add_task("[green]Installing dependencies...", total=100)
        task3 = progress.add_task("[yellow]Configuring...", total=100)

        for i in range(100):
            time.sleep(0.01)
            progress.update(task1, advance=1)

        for i in range(100):
            time.sleep(0.01)
            progress.update(task2, advance=1)

        for i in range(100):
            time.sleep(0.01)
            progress.update(task3, advance=1)

    console.print('[bold green]✓ Project created successfully![/bold green]')

@app.command('list')
def list_projects():
    """
    List all projects in a formatted table
    """
    table = Table(title="Projects")

    table.add_column("Name", style="cyan", no_wrap=True)
    table.add_column("Template", style="magenta")
    table.add_column("Status", style="green")
    table.add_column("Created", style="yellow")

    table.add_row("project-a", "react", "active", "2024-01-15")
    table.add_row("project-b", "vue", "active", "2024-01-18")
    table.add_row("project-c", "angular", "archived", "2024-01-10")

    console.print(table)

if __name__ == "__main__":
    app()
```

このコードは、プログレスバー付きのプロジェクト作成と、美しいテーブル表示を実現します。

## バリデーションとエラーハンドリング

Typerでは、カスタムバリデーションを簡単に実装できます。

```python
import typer
from pathlib import Path
from typing import Optional

app = typer.Typer()

def validate_project_name(name: str) -> str:
    """Validate project name format"""
    if not name.replace('-', '').replace('_', '').isalnum():
        raise typer.BadParameter('Project name must contain only alphanumeric characters, hyphens, and underscores')
    if len(name) < 3:
        raise typer.BadParameter('Project name must be at least 3 characters long')
    if len(name) > 50:
        raise typer.BadParameter('Project name must be at most 50 characters long')
    return name.lower()

def validate_path(path: str) -> Path:
    """Validate and convert path"""
    p = Path(path)
    if not p.exists():
        raise typer.BadParameter(f'Path does not exist: {path}')
    if not p.is_dir():
        raise typer.BadParameter(f'Path is not a directory: {path}')
    return p

@app.command()
def create(
    name: str = typer.Argument(..., callback=lambda x: validate_project_name(x)),
    output_dir: Optional[Path] = typer.Option(
        None,
        '--output', '-o',
        callback=lambda x: validate_path(x) if x else Path.cwd()
    )
):
    """
    Create a new project with validation
    """
    typer.echo(f'Creating project: {name}')
    typer.echo(f'Output directory: {output_dir}')

if __name__ == "__main__":
    app()
```

## Typer実装チェックリスト

Typerを使ったCLI開発では、以下の点を確認しましょう。

- [ ] すべての関数に型ヒントを適切に付与している
- [ ] docstringでコマンドの説明を記述している
- [ ] 長いオプション名と短いオプション名の両方を提供している
- [ ] デフォルト値が適切に設定されている
- [ ] Enumを使って選択肢を制限している
- [ ] バリデーションロジックを実装している
- [ ] エラーメッセージが分かりやすい
- [ ] Richを活用して視覚的に分かりやすい出力を実現している
- [ ] プログレスバーで長時間処理の進捗を表示している
- [ ] confirm()で危険な操作に確認を求めている

## まとめ

Typerは、型ヒントを活用することで、最小限のコードで型安全かつ保守性の高いCLIツールを開発できるモダンなフレームワークです。Richとの統合により、美しく使いやすいCLIツールを簡単に実装できます。

次章では、開発したPython CLIツールをpytestでテストする手法を学びます。

## 参考文献

- [Typer公式ドキュメント](https://typer.tiangolo.com/)
- [Rich公式ドキュメント](https://rich.readthedocs.io/)
- [Python Type Hints - PEP 484](https://peps.python.org/pep-0484/)
