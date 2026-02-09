---
title: "型ヒントとデータクラス"
---

# Chapter 07: 型ヒントとデータクラス

> 型の力でバグを未然に防ぎ、コードの品質を高める

## この章で学べること

- ✅ 型ヒントの基本と重要性
- ✅ コレクション型、Optional、Union
- ✅ ジェネリクス
- ✅ dataclass でデータを構造化
- ✅ TypedDict と Protocol

---

## なぜ型ヒントを学ぶのか

Chapter 03 で「Pythonは型を宣言しなくても動く」と学びました。では、なぜわざわざ型を書くのでしょうか？

それは、**プログラムが大きくなるほど「この変数に何が入っているか分からない」問題が深刻になる**からです。

> **たとえるなら**: 小さな家なら「あの引き出しにハサミがある」と覚えられます。でもオフィスビルだと、ラベルなしの引き出しでは何がどこにあるか分からなくなります。型ヒントは「引き出しのラベル」です。

型ヒントを書くメリット：
- VS Code が**自動補完**してくれるようになる（開発速度が上がる）
- **間違いをコードを動かす前に発見**できる
- コードが**ドキュメント代わり**になる

### 動的型付けの課題

```python
# ❌ 型ヒントなし: 何が渡されるか分からない
def process_data(data):
    return data.split(",")  # data が str でないとエラー

process_data(123)  # 実行時に AttributeError!
```

### 型ヒントの利点

```python
# ✅ 型ヒントあり: エディタが事前に警告してくれる
def process_data(data: str) -> list[str]:
    return data.split(",")

process_data(123)  # エディタが「int は str ではない」と警告
```

| 利点 | 説明 |
|------|------|
| **IDE補完** | 型に応じたメソッドが自動補完される |
| **バグ防止** | 型の不一致をコード実行前に検出 |
| **ドキュメント** | 関数の使い方が型から読み取れる |
| **リファクタリング** | 変更の影響範囲が型で分かる |

### 静的チェックツール

```bash
# mypy: 最も広く使われている型チェッカー
uv add --dev mypy
uv run mypy src/

# pyright: Microsoft製、高速（VS Code Python拡張に内蔵）
uv add --dev pyright
uv run pyright src/
```

---

## 基本的な型ヒント

### 変数の型ヒント

```python
# 基本型
name: str = "Python"
age: int = 33
pi: float = 3.14
is_active: bool = True
nothing: None = None

# 型推論が効くので、明らかな場合は省略可能
name = "Python"  # str と推論される
age = 33         # int と推論される
```

### 関数の型ヒント

```python
# 引数と戻り値に型を指定
def greet(name: str) -> str:
    return f"こんにちは、{name}さん！"

# None を返す関数
def log(message: str) -> None:
    print(f"[LOG] {message}")

# 複数の返り値
def divide(a: int, b: int) -> tuple[int, int]:
    return a // b, a % b
```

---

## コレクション型

### Python 3.9+ のビルトイン型

```python
# ✅ Python 3.9+: ビルトイン型を直接使う
names: list[str] = ["太郎", "花子"]
scores: dict[str, int] = {"数学": 85, "英語": 92}
point: tuple[float, float] = (10.0, 20.0)
tags: set[str] = {"python", "programming"}

# ❌ 古い書き方（Python 3.8 以前）
from typing import List, Dict, Tuple, Set
names: List[str] = ["太郎", "花子"]  # 不要
```

### ネストした型

```python
# 辞書のリスト
users: list[dict[str, str | int]] = [
    {"name": "太郎", "age": 25},
    {"name": "花子", "age": 30},
]

# リストのリスト（行列）
matrix: list[list[int]] = [
    [1, 2, 3],
    [4, 5, 6],
]
```

---

## Optional と Union

### Union型（Python 3.10+ の `|` 記法）

