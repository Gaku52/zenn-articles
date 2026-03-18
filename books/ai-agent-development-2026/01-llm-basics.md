---
title: "LLMの基礎知識"
---

この章では、AIエージェント開発の土台となる **LLM（Large Language Model: 大規模言語モデル）** の基礎を学びます。Transformerの仕組み、トークナイゼーション、主要モデルの特徴、APIの使い方、コスト感覚まで、実務で必要な知識を一通りカバーします。

## Transformerの仕組み

現代のLLMは、ほぼすべて **Transformer** というアーキテクチャをベースにしています。2017年にGoogleが発表した「Attention Is All You Need」論文で提案されました。

### Self-Attentionとは

Transformerの核心は **Self-Attention（自己注意機構）** です。これは、文中の各単語（正確にはトークン）が、他のすべての単語との関連度を計算する仕組みです。

たとえば「猫がマットの上で寝ている」という文では、「寝ている」がどの単語と関連が深いかを数値で算出します。この場合、「猫」との関連度が高くなります。

```
入力: "猫がマットの上で寝ている"

Self-Attentionの計算イメージ:
┌──────────────────────────────────┐
│  「寝ている」から見た関連度        │
│                                    │
│  猫     → 0.45  （高い：主語）     │
│  が     → 0.02                     │
│  マット → 0.15  （中程度：場所）   │
│  の     → 0.01                     │
│  上     → 0.20  （中程度：場所）   │
│  で     → 0.02                     │
│  寝ている → 0.15（自分自身）       │
└──────────────────────────────────┘
```

従来のRNN（再帰型ニューラルネットワーク）は単語を一つずつ順番に処理していたため、長い文の先頭の情報を忘れがちでした。Self-Attentionは全単語間の関係を一度に計算できるため、長い文脈も扱えますし、GPUによる並列処理との相性も抜群です。

### Transformerブロックの全体像

Transformerは、以下のコンポーネントを1つの「ブロック」として、それを何十層も積み重ねた構造をしています。

```
┌─────────────────────────────────┐
│         出力（次の層へ）          │
├─────────────────────────────────┤
│   Layer Norm + 残差接続          │
├─────────────────────────────────┤
│   Feed-Forward Network（FFN）   │
│   入力を高次元に拡張→非線形変換   │
│   →元の次元に戻す                │
├─────────────────────────────────┤
│   Layer Norm + 残差接続          │
├─────────────────────────────────┤
│   Multi-Head Self-Attention     │
│   複数の「視点」で同時に          │
│   関連度を計算する                │
├─────────────────────────────────┤
│   入力（トークン埋め込み）        │
└─────────────────────────────────┘
```

現代のLLM（GPT、Claude、Gemini、LLaMAなど）は、この中でも **Decoder-only** と呼ばれるタイプを採用しています。これは「左から右へ」テキストを生成する方式で、次のトークンを予測することで文章を生成します。

### 次トークン予測

LLMの本質はシンプルです。大量のテキストデータを使って「次に来る単語（トークン）は何か」を予測するように訓練されています。

```
入力: "今日の天気は"
          ↓
LLMの予測: "晴れ"（確率 0.35）
           "曇り"（確率 0.20）
           "雨"  （確率 0.15）
           ...
```

この単純な仕組みを、数千億のパラメータと数兆トークンの学習データでスケールさせることで、高度な言語理解と生成が実現されています。

## トークナイゼーション
### トークンとは何か

LLMはテキストをそのまま処理するのではなく、**トークン** という単位に分割してから処理します。トークンは「単語」とは異なり、単語より短い場合も長い場合もあります。

```
英語: "Hello, World!" → ["Hello", ",", " World", "!"]  (4トークン)
日本語: "大規模言語モデル" → ["大", "規模", "言語", "モデル"]  (4トークン)
※実際の分割はモデルにより異なります
```

### BPE（Byte Pair Encoding）

現代のLLMで最も広く使われているトークン分割手法が **BPE** です。仕組みはシンプルです。

1. 最初は1文字ずつに分割する
2. 最も頻繁に隣り合う文字ペアを見つける
3. そのペアを1つのトークンに統合する
4. 目標の語彙サイズになるまで2-3を繰り返す

```
BPEのマージプロセス（簡略版）:
初期:  ["l", "o", "w"]  ["l", "o", "w", "e", "r"]
Step1: ["lo", "w"]      ["lo", "w", "e", "r"]      ← "l"+"o" を統合
Step2: ["low"]           ["low", "e", "r"]           ← "lo"+"w" を統合
Step3: ["low"]           ["low", "er"]               ← "e"+"r" を統合
```

特に現代のモデルは **Byte-Level BPE** を採用しており、UTF-8のバイト列を基本単位とすることで、どんな言語や記号でも未知語なく処理できます。

### 日本語とトークン効率

日本語は英語に比べてトークン効率が低い（同じ意味でもより多くのトークンを消費する）傾向があります。これはコストに直結するため意識が必要です。

