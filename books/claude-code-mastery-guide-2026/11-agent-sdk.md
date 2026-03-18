---
title: "Claude Agent SDK"
---

# Claude Agent SDK

Claude Codeはインタラクティブなコマンドラインツールとして非常に強力ですが、さらに一歩進んで、この機能をプログラムから利用できたらどうでしょうか。

**Claude Agent SDK**は、まさにその目的のために設計されたSDKです。Claude Codeを動かすのと同じツール・エージェントループ・コンテキスト管理を、PythonやTypeScriptのコードから直接利用できます。

この章では、Claude Agent SDKの概要から実践的な使い方、さらには2026年時点での最新の統合状況まで、包括的に解説します。

## Agent SDKとは

**Claude Agent SDK**は、Claude Codeのコア機能をプログラマブルに利用するための公式SDKです。

旧称は「Claude Code SDK」でしたが、現在は**Claude Agent SDK**に改名されています。これは、単なるコード生成ツールではなく、自律的に動作する「エージェント」としての性格を強調するための変更です。

SDKには2つの実装があります。

- **Python版**: `anthropics/claude-agent-sdk-python`
- **TypeScript版**: `@anthropic-ai/claude-agent-sdk`

どちらも同等の機能を提供しており、プロジェクトの技術スタックに応じて選択できます。

### なぜSDKが必要なのか

Claude CodeのCLIは対話的な作業に最適ですが、以下のようなシナリオではプログラム的なアクセスが必要になります。

- **CI/CDパイプライン**: プルリクエストごとに自動的にコードレビューを実行
- **定期的なメンテナンス**: 夜間バッチでテストカバレッジを改善
- **カスタムツール**: 社内向けの開発支援ツールにClaude Codeの機能を組み込む
- **チャットボット**: SlackやDiscord経由でコード修正を依頼できる仕組み

これらのユースケースでは、人間が介在しない「非インタラクティブな実行」が求められます。Claude Agent SDKは、まさにこの用途のために設計されています。

## SDKの主要機能

Claude Agent SDKは、Claude CodeのCLIと同じ機能セットを提供します。

### 組み込みツールセット

SDKには、以下のツールがあらかじめ組み込まれています。

- **ファイル読み取り**: Read、Glob、Grep
- **ファイル編集**: Edit、Write
- **コマンド実行**: Bash
- **タスク管理**: Task（Agent Teams）
- **Web関連**: WebSearch、WebFetch

これらのツールは、SDKを利用するだけで自動的に使えるようになります。ツール実装を自分で書く必要はありません。

### エージェントループ

SDKは、Claude CodeのCLIと同じエージェントループを実行します。

1. プロンプトを受け取る
2. Claudeがツールを呼び出す
3. ツール結果をClaudeに返す
4. Claudeが次のアクションを決定
5. 完了するまで繰り返す（最大ターン数まで）

このループは完全に自動化されており、開発者はプロンプトを投げるだけで、Claudeが自律的にタスクを完了します。

### Agent Teams機能

SDKでは、Taskツールを使って複数のエージェントを並列実行できます。これは、Claude CodeのCLIで使える「Agent Teams」と同じ機能です。

例えば、「モジュールAとモジュールBに同時にテストを追加する」といったタスクを、2つのサブエージェントに委譲して並列実行できます。

## TypeScript SDKの使い方

TypeScript版のSDKは、Node.js環境で動作します。

### インストール

```bash
npm install @anthropic-ai/claude-agent-sdk
```

### 基本的な使い方

以下は、TypeScriptエラーを修正する最小限の例です。

```typescript
import { claude } from "@anthropic-ai/claude-agent-sdk";

const result = await claude({
  prompt: "Fix the TypeScript errors in src/",
  options: {
    model: "claude-sonnet-4-5",
    maxTurns: 10,
  }
});

console.log(result.output);
```

この例では、`claude()`関数に以下を渡しています。

