---
title: "開発環境のセットアップ"
---

# Chapter 02: 開発環境のセットアップ

> Pythonの開発環境を構築し、最初のプログラムを実行する

## この章で学べること

- ✅ Pythonのインストール方法
- ✅ モダンなパッケージマネージャー「uv」の使い方
- ✅ 仮想環境の概念と作成方法
- ✅ VS Codeの設定
- ✅ コードフォーマッター「Ruff」の導入
- ✅ プロジェクト構成のベストプラクティス

---

## なぜ環境構築が大切なのか

プログラミング学習で最も挫折しやすいのが、実はこの「環境構築」のステップです。「コードを書きたいのに、まだ準備段階で詰まっている...」という経験をした人は少なくありません。

でも安心してください。この章では、2026年時点で**最もシンプルで確実な方法**を紹介します。ここを乗り越えれば、あとはコードを書くだけです。

> **環境構築は「1回だけの作業」** です。最初に正しくセットアップすれば、あとは快適にPython開発を進められます。

---

## Pythonのインストール

### macOS

```bash
# Homebrew でインストール（推奨）
brew install python

# バージョン確認
python3 --version
# Python 3.14.2
```

### Windows

```bash
# Microsoft Store からインストール（推奨）
# または python.org からインストーラーをダウンロード

# バージョン確認
python --version
# Python 3.14.2
```

### Linux（Ubuntu/Debian）

```bash
sudo apt update
sudo apt install python3 python3-pip

python3 --version
# Python 3.14.2
```

### `python` vs `python3` の違い

| コマンド | 説明 |
|---------|------|
| `python3` | Python 3.x を明示的に実行（推奨） |
| `python` | 環境によって Python 2 か 3 が実行される |

> **推奨**: 常に `python3` を使いましょう。ただし、仮想環境内では `python` でも Python 3 が実行されます。

---

## モダンなパッケージマネージャー: uv

### uvとは何か

**uv**は、Astral社が開発した**Rust製の超高速パッケージマネージャー**です。2024年にリリースされ、2026年現在ではPythonプロジェクトの標準ツールとなりつつあります。

| 特徴 | 説明 |
|------|------|
| **速度** | pip の10〜100倍高速 |
| **オールインワン** | パッケージ管理、仮想環境、プロジェクト管理を統合 |
| **ロックファイル** | 再現可能なビルドを保証 |
| **Python管理** | Python自体のインストールも可能 |

### インストール

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Homebrew
brew install uv

# Windows
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# バージョン確認
uv --version
```

### プロジェクトの作成

```bash
# 新しいプロジェクトを作成
uv init my-first-project
cd my-first-project

# 作成されるファイル
# my-first-project/
# ├── pyproject.toml    ← プロジェクト設定
# ├── README.md
# ├── hello.py          ← メインファイル
# └── .python-version   ← Pythonバージョン指定
```

### パッケージの管理

```bash
# パッケージの追加
uv add requests
uv add fastapi

# 開発用パッケージの追加
uv add --dev pytest ruff mypy

# パッケージの削除
uv remove requests

# プログラムの実行
uv run python hello.py

# スクリプトの直接実行
uv run hello.py
```

### pip との比較

```bash
# ❌ 従来の方法（pip）
pip install requests          # グローバルに汚染される可能性
pip freeze > requirements.txt # 手動でロックファイル作成

# ✅ モダンな方法（uv）
uv add requests               # 仮想環境に自動インストール
                               # uv.lock が自動生成される
```

---

## 仮想環境

### なぜ仮想環境が必要か

仮想環境がない場合、すべてのプロジェクトが同じPython環境を共有します。

```
❌ 仮想環境なし:
プロジェクトA（requests 2.31が必要）─┐
プロジェクトB（requests 2.28が必要）─┤→ 同じ場所にインストール → 衝突！
プロジェクトC（requests不要）────────┘

✅ 仮想環境あり:
プロジェクトA → .venv/ に requests 2.31
プロジェクトB → .venv/ に requests 2.28  → 独立！
プロジェクトC → .venv/ に何もなし
```

### 仮想環境の作成と有効化

```bash
# uv を使う場合（推奨）
uv venv

# 有効化
# macOS / Linux
source .venv/bin/activate

# Windows
.venv\Scripts\activate

# 有効化されると、プロンプトが変わる
# (my-first-project) $

