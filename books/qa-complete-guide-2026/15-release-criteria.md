---
title: "リリース判定基準：品質ゲートの設定"
---

# リリース判定基準：品質ゲートの設定

## リリース判定基準とは

リリース判定基準（Release Criteria）は、ソフトウェアを本番環境にリリースしても良いかを判断するための明確な基準です。Entry Criteria（開始条件）とExit Criteria（終了条件）を定義し、客観的な判断を可能にします。

## Entry Criteria（テスト開始条件）

テストフェーズに入る前に満たすべき条件です。

### 基本的なEntry Criteria

```markdown
## テスト開始条件

### 必須条件（すべて満たす必要あり）
- [ ] すべての機能実装が完了している
- [ ] コードレビューが100%完了
- [ ] ユニットテストカバレッジが70%以上
- [ ] 静的解析でCritical問題なし
- [ ] テスト環境が正常に動作
- [ ] テストデータが準備完了
- [ ] APIドキュメントが最新

### 推奨条件（できる限り満たす）
- [ ] 統合テストが一通り動作確認済み
- [ ] パフォーマンステスト環境が準備済み
- [ ] セキュリティスキャン実施済み
```

### 機能別Entry Criteria

```typescript
interface EntryCriteria {
  feature: string;
  requirements: {
    implementation: boolean;
    unitTests: boolean;
    codeReview: boolean;
    documentation: boolean;
  };
}

const featureCriteria: EntryCriteria[] = [
  {
    feature: '決済機能',
    requirements: {
      implementation: true,
      unitTests: true,
      codeReview: true,
      documentation: true  // 決済は必須
    }
  },
  {
    feature: 'UI改善',
    requirements: {
      implementation: true,
      unitTests: false,     // UIテストは手動でも可
      codeReview: true,
      documentation: false
    }
  }
];
```

## Exit Criteria（テスト終了・リリース条件）

テストを終了し、リリースして良いと判断するための条件です。

### 基本的なExit Criteria

```markdown
## リリース判定基準

### 必須条件（すべて満たす必要あり）
- [ ] 計画されたテストケースの95%以上実行完了
- [ ] Critical/Highバグが0件
- [ ] Mediumバグの80%以上が解決済み
- [ ] リグレッションテストが全てパス
- [ ] パフォーマンス要件を満たしている
- [ ] セキュリティスキャンでCritical脆弱性なし
- [ ] UATが完了し、ステークホルダー承認取得

### 品質メトリクス基準
- [ ] テスト合格率 ≥ 95%
- [ ] コードカバレッジ ≥ 80%
- [ ] バグ検出効率 ≥ 90%
- [ ] 再オープン率 < 5%

### 非機能要件
- [ ] レスポンスタイム < 2秒（95パーセンタイル）
- [ ] 1000同時接続で安定動作
- [ ] ダウンタイム < 0.1%（月間）
```

### バグ判定基準の詳細

| 深刻度 | リリース判定 | 備考 |
|--------|------------|------|
| Critical | 0件必須 | システムダウン、データ損失 |
| High | 0件必須 | 主要機能が使えない |
| Medium | 残存許容（10件以下） | 回避策があるもの |
| Low | 数は問わない | 次回リリースで対応 |

### リリース判定の計算式

```typescript
interface ReleaseCriteria {
  testExecution: {
    plannedTests: number;
    executedTests: number;
    passedTests: number;
  };
  bugs: {
    critical: number;
    high: number;
    medium: number;
    low: number;
  };
  coverage: {
    code: number;
    requirements: number;
  };
  performance: {
    responseTime95th: number;
    errorRate: number;
  };
}

function evaluateReleaseCriteria(data: ReleaseCriteria): {
  canRelease: boolean;
  reasons: string[];
} {
  const reasons: string[] = [];

  // テスト実行率チェック
  const executionRate = (data.testExecution.executedTests /
                        data.testExecution.plannedTests) * 100;
  if (executionRate < 95) {
    reasons.push(`テスト実行率が不足: ${executionRate.toFixed(1)}% < 95%`);
  }

  // テスト合格率チェック
  const passRate = (data.testExecution.passedTests /
                   data.testExecution.executedTests) * 100;
  if (passRate < 95) {
    reasons.push(`テスト合格率が不足: ${passRate.toFixed(1)}% < 95%`);
  }

  // バグチェック
  if (data.bugs.critical > 0) {
    reasons.push(`Criticalバグが${data.bugs.critical}件残存`);
  }
  if (data.bugs.high > 0) {
    reasons.push(`Highバグが${data.bugs.high}件残存`);
  }
  if (data.bugs.medium > 10) {
    reasons.push(`Mediumバグが${data.bugs.medium}件残存（上限10件）`);
  }

  // カバレッジチェック
  if (data.coverage.code < 80) {
    reasons.push(`コードカバレッジ不足: ${data.coverage.code}% < 80%`);
  }

  // パフォーマンスチェック
  if (data.performance.responseTime95th > 2000) {
    reasons.push(`レスポンスタイム遅延: ${data.performance.responseTime95th}ms > 2000ms`);
  }
  if (data.performance.errorRate > 0.01) {
    reasons.push(`エラー率高: ${(data.performance.errorRate * 100).toFixed(2)}% > 1%`);
  }

  return {
    canRelease: reasons.length === 0,
    reasons
  };
}
```

