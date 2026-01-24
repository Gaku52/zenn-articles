---
title: "バグトラッキングツール：Jira、GitHub Issues、Linear"
---

# バグトラッキングツール：Jira、GitHub Issues、Linear

## バグトラッキングツールの選択

バグトラッキングツールは、バグの発見から修正まで のライフサイクルを管理します。チームの規模、ワークフロー、他ツールとの統合性を考慮して選択します。

## Jira

エンタープライズ向けの最も包括的なプロジェクト管理・バグトラッキングツールです。

### 基本設定

```yaml
# Jiraプロジェクト設定例
project:
  key: PROJ
  name: ECサイトプロジェクト
  type: Software

issue_types:
  - Bug
  - Story
  - Task
  - Epic

workflows:
  bug_workflow:
    - Open
    - In Progress
    - In Review
    - Resolved
    - Closed
    - Reopened
```

### カスタムフィールドの活用

```markdown
## バグチケットのカスタムフィールド

### 深刻度（Severity）
- Critical: システムダウン、データ損失
- High: 主要機能が使えない
- Medium: 一部機能に制限
- Low: 軽微な問題

### 優先度（Priority）
- P0: 即時対応（4時間以内）
- P1: 緊急（24時間以内）
- P2: 高（3日以内）
- P3: 中（1週間以内）
- P4: 低（時間があるとき）

### 検出環境
- Development
- Staging
- Production

### ブラウザ/OS
- Chrome 120 / Windows 11
- Safari 17 / macOS Sonoma
```

### 自動化ルール

```groovy
// Jira Automation Rule例
// Critical バグが作成されたらSlack通知

trigger: Issue Created
conditions:
  - Issue Type = Bug
  - Severity = Critical

actions:
  - Send Slack message to #critical-bugs
    message: "🚨 Criticalバグが報告されました: {{issue.key}} - {{issue.summary}}"
  - Assign to QA Lead
  - Set Priority to P0
```

### JQLクエリ例

```sql
-- 未解決のCritical/Highバグ
project = PROJ AND type = Bug
AND severity in (Critical, High)
AND status != Closed
ORDER BY created DESC

-- 今週作成されたバグ
project = PROJ AND type = Bug
AND created >= startOfWeek()

-- 再オープンされたバグ
project = PROJ AND type = Bug
AND status = Reopened

-- SLA違反しているバグ
project = PROJ AND type = Bug
AND "Due Date" < now()
AND status != Closed
```

### Jiraダッシュボード

```markdown
## QAダッシュボード構成

### ガジェット1: バグ深刻度分布（円グラフ）
JQL: project = PROJ AND type = Bug AND status != Closed

### ガジェット2: バグステータス推移（線グラフ）
- Open
- In Progress
- Resolved

### ガジェット3: SLA違反バグ（テーブル）
JQL: "Due Date" < now() AND status != Closed

### ガジェット4: 担当者別バグ数（棒グラフ）
JQL: project = PROJ AND type = Bug AND assignee is not EMPTY
```

## GitHub Issues

開発ワークフローと統合された軽量なバグトラッキングです。

### Issue テンプレート

```markdown
---
name: バグレポート
about: バグを報告する
labels: bug
assignees: ''
---

## バグの概要
簡潔に説明してください

## 再現手順
1.
2.
3.

## 期待される動作


## 実際の動作


## スクリーンショット
もしあれば

## 環境
- OS: [例: macOS 14.0]
- ブラウザ: [例: Chrome 120]
- バージョン: [例: v2.0.0]

## 追加情報
その他の関連情報
```

### ラベルの体系化

```yaml
# ラベル設計
severity:
  - severity/critical (赤)
  - severity/high (オレンジ)
  - severity/medium (黄色)
  - severity/low (緑)

priority:
  - priority/P0 (赤)
  - priority/P1 (オレンジ)
  - priority/P2 (黄色)
  - priority/P3 (緑)
  - priority/P4 (グレー)

type:
  - type/bug
  - type/regression
  - type/security

status:
  - status/confirmed
  - status/duplicate
  - status/wont-fix

area:
  - area/frontend
  - area/backend
  - area/database
  - area/api
```

### GitHub Actions連携

```yaml
# .github/workflows/bug-triage.yml
name: Bug Triage

on:
  issues:
    types: [opened, labeled]

jobs:
  triage:
    runs-on: ubuntu-latest
    steps:
      - name: Label critical bugs
        if: contains(github.event.issue.labels.*.name, 'severity/critical')
        uses: actions/github-script@v6
        with:
          script: |
            await github.rest.issues.update({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              labels: [...context.payload.issue.labels.map(l => l.name), 'priority/P0']
            });

      - name: Notify Slack for critical bugs
        if: contains(github.event.issue.labels.*.name, 'severity/critical')
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "🚨 Critical Bug: ${{ github.event.issue.title }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Critical Bug Reported*\n<${{ github.event.issue.html_url }}|${{ github.event.issue.title }}>"
                  }
                }
              ]
            }
```

### プロジェクトボード活用

```markdown
## バグ管理プロジェクトボード

### 列（Column）構成
1. **Backlog**: 新規報告、未トリアージ
2. **Triage**: 優先順位付け中
3. **Todo**: 対応予定
4. **In Progress**: 対応中
5. **In Review**: レビュー中
6. **Done**: 完了

### 自動化ルール
- Issueがopenされたら → Backlogに追加
- Issueに`priority/P0`ラベル → Todoに移動
- PRがマージされたら → Doneに移動
```

