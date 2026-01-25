---
title: "第8章 可読性・保守性レビュー"
---

# 第8章 可読性・保守性レビュー

可読性の高いコードは、バグの発見を容易にし、保守コストを大幅に削減します。本章では、命名規則、関数分割、コメント、マジックナンバーの排除など、可読性・保守性レビューの具体的な手法を学びます。

## 命名規則

### 変数名

```typescript
// 悪い例: 略語や意味不明な名前
const d = new Date();  // 何のデータ?
const arr = [];        // 何の配列?
const tmp = getValue(); // 一時的な何?
let x = 10;            // xは何を表す?

// 良い例: 意図が明確な名前
const currentDate = new Date();
const userList: User[] = [];
const fetchedUserData = getUserData();
let maxRetryCount = 10;

// 真偽値は is/has/can で始める
const isValid = checkValidity();
const hasPermission = checkPermission();
const canEdit = checkEditability();

// 配列は複数形
const users = await getUsers();
const products = await getProducts();

// 関数名は動詞で始める
function calculateTotal(items: Item[]): number {
    return items.reduce((sum, item) => sum + item.price, 0);
}

function validateEmail(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}
```

### クラス名とインターフェース名

```python
# 悪い例
class data:  # 小文字、抽象的
    pass

class UserMgr:  # 略語
    pass

# 良い例
class UserRepository:  # 名詞、責務が明確
    def find_by_id(self, user_id: str) -> Optional[User]:
        pass

class EmailService:
    def send_welcome_email(self, user: User) -> None:
        pass

# インターフェース（Protocol）
from typing import Protocol

class Serializable(Protocol):
    def to_dict(self) -> dict:
        ...

    def from_dict(self, data: dict) -> 'Serializable':
        ...
```

### 定数名

```swift
// 悪い例
let max = 100
let timeout = 30

// 良い例
let MAX_RETRY_COUNT = 100
let REQUEST_TIMEOUT_SECONDS = 30
let DEFAULT_PAGE_SIZE = 20

// Swift の命名規則に従う
struct Constants {
    static let maxRetryCount = 100
    static let requestTimeoutSeconds = 30
    static let defaultPageSize = 20
}

// 使用例
if retryCount > Constants.maxRetryCount {
    throw TooManyRetriesError()
}
```

参考: [Google Style Guides](https://google.github.io/styleguide/)

## 関数の適切な分割

### 単一責任の関数

```typescript
// 悪い例: 1つの関数が複数の責務を持つ
async function processOrder(orderId: string) {
    // 注文取得
    const order = await db.orders.findById(orderId);
    if (!order) throw new Error('Order not found');

    // 在庫チェック
    for (const item of order.items) {
        const product = await db.products.findById(item.productId);
        if (product.stock < item.quantity) {
            throw new Error('Out of stock');
        }
    }

    // 在庫減少
    for (const item of order.items) {
        await db.products.update(item.productId, {
            stock: product.stock - item.quantity,
        });
    }

    // 支払い処理
    const payment = await paymentGateway.charge({
        amount: order.total,
        source: order.paymentMethod,
    });

    // メール送信
    await emailService.send({
        to: order.customerEmail,
        subject: 'Order Confirmation',
        body: `Your order #${orderId} has been confirmed`,
    });

    // 注文更新
    await db.orders.update(orderId, { status: 'completed' });

    return order;
}

// 良い例: 責務ごとに関数を分割
class OrderService {
    async processOrder(orderId: string): Promise<Order> {
        const order = await this.getOrder(orderId);
        await this.validateStock(order);
        await this.reduceStock(order);
        const payment = await this.processPayment(order);
        await this.sendConfirmationEmail(order);
        await this.updateOrderStatus(orderId, 'completed');

        return order;
    }

    private async getOrder(orderId: string): Promise<Order> {
        const order = await this.orderRepository.findById(orderId);
        if (!order) {
            throw new OrderNotFoundError(orderId);
        }
        return order;
    }

