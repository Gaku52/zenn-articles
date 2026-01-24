---
title: "リグレッションテスト：継続的な品質保証"
---

# リグレッションテスト：継続的な品質保証

## リグレッションテストとは

リグレッションテスト（Regression Testing）は、ソフトウェアの変更後に既存機能が正しく動作し続けることを確認するテストです。新機能追加やバグ修正が、意図しない副作用（リグレッション）を引き起こしていないかを検証します。

## リグレッションテストの重要性

コードベースが成長するにつれ、変更の影響範囲は予測困難になります。業界調査によると、ソフトウェアのバグの約15-20%はリグレッション（過去動いていた機能の破壊）に起因すると報告されています[^1]。

## リグレッションテストの戦略

### 1. 完全リグレッション（Full Regression）

すべてのテストケースを実行する最も網羅的なアプローチです。

**適用シーン**:
- メジャーリリース前
- アーキテクチャ変更後
- 定期的な品質検証（月次・四半期）

**メリット**:
- 最高レベルの品質保証
- すべての機能の動作確認

**デメリット**:
- 実行時間が長い
- リソースコストが高い

### 2. 選択的リグレッション（Selective Regression）

変更の影響範囲を分析し、関連するテストケースのみ実行します。

**影響範囲分析の例**:

```typescript
// 変更: User.ts の getDisplayName() メソッド

影響範囲:
- UserProfile コンポーネント
- Header コンポーネント
- CommentList コンポーネント

実行すべきテスト:
- user-profile.test.ts
- header.test.ts
- comment-list.test.ts
```

**適用シーン**:
- 日次ビルド
- 小規模な機能追加
- バグ修正後

### 3. 優先順位ベースリグレッション

テストケースに優先度をつけ、時間制約内で重要なものから実行します。

| 優先度 | 対象 | 実行頻度 |
|-------|------|---------|
| P0 | Critical path（決済、ログインなど） | 毎ビルド |
| P1 | 主要機能 | 毎日 |
| P2 | 通常機能 | 週次 |
| P3 | 低頻度機能 | リリース前 |

### 4. リスクベースリグレッション

リスクの高い領域に焦点を当てます。

**高リスク領域の判定基準**:
- 過去のバグ密度が高い
- ビジネスクリティカル
- 複雑度が高い
- 変更頻度が高い
- ユーザー影響が大きい

## リグレッションテストの自動化

手動での完全リグレッションは非現実的なため、自動化が必須です。

### 自動化の ROI（投資対効果）

```
ROI = (節約できた工数 × 実行回数 - 自動化コスト) / 自動化コスト × 100

例:
手動実行: 8時間
自動実行: 15分
年間実行回数: 250回

節約時間 = (8 - 0.25) × 250 = 1,937.5時間
自動化コスト = 80時間

ROI = (1937.5 - 80) / 80 × 100 = 2,321%
```

### 自動化に適したテスト

✓ **適している**:
- 頻繁に実行されるテスト
- データドリブンテスト
- 安定したUIを持つテスト
- クリティカルパステスト

✗ **適していない**:
- 一度きりのテスト
- UI が頻繁に変わるテスト
- 複雑すぎて保守コストが高いテスト
- 探索的テスト

### リグレッションスイートの構成

```yaml
regression-suite:
  smoke-tests:  # 5分以内
    - login
    - search
    - checkout-flow

  core-tests:  # 30分以内
    - user-management
    - product-catalog
    - order-processing

  full-tests:  # 2時間以内
    - all-features
    - edge-cases
    - integration-tests
```

## CI/CDパイプラインへの統合

```yaml
# .github/workflows/regression.yml
name: Regression Tests

on:
  push:
    branches: [main, develop]
  pull_request:
  schedule:
    - cron: '0 2 * * *'  # 毎日2時に実行

jobs:
  smoke-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run Smoke Tests
        run: npm run test:smoke
      - name: Notify on failure
        if: failure()
        uses: slackapi/slack-github-action@v1

  regression-tests:
    needs: smoke-tests
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]  # 並列実行で高速化
    steps:
      - uses: actions/checkout@v3
      - name: Run Regression Tests
        run: npm run test:regression -- --shard=${{ matrix.shard }}/4
      - name: Upload Test Results
        uses: actions/upload-artifact@v3
        with:
          name: test-results
          path: test-results/
```

## リグレッションテストのメンテナンス

### 1. Flaky Tests（不安定なテスト）への対処

Flaky Testsは時々失敗する不安定なテストで、リグレッションテストの信頼性を損ないます。

