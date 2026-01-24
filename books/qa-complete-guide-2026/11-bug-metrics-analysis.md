---
title: "バグメトリクス分析：バグデータから学ぶ"
---

# バグメトリクス分析：バグデータから学ぶ

## バグメトリクスの重要性

バグデータは品質改善の宝庫です。バグの傾向を分析することで、コードの脆弱な部分、プロセスの課題、チームのスキルギャップなどが見えてきます。

## 基本的なバグメトリクス

### バグライフサイクルメトリクス

```
平均解決時間(MTTR) = 総解決時間 / 解決バグ数

平均オープン期間 = (クローズ日時 - 作成日時)の平均

再オープン率 = 再オープンバグ数 / クローズバグ数 × 100
```

### 深刻度別分布

| 深刻度 | 件数 | 割合 | SLA |
|--------|------|------|-----|
| Critical | 5 | 3% | 4時間 |
| High | 25 | 15% | 24時間 |
| Medium | 80 | 48% | 3日 |
| Low | 57 | 34% | 2週間 |

### ステータス別分布

```python
# バグステータスの可視化
import matplotlib.pyplot as plt

statuses = ['Open', 'In Progress', 'Resolved', 'Closed']
counts = [45, 32, 78, 245]

plt.pie(counts, labels=statuses, autopct='%1.1f%%')
plt.title('バグステータス分布')
plt.show()
```

## 高度なバグ分析

### バグ注入フェーズ分析

バグがどの工程で混入したかを分析します。

```
要件フェーズ: 15%
設計フェーズ: 25%
実装フェーズ: 45%
テストフェーズ: 10%
本番フェーズ: 5%
```

この分析から、実装フェーズでのコードレビュー強化が有効と判断できます。

### バグ検出フェーズ分析

どの段階でバグが発見されたかを分析します。

```python
# バグ検出効率
detection_phases = {
    'ユニットテスト': 120,
    '統合テスト': 85,
    'E2Eテスト': 45,
    'UAT': 20,
    '本番': 5
}

# 理想: 左側(早い段階)ほど多い
# 本番での発見は最小限に抑えたい
```

### モジュール別バグ密度

```python
# モジュール別バグ密度分析
modules = {
    'auth': {'loc': 2000, 'bugs': 8},      # 密度: 4.0
    'payment': {'loc': 3500, 'bugs': 21},  # 密度: 6.0 ⚠️
    'search': {'loc': 1500, 'bugs': 3},    # 密度: 2.0
    'admin': {'loc': 4000, 'bugs': 12},    # 密度: 3.0
}

for module, data in modules.items():
    density = (data['bugs'] / data['loc']) * 1000
    print(f'{module}: {density:.1f} bugs/KLOC')
```

決済モジュールの密度が高いため、優先的にリファクタリングやテスト強化が推奨されます。

## バグ傾向の可視化

### バーンダウンチャート

```python
# バグバーンダウンチャート
days = list(range(1, 31))
bugs_open = [120, 125, 130, 128, 120, 115, 110, 105, 100, 95,
             92, 88, 85, 80, 75, 72, 68, 65, 60, 55,
             52, 48, 45, 40, 38, 35, 30, 25, 20, 15]

plt.plot(days, bugs_open, marker='o')
plt.xlabel('日数')
plt.ylabel('未解決バグ数')
plt.title('バグバーンダウン')
plt.grid(True)
plt.show()
```

### 累積バグ推移

```python
# 発見と修正の累積推移
found = [10, 25, 42, 58, 75, 90, 102, 115, 125, 132]
fixed = [5, 18, 35, 50, 68, 85, 98, 112, 123, 130]

plt.plot(found, label='発見累積', marker='o')
plt.plot(fixed, label='修正累積', marker='s')
plt.xlabel('週')
plt.ylabel('バグ数')
plt.title('バグ累積推移')
plt.legend()
plt.grid(True)
plt.show()

# 収束判定: 発見曲線と修正曲線が接近したらリリース準備OK
```

## バグ根本原因分析（RCA）

### パレート分析

