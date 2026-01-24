---
title: "カバレッジメトリクス：テストの網羅性測定"
---

# カバレッジメトリクス：テストの網羅性測定

## カバレッジとは

カバレッジ（Coverage）は、テストがコードや要件のどれだけをカバーしているかを示す指標です。適切なカバレッジ測定により、テストの盲点を発見し、品質リスクを低減できます。

## コードカバレッジの種類

### 1. 行カバレッジ（Line Coverage）

最も基本的なカバレッジで、実行された行の割合を測定します。

```typescript
// 例: 行カバレッジ
function calculateDiscount(price: number): number {
  if (price > 10000) {        // Line 1
    return price * 0.1;        // Line 2
  }
  return 0;                    // Line 3
}

// テスト1つだけの場合
test('high price', () => {
  expect(calculateDiscount(15000)).toBe(1500);
});
// 行カバレッジ: 66% (Line 1,2のみ実行、Line 3未実行)
```

### 2. 分岐カバレッジ（Branch Coverage）

すべての条件分岐（if, switch）の true/false 両方をテストしているかを測定します。

```typescript
function isEligible(age: number, hasLicense: boolean): boolean {
  if (age >= 18 && hasLicense) {
    return true;
  }
  return false;
}

// 完全な分岐カバレッジには4ケース必要
test.each([
  [18, true, true],    // both true
  [17, true, false],   // age false, license true
  [18, false, false],  // age true, license false
  [17, false, false],  // both false
])('age %i, license %s => %s', (age, license, expected) => {
  expect(isEligible(age, license)).toBe(expected);
});
```

### 3. 関数カバレッジ（Function Coverage）

呼び出された関数の割合を測定します。

```
関数カバレッジ = 実行された関数数 / 総関数数 × 100
```

### 4. 条件カバレッジ（Condition Coverage）

複合条件の各サブ条件が true/false 両方を取るかを測定します。

```typescript
// A && B の場合
// 完全な条件カバレッジには以下が必要:
// 1. A=true, B=true
// 2. A=true, B=false
// 3. A=false, B=true
// 4. A=false, B=false
```

## カバレッジツールと設定

### JavaScript/TypeScript (Jest + Istanbul)

```json
// package.json
{
  "scripts": {
    "test": "jest --coverage"
  },
  "jest": {
    "collectCoverageFrom": [
      "src/**/*.{ts,tsx}",
      "!src/**/*.test.{ts,tsx}",
      "!src/**/*.stories.{ts,tsx}"
    ],
    "coverageThresholds": {
      "global": {
        "lines": 80,
        "branches": 75,
        "functions": 80,
        "statements": 80
      }
    }
  }
}
```

### Python (pytest + coverage.py)

```ini
# .coveragerc
[run]
source = src
omit =
    */tests/*
    */migrations/*

[report]
precision = 2
show_missing = True

[html]
directory = htmlcov
```

```bash
pytest --cov=src --cov-report=html --cov-report=term
```

### Go (built-in coverage)

```bash
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out -o coverage.html
```

## 要件カバレッジ

コードだけでなく、要件仕様がどれだけテストされているかも重要です。

### 要件トレーサビリティマトリクス

| 要件ID | 要件内容 | テストケース | ステータス |
|--------|---------|------------|-----------|
| REQ-001 | ユーザー登録 | TC-001, TC-002 | ✓ |
| REQ-002 | ログイン | TC-003, TC-004, TC-005 | ✓ |
| REQ-003 | パスワードリセット | TC-006 | ✓ |
| REQ-004 | 二要素認証 | - | ✗ 未実装 |

```
要件カバレッジ = テスト済み要件数 / 総要件数 × 100
= 3 / 4 × 100 = 75%
```

## カバレッジの目標設定

### 推奨カバレッジ目標

| コード種別 | 目標カバレッジ |
|-----------|-------------|
| ビジネスロジック | 85-95% |
| ユーティリティ関数 | 80-90% |
| UI コンポーネント | 60-75% |
| 設定ファイル | 0-30% |

