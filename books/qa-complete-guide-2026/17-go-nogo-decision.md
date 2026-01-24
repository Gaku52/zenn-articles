---
title: "Go/No-Go判定：リリース可否の最終決定"
---

# Go/No-Go判定：リリース可否の最終決定

## Go/No-Go判定とは

Go/No-Go判定は、収集した品質データとリリース基準に基づいて、リリースするか（Go）、延期するか（No-Go）を決定するプロセスです。客観的なデータと組織的な合意により、リスクを最小化します。

## 判定会議の構成

### 参加者

```markdown
## 必須参加者
- **QAリード**: 品質データの報告と判定
- **開発リード**: 技術的リスクの評価
- **プロダクトマネージャー**: ビジネス要件との整合性確認
- **SRE/DevOpsエンジニア**: インフラとデプロイメントの準備状況
- **セキュリティエンジニア**: セキュリティリスク評価（必要に応じて）

## オプション参加者
- カスタマーサポートリード
- 経営層（メジャーリリース時）
- 法務担当（コンプライアンス関連）
```

### アジェンダ（60分標準）

```markdown
## Go/No-Go判定会議アジェンダ

**1. イントロダクション（5分）**
- リリース概要説明
- 会議の目的確認

**2. Exit Criteria評価（15分）**
- QAリードによる品質メトリクス報告
- テスト実行結果の共有
- バグ状況の報告

**3. 残存リスク評価（15分）**
- 未解決バグのレビュー
- 技術的負債の確認
- インフラ準備状況

**4. ディスカッション（15分）**
- 懸念事項の議論
- 軽減策の検討

**5. Go/No-Go判定（10分）**
- 各ステークホルダーの意見表明
- 最終判定と理由の記録
- アクションアイテムの確認
```

## 判定マトリクス

### 判定基準の評価

```typescript
interface GoNoGoFactors {
  quality: {
    testCompleteness: number;      // %
    testPassRate: number;           // %
    criticalBugs: number;
    highBugs: number;
    mediumBugs: number;
  };
  technical: {
    codeReviewComplete: boolean;
    performanceMetsMet: boolean;
    securityScanPassed: boolean;
    infrastructureReady: boolean;
  };
  business: {
    featureComplete: boolean;
    stakeholderApproval: boolean;
    marketTimingCritical: boolean;
  };
  operational: {
    rollbackPlanReady: boolean;
    monitoringSetup: boolean;
    supportTeamBriefed: boolean;
  };
}

enum Decision {
  GO = 'GO',
  NO_GO = 'NO_GO',
  CONDITIONAL_GO = 'CONDITIONAL_GO'
}

function evaluateGoNoGo(factors: GoNoGoFactors): {
  decision: Decision;
  reason: string;
  conditions?: string[];
} {
  // 必須条件チェック
  if (factors.quality.criticalBugs > 0) {
    return {
      decision: Decision.NO_GO,
      reason: 'Criticalバグが残存している'
    };
  }

  if (factors.quality.testPassRate < 95) {
    return {
      decision: Decision.NO_GO,
      reason: `テスト合格率が基準未達（${factors.quality.testPassRate}% < 95%）`
    };
  }

  if (!factors.technical.securityScanPassed) {
    return {
      decision: Decision.NO_GO,
      reason: 'セキュリティスキャンで重大な問題が検出された'
    };
  }

  // 条件付きGOの判定
  const conditions: string[] = [];

  if (factors.quality.mediumBugs > 10) {
    conditions.push('Mediumバグ多数 - カナリアリリースで段階的展開');
  }

  if (!factors.technical.performanceMetsMet) {
    conditions.push('パフォーマンス基準未達 - 監視強化と即座のロールバック準備');
  }

  if (conditions.length > 0) {
    return {
      decision: Decision.CONDITIONAL_GO,
      reason: '一部基準未達だが軽減策あり',
      conditions
    };
  }

  // GO判定
  return {
    decision: Decision.GO,
    reason: 'すべての基準を満たしている'
  };
}
```

## 判定パターンと対応

### Pattern 1: クリアGO

```markdown
## 判定: ✅ **GO**

### 状況
- すべてのExit Criteriaを満たす
- 技術的リスクなし
- ステークホルダー全員が承認

### 決定
予定通りリリースを実施

### アクション
- [ ] リリースノート公開
- [ ] ユーザー通知送信
- [ ] 監視体制確保
```

### Pattern 2: クリアNo-Go

```markdown
## 判定: ❌ **NO-GO**

### 状況
- Criticalバグ3件残存
- セキュリティスキャンでCritical脆弱性検出
- パフォーマンステスト未完了

### 決定
リリースを延期し、問題解決後に再判定

### アクション
- [ ] Criticalバグの修正（優先度P0）
- [ ] セキュリティ脆弱性の修正
- [ ] パフォーマンステスト実施
- [ ] 1週間後に再判定会議（3/22）
- [ ] ステークホルダーへの延期通知
```

