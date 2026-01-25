---
title: "Python CLIの配布"
---

# Python CLIの配布

開発したCLIツールをユーザーに届けるには、適切な配布方法の選択が重要です。本章では、PyPIへの公開、バイナリ配布、パッケージ管理ツールの活用など、Python CLIツールの配布方法を学びます。

## パッケージング方法の選択

Python CLIツールの配布方法は、主に以下の3つがあります。

### PyPI経由での配布

最も一般的な方法で、`pip install`でインストールできます。Pythonエコシステムに馴染み、依存関係の管理が容易です。

### バイナリ配布

PyInstallerやNuitkaを使用して、Pythonインタープリタを含む単一の実行可能ファイルを生成します。ユーザーがPythonをインストールする必要がありません。

### コンテナ配布

Dockerイメージとして配布することで、環境の一貫性を保証できます。

## pyproject.tomlによるモダンな設定

PEP 518以降、`pyproject.toml`が標準的なパッケージ設定ファイルとなりました。

```toml
# pyproject.toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "my-cli-tool"
version = "1.0.0"
description = "A powerful CLI tool for project management"
readme = "README.md"
authors = [
    {name = "Your Name", email = "your.email@example.com"}
]
license = {text = "MIT"}
classifiers = [
    "Development Status :: 4 - Beta",
    "Environment :: Console",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Operating System :: OS Independent",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.8",
    "Programming Language :: Python :: 3.9",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
    "Topic :: Software Development :: Build Tools",
]
keywords = ["cli", "tool", "project-management"]
requires-python = ">=3.8"
dependencies = [
    "typer[all]>=0.9.0",
    "rich>=13.0.0",
    "httpx>=0.24.0",
    "pydantic>=2.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "pytest-cov>=4.0",
    "pytest-mock>=3.10",
    "black>=23.0",
    "ruff>=0.1.0",
    "mypy>=1.0",
]
docs = [
    "mkdocs>=1.4",
    "mkdocs-material>=9.0",
]

[project.urls]
Homepage = "https://github.com/username/my-cli-tool"
Documentation = "https://my-cli-tool.readthedocs.io"
Repository = "https://github.com/username/my-cli-tool"
Changelog = "https://github.com/username/my-cli-tool/blob/main/CHANGELOG.md"
"Bug Tracker" = "https://github.com/username/my-cli-tool/issues"

[project.scripts]
mycli = "my_cli.main:app"
my-cli = "my_cli.main:app"

[tool.setuptools]
package-dir = {"" = "src"}

[tool.setuptools.packages.find]
where = ["src"]
include = ["my_cli*"]
exclude = ["tests*"]
```

この設定により、以下のエントリーポイントが自動生成されます。

```bash
# インストール後、以下のコマンドが使用可能になる
mycli --help
my-cli --help
```

## バージョン管理戦略

セマンティックバージョニング（SemVer）に従ってバージョン管理を行います。

### __version__の定義

```python
# src/my_cli/__init__.py
"""My CLI Tool - A powerful project management CLI"""

__version__ = "1.0.0"
__author__ = "Your Name"
__email__ = "your.email@example.com"

from my_cli.main import app

__all__ = ["app", "__version__"]
```

### 動的バージョン表示

```python
# src/my_cli/main.py
import typer
from my_cli import __version__

app = typer.Typer()

def version_callback(value: bool):
    if value:
        typer.echo(f"My CLI Tool version: {__version__}")
        raise typer.Exit()

@app.callback()
def main(
    version: bool = typer.Option(
        None,
        "--version", "-v",
        callback=version_callback,
        is_eager=True,
        help="Show version and exit"
    )
):
    """My CLI Tool - A powerful project management CLI"""
    pass
```

使用例は以下の通りです。

```bash
mycli --version
# My CLI Tool version: 1.0.0
```

## PyPIへの公開手順

### 1. アカウント登録

PyPIとTestPyPIのアカウントを作成します。

