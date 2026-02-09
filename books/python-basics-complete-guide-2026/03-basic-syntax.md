---
title: "基本構文と型"
---

# Chapter 03: 基本構文と型

> Pythonの基本的な書き方とデータ型を理解する

## この章で学べること

- ✅ 変数の定義と命名規則
- ✅ 基本データ型（int, float, str, bool, None）
- ✅ 文字列操作とf-string
- ✅ 数値演算と比較演算子
- ✅ 型変換の方法

---

## なぜ基本構文が大切なのか

基本構文は、プログラミングにおける「文字の読み書き」にあたります。日本語を学ぶときに、まず「ひらがな」を覚えるのと同じです。

ここで学ぶ「変数」「型」「文字列」「演算子」は、この先のすべての章で使う基礎中の基礎です。焦らず、一つひとつ確実に理解していきましょう。

> **コツ**: サンプルコードは「見るだけ」ではなく、必ず自分で入力して実行してください。手を動かすことが最速の学習法です。

---

## 変数と代入

### 変数の定義

変数とは「データに名前をつけて保存する箱」です。Pythonでは、型宣言なしで変数を定義できます。

```python
# 変数の定義（型宣言不要）
name = "Python"
version = 3.14
is_awesome = True

# 型ヒント付き（推奨）
name: str = "Python"
version: float = 3.14
is_awesome: bool = True
```

### 命名規則（PEP 8）

```python
# ✅ 正しい命名（snake_case）
user_name = "太郎"
total_count = 42
is_active = True
MAX_RETRY = 3          # 定数は UPPER_SNAKE_CASE

# ❌ 避けるべき命名
userName = "太郎"      # camelCase（Pythonでは非推奨）
x = "太郎"             # 意味不明な名前
l = [1, 2, 3]          # 数字の1と紛らわしい
```

### 多重代入

```python
# 同時に複数の変数を定義
x, y, z = 1, 2, 3

# 値の交換（他の言語では一時変数が必要）
a, b = 1, 2
a, b = b, a  # a=2, b=1（Pythonならこれだけ！）

# 同じ値を複数の変数に
x = y = z = 0

# アンパック
first, *rest = [1, 2, 3, 4, 5]
# first = 1, rest = [2, 3, 4, 5]
```

---

## 基本データ型

### 型とは何か

プログラミングにおける「型」とは、データの種類のことです。「42」は整数、「"Hello"」は文字列、「True」は真偽値——このように、データにはそれぞれ「種類」があります。

> **たとえるなら**: 型は「容器の形」のようなもの。コップには液体、お皿には固体、箱には荷物——それぞれ入れるものが決まっています。

### 型の一覧

| 型 | 説明 | 例 |
|----|------|-----|
| `int` | 整数（任意精度） | `42`, `-10`, `1_000_000` |
| `float` | 浮動小数点数 | `3.14`, `-0.5`, `1e10` |
| `str` | 文字列 | `"Hello"`, `'Python'` |
| `bool` | 真偽値 | `True`, `False` |
| `None` | 値がないことを表す | `None` |

### int（整数）

```python
# 整数は任意の大きさを扱える（他の言語にはない特徴！）
small = 42
large = 1_000_000_000           # アンダースコアで区切れる
very_large = 10 ** 100          # 10の100乗も OK

# 2進数、8進数、16進数
binary = 0b1010     # 10
octal = 0o17        # 15
hexadecimal = 0xFF  # 255
```

### float（浮動小数点数）

```python
pi = 3.14159
negative = -0.5
scientific = 1.5e10  # 1.5 × 10^10

# ⚠️ 浮動小数点の罠
>>> 0.1 + 0.2
0.30000000000000004  # 正確に 0.3 にならない！

# 正確な計算が必要な場合は Decimal を使う
from decimal import Decimal
>>> Decimal("0.1") + Decimal("0.2")
Decimal('0.3')
```

### str（文字列）

```python
# シングルクォート、ダブルクォート、どちらでも OK
single = 'Hello'
double = "Hello"

# 複数行文字列
multi = """
これは
複数行の
文字列です
"""

# raw文字列（エスケープを無視）
path = r"C:\Users\name\Documents"  # \n がそのまま保持される
```

### bool（真偽値）

```python
is_active = True
is_deleted = False

# bool は int のサブクラス
>>> True + True
2
>>> False + 1
1

# Falsy な値（False と判定されるもの）
# False, 0, 0.0, "", [], {}, set(), None
```

### None

```python
# 値がないことを表す特別な値
result = None

# None の確認は is を使う
if result is None:
    print("値がありません")

# ❌ == で比較しない
if result == None:  # 動くが非推奨
    pass
```

### type() で型を確認

```python
>>> type(42)
<class 'int'>
>>> type(3.14)
<class 'float'>
>>> type("Hello")
<class 'str'>
>>> type(True)
<class 'bool'>
>>> type(None)
<class 'NoneType'>
```

