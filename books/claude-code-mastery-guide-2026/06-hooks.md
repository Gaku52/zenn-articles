---
title: "hooks -- ライフサイクル自動化"
---

# hooks -- ライフサイクル自動化

Claude Codeの開発フローを次のレベルに引き上げる強力な機能、それが**hooks**（フック）です。この章では、ライフサイクルイベントで自動実行されるアクションの仕組みを詳しく解説します。

## hooksとは

hooksは、Claude Codeのライフサイクルイベント（セッション開始、ツール実行前後、タスク完了など）で自動的に実行されるアクションです。`settings.json`の`"hooks"`セクションで設定することで、以下のようなワークフローを自動化できます。

- ファイル編集後に自動的にlintやフォーマットを実行
- 危険なBashコマンドを実行前にブロック
- タスク完了時に通知をSlackに送信
- セッション開始時に環境変数を検証

hooksの最大の特徴は、**Claude自身の動作を条件分岐させられる**点にあります。単なる「実行後の通知」ではなく、「実行前に止める」「実行内容を変更する」といった介入が可能です。

### 設定場所

hooksは`settings.json`の`"hooks"`セクションで設定します。

```json
{
  "hooks": {
    "SessionStart": [
      {
        "type": "command",
        "command": "echo 'セッション開始！'"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/lint-check.sh"
          }
        ]
      }
    ]
  }
}
```

## フックタイプ

hooksには4つのタイプがあり、それぞれ異なる実行方法を提供します。

### 1. Command（シェルコマンド）

最もシンプルで汎用的なタイプです。任意のシェルコマンドを実行できます。

```json
{
  "type": "command",
  "command": "npm run lint"
}
```

**利用シーン**：
- lintやフォーマッターの実行
- テストの自動実行
- ファイル操作やログ記録
- 環境変数の検証

**終了コードの扱い**：
- `0`: 成功（処理を続行）
- `2`: ブロック（操作を中断）
- その他: 非ブロッキングエラー（警告を表示して続行）

### 2. HTTP（POSTリクエスト）

外部サービスにHTTP POSTリクエストを送信します。

```json
{
  "type": "http",
  "url": "https://hooks.slack.com/services/YOUR/WEBHOOK/URL",
  "body": {
    "text": "Claude Codeでタスクが完了しました"
  }
}
```

**利用シーン**：
- Slack/Discord/Teams通知
- 外部ログサービスへの送信
- CI/CDパイプラインのトリガー
- 分析データの収集

**レスポンスの扱い**：
- HTTPステータス200番台: 成功
- それ以外: エラー（非ブロッキング）

### 3. Prompt（LLM判断）

Claude自身にプロンプトを送信し、LLMの判断を仰ぎます。

```json
{
  "type": "prompt",
  "prompt": "このBashコマンドは安全ですか？: $ARGUMENTS"
}
```

**利用シーン**：
- コマンドの安全性判定
- コミットメッセージの品質チェック
- コードの妥当性検証
- 自然言語ベースのルール適用

**LLMの判断**：
- Claudeが`continue: true`を返せば続行
- `continue: false`を返せばブロック
- `reason`フィールドで理由を説明

### 4. Agent（サブエージェント）

独立したサブエージェントを起動し、複雑な判断や処理を任せます。

```json
{
  "type": "agent",
  "prompt": "このPRレビューコメントは建設的ですか？",
  "project": "code-review-agent"
}
```

**利用シーン**：
- 複雑なコードレビュー
- 多段階の検証ロジック
- コンテキストを必要とする判断
- 外部ツールとの連携

**Agentとの違い**：
- Promptは単発の質問
- Agentは独立したセッションで複数ツールを使える

## 主要イベント（17種類）

Claude Codeは17種類のライフサイクルイベントを提供しています。それぞれのイベントで異なるタイミングとブロック可否が設定されています。

| イベント | タイミング | ブロック可 | 主な用途 |
|---------|-----------|-----------|---------|
| **SessionStart** | セッション開始/再開 | No | 環境検証、初期化 |
| **UserPromptSubmit** | プロンプト送信前 | Yes | ユーザー入力の検証 |
| **PreToolUse** | ツール実行前 | Yes | 危険操作の防止 |
| **PermissionRequest** | 許可ダイアログ表示前 | Yes | 自動許可/拒否 |
| **PostToolUse** | ツール成功後 | Yes | lint、テスト |
| **PostToolUseFailure** | ツール失敗後 | No | エラーログ記録 |
| **Stop** | メインエージェント終了 | Yes | 終了時の検証 |
| **TaskCompleted** | タスク完了 | Yes | 通知、レポート |
| **SubagentStart** | サブエージェント起動 | No | サブエージェント監視 |
| **SubagentStop** | サブエージェント終了 | Yes | 結果の検証 |
| **SessionEnd** | セッション終了 | No | クリーンアップ |
| **ConfigChange** | 設定変更 | Yes | 設定の妥当性検証 |
| **InstructionsLoaded** | CLAUDE.md読み込み | No | 命令の検証 |
| **SkillStart** | スキル実行開始 | No | スキル監視 |
| **SkillStop** | スキル実行終了 | Yes | スキル結果検証 |
| **GitCommit** | git commit実行前 | Yes | コミット内容検証 |
| **GitPush** | git push実行前 | Yes | プッシュ防止 |

