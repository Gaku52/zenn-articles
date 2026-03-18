---
title: "RAG（検索拡張生成）"
---

## RAG とは何か

**RAG（Retrieval-Augmented Generation）** とは、LLM が回答を生成する際に、外部の知識ベースから関連情報を検索して注入する手法です。LLM 単体ではカバーできない最新情報や社内ドキュメントの内容を、回答に組み込むことができます。

RAG が解決する課題は大きく3つあります。

1. **ハルシネーションの低減** -- 検索した事実に基づいて回答するため、でたらめな回答が減ります
2. **知識の鮮度** -- 学習データのカットオフに依存せず、最新情報を参照できます
3. **ドメイン特化** -- 社内文書や専門資料など、LLM が学習していない知識を扱えます

RAG は「LLM に外部記憶を持たせる仕組み」と捉えると理解しやすいでしょう。ファインチューニングがモデルそのものを書き換えるのに対し、RAG はモデルを変えずに知識だけを差し替えられる点が大きな利点です。

---

## RAG アーキテクチャの全体像

RAG パイプラインは、**インデックス構築（オフライン）** と **検索・生成（オンライン）** の2段階で構成されます。

```
【インデックス構築】
文書群 → チャンク分割 → Embedding → ベクトルDB格納

【検索・生成】
ユーザークエリ → Embedding → ベクトル検索 → 関連チャンク取得
                                                  ↓
              ユーザー質問 + 検索結果コンテキスト → LLM → 回答生成
```

### 各フェーズの役割

| フェーズ | 処理内容 | ポイント |
|---------|---------|---------|
| チャンク分割 | 文書を検索可能な単位に分割 | サイズとオーバーラップの調整が品質を左右します |
| Embedding | テキストを数値ベクトルに変換 | インデックス時と検索時で同じモデルを使用します |
| ベクトル検索 | クエリに意味的に近いチャンクを取得 | top-k の値で精度と速度のバランスを取ります |
| 生成 | 検索結果をコンテキストとして LLM に渡す | プロンプトで「コンテキスト外の情報を使わない」よう指示します |

---

## Embedding 生成

Embedding とは、テキストを高次元のベクトル（数値の配列）に変換する処理です。意味が近いテキストほどベクトルが近くなるという性質を利用して、検索を実現します。

### 主要な Embedding モデル

| モデル | 次元数 | 日本語 | 料金（$/1M tokens） | 特徴 |
|--------|-------|--------|-------------------|------|
| text-embedding-3-large | 3072 | 良 | $0.13 | OpenAI 最高精度 |
| text-embedding-3-small | 1536 | 中 | $0.02 | コスパ重視 |
| Cohere embed-v3 | 1024 | 優 | $0.10 | 多言語に強い |
| Voyage-3 | 1024 | 良 | $0.06 | MTEB 上位 |
| BGE-M3（OSS） | 1024 | 優 | 無料 | ローカル実行可 |

### Embedding 生成の実装例

```python
from openai import OpenAI
client = OpenAI()

def embed_text(text: str) -> list[float]:
    """テキストをベクトルに変換します"""
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text,
    )
    return response.data[0].embedding

query_vector = embed_text("RAGの仕組みを教えてください")
doc_vector = embed_text("RAGは検索拡張生成の略で...")
```

### コサイン類似度

2つのベクトルの類似度を測る指標として、コサイン類似度が最もよく使われます。値が 1 に近いほど意味が近く、0 に近いほど無関係です。

```python
import numpy as np

def cosine_similarity(a: list[float], b: list[float]) -> float:
    a, b = np.array(a), np.array(b)
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))
```

:::message
**Embedding モデル選定の鉄則**: インデックス構築時と検索時で必ず同じモデル・同じバージョンを使ってください。異なるモデルのベクトル空間は互換性がないため、検索結果が無意味になります。
:::

---

## ベクトル検索

ベクトル検索では、クエリの Embedding と最もコサイン類似度が高いチャンクを上位 k 件取得します。この処理を効率的に行うのがベクトルデータベースです。

