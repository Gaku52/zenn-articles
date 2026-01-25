---
title: "第13章 Pythonレビューガイド"
---

# 第13章 Pythonレビューガイド

本章ではPythonコードレビューにおける重要なチェックポイントと、具体的なレビュー観点について詳しく解説します。Pythonは読みやすく書きやすい言語として知られていますが、それ故にコーディング規約の遵守や、型ヒントの適切な活用が重要となります。

## Pythonコードレビューの特性

Pythonは動的型付け言語であり、柔軟性が高い一方で、型安全性の確保やパフォーマンスに関する配慮が必要です。レビューでは以下の観点を重点的に確認することが推奨されます。

### レビューの重点領域

- 型ヒント（Type Hints）の適切な活用
- PEP 8スタイルガイドの遵守
- リソース管理の安全性
- 例外処理の妥当性
- パフォーマンスと可読性のバランス

## 型ヒント（Type Hints）の活用

Python 3.5以降で導入された型ヒントは、コードの可読性と保守性を大幅に向上させます。レビューでは型ヒントの適切な使用を確認しましょう。

### 悪い例: 型ヒントなし

```python
def calculate_total(items, tax_rate):
    subtotal = sum(item['price'] * item['quantity'] for item in items)
    return subtotal * (1 + tax_rate)

def get_user_by_id(user_id):
    # データベースからユーザーを取得
    user = database.query(user_id)
    return user
```

この例では、引数や戻り値の型が不明確で、関数を使う側がドキュメントを読まなければ正しい使い方がわかりません。

### 良い例: 型ヒントを活用

```python
from typing import List, Dict, Optional
from decimal import Decimal

def calculate_total(
    items: List[Dict[str, Decimal]],
    tax_rate: Decimal
) -> Decimal:
    """商品リストから税込み合計金額を計算する

    Args:
        items: 商品情報のリスト（price, quantityを含む辞書）
        tax_rate: 税率（例: 0.10 = 10%）

    Returns:
        税込み合計金額
    """
    subtotal = sum(item['price'] * item['quantity'] for item in items)
    return subtotal * (1 + tax_rate)

def get_user_by_id(user_id: int) -> Optional[Dict[str, Any]]:
    """ユーザーIDからユーザー情報を取得する

    Args:
        user_id: ユーザーID

    Returns:
        ユーザー情報の辞書。見つからない場合はNone
    """
    user = database.query(user_id)
    return user if user else None
```

型ヒントを追加することで、IDEの補完機能が効くようになり、mypyなどの静的型チェッカーでバグを事前に検出できます。

## with文によるリソース管理

ファイルやデータベース接続などのリソース管理は、確実にクリーンアップされることが重要です。with文を使用することで、例外が発生した場合でも確実にリソースが解放されます。

### 悪い例: 手動でのリソース管理

```python
def read_config(filename):
    file = open(filename, 'r')
    config = json.load(file)
    file.close()  # 例外発生時にクローズされない
    return config

def save_data(data, db_connection):
    cursor = db_connection.cursor()
    cursor.execute("INSERT INTO data VALUES (?)", (data,))
    db_connection.commit()
    cursor.close()  # 例外発生時にクローズされない
```

この例では、json.loadで例外が発生するとファイルが閉じられないまま関数が終了します。

### 良い例: with文を使用

```python
from typing import Dict, Any
from contextlib import contextmanager
import json

def read_config(filename: str) -> Dict[str, Any]:
    """設定ファイルを読み込む

    Args:
        filename: 設定ファイルのパス

    Returns:
        設定情報の辞書

    Raises:
        FileNotFoundError: ファイルが存在しない場合
        json.JSONDecodeError: JSON形式が不正な場合
    """
    with open(filename, 'r', encoding='utf-8') as file:
        config = json.load(file)
    return config

def save_data(data: str, db_connection) -> None:
    """データベースにデータを保存する

    Args:
        data: 保存するデータ
        db_connection: データベース接続オブジェクト
    """
    with db_connection.cursor() as cursor:
        cursor.execute("INSERT INTO data VALUES (?)", (data,))
        db_connection.commit()
```

