---
title: "ファイル操作と例外処理"
---

# Chapter 10: ファイル操作と例外処理

> 外部データの読み書きとエラーへの適切な対処を学ぶ

## この章で学べること

- ✅ ファイルの読み書き（with文）
- ✅ pathlib でのファイルパス操作
- ✅ JSON / CSV の処理
- ✅ 例外処理（try / except）
- ✅ カスタム例外の作成

---

## なぜファイル操作と例外処理を学ぶのか

ここまでのプログラムは、実行するたびにデータが消えてしまいました。ファイル操作を覚えると、**データを永続的に保存**できるようになります。

- 設定ファイルの読み書き
- CSVデータの集計
- ログの記録

そして例外処理は、**プログラムが「想定外の事態」に出会ったときの対処法**です。ファイルが見つからない、ネットワークが繋がらない——こうした「もしも」に備えるのが例外処理です。

> **たとえるなら**: ファイル操作は「ノートに書き留める」こと。例外処理は「もしもの備え」——傘を持っていけば、急な雨でも大丈夫です。

---

## ファイルの読み書き

### with文の重要性

```python
# ✅ with文を使う（推奨: ファイルが自動的に閉じられる）
with open("hello.txt", "w", encoding="utf-8") as f:
    f.write("Hello, Python!\n")
    f.write("ファイル操作の練習です\n")
# with ブロックを抜けると自動的に f.close() される

# ❌ with を使わない（close 忘れのリスク）
f = open("hello.txt", "w", encoding="utf-8")
f.write("Hello")
f.close()  # 忘れるとデータが失われる可能性
```

### ファイルモード

| モード | 説明 |
|--------|------|
| `"r"` | 読み込み（デフォルト） |
| `"w"` | 書き込み（上書き） |
| `"a"` | 追記 |
| `"x"` | 新規作成（既存なら エラー） |
| `"b"` | バイナリモード |

### テキストファイルの読み込み

```python
# ファイル全体を一度に読む
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()
    print(content)

# 1行ずつ読む（メモリ効率が良い）
with open("data.txt", "r", encoding="utf-8") as f:
    for line in f:
        print(line.strip())  # strip() で改行を除去

# 全行をリストで取得
with open("data.txt", "r", encoding="utf-8") as f:
    lines = f.readlines()
    # ["行1\n", "行2\n", "行3\n"]
```

### テキストファイルの書き込み

```python
# 書き込み（上書き）
with open("output.txt", "w", encoding="utf-8") as f:
    f.write("1行目\n")
    f.write("2行目\n")

# 追記
with open("output.txt", "a", encoding="utf-8") as f:
    f.write("追記された行\n")

# 複数行を一度に書き込み
lines = ["行1\n", "行2\n", "行3\n"]
with open("output.txt", "w", encoding="utf-8") as f:
    f.writelines(lines)
```

---

## pathlib でのファイル操作

### 簡潔なファイル読み書き

```python
from pathlib import Path

# ✅ pathlib なら1行で読み書きできる
content = Path("data.txt").read_text(encoding="utf-8")
Path("output.txt").write_text("Hello, Python!", encoding="utf-8")

# バイナリファイル
data = Path("image.png").read_bytes()
Path("copy.png").write_bytes(data)
```

### パスの操作

```python
from pathlib import Path

# パスの結合
data_dir = Path("project") / "data" / "raw"

# ファイル情報
path = Path("report.csv")
path.suffix      # ".csv"
path.stem        # "report"
path.name        # "report.csv"
path.parent      # Path(".")

# 拡張子の変更
path.with_suffix(".json")  # Path("report.json")

# ディレクトリ作成
Path("output/reports").mkdir(parents=True, exist_ok=True)

# ファイル一覧
for f in Path("src").rglob("*.py"):
    print(f)

# ファイルサイズ
size = Path("data.csv").stat().st_size
print(f"{size:,} bytes")
```

---

## JSON の処理

### 読み書き