- **prompt**: Claudeに実行させたいタスク（CLIの`-p`オプションと同じ）
- **options.model**: 使用するモデル（省略時は`claude-sonnet-4-5`）
- **options.maxTurns**: エージェントループの最大ターン数（省略時は10）

`result.output`には、Claudeの最終応答が文字列で格納されます。

### オプションのカスタマイズ

`options`には、さまざまなパラメータを指定できます。

```typescript
const result = await claude({
  prompt: "Add unit tests for the auth module",
  options: {
    model: "claude-sonnet-4-5",
    maxTurns: 20,
    workingDirectory: "/path/to/project",
    addFile: ["src/auth.ts"],
    addDir: ["tests/"],
    outputFormat: "json",
  }
});
```

- **workingDirectory**: 作業ディレクトリ（省略時はカレントディレクトリ）
- **addFile**: コンテキストに追加するファイル（CLIの`--add-file`）
- **addDir**: コンテキストに追加するディレクトリ（CLIの`--add-dir`）
- **outputFormat**: 出力形式（`text`または`json`）

### JSON出力の利用

`outputFormat: "json"`を指定すると、結果が構造化されたJSON形式で返ります。

```typescript
const result = await claude({
  prompt: "List all TODO comments in src/",
  options: {
    outputFormat: "json",
  }
});

const data = JSON.parse(result.output);
console.log(data.todos); // { file: "src/app.ts", line: 42, text: "TODO: ..." }
```

これは、Claudeの応答を後続のプログラムで処理する際に便利です。

## Python SDKの使い方

Python版のSDKは、TypeScript版と同等の機能を提供します。

### インストール

```bash
pip install claude-agent-sdk
```

### 基本的な使い方

以下は、認証モジュールにユニットテストを追加する例です。

```python
from claude_agent_sdk import claude

result = claude(
    prompt="Add unit tests for the auth module",
    model="claude-sonnet-4-5",
    max_turns=10,
)

print(result.output)
```

TypeScript版と同様に、`claude()`関数にプロンプトとオプションを渡すだけです。

### オプションのカスタマイズ

Pythonでも、さまざまなオプションを指定できます。

```python
result = claude(
    prompt="Fix all linting errors",
    model="claude-sonnet-4-5",
    max_turns=20,
    working_directory="/path/to/project",
    add_file=["src/main.py"],
    add_dir=["tests/"],
    output_format="json",
)
```

パラメータ名はスネークケース（`max_turns`、`working_directory`）になっている点が、TypeScript版との違いです。

### ファイルコンテキストの追加

特定のファイルやディレクトリをコンテキストに追加することで、Claudeがより正確な作業を行えるようになります。

```python
result = claude(
    prompt="Refactor the database layer",
    add_dir=["src/db/", "src/models/"],
    add_file=["config/database.yaml"],
)
```

この例では、`src/db/`と`src/models/`ディレクトリ、および`config/database.yaml`ファイルがコンテキストに追加されます。

## CLIからの非インタラクティブ実行

SDKを使わなくても、Claude CodeのCLI自体に非インタラクティブモードが用意されています。

### `-p`オプション

`-p`オプションを使うと、プロンプトを渡してそのまま実行できます。

```bash
claude -p "Fix the TypeScript errors in src/"
```

これは、シェルスクリプトやCI/CDパイプラインから実行する際に便利です。

### JSON出力

`--output-format json`を指定すると、結果がJSON形式で出力されます。

```bash
claude -p "List all TODO comments" --output-format json > todos.json
```

これにより、`jq`などのツールで結果をパースできます。

```bash
claude -p "List all TODO comments" --output-format json | jq '.todos'
```

### パイプ入力

標準入力からファイルを渡すこともできます。

```bash
cat file.py | claude -p "Review this code and suggest improvements"
```

この方法は、特定のファイルだけをレビューしたい場合に便利です。

### 複数ファイルのスコープ指定

`--add-dir`オプションで、複数のディレクトリをコンテキストに追加できます。

