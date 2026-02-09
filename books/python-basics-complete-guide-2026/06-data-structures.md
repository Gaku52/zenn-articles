---
title: "データ構造"
---

# Chapter 06: データ構造

> Pythonの4大データ構造を使いこなす

## この章で学べること

- ✅ リスト（list）の操作と活用
- ✅ 辞書（dict）の操作と活用
- ✅ タプル（tuple）の特性と使いどころ
- ✅ 集合（set）の演算
- ✅ データ構造の選び方

---

## なぜデータ構造が大切なのか

プログラミングの世界には、こんな格言があります：

> **「正しいデータ構造を選べば、アルゴリズムは自明になる」**

データ構造とは、データを「どのような形で保存・整理するか」の方法です。

- **リスト**: 買い物リスト、TODOリストのように「順番に並んだデータ」
- **辞書**: 英和辞書のように「キーワードで引けるデータ」
- **タプル**: 座標 (x, y) のように「変わらないデータのセット」
- **集合**: 出席者の名簿のように「重複のないデータ」

適切なデータ構造を選べるようになると、プログラムが驚くほどシンプルになります。

---

## リスト（list）

### 作成と基本操作

```python
# リストの作成
fruits: list[str] = ["りんご", "バナナ", "オレンジ"]
numbers: list[int] = [1, 2, 3, 4, 5]
mixed: list = [1, "hello", True, 3.14]  # 型混在も可能（非推奨）

# 要素の追加
fruits.append("ぶどう")          # 末尾に追加
fruits.insert(1, "いちご")       # 指定位置に挿入
fruits.extend(["メロン", "桃"])  # 複数追加

# 要素の削除
fruits.remove("バナナ")   # 値で削除（最初の1つ）
last = fruits.pop()       # 末尾を取り出し
first = fruits.pop(0)     # 指定位置を取り出し
del fruits[1]             # インデックスで削除

# 要素の検索
"りんご" in fruits      # True（存在確認）
fruits.index("オレンジ")  # インデックスを返す
fruits.count("りんご")    # 出現回数
len(fruits)               # 要素数
```

### ソート

```python
numbers = [3, 1, 4, 1, 5, 9, 2, 6]

# sort(): 元のリストを変更（破壊的）
numbers.sort()
print(numbers)  # [1, 1, 2, 3, 4, 5, 6, 9]

numbers.sort(reverse=True)
print(numbers)  # [9, 6, 5, 4, 3, 2, 1, 1]

# sorted(): 新しいリストを返す（非破壊的）
original = [3, 1, 4, 1, 5]
sorted_list = sorted(original)
print(original)     # [3, 1, 4, 1, 5]（変更されない）
print(sorted_list)  # [1, 1, 3, 4, 5]

# key引数でカスタムソート
words = ["banana", "apple", "cherry", "date"]
by_length = sorted(words, key=len)
# ['date', 'apple', 'banana', 'cherry']

# 辞書のリストをソート
users = [
    {"name": "太郎", "age": 25},
    {"name": "花子", "age": 30},
    {"name": "次郎", "age": 20},
]
by_age = sorted(users, key=lambda u: u["age"])
# [次郎(20), 太郎(25), 花子(30)]
```

### コピーの注意点

```python
# ❌ 代入はコピーではない（同じオブジェクトを参照）
a = [1, 2, 3]
b = a
b.append(4)
print(a)  # [1, 2, 3, 4] ← a も変わってしまう！

# ✅ 浅いコピー
a = [1, 2, 3]
b = a.copy()      # 方法1
b = a[:]           # 方法2
b = list(a)        # 方法3
b.append(4)
print(a)  # [1, 2, 3]（a は変わらない）

# ⚠️ ネストしたリストは浅いコピーでは不十分
matrix = [[1, 2], [3, 4]]
shallow = matrix.copy()
shallow[0][0] = 99
print(matrix)  # [[99, 2], [3, 4]] ← 内部リストは共有！

# ✅ 深いコピー（ネストも完全にコピー）
import copy
matrix = [[1, 2], [3, 4]]
deep = copy.deepcopy(matrix)
deep[0][0] = 99
print(matrix)  # [[1, 2], [3, 4]]（影響なし）
```

---

## 辞書（dict）

### 作成と基本操作

```python
# 辞書の作成
user: dict[str, str | int] = {
    "name": "太郎",
    "age": 25,
    "city": "東京",
}

# 値の取得
print(user["name"])        # "太郎"
# print(user["email"])     # ❌ KeyError!

# .get() でデフォルト値を指定（安全）
print(user.get("email", "未設定"))  # "未設定"

# 値の追加・更新
user["email"] = "taro@example.com"  # 追加
user["age"] = 26                     # 更新

# 値の削除
del user["city"]
email = user.pop("email", None)  # 取り出し（なければ None）

# キー・値の取得
user.keys()    # dict_keys(['name', 'age'])
user.values()  # dict_values(['太郎', 26])
user.items()   # dict_items([('name', '太郎'), ('age', 26)])
```

### 辞書のイテレーション

