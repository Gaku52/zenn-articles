---
title: "本番デプロイ"
---

AI エージェントを開発環境で動かせるようになったら、次は本番環境へのデプロイです。この章では、エージェントを安定的に運用するために必要な**デプロイ戦略、コスト管理、レート制限、モニタリング、ログ収集、スケーリング**について解説します。

## デプロイ戦略の選択

エージェントのデプロイ方式は、大きく 3 つに分類できます。

### サーバーレス（Lambda / Cloud Functions）

```
[リクエスト] → [API Gateway] → [Lambda] → [LLM API]
```

- メリット: スケール自動、コスト従量制、インフラ管理不要
- デメリット: タイムアウト制限（通常 15 分）、コールドスタート遅延
- 適する場面: 短時間で完了するタスク、リクエスト頻度が低い場合

### コンテナ（ECS / Cloud Run / Kubernetes）

```
[リクエスト] → [ALB] → [ECS Fargate Container] → [LLM API]
```

- メリット: 柔軟な構成、長時間実行が可能、リソース制御が細かい
- デメリット: インフラ管理のコストが発生する
- 適する場面: 中〜大規模のプロダクション環境

### キューベース（非同期処理）

```
[リクエスト] → [API] → [SQS/Redis Queue] → [Worker] → [Callback/Webhook]
```

- メリット: 長時間タスクに最適、バックプレッシャー制御が可能
- デメリット: リアルタイム応答が難しい
- 適する場面: 数分以上かかるエージェントタスク

多くの本番環境では、**コンテナ + キューベースのハイブリッド構成**が現実的な選択肢です。短い応答は同期的にコンテナで処理し、長時間タスクはキュー経由でワーカーに委任します。

## コンテナデプロイの実装例

コンテナベースのデプロイでは、Dockerfile と docker-compose を使った構成が基本になります。

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# 依存関係のインストール（キャッシュ活用のため先にコピー）
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# アプリケーションコード
COPY src/ ./src/

# セキュリティ: 非rootユーザーで実行
RUN useradd --create-home appuser
USER appuser