```python
import json
from pathlib import Path

# Python → JSON ファイル
config = {
    "app_name": "MyApp",
    "version": "1.0.0",
    "debug": True,
    "database": {
        "host": "localhost",
        "port": 5432,
    },
    "tags": ["python", "web"],
}

# 書き込み
Path("config.json").write_text(
    json.dumps(config, ensure_ascii=False, indent=2),
    encoding="utf-8"
)

# 読み込み
loaded = json.loads(Path("config.json").read_text(encoding="utf-8"))
print(loaded["app_name"])  # MyApp
print(loaded["database"]["host"])  # localhost
```

### 型安全なJSON処理（TypedDict と組み合わせ）

```python
import json
from typing import TypedDict
from pathlib import Path

class DatabaseConfig(TypedDict):
    host: str
    port: int

class AppConfig(TypedDict):
    app_name: str
    version: str
    debug: bool
    database: DatabaseConfig

def load_config(path: Path) -> AppConfig:
    """設定ファイルを型安全に読み込む"""
    data = json.loads(path.read_text(encoding="utf-8"))
    return data  # 型チェッカーが構造を検証

config = load_config(Path("config.json"))
print(config["database"]["host"])  # 型補完が効く
```

---

## CSV の処理

### 基本的な読み書き

```python
import csv

# CSV書き込み
with open("students.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.writer(f)
    writer.writerow(["名前", "数学", "英語", "国語"])  # ヘッダー
    writer.writerow(["太郎", 85, 92, 78])
    writer.writerow(["花子", 95, 88, 91])
    writer.writerow(["次郎", 70, 65, 80])

# CSV読み込み
with open("students.csv", "r", encoding="utf-8") as f:
    reader = csv.reader(f)
    header = next(reader)  # ヘッダー行を取得
    for row in reader:
        name, math, eng, jpn = row
        avg = (int(math) + int(eng) + int(jpn)) / 3
        print(f"{name}: 平均 {avg:.1f}点")
```

### DictReader / DictWriter（推奨）

```python
import csv

# 辞書形式で読み込み（列名でアクセスできる）
with open("students.csv", "r", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(f"{row['名前']}: 数学 {row['数学']}点")

# 辞書形式で書き込み
students = [
    {"名前": "太郎", "数学": 85, "英語": 92},
    {"名前": "花子", "数学": 95, "英語": 88},
]

with open("output.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["名前", "数学", "英語"])
    writer.writeheader()
    writer.writerows(students)
```

---

## 例外処理の基本

### try / except / else / finally

```python
def safe_divide(a: float, b: float) -> float | None:
    try:
        result = a / b
    except ZeroDivisionError:
        print("エラー: 0で割ることはできません")
        return None
    except TypeError as e:
        print(f"エラー: 不正な型です - {e}")
        return None
    else:
        # 例外が発生しなかった場合に実行
        print(f"計算成功: {a} / {b} = {result}")
        return result
    finally:
        # 例外の有無にかかわらず必ず実行
        print("計算処理を終了しました")

safe_divide(10, 3)   # 計算成功: 10 / 3 = 3.333...
safe_divide(10, 0)   # エラー: 0で割ることはできません
```

### 具体的な例外をキャッチする

```python
# ✅ 具体的な例外を指定（推奨）
try:
    value = int(input("数値を入力: "))
    result = 100 / value
except ValueError:
    print("数値を入力してください")
except ZeroDivisionError:
    print("0以外を入力してください")

# ✅ 複数例外をまとめてキャッチ
try:
    data = json.loads(some_string)
except (json.JSONDecodeError, TypeError) as e:
    print(f"JSONパースエラー: {e}")

# ❌ bare except（すべての例外をキャッチ）
try:
    something()
except:  # ❌ KeyboardInterrupt まで吸収してしまう
    pass

# ❌ Exception を雑にキャッチして握りつぶす
try:
    something()
except Exception:
    pass  # ❌ エラーが隠蔽されてデバッグ不可能に
```

### 主要な組み込み例外

