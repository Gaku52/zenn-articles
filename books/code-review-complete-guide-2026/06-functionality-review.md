---
title: "第6章 機能性レビュー"
---

# 第6章 機能性レビュー

機能性レビューは、コードが意図した通りに動作し、要件を満たしているかを確認する最も基本的かつ重要なレビュー観点です。本章では、要件充足のチェック、エッジケースの発見、エラーハンドリングの検証など、機能性レビューの具体的な手法を学びます。

## 機能性レビューの目的

機能性レビューでは、以下の3つの主要な観点でコードを評価します。

```typescript
interface FunctionalityReview {
  // 要件充足の確認
  requirementsSatisfaction: {
    allRequirementsMet: boolean;
    acceptanceCriteriaMet: boolean;
    userStoryAlignment: boolean;
  };

  // 正確性の確認
  correctness: {
    logicCorrectness: boolean;
    edgeCasesHandled: boolean;
    errorHandlingComplete: boolean;
  };

  // データ整合性の確認
  dataIntegrity: {
    validationImplemented: boolean;
    consistencyMaintained: boolean;
    transactionsSafe: boolean;
  };
}
```

参考: [Google Engineering Practices - Functionality](https://google.github.io/eng-practices/review/reviewer/looking-for.html)

## 要件充足のチェック

### ユーザーストーリーとの照合

```typescript
// ユーザーストーリー例:
// "As a user, I want to filter products by price range
//  so that I can find products within my budget"

// 悪い例: 要件を満たしていない
class ProductFilter {
  filterByPrice(products: Product[], maxPrice: number): Product[] {
    return products.filter(p => p.price <= maxPrice);
    // 問題: 最小価格の指定ができない
    // 問題: 価格範囲ではなく上限のみ
  }
}

// 良い例: 要件を正確に満たしている
class ProductFilter {
  filterByPriceRange(
    products: Product[],
    minPrice: number,
    maxPrice: number
  ): Product[] {
    // 入力バリデーション
    if (minPrice < 0 || maxPrice < 0) {
      throw new Error('価格は0以上である必要があります');
    }

    if (minPrice > maxPrice) {
      throw new Error('最小価格は最大価格以下である必要があります');
    }

    return products.filter(
      p => p.price >= minPrice && p.price <= maxPrice
    );
  }
}
```

### 受け入れ基準の確認

```typescript
// 受け入れ基準:
// 1. 最小価格と最大価格の両方を指定できる
// 2. 価格が範囲内の商品のみ返す
// 3. 不正な価格範囲の場合はエラーを返す
// 4. 空の商品リストでも正常動作する

describe('ProductFilter.filterByPriceRange', () => {
  const products = [
    { id: 1, name: 'Product A', price: 100 },
    { id: 2, name: 'Product B', price: 200 },
    { id: 3, name: 'Product C', price: 300 },
  ];

  // 基準1: 範囲指定が可能
  test('範囲内の商品を正しくフィルタリング', () => {
    const filter = new ProductFilter();
    const result = filter.filterByPriceRange(products, 150, 250);

    expect(result).toHaveLength(1);
    expect(result[0].name).toBe('Product B');
  });

  // 基準2: 境界値を含む
  test('境界値を含む', () => {
    const filter = new ProductFilter();
    const result = filter.filterByPriceRange(products, 100, 300);

    expect(result).toHaveLength(3);
  });

  // 基準3: 不正な範囲でエラー
  test('不正な価格範囲でエラー', () => {
    const filter = new ProductFilter();

    expect(() => {
      filter.filterByPriceRange(products, 300, 100);
    }).toThrow('最小価格は最大価格以下である必要があります');
  });

  // 基準4: 空リストでも動作
  test('空リストでも正常動作', () => {
    const filter = new ProductFilter();
    const result = filter.filterByPriceRange([], 100, 200);

    expect(result).toEqual([]);
  });
});
```

## エッジケースの発見

### 空データとnull/undefinedの処理

```python
# 悪い例: エッジケースを考慮していない
def calculate_average(numbers):
    total = sum(numbers)
    return total / len(numbers)
    # 問題: 空リストで ZeroDivisionError
    # 問題: None が渡された場合の対処なし

# 良い例: エッジケースを適切に処理
from typing import List, Optional

def calculate_average(numbers: Optional[List[float]]) -> Optional[float]:
    """
    数値リストの平均を計算する

    Args:
        numbers: 数値のリスト（Noneまたは空リスト可）

    Returns:
        平均値、またはNone（リストが空またはNoneの場合）

    Examples:
        >>> calculate_average([1, 2, 3])
        2.0
        >>> calculate_average([])
        None
        >>> calculate_average(None)
        None
    """
    # Noneチェック
    if numbers is None:
        return None

    # 空リストチェック
    if len(numbers) == 0:
        return None

    # 正常処理
    return sum(numbers) / len(numbers)
```

### 境界値テスト

```swift
// Swift例: 配列インデックスの境界値チェック
struct ArrayUtils {
    // 悪い例: 境界チェックなし
    static func getElementBad<T>(_ array: [T], at index: Int) -> T {
        return array[index]
        // 問題: index < 0 でクラッシュ
        // 問題: index >= array.count でクラッシュ
    }

    // 良い例: 境界値を適切に処理
    static func getElement<T>(_ array: [T], at index: Int) -> T? {
        // 負のインデックスチェック
        guard index >= 0 else {
            return nil
        }

        // 上限チェック
        guard index < array.count else {
            return nil
        }

        return array[index]
    }
}

// テスト
import XCTest

class ArrayUtilsTests: XCTestCase {
    let testArray = [1, 2, 3, 4, 5]

    // 正常系
    func testValidIndex() {
        XCTAssertEqual(ArrayUtils.getElement(testArray, at: 0), 1)
        XCTAssertEqual(ArrayUtils.getElement(testArray, at: 4), 5)
    }

    // 境界値: 負のインデックス
    func testNegativeIndex() {
        XCTAssertNil(ArrayUtils.getElement(testArray, at: -1))
    }

    // 境界値: 配列長を超える
    func testOutOfBoundsIndex() {
        XCTAssertNil(ArrayUtils.getElement(testArray, at: 5))
        XCTAssertNil(ArrayUtils.getElement(testArray, at: 100))
    }

    // エッジケース: 空配列
    func testEmptyArray() {
        let empty: [Int] = []
        XCTAssertNil(ArrayUtils.getElement(empty, at: 0))
    }
}
```

### 大量データの処理

```go
// Go例: 大量データ処理のメモリ効率

// 悪い例: 全データをメモリに読み込む
func ProcessFilesBad(filenames []string) ([]Result, error) {
    var allData []Data

    // 問題: 大量ファイルで OutOfMemory の可能性
    for _, filename := range filenames {
        data, err := readFile(filename)
        if err != nil {
            return nil, err
        }
        allData = append(allData, data...)
    }

    return processData(allData), nil
}

// 良い例: ストリーミング処理
func ProcessFiles(filenames []string) ([]Result, error) {
    results := make([]Result, 0, len(filenames))

    // ファイルごとに処理し、メモリを効率的に使用
    for _, filename := range filenames {
        data, err := readFile(filename)
        if err != nil {
            return nil, fmt.Errorf("failed to read %s: %w", filename, err)
        }

        // データを処理後、すぐに解放
        result := processData(data)
        results = append(results, result)

        // データは自動的にGC対象になる
    }

    return results, nil
}

// さらに良い例: チャネルを使った並行処理
func ProcessFilesConcurrent(filenames []string) ([]Result, error) {
    const maxConcurrent = 10
    semaphore := make(chan struct{}, maxConcurrent)
    resultChan := make(chan Result, len(filenames))
    errorChan := make(chan error, len(filenames))

    var wg sync.WaitGroup

    for _, filename := range filenames {
        wg.Add(1)

        go func(fname string) {
            defer wg.Done()

            // 並行実行数を制限
            semaphore <- struct{}{}
            defer func() { <-semaphore }()

            data, err := readFile(fname)
            if err != nil {
                errorChan <- fmt.Errorf("failed to read %s: %w", fname, err)
                return
            }

            result := processData(data)
            resultChan <- result
        }(filename)
    }

    // 全ゴルーチンの完了を待つ
    wg.Wait()
    close(resultChan)
    close(errorChan)

    // エラーチェック
    if len(errorChan) > 0 {
        return nil, <-errorChan
    }

    // 結果を収集
    results := make([]Result, 0, len(filenames))
    for result := range resultChan {
        results = append(results, result)
    }

    return results, nil
}
```

## エラーハンドリングの検証

### エラーケースの網羅性

```typescript
// 悪い例: エラーハンドリング不足
async function createUser(email: string, password: string): Promise<User> {
    const user = await db.users.create({ email, password });
    return user;
    // 問題: メールアドレス重複時の処理なし
    // 問題: データベース接続エラーの処理なし
    // 問題: バリデーションエラーの処理なし
}

// 良い例: 包括的なエラーハンドリング
class UserCreationError extends Error {
    constructor(
        message: string,
        public readonly code: string,
        public readonly details?: unknown
    ) {
        super(message);
        this.name = 'UserCreationError';
    }
}

async function createUser(
    email: string,
    password: string
): Promise<User> {
    // 入力バリデーション
    if (!email || !isValidEmail(email)) {
        throw new UserCreationError(
            'Invalid email address',
            'INVALID_EMAIL',
            { email }
        );
    }

    if (!password || password.length < 8) {
        throw new UserCreationError(
            'Password must be at least 8 characters',
            'INVALID_PASSWORD'
        );
    }

    try {
        // 重複チェック
        const existingUser = await db.users.findByEmail(email);
        if (existingUser) {
            throw new UserCreationError(
                'Email already registered',
                'EMAIL_DUPLICATE',
                { email }
            );
        }

        // パスワードのハッシュ化
        const hashedPassword = await hashPassword(password);

        // ユーザー作成
        const user = await db.users.create({
            email,
            password: hashedPassword,
        });

        return user;

    } catch (error) {
        // UserCreationErrorはそのまま再スロー
        if (error instanceof UserCreationError) {
            throw error;
        }

        // データベースエラー
        if (error instanceof DatabaseError) {
            throw new UserCreationError(
                'Failed to create user due to database error',
                'DATABASE_ERROR',
                { originalError: error.message }
            );
        }

        // 予期しないエラー
        throw new UserCreationError(
            'Unexpected error during user creation',
            'UNKNOWN_ERROR',
            { originalError: error }
        );
    }
}
```

### エラーメッセージの明確性

```python
# 悪い例: 不明確なエラーメッセージ
def withdraw(account_id: str, amount: float) -> None:
    account = get_account(account_id)
    if account.balance < amount:
        raise ValueError("Error")  # 何が問題か不明
    account.balance -= amount

# 良い例: 明確で実用的なエラーメッセージ
class InsufficientFundsError(Exception):
    """残高不足エラー"""
    def __init__(self, account_id: str, balance: float, requested: float):
        self.account_id = account_id
        self.balance = balance
        self.requested = requested

        message = (
            f"Insufficient funds in account {account_id}. "
            f"Balance: ${balance:.2f}, "
            f"Requested: ${requested:.2f}, "
            f"Shortfall: ${requested - balance:.2f}"
        )
        super().__init__(message)

def withdraw(account_id: str, amount: float) -> None:
    """
    口座から金額を引き出す

    Args:
        account_id: 口座ID
        amount: 引き出し金額

    Raises:
        ValueError: 引き出し金額が不正な場合
        InsufficientFundsError: 残高不足の場合
        AccountNotFoundError: 口座が存在しない場合
    """
    if amount <= 0:
        raise ValueError(
            f"Withdrawal amount must be positive, got: ${amount:.2f}"
        )

    account = get_account(account_id)

    if account.balance < amount:
        raise InsufficientFundsError(
            account_id=account_id,
            balance=account.balance,
            requested=amount
        )

    account.balance -= amount
    log_transaction(account_id, 'withdrawal', amount)
```

### リカバリー処理の実装

```typescript
// リトライロジックの実装例
interface RetryOptions {
  maxAttempts: number;
  delayMs: number;
  backoffMultiplier: number;
  retryableErrors: Array<new (...args: any[]) => Error>;
}

async function withRetry<T>(
  operation: () => Promise<T>,
  options: RetryOptions
): Promise<T> {
  let lastError: Error;
  let delay = options.delayMs;

  for (let attempt = 1; attempt <= options.maxAttempts; attempt++) {
    try {
      return await operation();

    } catch (error) {
      lastError = error as Error;

      // リトライ可能なエラーかチェック
      const isRetryable = options.retryableErrors.some(
        ErrorClass => error instanceof ErrorClass
      );

      if (!isRetryable) {
        throw error;
      }

      // 最後の試行なら諦める
      if (attempt === options.maxAttempts) {
        break;
      }

      // 指数バックオフで待機
      console.log(
        `Attempt ${attempt} failed, retrying in ${delay}ms...`
      );
      await sleep(delay);
      delay *= options.backoffMultiplier;
    }
  }

  throw new Error(
    `Operation failed after ${options.maxAttempts} attempts: ${lastError.message}`
  );
}

// 使用例
class NetworkError extends Error {}
class TimeoutError extends Error {}

async function fetchUserData(userId: string): Promise<User> {
  return withRetry(
    async () => {
      const response = await fetch(`/api/users/${userId}`);

      if (!response.ok) {
        if (response.status >= 500) {
          throw new NetworkError('Server error');
        }
        throw new Error(`HTTP ${response.status}`);
      }

      return response.json();
    },
    {
      maxAttempts: 3,
      delayMs: 1000,
      backoffMultiplier: 2,
      retryableErrors: [NetworkError, TimeoutError],
    }
  );
}
```

## データ整合性の確認

### バリデーションの実装

```typescript
// Zodを使った堅牢なバリデーション例
import { z } from 'zod';

// スキーマ定義
const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  age: z.number().int().min(0).max(150),
  role: z.enum(['admin', 'user', 'guest']),
  createdAt: z.date(),
  profile: z.object({
    firstName: z.string().min(1).max(50),
    lastName: z.string().min(1).max(50),
    bio: z.string().max(500).optional(),
  }),
  tags: z.array(z.string()).max(10),
});

type User = z.infer<typeof UserSchema>;

// バリデーション関数
function validateUser(data: unknown): User {
  try {
    return UserSchema.parse(data);
  } catch (error) {
    if (error instanceof z.ZodError) {
      const messages = error.errors.map(
        e => `${e.path.join('.')}: ${e.message}`
      );
      throw new ValidationError(
        'User validation failed',
        messages
      );
    }
    throw error;
  }
}

// 使用例
async function createUser(input: unknown): Promise<User> {
  // バリデーション
  const validatedData = validateUser(input);

  // ビジネスロジック
  const user = await db.users.create(validatedData);

  return user;
}
```

### トランザクションの適切な使用

```python
# SQLAlchemyでのトランザクション管理例
from sqlalchemy.orm import Session
from contextlib import contextmanager

@contextmanager
def transaction_scope(session: Session):
    """トランザクションスコープを提供するコンテキストマネージャー"""
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

# 悪い例: トランザクション管理が不適切
def transfer_money_bad(from_account_id: str, to_account_id: str, amount: float):
    from_account = session.query(Account).get(from_account_id)
    to_account = session.query(Account).get(to_account_id)

    from_account.balance -= amount
    session.commit()  # 問題: 途中でコミット

    to_account.balance += amount
    session.commit()  # 問題: 2つの操作が分離している

# 良い例: トランザクションで原子性を保証
def transfer_money(
    session: Session,
    from_account_id: str,
    to_account_id: str,
    amount: float
) -> None:
    """
    口座間で送金を行う（アトミックな操作）

    Args:
        session: データベースセッション
        from_account_id: 送金元口座ID
        to_account_id: 送金先口座ID
        amount: 送金額

    Raises:
        ValueError: 金額が不正な場合
        InsufficientFundsError: 残高不足の場合
        AccountNotFoundError: 口座が存在しない場合
    """
    with transaction_scope(session):
        # 金額バリデーション
        if amount <= 0:
            raise ValueError(f"Amount must be positive: {amount}")

        # 口座の取得（行ロック）
        from_account = session.query(Account).with_for_update().get(from_account_id)
        to_account = session.query(Account).with_for_update().get(to_account_id)

        if not from_account:
            raise AccountNotFoundError(from_account_id)
        if not to_account:
            raise AccountNotFoundError(to_account_id)

        # 残高チェック
        if from_account.balance < amount:
            raise InsufficientFundsError(
                from_account_id,
                from_account.balance,
                amount
            )

        # 送金処理（両方成功するか、両方失敗する）
        from_account.balance -= amount
        to_account.balance += amount

        # トランザクションログ
        transaction = Transaction(
            from_account=from_account_id,
            to_account=to_account_id,
            amount=amount,
            timestamp=datetime.now()
        )
        session.add(transaction)

        # ここでコミット（with文の終了時）
```

## 機能性レビューチェックリスト

```markdown
## 要件充足
- [ ] 全ての要件を満たしているか
- [ ] 受け入れ基準を満たしているか
- [ ] ユーザーストーリーに沿っているか
- [ ] 仕様書と一致しているか

## エッジケース
- [ ] null/undefined/nilの処理が適切か
- [ ] 空データ（空配列、空文字列）の処理があるか
- [ ] 境界値（最小値、最大値）が正しく処理されるか
- [ ] 大量データでメモリ不足にならないか
- [ ] 不正な入力に対して適切に対応しているか

## エラーハンドリング
- [ ] 全てのエラーケースを処理しているか
- [ ] エラーメッセージが明確で実用的か
- [ ] エラーログが適切に記録されるか
- [ ] ユーザーに分かりやすいエラー表示か
- [ ] リカバリー処理（リトライ等）が実装されているか

## データ整合性
- [ ] 入力バリデーションが実装されているか
- [ ] データの一貫性が保たれるか
- [ ] トランザクション処理が適切か
- [ ] 並行処理での競合が考慮されているか
- [ ] ロールバック処理が正しく動作するか

## テストカバレッジ
- [ ] 正常系のテストがあるか
- [ ] 異常系のテストがあるか
- [ ] エッジケースのテストがあるか
- [ ] 境界値テストがあるか
```

参考: [Microsoft Code Review Checklist](https://docs.microsoft.com/en-us/azure/devops/learn/devops-at-microsoft/code-reviews-not-primarily-finding-bugs)

## まとめ

本章では、機能性レビューの具体的な手法を学びました。

- **要件充足**: ユーザーストーリーと受け入れ基準の確認
- **エッジケース**: 空データ、境界値、大量データの処理
- **エラーハンドリング**: 包括的なエラー処理とリカバリー
- **データ整合性**: バリデーションとトランザクション管理

次章では、設計・アーキテクチャレビューについて学びます。SOLID原則、デザインパターン、依存関係の評価など、より高度なレビュー観点を習得しましょう。
