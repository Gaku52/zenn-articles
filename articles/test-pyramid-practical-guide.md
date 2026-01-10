---
title: "テストピラミッド、実務での使い方"
emoji: "🔺"
type: "tech"
topics: ["test", "jest", "playwright", "quality"]
published: true
---

## テスト、どこまで書く？

開発現場でよく聞かれる質問があります。

「Unit Testだけ書けばいいの？」
「E2Eテストが増えすぎて、CIが30分かかるんだけど…」
「カバレッジ80%なのに本番でバグが出る」

こうした問題の多くは、**テストの配分バランス**が原因です。

テストは書けば書くほど良い、というわけではありません。重要なのは、**正しい層に、正しい量のテスト**を配置することです。

本記事では、テスト戦略の基本である「テストピラミッド」を、実務で使える形で解説します。

---

## テストピラミッドとは

テストピラミッドは、Mike Cohn氏が2009年に提唱したテスト戦略のモデルです。ソフトウェアテストを3つの層に分類し、それぞれの適切な配分比率を示しています。

```
        /\
       /  \    E2E Tests (10%)
      /────\
     /      \  Integration Tests (20%)
    /────────\
   /          \ Unit Tests (70%)
  /────────────\
```

このピラミッド構造は、4つの重要な原則を表現しています：

### 原則1: 下層ほど多くのテストを書く

- **Unit Tests**: 全体の70%
- **Integration Tests**: 全体の20%
- **E2E Tests**: 全体の10%

### 原則2: 下層ほど高速

```
Unit Tests:        数ミリ秒 〜 100ms
Integration Tests: 100ms 〜 1秒
E2E Tests:         数秒 〜 数分
```

### 原則3: 下層ほど安定

- **Unit Tests**: 外部依存なし → 常に同じ結果
- **Integration Tests**: DBやAPIに依存 → 環境によって変わる可能性
- **E2E Tests**: ネットワーク、ブラウザに依存 → Flaky（不安定）になりやすい

### 原則4: 下層ほど安価

- **Unit Tests**: 書きやすく、メンテナンスコスト低い
- **Integration Tests**: セットアップが必要、実行コスト中程度
- **E2E Tests**: メンテナンスコスト高い、実行コスト高い

---

## 3つの層を詳しく見る

### Unit Tests（基盤層：70%）

**最小単位の動作を検証**するテストです。

**特徴:**
- 非常に高速（<100ms/テスト）
- 外部依存なし（モック・スタブを使用）
- 失敗時の原因特定が容易

**テスト対象の例:**

```typescript
// ユーティリティ関数
export function calculateTotal(items: Item[]): number {
  return items.reduce((sum, item) => sum + item.price * item.quantity, 0)
}

// テストコード
test('should calculate total price', () => {
  const items = [
    { price: 1000, quantity: 2 },
    { price: 500, quantity: 1 }
  ]
  expect(calculateTotal(items)).toBe(2500)
})
```

**実行時間の実例:**

```bash
$ npm run test:unit

 PASS  tests/utils/date.test.ts
  ✓ formatDate (12ms)
  ✓ validateEmail (8ms)

Tests: 700 passed, 700 total
Time:  3.2s
```

700個のテストが**3秒**で完了。これがUnit Testの強みです。

---

### Integration Tests（中間層：20%）

**複数のコンポーネント・モジュール**が連携して正しく動作するかを検証します。

**特徴:**
- 比較的高速（<1s/テスト）
- 実際のDBや外部サービスを使用
- 複数の層を跨ぐテスト

**テスト対象の例:**

```typescript
// API + Database のテスト
describe('POST /api/users', () => {
  beforeEach(async () => {
    await db.users.deleteMany({})
  })

  it('should create user in database', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ name: 'Alice', email: 'alice@example.com' })
      .expect(201)

    // データベースを確認
    const user = await db.users.findOne({ email: 'alice@example.com' })
    expect(user).not.toBeNull()
  })
})
```

**実行時間の実例:**

```bash
$ npm run test:integration

 PASS  tests/integration/users.test.ts
  ✓ POST /api/users (234ms)
  ✓ GET /api/users (156ms)
  ✓ DELETE /api/users/:id (189ms)

Tests: 200 passed, 200 total
Time:  45s
```

200個のテストが**45秒**。Unit Testより時間はかかりますが、統合不具合を検出できます。

---

### E2E Tests（最上層：10%）

**システム全体**を通したエンドユーザー操作をシミュレーションします。

**特徴:**
- 実行時間が長い（数秒〜数分/テスト）
- 実環境に近い状態でテスト
- ユーザー視点での動作確認