### ベクトル DB の比較

| ベクトル DB | 形態 | 特徴 | 適したケース |
|------------|------|------|------------|
| Pinecone | マネージド | 運用不要、スケーラブル | 本番環境で素早く始めたい場合 |
| Qdrant | OSS / Cloud | 高機能フィルタリング | 柔軟な検索条件が必要な場合 |
| pgvector | PostgreSQL 拡張 | 既存 RDB に統合可能 | PostgreSQL を既に使っている場合 |
| Chroma | OSS | 軽量、開発向き | プロトタイピング |

### 基本的な RAG 実装

```python
from openai import OpenAI
from qdrant_client import QdrantClient
from qdrant_client.models import PointStruct, VectorParams, Distance

client = OpenAI()
qdrant = QdrantClient(":memory:")

qdrant.create_collection(
    collection_name="docs",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
)

def index_documents(documents: list[dict]):
    """ドキュメントをベクトル化してDBに格納します"""
    points = []
    for i, doc in enumerate(documents):
        embedding = client.embeddings.create(
            model="text-embedding-3-small", input=doc["text"],
        ).data[0].embedding
        points.append(PointStruct(
            id=i, vector=embedding,
            payload={"text": doc["text"], "source": doc["source"]},
        ))
    qdrant.upsert(collection_name="docs", points=points)

def rag_query(query: str, top_k: int = 5) -> str:
    """検索結果をもとにLLMで回答を生成します"""
    query_vector = client.embeddings.create(
        model="text-embedding-3-small", input=query,
    ).data[0].embedding

    results = qdrant.search(
        collection_name="docs", query_vector=query_vector, limit=top_k,
    )
    context = "\n\n".join([
        f"[出典: {r.payload['source']}]\n{r.payload['text']}"
        for r in results
    ])

    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": (
                "提供されたコンテキストのみに基づいて回答してください。"
                "情報が不足する場合は「情報が見つかりません」と回答してください。"
                "回答には出典を引用してください。"
            )},
            {"role": "user", "content": f"コンテキスト:\n{context}\n\n質問: {query}"},
        ],
    )
    return response.choices[0].message.content
```

---

## チャンク分割戦略

チャンク分割は RAG の品質を大きく左右する工程です。適切に分割しないと、検索精度が下がったり、LLM に必要な文脈が届かなかったりします。

### 主な分割手法

| 手法 | 仕組み | 長所 | 短所 |
|------|--------|------|------|
| 固定長分割 | 文字数やトークン数で機械的に分割 | 実装が簡単 | 文の途中で切れる |
| 再帰的分割 | 段落→行→文→単語の順で階層的に分割 | 文構造を保持しやすい | パラメータ調整が必要 |
| セマンティック分割 | Embedding の類似度変化点で分割 | 意味の一貫性が高い | 計算コストが高い |
| 構造ベース分割 | Markdown ヘッダーや HTML タグで分割 | 文書構造を尊重 | 構造化文書に限定 |

