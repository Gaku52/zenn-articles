---
title: "CI/CDモニタリングと可観測性 - パイプラインの健全性を保つ"
---

# Chapter 10 - CI/CDモニタリングと可観測性

## CI/CDモニタリングの重要性

CI/CDパイプラインは、開発チームの生産性を支える重要なインフラです。適切なモニタリングにより、問題を早期発見し、継続的な改善を実現できます。

### 想定される効果: モニタリング導入効果

あるSaaS企業(開発者30名)での想定シナリオ:

**最適化前:**
- パイプライン障害の検知: 平均3時間後
- 障害原因の特定: 平均2時間
- ビルド失敗の原因不明: 月15件
- 開発者の生産性損失: 週10時間

**最適化後:**
- ✅ 障害検知: 3時間 → 3分 (-99%)
- ✅ 原因特定時間: 2時間 → 15分 (-87%)
- ✅ 原因不明の失敗: 15件 → 2件 (-87%)
- ✅ 生産性損失: 10時間 → 2時間 (-80%)
- ✅ パイプライン安定性: 92% → 99.5%

## メトリクス収集

### 1. ビルドメトリクス

```yaml
# .github/workflows/ci-with-metrics.yml
name: CI with Metrics

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Record start time
        id: start
        run: echo "timestamp=$(date +%s)" >> $GITHUB_OUTPUT

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        id: install
        run: |
          START=$(date +%s)
          npm ci
          END=$(date +%s)
          echo "duration=$((END - START))" >> $GITHUB_OUTPUT

      - name: Build
        id: build
        run: |
          START=$(date +%s)
          npm run build
          END=$(date +%s)
          echo "duration=$((END - START))" >> $GITHUB_OUTPUT

      - name: Test
        id: test
        run: |
          START=$(date +%s)
          npm test -- --coverage
          END=$(date +%s)
          echo "duration=$((END - START))" >> $GITHUB_OUTPUT

          # カバレッジ率を抽出
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          echo "coverage=$COVERAGE" >> $GITHUB_OUTPUT

      - name: Send metrics to monitoring
        if: always()
        env:
          METRICS_ENDPOINT: ${{ secrets.METRICS_ENDPOINT }}
        run: |
          TOTAL_TIME=$(($(date +%s) - ${{ steps.start.outputs.timestamp }}))

          curl -X POST $METRICS_ENDPOINT \
            -H "Content-Type: application/json" \
            -d '{
              "workflow": "${{ github.workflow }}",
              "repository": "${{ github.repository }}",
              "branch": "${{ github.ref_name }}",
              "commit": "${{ github.sha }}",
              "status": "${{ job.status }}",
              "metrics": {
                "total_time": '$TOTAL_TIME',
                "install_time": ${{ steps.install.outputs.duration }},
                "build_time": ${{ steps.build.outputs.duration }},
                "test_time": ${{ steps.test.outputs.duration }},
                "coverage": ${{ steps.test.outputs.coverage || 0 }}
              },
              "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"
            }'
```

**想定効果:**
- ビルド時間のトレンド可視化: ボトルネック特定が容易に
- 異常値検知: 通常の2倍以上かかった場合にアラート

### 2. ワークフローサマリー

```yaml
- name: Generate workflow summary
  if: always()
  run: |
    cat >> $GITHUB_STEP_SUMMARY << 'EOF'
    ## ⏱️ ビルド時間分析

    | ステップ | 時間 | 状態 |
    |---------|------|------|
    | Dependencies | ${{ steps.install.outputs.duration }}s | ✅ |
    | Build | ${{ steps.build.outputs.duration }}s | ✅ |
    | Test | ${{ steps.test.outputs.duration }}s | ✅ |
    | **Total** | **${TOTAL_TIME}s** | **✅** |

    ### 📊 カバレッジ
    - Lines: ${{ steps.test.outputs.coverage }}%
    - Target: 80%

    ### 🔗 リンク
    - [テスト結果](https://example.com/tests/${{ github.run_id }})
    - [カバレッジレポート](https://example.com/coverage/${{ github.run_id }})
    EOF
```

## ログ集約と分析

### CloudWatch Logsへの送信

```yaml
# .github/workflows/ci-with-logs.yml
name: CI with CloudWatch Logs

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and capture logs
        id: build
        run: |
          npm run build 2>&1 | tee build.log
        continue-on-error: true

      - name: Send logs to CloudWatch
        if: always()
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: ap-northeast-1
        run: |
          # CloudWatch Logs にログを送信
          aws logs create-log-stream \
            --log-group-name /github-actions/ci \
            --log-stream-name ${{ github.run_id }} || true

          # ログを送信
          TIMESTAMP=$(date +%s000)
          LOG_MESSAGE=$(cat build.log | jq -Rs .)

          aws logs put-log-events \
            --log-group-name /github-actions/ci \
            --log-stream-name ${{ github.run_id }} \
            --log-events timestamp=$TIMESTAMP,message=$LOG_MESSAGE

      - name: Fail if build failed
        if: steps.build.outcome == 'failure'
        run: exit 1
```