### イベントの選び方

**ブロックしたい場合**：
- `PreToolUse`: ツール実行を止める
- `UserPromptSubmit`: ユーザー入力を検証
- `GitCommit`: コミット前に検証

**通知したい場合**：
- `PostToolUse`: 成功時の通知
- `TaskCompleted`: タスク完了通知
- `SessionEnd`: セッション終了ログ

**初期化したい場合**：
- `SessionStart`: 環境変数チェック
- `InstructionsLoaded`: CLAUDE.md検証

## 実践例1: ファイル編集後に自動lint

ファイルを編集するたびに自動的にlintを実行する設定です。

### settings.json

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/lint-check.sh"
          }
        ]
      }
    ]
  }
}
```

### .claude/hooks/lint-check.sh

```bash
#!/bin/bash

# 編集されたファイルパスを取得
FILE_PATH=$(echo "$ARGUMENTS" | jq -r '.file_path // empty')

if [ -z "$FILE_PATH" ]; then
  echo "ファイルパスが取得できませんでした"
  exit 0
fi

# JavaScriptファイルの場合のみlint
if [[ "$FILE_PATH" == *.js ]] || [[ "$FILE_PATH" == *.ts ]]; then
  echo "Linting $FILE_PATH..."
  npx eslint "$FILE_PATH" --fix

  if [ $? -ne 0 ]; then
    echo "Lint failed for $FILE_PATH"
    exit 1  # 警告表示して続行
  fi

  echo "Lint passed!"
fi

exit 0
```

**ポイント**：
- `matcher`: 正規表現で対象ツールを絞る
- `$ARGUMENTS`: ツールに渡された引数（JSON形式）
- `exit 0`: 成功、`exit 1`: 警告、`exit 2`: ブロック

## 実践例2: 危険なコマンドをブロック

`rm -rf`や`sudo`などの危険なコマンドを実行前にブロックする設定です。

### settings.json

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/security-check.sh"
          }
        ]
      }
    ]
  }
}
```

### .claude/hooks/security-check.sh

```bash
#!/bin/bash

# 実行予定のコマンドを取得
COMMAND=$(echo "$ARGUMENTS" | jq -r '.command // empty')

# 危険なパターンのリスト
DANGEROUS_PATTERNS=(
  "rm -rf"
  "sudo rm"
  "> /dev/sda"
  "dd if="
  "mkfs"
  ":(){ :|:& };:"  # fork bomb
)

# パターンマッチング
for pattern in "${DANGEROUS_PATTERNS[@]}"; do
  if [[ "$COMMAND" == *"$pattern"* ]]; then
    echo "🚨 危険なコマンドが検出されました: $pattern"
    echo "実行をブロックしました。"

    # JSON形式で結果を返す
    cat <<EOF
{
  "continue": false,
  "decision": "block",
  "reason": "危険なパターン '$pattern' が検出されました",
  "systemMessage": "セキュリティポリシーにより実行がブロックされました。"
}
EOF
    exit 2  # ブロック
  fi
done

# 安全なコマンド
echo "✅ コマンドは安全です"
cat <<EOF
{
  "continue": true,
  "decision": "approve"
}
EOF
exit 0
```

**ポイント**：
- `PreToolUse`: 実行**前**にチェック
- `exit 2`: 操作をブロック
- JSON出力: より詳細な情報を返せる

## 実践例3: LLMベース検証

Claude自身に「このコマンドは安全か？」と判断させる設定です。