| 例外 | 発生する場面 |
|------|-------------|
| `ValueError` | 値が不正（`int("abc")`） |
| `TypeError` | 型が不正（`"a" + 1`） |
| `KeyError` | 辞書にキーがない |
| `IndexError` | リストの範囲外アクセス |
| `FileNotFoundError` | ファイルが見つからない |
| `AttributeError` | 属性/メソッドがない |
| `ZeroDivisionError` | 0で除算 |
| `ImportError` | インポート失敗 |

---

## 例外の発生とカスタム例外

### raise で例外を投げる

```python
def withdraw(balance: int, amount: int) -> int:
    if amount <= 0:
        raise ValueError("出金額は正の数でなければなりません")
    if amount > balance:
        raise ValueError(f"残高不足（残高: {balance:,}円、出金額: {amount:,}円）")
    return balance - amount

try:
    new_balance = withdraw(10000, 50000)
except ValueError as e:
    print(f"エラー: {e}")
    # エラー: 残高不足（残高: 10,000円、出金額: 50,000円）
```

### カスタム例外クラス

```python
class AppError(Exception):
    """アプリケーションの基底例外"""
    pass

class ValidationError(AppError):
    """バリデーションエラー"""
    def __init__(self, field: str, message: str) -> None:
        self.field = field
        self.message = message
        super().__init__(f"{field}: {message}")

class NotFoundError(AppError):
    """リソースが見つからない"""
    def __init__(self, resource: str, id: int) -> None:
        self.resource = resource
        self.id = id
        super().__init__(f"{resource}（ID: {id}）が見つかりません")

# 使用例
def create_user(name: str, age: int) -> dict:
    if not name:
        raise ValidationError("name", "名前は必須です")
    if age < 0 or age > 150:
        raise ValidationError("age", "年齢は0〜150の範囲で指定してください")
    return {"name": name, "age": age}

try:
    user = create_user("", 25)
except ValidationError as e:
    print(f"バリデーションエラー: {e.field} - {e.message}")
```

### 例外チェーン（raise ... from ...）

```python
def load_config(path: str) -> dict:
    try:
        import json
        with open(path, encoding="utf-8") as f:
            return json.load(f)
    except FileNotFoundError as e:
        raise AppError(f"設定ファイルが見つかりません: {path}") from e
    except json.JSONDecodeError as e:
        raise AppError(f"設定ファイルの形式が不正です: {path}") from e
```

---

## 例外処理のベストプラクティス

### EAFP vs LBYL

```python
# ✅ EAFP (Easier to Ask Forgiveness than Permission) — Pythonic
# まずやってみて、ダメだったら例外で対処
try:
    value = data["key"]
except KeyError:
    value = "default"

# LBYL (Look Before You Leap)
# 事前に確認してから実行
if "key" in data:
    value = data["key"]
else:
    value = "default"

# ✅ さらに Pythonic: .get() を使う
value = data.get("key", "default")
```

```python
# ✅ EAFP: ファイル操作
try:
    content = Path("config.json").read_text(encoding="utf-8")
except FileNotFoundError:
    content = "{}"  # デフォルト値

# LBYL: ファイル操作
path = Path("config.json")
if path.exists() and path.is_file():
    content = path.read_text(encoding="utf-8")
else:
    content = "{}"
```

---

## 実践例

### 例1: 設定ファイルの読み書きクラス

```python
import json
from pathlib import Path
from dataclasses import dataclass, field, asdict

@dataclass
class Config:
    """アプリケーション設定"""
    app_name: str = "MyApp"
    debug: bool = False
    port: int = 8000
    allowed_hosts: list[str] = field(default_factory=lambda: ["localhost"])

class ConfigManager:
    def __init__(self, path: Path) -> None:
        self.path = path

    def load(self) -> Config:
        """設定ファイルを読み込む"""
        try:
            data = json.loads(self.path.read_text(encoding="utf-8"))
            return Config(**data)
        except FileNotFoundError:
            print(f"設定ファイルが見つかりません。デフォルト設定を使用します。")
            config = Config()
            self.save(config)
            return config
        except (json.JSONDecodeError, TypeError) as e:
            print(f"設定ファイルの読み込みエラー: {e}")
            return Config()

    def save(self, config: Config) -> None:
        """設定ファイルを保存する"""
        self.path.parent.mkdir(parents=True, exist_ok=True)
        self.path.write_text(
            json.dumps(asdict(config), ensure_ascii=False, indent=2),
            encoding="utf-8"
        )

# 使用例
manager = ConfigManager(Path("config.json"))
config = manager.load()
config.debug = True
manager.save(config)
```