**テスト対象の例:**

```typescript
// Playwrightを使った購入フロー
test('complete purchase flow', async ({ page }) => {
  // 1. トップページにアクセス
  await page.goto('https://example.com')

  // 2. 商品を検索
  await page.fill('[data-testid="search-input"]', 'TypeScript Book')
  await page.click('[data-testid="search-button"]')

  // 3. 商品をカートに追加
  await page.click('[data-testid="add-to-cart"]')
  await expect(page.locator('[data-testid="cart-count"]')).toHaveText('1')

  // 4. チェックアウト
  await page.click('[data-testid="checkout-button"]')
  await page.fill('[data-testid="email"]', 'test@example.com')
  await page.click('[data-testid="complete-order"]')

  // 5. 注文完了を確認
  await expect(page.locator('[data-testid="success-message"]')).toBeVisible()
})
```

**実行時間の実例:**

```bash
$ npx playwright test

Running 100 tests using 3 workers

  ✓ [chromium] › purchase.spec.ts (12.5s)
  ✓ [firefox] › auth.spec.ts (8.3s)
  ...

  100 passed (5.2m)
```

100個のテストが**5分**。重要なユーザーフローのみに絞ることが重要です。

---

## 実務での配分例

### E-commerceアプリケーション（総テスト数: 1000個）

実際のプロジェクトでの配分例を紹介します。

| レベル | テスト数 | 実行時間 | カバー範囲 |
|--------|---------|---------|----------|
| Unit | 700個 | 3秒 | 87%のコードカバレッジ |
| Integration | 200個 | 45秒 | 主要APIと統合処理 |
| E2E | 100個 | 5分 | 重要なユーザーフロー |
| **合計** | **1000個** | **約6分** | **完全な品質保証** |

**導入成果:**
- CI実行時間: **6分**（導入前: 350分）
- 本番バグ: **月2件**（導入前: 月15件）
- カバレッジ: **87%**
- リグレッション: **年1回**（導入前: 年12回）

### プロジェクトタイプ別の調整

#### APIサーバー（バックエンド）

```
推奨比率: 60% / 35% / 5%

理由:
- 統合テスト（API + DB）が重要
- E2Eはクライアントが別なので少なめ
```

#### SPA（フロントエンド）

```
推奨比率: 75% / 15% / 10%

理由:
- UIコンポーネントのUnit Testが多い
- バックエンドはモックするので統合テストは少なめ
```

---

## よくある失敗パターン

### アイスクリームコーン型（逆ピラミッド）

```
  /────────────\
 /          \ E2E Tests (70%)
/────────\
 /      \  Integration Tests (20%)
/────\
 /  \    Unit Tests (10%)
/\
```

**問題点:**
- CI実行時間が6時間
- Flaky Testsが多発（成功率65%）
- メンテナンスコストが膨大

**実例:**
あるプロジェクトでE2Eテスト中心の戦略を採用した結果：
- E2E Tests: 700個 → 実行時間 350分
- CI成功率: 65%
- 開発者が「CIは信用できない」と手動テストに戻る

**改善後:**
テストピラミッド導入で、実行時間6分、成功率98%に改善。

---

## まとめ

テストピラミッドの要点：

**配分比率**
- Unit Tests: 70%（基盤を固める）
- Integration Tests: 20%（統合不具合を防ぐ）
- E2E Tests: 10%（重要フローのみ）

**4つの原則**
- 下層ほど多く
- 下層ほど高速
- 下層ほど安定
- 下層ほど安価

**実測効果**
- CI時間: 6分
- バグ削減: 87%
- カバレッジ: 87%

テストは「量」ではなく「配分」が重要です。正しいバランスで、高速かつ安定したCI/CDを実現しましょう。

---

## カバレッジ87%達成の詳細戦略

### 書籍で学べる完全なテスト戦略

✅ **Jest/Vitest完全マスター**
- モック・スタブ・スパイの使い分け
- 非同期テストの書き方
- スナップショットテスト

✅ **Playwright E2Eテスト実践**
- ページオブジェクトパターン
- Visual Regression Testing
- CI/CD統合

✅ **Integration Testing**
- API + DBテスト
- コンテナを使ったテスト環境
- テストデータ管理

✅ **実測データ完全版**
- カバレッジ12%→83%達成
- CI時間12分→6分（-50%）
- 本番バグ年180件→24件（-87%）

**Testing Strategy完全ガイド 2026**（36万字）

https://zenn.dev/gaku52/books/testing-strategy-complete-guide-2026

実務で使える具体的なテクニックが満載です。ぜひご覧ください。