with文を使用することで、例外が発生してもリソースが確実に解放されます。

## 例外処理のベストプラクティス

Pythonの例外処理は強力ですが、適切に使用しなければコードの可読性と保守性を損ないます。

### 悪い例: 広範囲すぎる例外捕捉

```python
def process_user_data(user_id):
    try:
        user = get_user(user_id)
        data = transform_data(user)
        save_data(data)
        send_notification(user)
        return True
    except:  # すべての例外を捕捉してしまう
        return False
```

この例では、どの処理で例外が発生したのか、どんな種類の例外なのかが不明確です。

### 良い例: 具体的な例外処理

```python
from typing import Optional
import logging

logger = logging.getLogger(__name__)

class UserNotFoundError(Exception):
    """ユーザーが見つからない場合の例外"""
    pass

class DataTransformError(Exception):
    """データ変換に失敗した場合の例外"""
    pass

def process_user_data(user_id: int) -> bool:
    """ユーザーデータを処理する

    Args:
        user_id: ユーザーID

    Returns:
        処理が成功した場合True、失敗した場合False
    """
    try:
        user = get_user(user_id)
    except UserNotFoundError:
        logger.warning(f"User not found: {user_id}")
        return False

    try:
        data = transform_data(user)
    except DataTransformError as e:
        logger.error(f"Data transformation failed: {e}")
        return False

    try:
        save_data(data)
        send_notification(user)
    except IOError as e:
        logger.error(f"Failed to save data: {e}")
        return False
    except ConnectionError as e:
        logger.warning(f"Failed to send notification: {e}")
        # 通知失敗はエラーとしない

    return True
```

具体的な例外を捕捉し、適切なログを出力することで、問題の原因を特定しやすくなります。

## リスト内包表記 vs ループ

Pythonではリスト内包表記を使うことで、コードを簡潔に書けます。ただし、可読性を損なうほど複雑な内包表記は避けるべきです。

### 悪い例: 複雑すぎる内包表記

```python
# 読みにくい複雑な内包表記
result = [
    item['value'] * 2
    for sublist in data
    for item in sublist
    if item['type'] == 'active'
    if item['value'] > 100
    if item['category'] in valid_categories
]
```

### 良い例: 適切な複雑度の使い分け

```python
from typing import List, Dict, Any

def process_active_items(
    data: List[List[Dict[str, Any]]],
    valid_categories: set
) -> List[int]:
    """アクティブなアイテムの値を2倍にして返す

    Args:
        data: ネストされたアイテムリスト
        valid_categories: 有効なカテゴリーのセット

    Returns:
        処理済みの値のリスト
    """
    result = []
    for sublist in data:
        for item in sublist:
            if (item['type'] == 'active' and
                item['value'] > 100 and
                item['category'] in valid_categories):
                result.append(item['value'] * 2)
    return result

# シンプルな場合は内包表記が推奨される
def get_active_user_ids(users: List[Dict[str, Any]]) -> List[int]:
    """アクティブなユーザーのIDリストを返す"""
    return [user['id'] for user in users if user['is_active']]
```

## PEP 8スタイルガイドの遵守

PEP 8はPythonの公式スタイルガイドです。一貫したコーディングスタイルは、チーム全体の生産性向上に寄与します。

### チェックポイント

```python
# 良い例: PEP 8に準拠したコード

# インポートは標準ライブラリ、サードパーティ、ローカルの順
import os
import sys
from typing import List, Optional

import requests
from flask import Flask

from myapp.models import User
from myapp.utils import validate_email

# 定数は大文字のスネークケース
MAX_RETRY_COUNT = 3
DEFAULT_TIMEOUT = 30

# クラス名はキャメルケース
class UserRepository:
    """ユーザーデータのリポジトリ"""

    # メソッド名はスネークケース
    def get_active_users(self, limit: int = 100) -> List[User]:
        """アクティブなユーザーを取得する

        Args:
            limit: 取得する最大件数

        Returns:
            ユーザーのリスト
        """
        # 1行は79文字以内（ドキュメント文字列は72文字以内）
        return self._query_users(
            status='active',
            limit=limit,
            order_by='created_at'
        )
```

