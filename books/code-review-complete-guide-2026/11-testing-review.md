---
title: "第11章 テストレビュー"
---

# 第11章 テストレビュー

本章ではテストレビューについて、実践的な観点から詳しく解説します。テストコードは、プロダクトコードの品質を保証する重要な要素であり、適切なテストがなければ、リファクタリングや機能追加時に予期しない不具合が発生するリスクが高まります。

## 概要

テストレビューは、テストコードの品質、網羅性、保守性を評価するプロセスです。本章では、テストカバレッジの適切性、テストケースの網羅性（正常系・異常系・境界値）、モック・スタブの適切な使用、テストの保守性など、具体的なチェックポイントとコード例を通じて、効果的なレビュー方法を学びます。

## テストカバレッジの適切性

### カバレッジの指標と目標設定

テストカバレッジは、コードのどの部分がテストされているかを示す指標です。一般的には、以下の指標が使用されます。

- ライン（行）カバレッジ: テストで実行されたコード行の割合
- ブランチカバレッジ: すべての分岐（if文、switch文等）がテストされた割合
- 関数カバレッジ: テストで呼び出された関数の割合

カバレッジの目標値は、プロジェクトの性質により異なりますが、一般的には80%以上が推奨されます。ただし、カバレッジが高いことと、テストの品質が高いことは必ずしも一致しません。

```typescript
// カバレッジ設定例（Jest）
// jest.config.js
module.exports = {
  collectCoverage: true,
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  },
  coveragePathIgnorePatterns: [
    '/node_modules/',
    '/test/',
    '/dist/'
  ]
};
```

レビュー時には、カバレッジレポートを確認し、重要なビジネスロジックがテストされているか、カバレッジの低い部分がないかチェックしましょう。

### 意味のあるテストケース

```typescript
// 悪い例: カバレッジのためだけのテスト
describe('calculateDiscount', () => {
  it('should execute without error', () => {
    calculateDiscount(100, 0.1);
    // アサーションがない
  });
});

// 良い例: 期待される結果を検証
describe('calculateDiscount', () => {
  it('should calculate 10% discount correctly', () => {
    const result = calculateDiscount(100, 0.1);
    expect(result).toBe(90);
  });

  it('should handle zero discount', () => {
    const result = calculateDiscount(100, 0);
    expect(result).toBe(100);
  });

  it('should throw error for negative price', () => {
    expect(() => calculateDiscount(-100, 0.1)).toThrow('Price must be positive');
  });
});
```

単にカバレッジを上げるだけでなく、実際のバグを検出できるテストを書くことが重要です。

## テストケースの網羅性

### 正常系・異常系・境界値のテスト

包括的なテストには、正常系（Happy Path）、異常系（Error Cases）、境界値（Edge Cases）の3つの観点が必要です。

```python
# テスト対象の関数
def divide(a: float, b: float) -> float:
    if b == 0:
        raise ValueError("Division by zero")
    if not isinstance(a, (int, float)) or not isinstance(b, (int, float)):
        raise TypeError("Arguments must be numbers")
    return a / b

# pytest によるテスト例
import pytest

class TestDivide:
    # 正常系テスト
    def test_divide_positive_numbers(self):
        assert divide(10, 2) == 5.0

    def test_divide_negative_numbers(self):
        assert divide(-10, 2) == -5.0

    def test_divide_with_float(self):
        assert divide(10.5, 2) == 5.25

    # 異常系テスト
    def test_divide_by_zero(self):
        with pytest.raises(ValueError, match="Division by zero"):
            divide(10, 0)

    def test_divide_with_invalid_type(self):
        with pytest.raises(TypeError, match="Arguments must be numbers"):
            divide("10", 2)

    # 境界値テスト
    def test_divide_very_small_numbers(self):
        result = divide(0.000001, 0.000001)
        assert result == pytest.approx(1.0)

    def test_divide_very_large_numbers(self):
        result = divide(1e308, 1e308)
        assert result == pytest.approx(1.0)
```

レビュー時には、以下の点を確認しましょう。

- [ ] 正常系のテストケースが十分にあるか
- [ ] エラーハンドリングが適切にテストされているか
- [ ] 境界値（0、負の数、最大値、最小値等）がテストされているか
- [ ] nullやundefinedの処理がテストされているか

### 同値分割と境界値分析