### settings.json

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "以下のBashコマンドを分析してください。安全であれば continue: true、危険であれば continue: false を返してください。理由も説明してください。\n\nコマンド: $ARGUMENTS"
          }
        ]
      }
    ]
  }
}
```

**出力例**：

Claudeが以下のようなJSON形式で判断を返します。

```json
{
  "continue": false,
  "reason": "このコマンドは 'rm -rf /' を含んでおり、システム全体を削除する危険性があります。",
  "systemMessage": "危険なコマンドとして検出されました。"
}
```

**利点**：
- パターンマッチングでは検出できない微妙なケースに対応
- 自然言語でルールを記述できる
- コンテキストを考慮した判断

**欠点**：
- LLM呼び出しのコストと時間
- 100%の精度は保証されない

## 実践例4: タスク完了時にSlack通知

タスクが完了したらSlackに通知する設定です。

### settings.json

```json
{
  "hooks": {
    "TaskCompleted": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "https://hooks.slack.com/services/YOUR/WEBHOOK/URL",
            "body": {
              "text": "✅ Claude Codeでタスクが完了しました",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*タスク完了*\nセッションID: $SESSION_ID\nプロジェクト: $CLAUDE_PROJECT_DIR"
                  }
                }
              ]
            }
          }
        ]
      }
    ]
  }
}
```

**ポイント**：
- Slack Incoming Webhookを事前に設定
- `$SESSION_ID`などの変数を埋め込める
- ブロッキングしないため失敗しても処理は続行

## 入出力仕様

### 共通入力（環境変数）

すべてのhookスクリプトには以下の環境変数が渡されます。

| 変数名 | 説明 | 例 |
|-------|------|---|
| `SESSION_ID` | セッションID | `abc123` |
| `TRANSCRIPT_PATH` | 会話ログのパス | `/path/to/transcript.json` |
| `CWD` | 現在の作業ディレクトリ | `/Users/you/project` |
| `PERMISSION_MODE` | 許可モード | `auto`, `manual`, `ask` |
| `ARGUMENTS` | ツールの引数（JSON） | `{"file_path": "/path/to/file.js"}` |
| `TOOL_NAME` | ツール名 | `Write`, `Bash`, `Read` |
| `CLAUDE_PROJECT_DIR` | プロジェクトディレクトリ | `/Users/you/project` |
| `CLAUDE_ENV_FILE` | 環境変数ファイル | `.claude/.env` |

### 終了コード

hookスクリプトの終了コードで動作を制御します。

| 終了コード | 意味 | 動作 |
|----------|------|------|
| `0` | 成功 | 処理を続行 |
| `2` | ブロック | 操作を中断（ブロック可能イベントのみ） |
| その他 | エラー | 警告を表示して続行 |

### JSON出力（オプション）

より詳細な制御が必要な場合、標準出力にJSON形式で結果を返せます。

```json
{
  "continue": true,
  "suppressOutput": false,
  "systemMessage": "Lint check passed!",
  "decision": "approve",
  "reason": "コードスタイルに問題はありませんでした"
}
```

| フィールド | 型 | 説明 |
|----------|---|------|
| `continue` | boolean | `false`でブロック |
| `suppressOutput` | boolean | `true`でhookの出力を非表示 |
| `systemMessage` | string | ユーザーに表示するメッセージ |
| `decision` | string | `approve`, `block`, `warn` |
| `reason` | string | 判断理由 |

## 特殊変数

hookスクリプト内では、以下の特殊変数を使用できます。

### $ARGUMENTS

ツールに渡された引数がJSON形式で格納されます。

```bash
# file_pathを取得
FILE_PATH=$(echo "$ARGUMENTS" | jq -r '.file_path')

# commandを取得
COMMAND=$(echo "$ARGUMENTS" | jq -r '.command')
```

### $CLAUDE_PROJECT_DIR

Claudeプロジェクトのルートディレクトリ（`.claude/`がある場所）。

```bash
# プロジェクトルートからの相対パスで読み込み
source "$CLAUDE_PROJECT_DIR/.claude/hooks/common.sh"
```

### $CLAUDE_ENV_FILE

プロジェクト固有の環境変数ファイル（`.claude/.env`）。

```bash
# 環境変数を読み込み
if [ -f "$CLAUDE_ENV_FILE" ]; then
  source "$CLAUDE_ENV_FILE"
fi
```

### $TRANSCRIPT_PATH

会話ログのJSONファイルパス。過去のやり取りを参照できます。

```bash
# 最後のメッセージを取得
LAST_MESSAGE=$(jq -r '.messages[-1].content' "$TRANSCRIPT_PATH")
```

## 高度な活用例

### 1. コミットメッセージの品質チェック

```json
{
  "hooks": {
    "GitCommit": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "以下のコミットメッセージを評価してください。Conventional Commits形式に従っているか、説明は十分か確認し、問題があれば continue: false を返してください。\n\nメッセージ: $ARGUMENTS"
          }
        ]
      }
    ]
  }
}
```

### 2. ファイル保存時にテスト自動実行

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "npm test -- --findRelatedTests $(echo $ARGUMENTS | jq -r '.file_path')"
          }
        ]
      }
    ]
  }
}
```