```python
# バグ原因のパレート図
causes = {
    '要件漏れ': 45,
    'ロジックエラー': 38,
    'API連携ミス': 25,
    'UI/UX問題': 18,
    '性能問題': 12,
    'その他': 10
}

# 上位3つで70%以上を占める場合、そこに注力
total = sum(causes.values())
cumulative = 0
for cause, count in sorted(causes.items(), key=lambda x: x[1], reverse=True):
    cumulative += count
    ratio = cumulative / total * 100
    print(f'{cause}: {count}件 (累積: {ratio:.1f}%)')
```

### 5 Whys分析テンプレート

```markdown
## バグ根本原因分析 - BUG-123

**現象**: 決済完了後にエラー画面が表示される

**Why 1**: なぜエラーが発生したか？
→ 決済APIからのレスポンス処理に失敗

**Why 2**: なぜレスポンス処理に失敗したか？
→ 新しいステータスコード(202)に対応していなかった

**Why 3**: なぜ対応していなかったか？
→ API仕様変更が開発チームに伝わっていなかった

**Why 4**: なぜ伝わっていなかったか？
→ API仕様書の変更通知プロセスがなかった

**Why 5**: なぜプロセスがなかったか？
→ 外部API連携の変更管理ルールが未整備

**根本原因**: 外部API変更管理プロセスの欠如

**再発防止策**:
1. API仕様変更の通知フローを確立
2. API仕様書を監視するCIジョブを追加
3. 契約テストの導入を検討
```

## バグ予測モデル

### 単純な予測式

```python
# 残存バグ予測
import numpy as np

# 過去データから回帰分析
days = np.array([1, 2, 3, 4, 5])
bugs_found = np.array([20, 35, 48, 58, 65])

# 対数近似でバグ発見の飽和を予測
coeffs = np.polyfit(np.log(days), bugs_found, 1)
predict_day_10 = coeffs[0] * np.log(10) + coeffs[1]

print(f'10日後の累積発見バグ予測: {predict_day_10:.0f}件')
```

## バグメトリクスのSLA管理

### 深刻度別SLA

```yaml
bug_sla:
  critical:
    response_time: 1h
    resolution_time: 4h
    escalation: immediate

  high:
    response_time: 4h
    resolution_time: 24h
    escalation: 8h

  medium:
    response_time: 1d
    resolution_time: 3d
    escalation: 2d

  low:
    response_time: 3d
    resolution_time: 14d
    escalation: 7d
```

### SLA違反の追跡

```python
# SLA違反バグの検出
from datetime import datetime, timedelta

bugs = [
    {'id': 'BUG-001', 'severity': 'critical',
     'created': '2026-03-01 10:00', 'resolved': '2026-03-01 15:00'},
    {'id': 'BUG-002', 'severity': 'critical',
     'created': '2026-03-01 11:00', 'resolved': '2026-03-01 17:00'},
]

sla = {'critical': 4}  # 4時間

for bug in bugs:
    created = datetime.fromisoformat(bug['created'])
    resolved = datetime.fromisoformat(bug['resolved'])
    duration = (resolved - created).total_seconds() / 3600

    sla_limit = sla[bug['severity']]
    if duration > sla_limit:
        print(f"⚠️ SLA違反: {bug['id']} ({duration:.1f}h > {sla_limit}h)")
```

## バグメトリクスダッシュボード

### Grafanaパネル構成例

```json
{
  "dashboard": {
    "title": "バグメトリクス",
    "panels": [
      {
        "title": "深刻度別バグ数",
        "type": "piechart",
        "targets": [
          {
            "query": "SELECT severity, COUNT(*) FROM bugs GROUP BY severity"
          }
        ]
      },
      {
        "title": "バグトレンド",
        "type": "graph",
        "targets": [
          {
            "query": "SELECT date, COUNT(*) FROM bugs WHERE status='open' GROUP BY date"
          }
        ]
      },
      {
        "title": "平均解決時間",
        "type": "stat",
        "targets": [
          {
            "query": "SELECT AVG(resolution_time) FROM bugs WHERE resolved_at IS NOT NULL"
          }
        ]
      }
    ]
  }
}
```

## まとめ

バグメトリクスの分析により、品質の現状把握だけでなく、将来の予測や根本原因の特定が期待されます。深刻度別分布、モジュール別密度、検出フェーズ分析などを組み合わせることで、効果的な改善策を導き出せます。

パレート分析や5 Whysを使った根本原因分析により、表面的な対症療法ではなく、本質的な品質改善が期待されます。

次章では、テストカバレッジメトリクスについて詳しく解説します。
