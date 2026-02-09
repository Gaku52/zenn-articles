---
title: "制御フロー"
---

# Chapter 04: 制御フロー

> 条件分岐と繰り返しでプログラムの流れを制御する

## この章で学べること

- ✅ if / elif / else による条件分岐
- ✅ for文とwhile文による繰り返し
- ✅ match/case文（Python 3.10+）
- ✅ 内包表記による簡潔なデータ処理

---

## なぜ制御フローが大切なのか

ここまでのプログラムは「上から下へ順番に実行される」だけでした。しかし実際のプログラムは、**状況に応じて動きを変える**必要があります。

- 「ログインしている場合はマイページを表示し、していない場合はログイン画面を表示する」
- 「リストの中身を1つずつ処理する」
- 「入力が正しくなるまで何度もやり直す」

これを実現するのが**制御フロー（条件分岐と繰り返し）**です。この章を学ぶと、プログラムが一気に「賢く」なります。

---

## 条件分岐: if / elif / else

### 基本構文

```python
score = 85

if score >= 90:
    print("優秀")
elif score >= 70:
    print("良好")
elif score >= 50:
    print("合格")
else:
    print("不合格")

# 出力: 良好
```

> **重要**: Pythonでは**インデント（スペース4つ）**がブロックを定義します。波括弧 `{}` は使いません。

### 三項演算子（条件式）

```python
age = 20

# 通常の if 文
if age >= 18:
    status = "成人"
else:
    status = "未成年"

# 三項演算子（1行で書ける）
status = "成人" if age >= 18 else "未成年"
```

### Truthy / Falsy 値

Pythonでは、以下の値は条件式で `False` として扱われます：

```python
# Falsy な値（すべて False 扱い）
bool(False)    # False
bool(0)        # False
bool(0.0)      # False
bool("")       # False（空文字列）
bool([])       # False（空リスト）
bool({})       # False（空辞書）
bool(set())    # False（空集合）
bool(None)     # False

# それ以外はすべて Truthy（True 扱い）
bool(1)        # True
bool("hello")  # True
bool([1, 2])   # True
```

```python
# 実践的な活用
items = []

# ❌ 冗長
if len(items) > 0:
    print("アイテムがあります")

# ✅ Pythonic（Falsy を活用）
if items:
    print("アイテムがあります")

# ❌ 冗長
name = ""
if name != "":
    print(name)

# ✅ Pythonic
if name:
    print(name)
```

---

## 繰り返し: for文

### range() を使ったループ

```python
# 0から4まで（5は含まない）
for i in range(5):
    print(i)  # 0, 1, 2, 3, 4

# 1から10まで
for i in range(1, 11):
    print(i)  # 1, 2, 3, ..., 10

# 0から10まで2刻み
for i in range(0, 11, 2):
    print(i)  # 0, 2, 4, 6, 8, 10

# 逆順
for i in range(10, 0, -1):
    print(i)  # 10, 9, 8, ..., 1
```

### リストのイテレーション

```python
fruits = ["りんご", "バナナ", "オレンジ"]

# 要素を順番に処理
for fruit in fruits:
    print(fruit)
```

### enumerate() でインデックス付き

```python
fruits = ["りんご", "バナナ", "オレンジ"]

# ❌ インデックスを手動管理
i = 0
for fruit in fruits:
    print(f"{i}: {fruit}")
    i += 1

# ✅ enumerate() を使う（推奨）
for i, fruit in enumerate(fruits):
    print(f"{i}: {fruit}")

# 1始まりにもできる
for i, fruit in enumerate(fruits, start=1):
    print(f"{i}. {fruit}")
# 1. りんご
# 2. バナナ
# 3. オレンジ
```

### zip() で複数リストを同時に

```python
names = ["太郎", "花子", "次郎"]
scores = [85, 92, 78]

for name, score in zip(names, scores):
    print(f"{name}: {score}点")
# 太郎: 85点
# 花子: 92点
# 次郎: 78点
```

### break と continue

```python
# break: ループを途中で終了
for i in range(10):
    if i == 5:
        break
    print(i)  # 0, 1, 2, 3, 4

# continue: 現在の反復をスキップ
for i in range(10):
    if i % 2 == 0:
        continue
    print(i)  # 1, 3, 5, 7, 9
```

### for-else 構文

`for` ループが `break` されずに最後まで完了した場合、`else` ブロックが実行されます。

```python
# 素数判定の例
def is_prime(n: int) -> bool:
    if n < 2:
        return False
    for i in range(2, int(n ** 0.5) + 1):
        if n % i == 0:
            break
    else:
        # break されなかった = 約数が見つからなかった = 素数
        return True
    return False

print(is_prime(17))  # True
print(is_prime(15))  # False
```

