---
title: "テスト戦略ピラミッド：効率的なテスト配分"
---

# テスト戦略ピラミッド：効率的なテスト配分

## テストピラミッドとは

テストピラミッドは、Mike Cohnによって提唱されたテスト戦略のモデルです[^1]。ピラミッド型の構造で、下層に多くの高速なテストを配置し、上層に少数の低速なテストを配置することで、効率的かつ信頼性の高いテスト戦略を実現します。

```
       /\
      /  \      E2E Tests (少数・遅い・高コスト)
     /____\
    /      \    Integration Tests (中程度)
   /________\
  /          \  Unit Tests (多数・速い・低コスト)
 /____________\
```

## テストピラミッドの3層

### Layer 1: ユニットテスト（Unit Tests）

ピラミッドの基盤となる層で、全体の60-70%を占めることが推奨されます。

**特徴**
- 個別の関数、メソッド、クラスをテスト
- 外部依存を持たない（モック、スタブを使用）
- 実行速度が非常に速い（ミリ秒単位）
- 開発者が作成
- CI/CDパイプラインで毎回実行

**メリット**
- 問題の早期発見
- 高速なフィードバック
- リファクタリングの安全性向上
- ドキュメントとしての役割

**テストフレームワーク例**
- JavaScript/TypeScript: Jest, Vitest, Mocha
- Python: pytest, unittest
- Java: JUnit, TestNG
- Swift: XCTest
- Go: testing package

**ユニットテストの例（TypeScript + Jest）**

```typescript
// sum.ts
export function sum(a: number, b: number): number {
  return a + b;
}

// sum.test.ts
import { sum } from './sum';

describe('sum function', () => {
  test('adds 1 + 2 to equal 3', () => {
    expect(sum(1, 2)).toBe(3);
  });

  test('handles negative numbers', () => {
    expect(sum(-1, -2)).toBe(-3);
  });

  test('handles zero', () => {
    expect(sum(0, 5)).toBe(5);
  });
});
```

**推奨プラクティス**
- F.I.R.S.T 原則に従う
  - **Fast**: 高速に実行できる
  - **Independent**: 他のテストに依存しない
  - **Repeatable**: いつでも同じ結果
  - **Self-validating**: 明確な成功/失敗判定
  - **Timely**: コードと同時に書く
- カバレッジ目標：70-80%以上
- 境界値や異常系も網羅する

### Layer 2: 統合テスト（Integration Tests）

複数のコンポーネントやモジュールの連携をテストする層で、全体の20-30%を占めます。

**特徴**
- モジュール間のインターフェースをテスト
- データベース、API、外部サービスとの連携を検証
- ユニットテストより遅い（秒単位）
- 実際の依存関係を使用（一部モック可）

**テスト対象例**
- データベースアクセス層
- APIエンドポイント
- メッセージキューとの連携
- サードパーティサービス統合

**統合テストの例（Node.js + Supertest）**

```typescript
// api.test.ts
import request from 'supertest';
import { app } from './app';
import { db } from './database';

describe('User API', () => {
  beforeAll(async () => {
    await db.connect();
  });

  afterAll(async () => {
    await db.disconnect();
  });

  test('POST /users creates a new user', async () => {
    const response = await request(app)
      .post('/users')
      .send({
        name: 'Test User',
        email: 'test@example.com'
      })
      .expect(201);

    expect(response.body).toHaveProperty('id');
    expect(response.body.name).toBe('Test User');
  });

  test('GET /users/:id returns user', async () => {
    const response = await request(app)
      .get('/users/1')
      .expect(200);

    expect(response.body).toHaveProperty('name');
  });
});
```

**推奨プラクティス**
- テスト用データベースを使用
- テストデータのセットアップとクリーンアップ
- 外部APIはモックまたはスタブ環境を使用
- 重要な統合ポイントに絞る

### Layer 3: E2Eテスト（End-to-End Tests）

ユーザー視点でアプリケーション全体をテストする層で、全体の5-10%に抑えます。

**特徴**
- 実際のユーザー操作をシミュレート
- UI、バックエンド、データベース、外部サービスすべてを含む
- 最も遅い（分単位）
- メンテナンスコストが高い

**E2Eテストの例（Playwright）**

```typescript
// login.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Login Flow', () => {
  test('successful login redirects to dashboard', async ({ page }) => {
    await page.goto('https://example.com/login');

    await page.fill('input[name="email"]', 'user@example.com');
    await page.fill('input[name="password"]', 'password123');
    await page.click('button[type="submit"]');

    await expect(page).toHaveURL('https://example.com/dashboard');
    await expect(page.locator('h1')).toContainText('Welcome');
  });

  test('invalid credentials show error message', async ({ page }) => {
    await page.goto('https://example.com/login');

    await page.fill('input[name="email"]', 'wrong@example.com');
    await page.fill('input[name="password"]', 'wrongpassword');
    await page.click('button[type="submit"]');

    await expect(page.locator('.error-message')).toBeVisible();
    await expect(page.locator('.error-message')).toContainText('Invalid credentials');
  });
});
```

