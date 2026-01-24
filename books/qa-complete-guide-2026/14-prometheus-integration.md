---
title: "Prometheus連携：メトリクス収集の自動化"
---

# Prometheus連携：メトリクス収集の自動化

## Prometheusとは

Prometheusは、時系列データを収集・保存するオープンソースの監視システムです。Pull型アーキテクチャでメトリクスを定期的に収集し、Grafanaと連携して可視化できます。

## PrometheusとGrafanaの統合

### Docker Composeでのセットアップ

```yaml
# docker-compose.yml
version: '3'
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning

volumes:
  prometheus-data:
  grafana-data:
```

### Prometheus設定ファイル

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'qa-metrics'
    static_configs:
      - targets: ['qa-exporter:8080']
        labels:
          environment: 'staging'

  - job_name: 'test-results'
    static_configs:
      - targets: ['test-exporter:8081']
```

## QAメトリクスのエクスポーター作成

### Node.js Exporter例

```typescript
// qa-metrics-exporter.ts
import express from 'express';
import promClient from 'prom-client';

const app = express();
const register = new promClient.Registry();

// デフォルトメトリクス（CPU、メモリなど）
promClient.collectDefaultMetrics({ register });

// カスタムメトリクス定義
const testCounter = new promClient.Counter({
  name: 'qa_tests_total',
  help: 'Total number of tests executed',
  labelNames: ['status', 'type'],
  registers: [register]
});

const testDuration = new promClient.Histogram({
  name: 'qa_test_duration_seconds',
  help: 'Test execution duration in seconds',
  labelNames: ['test_suite'],
  buckets: [1, 5, 10, 30, 60, 120, 300],
  registers: [register]
});

const codeCoverage = new promClient.Gauge({
  name: 'qa_code_coverage_percent',
  help: 'Code coverage percentage',
  labelNames: ['module'],
  registers: [register]
});

const bugCounter = new promClient.Gauge({
  name: 'qa_bugs_open',
  help: 'Number of open bugs',
  labelNames: ['severity'],
  registers: [register]
});

// テスト結果の記録
export function recordTestResult(type: string, status: string) {
  testCounter.inc({ status, type });
}

// テスト実行時間の記録
export function recordTestDuration(suite: string, duration: number) {
  testDuration.observe({ test_suite: suite }, duration);
}

// カバレッジの更新
export function updateCoverage(module: string, percent: number) {
  codeCoverage.set({ module }, percent);
}

// バグ数の更新
export function updateBugCount(severity: string, count: number) {
  bugCounter.set({ severity }, count);
}

// メトリクスエンドポイント
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

app.listen(8080, () => {
  console.log('QA Metrics Exporter running on :8080');
});
```

### テストスイートとの統合

```typescript
// test-reporter.ts
import { recordTestResult, recordTestDuration } from './qa-metrics-exporter';

export class PrometheusReporter {
  onTestResult(test: any, result: any) {
    const status = result.status; // 'passed' | 'failed' | 'skipped'
    const type = test.type; // 'unit' | 'integration' | 'e2e'
    const duration = result.duration / 1000; // ミリ秒を秒に変換

    recordTestResult(type, status);
    recordTestDuration(test.suite, duration);
  }
}
```

### Jest連携

```typescript
// jest.config.js
module.exports = {
  reporters: [
    'default',
    ['./test-reporter.ts', {}]
  ]
};
```

## Prometheusクエリ（PromQL）

### 基本的なクエリ

```promql
# テスト合格率（過去1時間）
sum(rate(qa_tests_total{status="passed"}[1h])) /
sum(rate(qa_tests_total[1h])) * 100

# 平均テスト実行時間
avg(qa_test_duration_seconds)

# Critical バグ数
qa_bugs_open{severity="critical"}

# テスト失敗率のトレンド
rate(qa_tests_total{status="failed"}[5m])
```

### 高度なクエリ

```promql
# テストタイプ別の成功率（過去24時間）
sum by (type) (rate(qa_tests_total{status="passed"}[24h])) /
sum by (type) (rate(qa_tests_total[24h])) * 100