```typescript
// 年齢による料金計算のテスト例
function getTicketPrice(age: number): number {
  if (age < 0) throw new Error('Invalid age');
  if (age < 12) return 500;  // 子供料金
  if (age < 65) return 1000; // 大人料金
  return 700;                // シニア料金
}

describe('getTicketPrice', () => {
  // 境界値テスト
  describe('boundary values', () => {
    it('should return child price for age 11', () => {
      expect(getTicketPrice(11)).toBe(500);
    });

    it('should return adult price for age 12', () => {
      expect(getTicketPrice(12)).toBe(1000);
    });

    it('should return adult price for age 64', () => {
      expect(getTicketPrice(64)).toBe(1000);
    });

    it('should return senior price for age 65', () => {
      expect(getTicketPrice(65)).toBe(700);
    });
  });

  // 同値分割テスト
  describe('equivalence partitioning', () => {
    it('should return child price for typical child age', () => {
      expect(getTicketPrice(7)).toBe(500);
    });

    it('should return adult price for typical adult age', () => {
      expect(getTicketPrice(30)).toBe(1000);
    });

    it('should return senior price for typical senior age', () => {
      expect(getTicketPrice(70)).toBe(700);
    });
  });

  // 異常系テスト
  describe('error cases', () => {
    it('should throw error for negative age', () => {
      expect(() => getTicketPrice(-1)).toThrow('Invalid age');
    });
  });
});
```

境界値と代表値の両方をテストすることで、効率的かつ網羅的なテストケースを設計できます。

## モック・スタブの適切な使用

### モックとスタブの使い分け

モックとスタブは、外部依存を分離してテストするための技術です。

- スタブ: テスト対象に特定の値を返す代替実装
- モック: 呼び出しを記録し、期待される呼び出しがあったか検証

```typescript
// テスト対象のサービス
class UserService {
  constructor(private userRepository: UserRepository) {}

  async createUser(userData: UserData): Promise<User> {
    const user = await this.userRepository.save(userData);
    await this.emailService.sendWelcomeEmail(user.email);
    return user;
  }
}

// Jest によるモック・スタブの例
describe('UserService', () => {
  let userService: UserService;
  let userRepository: jest.Mocked<UserRepository>;
  let emailService: jest.Mocked<EmailService>;

  beforeEach(() => {
    // スタブ: 固定値を返す
    userRepository = {
      save: jest.fn().mockResolvedValue({
        id: '123',
        name: 'Test User',
        email: 'test@example.com'
      })
    } as any;

    // モック: 呼び出しを検証
    emailService = {
      sendWelcomeEmail: jest.fn().mockResolvedValue(undefined)
    } as any;

    userService = new UserService(userRepository, emailService);
  });

  it('should create user and send welcome email', async () => {
    const userData = { name: 'Test User', email: 'test@example.com' };

    const user = await userService.createUser(userData);

    // スタブからの戻り値を検証
    expect(user.id).toBe('123');

    // モックの呼び出しを検証
    expect(emailService.sendWelcomeEmail).toHaveBeenCalledWith('test@example.com');
    expect(emailService.sendWelcomeEmail).toHaveBeenCalledTimes(1);
  });
});
```

レビュー時には、以下の点を確認しましょう。

- [ ] モック・スタブが過度に使用されていないか
- [ ] モックの期待値が適切に設定されているか
- [ ] 外部API呼び出しが適切にモック化されているか
- [ ] データベースアクセスがテスト環境で分離されているか

### モックの過度な使用を避ける

```typescript
// 悪い例: すべてをモック化
describe('OrderService', () => {
  it('should calculate total price', () => {
    const calculator = {
      add: jest.fn().mockReturnValue(100),
      multiply: jest.fn().mockReturnValue(200)
    };

    const service = new OrderService(calculator);
    const result = service.calculateTotal();

    expect(result).toBe(200);
  });
});

// 良い例: 必要最小限のモック
describe('OrderService', () => {
  it('should calculate total price', () => {
    const service = new OrderService();
    const items = [
      { price: 100, quantity: 2 },
      { price: 50, quantity: 1 }
    ];

    const result = service.calculateTotal(items);

    expect(result).toBe(250);
  });
});
```

単純な計算ロジックまでモック化すると、テストが実装に密結合し、保守性が低下します。

## テストの保守性と可読性

### AAA（Arrange-Act-Assert）パターン

テストは、Arrange（準備）、Act（実行）、Assert（検証）の3つのセクションに分けて記述することで、可読性が向上します。

```typescript
describe('ShoppingCart', () => {
  it('should add item to cart', () => {
    // Arrange: テスト対象と必要なデータを準備
    const cart = new ShoppingCart();
    const item = { id: '1', name: 'Book', price: 1000 };

    // Act: テスト対象の操作を実行
    cart.addItem(item);

    // Assert: 期待される結果を検証
    expect(cart.getItems()).toHaveLength(1);
    expect(cart.getTotal()).toBe(1000);
  });
});
```

