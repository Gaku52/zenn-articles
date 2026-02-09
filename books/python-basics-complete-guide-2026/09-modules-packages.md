---
title: "モジュール・パッケージ・依存管理"
---

# Chapter 09: モジュール・パッケージ・依存管理

> コードを整理し、外部ライブラリを活用する

## この章で学べること

- ✅ モジュールとimportの仕組み
- ✅ パッケージの作成と構成
- ✅ 標準ライブラリの活用
- ✅ pyproject.toml の書き方
- ✅ uv での依存管理

---

## なぜモジュールを学ぶのか

プログラムが成長すると、1つのファイルにすべてのコードを書くのは限界があります。モジュールとパッケージを使うと、**コードを複数のファイルに分割して整理**できます。

> **たとえるなら**: 1冊のノートにすべてのメモを書くと探しにくくなりますよね。「数学」「英語」「理科」とノートを分ければ、必要な情報がすぐ見つかります。それがモジュールの考え方です。

また、Pythonには**40万以上の外部パッケージ**があり、他の人が作った便利な機能をすぐに使えます。「車輪の再発明」をせず、既存のパッケージを活用することが、効率的な開発の鍵です。

---

## モジュールとは

### 基本的な import

```python
# モジュール全体をインポート
import math
print(math.sqrt(16))  # 4.0
print(math.pi)        # 3.141592653589793

# 特定の名前だけインポート
from math import sqrt, pi
print(sqrt(16))  # 4.0
print(pi)        # 3.141592653589793

# エイリアスをつける
import datetime as dt
now = dt.datetime.now()

from collections import defaultdict as dd
counts = dd(int)
```

### import のアンチパターン

```python
# ❌ ワイルドカードインポート（名前空間が汚染される）
from math import *
# どの名前が math から来たのか分からなくなる

# ❌ 別名が意味不明
import pandas as p  # pd が慣習

# ✅ 明示的にインポート
from math import sqrt, pi, ceil, floor

# ✅ 慣習的なエイリアス
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
```

---

## パッケージとは

### ディレクトリ構成

```
my_app/
├── __init__.py      ← このディレクトリがパッケージであることを示す
├── main.py
├── models/
│   ├── __init__.py
│   ├── user.py
│   └── product.py
└── utils/
    ├── __init__.py
    ├── validators.py
    └── formatters.py
```

### __init__.py の役割

```python
# my_app/models/__init__.py
# パッケージの公開APIを定義

from .user import User
from .product import Product

__all__ = ["User", "Product"]
```

```python
# 使う側
from my_app.models import User, Product

# __init__.py で公開しておけば、内部構造を意識せずに使える
```

### 絶対インポート vs 相対インポート

```python
# ✅ 絶対インポート（推奨）
from my_app.models.user import User
from my_app.utils.validators import validate_email

# 相対インポート（同一パッケージ内）
from .user import User           # 同じディレクトリ
from ..utils.validators import validate_email  # 1つ上のディレクトリ
```

---

## 標準ライブラリの活用

### pathlib（ファイルパス操作）

```python
from pathlib import Path

# ✅ pathlib（推奨）
path = Path("data") / "output" / "result.csv"
print(path)  # data/output/result.csv

# ファイル操作
path.exists()      # 存在確認
path.is_file()     # ファイルか
path.is_dir()      # ディレクトリか
path.suffix        # ".csv"
path.stem          # "result"
path.parent        # Path("data/output")

# ディレクトリ内のファイル一覧
for f in Path(".").glob("*.py"):
    print(f)

# 再帰的な検索
for f in Path(".").rglob("*.md"):
    print(f)

# ❌ os.path（古い書き方）
import os
path = os.path.join("data", "output", "result.csv")
```

### json

```python
import json

# Python → JSON文字列
data = {"name": "太郎", "age": 25, "scores": [85, 92, 78]}
json_str = json.dumps(data, ensure_ascii=False, indent=2)
print(json_str)
# {
#   "name": "太郎",
#   "age": 25,
#   "scores": [85, 92, 78]
# }

# JSON文字列 → Python
parsed = json.loads(json_str)
print(parsed["name"])  # 太郎

# ファイルへの読み書き
from pathlib import Path

# 書き込み
Path("config.json").write_text(
    json.dumps(data, ensure_ascii=False, indent=2),
    encoding="utf-8"
)

# 読み込み
loaded = json.loads(Path("config.json").read_text(encoding="utf-8"))
```

### datetime