    private async validateStock(order: Order): Promise<void> {
        for (const item of order.items) {
            const product = await this.productRepository.findById(item.productId);
            if (product.stock < item.quantity) {
                throw new OutOfStockError(product.id, product.name);
            }
        }
    }

    private async reduceStock(order: Order): Promise<void> {
        for (const item of order.items) {
            await this.productRepository.decreaseStock(
                item.productId,
                item.quantity
            );
        }
    }

    private async processPayment(order: Order): Promise<Payment> {
        return this.paymentService.charge({
            amount: order.total,
            source: order.paymentMethod,
        });
    }

    private async sendConfirmationEmail(order: Order): Promise<void> {
        await this.emailService.sendOrderConfirmation(order);
    }

    private async updateOrderStatus(
        orderId: string,
        status: OrderStatus
    ): Promise<void> {
        await this.orderRepository.updateStatus(orderId, status);
    }
}
```

### 関数の長さ

```python
# 悪い例: 100行以上の長い関数
def process_user_data(user_id):
    # 100行以上のコード...
    # 読みにくく、テストも困難
    pass

# 良い例: 適切な長さ（20-50行程度）
def process_user_data(user_id: str) -> ProcessedUserData:
    """ユーザーデータを処理する"""
    user = get_user(user_id)
    validated_data = validate_user_data(user)
    enriched_data = enrich_user_data(validated_data)
    return transform_user_data(enriched_data)

def get_user(user_id: str) -> User:
    """ユーザーを取得する"""
    user = db.users.find_by_id(user_id)
    if not user:
        raise UserNotFoundError(f"User {user_id} not found")
    return user

def validate_user_data(user: User) -> User:
    """ユーザーデータを検証する"""
    if not user.email:
        raise ValidationError("Email is required")
    if not is_valid_email(user.email):
        raise ValidationError(f"Invalid email: {user.email}")
    return user
```

## コメントの書き方

### 良いコメント

```go
// 良い例: Whyを説明するコメント
func processPayment(amount float64) error {
    // タイムアウトを30秒に設定
    // 理由: 決済ゲートウェイのSLAが25秒のため、
    //      余裕を持たせて30秒に設定
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    return gateway.Charge(ctx, amount)
}

// 良い例: 複雑なアルゴリズムの説明
func calculateShippingCost(weight float64, distance float64) float64 {
    // 配送料金の計算式:
    // - 基本料金: 500円
    // - 重量料金: 100円/kg（10kgまで）
    // - 距離料金: 50円/10km
    // - 10kg超過分: 150円/kg

    const baseRate = 500.0
    const weightRatePerKg = 100.0
    const weightThreshold = 10.0
    const excessWeightRate = 150.0
    const distanceRatePerTenKm = 50.0

    cost := baseRate

    // 重量料金の計算
    if weight <= weightThreshold {
        cost += weight * weightRatePerKg
    } else {
        cost += weightThreshold * weightRatePerKg
        cost += (weight - weightThreshold) * excessWeightRate
    }

    // 距離料金の計算
    cost += (distance / 10.0) * distanceRatePerTenKm

    return cost
}
```

### 悪いコメント

```typescript
// 悪い例: Whatを説明するだけのコメント
// ユーザーの年齢を計算する
function calculateAge(birthDate: Date): number {
    // 現在の日付を取得
    const now = new Date();
    // 年の差を計算
    let age = now.getFullYear() - birthDate.getFullYear();
    // 月の差を計算
    const monthDiff = now.getMonth() - birthDate.getMonth();
    // 調整
    if (monthDiff < 0 || (monthDiff === 0 && now.getDate() < birthDate.getDate())) {
        age--;
    }
    // 年齢を返す
    return age;
}