### 例2: CSVデータの集計

```python
import csv
from pathlib import Path
from collections import defaultdict

def analyze_sales(filepath: Path) -> None:
    """売上CSVを読み込んで集計する"""
    if not filepath.exists():
        print(f"ファイルが見つかりません: {filepath}")
        return

    monthly_sales: defaultdict[str, int] = defaultdict(int)
    total_rows = 0
    error_rows = 0

    with open(filepath, "r", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        for row in reader:
            total_rows += 1
            try:
                month = row["date"][:7]  # "2026-01-15" → "2026-01"
                amount = int(row["amount"])
                monthly_sales[month] += amount
            except (KeyError, ValueError) as e:
                error_rows += 1
                print(f"  行 {total_rows}: スキップ（{e}）")

    print(f"\n=== 売上集計 ===")
    print(f"処理行数: {total_rows}（エラー: {error_rows}）\n")

    for month, total in sorted(monthly_sales.items()):
        print(f"  {month}: {total:>12,}円")

    grand_total = sum(monthly_sales.values())
    print(f"\n  合計: {grand_total:>14,}円")

# 使用例
analyze_sales(Path("sales.csv"))
```

---

## よくある間違い

### 間違い1: bare except で全例外をキャッチ

```python
# ❌ KeyboardInterrupt（Ctrl+C）まで吸収してしまう
try:
    while True:
        process()
except:
    pass  # 永遠に止められない！

# ✅ 具体的な例外を指定
try:
    while True:
        process()
except ProcessingError as e:
    log_error(e)
```

### 間違い2: 例外を握りつぶす

```python
# ❌ エラーを隠蔽
try:
    result = complex_operation()
except Exception:
    pass  # 何が起きたか分からない

# ✅ 少なくともログを残す
import logging

try:
    result = complex_operation()
except Exception:
    logging.exception("complex_operation でエラーが発生")
    result = default_value
```

### 間違い3: with文を使わないファイル操作

```python
# ❌ close 忘れのリスク
f = open("data.txt", "r")
content = f.read()
# f.close() を忘れると...

# ✅ with文で自動クローズ
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()
```

---

## やってみよう！

1. **日記プログラム**: 日付をキーにしたJSON形式の日記帳を作りましょう。日記の追加、一覧表示、キーワード検索ができるようにしてください
2. **CSVの成績表**: `students.csv` を読み込んで、各生徒の平均点、クラスの最高点、最低点を表示するプログラムを作りましょう
3. **堅牢なファイル読み込み**: ファイルが存在しない場合、JSON形式が壊れている場合、文字コードが違う場合、それぞれ適切なエラーメッセージを表示する関数を作ってみましょう

> **ファイル操作と例外処理ができれば、「実用的なツール」が作れるようになります。** この章の知識は、データ分析、Web開発、自動化——どの道に進んでも必ず使います。

---

## まとめ

この章では、ファイル操作と例外処理を学びました：

- ✅ with文でファイルを安全に読み書き
- ✅ pathlib で簡潔なファイルパス操作
- ✅ JSON / CSV の読み書き
- ✅ try / except で適切なエラーハンドリング
- ✅ カスタム例外でドメイン固有のエラーを表現
- ✅ EAFP パターンが Pythonic

次の章では、この本で学んだことを振り返り、さらなる成長への道を示します。

---

**次の章**: Chapter 11 - 次のステップ