```python
# ✅ Python 3.10+: パイプで Union を表現
def parse_id(value: str | int) -> int:
    if isinstance(value, str):
        return int(value)
    return value

# ❌ 古い書き方
from typing import Union
def parse_id(value: Union[str, int]) -> int: ...
```

### Optional（None を許容）

```python
# str | None = Optional[str]
def find_user(user_id: int) -> dict[str, str] | None:
    users = {1: {"name": "太郎"}, 2: {"name": "花子"}}
    return users.get(user_id)

# 呼び出し側: None チェックが必要
user = find_user(1)
if user is not None:
    print(user["name"])  # 安全にアクセス
```

---

## TypeAlias と型エイリアス

### type文（Python 3.12+）

```python
# ✅ Python 3.12+: type 文で型エイリアスを定義
type UserId = int
type JsonDict = dict[str, str | int | float | bool | None]
type Matrix = list[list[float]]

def get_user(user_id: UserId) -> JsonDict:
    return {"id": user_id, "name": "太郎"}

# 複雑な型を読みやすく
type Callback = Callable[[str, int], bool]
type EventHandler = dict[str, list[Callback]]
```

---

## ジェネリクス

### 基本的なジェネリック関数（Python 3.12+）

```python
# ✅ Python 3.12+: 角括弧で型パラメータを定義
def first[T](items: list[T]) -> T | None:
    """リストの最初の要素を返す"""
    return items[0] if items else None

# 型が自動推論される
name = first(["太郎", "花子"])   # str | None
number = first([1, 2, 3])        # int | None

def pair[T, U](a: T, b: U) -> tuple[T, U]:
    """2つの値をペアにする"""
    return (a, b)

result = pair("name", 42)  # tuple[str, int]
```

### ジェネリッククラス

```python
class Stack[T]:
    """型安全なスタック"""

    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        if not self._items:
            raise IndexError("スタックが空です")
        return self._items.pop()

    def peek(self) -> T:
        if not self._items:
            raise IndexError("スタックが空です")
        return self._items[-1]

    def is_empty(self) -> bool:
        return len(self._items) == 0

# 使用例
int_stack = Stack[int]()
int_stack.push(1)
int_stack.push(2)
print(int_stack.pop())  # 2

str_stack = Stack[str]()
str_stack.push("hello")
# str_stack.push(42)  # 型チェッカーが警告！
```

---

## dataclass

### 基本的な dataclass

```python
from dataclasses import dataclass

@dataclass
class User:
    name: str
    age: int
    email: str

# __init__, __repr__, __eq__ が自動生成される
user = User(name="太郎", age=25, email="taro@example.com")
print(user)       # User(name='太郎', age=25, email='taro@example.com')
print(user.name)  # 太郎

# 等価比較も自動
user2 = User(name="太郎", age=25, email="taro@example.com")
print(user == user2)  # True
```

### デフォルト値と field()

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class Article:
    title: str
    author: str
    content: str = ""
    tags: list[str] = field(default_factory=list)  # ミュータブルはfieldで
    created_at: datetime = field(default_factory=datetime.now)
    view_count: int = 0

article = Article(title="Python入門", author="太郎")
article.tags.append("python")
print(article)
```

> **重要**: ミュータブルなデフォルト値（list, dict等）は `field(default_factory=...)` を使います。`tags: list[str] = []` は ❌ エラーになります。

### frozen（イミュータブル）

```python
@dataclass(frozen=True)
class Point:
    x: float
    y: float

p = Point(1.0, 2.0)
# p.x = 3.0  # ❌ FrozenInstanceError!

# frozen にすると辞書のキーや集合の要素にできる
points = {Point(0, 0), Point(1, 1)}
```

### __post_init__ でバリデーション

```python
@dataclass
class Temperature:
    celsius: float

    def __post_init__(self):
        if self.celsius < -273.15:
            raise ValueError(
                f"温度は絶対零度（-273.15°C）以上でなければなりません: {self.celsius}"
            )

    @property
    def fahrenheit(self) -> float:
        return self.celsius * 9 / 5 + 32