### 3. セッション開始時に環境変数検証

```bash
#!/bin/bash
# .claude/hooks/validate-env.sh

REQUIRED_VARS=("DATABASE_URL" "API_KEY" "NODE_ENV")

for var in "${REQUIRED_VARS[@]}"; do
  if [ -z "${!var}" ]; then
    echo "❌ 必須環境変数 $var が設定されていません"
    exit 1
  fi
done

echo "✅ 環境変数の検証に成功しました"
exit 0
```

```json
{
  "hooks": {
    "SessionStart": [
      {
        "type": "command",
        "command": ".claude/hooks/validate-env.sh"
      }
    ]
  }
}
```

### 4. 複数hookの組み合わせ

1つのイベントに複数のhookを設定できます。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/lint.sh"
          },
          {
            "type": "command",
            "command": ".claude/hooks/format.sh"
          },
          {
            "type": "http",
            "url": "https://your-analytics.com/api/file-edited",
            "body": {
              "file": "$ARGUMENTS"
            }
          }
        ]
      }
    ]
  }
}
```

実行順序は配列の順番通りです。いずれかのhookが`exit 2`を返すとそこで処理が中断されます。

## トラブルシューティング

### hookが実行されない

**チェックポイント**：
- `settings.json`の構文エラーがないか確認
- `matcher`の正規表現が正しいか確認
- スクリプトファイルに実行権限があるか（`chmod +x`）
- スクリプトのシェバン（`#!/bin/bash`）が正しいか

### ブロックが効かない

**原因**：
- ブロック不可のイベントで`exit 2`を返している
- 上記の表で「ブロック可」が「No」のイベントではブロックできません

### JSON出力が認識されない

**対処法**：
- JSON形式が正しいか確認（jqでバリデーション）
- 余計な`echo`文が混ざっていないか確認
- 最後の出力がJSONのみになっているか確認

### hook実行が遅い

**対処法**：
- Prompt/Agentタイプは遅いため、Commandタイプを優先
- 重い処理はバックグラウンド実行（`&`）を検討
- 必要最小限のhookのみ有効化

## まとめ

| 項目 | 内容 |
|-----|------|
| **hooksとは** | ライフサイクルイベントで自動実行されるアクション |
| **設定場所** | `settings.json`の`"hooks"`セクション |
| **フックタイプ** | Command（シェル）、HTTP（POST）、Prompt（LLM）、Agent（サブエージェント） |
| **主要イベント** | 17種類（SessionStart, PreToolUse, PostToolUse, TaskCompleted等） |
| **ブロック機能** | `exit 2`で操作を中断（ブロック可能イベントのみ） |
| **入力** | 環境変数（SESSION_ID, ARGUMENTS, CWD等） |
| **出力** | 終了コード（0/2/その他）またはJSON形式 |
| **活用例** | 自動lint、危険コマンドブロック、Slack通知、環境変数検証 |
| **トラブル** | 実行権限、JSON形式、matcher正規表現を確認 |

## やってみよう！

hooksの理解を深めるため、以下の練習課題に挑戦してみましょう。

### 課題1: ファイル保存カウンター（初級）

ファイルを保存するたびにカウントし、10回ごとに「よく頑張ってます！」と表示するhookを作成してください。

**ヒント**：
- `PostToolUse`イベントを使用
- `matcher: "Write|Edit"`
- カウントは`.claude/hooks/counter.txt`に保存

### 課題2: 危険なgit操作の警告（中級）

`git push --force`や`git reset --hard`などの危険な操作を実行前に警告し、ユーザーに確認を求めるhookを作成してください。

**ヒント**：
- `PreToolUse`イベントとBashツールに対するmatcher
- `exit 2`でブロック
- `systemMessage`で警告文を表示

### 課題3: AI品質チェック（上級）

コミットメッセージがConventional Commits形式に従っているかをLLMで判定し、不適切な場合はブロックするhookを作成してください。

**ヒント**：
- `GitCommit`イベントを使用
- `type: "prompt"`でLLMに判定させる
- プロンプトで「feat:」「fix:」などの形式を説明

## 次の章では

次の章では、**Task Management**について学びます。複雑なタスクを整理し、サブエージェントを活用して大規模な開発を効率化する方法を解説します。hooksと組み合わせることで、タスク完了時の自動通知やタスク開始時の環境検証など、さらに高度な自動化が可能になります。