## Pythonレビューチェックリスト

レビュー時に確認すべき項目をチェックリストにまとめました。

### 型安全性

- [ ] 関数の引数と戻り値に型ヒントが付与されているか
- [ ] mypyやPyrightなどの型チェッカーが通るか
- [ ] Optionalが適切に使用されているか（Noneを返す可能性がある場合）

### コーディング規約

- [ ] PEP 8に準拠しているか（flake8やblackでチェック）
- [ ] 変数名・関数名・クラス名が命名規則に従っているか
- [ ] インポート文が適切に整理されているか

### リソース管理

- [ ] ファイル操作でwith文が使用されているか
- [ ] データベース接続が確実にクローズされるか
- [ ] 一時ファイルやディレクトリが適切にクリーンアップされるか

### 例外処理

- [ ] 適切な粒度で例外を捕捉しているか（bare exceptを避ける）
- [ ] カスタム例外クラスが適切に定義されているか
- [ ] 例外発生時のログが適切に出力されているか

### パフォーマンス

- [ ] リスト内包表記が適切に使用されているか
- [ ] ジェネレータが使えるケースで無駄なリスト生成をしていないか
- [ ] 不要な再計算が行われていないか

### ドキュメント

- [ ] docstringがGoogle形式やNumPy形式で記載されているか
- [ ] 関数の目的、引数、戻り値、例外が明確に説明されているか
- [ ] 複雑なロジックにコメントが付与されているか

## 実践例: APIエンドポイントのレビュー

実際のコードレビューの例を見てみましょう。

### レビュー対象コード

```python
from typing import Dict, Any, Optional
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel, validator
import logging

logger = logging.getLogger(__name__)
app = FastAPI()

class UserCreate(BaseModel):
    """ユーザー作成リクエスト"""
    username: str
    email: str
    age: int

    @validator('email')
    def validate_email(cls, v: str) -> str:
        """メールアドレスの妥当性検証"""
        if '@' not in v:
            raise ValueError('Invalid email format')
        return v.lower()

    @validator('age')
    def validate_age(cls, v: int) -> int:
        """年齢の妥当性検証"""
        if v < 0 or v > 150:
            raise ValueError('Age must be between 0 and 150')
        return v

@app.post("/users/", status_code=status.HTTP_201_CREATED)
async def create_user(user: UserCreate) -> Dict[str, Any]:
    """新規ユーザーを作成する

    Args:
        user: ユーザー作成情報

    Returns:
        作成されたユーザー情報

    Raises:
        HTTPException: ユーザー作成に失敗した場合
    """
    try:
        # データベースに保存
        with get_db_connection() as conn:
            cursor = conn.cursor()
            cursor.execute(
                "INSERT INTO users (username, email, age) VALUES (?, ?, ?)",
                (user.username, user.email, user.age)
            )
            conn.commit()
            user_id = cursor.lastrowid

        logger.info(f"User created: {user_id}")

        return {
            "id": user_id,
            "username": user.username,
            "email": user.email,
            "age": user.age
        }

    except ValueError as e:
        logger.warning(f"Invalid user data: {e}")
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e)
        )
    except Exception as e:
        logger.error(f"Failed to create user: {e}")
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Failed to create user"
        )
```

このコードは型ヒント、バリデーション、リソース管理、例外処理のベストプラクティスを実践しています。

## まとめ

本章ではPythonコードレビューの重要なポイントを解説しました。

主なポイント:
- 型ヒントを活用して型安全性を向上させる
- with文でリソース管理を確実に行う
- 具体的な例外を捕捉し、適切にハンドリングする
- PEP 8スタイルガイドを遵守する
- リスト内包表記は可読性を考慮して使う

次章では、Swiftコードレビューのガイドラインについて学びます。

## 参考文献

- [PEP 8 -- Style Guide for Python Code](https://peps.python.org/pep-0008/)
- [PEP 484 -- Type Hints](https://peps.python.org/pep-0484/)
- [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html)
- [Python Documentation - Context Managers](https://docs.python.org/3/library/contextlib.html)
- [Real Python - Type Checking](https://realpython.com/python-type-checking/)
