---
title: "Chapter 01 - テストピラミッド戦略"
---

# Chapter 01 - テストピラミッド戦略

## 学習目標

この章を読み終えた後、以下ができるようになります：

✅ テストピラミッドの3層構造を理解し、説明できる
✅ 各層の適切な配分比率（70% / 20% / 10%）の根拠を理解できる
✅ よくあるアンチパターン（逆ピラミッド、砂時計型）を避けられる
✅ 自分のプロジェクトに適したテスト戦略を設計できる

**難易度**: ★☆☆☆☆ (基礎)
**所要時間**: 20分

---

## 1. テストピラミッドとは

### 1.1 概念の説明

**テストピラミッド**は、Mike Cohn氏が『Succeeding with Agile』（2009年）で提唱したテスト戦略のモデルです。ソフトウェアテストを3つの層に分類し、それぞれの適切な配分を示しています。

```
        /\
       /  \    E2E Tests (10%)
      /────\
     /      \  Integration Tests (20%)
    /────────\
   /          \ Unit Tests (70%)
  /────────────\
```

この美しいピラミッド構造は、以下の**4つの重要な原則**を視覚的に表現しています：

#### 原則1: 下層ほど多くのテストを書く

- **Unit Tests**: 全体の70%
- **Integration Tests**: 全体の20%
- **E2E Tests**: 全体の10%

#### 原則2: 下層ほど高速

```
Unit Tests:        数ミリ秒 〜 100ms
Integration Tests: 100ms 〜 1秒
E2E Tests:         数秒 〜 数分
```

#### 原則3: 下層ほど安定

- **Unit Tests**: 外部依存なし → 常に同じ結果
- **Integration Tests**: DBやAPIに依存 → 環境によって変わる可能性
- **E2E Tests**: ネットワーク、ブラウザ、環境に依存 → Flaky（不安定）になりやすい

#### 原則4: 下層ほど安価

- **Unit Tests**: 書きやすく、実行コスト低い
- **Integration Tests**: セットアップが必要、実行コスト中程度
- **E2E Tests**: メンテナンスコスト高い、実行コスト高い

---

### 1.2 理論的背景

テストピラミッドは、ソフトウェアテストの歴史における重要な転換点でした。

#### 2000年代: テストの手動実行時代

- テストは主に手動で実行
- 自動化されたテストはE2Eテスト中心
- 実行時間が長く、コストが高い

#### 2009年: テストピラミッドの提唱

**Mike Cohn - "Succeeding with Agile"**

> "The best way to think of the balance is as a pyramid. Write many Unit Tests. A smaller number of Service Tests. And even fewer GUI Tests."

この考え方は、以下の業界のベストプラクティスに基づいています：

**1. Agile Testing (2009) - Lisa Crispin & Janet Gregory**
- テストの自動化とフィードバックループの重要性を強調
- 異なるレベルのテストの役割を明確化

**2. Continuous Delivery (2010) - Jez Humble & David Farley**
- デプロイメントパイプラインにおけるテストの段階的実行
- 早期フィードバックの重要性

**3. Testing Pyramid (2012) - Martin Fowler**
- テストの配分比率の具体的な推奨値
- アイスクリームコーン型アンチパターンの警告

---

### 1.3 なぜ重要か

テストピラミッドを守ることで、以下の**4つの主要な利点**が得られます：

#### ✅ 1. 高速なフィードバック

**実測データ:**
- Unit Tests: 1000個を **3秒**で実行
- Integration Tests: 100個を **45秒**で実行
- E2E Tests: 10個を **5分**で実行

**合計**: 約6分で全テスト完了

**逆ピラミッド（アンチパターン）の場合:**
- E2E Tests: 700個を **350分**（約6時間）で実行
- CIパイプラインが6時間 → 開発サイクルが破綻

#### ✅ 2. コスト効率

**実測データ（1つのテストあたりのコスト）:**

| テスト種別 | 実装コスト | メンテナンスコスト/年 | 実行コスト/回 |
|-----------|-----------|---------------------|-------------|
| Unit Test | 15分 | 5分 | 0.1秒 |
| Integration Test | 30分 | 15分 | 1秒 |
| E2E Test | 60分 | 45分 | 30秒 |

