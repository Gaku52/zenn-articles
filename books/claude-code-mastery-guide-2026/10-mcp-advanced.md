---
title: "MCP実践 -- 自作サーバーと高度な活用"
---

# MCP実践 -- 自作サーバーと高度な活用

これまでの章では、既存のMCPサーバーを設定して利用する方法を学びました。本章では、さらに一歩踏み込んで、自分でMCPサーバーを作成する方法や、エンタープライズ環境での高度な活用法、セキュリティ考慮事項について解説します。

## MCPサーバーの3つの機能

MCPサーバーは、Claudeに対して以下の3つの機能を提供できます。

### Tools（ツール）

ツールは、Claudeが呼び出せる関数です。APIリクエストの実行、データベースクエリ、ファイル操作など、あらゆる処理を実装できます。

```typescript
server.tool("get_weather", { city: z.string() }, async ({ city }) => {
  const weather = await fetchWeather(city);
  return { content: [{ type: "text", text: JSON.stringify(weather) }] };
});
```

### Resources（リソース）

リソースは、Claudeが読み取れるデータソースです。設定ファイル、データベースの内容、APIレスポンスなどを公開できます。

```typescript
server.resource("config", "config://app", async () => ({
  contents: [{ uri: "config://app", text: JSON.stringify(config) }],
}));
```

### Prompts（プロンプト）

プロンプトは、テンプレート化された指示や質問です。よく使うプロンプトをMCPサーバーで管理し、Claudeに提供できます。

```typescript
server.prompt("code-review", async ({ language }) => ({
  messages: [
    {
      role: "user",
      content: `Please review this ${language} code for best practices...`,
    },
  ],
}));
```

## TypeScriptでMCPサーバーを作成する

TypeScriptは、MCPサーバー開発で最も推奨される言語です。型安全性が高く、公式SDKが充実しています。

### プロジェクトのセットアップ

まず、新しいTypeScriptプロジェクトを作成します。

```bash
mkdir my-mcp-server
cd my-mcp-server
npm init -y
npm install @modelcontextprotocol/sdk zod
npm install -D typescript @types/node
```

`tsconfig.json`を作成します。

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true
  }
}
```

### 基本的なサーバー実装

`src/index.ts`に、天気情報を取得するMCPサーバーを実装します。

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

// サーバーインスタンスの作成
const server = new McpServer({
  name: "weather-server",
  version: "1.0.0",
});

// 天気情報を取得するツール
server.tool(
  "get_weather",
  {
    city: z.string().describe("都市名（例: Tokyo, New York）"),
    units: z.enum(["metric", "imperial"]).optional().describe("温度の単位"),
  },
  async ({ city, units = "metric" }) => {
    try {
      // 実際のAPIコール（ここでは簡略化）
      const weather = await fetchWeatherData(city, units);

      return {
        content: [
          {
            type: "text",
            text: JSON.stringify(weather, null, 2),
          },
        ],
      };
    } catch (error) {
      return {
        content: [
          {
            type: "text",
            text: `エラー: ${error.message}`,
          },
        ],
        isError: true,
      };
    }
  }
);

// 設定リソース
server.resource("config", "config://weather", async () => ({
  contents: [
    {
      uri: "config://weather",
      mimeType: "application/json",
      text: JSON.stringify({
        apiVersion: "1.0.0",
        supportedCities: ["Tokyo", "New York", "London", "Paris"],
      }),
    },
  ],
}));

// ダミーの天気データ取得関数
async function fetchWeatherData(city: string, units: string) {
  // 本来はここで実際のAPIを呼び出します
  return {
    city,
    temperature: units === "metric" ? 22 : 72,
    condition: "晴れ",
    humidity: 65,
    units,
  };
}

// サーバーの起動
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("Weather MCP Server started");
}

main().catch(console.error);
```

### Claude Codeでの設定

`package.json`に起動スクリプトを追加します。