### Datadog連携

```yaml
- name: Send metrics to Datadog
  if: always()
  env:
    DD_API_KEY: ${{ secrets.DD_API_KEY }}
  run: |
    curl -X POST "https://api.datadoghq.com/api/v1/series" \
      -H "Content-Type: application/json" \
      -H "DD-API-KEY: $DD_API_KEY" \
      -d @- << EOF
    {
      "series": [
        {
          "metric": "github.actions.build.duration",
          "points": [[${{ steps.start.outputs.timestamp }}, $TOTAL_TIME]],
          "type": "gauge",
          "tags": [
            "repository:${{ github.repository }}",
            "branch:${{ github.ref_name }}",
            "workflow:${{ github.workflow }}",
            "status:${{ job.status }}"
          ]
        },
        {
          "metric": "github.actions.test.coverage",
          "points": [[${{ steps.start.outputs.timestamp }}, ${{ steps.test.outputs.coverage }}]],
          "type": "gauge",
          "tags": [
            "repository:${{ github.repository }}",
            "branch:${{ github.ref_name }}"
          ]
        }
      ]
    }
    EOF
```

## アラート設定

### Slack通知の高度な活用

```yaml
# .github/workflows/ci-with-alerts.yml
name: CI with Smart Alerts

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build
        id: build
        run: npm run build

      - name: Test
        id: test
        run: npm test

      - name: Smart Slack notification
        if: always()
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
        run: |
          # 状態に応じた色と絵文字
          if [ "${{ job.status }}" = "success" ]; then
            COLOR="good"
            EMOJI="✅"
            MESSAGE="Build succeeded"
          else
            COLOR="danger"
            EMOJI="❌"
            MESSAGE="Build failed"
          fi

          # mainブランチの失敗は@channelでメンション
          MENTION=""
          if [ "${{ github.ref_name }}" = "main" ] && [ "${{ job.status }}" = "failure" ]; then
            MENTION="<!channel> "
          fi

          curl -X POST $SLACK_WEBHOOK \
            -H 'Content-Type: application/json' \
            -d '{
              "text": "'"$MENTION$EMOJI $MESSAGE"'",
              "attachments": [{
                "color": "'"$COLOR"'",
                "fields": [
                  {
                    "title": "Repository",
                    "value": "${{ github.repository }}",
                    "short": true
                  },
                  {
                    "title": "Branch",
                    "value": "${{ github.ref_name }}",
                    "short": true
                  },
                  {
                    "title": "Author",
                    "value": "${{ github.actor }}",
                    "short": true
                  },
                  {
                    "title": "Commit",
                    "value": "<https://github.com/${{ github.repository }}/commit/${{ github.sha }}|'"${GITHUB_SHA:0:7}"'>",
                    "short": true
                  },
                  {
                    "title": "Build Time",
                    "value": "'"$TOTAL_TIME"'s",
                    "short": true
                  }
                ],
                "actions": [
                  {
                    "type": "button",
                    "text": "View Logs",
                    "url": "https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}"
                  }
                ]
              }]
            }'
```

**想定効果:**
- 重要なアラートの見逃し: 削減
- mainブランチ失敗の検知時間: 平均3分以内

### PagerDuty連携(重要な障害)

```yaml
- name: Trigger PagerDuty incident
  if: failure() && github.ref == 'refs/heads/main'
  env:
    PAGERDUTY_KEY: ${{ secrets.PAGERDUTY_INTEGRATION_KEY }}
  run: |
    curl -X POST https://events.pagerduty.com/v2/enqueue \
      -H 'Content-Type: application/json' \
      -d '{
        "routing_key": "'"$PAGERDUTY_KEY"'",
        "event_action": "trigger",
        "payload": {
          "summary": "Production build failed: ${{ github.repository }}",
          "severity": "critical",
          "source": "GitHub Actions",
          "custom_details": {
            "repository": "${{ github.repository }}",
            "branch": "${{ github.ref_name }}",
            "commit": "${{ github.sha }}",
            "author": "${{ github.actor }}",
            "run_url": "https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          }
        }
      }'
```

## ダッシュボード構築

### Grafanaダッシュボード例

