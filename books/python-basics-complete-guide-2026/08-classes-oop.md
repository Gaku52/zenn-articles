---
title: "クラスとオブジェクト指向"
---

# Chapter 08: クラスとオブジェクト指向

> オブジェクト指向プログラミングでコードを構造化する

## この章で学べること

- ✅ クラスの定義とインスタンス
- ✅ 特殊メソッド（ダンダーメソッド）
- ✅ 継承と抽象クラス
- ✅ カプセル化と@property
- ✅ コンポジション vs 継承

---

## なぜクラスを学ぶのか

プログラムが大きくなると、関連するデータと処理がバラバラに散らばってしまいます。クラスを使うと、**「関連するデータと処理をひとまとめ」** にできます。

> **たとえるなら**: クラスは「設計図」で、オブジェクトは「設計図から作った製品」です。「犬」という設計図から、「ポチ」「ハチ」「タロウ」という個別の犬（オブジェクト）を作れます。

```
クラス（設計図）: 犬
  - データ: 名前、年齢、犬種
  - 操作: 吠える、お座りする

オブジェクト（実体）:
  - ポチ（柴犬、3歳）
  - ハチ（秋田犬、5歳）
```

> **初学者へ**: オブジェクト指向は「プログラミングの登竜門」と言われ、多くの人がここで悩みます。1回で完璧に理解する必要はありません。まずは「クラス = データと処理のセット」という感覚をつかんでください。

---

## クラスの基本

### class 定義

```python
class Dog:
    """犬を表すクラス"""

    # クラス変数（全インスタンスで共有）
    species: str = "Canis familiaris"

    def __init__(self, name: str, age: int) -> None:
        """コンストラクタ"""
        # インスタンス変数（各インスタンス固有）
        self.name = name
        self.age = age

    def bark(self) -> str:
        """吠える"""
        return f"{self.name}: ワン！"

    def describe(self) -> str:
        """自己紹介"""
        return f"{self.name}（{self.age}歳）"

# インスタンスの作成
dog1 = Dog("ポチ", 3)
dog2 = Dog("タロウ", 5)

print(dog1.bark())      # ポチ: ワン！
print(dog2.describe())  # タロウ（5歳）
print(Dog.species)      # Canis familiaris
```

### self の意味

```python
class Counter:
    def __init__(self) -> None:
        self.count = 0  # self = このインスタンス自身

    def increment(self) -> None:
        self.count += 1  # このインスタンスの count を増やす

    def get(self) -> int:
        return self.count

c1 = Counter()
c2 = Counter()
c1.increment()
c1.increment()
print(c1.get())  # 2
print(c2.get())  # 0（c2 は別のインスタンス）
```

---

## 特殊メソッド（ダンダーメソッド）

### __str__ と __repr__

```python
class Product:
    def __init__(self, name: str, price: int) -> None:
        self.name = name
        self.price = price

    def __str__(self) -> str:
        """人間向けの文字列表現（print()で使われる）"""
        return f"{self.name}: {self.price:,}円"

    def __repr__(self) -> str:
        """開発者向けの文字列表現（デバッグ用）"""
        return f"Product(name={self.name!r}, price={self.price})"

p = Product("Python本", 3000)
print(p)        # Python本: 3,000円（__str__）
print(repr(p))  # Product(name='Python本', price=3000)（__repr__）
```

### __eq__ と比較演算子

```python
from functools import total_ordering

@total_ordering  # __eq__ と __lt__ から他の比較演算子を自動生成
class Version:
    def __init__(self, major: int, minor: int, patch: int) -> None:
        self.major = major
        self.minor = minor
        self.patch = patch

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Version):
            return NotImplemented
        return (self.major, self.minor, self.patch) == (other.major, other.minor, other.patch)

    def __lt__(self, other: "Version") -> bool:
        return (self.major, self.minor, self.patch) < (other.major, other.minor, other.patch)

    def __repr__(self) -> str:
        return f"{self.major}.{self.minor}.{self.patch}"

v1 = Version(3, 14, 0)
v2 = Version(3, 14, 2)
print(v1 < v2)   # True
print(v1 >= v2)  # False（@total_ordering が自動生成）
```

### __len__ と __getitem__