- PyPI: https://pypi.org/account/register/
- TestPyPI: https://test.pypi.org/account/register/

### 2. APIトークン生成

PyPIのアカウント設定からAPIトークンを生成し、`~/.pypirc`に保存します。

```ini
# ~/.pypirc
[distutils]
index-servers =
    pypi
    testpypi

[pypi]
username = __token__
password = pypi-AgEIcHlwaS5vcmc...

[testpypi]
username = __token__
password = pypi-AgENdGVzdC5weXBpLm9yZw...
```

### 3. ビルドツールのインストール

```bash
pip install build twine
```

### 4. パッケージのビルド

```bash
# クリーンビルド
rm -rf dist/ build/ *.egg-info

# パッケージビルド
python -m build

# 生成されるファイル
# dist/
# ├── my_cli_tool-1.0.0-py3-none-any.whl
# └── my_cli_tool-1.0.0.tar.gz
```

### 5. TestPyPIでテスト

本番環境にアップロードする前に、TestPyPIでテストします。

```bash
# TestPyPIにアップロード
twine upload --repository testpypi dist/*

# TestPyPIからインストールしてテスト
pip install --index-url https://test.pypi.org/simple/ my-cli-tool

# 動作確認
mycli --help
```

### 6. PyPIへの公開

テストが成功したら、本番環境に公開します。

```bash
# PyPIにアップロード
twine upload dist/*

# ユーザーは以下でインストール可能
pip install my-cli-tool
```

## Poetryによるパッケージ管理

Poetryは、依存関係管理とパッケージングを統合したモダンなツールです。

### Poetryのインストール

```bash
curl -sSL https://install.python-poetry.org | python3 -
```

### プロジェクトの初期化

```bash
poetry init
```

### pyproject.toml（Poetry版）

```toml
# pyproject.toml
[tool.poetry]
name = "my-cli-tool"
version = "1.0.0"
description = "A powerful CLI tool for project management"
authors = ["Your Name <your.email@example.com>"]
license = "MIT"
readme = "README.md"
homepage = "https://github.com/username/my-cli-tool"
repository = "https://github.com/username/my-cli-tool"
documentation = "https://my-cli-tool.readthedocs.io"
keywords = ["cli", "tool", "project-management"]
classifiers = [
    "Development Status :: 4 - Beta",
    "Environment :: Console",
    "Intended Audience :: Developers",
]

[tool.poetry.dependencies]
python = "^3.8"
typer = {extras = ["all"], version = "^0.9.0"}
rich = "^13.0.0"
httpx = "^0.24.0"
pydantic = "^2.0.0"

[tool.poetry.group.dev.dependencies]
pytest = "^7.0"
pytest-cov = "^4.0"
pytest-mock = "^3.10"
black = "^23.0"
ruff = "^0.1.0"
mypy = "^1.0"

[tool.poetry.scripts]
mycli = "my_cli.main:app"
my-cli = "my_cli.main:app"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

### Poetryの主要コマンド

```bash
# 依存関係のインストール
poetry install

# 依存関係の追加
poetry add typer[all]
poetry add --group dev pytest

# 依存関係の更新
poetry update

# パッケージビルド
poetry build

# PyPIへの公開
poetry publish

# ビルドと公開を同時実行
poetry publish --build
```

## バイナリ配布（PyInstaller）

PyInstallerを使用して、単一の実行可能ファイルを生成します。

### PyInstallerのインストール

```bash
pip install pyinstaller
```

### 基本的なビルド

```bash
# 単一ファイルとしてビルド
pyinstaller --onefile \
    --name mycli \
    --add-data "src/my_cli/templates:my_cli/templates" \
    src/my_cli/main.py

# 生成されるファイル
# dist/mycli（Linux/macOS）
# dist/mycli.exe（Windows）
```

### spec ファイルによる詳細設定

```python
# mycli.spec
# -*- mode: python ; coding: utf-8 -*-

