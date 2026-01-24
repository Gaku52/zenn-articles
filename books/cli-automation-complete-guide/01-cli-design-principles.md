# CLI設計原則

優れたCLIツールを作るには、適切な設計原則の理解が不可欠です。本章では、使いやすく保守性の高いCLIツールを設計するための基本原則を学びます。

## UNIX哲学

CLIツール設計の基礎となるUNIX哲学を理解しましょう。

### Do One Thing Well

1つの機能に集中することで、ツールの品質と保守性が向上します。

```bash
# 推奨されないパターン: 多機能すぎる
mytool --create --name myapp --template react --install --deploy

# 推奨されるパターン: 機能ごとに分割
mytool create myapp --template react
mytool install
mytool deploy
```

### Composable（組み合わせ可能）

パイプで連携できる設計にすることで、他のツールとの組み合わせが容易になります。

```bash
# JSON出力をjqで加工
mytool list --format json | jq '.[] | select(.active == true)'

# CSV出力をgrepでフィルタ
mytool search "query" | grep -i "keyword"

# ファイルへの出力
mytool export --format csv > data.csv
```

### Silent Success（成功時は静か）

成功時は不必要な出力を避け、エラー時のみメッセージを表示します。

```bash
# 推奨されないパターン
$ mytool update
Updating...
Processing item 1...
Processing item 2...
Update complete!
Success!

# 推奨されるパターン
$ mytool update
# 何も出力しない（成功）

# 詳細が必要なら --verbose オプション
$ mytool update --verbose
Updating 42 items...
✓ Complete
```

## CLIのユーザビリティ原則

### 発見可能性（Discoverability）

ユーザーが機能を簡単に見つけられるようにします。

```bash
# ヘルプの提供
mytool --help
mytool create --help

# サブコマンド一覧
$ mytool
Available commands:
  create    Create a new project
  list      List all projects
  delete    Delete a project

# typoへのサジェスト
$ mytool crate myapp
Unknown command: crate
Did you mean: create?
```

### 一貫性（Consistency）

コマンド構造とオプションを統一します。

```bash
# 推奨されないパターン
mytool create --name myapp
mytool remove myapp        # --name がない
mytool list-all --verbose  # ハイフンの有無が不統一

# 推奨されるパターン
mytool create myapp --template react
mytool delete myapp
mytool list --all --verbose
```

### 安全性（Safety）

破壊的操作には確認を求めます。

```bash
# 確認プロンプト
$ mytool delete myapp
Are you sure you want to delete 'myapp'? (y/N)

# --force で確認をスキップ
$ mytool delete myapp --force

# Dry Run オプション
$ mytool deploy --dry-run
Would deploy:
  - app.js
  - index.html
  - styles.css
```

## コマンド設計

### コマンド構造

基本的なコマンド構造を理解しましょう。

```
<tool> <command> <subcommand> [arguments] [options]
```

実例:

```bash
# シンプルなコマンド
git commit -m "message"

# サブコマンド構造
docker container ls
npm run build

# 複数階層
kubectl get pods --namespace production
aws s3 cp file.txt s3://bucket/
```

### コマンド命名規則

**動詞ベース（CRUD操作）**:

```bash
mytool create <name>    # Create
mytool list             # Read
mytool update <name>    # Update
mytool delete <name>    # Delete

# その他の動詞
mytool start <service>
mytool stop <service>
mytool deploy <target>
```

**名詞ベース（リソース管理）**:

```bash
# Dockerスタイル
mytool container ls
mytool container rm <id>
mytool image pull <name>

# kubectlスタイル
mytool get pods
mytool describe pod <name>
```

## 引数とオプション

### 位置引数（Positional Arguments）

必須の引数を位置で指定します。

```bash
# 1つの引数
mytool create <name>

# 複数の引数
mytool copy <source> <destination>

# 可変長引数
mytool delete <file1> [file2] [file3...]
```

### オプション（Options/Flags）

振る舞いを変更するためのオプションを提供します。

```bash
# 短縮形と長形式の両方を提供
mytool create myapp -t react        # 短縮形
mytool create myapp --template react # 長形式

# 一般的な短縮形
-h, --help       # ヘルプ
-v, --version    # バージョン
-V, --verbose    # 詳細出力
-q, --quiet      # 静かに実行
-f, --force      # 強制実行
```

## 出力設計

### 標準出力と標準エラー出力

適切な出力先を使い分けます。

```bash
# stdout: データ出力
mytool list --format json | jq '.[] | .name'

# stderr: メッセージ・エラー
$ mytool create myapp 2> errors.log
Creating myapp...  # stderr
✓ Complete        # stderr
```

### 出力形式

人間向けとマシン向けの両方をサポートします。

```bash
# 人間向け（デフォルト）
$ mytool list
Projects:
  myapp      (React, TypeScript)
  dashboard  (Vue, JavaScript)

# JSON形式
$ mytool list --format json
[
  {"name": "myapp", "framework": "React"},
  {"name": "dashboard", "framework": "Vue"}
]

# CSV形式
$ mytool list --format csv
name,framework,lang
myapp,React,TypeScript
dashboard,Vue,JavaScript
```

### カラー出力

適切なカラーリングで可読性を向上させます。

```bash
# 成功: 緑
✅ Project created successfully

# エラー: 赤
❌ Failed to create project

# 警告: 黄
⚠️  Warning: No .gitignore found

# 情報: 青
ℹ️  Installing dependencies...
```

TTY判定を行い、パイプ時はプレーンテキストにします。

## エラーハンドリング

### エラーメッセージの設計

優れたエラーメッセージには、What（何が起きたか）、Why（なぜ起きたか）、How（どう直すか）が含まれます。

```bash
# 推奨されないパターン
$ mytool deploy
Error

# 推奨されるパターン
$ mytool deploy
❌ Deployment failed

Reason: No build files found in ./dist

Suggestion: Run 'mytool build' before deploying
```

### 終了コード

適切な終了コードを返します。

| コード | 意味 |
|--------|------|
| 0 | 成功 |
| 1 | ユーザーエラー（引数不正、設定ミスなど） |
| 2 | システムエラー（権限、ネットワークなど） |
| 130 | Ctrl+C で中断 |

## 設定管理

### 設定ファイルの優先順位

明確な優先順位を定義します。

```
1. コマンドライン引数（最優先）
2. 環境変数
3. プロジェクト設定ファイル（./mytool.config.js）
4. ユーザー設定ファイル（~/.mytoolrc）
5. デフォルト値（最低優先）
```

### 環境変数

命名規則を統一します。

```bash
# プレフィックス + アンダースコア + 大文字
MYTOOL_API_KEY="xxx"
MYTOOL_PORT="3000"
MYTOOL_VERBOSE="true"

# 使用例
$ MYTOOL_PORT=4000 mytool start
```

## まとめ

本章では、CLI設計の基本原則を学びました。

**重要ポイント**:
- UNIX哲学に従う（Do One Thing Well、Composable、Silent Success）
- ユーザビリティを重視（発見可能性、一貫性、安全性）
- 適切な出力設計（stdout/stderr、カラー、フォーマット）
- 分かりやすいエラーメッセージ
- 明確な設定管理

これらの原則を守ることで、使いやすく保守性の高いCLIツールを開発できます。次章では、具体的なアーキテクチャパターンを学びます。