```python
class Playlist:
    def __init__(self, name: str) -> None:
        self.name = name
        self._songs: list[str] = []

    def add(self, song: str) -> None:
        self._songs.append(song)

    def __len__(self) -> int:
        """len() で曲数を取得"""
        return len(self._songs)

    def __getitem__(self, index: int) -> str:
        """インデックスアクセスを可能にする"""
        return self._songs[index]

    def __contains__(self, song: str) -> bool:
        """in 演算子を可能にする"""
        return song in self._songs

playlist = Playlist("お気に入り")
playlist.add("Song A")
playlist.add("Song B")

print(len(playlist))          # 2
print(playlist[0])            # Song A
print("Song A" in playlist)   # True

# for ループも使える（__getitem__ があるので）
for song in playlist:
    print(song)
```

### コンテキストマネージャ

```python
class Timer:
    """処理時間を計測するコンテキストマネージャ"""

    def __enter__(self):
        import time
        self._start = time.perf_counter()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        import time
        self.elapsed = time.perf_counter() - self._start
        print(f"実行時間: {self.elapsed:.4f}秒")
        return False  # 例外を伝播させる

# with 文で使う
with Timer():
    total = sum(range(1_000_000))
# 実行時間: 0.0234秒
```

---

## 継承

### 基本的な継承

```python
class Animal:
    def __init__(self, name: str) -> None:
        self.name = name

    def speak(self) -> str:
        raise NotImplementedError("サブクラスで実装してください")

class Dog(Animal):
    def speak(self) -> str:
        return f"{self.name}: ワン！"

class Cat(Animal):
    def speak(self) -> str:
        return f"{self.name}: ニャー！"

# 多態性（ポリモーフィズム）
animals: list[Animal] = [Dog("ポチ"), Cat("タマ"), Dog("ハチ")]
for animal in animals:
    print(animal.speak())
# ポチ: ワン！
# タマ: ニャー！
# ハチ: ワン！
```

### super() の使い方

```python
class Employee:
    def __init__(self, name: str, salary: int) -> None:
        self.name = name
        self.salary = salary

    def describe(self) -> str:
        return f"{self.name}（年収: {self.salary:,}円）"

class Manager(Employee):
    def __init__(self, name: str, salary: int, team_size: int) -> None:
        super().__init__(name, salary)  # 親クラスの __init__ を呼ぶ
        self.team_size = team_size

    def describe(self) -> str:
        base = super().describe()  # 親クラスのメソッドを呼ぶ
        return f"{base} チーム: {self.team_size}名"

mgr = Manager("太郎", 8_000_000, 5)
print(mgr.describe())
# 太郎（年収: 8,000,000円） チーム: 5名
```

---

## 抽象クラス

```python
from abc import ABC, abstractmethod
import math

class Shape(ABC):
    """図形の抽象クラス"""

    @abstractmethod
    def area(self) -> float:
        """面積を計算する"""
        ...

    @abstractmethod
    def perimeter(self) -> float:
        """周囲長を計算する"""
        ...

    def describe(self) -> str:
        return f"面積: {self.area():.2f}, 周囲長: {self.perimeter():.2f}"

# shape = Shape()  # ❌ TypeError: 抽象クラスはインスタンス化できない

class Circle(Shape):
    def __init__(self, radius: float) -> None:
        self.radius = radius

    def area(self) -> float:
        return math.pi * self.radius ** 2

    def perimeter(self) -> float:
        return 2 * math.pi * self.radius

class Rectangle(Shape):
    def __init__(self, width: float, height: float) -> None:
        self.width = width
        self.height = height

    def area(self) -> float:
        return self.width * self.height

    def perimeter(self) -> float:
        return 2 * (self.width + self.height)

shapes: list[Shape] = [Circle(5), Rectangle(3, 4)]
for shape in shapes:
    print(shape.describe())
# 面積: 78.54, 周囲長: 31.42
# 面積: 12.00, 周囲長: 14.00
```

---

## カプセル化

### アクセス制御の慣習

```python
class BankAccount:
    def __init__(self, owner: str, balance: int = 0) -> None:
        self.owner = owner         # パブリック
        self._balance = balance    # プライベート慣習（外部からアクセスしないで）
        self.__pin = "0000"        # 名前マングリング（強い制限）

    @property
    def balance(self) -> int:
        """残高を取得（読み取り専用）"""
        return self._balance

    def deposit(self, amount: int) -> None:
        """入金"""
        if amount <= 0:
            raise ValueError("入金額は正の数でなければなりません")
        self._balance += amount

    def withdraw(self, amount: int) -> None:
        """出金"""
        if amount <= 0:
            raise ValueError("出金額は正の数でなければなりません")
        if amount > self._balance:
            raise ValueError("残高不足です")
        self._balance -= amount

account = BankAccount("太郎", 10000)
account.deposit(5000)
print(account.balance)  # 15000

# account._balance = 999999  # 技術的にはアクセス可能だが、慣習上NG
# account.__pin              # ❌ AttributeError（名前マングリング）
```