## リスクベースのリリース判定

すべての基準を満たせない場合、リスク評価に基づいて判断します。

### リスク評価マトリクス

```typescript
enum RiskLevel {
  Low = 1,
  Medium = 2,
  High = 3,
  Critical = 4
}

interface RiskItem {
  issue: string;
  probability: RiskLevel;  // 発生確率
  impact: RiskLevel;       // 影響度
  mitigation: string;      // 軽減策
}

function calculateRiskScore(item: RiskItem): number {
  return item.probability * item.impact;
}

const risks: RiskItem[] = [
  {
    issue: 'Mediumバグ12件残存',
    probability: RiskLevel.Medium,
    impact: RiskLevel.Low,
    mitigation: '回避策をドキュメント化、次回リリースで修正'
  },
  {
    issue: 'パフォーマンステスト未完了',
    probability: RiskLevel.Low,
    impact: RiskLevel.High,
    mitigation: 'カナリアリリースで段階的展開'
  }
];

// リスクスコア: 1-4=Low, 5-8=Medium, 9-12=High, 13-16=Critical
risks.forEach(risk => {
  const score = calculateRiskScore(risk);
  console.log(`${risk.issue}: スコア ${score}`);
});
```

### 条件付きリリース

```markdown
## 条件付きリリース許可基準

### シナリオ1: 軽微なバグ残存
**条件**:
- Mediumバグが15件（基準は10件）
- すべて回避策あり
- 次回リリース（2週間後）で修正予定

**判定**:
✅ リリース許可
**条件**:
- リリースノートに既知の問題として記載
- サポートチームに回避策を共有
- 2週間後の修正を約束

### シナリオ2: パフォーマンス基準未達
**条件**:
- レスポンスタイム2.3秒（基準2秒）
- トラフィックピーク時のみ発生

**判定**:
⚠️ 条件付きリリース
**条件**:
- カナリアリリースで10%から開始
- レスポンスタイム監視を強化
- 問題発生時は即座にロールバック
```

## Go/No-Go判定会議

### 会議の構成

```markdown
## Go/No-Go判定会議

**参加者**:
- QAリード（判定責任者）
- 開発リード
- プロダクトマネージャー
- SRE/DevOpsエンジニア

**アジェンダ**:
1. Exit Criteria評価（15分）
2. 残存バグレビュー（15分）
3. リスク評価（15分）
4. Go/No-Go判定（15分）
```

### 判定ドキュメント

```markdown
# リリース判定書 v2.0

**判定日**: 2026-03-14
**判定者**: QAリード 山田太郎
**リリース予定日**: 2026-03-15

## Exit Criteria評価

| 項目 | 基準 | 実績 | 判定 |
|------|------|------|------|
| テスト実行率 | ≥95% | 98% | ✅ |
| テスト合格率 | ≥95% | 97% | ✅ |
| Criticalバグ | 0件 | 0件 | ✅ |
| Highバグ | 0件 | 0件 | ✅ |
| Mediumバグ | ≤10件 | 8件 | ✅ |
| コードカバレッジ | ≥80% | 85% | ✅ |
| レスポンスタイム | <2s | 1.8s | ✅ |

## リスク評価

**リスク**: Mediumバグ8件残存
**スコア**: 4（Low）
**軽減策**: リリースノート記載、次回リリースで修正

## 総合判定

**結果**: ✅ **GO**

すべてのExit Criteriaを満たしており、残存リスクも許容範囲内です。
リリースを承認します。

**署名**:
- QAリード: 山田太郎
- 開発リード: 佐藤花子
- PM: 鈴木一郎
```

## まとめ

明確なリリース判定基準により、客観的で一貫性のあるリリース判断が期待されます。Entry/Exit Criteriaを定義し、バグ基準、カバレッジ、パフォーマンスなど多角的に評価することが重要です。

基準を満たせない場合も、リスク評価に基づいた条件付きリリースの選択肢を持つことで、ビジネスニーズとのバランスを取れます。

次章では、リリースチェックリストの具体例を紹介します。
