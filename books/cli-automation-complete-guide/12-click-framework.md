# Clickフレームワーク

Clickは、Pythonで最も人気のあるCLIフレームワークの一つです。本章では、Clickを使った引数パース、コマンド定義、オプション設定の基本を学びます。

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

## まとめ

Clickを使うことで、シンプルかつ強力なCLIツールを開発できます。次章では、より modern な Typer フレームワークを学びます。
