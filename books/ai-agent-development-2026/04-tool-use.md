---
title: "ツール利用（Function Calling）"
---

## Function Calling とは何か

LLM は膨大な知識を持っていますが、単体では「テキストを生成する」ことしかできません。最新の天気を調べたり、データベースを検索したり、メールを送信したりといった **外部の世界と対話する能力** は持っていません。

Function Calling は、この制約を突破する仕組みです。LLM に「どの関数を、どの引数で呼び出すべきか」を JSON 形式で出力させ、アプリケーション側が実際に関数を実行します。その結果を LLM に返すことで、自然言語による最終回答を生成させます。

つまり、LLM は関数を **直接実行しません**。あくまで「呼び出しの意図」を構造化データとして出力するだけです。実行の責任はアプリケーション側にあります。

## ツール実行フローの全体像

Function Calling の処理は、以下の 6 つのステップで進みます。

```
1. 開発者がツール定義（関数名・説明・パラメータ）を LLM に渡す
2. ユーザーがメッセージを送信する（例: 「東京の天気を教えて」）
3. LLM がツール呼び出しの必要性を判断し、JSON で関数名と引数を返す
4. アプリケーションが実際に関数を実行する
5. 実行結果を LLM に返却する
6. LLM が結果を踏まえて自然言語で最終回答を生成する
```

ポイントは、ステップ 3 と 4 の間にアプリケーションが介在することです。ここでバリデーション、権限チェック、ログ記録などの処理を挟むことができます。

## ツール定義の書き方（JSON Schema）

ツールの定義は JSON Schema に基づいて記述します。ここが Function Calling の精度を大きく左右する部分です。

### 基本構造

Anthropic Claude API の場合、ツール定義は以下の形式です。

```json
{
  "name": "get_weather",
  "description": "指定された都市の現在の天気情報を取得します。都市名を日本語または英語で指定できます。",
  "input_schema": {
    "type": "object",
    "properties": {
      "city": {
        "type": "string",
        "description": "天気を取得する都市名（例: 東京、Osaka）"
      },
      "unit": {
        "type": "string",
        "enum": ["celsius", "fahrenheit"],
        "description": "温度の単位。デフォルトは celsius"
      }
    },
    "required": ["city"]
  }
}
```

OpenAI API の場合は、`tools` 配列の中に `type: "function"` と `function` オブジェクトを含めます。

```json
{
  "type": "function",
  "function": {
    "name": "get_weather",
    "description": "指定された都市の現在の天気情報を取得します",
    "parameters": {
      "type": "object",
      "properties": {
        "city": {
          "type": "string",
          "description": "天気を取得する都市名"
        }
      },
      "required": ["city"]
    }
  }
}
```

### 良いツール定義を書くコツ

ツール定義の品質は、LLM が正しいツールを選択できるかどうかに直結します。以下の 4 つを意識してください。

1. **`name` は動詞_名詞 の snake_case** にします（例: `search_products`, `create_order`）
2. **`description` は具体的に** 書きます。何をするか、いつ使うか、入出力の形式を含めます
3. **各パラメータにも `description` を付ける** ことで、引数の精度が向上します
4. **`enum` や `minimum`/`maximum` で制約** を付けると、不正な値の生成を防げます

## エージェントループの実装

ツール呼び出しは 1 回で完了するとは限りません。LLM が複数のツールを連続的に呼び出したい場合があります。これを処理するのがエージェントループです。

```python
import anthropic
import json

client = anthropic.Anthropic()

tools = [
    {
        "name": "get_weather",
        "description": "指定された都市の現在の天気情報を取得します",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "都市名（例: 東京）"
                }
            },
            "required": ["city"]
        }
    },
    {
        "name": "search_restaurant",
        "description": "指定エリアのレストランを検索します",
        "input_schema": {
            "type": "object",
            "properties": {
                "area": {"type": "string", "description": "エリア名"},
                "budget": {"type": "integer", "description": "予算上限（円）"}
            },
            "required": ["area"]
        }
    }
]

def execute_tool(name: str, input_data: dict) -> str:
    """実際のツール実行（本来は外部 API 呼び出し等）"""
    if name == "get_weather":
        return json.dumps({"city": input_data["city"], "weather": "晴れ", "temp": 22})
    elif name == "search_restaurant":
        return json.dumps({"results": [{"name": "寿司太郎", "budget": 5000}]})
    return json.dumps({"error": "Unknown tool"})

def agent_loop(user_message: str) -> str:
    messages = [{"role": "user", "content": user_message}]

    for _ in range(10):  # ループ上限を設定
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            tools=tools,
            messages=messages
        )

        # 最終回答の場合はテキストを返す
        if response.stop_reason == "end_turn":
            return "".join(
                block.text for block in response.content
                if hasattr(block, "text")
            )

        # ツール呼び出しを処理
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result
                })

        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})

    return "処理が上限に達しました。"
```