# 95パーセンタイルのテスト実行時間
histogram_quantile(0.95, rate(qa_test_duration_seconds_bucket[5m]))

# カバレッジが80%未満のモジュール
qa_code_coverage_percent < 80

# バグ数の前日比
qa_bugs_open - qa_bugs_open offset 1d
```

## Grafanaダッシュボードでの活用

### Prometheusデータソース設定

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

### パネルクエリ例

```json
{
  "targets": [
    {
      "expr": "sum(rate(qa_tests_total{status=\"passed\"}[5m])) / sum(rate(qa_tests_total[5m])) * 100",
      "legendFormat": "合格率",
      "refId": "A"
    }
  ]
}
```

## アラートルールの定義

### Prometheus Alert Rules

```yaml
# alert-rules.yml
groups:
  - name: qa_alerts
    interval: 30s
    rules:
      - alert: HighTestFailureRate
        expr: |
          sum(rate(qa_tests_total{status="failed"}[5m])) /
          sum(rate(qa_tests_total[5m])) > 0.15
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "テスト失敗率が高い"
          description: "過去5分間のテスト失敗率が15%を超えています"

      - alert: CriticalBugsOpen
        expr: qa_bugs_open{severity="critical"} > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Critical バグが未解決"
          description: "{{ $value }}件のCriticalバグがオープンしています"

      - alert: LowCodeCoverage
        expr: avg(qa_code_coverage_percent) < 70
        for: 1h
        labels:
          severity: info
        annotations:
          summary: "カバレッジ低下"
          description: "平均コードカバレッジが70%を下回っています"

      - alert: SlowTestExecution
        expr: |
          histogram_quantile(0.95,
            rate(qa_test_duration_seconds_bucket[5m])
          ) > 300
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "テスト実行が遅い"
          description: "95パーセンタイルのテスト実行時間が5分を超えています"
```

## CI/CDパイプラインからのメトリクス送信

### GitHub Actions連携

```yaml
# .github/workflows/test-and-metrics.yml
name: Test and Send Metrics

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run tests
        run: npm test -- --json --outputFile=test-results.json

      - name: Send metrics to Prometheus Pushgateway
        run: |
          PASSED=$(jq '.numPassedTests' test-results.json)
          FAILED=$(jq '.numFailedTests' test-results.json)
          TOTAL=$(jq '.numTotalTests' test-results.json)

          cat <<EOF | curl --data-binary @- http://pushgateway:9091/metrics/job/ci_tests
          # TYPE qa_tests_total counter
          qa_tests_total{status="passed",type="ci"} $PASSED
          qa_tests_total{status="failed",type="ci"} $FAILED
          qa_tests_total{type="ci"} $TOTAL
          EOF
```

## メトリクスの保持期間設定

```bash
# Prometheusの起動オプション
prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.retention.time=30d \
  --storage.tsdb.retention.size=50GB
```

## ベストプラクティス

### 1. メトリクス命名規則

```
{namespace}_{subsystem}_{metric_name}_{unit}

例:
qa_tests_total
qa_test_duration_seconds
qa_code_coverage_percent
qa_bugs_open_count
```

### 2. ラベルの適切な使用

```typescript
// 良い例: 有限のラベル値
testCounter.inc({ status: 'passed', type: 'unit' });

// 悪い例: 無限のラベル値（カーディナリティ爆発）
testCounter.inc({ test_name: 'specific_test_12345' }); // NG
```

### 3. ヒストグラムのバケット設計

```typescript
// テスト実行時間に適したバケット
const testDuration = new promClient.Histogram({
  buckets: [0.1, 0.5, 1, 5, 10, 30, 60, 120, 300] // 秒単位
});
```

## まとめ

Prometheusとの連携により、QAメトリクスの自動収集と時系列データの蓄積が実現できます。カスタムエクスポーターを作成し、テストスイートやCI/CDパイプラインと統合することで、継続的なメトリクス監視が期待されます。

PromQLを使った柔軟なクエリとアラートルールにより、品質問題の早期検知と迅速な対応が可能になります。

次章では、リリース判定基準の設定方法について解説します。
