---
title: "LangGraphによるワークフロー"
---

## この章で学ぶこと

前章では LangChain を使ったエージェントの基礎を学びました。LangChain の `AgentExecutor` は手軽にエージェントを構築できますが、フローの制御が限られるという課題があります。本章では、LangChain エコシステムの一部である **LangGraph** を使い、状態管理付きのワークフローを明示的に設計・実装する方法を解説します。

この章を読み終えると、以下のことができるようになります。

- LangGraph の 3 つのコア概念（State / Node / Edge）を理解する
- 条件分岐やサイクル（ループ）を含むグラフを設計する
- Human-in-the-Loop で人間の承認を挟むワークフローを構築する
- チェックポイントによる状態の永続化と中断・再開を実装する

## LangGraph とは何か

### AgentExecutor の限界

LangChain の `AgentExecutor` は、LLM とツールの呼び出しをループで繰り返す仕組みです。シンプルなタスクには十分ですが、以下のような要件が出てくると対応が難しくなります。

- ツール呼び出しの前に人間の承認を挟みたい
- 特定の条件で異なるノードに分岐させたい
- 途中の状態を保存して、あとから再開したい
- 複数のエージェントを協調させたい

LangGraph はこれらの課題を解決するために設計されたフレームワークです。ワークフローを**有向グラフ**として明示的に定義し、各ステップの制御を開発者の手に委ねます。

```mermaid
flowchart LR
    subgraph "AgentExecutor (Black Box)"
        L1["LLM"] --> T1["Tool"] --> L2["LLM"] --> R["Response"]
    end
```

```mermaid
flowchart TD
    subgraph "LangGraph (Explicit State Graph)"
        A["Node A"] --> B{"Conditional Edge"}
        B --> C["Node B"]
        B --> D["Node C"]
        D --> A
        C --> E["END"]
    end
```

### 3 つのコア概念

LangGraph のグラフは、3 つの要素で構成されます。

**1. State（状態）**

グラフ全体で共有されるデータ構造です。`TypedDict` や Pydantic の `BaseModel` で型を定義します。各ノードはこの状態を読み取り、更新結果を返します。

**2. Node（ノード）**

状態を受け取り、更新された値を辞書で返す関数です。LLM の呼び出し、ツールの実行、データの変換など、あらゆる処理をノードとして定義できます。

**3. Edge（エッジ）**

ノード間の接続を定義します。無条件で次のノードに進む「通常エッジ」と、状態の値に応じて遷移先を切り替える「条件エッジ」の 2 種類があります。

## 基本的なグラフの構築

LangGraph を使うには `pip install langgraph langchain-anthropic langchain-core` でパッケージをインストールし、環境変数 `ANTHROPIC_API_KEY` を設定しておきます。

まずは最もシンプルなグラフを作ってみましょう。ノードが 1 つだけのチャットボットです。

```python
from langgraph.graph import StateGraph, END, START
from typing import TypedDict, Annotated
from langchain_anthropic import ChatAnthropic
import operator

# 1. 状態の定義
class ChatState(TypedDict):
    messages: Annotated[list, operator.add]  # メッセージは累積される
    response_count: int

# 2. LLM の準備
llm = ChatAnthropic(model="claude-sonnet-4-20250514")

# 3. ノード関数の定義
def chatbot(state: ChatState) -> dict:
    response = llm.invoke(state["messages"])
    return {
        "messages": [response],
        "response_count": state["response_count"] + 1
    }

# 4. グラフの構築
graph = StateGraph(ChatState)
graph.add_node("chatbot", chatbot)
graph.add_edge(START, "chatbot")
graph.add_edge("chatbot", END)

# 5. コンパイルと実行
app = graph.compile()
result = app.invoke({
    "messages": [("human", "LangGraphについて教えて")],
    "response_count": 0
})
```

ポイントは `Annotated[list, operator.add]` の部分です。これは「ノードが返した値を既存のリストに追加する」という意味のリデューサです。メッセージ履歴のように累積したいフィールドに使います。一方、`response_count` のようなフィールドはリデューサを指定しないため、単純に上書きされます。

