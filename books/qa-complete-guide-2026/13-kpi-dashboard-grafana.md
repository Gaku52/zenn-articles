---
title: "Grafana KPIダッシュボード：品質の可視化"
---

# Grafana KPIダッシュボード：品質の可視化

## Grafanaとは

Grafanaは、メトリクスを可視化するためのオープンソースのダッシュボードツールです。時系列データの可視化に優れており、QAメトリクスのリアルタイム監視に最適です。

## Grafanaのセットアップ

### Docker Composeでの起動

```yaml
# docker-compose.yml
version: '3'
services:
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_INSTALL_PLUGINS=grafana-piechart-panel
    volumes:
      - grafana-storage:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning

volumes:
  grafana-storage:
```

```bash
docker-compose up -d
# http://localhost:3000 でアクセス
```

## データソースの設定

### PostgreSQLデータソース

```yaml
# grafana/provisioning/datasources/postgres.yml
apiVersion: 1

datasources:
  - name: QA Metrics DB
    type: postgres
    url: postgres:5432
    database: qa_metrics
    user: grafana
    secureJsonData:
      password: 'your-password'
    jsonData:
      sslmode: 'disable'
      postgresVersion: 1500
```

### メトリクステーブルの作成

```sql
CREATE TABLE test_metrics (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMP NOT NULL,
    test_type VARCHAR(50),
    total_tests INTEGER,
    passed INTEGER,
    failed INTEGER,
    skipped INTEGER,
    duration_seconds INTEGER,
    coverage_percent DECIMAL(5,2)
);

CREATE TABLE bug_metrics (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMP NOT NULL,
    severity VARCHAR(20),
    status VARCHAR(20),
    count INTEGER
);
```

## QAダッシュボードの構築

### ダッシュボード構成

```json
{
  "dashboard": {
    "title": "QA メトリクス ダッシュボード",
    "timezone": "Asia/Tokyo",
    "refresh": "5m",
    "time": {
      "from": "now-7d",
      "to": "now"
    },
    "panels": []
  }
}
```

### パネル1: テスト実行状況

```json
{
  "title": "テスト実行トレンド",
  "type": "graph",
  "gridPos": {"x": 0, "y": 0, "w": 12, "h": 8},
  "targets": [
    {
      "rawSql": "SELECT timestamp, total_tests, passed, failed FROM test_metrics WHERE $__timeFilter(timestamp) ORDER BY timestamp",
      "format": "time_series"
    }
  ],
  "yaxes": [
    {"label": "テスト数", "format": "short"}
  ]
}
```

### パネル2: テスト合格率

```json
{
  "title": "テスト合格率",
  "type": "stat",
  "gridPos": {"x": 12, "y": 0, "w": 6, "h": 4},
  "targets": [
    {
      "rawSql": "SELECT (SUM(passed)::float / SUM(total_tests) * 100) as value FROM test_metrics WHERE timestamp > NOW() - INTERVAL '7 days'"
    }
  ],
  "options": {
    "unit": "percent",
    "thresholds": {
      "steps": [
        {"value": 0, "color": "red"},
        {"value": 90, "color": "yellow"},
        {"value": 95, "color": "green"}
      ]
    }
  }
}
```

### パネル3: バグ深刻度分布

```json
{
  "title": "バグ深刻度分布",
  "type": "piechart",
  "gridPos": {"x": 0, "y": 8, "w": 8, "h": 8},
  "targets": [
    {
      "rawSql": "SELECT severity as metric, SUM(count) as value FROM bug_metrics WHERE status = 'open' AND timestamp > NOW() - INTERVAL '1 day' GROUP BY severity"
    }
  ],
  "options": {
    "legend": {
      "displayMode": "table",
      "values": ["value", "percent"]
    }
  }
}
```

### パネル4: カバレッジトレンド

```json
{
  "title": "コードカバレッジ推移",
  "type": "graph",
  "gridPos": {"x": 8, "y": 8, "w": 16, "h": 8},
  "targets": [
    {
      "rawSql": "SELECT timestamp, coverage_percent FROM test_metrics WHERE $__timeFilter(timestamp) ORDER BY timestamp"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "min": 0,
      "max": 100,
      "thresholds": {
        "steps": [
          {"value": 0, "color": "red"},
          {"value": 70, "color": "yellow"},
          {"value": 80, "color": "green"}
        ]
      }
    }
  }
}
```