`for _ in range(10)` のようにループ上限を設定することが重要です。これがないと、LLM が無限にツールを呼び続ける可能性があります。

## 並列ツール呼び出し

LLM は、互いに依存関係のない複数のツールを 1 回のレスポンスで同時に呼び出すことがあります。例えば「渋谷のレストランと今の天気を教えて」という質問に対して、`search_restaurant` と `get_weather` が同時に返されます。

この場合、アプリケーション側でも並列に実行することでレイテンシを削減できます。

```python
import asyncio

async def execute_tools_parallel(tool_calls: list) -> list:
    """複数のツール呼び出しを並列実行します"""

    async def run_one(call):
        try:
            result = await asyncio.to_thread(
                execute_tool, call["name"], call["input"]
            )
            return {
                "type": "tool_result",
                "tool_use_id": call["id"],
                "content": result
            }
        except Exception as e:
            return {
                "type": "tool_result",
                "tool_use_id": call["id"],
                "content": json.dumps({"error": str(e)}),
                "is_error": True
            }

    return await asyncio.gather(*[run_one(c) for c in tool_calls])
```

OpenAI API では `parallel_tool_calls=True`（デフォルト）で並列呼び出しが有効になります。Anthropic API ではデフォルトで並列呼び出しに対応しています。

## tool_choice によるツール選択の制御

LLM がツールを使うかどうか、どのツールを使うかを制御するパラメータが `tool_choice` です。

| 値 | 動作 |
|---|---|
| `auto` | LLM が自動判断します（デフォルト） |
| `any` / `required` | 必ず何らかのツールを呼び出します |
| `none` | ツールを使わず直接回答します |
| 特定ツール指定 | 指定したツールを必ず呼び出します |

Anthropic では `{"type": "tool", "name": "get_weather"}` のように特定ツールを強制できます。OpenAI では `{"type": "function", "function": {"name": "get_weather"}}` の形式です。

## エラーハンドリング

ツール実行は外部システムに依存するため、さまざまなエラーが起こり得ます。適切に処理して LLM に伝えることで、代替案の提案やリトライが可能になります。

```python
def safe_execute_tool(name: str, args: dict) -> dict:
    """エラーハンドリング付きのツール実行"""
    # 関数の存在確認
    registry = {"get_weather": get_weather_impl, "search_restaurant": search_impl}
    if name not in registry:
        return {"error": f"Unknown tool: {name}", "status": "error"}

    # 引数のバリデーション
    try:
        validated = validate_args(name, args)
    except ValueError as e:
        return {"error": f"Invalid arguments: {e}", "status": "error"}

    # タイムアウト付き実行
    import signal

    def timeout_handler(signum, frame):
        raise TimeoutError("Tool execution timed out")

    signal.signal(signal.SIGALRM, timeout_handler)
    signal.alarm(10)  # 10秒タイムアウト

    try:
        result = registry[name](**validated)
        signal.alarm(0)  # タイマー解除
        return {"result": result, "status": "success"}
    except TimeoutError:
        return {
            "error": "タイムアウトしました。条件を変えて再試行してください。",
            "status": "timeout"
        }
    except Exception as e:
        signal.alarm(0)
        return {"error": str(e), "status": "error"}
```

LLM にエラーを返すときは、**何が起きたか**と**どうすればよいか**を含めると、LLM が適切にユーザーへ伝えられます。

```python
# 悪い例: 生のエラーメッセージ
{"error": "ConnectionRefusedError: [Errno 61]"}

# 良い例: LLM が理解しやすい形式
{
    "error": True,
    "message": "天気情報サービスに接続できませんでした。",
    "suggestion": "しばらく待ってから再試行するか、別の方法を提案してください。"
}
```

## カスタムツールの実装例：Web 検索ツール

実践的な例として、Web 検索ツールを実装してみましょう。ツール定義、実行関数、エージェントループの 3 つを組み合わせます。