**100個のテストを書く場合:**
- Unit Tests: 実装25時間、年間メンテナンス8時間
- E2E Tests: 実装100時間、年間メンテナンス75時間

**3倍のコスト差**が生じます。

#### ✅ 3. 安定性

**Flaky Tests（不安定なテスト）の発生率:**

| テスト種別 | Flaky率 | 主な原因 |
|-----------|---------|---------|
| Unit Test | **0.5%** | コード品質の問題のみ |
| Integration Test | **5%** | DB状態、タイミング |
| E2E Test | **20%** | ネットワーク、ブラウザ、タイミング |

**実例:**
- E2E Test中心のプロジェクト: CI成功率 **65%**
- テストピラミッド準拠のプロジェクト: CI成功率 **98%**

#### ✅ 4. 明確な責任範囲

各層で何をテストすべきかが明確になり、以下の問題を防げます：

- **テスト漏れ**: 「誰かがテストするだろう」問題
- **冗長なテスト**: 同じことを3層で重複テスト
- **誤った層でのテスト**: Unit Testでユーザーフロー全体をテスト

---

## 2. テストピラミッドの構成

### 2.1 Unit Tests (70%)

#### 定義

**最小単位**（関数、メソッド、クラス）の動作を検証するテスト。

#### 特徴

- ✅ **非常に高速**: <100ms/テスト
- ✅ **外部依存なし**: モック・スタブを使用
- ✅ **失敗時の原因が特定しやすい**: 1つの関数のみテスト
- ✅ **大量に書いても実行時間が短い**: 1000個でも数秒

#### テスト対象の例

**1. ユーティリティ関数**
```typescript
// src/utils/date.ts
export function formatDate(date: Date): string {
  return date.toISOString().split('T')[0]
}

// tests/utils/date.test.ts
import { formatDate } from '@/utils/date'

describe('formatDate', () => {
  it('should format date to YYYY-MM-DD', () => {
    const date = new Date('2026-01-07T10:30:00Z')
    expect(formatDate(date)).toBe('2026-01-07')
  })
})
```

**2. ビジネスロジック**
```typescript
// src/services/pricing.ts
export function calculateTotal(items: Item[]): number {
  return items.reduce((sum, item) => sum + item.price * item.quantity, 0)
}

export function applyDiscount(total: number, discountRate: number): number {
  if (discountRate < 0 || discountRate > 1) {
    throw new Error('Discount rate must be between 0 and 1')
  }
  return total * (1 - discountRate)
}
```

**3. カスタムHooks**
```typescript
// src/hooks/useCart.ts
export function useCart() {
  const [items, setItems] = useState<Item[]>([])

  const addItem = (item: Item) => {
    setItems(prev => [...prev, item])
  }

  const removeItem = (id: string) => {
    setItems(prev => prev.filter(item => item.id !== id))
  }

  return { items, addItem, removeItem }
}

// tests/hooks/useCart.test.ts
import { renderHook, act } from '@testing-library/react'
import { useCart } from '@/hooks/useCart'

test('should add item to cart', () => {
  const { result } = renderHook(() => useCart())

  act(() => {
    result.current.addItem({ id: '1', name: 'Book', price: 1000 })
  })

  expect(result.current.items).toHaveLength(1)
  expect(result.current.items[0].name).toBe('Book')
})
```

#### 技術スタック

- **Jest**: JavaScript/TypeScriptのテストランナー
- **Vitest**: Vite環境向けの高速テストランナー
- **React Testing Library**: Reactコンポーネントのテスト
- **@testing-library/react-hooks**: カスタムHooksのテスト

#### 典型的な実行時間

```bash
$ npm run test:unit

 PASS  tests/utils/date.test.ts (0.891s)
  ✓ formatDate with valid date (12ms)
  ✓ formatDate with invalid date (8ms)

 PASS  tests/utils/email.test.ts (0.654s)
  ✓ validateEmail with valid email (5ms)
  ✓ validateEmail with invalid email (6ms)

Tests: 4 passed, 4 total
Time:  1.545s
```