## 条件分岐とサイクル

### ツール使用付きエージェント

ツールを使うエージェントでは、条件分岐とサイクルが重要な役割を果たします。LLM がツール呼び出しを返したら `tools` ノードへ、そうでなければ終了へ分岐させます。

```python
from langgraph.prebuilt import ToolNode
from langchain_core.messages import HumanMessage
from langchain.tools import tool
from typing import Literal

class AgentState(TypedDict):
    messages: Annotated[list, operator.add]

@tool
def get_weather(city: str) -> str:
    """都市の天気を取得する"""
    return {"東京": "晴れ 22°C", "大阪": "曇り 20°C"}.get(city, "データなし")

tools = [get_weather]
llm_with_tools = llm.bind_tools(tools)

def agent_node(state: AgentState) -> dict:
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

def should_continue(state: AgentState) -> Literal["tools", "end"]:
    last = state["messages"][-1]
    return "tools" if hasattr(last, "tool_calls") and last.tool_calls else "end"

workflow = StateGraph(AgentState)
workflow.add_node("agent", agent_node)
workflow.add_node("tools", ToolNode(tools))
workflow.add_edge(START, "agent")
workflow.add_conditional_edges("agent", should_continue, {
    "tools": "tools", "end": END
})
workflow.add_edge("tools", "agent")  # ツール結果を LLM に戻す（サイクル）
app = workflow.compile()
```

`add_conditional_edges` が条件分岐の核心です。ルーティング関数が状態を見て遷移先を決め、`tools` から `agent` へのエッジがサイクル（ループ）を形成します。

ルーティング関数は**純粋関数**にしてください。状態を読み取って分岐先の文字列を返すだけにし、状態の変更はノード関数で行います。ルーティング関数内で状態を変更すると、予測困難なバグの原因になります。

### レビュー付き改善サイクル

LLM に生成させた文書をレビューし、品質が不十分なら修正を繰り返すパターンです。

```python
class ReviewState(TypedDict):
    task: str
    draft: str
    review: str
    revision_count: int
    is_approved: bool

def generate(state: ReviewState) -> dict:
    response = llm.invoke(f"タスク: {state['task']}")
    return {"draft": response.content, "revision_count": 0}

def review(state: ReviewState) -> dict:
    response = llm.invoke(
        f"レビューしてください。十分なら'APPROVED'と書いて:\n{state['draft']}"
    )
    return {"review": response.content,
            "is_approved": "APPROVED" in response.content}

def revise(state: ReviewState) -> dict:
    response = llm.invoke(
        f"原文:\n{state['draft']}\nレビュー:\n{state['review']}\n修正して:"
    )
    return {"draft": response.content,
            "revision_count": state["revision_count"] + 1}

def route_review(state: ReviewState) -> Literal["revise", "end"]:
    if state["is_approved"] or state["revision_count"] >= 3:
        return "end"
    return "revise"

graph = StateGraph(ReviewState)
graph.add_node("generate", generate)
graph.add_node("review", review)
graph.add_node("revise", revise)
graph.add_edge(START, "generate")
graph.add_edge("generate", "review")
graph.add_conditional_edges("review", route_review, {
    "revise": "revise", "end": END
})
graph.add_edge("revise", "review")  # サイクル
app = graph.compile()
```

`revision_count >= 3` の上限チェックが重要です。サイクルを設計するときは必ず**最大反復回数**を設けてください。無制限のサイクルは無限ループを引き起こします。

## Human-in-the-Loop

本番環境へのデプロイやメール送信のように取り消しが難しいアクションでは、実行前に人間の承認を挟むことが望ましいです。LangGraph は `interrupt_before` / `interrupt_after` パラメータでこれを実現します。