### 再帰的分割の実装例（推奨）

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
    separators=["\n\n", "\n", "。", "、", " ", ""],
)
chunks = splitter.split_text(document)
```

### チャンクサイズの目安

| パラメータ | 推奨範囲 | 小さすぎる場合 | 大きすぎる場合 |
|-----------|---------|--------------|--------------|
| chunk_size | 256-1024 tokens | 文脈が不足します | ノイズが混入します |
| chunk_overlap | サイズの10-20% | 境界の情報が欠落します | 重複でコストが増えます |
| top_k | 3-10 | 情報が不足します | コンテキスト長を消費します |

:::message alert
**よくあるミス**: チャンクサイズを 4000 文字のように大きくすると無関係な情報が混入し、50 文字のように小さすぎると文脈が失われます。500 文字前後から始めて評価しながら調整してください。
:::

### Parent-Child チャンキング

検索精度と文脈保持を両立するテクニックです。検索には小さなチャンク（Child）を使い、LLM への入力には大きなチャンク（Parent）を渡します。

```python
# Parent: 大きなチャンク（文脈保持用、2000文字）
# Child: 小さなチャンク（検索精度向上用、400文字）
# → Childで精密検索し、対応するParentチャンクをLLMに入力
```

---

## Naive RAG vs Advanced RAG

RAG には成熟度によっていくつかのレベルがあります。プロジェクトの段階に応じて適切なレベルを選択してください。

### Naive RAG（Level 1）

最もシンプルな RAG です。ベクトル検索の結果をそのまま LLM に渡して回答を生成します。

- 固定サイズのチャンク分割、単純なベクトル検索（top-k）
- 課題: キーワード完全一致に弱い、検索結果にノイズが混入しやすい

### Advanced RAG（Level 2）

Naive RAG の課題を解決するために、検索の前後に処理を追加した手法です。

1. **クエリ変換（Pre-Retrieval）**: ユーザーのクエリを検索に適した形に変換します
2. **ハイブリッド検索**: ベクトル検索とキーワード検索を組み合わせます
3. **リランキング（Post-Retrieval）**: 検索結果を再スコアリングして精度を上げます
4. **メタデータフィルタリング**: 日付や部署などの属性で検索範囲を絞ります

### 各レベルの比較

| 項目 | Naive RAG | Advanced RAG | Agentic RAG |
|------|-----------|-------------|-------------|
| 検索方式 | ベクトル検索のみ | ハイブリッド検索+リランキング | 自律的な検索判断 |
| クエリ処理 | そのまま使用 | 変換・拡張 | マルチステップ推論 |
| 精度 | 基準 | +10-25% 向上 | +20-30% 向上 |
| 実装コスト | 低 | 中 | 高 |
| レイテンシ | 低 | 中 | 高 |
| 適用場面 | MVP / 概念検証 | 本番環境 | 複雑な質問対応 |

---

## ハイブリッド検索

ベクトル検索は意味的な類似度に強い一方で、固有名詞や型番のような完全一致が必要なケースには弱い傾向があります。ベクトル検索とキーワード検索（BM25）を組み合わせることで、この弱点を補います。

### RRF（Reciprocal Rank Fusion）によるスコア統合

```python
def hybrid_search(query: str, top_k: int = 10) -> list:
    """ベクトル検索とBM25の結果をRRFで統合します"""
    vector_results = vector_search(query, top_k=top_k)
    keyword_results = bm25_search(query, top_k=top_k)

    rrf_scores = {}
    k = 60  # RRF定数

    for rank, result in enumerate(vector_results):
        rrf_scores[result.id] = rrf_scores.get(result.id, 0) + 1/(k+rank+1)
    for rank, result in enumerate(keyword_results):
        rrf_scores[result["id"]] = rrf_scores.get(result["id"], 0) + 1/(k+rank+1)

    sorted_ids = sorted(rrf_scores, key=rrf_scores.get, reverse=True)
    return sorted_ids[:top_k]
```

RRF は各検索結果の順位の逆数を合算してスコアとする手法です。両方の検索で上位に来るドキュメントが高スコアになります。

### リランキング

初期検索で広く候補を取得し（top-50）、リランカーで精密に絞り込みます（top-5）。検索精度を +10-25% 向上させる効果があります。

```python
import cohere
co = cohere.Client("YOUR_API_KEY")

def rerank(query: str, documents: list[str], top_n: int = 5) -> list:
    """Cohere Rerank で検索結果を再スコアリングします"""
    response = co.rerank(
        model="rerank-multilingual-v3.0",
        query=query, documents=documents, top_n=top_n,
    )
    return [{"index": r.index, "score": r.relevance_score} for r in response.results]