---

## 文字列操作

### f-string（フォーマット文字列）

f-string は Python で最も便利な機能の1つです。文字列の中に変数や計算結果を直接埋め込めます。

```python
name = "太郎"
age = 25

# ✅ f-string（推奨）
message = f"こんにちは、{name}さん！年齢は{age}歳です。"

# 式も埋め込める
print(f"来年は{age + 1}歳です")
print(f"大文字: {name.upper()}")

# フォーマット指定
price = 1234.5
print(f"価格: {price:,.1f}円")   # 価格: 1,234.5円
print(f"割合: {0.856:.1%}")      # 割合: 85.6%
print(f"ゼロ埋め: {42:05d}")     # ゼロ埋め: 00042
```

```python
# ❌ 古い方法（非推奨ではないが、f-stringの方が読みやすい）
message = "こんにちは、{}さん！".format(name)
message = "こんにちは、%sさん！" % name
```

### 文字列メソッド

```python
text = "  Hello, Python World!  "

# 大文字・小文字
text.upper()          # "  HELLO, PYTHON WORLD!  "
text.lower()          # "  hello, python world!  "
text.title()          # "  Hello, Python World!  "

# 空白除去
text.strip()          # "Hello, Python World!"
text.lstrip()         # "Hello, Python World!  "
text.rstrip()         # "  Hello, Python World!"

# 分割・結合
"a,b,c".split(",")    # ["a", "b", "c"]
"-".join(["2026", "02", "09"])  # "2026-02-09"

# 検索・置換
"Hello World".find("World")     # 6（位置）
"Hello World".replace("World", "Python")  # "Hello Python"

# 判定
"123".isdigit()       # True
"abc".isalpha()       # True
"hello".startswith("he")  # True
"hello".endswith("lo")    # True
```

### スライス

```python
text = "Python"

# インデックス（0始まり）
text[0]     # "P"
text[-1]    # "n"（末尾から）

# スライス [start:end]（end は含まない）
text[0:3]   # "Pyt"
text[2:]    # "thon"（2番目以降）
text[:3]    # "Pyt"（3番目まで）
text[::2]   # "Pto"（1つ飛ばし）
text[::-1]  # "nohtyP"（逆順）
```

---

## 数値演算

### 算術演算子

```python
# 基本演算
10 + 3    # 13（加算）
10 - 3    # 7（減算）
10 * 3    # 30（乗算）
10 / 3    # 3.3333...（除算、常に float）
10 // 3   # 3（整数除算、切り捨て）
10 % 3    # 1（剰余）
10 ** 3   # 1000（べき乗）

# 複合代入演算子
count = 0
count += 1   # count = count + 1
count -= 1   # count = count - 1
count *= 2   # count = count * 2
count //= 3  # count = count // 3
```

### 比較演算子と論理演算子

```python
# 比較演算子
10 == 10    # True（等しい）
10 != 5     # True（等しくない）
10 > 5      # True（より大きい）
10 >= 10    # True（以上）
10 < 20     # True（より小さい）
10 <= 10    # True（以下）

# チェーン比較（Pythonの便利な特徴！）
x = 15
1 <= x <= 100   # True（他の言語では 1 <= x && x <= 100）

# 論理演算子
True and False   # False
True or False    # True
not True         # False

# 実践的な例
age = 25
has_ticket = True
can_enter = age >= 18 and has_ticket  # True
```

### `is` vs `==` の違い

```python
# == は「値」の比較
a = [1, 2, 3]
b = [1, 2, 3]
a == b  # True（値が同じ）

# is は「同一オブジェクト」の比較
a is b  # False（別々のオブジェクト）

c = a
a is c  # True（同じオブジェクトを参照）

# ✅ None の比較には is を使う
value = None
if value is None:
    print("None です")
```

### `in` 演算子

```python
# 要素の存在確認
"P" in "Python"          # True
3 in [1, 2, 3, 4, 5]    # True
"key" in {"key": "val"}  # True（辞書はキーを検索）

# 否定
"X" not in "Python"      # True
```

---

## 型変換

### 明示的な型変換

```python
# int() — 整数に変換
int("42")       # 42
int(3.99)       # 3（切り捨て）
int(True)       # 1

# float() — 浮動小数点に変換
float("3.14")   # 3.14
float(42)       # 42.0

# str() — 文字列に変換
str(42)         # "42"
str(3.14)       # "3.14"
str(True)       # "True"

# bool() — 真偽値に変換
bool(1)         # True
bool(0)         # False
bool("")        # False
bool("hello")   # True
bool([])        # False
bool([1])       # True
```

### よくあるエラー