```python
from datetime import datetime, date, timedelta

# 現在日時
now = datetime.now()
print(now)  # 2026-02-09 21:30:00.123456

# 日付のフォーマット
print(now.strftime("%Y年%m月%d日"))  # 2026年02月09日
print(now.strftime("%H:%M:%S"))      # 21:30:00

# 文字列 → datetime
dt = datetime.strptime("2026-02-09", "%Y-%m-%d")

# 日付の計算
tomorrow = date.today() + timedelta(days=1)
next_week = date.today() + timedelta(weeks=1)
```

### collections

```python
from collections import Counter, defaultdict, deque

# Counter: 出現回数のカウント
words = ["apple", "banana", "apple", "cherry", "banana", "apple"]
counter = Counter(words)
print(counter.most_common(2))  # [('apple', 3), ('banana', 2)]

# defaultdict: デフォルト値付き辞書
groups = defaultdict(list)
for name, dept in [("太郎", "開発"), ("花子", "営業"), ("次郎", "開発")]:
    groups[dept].append(name)
print(dict(groups))  # {'開発': ['太郎', '次郎'], '営業': ['花子']}

# deque: 両端キュー（先頭への追加/削除がO(1)）
queue = deque(maxlen=3)
queue.append(1)
queue.append(2)
queue.append(3)
queue.append(4)  # 先頭の1が自動削除
print(list(queue))  # [2, 3, 4]
```

### itertools

```python
from itertools import chain, product, combinations, permutations

# chain: 複数のイテラブルを連結
list(chain([1, 2], [3, 4], [5]))  # [1, 2, 3, 4, 5]

# product: 直積（すべての組み合わせ）
list(product("AB", "12"))  # [('A','1'), ('A','2'), ('B','1'), ('B','2')]

# combinations: 組み合わせ
list(combinations("ABC", 2))  # [('A','B'), ('A','C'), ('B','C')]

# permutations: 順列
list(permutations("ABC", 2))  # [('A','B'), ('A','C'), ('B','A'), ...]
```

### functools

```python
from functools import lru_cache, partial

# lru_cache: 結果をキャッシュ（メモ化）
@lru_cache(maxsize=128)
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

print(fibonacci(100))  # 354224848179261915075（一瞬で計算）

# partial: 引数を一部固定した新しい関数を作る
def power(base: int, exponent: int) -> int:
    return base ** exponent

square = partial(power, exponent=2)
cube = partial(power, exponent=3)

print(square(5))  # 25
print(cube(3))    # 27
```

---

## pyproject.toml

### 完全な例

```toml
[project]
name = "my-awesome-app"
version = "1.0.0"
description = "Pythonで作った素晴らしいアプリ"
requires-python = ">=3.14"
license = {text = "MIT"}
authors = [
    {name = "GK", email = "gk@example.com"},
]
dependencies = [
    "fastapi>=0.115",
    "httpx>=0.27",
]

[dependency-groups]
dev = [
    "pytest>=8.0",
    "ruff>=0.8",
    "mypy>=1.13",
]

[tool.ruff]
line-length = 88
target-version = "py314"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP"]

[tool.mypy]
strict = true
python_version = "3.14"

[tool.pytest.ini_options]
testpaths = ["tests"]
```

---

## 依存管理: uv

### 日常のワークフロー

```bash
# プロジェクト作成
uv init my-project
cd my-project

# パッケージの追加
uv add requests          # 本番依存
uv add --dev pytest ruff # 開発依存

# パッケージの削除
uv remove requests

# ロックファイルの同期
uv lock    # uv.lock を生成/更新
uv sync    # uv.lock に基づいて環境を同期

# プログラムの実行
uv run python main.py
uv run pytest

# インストール済みパッケージの確認
uv pip list
```

### uv.lock ファイルの意味

```
uv.lock は依存関係の「正確なバージョン」を記録するファイルです。

pyproject.toml: "requests>=2.31"（範囲指定）
uv.lock:        "requests==2.32.3"（正確なバージョン + ハッシュ）

✅ uv.lock は Git にコミットする
✅ チーム全員が同じバージョンの依存関係を使える
✅ CI/CD でも再現可能なビルドが保証される
```

---

## 自作モジュールの作成

### __name__ == "__main__" パターン

