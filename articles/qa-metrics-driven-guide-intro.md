---
title: "「このくらいテストすれば大丈夫」を卒業する - メトリクス駆動QAの実践"
emoji: "📊"
type: "tech"
topics: ["qa", "品質保証", "テスト", "メトリクス", "grafana"]
published: true
---

# 品質を「感覚」で判断していませんか？

「このくらいテストすれば大丈夫でしょ」

リリース前、その判断の根拠は何ですか？

- ベテランの勘？
- 過去の経験？
- なんとなくの感覚？

それらは大切です。でも、**チーム全員で共有できる基準**にはなりません。

新人エンジニアに「どのくらいテストすればいいですか？」と聞かれて、明確に答えられますか？

「テストカバレッジ80%以上」
「クリティカルバグゼロ」
「レスポンスタイム500ms以下」

こうした**数値化された基準**があれば、誰でも品質を判断できます。

これが「**メトリクス駆動QA**」です。

## なぜ今、メトリクス駆動QAが必要なのか

### 1. リモートワークでの品質管理の難しさ

対面なら「ちょっと見せて」で済んだレビューが、リモートでは難しい。

**課題**：
- 口頭での品質確認が困難
- 認識のズレに気づきにくい
- 暗黙知の共有ができない

**解決策**：
明確な数値基準があれば、場所に関係なく品質を判断できます。

### 2. アジャイル開発での短いサイクル

2週間スプリントで、毎回リリース判定。

**課題**：
- 限られた時間で品質を保証
- 感覚だけでは判断が間に合わない
- リリース遅延のリスク

**解決策**：
メトリクスのダッシュボードを見れば、即座に判断可能。

### 3. チームメンバーの入れ替わり

ベテランが抜けたら、品質が落ちる。

**課題**：
- 個人の経験に依存
- ナレッジが属人化
- 引き継ぎが困難

**解決策**：
メトリクスとして基準を残せば、誰でも同じレベルで判断できます。

## メトリクス駆動QAの3つの柱

### 1. テストメトリクス

**測定すべき指標**：

```
✅ テストカバレッジ
- ライン カバレッジ：80%以上
- ブランチカバレッジ：70%以上
- 関数カバレッジ：90%以上

✅ テスト実行結果
- 成功率：98%以上
- 実行時間：10分以内
- Flaky率：5%以下

✅ テストケース数
- 単体テスト：実装ファイルの1.5倍
- 統合テスト：主要フロー全カバー
- E2Eテスト：クリティカルパス全カバー
```

**可視化の例（Jest + Istanbul）**：

```json
// package.json
{
  "jest": {
    "coverageThreshold": {
      "global": {
        "branches": 70,
        "functions": 90,
        "lines": 80,
        "statements": 80
      }
    }
  }
}
```

閾値を下回ると、CIが失敗します。

### 2. バグメトリクス

**測定すべき指標**：

```
✅ バグ検出数
- 総バグ数
- 重要度別（Critical/High/Medium/Low）
- 検出フェーズ別（開発中/QA/本番）

✅ バグ解決速度
- 平均修正時間（MTTR: Mean Time To Repair）
- Critical: 24時間以内
- High: 3日以内
- Medium: 1週間以内

✅ バグ再発率
- 同一箇所の再発：5%以下
- リグレッション率：3%以下
```

**Grafanaでの可視化例**：

```yaml
# Grafana Dashboard設定
panels:
  - title: "バグ推移（週次）"
    metrics:
      - 新規バグ数
      - 修正完了数
      - 未解決バグ数

  - title: "重要度別バグ分布"
    type: pie-chart
    metrics:
      - Critical
      - High
      - Medium
      - Low

  - title: "MTTR（平均修正時間）"
    target: 24時間
    alert: 48時間超過時
```

### 3. パフォーマンスメトリクス

**測定すべき指標**：

```
✅ レスポンスタイム
- API応答時間：500ms以下（P95）
- ページ読み込み：2秒以内（P95）
- Time to Interactive：3秒以内

✅ エラー率
- 4xx エラー：1%以下
- 5xx エラー：0.1%以下
- タイムアウト：0.5%以下

✅ リソース使用率
- CPU使用率：70%以下
- メモリ使用率：80%以下
- ディスクI/O：安定
```

## 実践例：リリース判定基準の作り方

### ステップ1：必須条件を定義

```markdown
## リリース必須条件（Go基準）

### テスト
- [ ] 全自動テストがパス（成功率100%）
- [ ] テストカバレッジ80%以上
- [ ] E2Eテストが全てグリーン

### バグ
- [ ] Criticalバグがゼロ
- [ ] Highバグがゼロ
- [ ] 既知のMediumバグに回避策あり

### パフォーマンス
- [ ] P95レスポンスタイム500ms以下
- [ ] エラー率1%以下
- [ ] 負荷テストクリア

### ドキュメント
- [ ] リリースノート作成済み
- [ ] ロールバック手順確認済み
```

### ステップ2：警告条件を定義

```markdown
## リリース警告条件（注意が必要）

### テスト
- ⚠️ カバレッジが前回より5%以上低下
- ⚠️ Flaky testが3件以上
- ⚠️ テスト実行時間が20%以上増加

### バグ
- ⚠️ Mediumバグが10件以上
- ⚠️ 直近1週間でバグ急増（前週比2倍以上）
- ⚠️ 特定機能にバグが集中

### パフォーマンス
- ⚠️ レスポンスタイムが前回より20%悪化
- ⚠️ メモリ使用量が前回より30%増加
```

これらの条件に引っかかった場合、リリース延期を検討します。

### ステップ3：自動判定の実装

