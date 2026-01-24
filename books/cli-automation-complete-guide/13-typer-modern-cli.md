# Typerモダンcli

Typerは型ヒントを活用した、モダンでシンプルなCLIフレームワークです。本章では、Typerの基本から実践的な使い方まで学びます。

## 基本的な使い方

```python
import typer

def main(name: str, verbose: bool = False):
    """Create a new project"""
    typer.echo(f'Creating: {name}')
    if verbose:
        typer.echo('Verbose mode')

if __name__ == "__main__":
    typer.run(main)
```

## 複数コマンド

```python
import typer

app = typer.Typer()

@app.command()
def create(name: str, template: str = typer.Option('default', help='Template')):
    """Create a project"""
    typer.echo(f'Creating {name} with {template}')

@app.command('list')
def list_projects(all: bool = typer.Option(False, '--all', '-a')):
    """List projects"""
    typer.echo('Listing projects...')

if __name__ == "__main__":
    app()
```

## Enum選択肢

```python
from enum import Enum
import typer

class Environment(str, Enum):
    dev = "dev"
    staging = "staging"
    production = "production"

@app.command()
def deploy(env: Environment):
    typer.echo(f'Deploying to {env.value}')
```

## プロンプト

```python
@app.command()
def configure(
    name: str = typer.Option(..., prompt=True),
    password: str = typer.Option(..., prompt=True, hide_input=True)
):
    typer.echo(f'Configured: {name}')
```

## Richとの統合

```python
from rich.console import Console
from rich.progress import track
import time

console = Console()

@app.command()
def create(name: str):
    console.print(f'[cyan]Creating project: {name}[/cyan]')

    for _ in track(range(100), description='Setting up...'):
        time.sleep(0.01)

    console.print('[green]✓ Complete![/green]')
```

## まとめ

Typerを使うことで、型安全で保守性の高いCLIツールを簡単に開発できます。次章では、Pythonc CLIのテスト手法を学びます。