### プロジェクトフェーズ別

| フェーズ | 目標 |
|---------|------|
| 新規開発（初期） | 70% |
| 機能追加期 | 75% |
| 安定期 | 80% |
| レガシーコード | 段階的に向上 |

## カバレッジの落とし穴

### 1. カバレッジの高さ ≠ テストの質

```typescript
// 悪い例: カバレッジは100%だが意味のないテスト
function add(a: number, b: number): number {
  return a + b;
}

test('calls add function', () => {
  add(1, 2);  // アサーションなし！
});
// カバレッジ100%だが、正しさを検証していない
```

### 2. 未到達コード（Dead Code）

```typescript
function processData(data: any) {
  if (data) {
    // 処理
  }
  return true;  // 常にtrueを返す

  console.log('Never executed');  // 到達不可能
}
// カバレッジでデッドコードが見つかる
```

### 3. 境界条件の欠如

```typescript
// カバレッジは達成しているが境界値テストがない
function validateAge(age: number): boolean {
  return age >= 18;
}

test('valid age', () => {
  expect(validateAge(25)).toBe(true);  // 中間値のみ
});
// 17, 18, 19 などの境界値をテストすべき
```

## カバレッジレポートの活用

### CI/CDでのカバレッジチェック

```yaml
# .github/workflows/test.yml
name: Test with Coverage

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run tests with coverage
        run: npm run test:coverage

      - name: Check coverage threshold
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "❌ Coverage $COVERAGE% is below 80%"
            exit 1
          fi

      - name: Upload to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json
```

### カバレッジバッジの表示

```markdown
# README.md
![Coverage](https://img.shields.io/codecov/c/github/username/repo)
```

## カバレッジ改善戦略

### 1. 低カバレッジファイルの特定

```bash
# カバレッジが50%未満のファイルを抽出
npm test -- --coverage --coverageReporters=json-summary

jq '.[] | select(.lines.pct < 50) | .file' coverage-summary.json
```

### 2. 段階的な改善

```markdown
## カバレッジ改善計画

### Phase 1 (今月)
- ビジネスロジック層を70%→80%に改善
- Critical pathを100%カバー

### Phase 2 (来月)
- API層を60%→75%に改善
- エラーハンドリングのカバレッジ強化

### Phase 3 (来々月)
- 全体で80%達成
- 継続的維持の仕組み構築
```

### 3. 未カバーコードのレビュー

```typescript
// カバレッジツールで未実行行を特定
if (process.env.NODE_ENV === 'production') {
  // このブランチがテスト環境で実行されていない
  sendToAnalytics(data);  // 未カバー
}

// 対策: テストで環境変数を変更
test('sends analytics in production', () => {
  process.env.NODE_ENV = 'production';
  // テスト実行
  process.env.NODE_ENV = 'test';  // 戻す
});
```

## カバレッジダッシュボード

### Grafanaでの可視化

```json
{
  "panels": [
    {
      "title": "コードカバレッジトレンド",
      "type": "graph",
      "targets": [{
        "query": "SELECT time, coverage FROM metrics WHERE project='myapp'"
      }]
    },
    {
      "title": "現在のカバレッジ",
      "type": "gauge",
      "targets": [{
        "query": "SELECT coverage FROM metrics ORDER BY time DESC LIMIT 1"
      }],
      "thresholds": [
        {"value": 0, "color": "red"},
        {"value": 70, "color": "yellow"},
        {"value": 80, "color": "green"}
      ]
    }
  ]
}
```

## まとめ

カバレッジメトリクスは、テストの網羅性を定量的に測定する重要な指標です。行カバレッジ、分岐カバレッジ、要件カバレッジなど複数の観点から測定し、目標値を設定して継続的に改善することが推奨されます。

ただし、カバレッジの高さがテストの質を保証するわけではありません。適切なアサーション、境界値テスト、意味のあるテストケースと組み合わせることで、真の品質向上が期待されます。

次章では、GrafanaによるKPIダッシュボードの構築について解説します。