```json
{
  "name": "weather-server",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

ビルドして、Claude Codeの設定ファイルに追加します。

```bash
npm run build
```

`~/.claude/mcp.json`:

```json
{
  "mcpServers": {
    "weather": {
      "command": "node",
      "args": ["/Users/yourname/my-mcp-server/dist/index.js"]
    }
  }
}
```

Claude Codeを再起動すると、天気情報ツールが利用可能になります。

## PythonでMCPサーバーを作成する

Pythonでもシンプルにサーバーを実装できます。

### プロジェクトのセットアップ

```bash
mkdir my-python-mcp
cd my-python-mcp
python -m venv venv
source venv/bin/activate  # Windowsの場合: venv\Scripts\activate
pip install mcp
```

### 基本的なサーバー実装

`server.py`:

```python
import asyncio
import json
from mcp.server import Server
import mcp.types as types

# サーバーインスタンス
server = Server("calculator-server")

@server.tool()
async def calculate(expression: str) -> str:
    """
    数式を評価して結果を返します。

    Args:
        expression: 評価する数式（例: "2 + 2", "10 * 5"）
    """
    try:
        # セキュリティ上、evalは本番では使用しないでください
        result = eval(expression, {"__builtins__": {}}, {})
        return json.dumps({
            "expression": expression,
            "result": result
        })
    except Exception as e:
        return json.dumps({
            "error": str(e)
        })

@server.resource("calculator://info")
async def get_info() -> str:
    """計算機の情報を返すリソース"""
    return json.dumps({
        "name": "Calculator Server",
        "version": "1.0.0",
        "capabilities": ["基本的な四則演算"]
    })

async def main():
    from mcp.server.stdio import stdio_server

    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            server.create_initialization_options()
        )

if __name__ == "__main__":
    asyncio.run(main())
```

### Claude Codeでの設定

`~/.claude/mcp.json`:

```json
{
  "mcpServers": {
    "calculator": {
      "command": "python",
      "args": ["/Users/yourname/my-python-mcp/server.py"]
    }
  }
}
```

## SSEトランスポート（リモートサーバー）

これまでの例はStdio（標準入出力）を使ったローカルサーバーでしたが、SSE（Server-Sent Events）を使えばHTTP経由でリモートサーバーとして動作できます。

### SSEサーバーの実装

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { SSEServerTransport } from "@modelcontextprotocol/sdk/server/sse.js";
import express from "express";

const app = express();
const server = new McpServer({
  name: "remote-server",
  version: "1.0.0",
});

// ツールの定義（前述と同様）
server.tool("example_tool", { input: z.string() }, async ({ input }) => {
  return { content: [{ type: "text", text: `処理結果: ${input}` }] };
});

// SSEエンドポイント
app.get("/sse", async (req, res) => {
  const transport = new SSEServerTransport("/messages", res);
  await server.connect(transport);
});

// メッセージ受信エンドポイント
app.post("/messages", express.json(), async (req, res) => {
  // MCPメッセージの処理
  await transport.handlePostMessage(req, res);
});

app.listen(3000, () => {
  console.log("SSE MCP Server running on http://localhost:3000");
});
```

### クライアント側の設定

```json
{
  "mcpServers": {
    "remote": {
      "url": "http://localhost:3000/sse"
    }
  }
}
```

SSEトランスポートの利点:

- 複数のクライアントが同じサーバーに接続可能
- Web APIとして公開でき、ファイアウォール越しでも利用可能
- 認証やロギングなどのミドルウェアを追加しやすい

## Tool Search機能

MCPサーバーが増えると、すべてのツール定義がClaude Codeの初期コンテキストに含まれ、トークンを圧迫します。Tool Search機能を使うと、必要なツールだけをオンデマンドでロードできます。

### 自動有効化

Claude Codeは、ツール定義がコンテキストの10%を超えると、自動的にTool Searchを有効にします。手動で有効化するには:

```bash
ENABLE_TOOL_SEARCH=true claude
```

### カスタム閾値

閾値を5%に設定する例:

```bash
ENABLE_TOOL_SEARCH=auto:5 claude
```