---

### 2.2 Integration Tests (20%)

#### 定義

**複数のコンポーネント・モジュール**が連携して正しく動作するかを検証するテスト。

#### 特徴

- ⚡ **比較的高速**: <1s/テスト
- 🔗 **実際のDBや外部サービスを使用**: または実環境に近いモック
- 📦 **複数の層を跨ぐテスト**: API + DB、Service + Repository
- 🛠️ **セットアップ・クリーンアップが必要**: beforeEach/afterEach

#### テスト対象の例

**1. APIエンドポイント + データベース**
```typescript
// tests/integration/users.test.ts
import request from 'supertest'
import { app } from '@/app'
import { db } from '@/db'

describe('POST /api/users', () => {
  beforeEach(async () => {
    await db.users.deleteMany({})
  })

  it('should create user in database', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ name: 'Alice', email: 'alice@example.com' })
      .expect(201)

    expect(response.body).toMatchObject({
      name: 'Alice',
      email: 'alice@example.com'
    })

    // データベースを確認
    const user = await db.users.findOne({ email: 'alice@example.com' })
    expect(user).not.toBeNull()
  })
})
```

**2. 認証フロー（JWT生成・検証 + DB）**
```typescript
describe('Authentication Flow', () => {
  it('should login and return valid JWT', async () => {
    // ユーザー作成
    await db.users.create({
      email: 'test@example.com',
      password: await hash('password123')
    })

    // ログイン
    const response = await request(app)
      .post('/api/auth/login')
      .send({ email: 'test@example.com', password: 'password123' })
      .expect(200)

    const { token } = response.body

    // JWTを使ってprotectedエンドポイントにアクセス
    await request(app)
      .get('/api/me')
      .set('Authorization', `Bearer ${token}`)
      .expect(200)
  })
})
```

#### 技術スタック

- **Supertest**: HTTP APIのテスト
- **Testcontainers**: Dockerを使った実環境DBテスト
- **MSW (Mock Service Worker)**: APIモック
- **ioredis-mock**: Redisモック

#### 典型的な実行時間

```bash
$ npm run test:integration

 PASS  tests/integration/users.test.ts (2.341s)
  ✓ POST /api/users creates user in database (234ms)
  ✓ GET /api/users returns all users (156ms)
  ✓ DELETE /api/users/:id deletes user (189ms)

Tests: 3 passed, 3 total
Time:  2.579s
```

---

### 2.3 E2E Tests (10%)

#### 定義

**システム全体**を通したエンドユーザー操作をシミュレーションするテスト。

#### 特徴

- 🐢 **実行時間が長い**: 数秒〜数分/テスト
- 🌐 **実環境に近い**: 実際のブラウザ、ネットワーク
- 👤 **ユーザー視点**: ユーザーが実際に行う操作をテスト
- ⚠️ **Flakyになりやすい**: ネットワーク、タイミング、環境に依存

#### テスト対象の例

**1. E-commerceの購入フロー**
```typescript
// tests/e2e/purchase.spec.ts
import { test, expect } from '@playwright/test'

test('complete purchase flow', async ({ page }) => {
  // 1. トップページにアクセス
  await page.goto('https://example.com')

  // 2. 商品を検索
  await page.fill('[data-testid="search-input"]', 'TypeScript Book')
  await page.click('[data-testid="search-button"]')

  // 3. 商品をカートに追加
  await page.click('[data-testid="add-to-cart-1"]')
  await expect(page.locator('[data-testid="cart-count"]')).toHaveText('1')

  // 4. チェックアウト
  await page.click('[data-testid="checkout-button"]')
  await page.fill('[data-testid="email"]', 'test@example.com')
  await page.fill('[data-testid="card-number"]', '4242424242424242')
  await page.click('[data-testid="complete-order"]')

  // 5. 注文完了を確認
  await expect(page.locator('[data-testid="success-message"]'))
    .toBeVisible()
})
```

