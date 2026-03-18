---
title: "MCP基礎 -- 外部ツール連携"
---

# MCP基礎 -- 外部ツール連携

これまでの章では、Claude Codeの基本的な使い方からプロンプトエンジニアリング、プロジェクトの初期化、Skillsの活用まで学んできました。この章では、Claude Codeの機能をさらに拡張する**MCP（Model Context Protocol）**について学びます。

MCPを使うと、GitHub、Notion、PostgreSQL、Figmaなど、さまざまな外部サービスとClaude Codeを連携できるようになります。これにより、AIがあなたの開発環境やビジネスツールと直接やり取りできるようになり、作業効率が飛躍的に向上します。

## MCPとは（Model Context Protocol）

**MCP（Model Context Protocol）**は、AIツールと外部サービスを統合するためのオープンソース標準プロトコルです。Anthropic社が開発し、現在では多くのAIツールや開発者コミュニティで採用されています。

### MCPの特徴

- **標準化されたプロトコル**: AIツールと外部サービスの接続方法を統一
- **豊富なエコシステム**: GitHub、Sentry、Notion、PostgreSQL、Figma、Slackなど、数百のMCPサーバーが存在
- **広範な対応ツール**: Claude Code、Claude Desktop、Cursor、Windsurf等のAIツールが対応
- **3つの主要機能**:
  - **Resources（リソース）**: 外部データへのアクセス（例: GitHubのIssue、Notionのページ）
  - **Prompts（プロンプト）**: 再利用可能なプロンプトテンプレート
  - **Tools（ツール）**: AIが実行できるアクション（例: Issue作成、データベースクエリ）

### MCPの仕組み

MCPは、Claude Codeと外部サービスの間に立つ「サーバー」として動作します。Claude Codeがリクエストを送ると、MCPサーバーがそれを外部サービスのAPIに変換し、結果をClaude Codeに返します。

```
[Claude Code] ←→ [MCPサーバー] ←→ [外部サービス（GitHub/Notion等）]
```

この仕組みにより、開発者は外部サービスのAPI仕様を気にすることなく、自然言語でAIに指示を出すだけで外部連携が実現します。

## MCPサーバーの追加方法

MCPサーバーを追加する方法は、サーバーの種類によって異なります。主に以下の3つのタイプがあります。

### 1. リモートHTTPサーバー（推奨）

最も一般的で推奨される方法です。HTTPプロトコルで通信するリモートサーバーを追加します。

**基本的な追加方法**:

```bash
claude mcp add --transport http <サーバー名> <URL>
```

**例: Notionサーバーを追加**:

```bash
claude mcp add --transport http notion https://mcp.notion.com/mcp
```

**認証ヘッダーを含める場合**:

```bash
claude mcp add --transport http secure-api https://api.example.com/mcp \
  --header "Authorization: Bearer your-token"
```

複数のヘッダーを追加する場合は、`--header`オプションを繰り返し指定します。

```bash
claude mcp add --transport http my-api https://api.example.com/mcp \
  --header "Authorization: Bearer token123" \
  --header "X-Custom-Header: value"
```

### 2. リモートSSEサーバー

SSE（Server-Sent Events）プロトコルを使用するサーバーです。リアルタイム更新が必要なサービスで使われます。

```bash
claude mcp add --transport sse <サーバー名> <URL>
```

**例: Asanaサーバーを追加**:

```bash
claude mcp add --transport sse asana https://mcp.asana.com/sse
```

### 3. ローカルstdioサーバー

ローカルマシンで実行されるプログラムと標準入出力（stdin/stdout）で通信するサーバーです。Node.jsパッケージやPythonスクリプトなどを実行する際に使用します。

```bash
claude mcp add --transport stdio <サーバー名> -- <コマンド>
```

**例: Airtable MCPサーバーを追加**:

```bash
claude mcp add --transport stdio --env AIRTABLE_API_KEY=YOUR_KEY airtable \
  -- npx -y airtable-mcp-server
```

**重要な注意点**:

- `--transport`, `--env`, `--scope`などのオプションは、必ず**サーバー名の前**に指定します
- `--`（ダブルダッシュ）の後にコマンドを記述します
- 環境変数は`--env KEY=VALUE`の形式で指定します

**Pythonスクリプトを実行する例**:

```bash
claude mcp add --transport stdio weather-service \
  -- python3 /path/to/weather_mcp_server.py
```

### オプションの指定順序

MCPサーバーを追加する際、オプションの順序は非常に重要です。基本的な構文は以下の通りです。

```bash
claude mcp add [グローバルオプション] <サーバー名> [サーバー固有の引数]
```