**原因**:
- タイミング依存（待機時間不足）
- テストデータの汚染
- 並列実行時の競合
- 外部サービス依存

**対策**:

```typescript
// 悪い例: 固定待機時間
await page.click('button');
await page.waitForTimeout(3000); // 不安定

// 良い例: 条件待機
await page.click('button');
await page.waitForSelector('.success-message', {
  state: 'visible',
  timeout: 10000
});
```

**Flaky Test の追跡**:

```typescript
// Flaky Test 検出スクリプト
// 同じテストを100回実行して成功率を計算
const results = [];
for (let i = 0; i < 100; i++) {
  const result = await runTest('login-test');
  results.push(result);
}

const successRate = results.filter(r => r.passed).length;
console.log(`Success rate: ${successRate}%`);

if (successRate < 95) {
  console.warn('⚠️ Flaky test detected!');
}
```

### 2. テストデータ管理

**問題**: 共有テストデータによる競合

```typescript
// 悪い例: 共有データ
test('update user name', async () => {
  await updateUser({ id: 1, name: 'New Name' });
  // 他のテストも id=1 を使うと競合
});

// 良い例: テストごとに独立したデータ
test('update user name', async () => {
  const user = await createTestUser(); // 毎回新規作成
  await updateUser({ id: user.id, name: 'New Name' });
  await cleanup(user.id); // テスト後クリーンアップ
});
```

### 3. テストの定期的な見直し

```markdown
## リグレッションスイート定期見直し（四半期ごと）

### チェック項目
- [ ] 重複テストケースの削除
- [ ] 廃止機能のテスト削除
- [ ] 実行時間の最適化
- [ ] Flaky Tests の修正または削除
- [ ] カバレッジギャップの特定

### メトリクス
- 総テスト数: 1,200 → 1,000（削減）
- 実行時間: 45分 → 30分（改善）
- Flaky率: 5% → 1%（改善）
```

## ビジュアルリグレッションテスト

UIの視覚的な変更を検出するテストです。

### スクリーンショット比較

```typescript
// Playwright Visual Regression
import { test, expect } from '@playwright/test';

test('homepage visual regression', async ({ page }) => {
  await page.goto('https://example.com');

  // スクリーンショット撮影と比較
  await expect(page).toHaveScreenshot('homepage.png', {
    maxDiffPixels: 100,  // 許容差分
  });
});
```

### ビジュアルテストツール

- **Percy**: クラウドベースのビジュアルテスト
- **Chromatic**: Storybookと統合
- **Applitools**: AIベースの画像比較
- **BackstopJS**: オープンソース

## データベーススキーマのリグレッション

スキーマ変更が既存機能に影響しないか確認します。

```sql
-- マイグレーションテスト
BEGIN TRANSACTION;

-- 1. 現在のデータをバックアップ
CREATE TABLE users_backup AS SELECT * FROM users;

-- 2. マイグレーション実行
ALTER TABLE users ADD COLUMN email_verified BOOLEAN DEFAULT FALSE;

-- 3. データ整合性確認
SELECT COUNT(*) FROM users WHERE email_verified IS NULL;
-- 期待: 0件

-- 4. 既存クエリの動作確認
SELECT id, name, email FROM users WHERE active = TRUE;
-- 期待: 正常に動作

ROLLBACK;  -- テスト環境では巻き戻し
```

## リグレッションテストのレポーティング

### ダッシュボード例

```markdown
## リグレッションテスト結果

**実行日時**: 2026-03-01 02:00

### サマリー
- 総テストケース: 1,000
- 成功: 985 (98.5%)
- 失敗: 10 (1.0%)
- スキップ: 5 (0.5%)
- 実行時間: 28分

### 失敗内訳
| テスト | 理由 | 担当 |
|--------|------|------|
| checkout-flow | タイムアウト | 山田 |
| user-profile | API変更 | 佐藤 |

### トレンド
- 先週比: +5件 改善
- カバレッジ: 82% → 85%
```

## まとめ

リグレッションテストは、継続的な品質保証の要です。完全、選択的、優先順位ベースなど複数の戦略を組み合わせ、自動化によって効率化することが推奨されます。

Flaky Testsの削減、テストデータの適切な管理、定期的な見直しにより、信頼性の高いリグレッションスイートを維持できます。CI/CDパイプラインへの統合により、すべての変更に対して自動的に品質を保証する仕組みが期待されます。

次章では、QAメトリクスの詳細な収集と分析方法について解説します。

[^1]: Capers Jones - "Software Quality: Analysis and Guidelines for Success" (1997)
