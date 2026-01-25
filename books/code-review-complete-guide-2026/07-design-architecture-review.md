---
title: "第7章 設計・アーキテクチャレビュー"
---

# 第7章 設計・アーキテクチャレビュー

優れた設計は、長期的な保守性と拡張性を保証します。本章では、SOLID原則の実践チェック、デザインパターンの適切性、依存関係の評価、レイヤー分離の検証など、設計・アーキテクチャレビューの具体的な手法を学びます。

## SOLID原則の実践チェック

### Single Responsibility Principle (単一責任原則)

```typescript
// 悪い例: 複数の責務を持つクラス
class UserManager {
    // 責務1: ユーザー作成
    async createUser(data: UserData): Promise<User> {
        const user = await this.db.create(data);
        return user;
    }

    // 責務2: メール送信
    async sendWelcomeEmail(user: User): Promise<void> {
        const transport = nodemailer.createTransport(/* config */);
        await transport.sendMail({
            to: user.email,
            subject: 'Welcome!',
            html: '<h1>Welcome</h1>',
        });
    }

    // 責務3: ログ記録
    log(message: string): void {
        console.log(`[${new Date().toISOString()}] ${message}`);
    }

    // 問題: 1つのクラスが3つの異なる責務を持っている
}

// 良い例: 責務を分離
class UserRepository {
    async create(data: UserData): Promise<User> {
        return this.db.create(data);
    }

    async findById(id: string): Promise<User | null> {
        return this.db.findById(id);
    }
}

class EmailService {
    constructor(private transport: MailTransport) {}

    async sendWelcomeEmail(user: User): Promise<void> {
        await this.transport.sendMail({
            to: user.email,
            subject: 'Welcome!',
            template: 'welcome',
            context: { userName: user.name },
        });
    }
}

class Logger {
    log(level: 'info' | 'warn' | 'error', message: string): void {
        const timestamp = new Date().toISOString();
        console.log(`[${timestamp}] [${level.toUpperCase()}] ${message}`);
    }
}

// 使用例
class UserService {
    constructor(
        private userRepository: UserRepository,
        private emailService: EmailService,
        private logger: Logger
    ) {}

    async createUser(data: UserData): Promise<User> {
        try {
            const user = await this.userRepository.create(data);
            this.logger.log('info', `User created: ${user.id}`);

            await this.emailService.sendWelcomeEmail(user);
            this.logger.log('info', `Welcome email sent to: ${user.email}`);

            return user;
        } catch (error) {
            this.logger.log('error', `Failed to create user: ${error.message}`);
            throw error;
        }
    }
}
```

### Open/Closed Principle (開放/閉鎖原則)

```python
# 悪い例: 拡張時に既存コードを変更する必要がある
class PaymentProcessor:
    def process(self, payment_type: str, amount: float) -> bool:
        if payment_type == "credit_card":
            return self._process_credit_card(amount)
        elif payment_type == "paypal":
            return self._process_paypal(amount)
        # 問題: 新しい支払い方法を追加するたびに変更が必要

# 良い例: 拡張に対して開いており、修正に対して閉じている
from abc import ABC, abstractmethod

class PaymentMethod(ABC):
    @abstractmethod
    def process(self, amount: float) -> bool:
        pass

class CreditCardPayment(PaymentMethod):
    def process(self, amount: float) -> bool:
        # クレジットカード決済処理
        return True

class PayPalPayment(PaymentMethod):
    def process(self, amount: float) -> bool:
        # PayPal決済処理
        return True

class PaymentProcessor:
    def __init__(self, payment_method: PaymentMethod):
        self.payment_method = payment_method

    def process_payment(self, amount: float) -> bool:
        return self.payment_method.process(amount)

# 新しい支払い方法を追加（既存コード変更不要）
class CryptoPayment(PaymentMethod):
    def process(self, amount: float) -> bool:
        return True
```

### Dependency Inversion Principle (依存性逆転原則)

