---
title: "関数とスコープ"
---

# Chapter 05: 関数とスコープ

> 再利用可能なコードの単位を作り、プログラムを整理する

## この章で学べること

- ✅ 関数の定義と呼び出し
- ✅ 引数の種類（位置引数、キーワード引数、*args、**kwargs）
- ✅ スコープとLEGBルール
- ✅ lambda式
- ✅ デコレータ入門

---

## なぜ関数が大切なのか

ここまでのプログラムでは、同じ処理を何度も書いていたかもしれません。関数を使うと、**一度書いた処理に名前をつけて、何度でも呼び出せる**ようになります。

> **たとえるなら**: 関数は「レシピ」のようなもの。「カレーの作り方」というレシピを一度書けば、何度でもカレーを作れます。材料（引数）を変えれば、ビーフカレーもチキンカレーも作れます。

関数を使いこなせるようになると、コードが劇的に整理され、バグも減ります。**この章は特に重要**なので、じっくり読んでください。

---

## 関数の基本

### 定義と呼び出し

```python
# 基本的な関数定義
def greet(name: str) -> str:
    """指定された名前で挨拶メッセージを返す"""
    return f"こんにちは、{name}さん！"

# 呼び出し
message = greet("太郎")
print(message)  # こんにちは、太郎さん！
```

### 複数の値を返す

```python
def divide(a: int, b: int) -> tuple[int, int]:
    """商と余りを返す"""
    quotient = a // b
    remainder = a % b
    return quotient, remainder  # タプルとして返す

# アンパックで受け取る
q, r = divide(17, 5)
print(f"商: {q}, 余り: {r}")  # 商: 3, 余り: 2
```

### None を返す関数

```python
def print_header(title: str) -> None:
    """ヘッダーを表示する（返り値なし）"""
    print("=" * 40)
    print(f"  {title}")
    print("=" * 40)

# return 文がない関数は暗黙的に None を返す
result = print_header("メニュー")
print(result)  # None
```

### docstring の書き方

```python
def calculate_bmi(weight: float, height: float) -> float:
    """BMIを計算する。

    Args:
        weight: 体重（kg）
        height: 身長（m）

    Returns:
        BMI値（小数点以下1桁）

    Raises:
        ValueError: height が 0 以下の場合

    Examples:
        >>> calculate_bmi(70, 1.75)
        22.9
    """
    if height <= 0:
        raise ValueError("身長は正の数でなければなりません")
    return round(weight / (height ** 2), 1)
```

---

## 引数の種類

### 位置引数とキーワード引数

```python
def create_user(name: str, age: int, city: str) -> dict:
    return {"name": name, "age": age, "city": city}

# 位置引数（順番で渡す）
user1 = create_user("太郎", 25, "東京")

# キーワード引数（名前で渡す、順番は自由）
user2 = create_user(city="大阪", name="花子", age=30)

# 混在（位置引数を先に）
user3 = create_user("次郎", age=20, city="名古屋")
```

### デフォルト引数

```python
def greet(name: str, greeting: str = "こんにちは") -> str:
    return f"{greeting}、{name}さん！"

greet("太郎")                    # こんにちは、太郎さん！
greet("太郎", "おはよう")         # おはよう、太郎さん！
greet("太郎", greeting="さようなら")  # さようなら、太郎さん！
```

### ⚠️ ミュータブルデフォルトの罠（最重要！）

```python
# ❌ 危険: リストがデフォルト引数
def add_item(item: str, items: list[str] = []) -> list[str]:
    items.append(item)
    return items

print(add_item("a"))  # ['a']
print(add_item("b"))  # ['a', 'b'] ← 前回の結果が残る！
print(add_item("c"))  # ['a', 'b', 'c'] ← さらに蓄積！

# ✅ 安全: None をデフォルトにして関数内で初期化
def add_item(item: str, items: list[str] | None = None) -> list[str]:
    if items is None:
        items = []
    items.append(item)
    return items

print(add_item("a"))  # ['a']
print(add_item("b"))  # ['b'] ← 正しい！
```

> **理由**: デフォルト引数は**関数定義時に1回だけ**評価されます。ミュータブルなオブジェクトは全呼び出しで共有されてしまいます。

### 可変長引数 *args と **kwargs

```python
# *args: 可変長の位置引数（タプルとして受け取る）
def sum_all(*args: int) -> int:
    return sum(args)

sum_all(1, 2, 3)       # 6
sum_all(1, 2, 3, 4, 5)  # 15

# **kwargs: 可変長のキーワード引数（辞書として受け取る）
def create_tag(tag: str, **kwargs: str) -> str:
    attrs = " ".join(f'{k}="{v}"' for k, v in kwargs.items())
    return f"<{tag} {attrs}>"

create_tag("div", id="main", class_="container")
# <div id="main" class_="container">

# 組み合わせ
def flexible(a: int, b: int, *args: int, **kwargs: str) -> None:
    print(f"a={a}, b={b}")
    print(f"args={args}")
    print(f"kwargs={kwargs}")

flexible(1, 2, 3, 4, name="太郎", age="25")
# a=1, b=2
# args=(3, 4)
# kwargs={'name': '太郎', 'age': '25'}
```