```python
scores = {"数学": 85, "英語": 92, "国語": 78}

# キーでループ
for subject in scores:
    print(subject)

# キーと値でループ（推奨）
for subject, score in scores.items():
    print(f"{subject}: {score}点")

# 値でループ
for score in scores.values():
    print(score)
```

### 辞書内包表記

```python
# 基本
squares = {i: i ** 2 for i in range(6)}
# {0: 0, 1: 1, 2: 4, 3: 9, 4: 16, 5: 25}

# 条件付き
high_scores = {k: v for k, v in scores.items() if v >= 80}
# {"数学": 85, "英語": 92}

# キーと値の変換
inverted = {v: k for k, v in scores.items()}
# {85: "数学", 92: "英語", 78: "国語"}
```

### collections.defaultdict

```python
from collections import defaultdict

# 通常の辞書ではキーが存在しないとエラー
# ❌ counts = {}
# counts["apple"] += 1  # KeyError!

# ✅ defaultdict なら自動で初期値が設定される
counts: defaultdict[str, int] = defaultdict(int)
counts["apple"] += 1
counts["banana"] += 1
counts["apple"] += 1
print(dict(counts))  # {'apple': 2, 'banana': 1}

# リストの defaultdict
groups: defaultdict[str, list] = defaultdict(list)
groups["fruit"].append("apple")
groups["fruit"].append("banana")
groups["vegetable"].append("carrot")
print(dict(groups))
# {'fruit': ['apple', 'banana'], 'vegetable': ['carrot']}
```

---

## タプル（tuple）

### イミュータブルなシーケンス

```python
# タプルの作成
point: tuple[int, int] = (10, 20)
rgb: tuple[int, int, int] = (255, 128, 0)
single: tuple[int] = (42,)  # 要素1つの場合、カンマが必要

# 要素のアクセス（リストと同じ）
print(point[0])  # 10
print(point[1])  # 20

# ❌ タプルは変更できない（イミュータブル）
# point[0] = 30  # TypeError!
```

### タプルを使うべき場面

```python
# 1. 関数の複数返り値
def get_min_max(numbers: list[int]) -> tuple[int, int]:
    return min(numbers), max(numbers)

low, high = get_min_max([3, 1, 4, 1, 5, 9])

# 2. 辞書のキー（リストはキーにできない）
locations: dict[tuple[float, float], str] = {
    (35.6762, 139.6503): "東京",
    (34.6937, 135.5023): "大阪",
}

# 3. 変更されたくないデータ
DAYS_OF_WEEK = ("月", "火", "水", "木", "金", "土", "日")
```

### 名前付きタプル

```python
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float

class User(NamedTuple):
    name: str
    age: int
    email: str

# インデックスでも名前でもアクセス可能
p = Point(3.0, 4.0)
print(p.x)    # 3.0
print(p[0])   # 3.0

user = User("太郎", 25, "taro@example.com")
print(user.name)   # "太郎"
print(user.email)  # "taro@example.com"
```

---

## 集合（set）

### 重複排除

```python
# 集合の作成
colors: set[str] = {"赤", "青", "緑", "赤", "青"}
print(colors)  # {'赤', '青', '緑'}（重複が除去される）

# リストから重複を排除
numbers = [1, 2, 2, 3, 3, 3, 4]
unique = list(set(numbers))
print(unique)  # [1, 2, 3, 4]（順序は保証されない）

# 順序を保持して重複排除（Python 3.7+ の dict の順序保証を利用）
unique_ordered = list(dict.fromkeys(numbers))
print(unique_ordered)  # [1, 2, 3, 4]（順序が保持される）
```

### 集合演算

```python
a = {1, 2, 3, 4, 5}
b = {4, 5, 6, 7, 8}

# 和集合（AまたはB）
a | b            # {1, 2, 3, 4, 5, 6, 7, 8}
a.union(b)

# 積集合（AかつB）
a & b            # {4, 5}
a.intersection(b)

# 差集合（Aにあって Bにない）
a - b            # {1, 2, 3}
a.difference(b)

# 対称差（どちらか一方にだけある）
a ^ b            # {1, 2, 3, 6, 7, 8}
a.symmetric_difference(b)

# 部分集合の判定
{1, 2} <= {1, 2, 3}     # True（部分集合）
{1, 2, 3} >= {1, 2}     # True（上位集合）
```

### 実用例

```python
# ユーザーの権限管理
admin_perms = {"read", "write", "delete", "admin"}
editor_perms = {"read", "write"}
viewer_perms = {"read"}

# 管理者だけが持つ権限
admin_only = admin_perms - editor_perms
print(admin_only)  # {'delete', 'admin'}

# 共通の権限
common = admin_perms & editor_perms
print(common)  # {'read', 'write'}
```

---

## データ構造の比較と選び方

### パフォーマンス比較

| 操作 | list | dict | set |
|------|------|------|-----|
| 要素のアクセス | O(1)（インデックス） | O(1)（キー） | - |
| 要素の検索 | **O(n)**（遅い） | **O(1)**（速い） | **O(1)**（速い） |
| 要素の追加 | O(1)（末尾） | O(1) | O(1) |
| 要素の削除 | O(n) | O(1) | O(1) |