```bash
claude --add-dir ../lib --add-dir ../common -p "Fix import errors"
```

これは、モノレポや複数プロジェクトにまたがるタスクで役立ちます。

## 実践的なユースケース

ここからは、Claude Agent SDKを実際のプロジェクトで活用する具体例を見ていきましょう。

### CI/CDでのコードレビュー自動化

プルリクエストごとに、Claudeがコードレビューを実行します。

**GitHub Actions（TypeScript）**

```yaml
name: Claude Code Review
on: pull_request

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm install @anthropic-ai/claude-agent-sdk
      - name: Review PR
        run: |
          node -e "
          import { claude } from '@anthropic-ai/claude-agent-sdk';
          const result = await claude({
            prompt: 'Review this PR for code quality, security, and best practices',
            options: { outputFormat: 'json' }
          });
          console.log(result.output);
          "
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

この例では、PRの差分をClaudeがレビューし、結果をJSON形式で出力します。

### テスト生成の自動化

毎日のCronジョブで、テストカバレッジが低いファイルに対して自動的にテストを追加します。

**Python（Cron）**

```python
#!/usr/bin/env python3
from claude_agent_sdk import claude
import subprocess

# カバレッジが低いファイルを特定
coverage_output = subprocess.check_output(["pytest", "--cov", "--cov-report=json"])
low_coverage_files = parse_coverage_report(coverage_output)  # 独自の解析関数

for file in low_coverage_files:
    result = claude(
        prompt=f"Add unit tests for {file} to improve coverage",
        add_file=[file],
    )
    print(f"Tests added for {file}: {result.output}")
```

この仕組みにより、テストカバレッジが自動的に改善されていきます。

### 定期的なコード品質チェック

週次で、コードベース全体をスキャンして潜在的な問題を検出します。

**TypeScript（週次バッチ）**

```typescript
import { claude } from "@anthropic-ai/claude-agent-sdk";

async function weeklyAudit() {
  const tasks = [
    "Find and fix unused imports in src/",
    "Identify functions with high cyclomatic complexity",
    "Check for security vulnerabilities in dependencies",
  ];

  for (const task of tasks) {
    const result = await claude({
      prompt: task,
      options: { maxTurns: 15, outputFormat: "json" },
    });
    console.log(`[${task}]`, result.output);
  }
}

weeklyAudit();
```

これをCronやGitHub Actionsのスケジュールで実行すれば、コード品質が継続的に改善されます。

### チャットボットへの組み込み

Slack経由でコード修正を依頼できるボットを作成します。

**TypeScript（Slack Bot）**

```typescript
import { App } from "@slack/bolt";
import { claude } from "@anthropic-ai/claude-agent-sdk";

const app = new App({
  token: process.env.SLACK_BOT_TOKEN,
  signingSecret: process.env.SLACK_SIGNING_SECRET,
});

app.message(/^fix:/, async ({ message, say }) => {
  const prompt = message.text.replace(/^fix:\s*/, "");

  const result = await claude({
    prompt,
    options: { maxTurns: 10 },
  });

  await say(`Done! ${result.output}`);
});

await app.start(3000);
```

このボットにSlackで「`fix: Add error handling to src/api.ts`」と話しかけると、Claudeがコードを修正してくれます。

## 2026年の統合状況

2026年3月時点で、Claude Agent SDKは多くの開発環境やツールと統合されています。

### Apple Xcode 26.3でのネイティブ統合

Xcodeの最新版（26.3）では、Claude Codeがネイティブに統合されています。

- エディタ右クリックから「Claude: Fix Errors」を選択
- Xcodeのビルドログから直接Claudeを呼び出す
- SwiftUIプレビュー内でリアルタイムに修正を提案

この統合により、SwiftやObjective-Cの開発体験が大幅に向上すると期待されています。

### Microsoft Agent Framework統合

Microsoftの**Agent Framework**（Copilot Studioの後継）では、Claude Codeをカスタムエージェントとして登録できます。

```yaml
# agent.yaml
name: ClaudeCodeAgent
runtime: anthropic-claude-agent-sdk
model: claude-sonnet-4-5
tools:
  - file-edit
  - bash
  - web-search