// 良い例: コメント不要（コードが自己説明的）
function calculateAge(birthDate: Date): number {
    const now = new Date();
    let age = now.getFullYear() - birthDate.getFullYear();

    const hasBirthdayOccurred =
        now.getMonth() > birthDate.getMonth() ||
        (now.getMonth() === birthDate.getMonth() &&
         now.getDate() >= birthDate.getDate());

    if (!hasBirthdayOccurred) {
        age--;
    }

    return age;
}
```

### TODOコメント

```swift
// 悪い例: 責任者と期限が不明
// TODO: これを修正する

// 良い例: 責任者、期限、理由が明確
// TODO(@username): 2026-02-01までに実装
// 理由: API v2への移行のため、v1の廃止に合わせて対応が必要
// 関連Issue: #1234
func fetchUserData() {
    // 暫定的な実装
}
```

## マジックナンバーの排除

```python
# 悪い例: マジックナンバー
def calculate_discount(price: float, customer_type: str) -> float:
    if customer_type == "premium":
        return price * 0.8  # 0.8は何?
    elif customer_type == "regular":
        return price * 0.95  # 0.95は何?
    return price

def validate_password(password: str) -> bool:
    return len(password) >= 8  # 8の根拠は?

# 良い例: 定数化
class DiscountRates:
    PREMIUM_CUSTOMER = 0.20  # 20% off
    REGULAR_CUSTOMER = 0.05  # 5% off

class PasswordPolicy:
    MIN_LENGTH = 8  # NIST推奨の最小長
    REQUIRE_SPECIAL_CHAR = True
    REQUIRE_NUMBER = True

def calculate_discount(price: float, customer_type: str) -> float:
    if customer_type == "premium":
        return price * (1 - DiscountRates.PREMIUM_CUSTOMER)
    elif customer_type == "regular":
        return price * (1 - DiscountRates.REGULAR_CUSTOMER)
    return price

def validate_password(password: str) -> bool:
    if len(password) < PasswordPolicy.MIN_LENGTH:
        return False

    if PasswordPolicy.REQUIRE_SPECIAL_CHAR:
        if not re.search(r'[!@#$%^&*()]', password):
            return False

    if PasswordPolicy.REQUIRE_NUMBER:
        if not re.search(r'\d', password):
            return False

    return True
```

## ネストの深さ制限

```typescript
// 悪い例: ネストが深すぎる（5階層）
function processData(data: any[]) {
    if (data) {
        if (data.length > 0) {
            for (const item of data) {
                if (item.isValid) {
                    if (item.type === 'special') {
                        // 処理
                    }
                }
            }
        }
    }
}

// 良い例: 早期リターンでネストを削減
function processData(data: any[]): void {
    if (!data || data.length === 0) {
        return;
    }

    for (const item of data) {
        if (!item.isValid) {
            continue;
        }

        if (item.type !== 'special') {
            continue;
        }

        processSpecialItem(item);
    }
}

function processSpecialItem(item: any): void {
    // 処理ロジック
}
```

## 可読性チェックリスト

```markdown
## 命名
- [ ] 変数名が意図を明確に表しているか
- [ ] 関数名が動作を正確に表しているか
- [ ] クラス名が責務を表しているか
- [ ] 略語を避けているか

## 関数
- [ ] 関数が単一責任を持っているか
- [ ] 関数の長さが適切か（20-50行程度）
- [ ] 引数の数が適切か（3個以下推奨）
- [ ] ネストが深すぎないか（3階層以下推奨）

## コメント
- [ ] Whyを説明するコメントがあるか
- [ ] 不要なコメントを削除したか
- [ ] TODOに責任者と期限があるか

## 定数
- [ ] マジックナンバーを排除したか
- [ ] 定数に意味のある名前を付けたか
```

参考: [Clean Code - Robert C. Martin](https://www.oreilly.com/library/view/clean-code-a/9780136083238/)

## まとめ

本章では、可読性・保守性レビューの具体的な手法を学びました。

- **命名規則**: 意図が明確な名前の付け方
- **関数分割**: 単一責任と適切な長さ
- **コメント**: Whyを説明し、Whatは避ける
- **マジックナンバー**: 定数化による意図の明確化

次章では、パフォーマンスレビューについて学びます。