```python
# utils/math_utils.py

def factorial(n: int) -> int:
    """階乗を計算する"""
    if n < 0:
        raise ValueError("負の数の階乗は定義されていません")
    if n <= 1:
        return 1
    return n * factorial(n - 1)

def is_prime(n: int) -> bool:
    """素数判定"""
    if n < 2:
        return False
    for i in range(2, int(n ** 0.5) + 1):
        if n % i == 0:
            return False
    return True

# このファイルが直接実行された場合のみ動作
if __name__ == "__main__":
    # テスト用コード
    print(f"5! = {factorial(5)}")
    print(f"17 is prime: {is_prime(17)}")
```

```python
# main.py（別ファイルから使う）
from utils.math_utils import factorial, is_prime

print(factorial(10))  # 3628800
print(is_prime(97))   # True
```

---

## 実践例

### 例1: プロジェクト構成の設計

```
todo-app/
├── pyproject.toml
├── uv.lock
├── README.md
├── src/
│   └── todo/
│       ├── __init__.py
│       ├── main.py          # エントリーポイント
│       ├── models.py         # データモデル
│       ├── repository.py     # データ保存
│       └── cli.py            # CLIインターフェース
└── tests/
    ├── __init__.py
    ├── test_models.py
    └── test_repository.py
```

### 例2: ユーティリティパッケージ

```python
# src/todo/models.py
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class Todo:
    title: str
    done: bool = False
    created_at: datetime = field(default_factory=datetime.now)

    def toggle(self) -> None:
        self.done = not self.done

    def __str__(self) -> str:
        status = "✅" if self.done else "⬜"
        return f"{status} {self.title}"
```

```python
# src/todo/repository.py
import json
from pathlib import Path
from .models import Todo

class TodoRepository:
    def __init__(self, filepath: Path) -> None:
        self.filepath = filepath
        self._todos: list[Todo] = []
        self._load()

    def _load(self) -> None:
        if self.filepath.exists():
            data = json.loads(self.filepath.read_text(encoding="utf-8"))
            self._todos = [Todo(title=t["title"], done=t["done"]) for t in data]

    def _save(self) -> None:
        data = [{"title": t.title, "done": t.done} for t in self._todos]
        self.filepath.write_text(
            json.dumps(data, ensure_ascii=False, indent=2),
            encoding="utf-8"
        )

    def add(self, title: str) -> Todo:
        todo = Todo(title=title)
        self._todos.append(todo)
        self._save()
        return todo

    def all(self) -> list[Todo]:
        return self._todos.copy()

    def toggle(self, index: int) -> None:
        self._todos[index].toggle()
        self._save()
```

---

## よくある間違い

### 間違い1: 循環インポート

```python
# ❌ a.py が b.py をインポートし、b.py が a.py をインポート
# a.py
from b import func_b  # b.py を読み込む

# b.py
from a import func_a  # a.py を読み込む → ImportError!

# ✅ 解決策: 構造を見直す、または遅延インポート
# b.py
def some_function():
    from a import func_a  # 関数内でインポート（遅延）
    func_a()
```

### 間違い2: from module import * の乱用

```python
# ❌ 名前空間が汚染される
from os.path import *
from math import *
# 両方に同じ名前があったら衝突する！

# ✅ 明示的にインポート
from os.path import join, exists
from math import sqrt, pi
```

### 間違い3: 仮想環境なしでインストール

```python
# ❌ システムPythonに直接インストール
# pip install requests  # 他のプロジェクトに影響する

# ✅ プロジェクトごとに uv を使う
# uv add requests  # .venv 内にインストール
```

---

## やってみよう！

1. **標準ライブラリ探検**: `datetime` を使って、自分の誕生日から今日まで何日経ったかを計算するプログラムを書いてみましょう
2. **自作モジュール**: `utils.py` に便利な関数を3つ以上まとめて、`main.py` から import して使ってみましょう
3. **パッケージ体験**: `uv init my-tool && cd my-tool` で新しいプロジェクトを作り、`uv add httpx` で外部パッケージを追加して、Web APIからデータを取得するプログラムを書いてみましょう

> **モジュールとパッケージを理解すると、「小さなスクリプト」から「本格的なプロジェクト」へステップアップできます。** この章は実務で毎日使う知識なので、しっかり身につけておきましょう。

---

## まとめ

この章では、モジュールとパッケージを学びました：

- ✅ import でモジュールを読み込み、コードを再利用
- ✅ `__init__.py` でパッケージの公開APIを定義
- ✅ 標準ライブラリ（pathlib, json, datetime, collections）を活用
- ✅ pyproject.toml でプロジェクトを設定
- ✅ uv で依存管理（uv.lock で再現可能なビルド）

次の章では、ファイル操作と例外処理を学びます。

---

**次の章**: Chapter 10 - ファイル操作と例外処理