---

## 繰り返し: while文

### 基本構文

```python
count = 0
while count < 5:
    print(count)
    count += 1
# 0, 1, 2, 3, 4
```

### 無限ループと break

```python
# ユーザー入力を繰り返し受け付ける
while True:
    command = input("コマンド（quit で終了）: ")
    if command == "quit":
        break
    print(f"実行: {command}")
```

### for vs while の使い分け

```python
# ✅ for: 回数が決まっている、コレクションを処理する
for item in items:
    process(item)

for i in range(10):
    print(i)

# ✅ while: 条件が満たされるまで繰り返す
while not is_ready():
    wait()

while user_input != "quit":
    user_input = input("> ")
```

---

## match/case文（Python 3.10+）

Python 3.10で導入された構造的パターンマッチングです。他の言語の `switch` 文に似ていますが、はるかに強力です。

### 基本構文

```python
def handle_status(status: int) -> str:
    match status:
        case 200:
            return "OK"
        case 404:
            return "Not Found"
        case 500:
            return "Internal Server Error"
        case _:  # デフォルト（ワイルドカード）
            return f"Unknown status: {status}"

print(handle_status(200))  # OK
print(handle_status(404))  # Not Found
print(handle_status(999))  # Unknown status: 999
```

### シーケンスパターン

```python
def process_command(command: list[str]) -> str:
    match command:
        case ["quit"]:
            return "終了します"
        case ["hello", name]:
            return f"こんにちは、{name}さん！"
        case ["add", *items]:
            return f"追加: {', '.join(items)}"
        case []:
            return "コマンドが空です"
        case _:
            return "不明なコマンド"

print(process_command(["hello", "太郎"]))   # こんにちは、太郎さん！
print(process_command(["add", "a", "b"]))   # 追加: a, b
```

### マッピングパターン

```python
def handle_event(event: dict) -> str:
    match event:
        case {"type": "click", "button": button}:
            return f"{button}ボタンがクリックされました"
        case {"type": "keypress", "key": key}:
            return f"{key}キーが押されました"
        case {"type": "error", "message": msg}:
            return f"エラー: {msg}"
        case _:
            return "不明なイベント"

print(handle_event({"type": "click", "button": "送信"}))
# 送信ボタンがクリックされました
```

### ガード条件

```python
def classify_number(n: int) -> str:
    match n:
        case x if x < 0:
            return "負の数"
        case 0:
            return "ゼロ"
        case x if x % 2 == 0:
            return "正の偶数"
        case _:
            return "正の奇数"

print(classify_number(-5))  # 負の数
print(classify_number(0))   # ゼロ
print(classify_number(4))   # 正の偶数
print(classify_number(7))   # 正の奇数
```

### match vs if-elif の使い分け

```python
# ✅ match が適している場面: 値のパターンで分岐
match http_method:
    case "GET":    handle_get()
    case "POST":   handle_post()
    case "PUT":    handle_put()
    case "DELETE": handle_delete()

# ✅ if-elif が適している場面: 範囲や複雑な条件
if score >= 90:
    grade = "A"
elif score >= 80:
    grade = "B"
```

---

## 内包表記

### リスト内包表記

```python
# 通常の for ループ
squares = []
for i in range(10):
    squares.append(i ** 2)

# ✅ リスト内包表記（1行で書ける）
squares = [i ** 2 for i in range(10)]
# [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

# 条件付き
evens = [i for i in range(20) if i % 2 == 0]
# [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

# 変換 + 条件
names = ["Alice", "Bob", "Charlie", "Dave"]
long_names = [name.upper() for name in names if len(name) > 3]
# ["ALICE", "CHARLIE", "DAVE"]
```

### 辞書内包表記

```python
# 辞書の作成
squares_dict = {i: i ** 2 for i in range(5)}
# {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}

# キーと値の変換
original = {"a": 1, "b": 2, "c": 3}
inverted = {v: k for k, v in original.items()}
# {1: "a", 2: "b", 3: "c"}
```

### 集合内包表記

```python
# 重複を自動排除
words = ["hello", "world", "hello", "python"]
unique_lengths = {len(w) for w in words}
# {5, 6}
```

### ジェネレータ式

```python
# メモリ効率が良い（全データをメモリに載せない）
total = sum(i ** 2 for i in range(1_000_000))

# リスト内包表記だと全データをメモリに載せる
# total = sum([i ** 2 for i in range(1_000_000)])  # メモリ消費大
```

### 読みやすさの限界