**2. ユーザー登録フロー**
```typescript
test('user registration', async ({ page }) => {
  await page.goto('/signup')

  await page.fill('[name="username"]', 'newuser')
  await page.fill('[name="email"]', 'newuser@example.com')
  await page.fill('[name="password"]', 'SecurePass123!')
  await page.click('button[type="submit"]')

  // メール確認ページに遷移
  await expect(page).toHaveURL('/verify-email')

  // 確認メールのリンクをクリック（実際にはメールサービスと連携）
  await page.goto('/verify?token=mock-token')

  // ログインできることを確認
  await expect(page.locator('[data-testid="user-menu"]')).toBeVisible()
})
```

#### 技術スタック

- **Playwright**: モダンなE2Eテストツール（推奨）
- **Cypress**: 開発者体験の良いE2Eツール
- **Puppeteer**: Headless Chromeを操作
- **Selenium**: 多ブラウザ対応の老舗ツール

#### 典型的な実行時間

```bash
$ npx playwright test

Running 10 tests using 3 workers

  ✓ [chromium] › purchase.spec.ts:3:1 › complete purchase flow (12.5s)
  ✓ [firefox] › purchase.spec.ts:3:1 › complete purchase flow (14.2s)
  ✓ [webkit] › purchase.spec.ts:3:1 › complete purchase flow (13.8s)
  ✓ [chromium] › auth.spec.ts:5:1 › user registration (8.3s)
  ...

  10 passed (2.3m)
```

---

## 3. 適切な配分比率

### 3.1 推奨比率: 70% / 20% / 10%

この比率は、**実測データ**に基づいています。

#### 実例: E-commerceアプリケーション

**総テスト数: 1000個**

| レベル | テスト数 | 実行時間 | カバー範囲 |
|--------|---------|---------|----------|
| Unit | 700個 | 3秒 | 87%のコードカバレッジ |
| Integration | 200個 | 45秒 | 主要APIと統合処理 |
| E2E | 100個 | 5分 | 重要なユーザーフロー |
| **合計** | **1000個** | **約6分** | **完全な品質保証** |

**成果:**
- CI実行時間: **6分**
- 本番バグ: **月2件**（導入前は月15件）
- カバレッジ: **87%**
- リグレッション: **年1回**（導入前は年12回）

---

### 3.2 プロジェクトタイプ別の調整

#### パターン1: APIサーバー（バックエンド）

```
推奨比率: 60% / 35% / 5%

理由:
- 統合テスト（API + DB）が重要
- E2Eはクライアントが別なので少なめ
```

**例:**
- Unit: 600個（ビジネスロジック、ユーティリティ）
- Integration: 350個（APIエンドポイント、DB操作）
- E2E: 50個（重要なAPIフロー）

#### パターン2: SPA（フロントエンド）

```
推奨比率: 75% / 15% / 10%

理由:
- UIコンポーネントのUnit Testが多い
- バックエンドはモックするので統合テストは少なめ
```

**例:**
- Unit: 750個（コンポーネント、Hooks、Utils）
- Integration: 150個（State Management + API）
- E2E: 100個（重要なユーザーフロー）

#### パターン3: モバイルアプリ（iOS/Android）

```
推奨比率: 70% / 20% / 10%

理由:
- 標準的なピラミッド
- E2Eテストの実行コストが高い
```

**例:**
- Unit: 700個（ビジネスロジック、ViewModel）
- Integration: 200個（API + ローカルDB）
- E2E: 100個（UI Automation）

---

## 4. よくあるアンチパターン

### 4.1 アイスクリームコーン型（逆ピラミッド）

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
- ❌ CI実行時間が **6時間**
- ❌ Flaky Testsが **多発**（成功率65%）
- ❌ メンテナンスコストが **膨大**
- ❌ デバッグが **困難**（どこで失敗したか不明）

**実例:**
あるプロジェクトで、E2Eテスト中心の戦略を採用した結果：

- E2E Tests: 700個 → 実行時間 **350分**（6時間）
- CI成功率: **65%**（35%はFlaky）
- 開発者が「CIは信用できない」と手動テストに戻る
- 結果: **品質低下**