```python
import anthropic, json, requests

client = anthropic.Anthropic()

# 1. ツール定義
web_search_tool = {
    "name": "web_search",
    "description": (
        "Web上の情報を検索します。最新のニュース、技術情報、"
        "一般的な事実の確認に使用してください。"
        "結果は最大5件のタイトル・スニペット・URLを返します。"
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "検索クエリ（例: 'Python 3.13 新機能'）"
            },
            "num_results": {
                "type": "integer", "minimum": 1, "maximum": 10,
                "description": "取得件数（デフォルト: 5）"
            }
        },
        "required": ["query"]
    }
}

# 2. ツール実行関数（エラーハンドリング付き）
def execute_web_search(query: str, num_results: int = 5) -> str:
    try:
        resp = requests.get(
            "https://api.example.com/search",
            params={"q": query, "num": num_results}, timeout=10
        )
        resp.raise_for_status()
        items = resp.json().get("results", [])[:num_results]
        results = [{"title": i["title"], "snippet": i["snippet"][:300],
                     "url": i["url"]} for i in items]
        return json.dumps({"query": query, "count": len(results),
                           "results": results}, ensure_ascii=False)
    except requests.Timeout:
        return json.dumps({"error": "タイムアウト",
                           "suggestion": "クエリを短くして再試行してください"})
    except Exception as e:
        return json.dumps({"error": f"検索エラー: {e}"})

# 3. エージェントループ（前述のパターンと同じ構造）
def search_agent(question: str) -> str:
    messages = [{"role": "user", "content": question}]
    for _ in range(5):
        response = client.messages.create(
            model="claude-sonnet-4-20250514", max_tokens=4096,
            tools=[web_search_tool], messages=messages
        )
        if response.stop_reason == "end_turn":
            return "".join(b.text for b in response.content if hasattr(b, "text"))
        for block in response.content:
            if block.type == "tool_use":
                result = execute_web_search(**block.input)
                messages.append({"role": "assistant", "content": response.content})
                messages.append({"role": "user", "content": [{
                    "type": "tool_result",
                    "tool_use_id": block.id, "content": result}]})
    return "検索回数の上限に達しました。"
```

## ツール設計のアンチパターン

Function Calling を使う際に避けるべき 4 つのパターンです。

1. **ツールの過剰提供** -- 30 個以上を一度に渡すと選択精度が下がります。5-15 個に絞り、多い場合は動的にフィルタリングしてください
2. **説明文の不足** -- 「データを処理する」のような曖昧な `description` では LLM が正しく選べません。何をするか、いつ使うか、戻り値は何かを具体的に書きます
3. **エラーハンドリングの欠如** -- try-except なしでツールを実行すると、例外発生時にエージェントループ全体がクラッシュします
4. **結果の肥大化** -- 10,000 行の結果をそのまま返すとコンテキストウィンドウを浪費します。件数制限やページネーションを適用してください

## セキュリティ上の注意点

Function Calling には固有のセキュリティリスクがあります。

- **SQL インジェクション**: LLM が生成した SQL をそのまま実行してはいけません。SELECT のみ許可し、テーブルのホワイトリストを設けます
- **パストラバーサル**: ファイルパスを受け取るツールでは、許可ディレクトリ外へのアクセスを防止します
- **破壊的操作の制御**: `delete_file` や `send_email` のような副作用のあるツールは、実行前にユーザー確認を挟みます

```python
# 人間の承認を挟む例
DESTRUCTIVE_TOOLS = {"delete_file", "send_email", "create_order"}

def execute_with_approval(tool_name: str, args: dict) -> dict:
    if tool_name in DESTRUCTIVE_TOOLS:
        print(f"[確認] {tool_name} を実行しますか？ 引数: {args}")
        approval = input("yes/no: ")
        if approval != "yes":
            return {"status": "rejected", "message": "ユーザーが操作を拒否しました"}
    return execute_tool(tool_name, args)
```

## まとめ

| 項目 | 内容 |
|---|---|
| Function Calling の本質 | LLM が「どの関数を、どの引数で呼ぶか」を JSON で出力する仕組みです |
| 実行責任 | アプリケーション側にあります。LLM は関数を直接実行しません |
| ツール定義 | JSON Schema で記述します。`description` の品質が精度を決めます |
| 並列呼び出し | 主要プロバイダ（OpenAI・Anthropic・Google）すべてが対応しています |
| エラーハンドリング | タイムアウト、ループ上限、入力バリデーションが必須です |
| セキュリティ | 入力サニタイズ、権限チェック、破壊的操作の承認フローを実装します |
| tool_choice | `auto`/`any`/`none`/特定ツール指定 で呼び出しを制御できます |
| 推奨ツール数 | 1 リクエストあたり 5 - 15 個。多い場合は動的にフィルタリングします |

## やってみよう！

- [ ] 天気取得ツールを定義して、エージェントループで実際に動かしてみましょう
- [ ] `tool_choice` を `auto`、`any`、`none` に切り替えて、LLM の動作の違いを観察しましょう
- [ ] ツールの `description` を意図的に曖昧にした場合と、詳細に書いた場合で精度がどう変わるか比較しましょう
- [ ] 2 つ以上のツールを定義して、並列呼び出しが発生するプロンプトを試してみましょう
- [ ] エラー時にリトライや代替手段を提案するエラーハンドリングを実装してみましょう