**グローバルオプション（サーバー名の前に指定）**:
- `--transport`: 通信方式（http, sse, stdio）
- `--env`: 環境変数
- `--scope`: スコープ（local, project, user）
- `--header`: HTTPヘッダー（HTTP transportの場合）
- `--client-id`, `--client-secret`: OAuth認証情報

**正しい例**:

```bash
claude mcp add --transport stdio --env API_KEY=abc123 --scope user myserver -- node server.js
```

**間違った例**:

```bash
# ❌ オプションがサーバー名の後ろにある
claude mcp add myserver --transport stdio -- node server.js
```

## MCPサーバー管理

追加したMCPサーバーは、以下のコマンドで管理できます。

### サーバー一覧の表示

```bash
claude mcp list
```

このコマンドを実行すると、追加済みのすべてのMCPサーバーが表示されます。

**出力例**:

```
local scope:
  - github (http)
  - notion (http)

user scope:
  - postgres (stdio)
  - weather-api (http)
```

### 特定サーバーの詳細確認

```bash
claude mcp get <サーバー名>
```

**例**:

```bash
claude mcp get github
```

**出力例**:

```json
{
  "name": "github",
  "transport": "http",
  "url": "https://mcp.github.com/mcp",
  "scope": "local",
  "status": "connected"
}
```

### サーバーの削除

```bash
claude mcp remove <サーバー名>
```

**例**:

```bash
claude mcp remove github
```

削除前に確認メッセージが表示されます。

### Claude Code内での確認

Claude Codeのチャット内で以下のコマンドを実行すると、MCPサーバーの接続状態を確認できます。

```
/mcp
```

このコマンドは、各サーバーの状態（接続済み、認証待ち、エラーなど）を表示します。

## スコープ -- MCPサーバーの適用範囲

MCPサーバーには、3つのスコープ（適用範囲）があります。

### 1. local（デフォルト）

現在のプロジェクトディレクトリでのみ有効です。スコープを指定しない場合、自動的に`local`になります。

```bash
claude mcp add --transport http github https://mcp.github.com/mcp
```

設定は`.claude/mcp.json`に保存されます。このファイルは通常、`.gitignore`に追加してバージョン管理から除外します（APIキーなどの秘密情報が含まれる可能性があるため）。

### 2. project

プロジェクトの`.mcp.json`ファイルに保存され、チーム全体で共有できます。

```bash
claude mcp add --scope project --transport http shared-api https://api.team.com/mcp
```

この設定は`.mcp.json`に保存され、Gitでコミットできます。チームメンバーが同じプロジェクトをクローンすると、自動的に同じMCPサーバーが利用可能になります。

**注意**: APIキーなどの秘密情報は含めず、環境変数で管理することを推奨します。

### 3. user

すべてのプロジェクトで利用可能になります。グローバル設定として`~/.config/claude/mcp.json`に保存されます。

```bash
claude mcp add --scope user --transport stdio postgres \
  -- psql -h localhost -U myuser
```

よく使うツール（例: データベース接続、天気情報API等）は`user`スコープで追加すると便利です。

### スコープの優先順位

同じ名前のMCPサーバーが複数のスコープで定義されている場合、以下の優先順位で適用されます。

```
local > project > user
```

つまり、ローカルスコープの設定が最優先されます。

## OAuth認証

一部のMCPサーバー（GitHub、Notion、Slackなど）は、OAuth認証を必要とします。

### OAuth認証の流れ

1. **MCPサーバーを追加**:

```bash
claude mcp add --transport http github https://mcp.github.com/mcp
```

2. **Claude Code内で認証**:

```
/mcp
```

このコマンドを実行すると、認証が必要なサーバーが表示されます。

3. **認証URLをクリック**:

表示されたURLをブラウザで開き、外部サービスにログインして認証を完了します。

4. **認証完了**:

ブラウザで認証が完了すると、Claude Codeに自動的に戻り、サーバーが利用可能になります。

### 事前設定済みOAuth

一部のサーバーでは、OAuth認証情報を事前に設定できます。

```bash
claude mcp add --transport http \
  --client-id YOUR_CLIENT_ID \
  --client-secret YOUR_CLIENT_SECRET \
  --callback-port 3000 \
  github https://mcp.github.com/mcp
```

これにより、認証フローが自動化され、より迅速にサーバーを利用開始できます。

## JSON設定から追加

複雑な設定を持つMCPサーバーは、JSON形式で直接追加できます。

```bash
claude mcp add-json <サーバー名> '<JSON設定>'
```

**例: HTTPサーバーをJSON設定で追加**:

```bash
claude mcp add-json weather-api '{
  "type": "http",
  "url": "https://api.weather.com/mcp",
  "headers": {
    "Authorization": "Bearer token123",
    "X-API-Version": "2.0"
  }
}'
```

**例: stdioサーバーをJSON設定で追加**:

```bash
claude mcp add-json custom-tool '{
  "type": "stdio",
  "command": "node",
  "args": ["/path/to/server.js"],
  "env": {
    "API_KEY": "your-key",
    "DEBUG": "true"
  }
}'
```

JSON設定を使うと、複数のヘッダーや環境変数を一度に設定できるため、複雑な構成のサーバーに便利です。

## Claude Desktopからインポート

すでにClaude Desktopを使っている場合、そこで設定したMCPサーバーをClaude Codeにインポートできます。

```bash
claude mcp add-from-claude-desktop
```

このコマンドを実行すると、Claude Desktopの設定ファイル（`~/Library/Application Support/Claude/claude_desktop_config.json`）から、すべてのMCPサーバーがインポートされます。

**確認プロンプトの例**:

```
Found 3 MCP servers in Claude Desktop:
  - github (http)
  - notion (http)
  - postgres (stdio)

Import all servers? (y/n):
```

`y`を入力すると、すべてのサーバーがClaude Codeに追加されます。

## MCPリソース参照

MCPサーバーを追加すると、Claude Codeのチャット内で`@`メンションを使ってリソースを参照できるようになります。

### リソースの種類

MCPサーバーは、以下の3種類の機能を提供します。

#### 1. Resources（リソース）

外部データへのアクセスを提供します。

**例**:

```
@github:issue://123
```

これにより、GitHubのIssue #123の内容がClaude Codeに読み込まれます。

```
@notion:page://abc123
```

NotionのページID `abc123`の内容を参照します。

#### 2. Prompts（プロンプト）

再利用可能なプロンプトテンプレートです。

**例**:

```
@github:create-issue
```

GitHub Issueを作成するためのプロンプトテンプレートが呼び出されます。

#### 3. Tools（ツール）

AIが実行できるアクションです。これらはClaude Codeが自動的に検出し、必要に応じて実行します。

**例**:
- `github_create_issue`: GitHubにIssueを作成
- `postgres_execute_query`: PostgreSQLクエリを実行
- `slack_send_message`: Slackメッセージを送信

ツールは明示的に呼び出す必要はなく、Claude Codeが文脈に応じて自動的に使用します。

### リソース参照の実例

**GitHubのPull Requestをレビュー**:

```
@github:pr://456 をレビューして、改善点を教えてください
```

Claude Codeは、PR #456の内容を読み込み、コードレビューを実施します。

**Notionのドキュメントを要約**:

```
@notion:page://project-spec を読んで、主要な機能を箇条書きにしてください
```

Notionのページ内容を解析し、要約を生成します。

## 人気MCPサーバー紹介

ここでは、開発者によく使われる人気のMCPサーバーをいくつか紹介します。

### 1. GitHub MCP

GitHubのIssue、Pull Request、リポジトリ情報にアクセスできます。

**追加方法**:

```bash
claude mcp add --transport http github https://mcp.github.com/mcp
```

**できること**:
- Issue/PRの閲覧、作成、更新
- リポジトリのファイル閲覧
- コミット履歴の確認
- ブランチ管理

**使用例**:

```
GitHub Issue #123を確認して、対応方法を提案してください
```

### 2. Figma MCP

Figmaのデザインファイルと連携し、デザイン仕様を取得できます。

**追加方法**:

```bash
claude mcp add --transport http figma https://mcp.figma.com/mcp
```

**できること**:
- デザインファイルの閲覧
- コンポーネント情報の取得
- カラーパレット、タイポグラフィの抽出
- デザイントークンの生成

**使用例**:

```
@figma:file://abc123 のボタンコンポーネントをReactで実装してください
```

### 3. Playwright MCP

ブラウザ操作を自動化し、Webスクレイピングやテストを実行できます。

**追加方法**:

```bash
claude mcp add --transport stdio playwright -- npx -y @modelcontextprotocol/server-playwright
```

**できること**:
- Webページの自動操作
- スクリーンショットの取得
- フォーム入力、ボタンクリック
- 要素の検証

**使用例**:

```
https://example.com にアクセスして、ログインフォームをテストしてください
```

### 4. Supabase MCP

Supabaseデータベースと連携し、データのクエリや操作ができます。

**追加方法**:

```bash
claude mcp add --transport http \
  --env SUPABASE_URL=https://your-project.supabase.co \
  --env SUPABASE_KEY=your-anon-key \
  supabase https://mcp.supabase.com/mcp
```

**できること**:
- テーブルのクエリ
- データの挿入、更新、削除
- スキーマ情報の取得
- RPC（リモートプロシージャコール）の実行

**使用例**:

```
usersテーブルから、最近登録した10人のユーザーを取得してください
```

### 5. PostgreSQL MCP

PostgreSQLデータベースに直接アクセスし、SQLクエリを実行できます。

**追加方法**:

```bash
claude mcp add --transport stdio \
  --env PGHOST=localhost \
  --env PGUSER=myuser \
  --env PGPASSWORD=mypassword \
  --env PGDATABASE=mydb \
  postgres -- npx -y @modelcontextprotocol/server-postgres
```

**できること**:
- SQLクエリの実行
- テーブルスキーマの確認
- データの分析
- パフォーマンス最適化の提案

**使用例**:

```
ordersテーブルから、今月の売上合計を計算してください
```

### 6. Sentry MCP

Sentryのエラートラッキング情報にアクセスし、バグ解析を支援します。

**追加方法**:

```bash
claude mcp add --transport http \
  --header "Authorization: Bearer YOUR_SENTRY_TOKEN" \
  sentry https://mcp.sentry.io/mcp
```

**できること**:
- エラーログの閲覧
- スタックトレースの分析
- エラー頻度の統計
- 修正提案の生成

**使用例**:

```
直近24時間のエラーを確認して、優先度の高いものを教えてください
```

### 7. Slack MCP

Slackとの連携により、チャンネルへのメッセージ送信や情報取得が可能です。

**追加方法**:

```bash
claude mcp add --transport http slack https://mcp.slack.com/mcp
```

**できること**:
- メッセージの送信
- チャンネル一覧の取得
- 過去のメッセージ検索
- リアクションの追加

**使用例**:

```
#general チャンネルに、今日のリリース完了を報告してください
```

## まとめ

この章では、MCPの基礎について学びました。以下のポイントを押さえておきましょう。

| 項目 | 内容 |
|------|------|
| **MCPとは** | AIツールと外部サービスを統合するオープンソース標準プロトコル |
| **主な通信方式** | HTTP（推奨）、SSE、stdio の3種類 |
| **追加コマンド** | `claude mcp add --transport <type> <name> <url/command>` |
| **管理コマンド** | `list`（一覧）、`get`（詳細）、`remove`（削除） |
| **スコープ** | `local`（プロジェクト）、`project`（共有）、`user`（全体） |
| **OAuth認証** | `/mcp`コマンドで認証フローを開始 |
| **リソース参照** | `@server:resource://id` の形式で参照 |
| **人気サーバー** | GitHub, Figma, Playwright, Supabase, PostgreSQL, Sentry, Slack |

MCPを活用することで、Claude Codeは単なるコード生成ツールから、開発環境全体を統合する強力なアシスタントへと進化します。

## やってみよう！

実際にMCPサーバーを追加して、動作を確認してみましょう。

### 演習1: Playwrightサーバーの追加

1. Playwright MCPサーバーを追加します。

```bash
claude mcp add --transport stdio playwright -- npx -y @modelcontextprotocol/server-playwright
```

2. Claude Codeを起動します。

```bash
claude
```

3. Playwrightを使ってWebページにアクセスしてみます。

```
https://example.com にアクセスして、ページタイトルを教えてください
```

Claude Codeが自動的にPlaywright MCPサーバーを使い、ブラウザを起動してページ情報を取得します。

### 演習2: MCPサーバーの管理

1. 追加したサーバーの一覧を確認します。

```bash
claude mcp list
```

2. Playwrightサーバーの詳細を確認します。

```bash
claude mcp get playwright
```

3. Claude Code内でサーバーの状態を確認します。

```bash
claude
```

チャット内で:

```
/mcp
```

### 演習3: スコープの使い分け

1. プロジェクト固有のサーバーを追加します（localスコープ）。

```bash
claude mcp add --transport http project-api https://api.myproject.com/mcp
```

2. 全プロジェクトで使うサーバーを追加します（userスコープ）。

```bash
claude mcp add --scope user --transport http weather https://api.weather.com/mcp
```

3. 両方のスコープを確認します。

```bash
claude mcp list
```

local と user の両方のスコープにサーバーが表示されることを確認してください。

---

次の章では、MCPをより実践的に活用する方法を学びます。具体的なユースケースを通じて、GitHub連携、データベース操作、ブラウザ自動化など、実務で役立つMCPの活用法を詳しく解説します。