```typescript
// release-checker.ts
interface ReleaseMetrics {
  testCoverage: number;
  criticalBugs: number;
  highBugs: number;
  p95ResponseTime: number;
  errorRate: number;
}

function canRelease(metrics: ReleaseMetrics): {
  canRelease: boolean;
  reasons: string[];
} {
  const reasons: string[] = [];

  // 必須条件チェック
  if (metrics.testCoverage < 80) {
    reasons.push(`テストカバレッジ不足: ${metrics.testCoverage}% < 80%`);
  }

  if (metrics.criticalBugs > 0) {
    reasons.push(`Criticalバグあり: ${metrics.criticalBugs}件`);
  }

  if (metrics.highBugs > 0) {
    reasons.push(`Highバグあり: ${metrics.highBugs}件`);
  }

  if (metrics.p95ResponseTime > 500) {
    reasons.push(`レスポンス遅延: ${metrics.p95ResponseTime}ms > 500ms`);
  }

  if (metrics.errorRate > 0.01) {
    reasons.push(`エラー率高い: ${metrics.errorRate * 100}% > 1%`);
  }

  return {
    canRelease: reasons.length === 0,
    reasons,
  };
}

// CI/CDで実行
const metrics = await collectMetrics();
const result = canRelease(metrics);

if (!result.canRelease) {
  console.error('❌ リリース不可:');
  result.reasons.forEach(r => console.error(`  - ${r}`));
  process.exit(1);
}

console.log('✅ リリース可能');
```

## Grafana + Prometheusでの実装例

### Prometheusでメトリクス収集

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'api-server'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['localhost:3000']
    scrape_interval: 15s
```

```typescript
// アプリケーション側（Node.js + prom-client）
import { Counter, Histogram, Registry } from 'prom-client';

const register = new Registry();

// リクエスト数カウンター
const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'status'],
  registers: [register],
});

// レスポンスタイムヒストグラム
const httpRequestDuration = new Histogram({
  name: 'http_request_duration_ms',
  help: 'HTTP request duration in ms',
  labelNames: ['method', 'route'],
  buckets: [50, 100, 200, 500, 1000, 2000],
  registers: [register],
});

// ミドルウェアで計測
app.use((req, res, next) => {
  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;
    httpRequestsTotal.inc({ method: req.method, status: res.statusCode });
    httpRequestDuration.observe({ method: req.method, route: req.route?.path }, duration);
  });

  next();
});

// メトリクスエンドポイント
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

### Grafanaでダッシュボード構築

```json
{
  "dashboard": {
    "title": "QA Metrics Dashboard",
    "panels": [
      {
        "title": "API Response Time (P95)",
        "targets": [{
          "expr": "histogram_quantile(0.95, http_request_duration_ms_bucket)"
        }],
        "alert": {
          "conditions": [{
            "evaluator": { "params": [500], "type": "gt" },
            "query": { "params": ["A", "5m", "now"] }
          }]
        }
      },
      {
        "title": "Error Rate",
        "targets": [{
          "expr": "rate(http_requests_total{status=~\"5..\"}[5m]) / rate(http_requests_total[5m])"
        }]
      }
    ]
  }
}
```

## よくある質問

**Q: メトリクスを見る時間がない場合は？**
A: 最初は3つだけ。「カバレッジ」「Criticalバグ数」「エラー率」。これだけでも大きく変わります。

**Q: 既存プロジェクトに導入する場合、どこから？**
A: まず現状を測定。現在のメトリクスをベースラインとして、少しずつ改善目標を設定します。

**Q: 数値だけで判断して、見落としはない？**
A: メトリクスは「最低ライン」。クリアした上で、探索的テストやユーザビリティテストを実施します。

## こんな人におすすめ

- リリース判断を明確な基準で行いたいQAエンジニア
- チームに品質基準を導入したいリーダー
- 品質を数値で可視化したい開発者
- GrafanaやPrometheusの実践的な使い方を学びたい方
- 属人化した品質管理から脱却したい組織

## さらに深く学ぶには

本記事で紹介したのは、メトリクス駆動QAの入り口です。

**もっと詳しく学びたい方には、こちらの本がおすすめです**：

https://zenn.dev/gaku/books/qa-complete-guide-2026

本書では、以下の内容を21章・13万文字で詳しく解説しています：

### 📚 本書の内容

**Part 1: QAプロセス基礎（全4章）**
- QAの基本原則と組織設計
- 品質メトリクスの選び方
- QAチームの立ち上げ方

**Part 2: テスト戦略・計画（全5章）**
- テストピラミッドの実践
- テスト計画テンプレート（そのまま使える）
- テストケース設計手法
- 探索的テスト・リグレッションテスト

**Part 3: メトリクス・KPI（全5章）**
- QAメトリクス完全ガイド
- バグメトリクスの分析方法
- カバレッジメトリクスの正しい見方
- **KPIダッシュボード構築（Grafana実例付き）**
- **Prometheus統合の実装**

**Part 4: リリース管理（全4章）**
- リリース判定基準の作り方
- リリースチェックリスト（テンプレート付き）
- Go/NoGo判断の実践
- ポストリリース監視

**Part 5: ツール・自動化（全3章）**
- バグトラッキングツールの選定と運用
- QA自動化ツールの実践

### 💰 価格：1,500円

全21章、129,768文字のボリュームで、実務で即使えるテンプレートとダッシュボード設定例付き。

## まとめ

品質を「感覚」ではなく「数値」で判断する。

それだけで：
- リリース判断が明確になる
- チーム全員が同じ基準で動ける
- 品質が属人化しない
- 継続的な改善が可能になる

「このくらいテストすれば大丈夫」

その判断を、誰でも再現できる基準に変えませんか？

今日から、3つのメトリクスを測定することから始めてみてください：
1. テストカバレッジ
2. Criticalバグ数
3. エラー率

それが、メトリクス駆動QAへの第一歩です。
