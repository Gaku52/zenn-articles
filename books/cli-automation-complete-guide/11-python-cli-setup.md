---
title: "Python CLIセットアップ"
---

# Python CLIセットアップ

本章では、Pythonでプロフェッショナルなc

LIツールを開発するためのプロジェクトセットアップを学びます。適切な環境構築により、開発効率が大きく向上します。

## プロジェクト初期化

### 基本セットアップ

```bash
# プロジェクト作成
mkdir my-cli-tool
cd my-cli-tool

# 仮想環境作成
python -m venv venv

# 仮想環境有効化
source venv/bin/activate  # macOS/Linux
venv\Scripts\activate     # Windows

# 必須パッケージインストール
pip install click typer rich
```

### pyproject.toml（推奨）

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-cli-tool"
version = "1.0.0"
description = "A powerful CLI tool"
readme = "README.md"
requires-python = ">=3.8"
license = {text = "MIT"}
authors = [
    {name = "Your Name", email = "your.email@example.com"}
]
keywords = ["cli", "tool"]
classifiers = [
    "Development Status :: 4 - Beta",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.8",
    "Programming Language :: Python :: 3.9",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
]

dependencies = [
    "typer>=0.9.0",
    "rich>=13.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0.0",
    "black>=23.0.0",
    "ruff>=0.1.0",
    "mypy>=1.0.0",
]

[project.scripts]
mycli = "my_cli.main:app"

[project.urls]
Homepage = "https://github.com/username/my-cli-tool"
Repository = "https://github.com/username/my-cli-tool.git"
Documentation = "https://my-cli-tool.readthedocs.io"

[tool.black]
line-length = 100
target-version = ["py38", "py39", "py310", "py311"]

[tool.ruff]
line-length = 100
target-version = "py38"

[tool.mypy]
python_version = "3.8"
strict = true
```

## ディレクトリ構造

```
my-cli-tool/
├── src/
│   └── my_cli/
│       ├── __init__.py
│       ├── main.py          # エントリーポイント
│       ├── commands/        # コマンド実装
│       │   ├── __init__.py
│       │   ├── create.py
│       │   ├── list.py
│       │   └── delete.py
│       ├── core/           # ビジネスロジック
│       │   ├── __init__.py
│       │   ├── project.py
│       │   └── template.py
│       └── utils/          # ユーティリティ
│           ├── __init__.py
│           ├── logger.py
│           └── config.py
├── tests/
│   ├── __init__.py
│   ├── test_commands.py
│   └── test_core.py
├── templates/
├── pyproject.toml
├── README.md
├── LICENSE
└── .gitignore
```

## エントリーポイント

### Typerベースのエントリーポイント

```python
# src/my_cli/main.py
import typer
from my_cli.commands import create, list_cmd, delete

app = typer.Typer()

# コマンド登録
app.command()(create.create)
app.command("list")(list_cmd.list_projects)
app.command()(delete.delete)

if __name__ == "__main__":
    app()
```

### Shebang

```python
#!/usr/bin/env python3

import typer

app = typer.Typer()

# ...
```

## 依存関係管理

### Poetryを使用（推奨）

```bash
# Poetry インストール
curl -sSL https://install.python-poetry.org | python3 -

# プロジェクト初期化
poetry init

# 依存関係追加
poetry add typer rich
poetry add --group dev pytest black ruff mypy

# 仮想環境有効化
poetry shell

# インストール
poetry install
```

```toml
# pyproject.toml (Poetry)
[tool.poetry]
name = "my-cli-tool"
version = "1.0.0"
description = "A powerful CLI tool"
authors = ["Your Name <your.email@example.com>"]

[tool.poetry.dependencies]
python = "^3.8"
typer = "^0.9.0"
rich = "^13.0.0"

[tool.poetry.group.dev.dependencies]
pytest = "^7.0.0"
black = "^23.0.0"
ruff = "^0.1.0"
mypy = "^1.0.0"

[tool.poetry.scripts]
mycli = "my_cli.main:app"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

## リンター・フォーマッター

### Black（フォーマッター）

```bash
pip install black
```

```bash
# フォーマット実行
black src/

# チェックのみ
black --check src/
```

### Ruff（高速リンター）

```bash
pip install ruff
```

```bash
# リント実行
ruff check src/

# 自動修正
ruff check --fix src/
```

```toml
# pyproject.toml
[tool.ruff]
line-length = 100
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes
    "I",   # isort
    "N",   # pep8-naming
    "UP",  # pyupgrade
]
ignore = []

[tool.ruff.per-file-ignores]
"__init__.py" = ["F401"]
```

### Mypy（型チェック）

```bash
pip install mypy
```

```bash
# 型チェック実行
mypy src/
```

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.8"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
```

## テスト環境

### pytest

```bash
pip install pytest pytest-cov
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_functions = ["test_*"]
addopts = "--verbose --cov=my_cli --cov-report=term-missing"
```

```bash
# テスト実行
pytest

# カバレッジ付き
pytest --cov=my_cli --cov-report=html
```

## 開発ツール

### pre-commit

```bash
pip install pre-commit
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/psf/black
    rev: 23.12.1
    hooks:
      - id: black
        language_version: python3.11

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.9
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        additional_dependencies: [types-all]
```

```bash
# インストール
pre-commit install

# 全ファイルに実行
pre-commit run --all-files
```

## ローカルテスト

### 開発モードでインストール

```bash
# pipの場合
pip install -e .

# Poetryの場合
poetry install

# 実行
mycli --help
```

### ビルドとインストール

```bash
# ビルド
python -m build

# インストール
pip install dist/my_cli_tool-1.0.0-py3-none-any.whl

# 実行
mycli --help
```

## Git設定

### .gitignore

```
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
*.egg-info/
.installed.cfg
*.egg

# Virtual environments
venv/
ENV/
env/

# Testing
.pytest_cache/
.coverage
htmlcov/

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db
```

## まとめ

本章では、Python CLIプロジェクトのセットアップを学びました。

**重要ポイント**:
- pyproject.tomlで依存関係を管理
- Poetryで仮想環境を管理（推奨）
- Black/Ruffでコード品質を維持
- Mypyで型安全性を確保
- pytestでテストを自動化
- pre-commitでコミット前チェックを自動化

次章では、Clickフレームワークを使った実践的なCLI開発を学びます。
