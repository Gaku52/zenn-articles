---
title: "シングルエージェントパターン"
---

## シングルエージェントとは

AIエージェント開発において、最初に検討すべきアーキテクチャが**シングルエージェント**です。1つのLLMインスタンスがループの中でツールを使いながらタスクを遂行するパターンで、多くのユースケースに対応できます。

```
 シンプル                                              複雑
 +--------+--------+-----------+-----------+-----------+
 | LLM    | Chain  | Single    | Multi     | Autonomous|
 | 直接   | (直列) | Agent     | Agent     | Agent     |
 | 呼出   |        | (ReAct)   | (協調)    | (自律)    |
 +--------+--------+-----------+-----------+-----------+
                    ^^^^^^^^^^^
                    この章の範囲
```

シングルエージェントには次のような利点があります。

- **デバッグが容易**: 1つのLLMの思考過程を追跡するだけで済みます
- **コスト予測が可能**: API呼び出し回数を制御しやすいです
- **レイテンシが低い**: マルチエージェント間の通信オーバーヘッドがありません
- **実装がシンプル**: 動くプロトタイプを素早く構築できます

まず「シングルエージェントで解決できないか？」を検討し、不十分な場合にのみマルチエージェントへ進むのが設計の鉄則です。

## ReActパターンの設計と実装

ReActは **Re**asoning + **Act**ing の略で、「考えてから行動する」を繰り返すパターンです。シングルエージェントの中核となる思考フレームワークです。

```
ReAct ループ

  Thought ─────> Action ─────> Observation
     ^                              |
     |______________________________|
            (繰り返し)

  最終的に Final Answer を出力して終了
```

1. **Thought（思考）**: 現在の状況を分析し、次に何をすべきか推論します
2. **Action（行動）**: 選択したツールを実行します
3. **Observation（観察）**: ツールからの結果を受け取り、判断します

現在のエージェント開発では、テキストパースに頼るReActよりも**Function Calling**（構造化JSON）が主流です。以下はClaude APIの`tool_use`を使った実装です。

```python
import anthropic
from dataclasses import dataclass
from typing import Callable

@dataclass
class ToolDefinition:
    name: str
    description: str
    input_schema: dict
    handler: Callable
    dangerous: bool = False

class SingleAgent:
    def __init__(self, tools: list[ToolDefinition],
                 system_prompt: str = "",
                 model: str = "claude-sonnet-4-20250514"):
        self.client = anthropic.Anthropic()
        self.tools = tools
        self.system_prompt = system_prompt
        self.model = model
        self._handlers = {t.name: t for t in tools}

    def run(self, query: str, max_steps: int = 15) -> str:
        messages = [{"role": "user", "content": query}]
        api_tools = [
            {"name": t.name, "description": t.description,
             "input_schema": t.input_schema}
            for t in self.tools
        ]
        for step in range(max_steps):
            response = self.client.messages.create(
                model=self.model, max_tokens=4096,
                system=self.system_prompt,
                tools=api_tools, messages=messages
            )
            if response.stop_reason == "end_turn":
                return self._extract_text(response)

            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    td = self._handlers.get(block.name)
                    try:
                        result = td.handler(**block.input) if td else "Unknown tool"
                    except Exception as e:
                        result = f"Error: {e}"
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": str(result)
                    })
            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": tool_results})
        return "最大ステップ数に達しました。"

    def _extract_text(self, response) -> str:
        for block in response.content:
            if hasattr(block, "text"):
                return block.text
        return ""
```

ポイントは3つです。`stop_reason == "end_turn"` で終了判定、ツール結果は `tool_use_id` で紐付け、ループ上限 `max_steps` で安全性を確保します。

## ツールチェーンの設計

エージェントの精度はツール定義の品質に直結します。以下の原則に従ってください。