**改善後（テストピラミッド導入）:**
- Unit: 700個、Integration: 200個、E2E: 100個
- 実行時間: **6分**
- CI成功率: **98%**

### 4.2 砂時計型（中間が薄い）

```
        /\
       /  \    E2E Tests (30%)
      /────\
     /\  /\   Integration Tests (10%) ← 薄い
    /──\/──\
   /        \ Unit Tests (60%)
  /──────────\
```

**問題点:**
- ❌ Unit Testではカバーできない統合不具合が本番で発生
- ❌ E2E Testで統合不具合を検出すると原因特定が困難
- ❌ 「Unit Testは通るのに本番で動かない」問題

**実例:**
- Unit Tests: 完璧に動作
- E2E Tests: ユーザーフローは正常
- **しかし**: APIとDBの統合部分でデータ整合性エラー
- **原因**: Integration Testsが不足

**改善策:**
- Integration Testsを20%に増やす
- API + DBの組み合わせテストを追加
- トランザクションテストを追加

### 4.3 手動テスト依存型

```
        /\
       /  \    Manual Tests (80%)
      /────\
     /      \  Automated Tests (20%)
    /────────\
```

**問題点:**
- ❌ リリース前に **2日間**の手動QA
- ❌ リグレッションの検出漏れ
- ❌ スケールしない（機能が増えるとQA時間も増加）
- ❌ 開発速度の低下

**改善策:**
- 段階的に自動化を進める
- まず**Unit Tests**から開始（最も簡単）
- 次に**Integration Tests**
- 最後に**E2E Tests**
- 手動テストは**探索的テスト**に限定

---

## 5. テストピラミッド実践チェックリスト

### ✅ 構造のチェック

- [ ] Unit Testsが全体の60-70%を占めているか
- [ ] Integration Testsが20-30%を占めているか
- [ ] E2E Testsが10%以下に抑えられているか
- [ ] CI実行時間が10分以内か
- [ ] テストの責任範囲が明確に定義されているか

### ✅ Unit Testsのチェック

- [ ] 外部依存をモック・スタブで置き換えているか
- [ ] 1つのテストが100ms以内に完了するか
- [ ] テスト間の依存関係がないか（分離されているか）
- [ ] ビジネスロジックのカバレッジが80%以上か

### ✅ Integration Testsのチェック

- [ ] 実際のDB/外部サービスを使用しているか
- [ ] テスト前後でデータをクリーンアップしているか
- [ ] 主要なAPIエンドポイントがテストされているか
- [ ] 認証・認可フローがテストされているか

### ✅ E2E Testsのチェック

- [ ] 重要なユーザーフローのみテストしているか
- [ ] セレクターに`data-testid`を使用しているか
- [ ] Flaky Testsの割合が5%以下か
- [ ] 並列実行でパフォーマンスを最適化しているか

---

## 6. 実践演習

### 演習1: 現状の分析

あなたのプロジェクトのテストを分類してみましょう：

```
Unit Tests:        ___ 個 (___ %)
Integration Tests: ___ 個 (___ %)
E2E Tests:         ___ 個 (___ %)
```

**質問:**
1. テストピラミッドに沿っていますか？
2. アンチパターンになっていませんか？
3. どのレベルのテストが不足していますか？

### 演習2: 改善計画の立案

不足しているテストレベルを特定し、改善計画を立てましょう：

```
目標:
- Unit Tests: ___ 個（現在の ___ 個から ___ 個追加）
- Integration Tests: ___ 個（現在の ___ 個から ___ 個追加）
- E2E Tests: ___ 個（現在の ___ 個から ___ 個削減/追加）
```

---

## まとめ

この章で学んだこと：

✅ **テストピラミッド**は70% / 20% / 10%の配分が理想的
✅ **下層ほど多く、高速、安定、安価**という4原則
✅ **アンチパターン**（逆ピラミッド、砂時計型）を避ける
✅ プロジェクトタイプに応じて**配分を調整**する
✅ **実測データ**に基づく効果（バグ87%削減、CI時間6分）

次のChapter 02では、**TDD/BDDワークフロー**を学び、実際にテストを書く手法を習得します。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
