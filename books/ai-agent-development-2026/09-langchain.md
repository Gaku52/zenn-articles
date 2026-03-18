---
title: "LangChainによるエージェント開発"
---

# LangChainによるエージェント開発

この章では、LLMアプリケーション開発フレームワークの代表格である**LangChain**を使い、実際にエージェントを構築する方法を解説します。

LangChainは「コンポーザビリティ（組み合わせ可能性）」を設計思想の中心に据えています。LLM・プロンプト・ツール・メモリといったコンポーネントを、レゴブロックのように組み合わせてエージェントを構築できます。

## LangChainのアーキテクチャ

LangChainは複数のパッケージで構成されています。

```
langchain-core       ... コアインターフェース・抽象クラス（最も安定）
langchain            ... チェーン・エージェントの実装
langchain-community  ... サードパーティ統合
langchain-anthropic  ... Anthropic公式統合
langgraph            ... ステートフルなグラフベースワークフロー
langsmith            ... テスト・デバッグ・モニタリング
```

コンポーネント間の関係は以下の通りです。

```
+---------------------------------------------------+
|               Application Layer                    |
|  [AgentExecutor]  [Chain (LCEL)]  [Retriever]     |
+---------------------------------------------------+
|               Core Components                      |
|  [ChatModel]  [PromptTemplate]  [Tools]  [Memory] |
+---------------------------------------------------+
|               Integrations                         |
|  [Anthropic]  [OpenAI]  [Chroma]  [Pinecone]     |
+---------------------------------------------------+
```

すべてのコンポーネントは**Runnableプロトコル**を実装しており、`invoke()`・`stream()`・`batch()`という統一インターフェースを持ちます。これがLangChainの柔軟性の源泉です。

## ChatModel -- LLMとの対話

エージェント開発の出発点はChatModelです。各LLMプロバイダに対応したクラスが用意されています。

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import HumanMessage, SystemMessage

llm = ChatAnthropic(model="claude-sonnet-4-20250514", temperature=0)
messages = [
    SystemMessage(content="あなたはPythonの専門家です。"),
    HumanMessage(content="デコレータの仕組みを教えてください。")
]

response = llm.invoke(messages)        # 同期呼び出し
for chunk in llm.stream(messages):     # ストリーミング
    print(chunk.content, end="", flush=True)
```

## Chain -- LCELによるパイプライン構築

LangChainの真価は**LCEL（LangChain Expression Language）**にあります。パイプ演算子 `|` でコンポーネントを接続し、データ処理パイプラインを宣言的に記述できます。

### 基本チェーン

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

llm = ChatAnthropic(model="claude-sonnet-4-20250514", temperature=0)
prompt = ChatPromptTemplate.from_messages([
    ("system", "あなたは{role}です。{style}で回答してください。"),
    ("human", "{input}")
])
output_parser = StrOutputParser()

# パイプ演算子でコンポーネントを接続
chain = prompt | llm | output_parser

result = chain.invoke({
    "role": "Python専門家",
    "style": "簡潔",
    "input": "リスト内包表記の使い方を教えて"
})
```

`prompt`がテンプレートに変数を埋め込み、`llm`がLLMに送信し、`output_parser`が文字列として取り出す流れが一行で表現されています。

### 並列チェーンと分岐チェーン

複数処理の同時実行や条件分岐もLCELで表現できます。

```python
from langchain_core.runnables import RunnableParallel, RunnableBranch

# 並列: 1回の呼び出しで3つの分析を同時実行
parallel_chain = RunnableParallel(
    summary=ChatPromptTemplate.from_template(
        "3行で要約: {text}"
    ) | llm | StrOutputParser(),
    keywords=ChatPromptTemplate.from_template(
        "キーワードを5つ抽出: {text}"
    ) | llm | StrOutputParser(),
)

# 分岐: 条件に応じて異なるチェーンを実行
branch = RunnableBranch(
    (lambda x: "technical" in x["type"],
     technical_prompt | llm | StrOutputParser()),
    general_prompt | llm | StrOutputParser()  # デフォルト
)
```

### フォールバック

プライマリモデルのエラー時に別モデルへ自動切り替えする仕組みも組み込めます。

```python
from langchain_openai import ChatOpenAI

primary = ChatAnthropic(model="claude-sonnet-4-20250514", max_retries=2)
fallback = ChatOpenAI(model="gpt-4o")
resilient_llm = primary.with_fallbacks([fallback])
```

## Agent -- ツールを使うLLM

チェーンは入力から出力への一方通行ですが、**Agent**はLLMが自律的にツールを選択・実行し、結果を踏まえて次のアクションを決定します。

### AgentExecutorによる構築