### テストデータのファクトリー化

```typescript
// テストデータファクトリー
class UserFactory {
  static create(overrides: Partial<User> = {}): User {
    return {
      id: '123',
      name: 'Test User',
      email: 'test@example.com',
      role: 'user',
      ...overrides
    };
  }

  static createAdmin(overrides: Partial<User> = {}): User {
    return this.create({ role: 'admin', ...overrides });
  }
}

// 使用例
describe('UserService', () => {
  it('should allow admin to delete user', () => {
    const admin = UserFactory.createAdmin();
    const user = UserFactory.create();

    const result = userService.deleteUser(admin, user.id);

    expect(result).toBe(true);
  });

  it('should not allow regular user to delete user', () => {
    const user = UserFactory.create();
    const targetUser = UserFactory.create({ id: '456' });

    expect(() => userService.deleteUser(user, targetUser.id))
      .toThrow('Unauthorized');
  });
});
```

テストデータをファクトリー化することで、テストコードの重複を減らし、保守性を向上させます。

## E2Eテストの必要性判断

### ユニットテストとE2Eテストのバランス

テストピラミッドの考え方に基づき、ユニットテストを土台に、統合テスト、E2Eテストと層を重ねることが推奨されます。

```typescript
// E2Eテストの例（Playwright）
import { test, expect } from '@playwright/test';

test.describe('User Registration', () => {
  test('should register new user successfully', async ({ page }) => {
    // ページにアクセス
    await page.goto('/register');

    // フォームに入力
    await page.fill('input[name="name"]', 'Test User');
    await page.fill('input[name="email"]', 'test@example.com');
    await page.fill('input[name="password"]', 'SecurePassword123');

    // 送信ボタンをクリック
    await page.click('button[type="submit"]');

    // リダイレクトを確認
    await expect(page).toHaveURL('/dashboard');

    // ウェルカムメッセージを確認
    await expect(page.locator('h1')).toContainText('Welcome, Test User');
  });

  test('should show validation error for invalid email', async ({ page }) => {
    await page.goto('/register');

    await page.fill('input[name="email"]', 'invalid-email');
    await page.click('button[type="submit"]');

    await expect(page.locator('.error-message'))
      .toContainText('Invalid email format');
  });
});
```

E2Eテストは、以下の場合に特に有効です。

- クリティカルなユーザーフロー（決済、ログイン等）
- 複数のコンポーネント・サービスが連携する処理
- UIの動作確認が必要な場合

レビュー時には、E2Eテストの範囲が適切か、実行時間が長すぎないかを確認しましょう。

## テストレビューチェックリスト

レビュー時に確認すべき項目をチェックリスト形式でまとめます。

- [ ] テストカバレッジが目標値（80%以上推奨）を満たしているか
- [ ] 重要なビジネスロジックがテストされているか
- [ ] 正常系、異常系、境界値のテストケースが含まれているか
- [ ] モック・スタブが適切に使用されているか（過度に使用していないか）
- [ ] テストが独立しており、実行順序に依存していないか
- [ ] AAA（Arrange-Act-Assert）パターンが守られているか
- [ ] テストケース名が明確で、何をテストしているか分かりやすいか
- [ ] テストデータがハードコードされず、ファクトリー等で管理されているか
- [ ] 非同期処理のテストが適切に実装されているか（async/await、done等）
- [ ] E2Eテストがクリティカルなフローをカバーしているか
- [ ] テストの実行時間が妥当か（長すぎるテストは改善が必要）
- [ ] フレイキー（不安定）なテストがないか

## まとめ

本章では、テストレビューにおける重要なポイントを解説しました。

主なポイント:
- テストカバレッジの適切性と意味のあるテストケース
- 正常系・異常系・境界値を網羅したテスト設計
- モック・スタブの適切な使用と過度な使用の回避
- AAAパターンやファクトリーによるテストの保守性向上
- ユニットテスト、統合テスト、E2Eテストの適切なバランス

品質の高いテストコードは、プロダクトコードの品質を保証し、安心してリファクタリングや機能追加を行える基盤となります。テストレビューを通じて、テストの質を継続的に向上させましょう。

## 参考文献

- [Jest Documentation](https://jestjs.io/docs/getting-started)
- [Pytest Documentation](https://docs.pytest.org/)
- [Playwright Documentation](https://playwright.dev/)
- [Testing Library](https://testing-library.com/docs/)
- [Martin Fowler - Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)
- [Google Testing Blog](https://testing.googleblog.com/)