### 仕組み

Tool Searchが有効な場合:

1. Claudeはツールの名前と簡潔な説明のみを受け取ります
2. Claudeが特定のツールを使用したいとき、Tool Searchクエリを発行します
3. Claude Codeが該当するツールの完全な定義を取得します
4. Claudeがそのツールを呼び出します

これにより、数百のツールを持つ環境でも効率的に動作します。

## 管理されたMCP設定（エンタープライズ）

エンタープライズ環境では、組織全体でMCPサーバーの利用を管理する必要があります。`managed-mcp.json`を使うと、許可されたサーバーのみを利用できるよう制御できます。

### managed-mcp.jsonの配置

組織全体の設定: `~/.claude/managed-mcp.json`

プロジェクト固有の設定: `<project>/.claude/managed-mcp.json`

### 設定例

```json
{
  "allowedMcpServers": [
    {
      "serverName": "github",
      "comment": "GitHub統合は全プロジェクトで許可"
    },
    {
      "serverUrl": "https://mcp.company.com/*",
      "comment": "社内MCPサーバーのみ許可"
    },
    {
      "serverName": "filesystem",
      "allowedPaths": ["/home/*/projects/*"],
      "comment": "ファイルシステムアクセスはprojectsディレクトリのみ"
    }
  ],
  "blockedMcpServers": [
    {
      "serverName": "external-api",
      "reason": "セキュリティレビュー未完了"
    }
  ],
  "requireApproval": {
    "newServers": true,
    "comment": "新しいサーバーは承認が必要"
  }
}
```

### 設定項目

| 項目 | 説明 |
|------|------|
| `allowedMcpServers` | 許可するサーバーのリスト（ホワイトリスト） |
| `blockedMcpServers` | 禁止するサーバーのリスト（ブラックリスト） |
| `serverName` | サーバー名による指定 |
| `serverUrl` | URL パターンによる指定（ワイルドカード対応） |
| `allowedPaths` | ファイルシステムサーバーで許可するパス |
| `requireApproval` | 新規サーバー追加時の承認要求 |

組織の管理者が`managed-mcp.json`を配布することで、開発者は承認されたツールのみを利用できます。

## Claude Code自体をMCPサーバーとして使う

Claude Code v1.10以降では、`claude mcp serve`モードが利用可能です。これにより、Claude Code自体がMCPサーバーとして動作し、他のMCPクライアントから利用できます。

### サーバーモードの起動

```bash
claude mcp serve --port 3000
```

これで、Claude Codeのファイル編集、コマンド実行、検索などの機能がMCPプロトコルで公開されます。

### 利用例

別のClaude Codeインスタンスや、他のMCPクライアントから接続できます。

```json
{
  "mcpServers": {
    "remote-claude": {
      "url": "http://localhost:3000/sse"
    }
  }
}
```

これにより、以下のようなユースケースが可能になります:

- リモートサーバー上のClaude Codeを操作
- 複数のClaude Codeインスタンスを連携
- CI/CD パイプラインからClaude Codeを呼び出し
- カスタムUIからClaude Codeを制御

### 認証の設定

本番環境では必ず認証を設定してください。

```bash
claude mcp serve --port 3000 --auth-token your-secret-token
```

クライアント側:

```json
{
  "mcpServers": {
    "remote-claude": {
      "url": "http://localhost:3000/sse",
      "headers": {
        "Authorization": "Bearer your-secret-token"
      }
    }
  }
}
```

## セキュリティ考慮事項

MCPサーバーを開発・運用する際は、以下のセキュリティ対策が重要です。

### 入力バリデーション

すべてのツール入力を厳密に検証します。

```typescript
server.tool(
  "execute_command",
  {
    command: z.string().regex(/^[a-zA-Z0-9\s\-_]+$/, "無効な文字が含まれています"),
  },
  async ({ command }) => {
    // コマンドインジェクション対策
    const allowedCommands = ["ls", "pwd", "date"];
    const cmd = command.split(" ")[0];

    if (!allowedCommands.includes(cmd)) {
      throw new Error("このコマンドは許可されていません");
    }

    // 実行処理
  }
);
```