```python
from langchain.agents import AgentExecutor, create_tool_calling_agent
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

prompt = ChatPromptTemplate.from_messages([
    ("system", "必要に応じてツールを使って回答してください。"),
    MessagesPlaceholder(variable_name="chat_history", optional=True),
    ("human", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad")
])

agent = create_tool_calling_agent(llm, tools, prompt)

executor = AgentExecutor(
    agent=agent,
    tools=tools,
    max_iterations=15,        # 無限ループ防止
    max_execution_time=120,   # タイムアウト（秒）
    handle_parsing_errors=True,
    return_intermediate_steps=True
)

result = executor.invoke({"input": "東京の天気を調べて教えて"})
```

`AgentExecutor`は「LLM呼び出し → ツール実行 → 結果をLLMに返す」のループを回します。`max_iterations`と`max_execution_time`は安全装置として必ず設定してください。

## ツール統合 -- エージェントに能力を与える

ツールはエージェントが外部世界と対話するためのインターフェースです。主に3つの方法で定義します。

### @tool デコレータ（最も手軽）

```python
from langchain.tools import tool

@tool
def search_database(query: str, limit: int = 10) -> str:
    """データベースを検索します。

    Args:
        query: 検索キーワード
        limit: 最大結果数
    """
    import sqlite3
    conn = sqlite3.connect("app.db")
    cursor = conn.execute(
        "SELECT * FROM products WHERE name LIKE ? LIMIT ?",
        (f"%{query}%", limit)
    )
    results = cursor.fetchall()
    conn.close()
    return str(results)
```

docstringがツールの説明としてLLMに渡されます。**説明は具体的かつ明確に**書くことが重要です。

### StructuredTool（型安全）

```python
from langchain.tools import StructuredTool
from pydantic import BaseModel, Field

class EmailInput(BaseModel):
    to: str = Field(description="送信先メールアドレス")
    subject: str = Field(description="件名")
    body: str = Field(description="本文")

email_tool = StructuredTool.from_function(
    func=send_email,
    name="send_email",
    description="メールを送信する",
    args_schema=EmailInput
)
```

### BaseTool継承（最も柔軟）

```python
from langchain.tools import BaseTool

class WebScraperTool(BaseTool):
    name: str = "web_scraper"
    description: str = "指定URLのWebページ内容を取得する"

    def _run(self, url: str) -> str:
        import requests
        return requests.get(url, timeout=10).text[:2000]

    async def _arun(self, url: str) -> str:
        import aiohttp
        async with aiohttp.ClientSession() as session:
            async with session.get(url) as resp:
                return (await resp.text())[:2000]
```

| 方法 | 手軽さ | 型安全性 | 推奨場面 |
|------|--------|---------|---------|
| @tool デコレータ | 高 | 中 | プロトタイプ・簡易ツール |
| StructuredTool | 中 | 高 | 引数の多い本番ツール |
| BaseTool 継承 | 低 | 高 | 複雑なロジック・状態管理 |

ツールが増えるとLLMの選択精度が低下します。**10個以下**を目安にしてください。

## ストリーミング -- リアルタイム応答

### チェーンのストリーミング

```python
for chunk in chain.stream({"input": "非同期処理を解説して"}):
    print(chunk, end="", flush=True)
```

### エージェントのイベントストリーム

エージェントではツール呼び出しも含めたイベントを取得できます。

```python
async def stream_agent():
    async for event in executor.astream_events(
        {"input": "AIニュースを調べて要約して"}, version="v2"
    ):
        kind = event["event"]
        if kind == "on_chat_model_stream":
            print(event["data"]["chunk"].content, end="", flush=True)
        elif kind == "on_tool_start":
            print(f"\n[ツール実行中: {event['name']}]")
        elif kind == "on_tool_end":
            print(f"[ツール完了: {event['name']}]")
```

FastAPIと組み合わせれば、SSE（Server-Sent Events）でフロントエンドにリアルタイム配信することも可能です。

## 実装例 -- カスタマーサポートエージェント

ここまでの知識を統合した実践例です。