```

| リランカー | 精度 | 速度 | コスト | 日本語対応 |
|-----------|------|------|--------|----------|
| Cohere Rerank v3 | 高 | 高速 | 有料 | 優秀 |
| Cross-Encoder（OSS） | 中〜高 | 中 | 無料 | 限定的 |
| BGE-Reranker-v2 | 高 | 中 | 無料 | 良好 |
| LLM Rerank（GPT-4o-mini） | 最高 | 低速 | 中 | 優秀 |

---

## 高度なクエリ変換テクニック

### Multi-Query

ユーザーのクエリを LLM で複数の異なる観点に分解して検索し、結果を統合します。曖昧なクエリに対して特に有効です。元のクエリを3パターンに言い換え、それぞれで検索した結果をリランキングで統合します。

### HyDE（Hypothetical Document Embedding）

クエリに対する「仮想的な回答文書」を LLM で生成し、その文書のベクトルで検索します。クエリと文書の語彙ギャップを埋めるのに効果的です。専門用語が多いドメインや、短いクエリで情報量が少ない場合に威力を発揮します。

---

## メタデータの活用

チャンクにメタデータ（出典、日時、カテゴリなど）を付与することで、検索時にフィルタリングが可能になります。

```python
chunk_with_metadata = {
    "text": "有給休暇は入社6ヶ月後から付与されます...",
    "metadata": {
        "source": "hr_manual.pdf",
        "department": "HR",
        "document_type": "policy",
        "created_at": "2026-01-15",
    }
}

# 人事部のドキュメントだけに絞って検索
results = qdrant.search(
    collection_name="docs",
    query_vector=query_vector,
    query_filter=Filter(must=[
        FieldCondition(key="department", match=MatchValue(value="HR")),
    ]),
    limit=5,
)
```

---

## RAG 構築時のアンチパターン

### 1. プロンプト指示の不備

```python
# NG: 指示が曖昧 → LLMが自身の知識で回答しハルシネーション発生
prompt = f"質問: {query}\n参考: {context}\n回答:"

# OK: 明確なガードレール付き
prompt = f"""提供されたコンテキストのみに基づいて回答してください。
コンテキストに含まれない情報で推測しないでください。
情報が不足する場合は「この情報は見つかりません」と明示してください。
回答には出典を含めてください。

コンテキスト:
{context}

質問: {query}"""
```

### 2. 検索結果の無検証利用

検索結果のスコアを確認せずに全件を LLM に渡すと、無関係な情報がノイズになります。スコアの閾値を設けるか、リランキングを挟んでください。

### 3. チャンクサイズの不適切な設定

まずは 256-512 トークン、オーバーラップ 10-20% から始めて、評価データで検証しながら調整するのが堅実です。

---

## まとめ

| 項目 | 推奨 |
|------|------|
| チャンク分割 | RecursiveCharacterTextSplitter（512 tokens, 12% overlap） |
| Embedding モデル | text-embedding-3-small（コスパ）/ BGE-M3（日本語+OSS） |
| ベクトル DB | Qdrant / pgvector（自前）/ Pinecone（マネージド） |
| 検索方式 | ハイブリッド検索（ベクトル + BM25） |
| リランキング | Cohere Rerank v3 / Cross-Encoder |
| クエリ変換 | Multi-Query / HyDE（精度要求が高い場合） |
| メタデータ | 出典・日時・カテゴリを必ず付与 |
| 評価指標 | Faithfulness, Relevancy, Answer Correctness |
| 開始レベル | Naive RAG で検証 → Advanced RAG で改善 |

---

## やってみよう！

以下のステップで、手元で RAG パイプラインを構築してみましょう。

- [ ] **Step 1**: 3〜5件のテキストドキュメントを用意し、`RecursiveCharacterTextSplitter` で分割してみましょう。chunk_size を 200 / 500 / 1000 に変えて違いを確認してください
- [ ] **Step 2**: OpenAI の `text-embedding-3-small` でチャンクをベクトル化し、Qdrant（`:memory:` モード）に格納してみましょう
- [ ] **Step 3**: 本章の「基本的な RAG 実装」を動かし、質問に対して文書ベースの回答が返ることを確認してください
- [ ] **Step 4**: BM25 によるキーワード検索を追加し、RRF でハイブリッド検索を実装してみましょう
- [ ] **Step 5**: Cohere Rerank でリランキングを追加し、top-50 から top-5 に絞り込む効果を測定してみましょう