### @property（getter/setter）

```python
class Temperature:
    def __init__(self, celsius: float) -> None:
        self.celsius = celsius  # setter が呼ばれる

    @property
    def celsius(self) -> float:
        return self._celsius

    @celsius.setter
    def celsius(self, value: float) -> None:
        if value < -273.15:
            raise ValueError("絶対零度以下にはできません")
        self._celsius = value

    @property
    def fahrenheit(self) -> float:
        """華氏（読み取り専用）"""
        return self._celsius * 9 / 5 + 32

temp = Temperature(100)
print(temp.celsius)     # 100
print(temp.fahrenheit)  # 212.0
temp.celsius = 0        # setter でバリデーション
# temp.celsius = -300   # ❌ ValueError!
```

---

## コンポジション vs 継承

### ✅ コンポジション（推奨）

```python
class Engine:
    def __init__(self, horsepower: int) -> None:
        self.horsepower = horsepower

    def start(self) -> str:
        return f"エンジン始動（{self.horsepower}馬力）"

class GPS:
    def navigate(self, destination: str) -> str:
        return f"{destination}までナビゲーション開始"

class Car:
    """コンポジション: 車はエンジンとGPSを「持つ」"""
    def __init__(self, name: str, engine: Engine, gps: GPS) -> None:
        self.name = name
        self.engine = engine  # has-a 関係
        self.gps = gps        # has-a 関係

    def drive(self, destination: str) -> None:
        print(self.engine.start())
        print(self.gps.navigate(destination))

car = Car("プリウス", Engine(150), GPS())
car.drive("東京タワー")
```

### ❌ 過度な継承（非推奨）

```python
# ❌ 不自然な継承チェーン
class Vehicle:
    pass

class Car(Vehicle):
    pass

class ElectricCar(Car):
    pass

class SelfDrivingElectricCar(ElectricCar):
    pass

# ✅ コンポジションで機能を組み合わせる方が柔軟
class Car:
    def __init__(
        self,
        engine: Engine,
        autopilot: AutoPilot | None = None,
    ) -> None:
        self.engine = engine
        self.autopilot = autopilot
```

---

## クラスメソッドとスタティックメソッド

```python
class Date:
    def __init__(self, year: int, month: int, day: int) -> None:
        self.year = year
        self.month = month
        self.day = day

    def __repr__(self) -> str:
        return f"{self.year}-{self.month:02d}-{self.day:02d}"

    @classmethod
    def from_string(cls, date_str: str) -> "Date":
        """文字列からDateを作成するファクトリメソッド"""
        year, month, day = map(int, date_str.split("-"))
        return cls(year, month, day)

    @classmethod
    def today(cls) -> "Date":
        """今日の日付を返すファクトリメソッド"""
        from datetime import date
        d = date.today()
        return cls(d.year, d.month, d.day)

    @staticmethod
    def is_leap_year(year: int) -> bool:
        """うるう年判定（インスタンスに依存しない）"""
        return year % 4 == 0 and (year % 100 != 0 or year % 400 == 0)

# 使い方
d1 = Date(2026, 2, 9)
d2 = Date.from_string("2026-12-25")  # ファクトリメソッド
d3 = Date.today()                     # ファクトリメソッド
print(Date.is_leap_year(2024))        # True（スタティックメソッド）
```

---

## 実践例

### 例1: 銀行口座システム

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class Transaction:
    """取引履歴"""
    type: str  # "deposit" or "withdrawal"
    amount: int
    timestamp: datetime = field(default_factory=datetime.now)

class BankAccount:
    def __init__(self, owner: str, balance: int = 0) -> None:
        self.owner = owner
        self._balance = balance
        self._transactions: list[Transaction] = []

    @property
    def balance(self) -> int:
        return self._balance

    def deposit(self, amount: int) -> None:
        if amount <= 0:
            raise ValueError("入金額は正の数にしてください")
        self._balance += amount
        self._transactions.append(Transaction("deposit", amount))

    def withdraw(self, amount: int) -> None:
        if amount <= 0:
            raise ValueError("出金額は正の数にしてください")
        if amount > self._balance:
            raise ValueError(f"残高不足（残高: {self._balance:,}円）")
        self._balance -= amount
        self._transactions.append(Transaction("withdrawal", amount))

    def statement(self) -> str:
        lines = [f"=== {self.owner}の口座明細 ==="]
        for t in self._transactions:
            sign = "+" if t.type == "deposit" else "-"
            lines.append(f"  {sign}{t.amount:,}円")
        lines.append(f"  残高: {self._balance:,}円")
        return "\n".join(lines)

