---
title: "コンテナ監視"
---

## この章で学ぶこと

コンテナを本番環境で運用するとき、「今、何が起きているか」を把握できる仕組みが不可欠です。この章では、Docker 環境におけるモニタリングの基本から実践的な監視スタックの構築まで、段階的に学んでいきます。

- `docker stats` による簡易モニタリング
- cAdvisor でコンテナメトリクスを収集する方法
- Prometheus + Grafana による本格的な監視ダッシュボード構築
- Fluentd / Loki を使ったログ収集の仕組み
- アラート設定で障害を即座に検知する方法

---

## docker stats で始める簡易モニタリング

まずは Docker に標準搭載されているコマンドから始めましょう。`docker stats` は追加ツールなしでコンテナのリソース状況をリアルタイムに確認できます。

```bash
# 全コンテナのリソース使用状況をリアルタイム表示
docker stats

# 出力例:
# CONTAINER ID   NAME   CPU %   MEM USAGE / LIMIT    MEM %   NET I/O
# a1b2c3d4e5f6   api    2.35%   128.5MiB / 512MiB    25.10%  1.2MB / 850kB
# f6e5d4c3b2a1   db     0.89%   256.1MiB / 1GiB      25.01%  900kB / 1.5MB

# 特定コンテナのみ表示
docker stats api db

# ストリーミングせず1回だけ取得（スクリプト向き）
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

`docker stats` は手軽ですが、データの保存やグラフ化はできません。本番運用では、次に紹介するツール群を組み合わせた監視基盤を構築します。

---

## cAdvisor でコンテナメトリクスを収集する

cAdvisor（Container Advisor）は Google が開発したオープンソースツールです。各コンテナの CPU、メモリ、ネットワーク、ディスク I/O のメトリクスを自動的に収集し、Web UI や API で公開します。

```yaml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    container_name: cadvisor
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"
```

起動後、`http://localhost:8080` で各コンテナのリソース使用状況をグラフで確認できます。cAdvisor が真価を発揮するのは Prometheus と連携するときです。`http://cadvisor:8080/metrics` から Prometheus 形式のメトリクスを取得できます。

---

## Prometheus + Grafana で本格的な監視基盤を構築する

Prometheus はメトリクスを定期的に「Pull（取得）」し、時系列データとして保存します。Grafana はそのデータをダッシュボードとして可視化します。全体の流れは **アプリ -> cAdvisor -> Prometheus -> Grafana / Alertmanager** です。

### Docker Compose で監視スタックを構築する

```yaml
# docker-compose.monitoring.yml
services:
  prometheus:
    image: prom/prometheus:v2.51.0
    container_name: prometheus
    restart: unless-stopped
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=30d"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alert-rules.yml:/etc/prometheus/alert-rules.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - monitoring

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    container_name: cadvisor
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:10.4.0
    container_name: grafana
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin}
    volumes:
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus-data:
  grafana-data:
```

### Prometheus の設定ファイル

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert-rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]

  - job_name: "app"
    static_configs:
      - targets: ["api:8080"]
    metrics_path: /metrics
```

### よく使う PromQL クエリ

Grafana でダッシュボードを作成する際に、以下のクエリが役立ちます。

```promql
# コンテナ別 CPU 使用率（%）
sum(rate(container_cpu_usage_seconds_total{name=~".+"}[5m])) by (name) * 100

# コンテナ別メモリ使用量（MB）
container_memory_usage_bytes{name=~".+"} / 1024 / 1024

# ネットワーク受信量（バイト/秒）
sum(rate(container_network_receive_bytes_total{name=~".+"}[5m])) by (name)

# HTTP リクエストレート（リクエスト/秒）
sum(rate(http_requests_total[5m])) by (service)

# レスポンスタイム P95
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

Grafana にログインしたら、データソースに Prometheus（`http://prometheus:9090`）を追加し、これらのクエリでパネルを作成していきます。

---

## ログ収集の仕組み（Fluentd / Loki）

メトリクスだけでなく、ログの集約も運用には欠かせません。代表的な 2 つのアプローチを紹介します。

### Fluentd によるログ転送

Fluentd は CNCF 卒業プロジェクトで、多様な入出力プラグインを持つログコレクターです。Docker のログドライバーとして統合できます。

```yaml
services:
  fluentd:
    image: fluent/fluentd:v1.16-1
    volumes:
      - ./fluentd/conf:/fluentd/etc
    ports:
      - "24224:24224"

  app:
    image: my-app:latest
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: app.{{.Name}}
```

アプリケーションのログが自動的に Fluentd に転送され、Elasticsearch や S3 など任意の出力先に保存できます。

### Loki + Promtail（Grafana エコシステム）

Grafana Loki は「ログ版 Prometheus」とも呼ばれる軽量なログ集約システムです。Grafana との親和性が高く、メトリクスとログを同一画面で確認できます。

```yaml
# docker-compose.logging.yml
services:
  loki:
    image: grafana/loki:2.9.6
    container_name: loki
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - ./loki/config.yaml:/etc/loki/local-config.yaml:ro
      - loki-data:/loki
    ports:
      - "3100:3100"

  promtail:
    image: grafana/promtail:2.9.6
    container_name: promtail
    command: -config.file=/etc/promtail/config.yml
    volumes:
      - ./promtail/config.yml:/etc/promtail/config.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro

volumes:
  loki-data:
```