### パネル5: 平均テスト実行時間

```json
{
  "title": "平均テスト実行時間",
  "type": "stat",
  "gridPos": {"x": 18, "y": 0, "w": 6, "h": 4},
  "targets": [
    {
      "rawSql": "SELECT AVG(duration_seconds) as value FROM test_metrics WHERE timestamp > NOW() - INTERVAL '7 days'"
    }
  ],
  "options": {
    "unit": "s",
    "graphMode": "area",
    "colorMode": "value"
  }
}
```

## 変数（Variables）の活用

### プロジェクト選択変数

```json
{
  "templating": {
    "list": [
      {
        "name": "project",
        "type": "query",
        "query": "SELECT DISTINCT project_name FROM test_metrics",
        "current": {"text": "All", "value": "$__all"},
        "includeAll": true
      }
    ]
  }
}
```

クエリで変数を使用:
```sql
SELECT * FROM test_metrics
WHERE project_name = '$project'
AND $__timeFilter(timestamp)
```

## アラート設定

### テスト失敗率アラート

```json
{
  "alert": {
    "name": "High Test Failure Rate",
    "conditions": [
      {
        "type": "query",
        "query": {
          "model": {
            "rawSql": "SELECT (SUM(failed)::float / SUM(total_tests) * 100) as value FROM test_metrics WHERE timestamp > NOW() - INTERVAL '1 hour'"
          }
        },
        "reducer": {"type": "avg"},
        "evaluator": {
          "type": "gt",
          "params": [15]
        }
      }
    ],
    "frequency": "5m",
    "for": "10m",
    "notifications": [
      {"uid": "slack-channel"}
    ],
    "message": "テスト失敗率が15%を超えています"
  }
}
```

### Critical バグアラート

```json
{
  "alert": {
    "name": "Critical Bugs Detected",
    "conditions": [
      {
        "query": {
          "rawSql": "SELECT SUM(count) as value FROM bug_metrics WHERE severity = 'critical' AND status = 'open' AND timestamp > NOW() - INTERVAL '1 hour'"
        },
        "evaluator": {
          "type": "gt",
          "params": [0]
        }
      }
    ],
    "notifications": [
      {"uid": "pagerduty"}
    ],
    "message": "Critical バグが検出されました"
  }
}
```

## 通知チャネルの設定

### Slack通知

```yaml
# grafana/provisioning/notifiers/slack.yml
notifiers:
  - name: QA Slack Channel
    type: slack
    uid: slack-channel
    settings:
      url: https://hooks.slack.com/services/YOUR/WEBHOOK/URL
      recipient: '#qa-alerts'
      username: Grafana QA
```

### メール通知

```yaml
notifiers:
  - name: QA Team Email
    type: email
    uid: qa-email
    settings:
      addresses: qa-team@example.com
```

## ダッシュボードのエクスポート/インポート

### エクスポート

```bash
# ダッシュボードをJSON形式でエクスポート
curl -H "Authorization: Bearer $API_KEY" \
  http://localhost:3000/api/dashboards/uid/qa-dashboard > dashboard.json
```

### インポート

```bash
# ダッシュボードをインポート
curl -X POST -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d @dashboard.json \
  http://localhost:3000/api/dashboards/db
```

## ベストプラクティス

### 1. 階層的なダッシュボード

- **概要ダッシュボード**: 経営層向け（KPIサマリー）
- **詳細ダッシュボード**: QAチーム向け（詳細メトリクス）
- **リアルタイムダッシュボード**: 日常業務用（テスト実行状況）

### 2. 適切なリフレッシュレート

```
リアルタイム監視: 30秒〜1分
日次レポート: 5〜10分
週次レビュー: リフレッシュなし（手動）
```

### 3. パネルの配置

```
重要なメトリクスは左上に配置
詳細データは下部に配置
関連するパネルは隣接配置
```

### 4. カラースキーマの統一

```
緑: 正常/成功/合格
黄: 警告/注意
赤: エラー/失敗/Critical
青: 情報/中立
```

## まとめ

Grafanaを使ったKPIダッシュボードにより、QAメトリクスのリアルタイム可視化が実現できます。適切なパネル構成、変数の活用、アラート設定により、データドリブンな品質管理が期待されます。

ダッシュボードは一度作って終わりではなく、チームのフィードバックを基に継続的に改善していくことが推奨されます。

次章では、Prometheusとの統合によるメトリクス収集について解説します。