```python
# ❌ 文字列と数値の連結はエラー
>>> "年齢: " + 25
TypeError: can only concatenate str (not "int") to str

# ✅ 解決策1: str() で変換
>>> "年齢: " + str(25)
'年齢: 25'

# ✅ 解決策2: f-string を使う（推奨）
>>> f"年齢: {25}"
'年齢: 25'

# ❌ 数値に変換できない文字列
>>> int("hello")
ValueError: invalid literal for int() with base 10: 'hello'
```

---

## コメントとドキュメント

```python
# 1行コメント（# の後にスペースを1つ入れる）
x = 42  # 変数の説明

# 複数行のコメント
# この処理は〇〇のために
# △△を計算しています

# docstring（関数やクラスの説明）
def calculate_bmi(weight: float, height: float) -> float:
    """BMIを計算する。

    Args:
        weight: 体重（kg）
        height: 身長（m）

    Returns:
        BMI値
    """
    return weight / (height ** 2)
```

---

## 実践例

### 例1: BMI計算機

```python
def main():
    print("=== BMI計算機 ===")

    weight = float(input("体重（kg）: "))
    height = float(input("身長（cm）: "))

    # cmをmに変換
    height_m = height / 100

    # BMI計算
    bmi = weight / (height_m ** 2)

    # 結果表示
    print(f"\nあなたのBMI: {bmi:.1f}")

    if bmi < 18.5:
        print("判定: 低体重")
    elif bmi < 25.0:
        print("判定: 普通体重")
    elif bmi < 30.0:
        print("判定: 肥満（1度）")
    else:
        print("判定: 肥満（2度以上）")

main()
```

### 例2: 温度変換プログラム

```python
def celsius_to_fahrenheit(celsius: float) -> float:
    """摂氏から華氏に変換する"""
    return celsius * 9 / 5 + 32

def fahrenheit_to_celsius(fahrenheit: float) -> float:
    """華氏から摂氏に変換する"""
    return (fahrenheit - 32) * 5 / 9

def main():
    print("=== 温度変換 ===")
    print("[1] 摂氏 → 華氏")
    print("[2] 華氏 → 摂氏")

    choice = input("選択: ").strip()

    if choice == "1":
        c = float(input("摂氏温度: "))
        f = celsius_to_fahrenheit(c)
        print(f"{c}°C = {f:.1f}°F")
    elif choice == "2":
        f = float(input("華氏温度: "))
        c = fahrenheit_to_celsius(f)
        print(f"{f}°F = {c:.1f}°C")
    else:
        print("無効な選択です")

main()
```

---

## よくある間違い

### 間違い1: 予約語を変数名に使う

```python
# ❌ 予約語は変数名に使えない
list = [1, 2, 3]    # 組み込み関数 list() を上書きしてしまう
type = "admin"       # 組み込み関数 type() を上書き

# ✅ 別の名前を使う
items = [1, 2, 3]
user_type = "admin"
```

### 間違い2: `==` と `=` の混同

```python
# ❌ 条件文で = を使う（代入であり比較ではない）
# if x = 10:  # SyntaxError

# ✅ 比較には == を使う
if x == 10:
    print("xは10です")
```

### 間違い3: `is` と `==` の混同

```python
# ❌ リストの比較に is を使う
a = [1, 2, 3]
b = [1, 2, 3]
if a is b:  # False（別のオブジェクト）
    print("同じ")

# ✅ 値の比較には == を使う
if a == b:  # True（値が同じ）
    print("同じ")
```

### 間違い4: 文字列と数値の連結忘れ

```python
# ❌ TypeError が発生
age = 25
# message = "年齢は" + age + "歳です"

# ✅ f-string を使う
message = f"年齢は{age}歳です"
```

---

## やってみよう！

以下のプログラムを自分で書いてみましょう：

1. 自分の名前と年齢を変数に入れて、f-string で `「私の名前は〇〇で、△歳です。」` と表示する
2. 好きな数字を2つ変数に入れて、四則演算（＋ー×÷）の結果をすべて表示する
3. 文字列 `"Python Programming"` を使って、以下を試す：
   - すべて大文字にする
   - 逆順にする（スライスを使う）
   - `"Programming"` を `"Basics"` に置き換える

> ヒント: すべてこの章で学んだ知識だけでできます。分からなければ、該当箇所を読み返してみてください。

---

## まとめ

この章では、Pythonの基本構文と型を学びました：

- ✅ 変数は型宣言不要で定義（snake_caseが規則）
- ✅ 基本型: int, float, str, bool, None
- ✅ f-stringで読みやすい文字列フォーマット
- ✅ スライスで柔軟な文字列・リスト操作
- ✅ `==` は値の比較、`is` は同一性の比較
- ✅ 型変換は int(), float(), str(), bool()

次の章では、if文やforループなどの制御フローを学びます。

---

**次の章**: Chapter 04 - 制御フロー