### 認証とアクセス制御

SSEサーバーでは、必ず認証を実装します。

```typescript
app.use((req, res, next) => {
  const token = req.headers.authorization?.replace("Bearer ", "");

  if (token !== process.env.AUTH_TOKEN) {
    return res.status(401).json({ error: "認証が必要です" });
  }

  next();
});
```

### レート制限

APIの過度な使用を防ぎます。

```typescript
import rateLimit from "express-rate-limit";

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15分
  max: 100, // 100リクエストまで
});

app.use(limiter);
```

### 最小権限の原則

ツールに必要最小限の権限のみを与えます。

```typescript
// 悪い例: 任意のファイルを読める
server.tool("read_file", { path: z.string() }, async ({ path }) => {
  return fs.readFileSync(path, "utf-8");
});

// 良い例: 特定のディレクトリ内のみ
server.tool("read_file", { path: z.string() }, async ({ path }) => {
  const basePath = "/safe/directory";
  const fullPath = path.resolve(basePath, path);

  if (!fullPath.startsWith(basePath)) {
    throw new Error("アクセスが拒否されました");
  }

  return fs.readFileSync(fullPath, "utf-8");
});
```

### ログとモニタリング

すべてのツール呼び出しをログに記録します。

```typescript
server.tool("sensitive_operation", schema, async (params) => {
  console.log({
    timestamp: new Date().toISOString(),
    tool: "sensitive_operation",
    params: JSON.stringify(params),
    user: getCurrentUser(),
  });

  // 処理
});
```

### 環境変数の管理

機密情報はコードに直接書かず、環境変数で管理します。

```typescript
const API_KEY = process.env.API_KEY;

if (!API_KEY) {
  throw new Error("API_KEY環境変数が設定されていません");
}
```

`.env`ファイル:

```
API_KEY=your-secret-key
DATABASE_URL=postgres://user:pass@localhost/db
```

Gitに`.env`をコミットしないよう、`.gitignore`に追加します。

```
.env
.env.local
```

## まとめ

| 項目 | 内容 |
|------|------|
| **MCPの3機能** | Tools（関数）、Resources（データ）、Prompts（テンプレート） |
| **TypeScript実装** | `@modelcontextprotocol/sdk`を使用、型安全で推奨 |
| **Python実装** | `mcp`パッケージでシンプルに実装可能 |
| **Stdioトランスポート** | ローカルプロセスとして動作、設定が簡単 |
| **SSEトランスポート** | HTTP経由でリモート接続、複数クライアント対応 |
| **Tool Search** | 多数のツールを効率的に管理、自動または手動で有効化 |
| **managed-mcp.json** | エンタープライズ環境でのホワイトリスト/ブラックリスト管理 |
| **claude mcp serve** | Claude Code自体をMCPサーバーとして公開 |
| **セキュリティ** | 入力検証、認証、レート制限、最小権限、ログが重要 |

## やってみよう！

以下の要件で簡単なMCPサーバーを作成してみましょう。

**課題: メモ管理サーバー**

以下の機能を持つMCPサーバーを実装してください:

1. `create_note`: タイトルと本文を受け取り、JSONファイルとして保存
2. `list_notes`: すべてのメモのタイトル一覧を返す
3. `get_note`: タイトルを指定してメモの内容を取得
4. `notes://list`: メモ一覧を返すリソース

ヒント:

```typescript
// メモの保存先
const NOTES_DIR = path.join(os.homedir(), ".mcp-notes");

// ファイル操作
fs.writeFileSync(
  path.join(NOTES_DIR, `${title}.json`),
  JSON.stringify({ title, content, createdAt: new Date() })
);
```

完成したら、Claude Codeで「今日のタスクをメモに保存して」と依頼してみましょう。

次の章では、Claude Codeを使った実践的なプロジェクト開発のワークフローを解説します。