## Linear

モダンで高速なイシュートラッキングツールです。

### プロジェクト構成

```typescript
// Linear設定（TypeScript型定義風）
interface LinearConfig {
  team: {
    key: 'QA';
    name: 'Quality Assurance';
  };

  workflow: {
    states: [
      { name: 'Backlog', type: 'backlog' },
      { name: 'Todo', type: 'unstarted' },
      { name: 'In Progress', type: 'started' },
      { name: 'In Review', type: 'started' },
      { name: 'Done', type: 'completed' },
      { name: 'Canceled', type: 'canceled' }
    ];
  };

  labels: [
    { name: 'bug', color: '#e5484d' },
    { name: 'critical', color: '#dc2626' },
    { name: 'regression', color: '#f59e0b' }
  ];

  priorities: [
    { priority: 0, label: 'No priority' },
    { priority: 1, label: 'Urgent' },
    { priority: 2, label: 'High' },
    { priority: 3, label: 'Medium' },
    { priority: 4, label: 'Low' }
  ];
}
```

### Linear API活用

```typescript
// Linearからバグデータ取得
import { LinearClient } from '@linear/sdk';

const linear = new LinearClient({ apiKey: process.env.LINEAR_API_KEY });

async function getCriticalBugs() {
  const issues = await linear.issues({
    filter: {
      team: { key: { eq: 'QA' } },
      labels: { name: { in: ['bug', 'critical'] } },
      state: { type: { neq: 'completed' } }
    }
  });

  return issues.nodes.map(issue => ({
    id: issue.identifier,
    title: issue.title,
    priority: issue.priority,
    assignee: issue.assignee?.name,
    createdAt: issue.createdAt,
    url: issue.url
  }));
}
```

### Slack統合

```markdown
## Linear Slack連携

### 通知設定
1. Linearで`/linear`コマンド使用
2. チャンネルに通知を設定
   - 新しいバグ作成時
   - Criticalバグの状態変更時
   - SLA期限切れ時

### コマンド例
- `/linear create Bug in payment flow`
- `/linear list priority:1`
- `/linear QA-123` （バグ詳細表示）
```

## ツール選択のガイドライン

### チーム規模別推奨

| チームサイズ | 推奨ツール | 理由 |
|------------|----------|------|
| 1-5人 | GitHub Issues | シンプル、無料、開発ワークフローと統合 |
| 5-20人 | Linear | 高速、モダンUI、適度な機能 |
| 20人以上 | Jira | 包括的機能、カスタマイズ性、エンタープライズ対応 |

### 機能比較

| 機能 | Jira | GitHub Issues | Linear |
|------|------|---------------|--------|
| カスタマイズ性 | ◎ | △ | ○ |
| 開発ワークフロー統合 | ○ | ◎ | ◎ |
| レポート機能 | ◎ | △ | ○ |
| 価格 | 高 | 無料 | 中 |
| 学習コスト | 高 | 低 | 中 |
| 動作速度 | △ | ○ | ◎ |

## バグトラッキングのベストプラクティス

### 1. 明確なバグレポート

```markdown
## 良いバグレポートの例

**タイトル**: [決済] クレジットカード決済完了後にエラー画面が表示される

**環境**:
- ブラウザ: Chrome 120
- OS: Windows 11
- URL: https://example.com/checkout

**再現手順**:
1. 商品をカートに追加
2. 決済ページに進む
3. クレジットカード情報を入力
   - カード番号: 4242 4242 4242 4242（テストカード）
   - 有効期限: 12/25
   - CVV: 123
4. "支払う"ボタンをクリック

**期待される動作**:
決済完了画面が表示され、確認メールが送信される

**実際の動作**:
500エラー画面が表示される

**スクリーンショット**:
[添付画像]

**追加情報**:
- 決済は完了している（Stripe側で確認）
- エラーログ: "TypeError: Cannot read property 'email' of undefined"
```

### 2. 適切なトリアージ

```typescript
interface BugTriage {
  severity: 'critical' | 'high' | 'medium' | 'low';
  priority: 'P0' | 'P1' | 'P2' | 'P3' | 'P4';
  assignee: string;
  labels: string[];
}

function triageBug(bug: Bug): BugTriage {
  // 深刻度と影響範囲から優先度を決定
  if (bug.severity === 'critical' && bug.affectsProduction) {
    return {
      severity: 'critical',
      priority: 'P0',
      assignee: 'on-call-engineer',
      labels: ['critical', 'production', 'hotfix']
    };
  }

  // 通常のトリアージロジック
  return determinePriority(bug);
}
```

### 3. SLA管理

```markdown
## バグ対応SLA

| 深刻度 | 初回応答 | 解決目標 |
|--------|---------|---------|
| Critical | 1時間 | 4時間 |
| High | 4時間 | 24時間 |
| Medium | 1日 | 3日 |
| Low | 3日 | 2週間 |
```

## まとめ

バグトラッキングツールは、バグ管理の中核を担います。Jira、GitHub Issues、Linearそれぞれに特徴があり、チームの規模やワークフローに応じて選択することが推奨されます。

明確なバグレポート、適切なトリアージ、SLA管理により、効率的なバグ解決プロセスが期待されます。ツールの自動化機能を活用し、手作業を最小限に抑えることも重要です。

次章では、QA自動化ツールとフレームワークについて解説します。