temp = Temperature(100)
print(temp.fahrenheit)  # 212.0
# Temperature(-300)  # ValueError!
```

### slots=True でメモリ最適化

```python
@dataclass(slots=True)
class Particle:
    x: float
    y: float
    velocity: float

# slots=True により:
# - メモリ使用量が削減される
# - 属性アクセスが高速化される
# - 存在しない属性への代入が禁止される
p = Particle(0.0, 0.0, 1.0)
# p.color = "red"  # ❌ AttributeError!（slots により禁止）
```

### 比較表

| 特徴 | 通常のクラス | dataclass | NamedTuple | TypedDict |
|------|-------------|-----------|------------|-----------|
| 変更可能 | ✅ | ✅（frozen=Falseなら） | ❌ | ✅ |
| 型安全 | 手動 | ✅ | ✅ | ✅ |
| 自動__init__ | ❌ | ✅ | ✅ | ❌ |
| 自動__eq__ | ❌ | ✅ | ✅ | ❌ |
| 辞書のキー | 手動 | frozen=True | ✅ | ❌ |
| 主な用途 | ロジック | データ保持 | 定数データ | JSON型定義 |

---

## TypedDict

### JSONレスポンスの型定義

```python
from typing import TypedDict, NotRequired

class UserResponse(TypedDict):
    id: int
    name: str
    email: str
    age: NotRequired[int]  # オプショナルなキー

# 正しい使い方
user: UserResponse = {
    "id": 1,
    "name": "太郎",
    "email": "taro@example.com",
}

# 型チェッカーが構造を検証
print(user["name"])  # 補完が効く
```

### APIレスポンスの型定義

```python
class PaginationMeta(TypedDict):
    total: int
    page: int
    per_page: int

class ApiResponse(TypedDict):
    data: list[UserResponse]
    meta: PaginationMeta

def fetch_users() -> ApiResponse:
    return {
        "data": [
            {"id": 1, "name": "太郎", "email": "taro@example.com"},
        ],
        "meta": {"total": 1, "page": 1, "per_page": 20},
    }
```

---

## Protocol（構造的部分型）

### Protocolとは

Protocolは「このメソッドを持っていればOK」という型の定義方法です。GoのインターフェースやTypeScriptの構造的型付けに似ています。

```python
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> str: ...

class Circle:
    def __init__(self, radius: float):
        self.radius = radius

    def draw(self) -> str:
        return f"Circle(r={self.radius})"

class Square:
    def __init__(self, side: float):
        self.side = side

    def draw(self) -> str:
        return f"Square(s={self.side})"

# Drawable を明示的に継承していなくても、draw() を持っていれば OK
def render(shape: Drawable) -> None:
    print(shape.draw())

render(Circle(5))   # ✅ Circle(r=5)
render(Square(3))   # ✅ Square(s=3)
```

### 実践的なProtocol

```python
from typing import Protocol

class Repository[T](Protocol):
    def get(self, id: int) -> T | None: ...
    def save(self, item: T) -> None: ...
    def delete(self, id: int) -> bool: ...

# このプロトコルに合致するクラスなら何でも使える
class InMemoryUserRepo:
    def __init__(self):
        self._data: dict[int, dict] = {}

    def get(self, id: int) -> dict | None:
        return self._data.get(id)

    def save(self, item: dict) -> None:
        self._data[item["id"]] = item

    def delete(self, id: int) -> bool:
        return self._data.pop(id, None) is not None
```

---

## 実践例

### 例1: 型安全な設定管理

```python
from dataclasses import dataclass, field

@dataclass(frozen=True)
class DatabaseConfig:
    host: str = "localhost"
    port: int = 5432
    name: str = "mydb"
    user: str = "admin"
    password: str = ""

    @property
    def url(self) -> str:
        return f"postgresql://{self.user}:{self.password}@{self.host}:{self.port}/{self.name}"