```python
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o")

en_text = "The capital of Japan is Tokyo."
ja_text = "日本の首都は東京です。"

print(f"英語: {len(enc.encode(en_text))} トークン ({len(en_text)} 文字)")
print(f"日本語: {len(enc.encode(ja_text))} トークン ({len(ja_text)} 文字)")
# 日本語は文字あたりのトークン消費が多い
```

大まかな目安として覚えておくと便利です。

| 言語 | 目安 |
|------|------|
| 英語 | 1トークン ≒ 4文字 |
| 日本語 | 1トークン ≒ 1〜2文字 |
| コード | 1トークン ≒ 3文字 |

## 主要モデルの比較

2026年現在、エージェント開発で主に使われるLLMを整理します。

### Claude（Anthropic）

Anthropicが開発するモデルです。本書ではClaudeを中心にエージェント開発を進めます。

- **Claude Opus 4**: 最高性能。複雑な推論、長時間のコーディングタスクに最適
- **Claude Sonnet 4**: 性能とコストのバランスが良い。多くの実務タスクに最適
- **Claude Haiku 3.5**: 高速・低コスト。分類や定型処理向き

200Kトークンのコンテキストウィンドウ、ツール使用への高い対応力、拡張思考機能が特徴です。

### GPT（OpenAI）

OpenAIが開発するモデルで、LLMブームの火付け役です。

- **GPT-4o**: マルチモーダル対応。テキスト・画像・音声を統合的に処理
- **GPT-4o mini**: 低コストで高速。軽量タスクに最適
- **o1 / o3**: 推論特化モデル。数学やコーディングの複雑な問題に強い

### Gemini（Google）

Google DeepMindが開発するモデルです。

- **Gemini 2.5 Pro**: 高性能な推論モデル。100万トークンの超長文コンテキスト
- **Gemini 2.0 Flash**: 高速・低コスト。エージェント用途に最適化

Google Cloudとの統合やVertex AIでのエンタープライズ利用が強みです。

### モデル選択の指針

| 用途 | 推奨モデル | 理由 |
|------|-----------|------|
| 複雑な推論・分析 | Claude Opus 4 / o3 | 高い推論能力 |
| 日常的な開発タスク | Claude Sonnet 4 / GPT-4o | コスパのバランス |
| 大量の定型処理 | Claude Haiku 3.5 / GPT-4o mini | 低コスト・高速 |
| 超長文の処理 | Gemini 2.5 Pro | 100万トークン対応 |
| エージェント開発 | Claude Sonnet 4 | ツール使用の安定性 |

## APIの基本的な使い方

### Anthropic API（Claude）

ClaudeのAPIは、Messages APIを通じてアクセスします。

```python
from anthropic import Anthropic

client = Anthropic()  # ANTHROPIC_API_KEY 環境変数を自動読み取り

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Pythonでフィボナッチ数列を計算する関数を書いてください"}
    ]
)

print(response.content[0].text)
```

システムプロンプトで振る舞いを指定することもできます。

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system="あなたはシニアPythonエンジニアです。コードは型ヒント付きで、docstringを必ず含めてください。",
    messages=[
        {"role": "user", "content": "リトライ機能付きのHTTPクライアントを実装してください"}
    ]
)
```

### OpenAI API（GPT）

OpenAIのAPIも構造はよく似ています。

```python
from openai import OpenAI

client = OpenAI()  # OPENAI_API_KEY 環境変数を自動読み取り

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "あなたは親切なアシスタントです。"},
        {"role": "user", "content": "Pythonでクイックソートを実装してください"}
    ],
    max_tokens=1024,
    temperature=0.7,
)

print(response.choices[0].message.content)
```

### 主要パラメータ

APIに渡すパラメータの中で、特に重要なものを押さえておきましょう。

| パラメータ | 説明 | 典型的な値 |
|-----------|------|-----------|
| `model` | 使用するモデルID | `claude-sonnet-4-20250514` |
| `max_tokens` | 生成する最大トークン数 | 1024〜4096 |
| `temperature` | ランダム性の制御（0=決定的、高い=多様） | 0.0〜1.0 |
| `top_p` | 上位確率のトークンのみ選択 | 0.9〜1.0 |
| `system` | システムプロンプト（振る舞いの指示） | 任意のテキスト |

`temperature` は用途に応じて使い分けます。

- **0.0**: コード生成、分類、データ抽出など正確性が重要な場合
- **0.3〜0.7**: 一般的な質問応答、要約
- **0.8〜1.0**: 創作、ブレインストーミングなど多様性が欲しい場合

### ストリーミング

長い応答を待たせないために、生成されたトークンを逐次受け取る **ストリーミング** が実務では重要です。

```python
from anthropic import Anthropic

client = Anthropic()