```typescript
// 悪い例: 具象クラスに依存
class UserService {
    private db: MySQLDatabase;  // 問題: 具象クラスに直接依存

    constructor() {
        this.db = new MySQLDatabase();
    }
}

// 良い例: 抽象に依存
interface Database {
    connect(): Promise<void>;
    query<T>(sql: string, params?: any[]): Promise<T[]>;
}

class MySQLDatabaseImpl implements Database {
    async connect(): Promise<void> {
        console.log('Connected to MySQL');
    }

    async query<T>(sql: string, params?: any[]): Promise<T[]> {
        return [];
    }
}

class UserServiceGood {
    constructor(private db: Database) {}  // インターフェースに依存

    async getUser(id: string): Promise<User | null> {
        const result = await this.db.query<User>(
            'SELECT * FROM users WHERE id = ?',
            [id]
        );
        return result[0] || null;
    }
}
```

参考: [SOLID Principles - Uncle Bob](https://blog.cleancoder.com/uncle-bob/2020/10/18/Solid-Relevance.html)

## デザインパターンの適切性

### Strategyパターン

```python
from abc import ABC, abstractmethod

class SortStrategy(ABC):
    @abstractmethod
    def sort(self, data: list) -> list:
        pass

class QuickSort(SortStrategy):
    def sort(self, data: list) -> list:
        if len(data) <= 1:
            return data
        pivot = data[len(data) // 2]
        left = [x for x in data if x < pivot]
        middle = [x for x in data if x == pivot]
        right = [x for x in data if x > pivot]
        return self.sort(left) + middle + self.sort(right)

class DataSorter:
    def __init__(self, strategy: SortStrategy):
        self._strategy = strategy

    def sort(self, data: list) -> list:
        return self._strategy.sort(data)
```

### Observerパターン

```typescript
interface Observer<T> {
    update(data: T): void;
}

class UserActivityTracker {
    private observers: Set<Observer<UserActivity>> = new Set();

    attach(observer: Observer<UserActivity>): void {
        this.observers.add(observer);
    }

    notify(activity: UserActivity): void {
        for (const observer of this.observers) {
            observer.update(activity);
        }
    }
}

class ActivityLogger implements Observer<UserActivity> {
    update(activity: UserActivity): void {
        console.log(`[LOG] ${activity.type} at ${activity.timestamp}`);
    }
}
```

参考: [Refactoring Guru - Design Patterns](https://refactoring.guru/design-patterns)

## レイヤー分離の検証

```typescript
// Domain Layer
interface User {
    id: string;
    email: string;
}

interface UserRepository {
    findById(id: string): Promise<User | null>;
}

// Use Case Layer
class GetUserUseCase {
    constructor(private userRepository: UserRepository) {}

    async execute(userId: string): Promise<User> {
        const user = await this.userRepository.findById(userId);
        if (!user) {
            throw new Error(`User not found: ${userId}`);
        }
        return user;
    }
}

// Infrastructure Layer
class PostgresUserRepository implements UserRepository {
    async findById(id: string): Promise<User | null> {
        const result = await this.db.query(
            'SELECT * FROM users WHERE id = $1',
            [id]
        );
        return result.rows[0] || null;
    }
}
```

## 設計レビューチェックリスト

```markdown
## SOLID原則
- [ ] 単一責任原則: 各クラス/モジュールが1つの責務のみ持つか
- [ ] 開放/閉鎖原則: 拡張に対して開いているか
- [ ] 依存性逆転原則: 抽象に依存しているか

## デザインパターン
- [ ] 適切なパターンが使用されているか
- [ ] パターンの過剰適用がないか

## 依存関係
- [ ] 循環依存がないか
- [ ] レイヤー間の依存方向が正しいか
```

参考: [Clean Architecture - Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

## まとめ

本章では、設計・アーキテクチャレビューの具体的な手法を学びました。

- **SOLID原則**: 5つの原則の実践的な適用方法
- **デザインパターン**: Strategy、Observerパターンの活用
- **レイヤー分離**: クリーンアーキテクチャの実践

次章では、可読性・保守性レビューについて学びます。