# ヘルスチェック
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080
CMD ["python", "-m", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

ポイントは 3 つです。

1. **非 root ユーザー**で実行することで、コンテナ内の権限を最小化します
2. **ヘルスチェック**を設定し、異常時に自動再起動させます
3. **依存関係を先にコピー**することで、Docker のビルドキャッシュを活用します

## コスト管理

LLM API の利用コストはエージェント運用で最大の費用項目です。制御なしでは予想外の請求が発生します。

### トークンコストの計算

```python
class CostController:
    """タスク単位・日次単位でコストを制御するクラス"""

    # モデル別の料金（1Mトークンあたり USD）
    RATES = {
        "claude-sonnet-4-20250514": {"input": 3.0, "output": 15.0},
        "claude-haiku-4-20250514": {"input": 0.25, "output": 1.25},
    }

    def __init__(self, daily_budget: float = 100.0, per_task_limit: float = 5.0):
        self.daily_budget = daily_budget
        self.per_task_limit = per_task_limit
        self.daily_spend = 0.0
        self.task_spend = 0.0

    def track_usage(self, input_tokens: int, output_tokens: int, model: str):
        rates = self.RATES.get(model, {"input": 3.0, "output": 15.0})
        cost = (input_tokens * rates["input"] + output_tokens * rates["output"]) / 1_000_000
        self.daily_spend += cost
        self.task_spend += cost

    def check_budget(self):
        if self.daily_spend >= self.daily_budget:
            raise Exception("日次予算の上限に達しました")
        if self.task_spend >= self.per_task_limit:
            raise Exception("タスク予算の上限に達しました")
```

### コスト最適化の 3 つの戦略

**1. モデルルーティング**: タスクの複雑さに応じてモデルを切り替えます。簡単な分類タスクには Haiku、複雑な推論には Sonnet を使い分けることで、コストを大幅に削減できます。

**2. レスポンスキャッシュ**: 同一入力に対する LLM の応答を Redis 等にキャッシュします。FAQ 対応のようなパターンでは、キャッシュヒット率が高くなります。

**3. プロンプト最適化**: 不要なコンテキストを削減し、入力トークン数を抑えます。会話履歴が長くなった場合は古いメッセージを要約することも有効です。

## レート制限への対処

LLM API にはリクエスト数（RPM）とトークン数（TPM）のレート制限があります。本番環境では、この制限に適応する仕組みが必須です。

```python
import asyncio
import time
from collections import deque

class AdaptiveRateLimiter:
    """LLM APIのレート制限に適応するリミッター"""

    def __init__(self, requests_per_minute: int = 60, tokens_per_minute: int = 100_000):
        self.rpm_limit = requests_per_minute
        self.tpm_limit = tokens_per_minute
        self.request_timestamps = deque()
        self.token_usage = deque()
        self._lock = asyncio.Lock()

    async def acquire(self, estimated_tokens: int = 1000):
        """レート制限を考慮してリクエスト権を取得"""
        async with self._lock:
            now = time.time()
            cutoff = now - 60

            # 1分以上前のエントリを除去
            while self.request_timestamps and self.request_timestamps[0] < cutoff:
                self.request_timestamps.popleft()
            while self.token_usage and self.token_usage[0][0] < cutoff:
                self.token_usage.popleft()

            # RPM超過の場合は待機
            if len(self.request_timestamps) >= self.rpm_limit:
                wait_time = 60 - (now - self.request_timestamps[0])
                if wait_time > 0:
                    await asyncio.sleep(wait_time)

            # TPM超過の場合は待機
            current_tokens = sum(t[1] for t in self.token_usage)
            if current_tokens + estimated_tokens > self.tpm_limit:
                wait_time = 60 - (now - self.token_usage[0][0])
                if wait_time > 0:
                    await asyncio.sleep(wait_time)

            self.request_timestamps.append(time.time())
            self.token_usage.append((time.time(), estimated_tokens))
```

429（Too Many Requests）エラーを受け取った場合は、**指数バックオフ**で再試行します。1 秒、2 秒、4 秒...と待機時間を倍増させることで、API 側の負荷を軽減しながら復帰を待ちます。

## モニタリング

エージェントの本番運用では、以下の 4 カテゴリのメトリクスを監視します。

### 収集すべきメトリクス

| カテゴリ | メトリクス | 目的 |
|---------|----------|------|
| パフォーマンス | レイテンシ（P50/P95/P99） | 応答速度の把握 |
| パフォーマンス | ステップ数の分布 | エージェントの効率性 |
| 信頼性 | 成功率 / エラー率 | 障害検知 |
| 信頼性 | アクティブエージェント数 | リソース使用状況 |
| コスト | API コスト（時間/日次） | 予算管理 |
| コスト | トークン消費量 | 使用量の傾向把握 |
| ツール | ツール呼び出し回数・成功率 | ツールの健全性 |

### Prometheus によるメトリクス収集

```python
from prometheus_client import Counter, Histogram, Gauge

# リクエスト数
AGENT_REQUESTS = Counter(
    "agent_requests_total", "Total agent requests",
    ["status", "intent"]
)
# レイテンシ分布
AGENT_LATENCY = Histogram(
    "agent_latency_seconds", "Agent response latency",
    buckets=[1, 5, 10, 30, 60, 120, 300]
)
# APIコスト
AGENT_COST = Counter("agent_cost_usd", "Total API cost in USD")
# 現在のアクティブエージェント数
ACTIVE_AGENTS = Gauge("active_agents", "Currently running agent instances")
```

これらのメトリクスを Grafana ダッシュボードで可視化することで、リアルタイムにシステム状態を把握できます。

### アラート設計

以下のようなアラートルールを設定しておくと、問題の早期検知が可能です。

```yaml
# Prometheusアラートルールの例
groups:
  - name: agent_alerts
    rules:
      - alert: AgentHighErrorRate
        expr: >
          sum(rate(agent_requests_total{status="error"}[5m]))
          / sum(rate(agent_requests_total[5m])) > 0.1
        for: 5m
        annotations:
          summary: "エラー率が10%を超えています"

      - alert: AgentCostExceeded
        expr: increase(agent_cost_usd[1h]) > 50
        annotations:
          summary: "1時間あたりのコストが$50を超えました"
```

## ログ収集

エージェントのデバッグには、**構造化ログ**が不可欠です。`print` 文ではなく、JSON 形式のログを出力しましょう。

```python
import structlog

logger = structlog.get_logger()

class LoggedAgent:
    def run(self, task: str, request_id: str):
        log = logger.bind(request_id=request_id)
        log.info("agent_started", task=task[:100])

        for step in range(self.max_steps):
            log.info("llm_call", step=step, model=self.model)
            response = self._call_llm()

            for tool_call in response.tool_calls:
                log.info("tool_call", step=step, tool=tool_call.name)
                result = self._execute_tool(tool_call)
                log.info("tool_result", step=step, tool=tool_call.name,
                         success=not str(result).startswith("Error"))

        log.info("agent_completed",
                 total_steps=step, total_tokens=self.token_count,
                 cost_usd=self.cost)
```

構造化ログのメリットは次の通りです。

1. **検索性**: `request_id` でリクエスト全体のログをフィルタリングできます
2. **集計**: ツールごとの成功率やステップ数を集計できます
3. **可視化**: Elasticsearch + Kibana や Loki + Grafana でダッシュボード化できます

さらに高度な運用では、**OpenTelemetry による分散トレーシング**を導入します。LLM 呼び出し、ツール実行、外部 API コールの各スパンを紐付けることで、ボトルネックの特定が容易になります。

## スケーリング

エージェントのスケーリングでは、通常の Web アプリケーションとは異なる課題があります。1 リクエストあたりの処理時間が長く（数秒〜数分）、外部 API のレート制限という制約があるためです。

### 水平スケーリングの設計

```
                    +---> [Worker 1] --+
                    |                  |
[Queue] --dispatch--+---> [Worker 2] --+---> [Result Store]
                    |                  |
                    +---> [Worker N] --+

スケーリングポリシー:
- キュー深さ > 100  → ワーカー追加
- キュー深さ < 10   → ワーカー削減
- CPU使用率 > 70%   → ワーカー追加
- LLM APIレート制限  → リクエスト間隔を調整
```

Kubernetes の HPA（Horizontal Pod Autoscaler）を使う場合は、CPU 使用率だけでなく**キューの深さ**をカスタムメトリクスとしてスケーリングの判断基準に加えましょう。エージェントは CPU よりも LLM API のレスポンス待ちで時間を消費するため、CPU だけでは適切にスケーリングできません。

## 障害回復の仕組み

本番環境では障害は「起きるかどうか」ではなく「いつ起きるか」の問題です。3 つのパターンを組み合わせて備えましょう。

### 1. リトライ（指数バックオフ）

一時的な障害に対して、待機時間を倍増させながら再試行します。

```
[失敗] → [1秒待機] → [再試行] → [2秒待機] → [再試行] → [4秒待機] → ...
```

### 2. フォールバック

プライマリの LLM API が使えない場合に、別のプロバイダやモデルに切り替えます。

```
[Claude API 障害] → [OpenAI API にフォールバック]
[Sonnet が遅延]   → [Haiku にダウングレード]
```

### 3. サーキットブレーカー

連続してエラーが発生した場合に、一定時間 API への呼び出しを停止します。障害中の API に無駄なリクエストを送り続けることを防ぎます。

```python
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"       # 正常: リクエストを通す
    OPEN = "open"           # 障害中: リクエストを拒否
    HALF_OPEN = "half_open" # 回復確認中: 一部だけ通す

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30):
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.last_failure_time = 0

    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
            else:
                raise Exception("サーキットブレーカーがオープン状態です")
        try:
            result = func(*args, **kwargs)
            self.failure_count = 0
            self.state = CircuitState.CLOSED
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = CircuitState.OPEN
            raise
```

## デプロイ前チェックリスト

本番デプロイの前に、以下の項目を確認しましょう。

- [ ] ヘルスチェックエンドポイントが実装されている
- [ ] API キーがシークレットマネージャーで管理されている（ハードコードしない）
- [ ] コスト制限（日次・タスク単位）が設定されている
- [ ] 構造化ログが実装されている
- [ ] メトリクス収集とアラートが設定されている
- [ ] リトライ・フォールバックの障害回復が実装されている
- [ ] 非 root ユーザーでコンテナが実行される
- [ ] オートスケーリングポリシーが設定されている
- [ ] ロールバック手順が文書化されている

## まとめ

| 項目 | ポイント |
|------|---------|
| デプロイ方式 | コンテナ + キューのハイブリッドが現実的。サーバーレスは短時間タスク向き |
| コスト管理 | 日次/タスク単位の予算制限を必ず設定。モデルルーティングとキャッシュで最適化 |
| レート制限 | 適応型リミッターで RPM/TPM を管理。429 エラーには指数バックオフで対処 |
| モニタリング | レイテンシ、エラー率、コスト、ツール呼び出しの 4 カテゴリを監視 |
| ログ収集 | 構造化ログ + request_id による追跡。print 文は使わない |
| スケーリング | キュー深さベースの水平スケーリング。CPU だけでなくカスタムメトリクスを活用 |
| 障害回復 | リトライ、フォールバック、サーキットブレーカーの 3 層で備える |

## やってみよう!

1. **コスト計算**: 自分のエージェントが 1 日 1,000 リクエストを処理する場合、月間の LLM API コストを試算してみましょう。Haiku のみの場合と Sonnet のみの場合を比較すると、モデルルーティングの重要性が実感できます

2. **ヘルスチェック実装**: FastAPI で `/health` と `/ready` の 2 つのエンドポイントを実装してみましょう。`/health` はプロセスの生存確認、`/ready` は LLM API への疎通確認を行います

3. **構造化ログ導入**: 既存のエージェントコードに `structlog` を導入し、`request_id`、`step`、`tool_name`、`latency` を含むログを出力してみましょう。ログを JSON で出力し、`jq` コマンドでフィルタリングする練習もしてみてください

4. **サーキットブレーカー実験**: 本章のサーキットブレーカーを実装し、意図的にエラーを発生させて状態遷移（CLOSED -> OPEN -> HALF_OPEN -> CLOSED）を確認してみましょう