```python
# NG: 曖昧な説明 → LLMが正しく選択できない
bad = {"name": "search", "description": "検索する"}

# OK: 用途・使い分け・制約が明確
good = {
    "name": "search_products",
    "description": (
        "商品データベースを検索して条件に合う商品を返す。"
        "注意: 在庫情報はcheck_inventoryツールを使用すること。"
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {"type": "string", "description": "検索キーワード"},
            "category": {
                "type": "string",
                "enum": ["electronics", "clothing", "food", "books"]
            }
        },
        "required": ["query"]
    }
}
```

| 観点 | 悪い例 | 良い例 |
|------|--------|--------|
| 名前 | `search` | `search_products` |
| 説明 | 「検索する」 | 用途・使い分け・制約を明記 |
| パラメータ名 | `q` | `query` |
| 型制約 | 自由文字列のみ | `enum`、`minimum`等を活用 |

1エージェントあたりのツール数は**5~15個**が推奨です。16個以上になる場合は、タスク種別に応じてツールセットを動的に切り替えるか、マルチエージェント化を検討してください。

## ガードレール設計

エージェントを本番運用するには、安全性を担保する**ガードレール**が不可欠です。入力・実行中・出力の3層で設計します。

```
入力ガードレール       実行中ガードレール       出力ガードレール
├ インジェクション検出  ├ ステップ数上限         ├ PII漏洩チェック
└ 入力長制限           ├ ループ検出             └ 有害コンテンツフィルタ
                       ├ トークン予算管理
                       └ 破壊的操作の確認
```

```python
import json

class AgentGuardrails:
    def __init__(self):
        self.max_steps = 25
        self._step_count = 0
        self._action_history: list[str] = []

    def check_input(self, user_input: str) -> tuple[bool, str | None]:
        """プロンプトインジェクション検出 + 入力長チェック"""
        for pattern in ["ignore previous instructions", "system prompt"]:
            if pattern in user_input.lower():
                return False, f"インジェクション検出: '{pattern}'"
        if len(user_input) > 10_000:
            return False, "入力が長すぎます（最大10,000文字）"
        return True, None

    def check_tool_call(self, tool_name: str,
                        tool_input: dict) -> tuple[bool, str | None]:
        """ステップ上限 + ループ検出"""
        self._step_count += 1
        if self._step_count > self.max_steps:
            return False, f"最大ステップ数（{self.max_steps}）に到達"
        key = f"{tool_name}:{json.dumps(tool_input, sort_keys=True)}"
        self._action_history.append(key)
        recent = self._action_history[-3:]
        if len(recent) == 3 and len(set(recent)) == 1:
            return False, f"ループ検出: '{tool_name}' が同じ入力で3回連続"
        return True, None
```

**ループ検出**は特に重要です。ループに陥ったエージェントはトークンとコストを際限なく消費します。検出したら「別のアプローチを試してください」と明示的に指示するか、部分的な結果を返して強制終了します。

## 実践例: コーディングエージェント

ここまでの概念を統合し、**コーディングエージェント**を構築してみましょう。ファイル読み書き・コマンド実行の3ツール構成です。