### 位置専用引数 / キーワード専用引数

```python
# / の前は位置専用、* の後はキーワード専用
def example(pos_only: int, /, normal: int, *, kw_only: int) -> None:
    print(f"{pos_only}, {normal}, {kw_only}")

example(1, 2, kw_only=3)       # ✅ OK
example(1, normal=2, kw_only=3)  # ✅ OK
# example(pos_only=1, normal=2, kw_only=3)  # ❌ TypeError
# example(1, 2, 3)                           # ❌ TypeError
```

---

## スコープとライフタイム

### LEGBルール

Pythonは変数を以下の順序で探します：

```
L (Local)     → 関数内のローカル変数
E (Enclosing) → 外側の関数のローカル変数
G (Global)    → モジュールレベルのグローバル変数
B (Built-in)  → 組み込み名前空間（print, len など）
```

```python
x = "global"  # G: グローバル

def outer():
    x = "enclosing"  # E: 外側の関数

    def inner():
        x = "local"  # L: ローカル
        print(x)      # "local"（Lが最優先）

    inner()
    print(x)  # "enclosing"

outer()
print(x)  # "global"
```

### global と nonlocal

```python
count = 0

def increment():
    global count  # グローバル変数を変更する（⚠️ 非推奨）
    count += 1

increment()
print(count)  # 1

# ✅ 推奨: 引数と戻り値で状態を管理
def increment(count: int) -> int:
    return count + 1

count = 0
count = increment(count)  # 明示的で安全
```

```python
def counter():
    count = 0

    def increment() -> int:
        nonlocal count  # 外側の関数の変数を変更
        count += 1
        return count

    return increment

my_counter = counter()
print(my_counter())  # 1
print(my_counter())  # 2
print(my_counter())  # 3
```

---

## lambda式

### 基本構文

```python
# 通常の関数
def square(x: int) -> int:
    return x ** 2

# lambda式（無名関数）
square = lambda x: x ** 2

print(square(5))  # 25
```

### 実践的な活用

```python
# sorted() の key 引数
students = [
    {"name": "太郎", "score": 85},
    {"name": "花子", "score": 92},
    {"name": "次郎", "score": 78},
]

# スコア順でソート
by_score = sorted(students, key=lambda s: s["score"], reverse=True)
# [花子(92), 太郎(85), 次郎(78)]

# 名前順でソート
by_name = sorted(students, key=lambda s: s["name"])

# max() / min() の key 引数
top = max(students, key=lambda s: s["score"])
print(top["name"])  # 花子
```

### lambda vs def の使い分け

```python
# ✅ lambda が適切: 短い、1回だけ使う、key 引数
sorted(items, key=lambda x: x.name)

# ❌ lambda が不適切: 複雑なロジック
# bad = lambda x: x if x > 0 else -x if x < -10 else 0

# ✅ 複雑なら def で定義
def normalize(x: int) -> int:
    if x > 0:
        return x
    elif x < -10:
        return -x
    else:
        return 0
```

---

## デコレータ入門

### デコレータとは

デコレータは「関数を受け取って、機能を追加した新しい関数を返す関数」です。

```python
import time
import functools

def timer(func):
    """関数の実行時間を計測するデコレータ"""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        end = time.perf_counter()
        print(f"{func.__name__}: {end - start:.4f}秒")
        return result
    return wrapper

@timer
def slow_function():
    """重い処理のシミュレーション"""
    time.sleep(1)
    return "完了"

result = slow_function()
# slow_function: 1.0012秒
```

### @functools.wraps の重要性

```python
# ❌ wraps なし: 元の関数情報が失われる
def bad_decorator(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@bad_decorator
def my_func():
    """私の関数"""
    pass

print(my_func.__name__)  # "wrapper" ← 元の名前が失われた
print(my_func.__doc__)   # None

# ✅ wraps あり: 元の関数情報を保持
def good_decorator(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@good_decorator
def my_func():
    """私の関数"""
    pass

print(my_func.__name__)  # "my_func" ← 保持されている
print(my_func.__doc__)   # "私の関数"
```

### 組み込みデコレータ

```python
class Circle:
    def __init__(self, radius: float):
        self._radius = radius

    @property
    def radius(self) -> float:
        """半径を取得"""
        return self._radius

    @radius.setter
    def radius(self, value: float) -> None:
        """半径を設定（バリデーション付き）"""
        if value <= 0:
            raise ValueError("半径は正の数でなければなりません")
        self._radius = value

    @property
    def area(self) -> float:
        """面積を計算（読み取り専用）"""
        import math
        return math.pi * self._radius ** 2

c = Circle(5)
print(c.radius)  # 5（属性のようにアクセス）
print(c.area)    # 78.539...
c.radius = 10    # setter が呼ばれる
# c.radius = -1  # ValueError!
```