# 無効化
deactivate
```

> **ポイント**: `uv run` を使えば、仮想環境を明示的に有効化しなくても自動的に使われます。

---

## エディタの設定: VS Code

### 必須の拡張機能

1. **Python**（Microsoft公式）
   - IntelliSense（自動補完）
   - デバッグ機能
   - テスト実行

2. **Ruff**（Astral公式）
   - リント（コード品質チェック）
   - フォーマット（コード整形）

### 推奨設定

VS Codeの設定ファイル（`.vscode/settings.json`）に以下を追加します：

```json
{
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll.ruff": "explicit",
      "source.organizeImports.ruff": "explicit"
    }
  },
  "python.analysis.typeCheckingMode": "basic"
}
```

**設定の意味**：
- 保存時にRuffで自動フォーマット
- import文の自動整理
- 基本的な型チェック

---

## 最初のプログラム

### hello.py を作成して実行

```python
# hello.py
print("Hello, Python!")
print("私のPython学習が始まります！")
```

```bash
# 実行
uv run python hello.py
# Hello, Python!
# 私のPython学習が始まります！
```

### REPL（対話モード）

Pythonには対話的にコードを試せるREPL（Read-Eval-Print Loop）があります。

```bash
$ python3
Python 3.14.2 (main, Jan 14 2026, 12:00:00)
>>> 2 + 3
5
>>> "Hello" + " " + "World"
'Hello World'
>>> [i ** 2 for i in range(5)]
[0, 1, 4, 9, 16]
>>> exit()
```

> **活用場面**: ちょっとした計算や文法の確認に便利です。この本のコード例も REPL で試してみてください。
>
> REPL は「プログラミングの電卓」のようなもの。思いついたことをすぐ試せる最強の学習ツールです。

---

## プロジェクト構成

### 基本的なディレクトリ構成

```
my-project/
├── pyproject.toml        ← プロジェクト設定（最重要）
├── uv.lock               ← 依存関係のロックファイル（自動生成）
├── README.md             ← プロジェクト説明
├── src/
│   └── my_project/
│       ├── __init__.py   ← パッケージの印
│       └── main.py       ← メインコード
├── tests/
│   ├── __init__.py
│   └── test_main.py      ← テストコード
└── .venv/                ← 仮想環境（Git管理しない）
```

### pyproject.toml の基本

```toml
[project]
name = "my-project"
version = "0.1.0"
description = "My first Python project"
requires-python = ">=3.14"
dependencies = [
    "requests>=2.31",
]

[dependency-groups]
dev = [
    "pytest>=8.0",
    "ruff>=0.8",
]

[tool.ruff]
line-length = 88
target-version = "py314"
```

---

## コードフォーマッターとリンター: Ruff

### Ruffとは

**Ruff**は、Astral社が開発した**Rust製の超高速リンター/フォーマッター**です。従来のflake8、black、isortの機能をすべて1つのツールで提供します。

| 比較 | 従来 | Ruff |
|------|------|------|
| リンター | flake8 | `ruff check` |
| フォーマッター | black | `ruff format` |
| import整理 | isort | `ruff check --select I` |
| 速度 | 数秒〜 | **ミリ秒** |

### 使い方

```bash
# インストール（プロジェクトに追加）
uv add --dev ruff

# コードチェック
uv run ruff check .

# 自動修正
uv run ruff check --fix .

# フォーマット
uv run ruff format .
```

---

## よくあるトラブルと解決策

### 1. `python` コマンドが見つからない

```bash
# ❌ エラー
$ python
command not found: python

# ✅ 解決策1: python3 を使う
$ python3 --version

# ✅ 解決策2: エイリアスを設定
# ~/.zshrc または ~/.bashrc に追加
alias python=python3
```

### 2. pip と uv の混在

```bash
# ❌ 混在させると混乱する
pip install requests
uv add pandas

# ✅ uv に統一する
uv add requests
uv add pandas
```

### 3. 仮想環境を忘れてグローバルにインストール

```bash
# ❌ 仮想環境なしでインストール（システム全体に影響）
pip install some-package

# ✅ 常にプロジェクト内で uv を使う
cd my-project
uv add some-package
```

---

## やってみよう！

環境構築ができたら、以下を試してみましょう：

1. ターミナルで `python3 --version` を実行して、Python 3.14 がインストールされていることを確認する
2. `uv init hello-python && cd hello-python` で新しいプロジェクトを作る
3. `hello.py` に `print("Hello, Python! 環境構築完了！")` と書いて `uv run python hello.py` で実行する
4. REPL を起動して `2 ** 10`（2の10乗）を計算してみる

> すべてできたら、環境構築は完了です。**おめでとうございます！** ここを乗り越えたあなたは、もう「Pythonプログラマー」の第一歩を踏み出しています。

---

## まとめ

この章では、Python の開発環境を構築しました：

- ✅ Python 3.14 のインストール
- ✅ uv（モダンなパッケージマネージャー）の導入と基本操作
- ✅ 仮想環境の概念と作成方法
- ✅ VS Code + Ruff の設定
- ✅ 最初のプログラムの実行
- ✅ プロジェクト構成のベストプラクティス

次の章から、Pythonの文法を本格的に学んでいきます。

---

**次の章**: Chapter 03 - 基本構文と型