account = BankAccount("太郎", 100_000)
account.deposit(50_000)
account.withdraw(30_000)
print(account.statement())
```

### 例2: 図形の面積計算

```python
from abc import ABC, abstractmethod
import math

class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

    @abstractmethod
    def __str__(self) -> str: ...

class Circle(Shape):
    def __init__(self, radius: float) -> None:
        self.radius = radius

    def area(self) -> float:
        return math.pi * self.radius ** 2

    def __str__(self) -> str:
        return f"円（半径={self.radius}）"

class Rectangle(Shape):
    def __init__(self, width: float, height: float) -> None:
        self.width = width
        self.height = height

    def area(self) -> float:
        return self.width * self.height

    def __str__(self) -> str:
        return f"長方形（{self.width}×{self.height}）"

class Triangle(Shape):
    def __init__(self, base: float, height: float) -> None:
        self.base = base
        self.height = height

    def area(self) -> float:
        return self.base * self.height / 2

    def __str__(self) -> str:
        return f"三角形（底辺={self.base}, 高さ={self.height}）"

# 多態性の活用
shapes: list[Shape] = [Circle(5), Rectangle(3, 4), Triangle(6, 8)]
total_area = sum(s.area() for s in shapes)

for shape in shapes:
    print(f"{shape}: 面積 = {shape.area():.2f}")
print(f"合計面積: {total_area:.2f}")
```

---

## よくある間違い

### 間違い1: クラス変数とインスタンス変数の混同

```python
# ❌ ミュータブルなクラス変数は全インスタンスで共有される
class BadStudent:
    grades: list[int] = []  # クラス変数（全インスタンスで共有！）

s1 = BadStudent()
s2 = BadStudent()
s1.grades.append(90)
print(s2.grades)  # [90] ← s2 にも影響！

# ✅ __init__ でインスタンス変数として初期化
class GoodStudent:
    def __init__(self) -> None:
        self.grades: list[int] = []  # インスタンス変数（各自固有）

s1 = GoodStudent()
s2 = GoodStudent()
s1.grades.append(90)
print(s2.grades)  # []（影響なし）
```

### 間違い2: super() の呼び忘れ

```python
# ❌ super().__init__() を忘れる
class Child(Parent):
    def __init__(self, extra: str) -> None:
        # super().__init__() を忘れている！
        self.extra = extra

# ✅ 必ず super() を呼ぶ
class Child(Parent):
    def __init__(self, extra: str) -> None:
        super().__init__()
        self.extra = extra
```

### 間違い3: 不要なクラスの作成

```python
# ❌ 関数で十分なのにクラスにする
class Greeter:
    def greet(self, name: str) -> str:
        return f"Hello, {name}"

g = Greeter()
g.greet("太郎")

# ✅ 関数で十分
def greet(name: str) -> str:
    return f"Hello, {name}"

greet("太郎")
```

---

## やってみよう！

1. **図書管理クラス**: `Book` クラス（タイトル、著者、ページ数、読了フラグ）を作り、`Library` クラスで複数の本を管理してみましょう。本の追加、検索、読了率の計算ができるようにしてください
2. **特殊メソッドの活用**: `Money` クラスを作り、`+` で足し算、`str()` で `"¥1,234"` 形式の表示ができるようにしてみましょう
3. **継承 vs コンポジション**: 「犬」と「猫」を表すクラスを、(a) 継承で作るパターン、(b) コンポジションで作るパターンの両方で書いてみて、違いを比べてみましょう

> **オブジェクト指向は「理解する」より「使って慣れる」方が効果的です。** 完璧に理解してから進もうとすると止まってしまうので、8割理解できたら次に進みましょう。実際のコードを書く中で、理解が深まっていきます。

---

## まとめ

この章では、クラスとオブジェクト指向を学びました：

- ✅ `class` でデータとメソッドをまとめる
- ✅ 特殊メソッドでPythonの組み込み機能と統合
- ✅ 継承で共通機能を共有、抽象クラスで契約を定義
- ✅ `@property` でバリデーション付きアクセスを実現
- ✅ コンポジション > 継承（柔軟性が高い）

次の章では、モジュールとパッケージでコードを整理する方法を学びます。

---

**次の章**: Chapter 09 - モジュール・パッケージ・依存管理