### Pattern 3: 条件付きGO

```markdown
## 判定: ⚠️ **CONDITIONAL GO**

### 状況
- Mediumバグ15件残存（基準10件）
- すべて回避策あり
- ビジネス上のリリース期限が迫っている

### 決定
条件付きでリリースを許可

### 条件
1. カナリアリリースで10%から開始
2. 24時間監視を強化
3. エラー率0.5%超過で即座にロールバック
4. 2週間後の次回リリースで残存バグを修正

### アクション
- [ ] カナリアデプロイメント設定
- [ ] 監視ダッシュボード準備
- [ ] ロールバック手順確認
- [ ] サポートチームに既知の問題を共有
- [ ] リリースノートに既知の問題を記載
```

## 判定ドキュメントテンプレート

```markdown
# Go/No-Go判定書

**プロジェクト**: ECサイトリニューアル v2.0
**判定日時**: 2026-03-14 14:00-15:00
**リリース予定**: 2026-03-15 02:00 JST

## 参加者
- QAリード: 山田太郎 ✓
- 開発リード: 佐藤花子 ✓
- PM: 鈴木一郎 ✓
- SRE: 田中次郎 ✓

## Exit Criteria評価

| 項目 | 基準 | 実績 | 判定 |
|------|------|------|------|
| テスト実行率 | ≥95% | 98% | ✅ |
| テスト合格率 | ≥95% | 97% | ✅ |
| Criticalバグ | 0件 | 0件 | ✅ |
| Highバグ | 0件 | 0件 | ✅ |
| Mediumバグ | ≤10件 | 8件 | ✅ |
| カバレッジ | ≥80% | 85% | ✅ |
| レスポンスタイム | <2s | 1.8s | ✅ |
| セキュリティスキャン | 0 Critical | 0件 | ✅ |

## リスク評価

### 残存リスク
1. **Mediumバグ8件残存**
   - リスクレベル: Low
   - 軽減策: すべて回避策あり、リリースノート記載

2. **新決済プロバイダー統合**
   - リスクレベル: Medium
   - 軽減策: Sandboxで十分テスト済み、カナリアリリース実施

### 技術的準備状況
- [x] インフラ準備完了
- [x] データベースバックアップ完了
- [x] ロールバック手順確認済み
- [x] 監視アラート設定完了

## ディスカッション要点

**佐藤（開発）**: 決済機能は十分テスト済み。問題ない。

**鈴木（PM）**: ビジネス的には予定通りリリースしたい。

**田中（SRE）**: インフラ準備OK。カナリアリリースで段階展開推奨。

**山田（QA）**: すべての基準を満たしている。リリース可と判断。

## 最終判定

**決定**: ✅ **GO**

**理由**: すべてのExit Criteriaを満たしており、残存リスクも許容範囲内。

**条件**:
- カナリアリリースで10%→50%→100%と段階的に展開
- 各段階で2時間の監視期間を設ける

## アクションアイテム

| アクション | 担当 | 期限 |
|-----------|------|------|
| カナリアリリース設定 | 田中 | 3/14 18:00 |
| リリースノート公開 | 鈴木 | 3/14 20:00 |
| 監視ダッシュボード確認 | 田中 | 3/14 22:00 |
| サポートチームブリーフィング | 佐藤 | 3/14 23:00 |

## 署名

- QAリード: 山田太郎 ✓ 2026-03-14
- 開発リード: 佐藤花子 ✓ 2026-03-14
- PM: 鈴木一郎 ✓ 2026-03-14
- SRE: 田中次郎 ✓ 2026-03-14
```

## 判定会議のベストプラクティス

### 1. データドリブンな判断

感情や政治的圧力ではなく、客観的なデータに基づいて判断します。

### 2. 全員の意見を尊重

QAだけでなく、すべてのステークホルダーの懸念を聞きます。

### 3. リスクの透明性

隠れたリスクがないか、オープンに議論します。

### 4. ビジネスと品質のバランス

ビジネス要件と品質基準のバランスを取ります。

### 5. 記録の保持

判定理由とプロセスを文書化し、将来の参考にします。

## まとめ

Go/No-Go判定は、リリースの成否を左右する重要な意思決定プロセスです。明確な基準、客観的なデータ、組織的な合意により、リスクを最小化しながらビジネス目標を達成できます。

条件付きGOの選択肢を持つことで、完璧を求めすぎずに現実的な判断が期待されます。すべての決定を文書化し、学習と改善に活用することが推奨されます。

次章では、リリース後の監視とモニタリングについて解説します。