**E2Eテストフレームワーク**
- Playwright: モダンで高速、複数ブラウザ対応
- Cypress: 開発者フレンドリー、リアルタイムリロード
- Selenium: 最も歴史があり、幅広い言語対応
- Puppeteer: Chromeに特化

**推奨プラクティス**
- Critical User Journey（重要なユーザーフロー）に絞る
- Page Object Modelパターンを使用
- 待機処理を適切に実装
- テストデータの独立性を保つ
- 並列実行でテスト時間を短縮

## テストピラミッドのバリエーション

### テスティングトロフィー（Testing Trophy）

Kent C. Doddsが提唱したモデルで、統合テストにより多くの重点を置きます[^2]。

```
       /\
      /  \      E2E Tests
     /____\
    /      \    Integration Tests (最も重視)
   /________\
  /          \  Unit Tests
 /____________\
  Static Analysis (静的解析)
```

**特徴**
- 統合テストを40-50%に増やす
- 実際のユーザー体験に近いテストを重視
- 過度なモックを避ける

**適用シーン**
- フロントエンド開発
- マイクロサービスアーキテクチャ
- APIドリブンなアプリケーション

### モバイルアプリのテストピラミッド

モバイルアプリでは少し異なるバランスが推奨されます。

```
       /\
      /  \      Manual Exploratory Tests
     /____\
    /      \    UI Tests (15-20%)
   /________\
  /          \  Integration Tests (30-40%)
 /____________\
  Unit Tests (40-50%)
```

**特徴**
- UI層のテストが比較的多い
- デバイス依存のテストが必要
- 手動探索的テストの重要性が高い

## アンチパターン：テストアイスクリームコーン

テストピラミッドの逆、つまりE2Eテストに偏った構成は避けるべきです。

```
 ______________
  \          /  E2E Tests (多数)
   \________/
    \      /    Integration Tests
     \____/
      \  /      Unit Tests (少数)
       \/
```

**問題点**
- テスト実行時間が長い
- フィードバックが遅い
- メンテナンスコストが高い
- 不安定なテスト（Flaky Tests）が発生しやすい
- デバッグが困難

## テスト戦略の策定

プロジェクトに最適なテスト戦略を決定するための考慮事項です。

### アプリケーションタイプによる調整

**バックエンドAPI**
- ユニットテスト：70%
- 統合テスト：25%
- E2Eテスト：5%

**フロントエンドSPA**
- ユニットテスト：50%
- 統合テスト（コンポーネントテスト）：40%
- E2Eテスト：10%

**モバイルアプリ**
- ユニットテスト：45%
- 統合テスト：35%
- UIテスト：15%
- 手動探索テスト：5%

### リスクベースアプローチ

すべての機能を均等にテストするのではなく、リスクに応じてテスト配分を調整します。

**高リスク領域（手厚いテスト）**
- 決済処理
- 認証・認可
- データ損失の可能性がある処理
- セキュリティ関連機能
- 法規制対象の機能

**中リスク領域（標準的なテスト）**
- 主要な業務ロジック
- ユーザーが頻繁に使う機能

**低リスク領域（最小限のテスト）**
- UIの装飾的な要素
- 単純なCRUD操作
- 使用頻度の低い機能

## テスト自動化の ROI（投資対効果）

すべてのテストを自動化すべきではありません。自動化のコストと効果を評価します。

### 自動化に適したテスト
- 頻繁に実行されるテスト（リグレッションテスト）
- データドリブンテスト（同じシナリオで異なるデータ）
- 複数環境で実行するテスト
- 手動では時間がかかるテスト（パフォーマンステスト）

### 手動テストが適している領域
- 探索的テスト
- ユーザビリティテスト
- アドホックテスト
- 一度きりのテスト
- 自動化コストが高すぎるテスト

## まとめ

テストピラミッドは、高速で信頼性の高い、メンテナンス性に優れたテスト戦略の指針です。ユニットテストを基盤とし、適度な統合テスト、最小限のE2Eテストでバランスを取ることが推奨されます。

ただし、ピラミッドは絶対的なルールではなく、プロジェクトの特性に応じて調整が必要です。重要なのは、各層のテストの特性を理解し、効率的かつ効果的なテスト戦略を構築することです。

次章では、テスト計画書のテンプレートと作成方法について詳しく解説します。

[^1]: Mike Cohn - "Succeeding with Agile: Software Development Using Scrum" (2009)
[^2]: Kent C. Dodds - "Write tests. Not too many. Mostly integration." (TestingJavaScript.com)