```python
from langgraph.checkpoint.memory import MemorySaver

class ApprovalState(TypedDict):
    task: str
    plan: str
    approved: bool
    result: str

def create_plan(state: ApprovalState) -> dict:
    plan = llm.invoke(f"タスク: {state['task']}\n実行計画を作成:")
    return {"plan": plan.content}

def execute_plan(state: ApprovalState) -> dict:
    result = llm.invoke(f"計画に従って実行:\n{state['plan']}")
    return {"result": result.content}

def check_approval(state: ApprovalState) -> Literal["execute", "end"]:
    return "execute" if state.get("approved") else "end"

graph = StateGraph(ApprovalState)
graph.add_node("plan", create_plan)
graph.add_node("execute", execute_plan)
graph.add_edge(START, "plan")
graph.add_conditional_edges("plan", check_approval, {
    "execute": "execute", "end": END
})
graph.add_edge("execute", END)

app = graph.compile(
    checkpointer=MemorySaver(),
    interrupt_before=["execute"]  # execute の前で中断
)

# Step 1: 計画作成まで実行（ここで中断する）
config = {"configurable": {"thread_id": "approval-1"}}
result = app.invoke({"task": "本番DBのマイグレーション"}, config=config)

# Step 2: 人間が計画を確認して承認
print(f"計画: {result['plan']}")
app.update_state(config, {"approved": True})

# Step 3: 中断したところから再開
result = app.invoke(None, config=config)
```

`interrupt_before=["execute"]` を指定すると、`execute` ノードに入る直前でグラフの実行が一時停止します。`update_state` で状態を更新してから `invoke(None, config)` を呼ぶと、中断した地点から実行が再開されます。

この仕組みが動作するには**チェックポイント**が必要です。次のセクションで詳しく説明します。

## チェックポイントによる状態の永続化

チェックポイントは、グラフの実行状態をスナップショットとして保存する仕組みです。以下のような場面で役立ちます。

- 会話の継続（スレッド管理）
- Human-in-the-Loop での中断・再開
- エラー発生時のリトライ
- 実行履歴の確認とロールバック

### チェックポインタの種類

LangGraph は 3 種類のチェックポインタを提供しています。

```python
# 1. メモリベース（開発・テスト用）
from langgraph.checkpoint.memory import MemorySaver
checkpointer = MemorySaver()

# 2. SQLite（小規模な本番環境向け）
from langgraph.checkpoint.sqlite import SqliteSaver
checkpointer = SqliteSaver.from_conn_string("checkpoints.db")

# 3. PostgreSQL（本番環境向け）
from langgraph.checkpoint.postgres import PostgresSaver
checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:password@localhost:5432/langgraph_db"
)
checkpointer.setup()  # テーブル初期化（初回のみ）
```

開発中は `MemorySaver` で十分ですが、本番環境では `PostgresSaver` を使うことで複数インスタンス間で状態を共有できます。

### 状態の確認と操作

チェックポイントを設定すると、`get_state` で現在の状態を取得したり、`get_state_history` で過去の任意のチェックポイントに戻ったりできます。

```python
config = {"configurable": {"thread_id": "debug-session"}}
result = app.invoke({"messages": [HumanMessage(content="テスト")]}, config=config)

current_state = app.get_state(config)
print(f"状態: {current_state.values}")
print(f"次のノード: {current_state.next}")
```

## 実践：リサーチエージェントの構築

ここまでの知識を総合して、検索・分析・レポート作成を行うリサーチエージェントを構築してみましょう。条件分岐、サイクル、チェックポイントを組み合わせた実践的な例です。