@dataclass(frozen=True)
class AppConfig:
    debug: bool = False
    db: DatabaseConfig = field(default_factory=DatabaseConfig)
    allowed_origins: tuple[str, ...] = ("http://localhost:3000",)

config = AppConfig(
    debug=True,
    db=DatabaseConfig(host="db.example.com", password="secret"),
)
print(config.db.url)
# postgresql://admin:secret@db.example.com:5432/mydb
```

### 例2: ジェネリックなリポジトリ

```python
from dataclasses import dataclass, field

@dataclass
class Entity:
    id: int

@dataclass
class User(Entity):
    name: str
    email: str

class InMemoryRepository[T: Entity]:
    """ID で管理するジェネリックなリポジトリ"""

    def __init__(self) -> None:
        self._store: dict[int, T] = {}

    def add(self, item: T) -> None:
        self._store[item.id] = item

    def get(self, id: int) -> T | None:
        return self._store.get(id)

    def all(self) -> list[T]:
        return list(self._store.values())

    def delete(self, id: int) -> bool:
        return self._store.pop(id, None) is not None

# 使用例
repo = InMemoryRepository[User]()
repo.add(User(id=1, name="太郎", email="taro@example.com"))
repo.add(User(id=2, name="花子", email="hanako@example.com"))

user = repo.get(1)
if user:
    print(user.name)  # 太郎（型安全に補完が効く）
```

---

## よくある間違い

### 間違い1: 型ヒントが実行時にチェックされると思う

```python
# ❌ 型ヒントは実行時に強制されない
def greet(name: str) -> str:
    return f"Hello, {name}"

greet(123)  # 実行時エラーにはならない（型チェッカーが警告するだけ）
```

> 型ヒントはあくまで**静的チェックツール用のメタデータ**です。実行時のチェックが必要なら `isinstance()` やバリデーションライブラリを使います。

### 間違い2: Any の乱用

```python
from typing import Any

# ❌ Any は型チェックを無効化する
def process(data: Any) -> Any:
    return data  # 何のチェックもされない

# ✅ 適切な型を定義する
def process(data: str | int | float) -> str:
    return str(data)
```

### 間違い3: 古い書き方を使い続ける

```python
# ❌ Python 3.8 以前の書き方（不要）
from typing import List, Dict, Optional, Union

def old_style(items: List[str]) -> Optional[Dict[str, int]]: ...

# ✅ Python 3.10+ の書き方
def new_style(items: list[str]) -> dict[str, int] | None: ...
```

---

## やってみよう！

1. **型付き関数を書く**: Chapter 05 で作った関数に型ヒントを追加してみましょう。引数と返り値すべてに型をつけてください
2. **データクラスで自己紹介**: 自分の情報（名前、年齢、趣味リスト、住所）を `@dataclass` で定義し、`__str__` で見やすく表示できるようにしましょう
3. **mypy で検証**: `uv add --dev mypy` して、自分のコードに対して `uv run mypy your_file.py` を実行してみましょう。型エラーが出たら修正してみてください

> **型ヒントは「書かなくても動く」ものですが、書くことで「プロのPythonプログラマー」に近づけます。** 最初は面倒に感じるかもしれませんが、慣れると型なしのコードに不安を感じるようになります。

---

## まとめ

この章では、型ヒントとデータクラスを学びました：

- ✅ 型ヒントで IDE 補完、バグ防止、ドキュメントを実現
- ✅ `str | None` でOptional、`str | int` でUnionを表現
- ✅ ジェネリクスで再利用可能な型安全コードを記述
- ✅ dataclass でボイラープレートを削減
- ✅ TypedDict で JSON の型を定義
- ✅ Protocol で柔軟なインターフェースを定義

次の章では、クラスとオブジェクト指向プログラミングを学びます。

---

**次の章**: Chapter 08 - クラスとオブジェクト指向