```python
from langchain_anthropic import ChatAnthropic
from langchain.agents import AgentExecutor, create_tool_calling_agent
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain.tools import tool
from langchain.memory import ConversationSummaryBufferMemory

@tool
def search_faq(query: str, category: str = "all") -> str:
    """FAQデータベースを検索します。
    Args:
        query: 検索キーワード
        category: カテゴリ（shipping, payment, returns, all）
    """
    faqs = {
        "shipping": [{"q": "配送日数は？", "a": "通常2-3営業日です。"}],
        "returns": [{"q": "返品ポリシー", "a": "到着後14日以内、未使用品に限り返品可能。"}],
    }
    results = []
    for cat in ([category] if category != "all" else faqs.keys()):
        for faq in faqs.get(cat, []):
            if query in faq["q"] or query in faq["a"]:
                results.append(f"[{cat}] Q: {faq['q']} A: {faq['a']}")
    return "\n".join(results) or "該当なし"

@tool
def check_order_status(order_id: str) -> str:
    """注文のステータスを確認します。
    Args:
        order_id: 注文番号（例: ORD-2024-001）
    """
    orders = {
        "ORD-2024-001": {"status": "配送中", "tracking": "JP123456789"},
        "ORD-2024-002": {"status": "処理中", "tracking": None},
    }
    import json
    return json.dumps(orders.get(order_id, {"error": "注文が見つかりません"}),
                      ensure_ascii=False)

@tool
def escalate_to_human(reason: str, priority: str = "normal") -> str:
    """問題をヒューマンオペレーターにエスカレーションします。
    Args:
        reason: エスカレーションの理由
        priority: 優先度（low, normal, high, urgent）
    """
    from datetime import datetime
    ticket_id = f"TICKET-{datetime.now().strftime('%Y%m%d%H%M%S')}"
    return f"チケット作成: {ticket_id}（優先度: {priority}）"

# エージェント構築
llm = ChatAnthropic(model="claude-sonnet-4-20250514", temperature=0)

prompt = ChatPromptTemplate.from_messages([
    ("system", """あなたは「テックストア」のカスタマーサポートAIです。
- 不明な点はFAQを検索して確認する
- 注文の問い合わせは注文番号を確認する
- 解決できない問題はエスカレーションする"""),
    MessagesPlaceholder(variable_name="chat_history", optional=True),
    ("human", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad")
])

tools = [search_faq, check_order_status, escalate_to_human]
agent = create_tool_calling_agent(llm, tools, prompt)

memory = ConversationSummaryBufferMemory(
    llm=ChatAnthropic(model="claude-haiku-4-20250514"),
    memory_key="chat_history",
    return_messages=True,
    max_token_limit=2000
)

support_agent = AgentExecutor(
    agent=agent, tools=tools, memory=memory,
    max_iterations=8, max_execution_time=60,
    handle_parsing_errors=True
)

result = support_agent.invoke({"input": "注文ORD-2024-001の配送状況を教えてください"})
print(result["output"])
# メモリが文脈を保持するので続けて質問できる
result = support_agent.invoke({"input": "届くのはいつ頃ですか？"})
```

## よくある落とし穴

LangChainでエージェントを開発する際に注意すべきポイントです。

**ツール説明の不備**: docstringが曖昧だとLLMが正しいツールを選べません。「何を」「どんな入力で」「何が返るか」を具体的に書きましょう。

**メモリリーク**: `ConversationBufferMemory`は会話が無限に蓄積されます。本番では`ConversationSummaryBufferMemory`を使い、トークン数に上限を設けてください。

**verbose=Trueの本番利用**: 本番ではセキュリティリスクになります。環境変数で切り替えましょう。

**同期処理のブロッキング**: 非同期フレームワークでは`executor.ainvoke()`を使ってください。

## まとめ

| コンポーネント | 役割 | 主要クラス |
|--------------|------|-----------|
| ChatModel | LLMとの対話 | `ChatAnthropic`, `ChatOpenAI` |
| PromptTemplate | プロンプトの構造化 | `ChatPromptTemplate` |
| Chain (LCEL) | パイプライン構築 | `RunnableParallel`, `RunnableBranch` |
| Tool | 外部機能の提供 | `@tool`, `StructuredTool`, `BaseTool` |
| AgentExecutor | エージェントの実行ループ | `AgentExecutor` |
| Memory | 会話履歴の管理 | `ConversationSummaryBufferMemory` |
| Streaming | リアルタイム応答 | `astream_events` |
| Callback | 監視・ログ | `BaseCallbackHandler` |

シンプルなチェーンから始めて段階的に機能を追加するアプローチが有効です。複雑な条件分岐やマルチエージェント構成が必要になったら、次章のLangGraphへの移行を検討してください。

## やってみよう！

1. **基本チェーンの構築**: `ChatPromptTemplate | ChatAnthropic | StrOutputParser`のチェーンを作成し、`stream()`でストリーミング出力を試しましょう。

2. **カスタムツールの作成**: `@tool`デコレータで天気ツールと為替ツールを作り（モックデータ可）、AgentExecutorに組み込んで「東京の天気とドル円レートを教えて」と聞いてみましょう。

3. **メモリ付きエージェント**: `ConversationBufferWindowMemory`を追加し、「さっきの天気は何度だった？」のような追加質問で文脈が保持されることを確認しましょう。

4. **ストリーミング実装**: `astream_events`でエージェントのツール呼び出し過程をリアルタイム表示するスクリプトを作成しましょう。