```python
class ResearchState(TypedDict):
    query: str
    sources: Annotated[list[dict], operator.add]
    analysis: str
    is_sufficient: bool
    search_count: int
    final_report: str

def search(state: ResearchState) -> dict:
    query = state["query"]
    if state.get("analysis"):
        query = llm.invoke(f"不足情報の検索クエリを生成:\n{state['analysis']}").content
    results = [{"title": f"Result: {query}", "content": "..."}]
    return {"sources": results, "search_count": state["search_count"] + 1}

def analyze(state: ResearchState) -> dict:
    sources_text = "\n".join(f"- {s['title']}" for s in state["sources"])
    response = llm.invoke(
        f"クエリ: {state['query']}\n情報:\n{sources_text}\n"
        f"分析してください。十分なら'SUFFICIENT'と明記してください。"
    )
    return {"analysis": response.content,
            "is_sufficient": "SUFFICIENT" in response.content}

def write_report(state: ResearchState) -> dict:
    return {"final_report": llm.invoke(
        f"分析をもとにレポート作成:\n\n{state['analysis']}"
    ).content}

def route_analysis(state: ResearchState) -> Literal["search_more", "report"]:
    if state["is_sufficient"] or state["search_count"] >= 5:
        return "report"
    return "search_more"

graph = StateGraph(ResearchState)
graph.add_node("search", search)
graph.add_node("analyze", analyze)
graph.add_node("report", write_report)
graph.add_edge(START, "search")
graph.add_edge("search", "analyze")
graph.add_conditional_edges("analyze", route_analysis, {
    "search_more": "search", "report": "report"
})
graph.add_edge("report", END)
app = graph.compile(checkpointer=MemorySaver())
```

このグラフは以下のフローで動作します。

1. `search` ノードで情報を検索する
2. `analyze` ノードで情報が十分かを分析する
3. 不十分なら `search` に戻り追加検索する（最大 5 回）
4. 十分になったら `report` ノードで最終レポートを生成する

## グラフの可視化

`app.get_graph().draw_mermaid()` を呼ぶと、グラフ構造を Mermaid 記法で出力できます。Zenn や GitHub の Markdown に貼れば図として確認できるため、デバッグや設計レビューに活用してください。

## よくある落とし穴

LangGraph を使い始めると陥りやすい問題をまとめます。

**状態の肥大化**: すべてのデータを状態に入れるのではなく、大きなデータは外部ストアに保存し、ID だけを状態に持たせましょう。

**無制限のサイクル**: サイクルには必ず最大反復回数のガードを設けてください。LLM の判断に完全に委ねると、無限ループに陥る可能性があります。

**ルーティング関数での副作用**: ルーティング関数は状態を読み取るだけにしてください。状態の変更はノード関数の責務です。

**不要なチェックポイント**: ワンショットの短い処理にチェックポイントを設定すると、オーバーヘッドだけが増えます。中断・再開や会話の継続が不要な場面ではチェックポイントなしで十分です。

## まとめ

| 概念 | 説明 |
|------|------|
| State | グラフ全体で共有されるデータ。TypedDict や Pydantic で定義します |
| Node | 状態を受け取り更新を返す関数。LLM 呼び出しやツール実行を担います |
| Edge | ノード間の接続。通常エッジと条件エッジがあります |
| 条件分岐 | `add_conditional_edges` でルーティング関数に基づき遷移先を切り替えます |
| サイクル | エッジでループを作り、改善・検索の反復処理を実現します |
| Human-in-the-Loop | `interrupt_before` / `interrupt_after` で人間の承認を挟みます |
| チェックポイント | Memory / SQLite / PostgreSQL で状態を永続化し中断・再開を可能にします |
| 可視化 | `draw_mermaid()` でグラフ構造を Mermaid 記法として出力できます |

## やってみよう！

以下の課題に取り組んで、LangGraph の理解を深めましょう。

1. **基本グラフ**: ユーザーの質問を受け取り、回答を生成するチャットボットを LangGraph で構築してみてください。`MemorySaver` を使い、同じ `thread_id` で複数回会話が継続することを確認しましょう。

2. **条件分岐**: ユーザーの入力内容に応じて「翻訳ノード」「要約ノード」「Q&A ノード」に振り分けるルーターグラフを作ってみてください。

3. **改善サイクル**: コードを生成し、テストを実行し、失敗したら修正を繰り返す自己修正エージェントを構築してみてください。最大反復回数を 3 回に設定し、無限ループを防ぐガードも実装しましょう。

4. **Human-in-the-Loop**: メール下書きを LLM に生成させ、人間がレビューして「修正」「承認」「キャンセル」のいずれかを選べるワークフローを構築してみてください。`interrupt_after` を使って下書き作成後に中断し、`update_state` で人間の判断を反映させます。

次の章では、LLM にツールサーバーを接続する標準プロトコル「MCP（Model Context Protocol）」を学びます。