---

## 高階関数

### 関数を引数として渡す

```python
def apply_operation(numbers: list[int], operation) -> list[int]:
    """各要素に操作を適用する"""
    return [operation(n) for n in numbers]

numbers = [1, 2, 3, 4, 5]

doubled = apply_operation(numbers, lambda x: x * 2)
# [2, 4, 6, 8, 10]

squared = apply_operation(numbers, lambda x: x ** 2)
# [1, 4, 9, 16, 25]
```

### map(), filter()

```python
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# map: 各要素に関数を適用
squares = list(map(lambda x: x ** 2, numbers))

# filter: 条件に合う要素を抽出
evens = list(filter(lambda x: x % 2 == 0, numbers))

# ✅ 内包表記の方が Pythonic（推奨）
squares = [x ** 2 for x in numbers]
evens = [x for x in numbers if x % 2 == 0]
```

---

## 実践例

### 例1: バリデーション関数群

```python
import re

def validate_email(email: str) -> bool:
    """メールアドレスの簡易バリデーション"""
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return bool(re.match(pattern, email))

def validate_password(password: str) -> list[str]:
    """パスワードの強度チェック。問題点のリストを返す"""
    errors: list[str] = []
    if len(password) < 8:
        errors.append("8文字以上にしてください")
    if not re.search(r'[A-Z]', password):
        errors.append("大文字を含めてください")
    if not re.search(r'[a-z]', password):
        errors.append("小文字を含めてください")
    if not re.search(r'[0-9]', password):
        errors.append("数字を含めてください")
    return errors

# 使用例
print(validate_email("user@example.com"))  # True
print(validate_email("invalid-email"))     # False

errors = validate_password("abc")
print(errors)
# ['8文字以上にしてください', '大文字を含めてください', '数字を含めてください']

errors = validate_password("Passw0rd123")
print(errors)  # []（問題なし）
```

### 例2: 簡易キャッシュデコレータ

```python
import functools

def cache(func):
    """結果をキャッシュするデコレータ"""
    memo: dict = {}

    @functools.wraps(func)
    def wrapper(*args):
        if args not in memo:
            memo[args] = func(*args)
            print(f"  計算: {func.__name__}{args}")
        else:
            print(f"  キャッシュ: {func.__name__}{args}")
        return memo[args]

    return wrapper

@cache
def fibonacci(n: int) -> int:
    """フィボナッチ数を計算"""
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

print(fibonacci(10))  # 55（最初は計算、2回目以降はキャッシュ）
print(fibonacci(10))  # 55（すべてキャッシュから）
```

> **補足**: 実務では `@functools.lru_cache` を使えば同じことが1行でできます。

---

## よくある間違い

### 間違い1: ミュータブルデフォルト引数（再掲、最重要）

```python
# ❌ この書き方は絶対に避ける
def add_tag(tag: str, tags: list[str] = []) -> list[str]:
    tags.append(tag)
    return tags

# ✅ None パターンを使う
def add_tag(tag: str, tags: list[str] | None = None) -> list[str]:
    if tags is None:
        tags = []
    tags.append(tag)
    return tags
```

### 間違い2: グローバル変数の乱用

```python
# ❌ グローバル変数で状態管理
total = 0

def add(n: int) -> None:
    global total
    total += n

# ✅ 引数と戻り値で状態管理
def add(total: int, n: int) -> int:
    return total + n

total = 0
total = add(total, 10)
```

### 間違い3: return 忘れ

```python
# ❌ return がない → None が返る
def double(x: int):
    x * 2  # return を忘れている！

result = double(5)
print(result)  # None

# ✅ return を明示
def double(x: int) -> int:
    return x * 2
```

---

## やってみよう！

1. **じゃんけん関数**: `janken(player_hand: str) -> str` を作ってみましょう。コンピュータの手をランダムに決めて、勝敗を返す関数です
2. **文字列カウンター**: 文字列を受け取って、各文字が何回出現するかを辞書で返す関数 `count_chars(text: str) -> dict[str, int]` を作ってみましょう
3. **デコレータに挑戦**: 関数の呼び出し回数をカウントするデコレータ `@call_counter` を作ってみましょう

> **関数が書けるようになると、「プログラムを作る」感覚が一気に変わります。** 小さな関数の組み合わせで、複雑なプログラムを構築できるようになります。

---

## まとめ

この章では、関数とスコープを学びました：

- ✅ def で関数を定義し、型ヒントで安全性を高める
- ✅ 引数の種類: 位置、キーワード、デフォルト、*args、**kwargs
- ✅ ミュータブルデフォルト引数の罠と回避方法
- ✅ LEGB ルールでスコープを理解
- ✅ lambda式は短い処理に、def は複雑な処理に
- ✅ デコレータで関数に機能を追加

次の章では、Pythonの強力なデータ構造を学びます。

---

**次の章**: Chapter 06 - データ構造