```python
# ✅ 読みやすい内包表記
result = [x * 2 for x in items if x > 0]

# ❌ 複雑すぎる内包表記（for ループに戻すべき）
result = [
    transform(x, y)
    for x in range(10)
    for y in range(10)
    if x != y
    if is_valid(x, y)
]

# ✅ for ループで書き直す
result = []
for x in range(10):
    for y in range(10):
        if x != y and is_valid(x, y):
            result.append(transform(x, y))
```

---

## 実践例

### 例1: じゃんけんゲーム

```python
import random

def janken():
    hands = {"1": "グー", "2": "チョキ", "3": "パー"}
    wins = 0
    losses = 0

    while True:
        print(f"\n現在の成績: {wins}勝 {losses}敗")
        print("[1]グー [2]チョキ [3]パー [q]終了")
        choice = input("選択: ").strip()

        if choice == "q":
            break

        if choice not in hands:
            print("1, 2, 3 のいずれかを入力してください")
            continue

        player = hands[choice]
        cpu = random.choice(list(hands.values()))
        print(f"あなた: {player}  vs  CPU: {cpu}")

        match (player, cpu):
            case (p, c) if p == c:
                print("あいこ！")
            case ("グー", "チョキ") | ("チョキ", "パー") | ("パー", "グー"):
                print("あなたの勝ち！")
                wins += 1
            case _:
                print("あなたの負け...")
                losses += 1

    print(f"\n最終成績: {wins}勝 {losses}敗")

janken()
```

### 例2: FizzBuzz（複数の解法）

```python
# 解法1: if-elif
for i in range(1, 31):
    if i % 15 == 0:
        print("FizzBuzz")
    elif i % 3 == 0:
        print("Fizz")
    elif i % 5 == 0:
        print("Buzz")
    else:
        print(i)

# 解法2: 文字列結合
for i in range(1, 31):
    result = ""
    if i % 3 == 0:
        result += "Fizz"
    if i % 5 == 0:
        result += "Buzz"
    print(result or i)

# 解法3: 内包表記（ワンライナー）
print("\n".join(
    "FizzBuzz" if i % 15 == 0
    else "Fizz" if i % 3 == 0
    else "Buzz" if i % 5 == 0
    else str(i)
    for i in range(1, 31)
))
```

---

## よくある間違い

### 間違い1: インデントの不一致

```python
# ❌ TabとSpaceの混在（SyntaxError）
if True:
	print("Tab")
    print("Space")  # IndentationError!

# ✅ スペース4つに統一（PEP 8 推奨）
if True:
    print("一貫したインデント")
    print("スペース4つ")
```

### 間違い2: for文中でリストを変更

```python
# ❌ ループ中にリストを変更（予期しない動作）
numbers = [1, 2, 3, 4, 5]
for n in numbers:
    if n % 2 == 0:
        numbers.remove(n)  # 要素がスキップされる！
print(numbers)  # [1, 3, 5] ではなく [1, 3, 5] になる場合とならない場合がある

# ✅ 新しいリストを作る
numbers = [1, 2, 3, 4, 5]
odds = [n for n in numbers if n % 2 != 0]
print(odds)  # [1, 3, 5]
```

### 間違い3: 内包表記の過度なネスト

```python
# ❌ 読みにくい
matrix = [[1,2,3],[4,5,6],[7,8,9]]
flat = [x for row in matrix for x in row if x > 3]

# ✅ for ループで明示的に
flat = []
for row in matrix:
    for x in row:
        if x > 3:
            flat.append(x)
```

### 間違い4: 無限ループ

```python
# ❌ 条件が永遠に True（カウンタの更新忘れ）
# count = 0
# while count < 10:
#     print(count)
#     # count += 1 を忘れている！

# ✅ カウンタを必ず更新
count = 0
while count < 10:
    print(count)
    count += 1
```

---

## やってみよう！

1. **数当てゲーム**: 1〜100のランダムな数を当てるゲームを作ってみましょう（ヒント: `import random` → `random.randint(1, 100)`、while文とif文を使う）
2. **九九の表**: for文を2重にして、九九の表を表示してみましょう
3. **FizzBuzz改造**: FizzBuzzに「7の倍数なら "Bizz"」というルールを追加してみましょう

> **ここまで来たら、もう「プログラミングの基本」は半分以上理解しています。** if文とfor文が使えれば、驚くほど多くのことが実現できます。自信を持って次に進みましょう！

---

## まとめ

この章では、プログラムの流れを制御する方法を学びました：

- ✅ if / elif / else で条件分岐
- ✅ Truthy / Falsy を活用した Pythonic な条件式
- ✅ for 文と range(), enumerate(), zip()
- ✅ while 文と break / continue
- ✅ match/case 文で強力なパターンマッチング
- ✅ 内包表記で簡潔なデータ処理

次の章では、関数の定義と活用方法を学びます。

---

**次の章**: Chapter 05 - 関数とスコープ