block_cipher = None

a = Analysis(
    ['src/my_cli/main.py'],
    pathex=[],
    binaries=[],
    datas=[
        ('src/my_cli/templates', 'my_cli/templates'),
        ('README.md', '.'),
    ],
    hiddenimports=[
        'typer',
        'rich',
        'httpx',
    ],
    hookspath=[],
    hooksconfig={},
    runtime_hooks=[],
    excludes=[],
    win_no_prefer_redirects=False,
    win_private_assemblies=False,
    cipher=block_cipher,
    noarchive=False,
)

pyz = PYZ(a.pure, a.zipped_data, cipher=block_cipher)

exe = EXE(
    pyz,
    a.scripts,
    a.binaries,
    a.zipfiles,
    a.datas,
    [],
    name='mycli',
    debug=False,
    bootloader_ignore_signals=False,
    strip=False,
    upx=True,
    upx_exclude=[],
    runtime_tmpdir=None,
    console=True,
    disable_windowed_traceback=False,
    argv_emulation=False,
    target_arch=None,
    codesign_identity=None,
    entitlements_file=None,
)
```

specファイルを使ってビルドします。

```bash
pyinstaller mycli.spec
```

### クロスプラットフォームビルド

PyInstallerは実行環境のプラットフォーム用にのみビルドできるため、複数プラットフォーム向けには各環境でビルドが必要です。

```bash
# Linux向けビルド（Linuxマシンで実行）
pyinstaller --onefile --name mycli-linux src/my_cli/main.py

# macOS向けビルド（macOSマシンで実行）
pyinstaller --onefile --name mycli-macos src/my_cli/main.py

# Windows向けビルド（Windowsマシンで実行）
pyinstaller --onefile --name mycli-windows.exe src/my_cli/main.py
```

## GitHub Actionsでの自動リリース

CI/CDパイプラインで自動的にパッケージをビルド・公開します。

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build-and-publish:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install build twine

      - name: Build package
        run: python -m build

      - name: Publish to PyPI
        env:
          TWINE_USERNAME: __token__
          TWINE_PASSWORD: ${{ secrets.PYPI_API_TOKEN }}
        run: twine upload dist/*

  build-binaries:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]

    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pyinstaller

      - name: Build binary
        run: pyinstaller --onefile --name mycli src/my_cli/main.py

      - name: Upload binary
        uses: actions/upload-artifact@v4
        with:
          name: mycli-${{ matrix.os }}
          path: dist/mycli*
```

## Python CLI配布チェックリスト

効果的な配布のために、以下を確認しましょう。

- [ ] pyproject.tomlに必要な情報を記述している
- [ ] バージョン番号がセマンティックバージョニングに従っている
- [ ] README.mdに使用方法を明記している
- [ ] LICENSEファイルを含めている
- [ ] CHANGELOGを管理している
- [ ] TestPyPIでテストしてから本番公開している
- [ ] エントリーポイントが正しく設定されている
- [ ] 依存関係のバージョン範囲が適切である
- [ ] classifiersが適切に設定されている
- [ ] CI/CDでリリースプロセスが自動化されている

## まとめ

Python CLIツールの配布方法は多様ですが、PyPI経由での配布が最も標準的です。pyproject.tomlによるモダンな設定と、Poetryによる依存関係管理を活用することで、保守性の高いパッケージングを実現できます。バイナリ配布やCI/CDでの自動化も、プロジェクトの要件に応じて検討しましょう。

次章では、Python CLIツール開発のベストプラクティスを学びます。

## 参考文献

- [Python Packaging User Guide](https://packaging.python.org/)
- [PEP 518 - Specifying Minimum Build System Requirements](https://peps.python.org/pep-0518/)
- [Poetry公式ドキュメント](https://python-poetry.org/docs/)
- [PyInstaller公式ドキュメント](https://pyinstaller.org/)
- [Twine公式ドキュメント](https://twine.readthedocs.io/)