```

この設定ファイルを使うことで、Azure DevOps PipelinesやPower Automateから直接Claudeを呼び出せます。

### マルチターン会話の改善

2026年版のSDKでは、マルチターン会話の精度が大幅に改善されました。

- **会話履歴の保持**: 前回の実行結果を次の実行に引き継げる
- **並列ツール呼び出し**: 複数のファイル編集を同時に実行してターン数を削減
- **プロンプトキャッシング**: 大規模コードベースでも高速化

これにより、複雑なタスクでも少ないターン数で完了するようになっています。

### MCPサーバー統合

Claude Agent SDKは、MCPサーバーを自動的に検出して利用します。

例えば、以下のようなMCPサーバーがインストールされている場合、SDKから自動的に使えます。

- **@modelcontextprotocol/server-postgres**: PostgreSQLクエリ実行
- **@modelcontextprotocol/server-slack**: Slackメッセージ送信
- **@modelcontextprotocol/server-github**: GitHub API操作

```typescript
const result = await claude({
  prompt: "Query the users table and post a summary to #dev channel",
  // MCPサーバーは自動検出される
});
```

この例では、Claudeが自動的に以下のツールを使います。

1. PostgreSQLサーバーに接続してクエリ実行
2. 結果をフォーマット
3. Slackの#devチャンネルに投稿

開発者は、MCPサーバーの存在を意識する必要がありません。

## まとめ

この章では、Claude Agent SDKの全体像を解説しました。

| 項目 | 内容 |
|------|------|
| **SDKとは** | Claude Codeのコア機能をプログラムから利用できるSDK（Python/TypeScript） |
| **主要機能** | 組み込みツール、エージェントループ、Agent Teams |
| **TypeScript版** | `@anthropic-ai/claude-agent-sdk`、Node.js環境で動作 |
| **Python版** | `claude-agent-sdk`、Pythonスクリプトやバッチジョブで利用可能 |
| **CLIモード** | `-p`オプションで非インタラクティブ実行、CI/CDに最適 |
| **ユースケース** | コードレビュー自動化、テスト生成、品質チェック、チャットボット |
| **2026年の進化** | Xcode統合、Microsoft Agent Framework、マルチターン改善、MCP統合 |

Claude Agent SDKは、Claude Codeの能力を自動化やカスタムツールに応用するための強力な基盤です。インタラクティブなCLIとプログラマブルなSDKを組み合わせることで、開発ワークフロー全体を最適化できます。

## やってみよう！

Claude Agent SDKの理解を深めるために、以下の課題に挑戦してみましょう。

### 課題1: 簡単なスクリプトを書く

TypeScriptまたはPythonで、以下を実行するスクリプトを作成してください。

1. `src/`ディレクトリ内の全ファイルを読み取る
2. Claudeに「すべてのTODOコメントをリスト化して」と指示
3. 結果をJSON形式で`todos.json`に保存

### 課題2: CI/CDパイプラインに組み込む

GitHub ActionsまたはGitLab CIで、以下のワークフローを作成してください。

1. プルリクエスト作成時にトリガー
2. Claudeがコードレビューを実行
3. 結果をPRコメントとして投稿

### 課題3: チャットボットを作る

SlackまたはDiscordボットで、以下の機能を実装してください。

1. メッセージで「`/fix [タスク]`」を受け取る
2. Claude Agent SDKでタスクを実行
3. 結果をチャットに返信

これらの課題を通じて、SDKの実践的な使い方が身につきます。

---

次の章では、Claude Codeのプロンプト設計テクニックを深掘りします。Claudeにより正確な指示を出すための「プロンプトエンジニアリング」の手法を学び、複雑なタスクでも高い成功率を実現する方法を解説します。