```python
import subprocess

def read_file(path: str) -> str:
    """ファイルを読み込んで内容を返す"""
    with open(path) as f:
        content = f.read()
    return content[:8000] + "\n...(truncated)" if len(content) > 8000 else content

def write_file(path: str, content: str) -> str:
    """ファイルに内容を書き込む"""
    with open(path, "w") as f:
        f.write(content)
    return f"wrote {len(content)} chars to {path}"

def run_command(command: str) -> str:
    """シェルコマンドを実行（タイムアウト30秒）"""
    result = subprocess.run(
        command, shell=True, capture_output=True, text=True, timeout=30
    )
    output = result.stdout + result.stderr
    return output[:5000] if output else "(no output)"

# エージェント組み立て
agent = SingleAgent(
    tools=[
        ToolDefinition("read_file", "ファイルを読み込む。コード確認に使用。",
            {"type":"object","properties":{"path":{"type":"string"}},"required":["path"]},
            read_file),
        ToolDefinition("write_file", "ファイルに書き込む。新規作成・修正に使用。",
            {"type":"object","properties":{"path":{"type":"string"},"content":{"type":"string"}},"required":["path","content"]},
            write_file, dangerous=True),
        ToolDefinition("run_command", "シェルコマンドを実行。テストやlintに使用。",
            {"type":"object","properties":{"command":{"type":"string"}},"required":["command"]},
            run_command, dangerous=True),
    ],
    system_prompt=(
        "あなたはコーディングエージェントです。"
        "1. まずread_fileで対象コードを確認する "
        "2. 修正方針を説明してからwrite_fileで書き込む "
        "3. 修正後は必ずrun_commandでテストを実行する "
        "4. テスト失敗時は原因を分析して修正する"
    )
)
result = agent.run("src/utils.pyのparse_date関数にタイムゾーン対応を追加してください")
```

### 実行フローの例

```
ユーザー: 「parse_date関数にタイムゾーン対応を追加して」
 ↓
[Step 1] read_file("src/utils.py")          → 現在のコードを確認
[Step 2] write_file("src/utils.py", "...")   → TZ対応を実装
[Step 3] write_file("tests/test_utils.py")   → テストを作成
[Step 4] run_command("pytest tests/ -v")     → 3 passed, 1 failed
[Step 5] write_file("src/utils.py", "...")   → 失敗ケースを修正
[Step 6] run_command("pytest tests/ -v")     → 4 passed ✓
 ↓
Final Answer: タイムゾーン対応を追加しました。...
```

「読む→書く→テスト→修正」のサイクルを自律的に回すのがポイントです。テスト駆動で品質を担保しています。

## エラーハンドリングの階層

本番運用では、エラーを段階的にエスカレーションさせる設計が必要です。

| Level | 種別 | 対処 |
|-------|------|------|
| 1 | ツール実行エラー | 再試行（最大3回、指数バックオフ） |
| 2 | ツール選択ミス | エラーメッセージで正しいツールを示唆 |
| 3 | ループ | 検出→別アプローチ指示→強制終了 |
| 4 | ガードレール違反 | 操作を拒否し、ユーザーに確認 |
| 5 | 目標達成不可能 | 部分的成果を返却し、ユーザーに報告 |

## まとめ

| 項目 | 内容 |
|------|------|
| シングルエージェント | 1つのLLMがループ内でツールを使いタスクを遂行するパターン |
| ReAct | Thought → Action → Observation の反復で推論と行動を統合 |
| Function Calling | 構造化JSONによるツール呼び出し。テキストパースより確実で推奨 |
| ツール設計 | 明確な説明、enum/制約の活用、1エージェント5~15個が目安 |
| ガードレール | 入力検証・ループ検出・トークン予算・PII除外の3層で設計 |
| エラー処理 | 再試行 → 代替手段 → 強制終了 → ユーザー報告の階層的対応 |
| 適用範囲 | ツール15個以下、10ステップ以内、単一専門領域のタスクに最適 |
| 設計原則 | まずシングルで始め、不十分な場合にのみマルチエージェントへ |

## やってみよう！

以下の課題に取り組むことで、シングルエージェントの理解が深まります。

1. **基本**: 天気APIとWeb検索の2ツールを持つエージェントを実装し、「今日東京で傘は必要？」に答えさせてみましょう
2. **ツール設計**: 自分の業務で使う3~5個のツールを定義し、シングルエージェントに組み込んでみましょう。ツール説明文の書き方でエージェントの精度がどう変わるか観察してください
3. **ガードレール**: ループ検出とトークン予算管理を実装し、意図的に異常系を発生させてテストしましょう
4. **発展**: 本章のコーディングエージェントに`grep`ツールと`list_files`ツールを追加し、「未使用importを見つけて削除して」というタスクを解かせてみましょう