Promtail は Docker ソケット経由でコンテナのログを自動検出し、Loki に転送します。Grafana のデータソースに Loki（`http://loki:3100`）を追加すると、LogQL でログを検索できます。

```logql
# 特定コンテナのエラーログを検索
{container="api"} |= "error"

# JSON ログからステータスコード 500 以上を抽出
{service="api"} | json | status >= 500

# エラーログの発生頻度を時系列で表示
rate({service="api"} |= "error" [5m])
```

### Fluentd vs Loki：どちらを選ぶか

| 観点 | Fluentd | Loki + Promtail |
|------|---------|-----------------|
| 用途 | 多様な出力先への転送 | Grafana でのログ可視化 |
| セットアップ | プラグイン設定が必要 | 比較的シンプル |
| リソース消費 | 中程度 | 軽量 |
| Grafana 連携 | プラグイン経由 | ネイティブ対応 |
| おすすめ場面 | 既存の Elasticsearch/S3 連携 | Prometheus + Grafana 環境 |

Prometheus + Grafana の監視スタックを構築しているなら、Loki を選ぶとシームレスに統合できます。

---

## アラート設定で障害を即座に検知する

監視データを集めても、人が常にダッシュボードを見ているわけにはいきません。異常を自動検知して通知する仕組みが必要です。

### Prometheus アラートルールの定義

```yaml
# prometheus/alert-rules.yml
groups:
  - name: container-alerts
    rules:
      - alert: ContainerDown
        expr: absent(container_last_seen{name=~".+"})
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "コンテナ {{ $labels.name }} が停止しています"

      - alert: ContainerHighCPU
        expr: >
          sum(rate(container_cpu_usage_seconds_total{name=~".+"}[5m]))
          by (name) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU使用率が80%超過（{{ $value | printf \"%.1f\" }}%）"

      - alert: ContainerHighMemory
        expr: >
          container_memory_usage_bytes{name=~".+"}
          / container_spec_memory_limit_bytes{name=~".+"} * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "メモリ使用率が85%超過（{{ $value | printf \"%.1f\" }}%）"

      - alert: ContainerOOMKilled
        expr: increase(container_oom_events_total{name=~".+"}[5m]) > 0
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.name }} がOOM Killされました"
```

`for` フィールドは「その状態がどれだけ続いたら発報するか」を指定します。短すぎるとアラート疲れを起こし、長すぎると検知が遅れます。5 分程度から始めて調整するのがおすすめです。

### Alertmanager で通知先を設定する

Alertmanager は Prometheus からのアラートを受け取り、Slack やメールに通知します。Docker Compose に以下を追加します。

```yaml
# docker-compose.monitoring.yml に追加
  alertmanager:
    image: prom/alertmanager:v0.27.0
    container_name: alertmanager
    command:
      - "--config.file=/etc/alertmanager/alertmanager.yml"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    ports:
      - "9093:9093"
    networks:
      - monitoring
```

```yaml
# alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m

route:
  group_by: ["alertname"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: "slack-notifications"
  routes:
    - match:
        severity: critical
      receiver: "slack-critical"
      repeat_interval: 1h

receivers:
  - name: "slack-notifications"
    slack_configs:
      - channel: "#alerts"
        api_url: "${SLACK_WEBHOOK_URL}"
        title: '{{ .GroupLabels.alertname }}'
        text: >-
          {{ range .Alerts }}
          *{{ .Annotations.summary }}*
          {{ end }}
        send_resolved: true

  - name: "slack-critical"
    slack_configs:
      - channel: "#alerts-critical"
        api_url: "${SLACK_WEBHOOK_URL}"
        send_resolved: true
```

`severity: critical` のアラートは専用チャンネルに送り、1 時間ごとに再通知します。`send_resolved: true` を設定すると、問題が解消されたときにも通知が届くため、対応完了の確認に役立ちます。

---

## まとめ

| ツール/手法 | 役割 | 適用場面 |
|------------|------|---------|
| `docker stats` | リアルタイムのリソース確認 | 開発中・簡易的な確認 |
| cAdvisor | コンテナメトリクスの収集・公開 | Prometheus と連携した本格監視 |
| Prometheus | メトリクスの保存・クエリ | 時系列データの蓄積と分析 |
| Grafana | ダッシュボードによる可視化 | チーム全体での状況共有 |
| Fluentd | 多様な出力先へのログ転送 | Elasticsearch/S3 など既存基盤との連携 |
| Loki + Promtail | 軽量なログ集約・検索 | Grafana エコシステムでの統合監視 |
| Alertmanager | アラート通知のルーティング | Slack/メールへの障害通知 |

---

## やってみよう！

1. `docker stats` コマンドを実行して、手元で動いているコンテナのリソース使用状況を確認してみましょう
2. cAdvisor を Docker Compose で起動し、`http://localhost:8080` でコンテナメトリクスを Web UI で確認してみましょう
3. Prometheus + Grafana の監視スタックを構築し、cAdvisor のメトリクスをダッシュボードに表示してみましょう
4. Loki + Promtail を追加して、Grafana 上でコンテナのログを検索してみましょう
5. アラートルールを 1 つ作成し、意図的にしきい値を超える負荷をかけて通知が届くか確認してみましょう