### 選択ガイド

```python
# ✅ list: 順序が重要、重複を許す
tasks = ["買い物", "掃除", "料理", "買い物"]

# ✅ dict: キーで値を引く
config = {"host": "localhost", "port": 8080}

# ✅ tuple: 変更不要、関数の返り値
point = (10, 20)

# ✅ set: 重複排除、集合演算
tags = {"python", "programming", "tutorial"}

# ❌ 不適切な例: 検索に list を使う
# 10万件のデータから検索
users_list = [...]  # ❌ O(n) — 遅い
users_dict = {...}  # ✅ O(1) — 速い
```

---

## 実践例

### 例1: 単語カウンター

```python
from collections import Counter

def count_words(text: str) -> list[tuple[str, int]]:
    """テキスト内の単語出現回数をカウントする"""
    words = text.lower().split()
    # 記号を除去
    words = [w.strip(".,!?;:") for w in words]
    counter = Counter(words)
    return counter.most_common(10)

text = """
Python is a great programming language.
Python is easy to learn. Python is powerful.
I love Python programming.
"""

for word, count in count_words(text):
    print(f"  {word}: {count}回")
# python: 4回
# is: 3回
# programming: 2回
# ...
```

### 例2: 生徒の成績管理

```python
def manage_students():
    """生徒の成績を管理するプログラム"""
    students: dict[str, list[int]] = {}

    # データ登録
    students["太郎"] = [85, 92, 78]
    students["花子"] = [95, 88, 91]
    students["次郎"] = [70, 65, 80]

    # 各生徒の平均点
    for name, scores in students.items():
        avg = sum(scores) / len(scores)
        print(f"{name}: 平均 {avg:.1f}点")

    # 全体の最高平均点の生徒
    top_student = max(
        students.items(),
        key=lambda item: sum(item[1]) / len(item[1])
    )
    print(f"\n最優秀: {top_student[0]}")

    # 科目ごとの平均（zip で転置）
    subjects = ["数学", "英語", "国語"]
    all_scores = list(students.values())
    for subject, scores in zip(subjects, zip(*all_scores)):
        avg = sum(scores) / len(scores)
        print(f"{subject}の平均: {avg:.1f}点")

manage_students()
```

---

## よくある間違い

### 間違い1: リストのコピーで代入

```python
# ❌ 代入はコピーではない
original = [1, 2, 3]
copied = original  # 同じオブジェクトを参照
copied.append(4)
print(original)  # [1, 2, 3, 4] ← 元も変わる！

# ✅ .copy() を使う
original = [1, 2, 3]
copied = original.copy()
copied.append(4)
print(original)  # [1, 2, 3]
```

### 間違い2: for文中のリスト変更

```python
# ❌ ループ中にリストを変更
items = [1, 2, 3, 4, 5]
for item in items:
    if item % 2 == 0:
        items.remove(item)  # 予期しない動作！

# ✅ 新しいリストを作る
items = [1, 2, 3, 4, 5]
items = [item for item in items if item % 2 != 0]
```

### 間違い3: 存在しないキーへのアクセス

```python
user = {"name": "太郎"}

# ❌ KeyError が発生
# print(user["email"])

# ✅ .get() でデフォルト値を指定
print(user.get("email", "未設定"))

# ✅ in で事前確認
if "email" in user:
    print(user["email"])
```

### 間違い4: ミュータブルを辞書のキーに使う

```python
# ❌ リストはハッシュ不可なのでキーにできない
# d = {[1, 2]: "value"}  # TypeError!

# ✅ タプルはハッシュ可能なのでキーにできる
d = {(1, 2): "value"}  # OK
```

---

## やってみよう！

1. **自己紹介辞書**: 自分の情報（名前、年齢、趣味のリスト、住所）を辞書で作り、f-string で整形して表示する
2. **重複チェッカー**: リストを受け取って、重複している要素だけを返す関数を作ってみましょう（ヒント: set を使う）
3. **簡易電話帳**: 辞書を使って、名前で電話番号を検索・追加・削除できるプログラムを作ってみましょう

> **ここまでの知識（変数、制御フロー、関数、データ構造）で、実用的なプログラムが書けるようになりました！** Chapter 01 で見た TODO アプリも、もう自力で作れるはずです。ぜひ挑戦してみてください。

---

## まとめ

この章では、Pythonの4大データ構造を学びました：

- ✅ **list**: 順序あり、変更可能、重複OK → 最も汎用的
- ✅ **dict**: キーと値のペア、O(1)検索 → 構造化データに
- ✅ **tuple**: 順序あり、変更不可 → 定数や返り値に
- ✅ **set**: 順序なし、重複なし → 集合演算に
- ✅ 検索が多い場面では dict/set（O(1)）を使う

次の章では、型ヒントとデータクラスで型安全なコードを書く方法を学びます。

---

**次の章**: Chapter 07 - 型ヒントとデータクラス