```json
{
  "dashboard": {
    "title": "GitHub Actions CI/CD Metrics",
    "panels": [
      {
        "title": "Build Duration Trend",
        "type": "graph",
        "targets": [
          {
            "expr": "avg(github_actions_build_duration_seconds) by (repository)"
          }
        ]
      },
      {
        "title": "Build Success Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(github_actions_builds{status='success'}) / sum(github_actions_builds) * 100"
          }
        ]
      },
      {
        "title": "Test Coverage",
        "type": "graph",
        "targets": [
          {
            "expr": "github_actions_test_coverage_percent"
          }
        ]
      },
      {
        "title": "Failed Builds by Repository",
        "type": "table",
        "targets": [
          {
            "expr": "sum by (repository) (github_actions_builds{status='failure'})"
          }
        ]
      }
    ]
  }
}
```

### GitHub Actions ネイティブメトリクス

```yaml
# .github/workflows/publish-metrics.yml
name: Publish Metrics

on:
  schedule:
    - cron: '0 * * * *'  # 毎時

jobs:
  metrics:
    runs-on: ubuntu-latest
    steps:
      - name: Fetch workflow runs
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          # 過去24時間のワークフロー実行を取得
          gh api repos/${{ github.repository }}/actions/runs \
            --jq '.workflow_runs[] | select(.created_at > (now - 86400 | todate)) | {
              id: .id,
              name: .name,
              status: .status,
              conclusion: .conclusion,
              duration: (.updated_at | fromdateiso8601) - (.created_at | fromdateiso8601)
            }' > metrics.json

      - name: Calculate metrics
        run: |
          # 成功率計算
          TOTAL=$(jq '. | length' metrics.json)
          SUCCESS=$(jq '[.[] | select(.conclusion == "success")] | length' metrics.json)
          SUCCESS_RATE=$(echo "scale=2; $SUCCESS / $TOTAL * 100" | bc)

          # 平均実行時間
          AVG_DURATION=$(jq '[.[] | .duration] | add / length' metrics.json)

          echo "Success Rate: $SUCCESS_RATE%"
          echo "Average Duration: ${AVG_DURATION}s"

          # メトリクスを保存
          cat > summary.json << EOF
          {
            "success_rate": $SUCCESS_RATE,
            "average_duration": $AVG_DURATION,
            "total_runs": $TOTAL,
            "successful_runs": $SUCCESS,
            "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
          }
          EOF

      - name: Publish to monitoring system
        run: |
          curl -X POST ${{ secrets.METRICS_ENDPOINT }} \
            -H 'Content-Type: application/json' \
            -d @summary.json
```

## トラブルシューティング

### 問題1: メトリクスが記録されない

**症状:**
```
メトリクス送信ステップは成功しているが、ダッシュボードにデータが表示されない
```

**対処法:**

```yaml
# デバッグモードで実行
- name: Send metrics (debug)
  run: |
    set -x  # デバッグモード有効化

    # 送信するデータを確認
    cat << EOF | tee metrics.json
    {
      "workflow": "${{ github.workflow }}",
      "metrics": {
        "duration": $TOTAL_TIME
      }
    }
    EOF

    # レスポンスを確認
    curl -v -X POST $METRICS_ENDPOINT \
      -H "Content-Type: application/json" \
      -d @metrics.json
```

### 問題2: アラートが多すぎる

**対処法: スマートアラート設定**

```yaml
- name: Smart alerting
  if: failure()
  run: |
    # 連続失敗回数を取得
    CONSECUTIVE_FAILURES=$(gh api \
      repos/${{ github.repository }}/actions/runs \
      --jq '[.workflow_runs[] |
             select(.name == "${{ github.workflow }}") |
             select(.conclusion == "failure")] |
             length')

    # 3回連続失敗した場合のみアラート
    if [ $CONSECUTIVE_FAILURES -ge 3 ]; then
      echo "Triggering alert: $CONSECUTIVE_FAILURES consecutive failures"
      # Slack通知を送信
    else
      echo "Skipping alert: Only $CONSECUTIVE_FAILURES failures"
    fi
```

## まとめ

この章では、CI/CDパイプラインのモニタリングを学びました:

✅ **メトリクス収集**: ビルド時間、成功率、カバレッジの計測
✅ **ログ集約**: CloudWatch、Datadogへの集約
✅ **アラート設定**: Slack、PagerDutyでの通知
✅ **ダッシュボード**: Grafanaでの可視化
✅ **トレンド分析**: 継続的な改善のための分析

### 重要な想定される効果まとめ

| 項目 | 効果 |
|------|------|
| 障害検知時間 | 3時間→3分 (-99%) |
| 原因特定時間 | 2時間→15分 (-87%) |
| 原因不明の失敗 | 15件→2件 (-87%) |
| 開発者の生産性損失 | 10時間→2時間 (-80%) |
| パイプライン安定性 | 92%→99.5% |

### 次のステップ

**Chapter 11 - トラブルシューティングガイド**では、よくある問題とその解決方法を学びます。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