with client.messages.stream(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[{"role": "user", "content": "AIエージェントの仕組みを説明してください"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

## コスト感覚

### 料金体系の基本

LLMのAPI料金は **トークン単位の従量課金** です。入力と出力で単価が異なり、通常は出力の方が高額です。

| モデル | 入力（$/1Mトークン） | 出力（$/1Mトークン） |
|--------|---------------------|---------------------|
| Claude Opus 4 | $15.00 | $75.00 |
| Claude Sonnet 4 | $3.00 | $15.00 |
| Claude Haiku 3.5 | $0.80 | $4.00 |
| GPT-4o | $2.50 | $10.00 |
| GPT-4o mini | $0.15 | $0.60 |
| Gemini 2.5 Pro | $1.25〜 | $10.00〜 |

### 具体的なコスト計算

実際にどれくらいかかるのか、イメージを掴みましょう。

```python
def estimate_cost(input_tokens: int, output_tokens: int,
                  model: str = "claude-sonnet-4") -> float:
    """API呼び出し1回あたりのコスト見積もり"""
    pricing = {
        "claude-opus-4":    {"input": 15.00, "output": 75.00},
        "claude-sonnet-4":  {"input": 3.00,  "output": 15.00},
        "claude-haiku-3.5": {"input": 0.80,  "output": 4.00},
        "gpt-4o":           {"input": 2.50,  "output": 10.00},
        "gpt-4o-mini":      {"input": 0.15,  "output": 0.60},
    }
    p = pricing[model]
    cost = (input_tokens / 1_000_000 * p["input"]
            + output_tokens / 1_000_000 * p["output"])
    return cost

# 例: 1回の問い合わせ（入力500トークン、出力1000トークン）
cost = estimate_cost(500, 1000, "claude-sonnet-4")
print(f"1回の呼び出し: ${cost:.4f}")  # 約 $0.0165

# 例: 1日1000回呼び出した場合の月額
monthly = cost * 1000 * 30
print(f"月額見積もり: ${monthly:.2f}")  # 約 $495
```

### コスト最適化のポイント

実務でコストを抑えるためのテクニックをいくつか紹介します。

1. **モデルの使い分け**: 簡単なタスクにはHaikuやminiを使い、複雑なタスクだけ上位モデルを使う
2. **Prompt Caching**: 同じシステムプロンプトを繰り返す場合、キャッシュを活用すると入力コストが最大90%削減
3. **プロンプトの最適化**: 冗長な指示を削り、必要最小限の入力に絞る
4. **`max_tokens`の適切な設定**: 不必要に大きな値を設定しない
5. **バッチAPI**: 即時性が不要ならバッチAPIで50%割引（Anthropic/OpenAIとも提供）

迷ったら「まずHaikuで試して、精度が足りなければSonnetに上げる」というアプローチがおすすめです。

## まとめ

| 項目 | 要点 |
|------|------|
| Transformer | Self-Attentionで全トークン間の関連度を同時計算。LLMの基盤技術 |
| 次トークン予測 | LLMの学習原理。「次に来る単語を当てる」をスケールさせたもの |
| トークナイゼーション | テキストをサブワード単位に分割。BPE（Byte Pair Encoding）が主流 |
| 日本語のトークン効率 | 英語の1.5〜2倍のトークンを消費。コスト見積もりに注意 |
| Claude | 200Kコンテキスト、ツール使用に強い。エージェント開発に最適 |
| GPT | エコシステムが充実。o1/o3は推論特化 |
| Gemini | 100万トークンの超長文対応。Google Cloudとの統合が強み |
| API利用 | Messages/Chat Completions形式。temperature等で出力を制御 |
| コスト感覚 | トークン単位の従量課金。入力より出力が高額 |
| コスト最適化 | モデルの使い分け、Prompt Caching、バッチAPIの活用が鍵 |

## やってみよう！

以下の実践課題に取り組んで、この章の内容を手を動かして確認しましょう。

### 課題1: トークン数を体感する

`tiktoken` を使って、同じ内容の日本語テキストと英語テキストのトークン数を比較してみましょう。

```bash
pip install tiktoken
```

```python
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o")

texts = {
    "英語": "Artificial intelligence is transforming the way we work and live.",
    "日本語": "人工知能は私たちの働き方と生活を変革しています。",
    "コード": "def fibonacci(n): return n if n <= 1 else fibonacci(n-1) + fibonacci(n-2)",
}

for label, text in texts.items():
    tokens = enc.encode(text)
    print(f"{label}: {len(tokens)}トークン / {len(text)}文字 "
          f"(1トークンあたり{len(text)/len(tokens):.1f}文字)")
```

### 課題2: APIを呼び出してみる

Anthropic APIキーを取得して、実際にClaude APIを呼び出してみましょう。

1. [Anthropic Console](https://console.anthropic.com/) でアカウントを作成します
2. APIキーを発行します
3. 以下のコードを実行します

```bash
pip install anthropic
export ANTHROPIC_API_KEY="your-api-key-here"
```

```python
from anthropic import Anthropic

client = Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=256,
    messages=[
        {"role": "user", "content": "「LLM」を小学生にもわかるように1文で説明してください"}
    ]
)

print(response.content[0].text)
print(f"\n--- 使用トークン ---")
print(f"入力: {response.usage.input_tokens} トークン")
print(f"出力: {response.usage.output_tokens} トークン")
```

### 課題3: コストを見積もる

あなたが作りたいエージェントの1回の呼び出しにおける入出力トークン数を見積もり、想定の呼び出し回数から月額コストを計算してみましょう。どのモデルが最適か検討してみてください。
