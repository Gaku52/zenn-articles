---
title: "READMEの基本構成"
---

# READMEの基本構成

## この章で学ぶこと

READMEは、プロジェクトの「玄関」です。初めて訪れた人が最初に目にするドキュメントであり、プロジェクトの第一印象を決定づけます。

この章では、以下の内容を学びます：

- READMEの目的と重要性
- 必須セクションと任意セクション
- 読者の目的別に情報を整理する方法
- Markdownの効果的な活用方法
- バッジ、目次、リンクの使い方

**この章の重要な原則：**

1. **読者ファースト**: 読者が求める情報を素早く見つけられる構成
2. **段階的開示**: 概要から詳細へ、段階的に情報を提供
3. **一貫性**: セクション構成やスタイルの統一
4. **保守性**: 更新しやすく、古くなりにくい構成

---

## READMEの目的

### なぜREADMEが必要なのか

READMEは、プロジェクトに関する最も基本的な情報を提供するドキュメントです。

**READMEの主な目的：**

1. **プロジェクトの概要を伝える**: 何を、なぜ、誰のために作ったのか
2. **始め方を示す**: インストール、セットアップ、基本的な使い方
3. **価値を伝える**: なぜこのプロジェクトを使うべきか
4. **次のステップを示す**: 詳細ドキュメント、サンプル、コミュニティへの導線

### READMEの読者

READMEには、様々な目的を持った読者が訪れます。それぞれの読者が求める情報を理解しましょう。

#### 1. 検討者（Evaluator）

**目的**: このプロジェクトを使うべきか判断したい

**求める情報:**
- プロジェクトの概要（1分で理解できる説明）
- 主要な機能
- ライセンス
- 活発に開発されているか（最終更新日、スター数など）

**行動パターン:**
```
1. タイトルとバッジを見る（5秒）
2. 概要説明を読む（30秒）
3. スクリーンショット・図を見る（30秒）
4. 機能一覧をスキャン（30秒）
5. ライセンスを確認（5秒）
→ 合計2分で判断
```

#### 2. 導入者（Implementer）

**目的**: 実際にプロジェクトを使い始めたい

**求める情報:**
- インストール方法
- 前提条件（Node.js、Python、OSなど）
- 基本的な使い方
- サンプルコード

**行動パターン:**
```
1. インストールセクションに直接ジャンプ
2. コマンドをコピー＆ペースト
3. 基本的な使い方を試す
4. うまく動かない場合、トラブルシューティングを探す
```

#### 3. コントリビューター（Contributor）

**目的**: プロジェクトに貢献したい

**求める情報:**
- 開発環境のセットアップ方法
- コントリビューションガイドライン
- 行動規範
- テストの実行方法

**行動パターン:**
```
1. Contributingセクションを探す
2. 開発環境のセットアップ手順を実行
3. テストを実行して環境を確認
4. Issue、PR、ディスカッションを確認
```

#### 4. メンテナー（Maintainer）

**目的**: プロジェクトの全体像を把握したい

**求める情報:**
- アーキテクチャ概要
- デプロイ方法
- リリースプロセス
- ドキュメントの場所

**行動パターン:**
```
1. 目次から必要なセクションにジャンプ
2. 詳細ドキュメントへのリンクを辿る
3. 設計判断の記録（ADR）などを確認
```

---

## READMEの基本構成

### 必須セクション

すべてのREADMEに含めるべき基本的なセクションです。

#### 1. タイトルと概要説明

プロジェクトの名前と、1文で何をするプロジェクトかを説明します。

```markdown
# プロジェクト名

> 1文での説明（What this project does）
```

**良い例:**

```markdown
# Claude Code Skills

> A comprehensive collection of mathematically rigorous algorithm proofs,
> distributed systems theory, and formal verification, achieving MIT
> master's thesis level standards.
```

**ポイント:**
- プロジェクト名は大きく目立つように
- 1文の説明は、**What（何をするか）** を明確に
- 長すぎない（1-2行）

**悪い例:**

```markdown
❌ 悪い例:
# MyApp

このアプリは便利です。
```

**問題点:**
- 何が便利なのかわからない
- 何をするアプリなのか不明
- 読者に何の価値も伝えていない

#### 2. バッジ（Badges）

プロジェクトのステータスや品質指標を視覚的に示します。

**一般的なバッジ:**

```markdown
![Build Status](https://img.shields.io/github/actions/workflow/status/user/repo/ci.yml?branch=main)
![Coverage](https://img.shields.io/codecov/c/github/user/repo)
![Version](https://img.shields.io/npm/v/package-name)
![License](https://img.shields.io/badge/License-MIT-yellow.svg)
```

**実例（claude-code-skills）:**

```markdown
![Progress](https://img.shields.io/badge/Progress-100%25-green)
![Skills](https://img.shields.io/badge/Skills-25%2F25-blue)
![Characters](https://img.shields.io/badge/Characters-3485K-informational)
![Guides](https://img.shields.io/badge/Guides-86-success)

[![MIT Master's Level](https://img.shields.io/badge/MIT%20Level-90%2F100-success)](https://github.com/Gaku52/claude-code-skills)
[![Theoretical Rigor](https://img.shields.io/badge/Theoretical%20Rigor-20%2F20-brightgreen)](#theoretical-rigor)
[![Reproducibility](https://img.shields.io/badge/Reproducibility-20%2F20-brightgreen)](#reproducibility)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
```

**バッジの選び方:**

| バッジ | 目的 | 推奨度 |
|--------|------|--------|
| Build Status | CI/CDの状態を示す | ★★★★★ |
| Coverage | テストカバレッジ | ★★★★☆ |
| Version | 最新バージョン（npm、PyPIなど） | ★★★★★ |
| License | ライセンス | ★★★★★ |
| Downloads | 人気度 | ★★★☆☆ |
| Dependencies | 依存関係の状態 | ★★★☆☆ |

**注意:**
- バッジは多すぎると煩雑になる（5-10個程度）
- 更新されないバッジは信頼性を損なう
- カスタムバッジは [shields.io](https://shields.io/) で作成可能

#### 3. プロジェクト概要（Overview/About）

プロジェクトの詳細な説明です。What、Why、Whoを明確にします。

**構成:**

```markdown
## 📖 Project Overview

**What（何を）:**
このプロジェクトの内容と主要な機能

**Why（なぜ）:**
このプロジェクトを作った理由、解決する問題

**Who（誰のために）:**
対象ユーザー、ターゲット層
```

**実例（claude-code-skills）:**

```markdown
## 🎯 Project Overview

This repository contains **34 complete mathematical proofs** with
**255+ peer-reviewed paper citations**, covering:
- **25 Algorithm Proofs**: Data structures, sorting, graphs, string matching
- **5 Distributed Systems Proofs**: CAP theorem, Paxos, Raft, 2PC/3PC, CRDT
- **3 TLA+ Formal Specifications**: Model checking with 152,500+ verified states
- **Statistical Rigor**: All experiments with n≥30, p<0.001, R²>0.999

**Current Score**: **90/100 points** (MIT+ Level) ✅
```

**ポイント:**
- **具体的な数値**: 34 proofs、255+ papers（実際のプロジェクトの実測値）
- **階層構造**: 箇条書きで見やすく
- **強調**: 重要な部分は太字
- **絵文字**: 適度に使用（視覚的な区別）

#### 4. 目次（Table of Contents）

長いREADMEでは、目次が必須です。

```markdown
## 📚 Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Features](#features)
- [Usage](#usage)
- [API Documentation](#api-documentation)
- [Contributing](#contributing)
- [License](#license)
```

**自動生成ツール:**

VS Codeの拡張機能や、以下のようなツールで自動生成できます：

- [markdown-toc](https://github.com/jonschlinkert/markdown-toc) (Node.js)
- GitHub自動生成（Wiki）

**注意:**
- アンカーリンクはGitHub Markdown形式に準拠
- セクション名の変更時は目次も更新

#### 5. インストール（Installation）

プロジェクトのインストール方法を明確に示します。

**基本構成:**

```markdown
## 🚀 Installation

### Prerequisites

- Node.js 18.0.0 or higher
- npm 9.0.0 or higher

### Install

```bash
npm install package-name
```

または

```bash
yarn add package-name
```
```

**実例（npmパッケージ）:**

```markdown
## 🚀 Quick Start

### npm Packages

```bash
# Statistical Analysis Library
npm install @claude-code-skills/stats

# CRDT Library
npm install @claude-code-skills/crdt
```

**Statistics Example:**
```typescript
import { pairedTTest, runBeforeAfterExperiment } from '@claude-code-skills/stats';

const before = [12.5, 13.2, 11.8, 14.1, 12.9];
const after = [4.8, 5.2, 4.5, 5.5, 4.9];
const result = pairedTTest(before, after);

console.log(`p-value: ${result.p < 0.001 ? '<0.001' : result.p.toFixed(3)}`);
console.log(`Cohen's d: ${result.d.toFixed(2)}`);
```
```

**ポイント:**
- **前提条件を明記**: バージョン、OS、その他の依存関係
- **コピペ可能**: そのまま実行できるコマンド
- **複数の方法**: npm、yarn、pnpmなど
- **動作確認**: インストール後の確認方法も記載

#### 6. 基本的な使い方（Usage/Getting Started）

最小限のコードで、プロジェクトの使い方を示します。

**原則:**
- **5分で試せる**: 複雑すぎない
- **動作するコード**: コピペで動く
- **段階的**: 最小限の例 → 詳細な例

```markdown
## 📖 Usage

### Basic Example

```typescript
import { createApp } from 'package-name';

const app = createApp({
  port: 3000
});

app.start();
```

### Advanced Example

より詳細な例は [examples/](examples/) を参照してください。
```

#### 7. ライセンス（License）

プロジェクトのライセンスを明記します。

```markdown
## 📜 License

MIT License - See [LICENSE](LICENSE) file for details.
```

**注意:**
- ライセンスファイル（LICENSE）も必ず作成
- OSSでは一般的なライセンス（MIT、Apache 2.0、GPLなど）を使用
- ライセンスを明記しないと、誰も使えない

---

### 推奨セクション

必須ではないが、プロジェクトの性質に応じて含めるべきセクションです。

#### 1. 主要機能（Features）

プロジェクトの主要な機能を箇条書きで示します。

```markdown
## 🌟 Key Features

- ✅ Feature 1: Description
- ✅ Feature 2: Description
- ✅ Feature 3: Description
```

**実例（claude-code-skills）:**

```markdown
## 🌟 Key Features

### 1. Mathematical Rigor

**Every proof includes**:
- ✅ Complete mathematical proof (induction, contradiction, loop invariants)
- ✅ Time/space complexity analysis with Master theorem
- ✅ TypeScript/Swift implementation
- ✅ Performance measurements (n≥30, 95% CI, p<0.001)
- ✅ 4-6 peer-reviewed papers per proof

**Example**: Binary Search achieves **4,027× speedup** with R²=0.9997
theoretical validation.
```

**ポイント:**
- チェックマーク（✅）で視覚的にわかりやすく
- 具体的な数値を示す（実際のプロジェクトの実測値）
- 階層構造で整理

#### 2. ディレクトリ構成（Directory Structure）

プロジェクトのファイル構成を示します。

```markdown
## 📂 Directory Structure

```
project/
├── src/               # Source code
│   ├── api/          # API endpoints
│   ├── models/       # Data models
│   └── utils/        # Utility functions
├── tests/            # Test files
├── docs/             # Documentation
├── examples/         # Example code
└── README.md
```
```

**実例（claude-code-skills）:**

```markdown
## 📚 Repository Structure

```
claude-code-skills/
├── backend-development/
│   └── guides/algorithms/           # 25 algorithm proofs
│       ├── binary-search-proof.md   # 4,027× speedup
│       ├── fft-proof.md             # 852× speedup
│       └── ...
│
├── _IMPROVEMENTS/
│   ├── phase1/                      # Statistical rigor (4 skills)
│   ├── phase2/                      # 25 algorithm proofs
│   └── phase3/
│       ├── distributed-systems/     # 5 distributed proofs
│       ├── tla-plus/                # 3 TLA+ specifications
│       └── experiment-templates/    # Statistical templates
│
├── packages/                        # npm packages
│   ├── stats/                       # Statistical analysis library ✅
│   └── crdt/                        # CRDT implementations ✅
│
└── demos/                           # Interactive demos ✅
    ├── stats-playground/            # Statistical analysis tool ✅
    └── crdt-demo/                   # CRDT interactive demo ✅
```
```

**ポイント:**
- 主要なディレクトリのみ記載（全てのファイルを列挙しない）
- コメントで役割を説明
- インデントで階層を表現

#### 3. パフォーマンス・ベンチマーク

性能が重要なプロジェクトでは、ベンチマーク結果を示します。

**注意: 誠実性の原則**

ベンチマーク結果を記載する場合、以下の原則を守ります：

```markdown
## 📈 Performance Benchmarks

### Algorithm Performance

以下は、claude-code-skillsプロジェクトで実際に測定された結果です：

| Algorithm | Speedup | p-value | Effect Size | R² |
|-----------|---------|---------|-------------|-----|
| FFT | **852×** | <0.001 | d=30.9 | 0.9997 |
| Binary Search | **4,027×** | <0.001 | d=67.3 | 0.9997 |
| Fenwick Tree | **1,736×** | <0.001 | d=51.6 | 0.9998 |

**測定環境:**
- CPU: Apple M2 Pro
- Memory: 32GB
- Sample size: n=30
- Statistical test: Paired t-test
```

**ポイント:**
- **実際のプロジェクトの実測値のみ記載**
- **測定環境を明記**: CPU、メモリ、OS
- **統計的根拠**: サンプルサイズ、有意水準など
- **再現可能性**: 誰が実行しても同様の結果が得られる

**誠実な書き方:**

```markdown
✅ 良い例（公式ベンチマークの引用）:

Fastify公式ベンチマーク（https://fastify.dev/benchmarks/）によると：
- Fastify: 78,513 req/sec
- Express: 14,200 req/sec

※ 環境や条件により結果は異なります
```

```markdown
✅ 良い例（理論的な説明）:

Binary Searchは、線形探索と比較して理論的に以下のような性能特性があります：
- 時間計算量: O(log n) vs O(n)
- 大規模データ（n=100万）の場合、約20倍の性能改善が期待できます

実際の性能は、データの分布やキャッシュ効率により異なります。
```

```markdown
❌ 悪い例（架空の数値）:

このライブラリを使うと、レスポンスタイムが3.2秒から0.8秒に改善します。
（※ 実測していないのに具体的な数値を記載している）
```

#### 4. スクリーンショット・デモ

視覚的な要素がある場合、スクリーンショットやGIFを含めます。

```markdown
## 🎮 Demo

![Demo Animation](./assets/demo.gif)

**Try it live**: [https://example.com/demo](https://example.com/demo)
```

**実例（claude-code-skills）:**

```markdown
## 🎮 Interactive Demos

**Try it live**: [https://gaku52.github.io/claude-code-skills/](https://gaku52.github.io/claude-code-skills/)

- **Statistics Playground**: Calculate t-tests, confidence intervals, and effect sizes in your browser
- **CRDT Demo**: Experience distributed data types with strong eventual consistency
```

**ポイント:**
- 画像は相対パスで指定
- 画像ファイルはリポジトリに含める（外部URLは避ける）
- GIFは3MB以下に圧縮
- 代替テキスト（alt text）を記載

#### 5. コントリビューション（Contributing）

OSSプロジェクトでは、貢献方法を明記します。

```markdown
## 🤝 Contributing

Contributions are welcome! Please read our [Contributing Guide](CONTRIBUTING.md) for details.

### Development Setup

```bash
# Clone the repository
git clone https://github.com/user/repo.git

# Install dependencies
npm install

# Run tests
npm test

# Build
npm run build
```

### Pull Request Process

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request
```

**ポイント:**
- 詳細はCONTRIBUTING.mdに分離
- 開発環境のセットアップ手順
- PR作成の手順

#### 6. トラブルシューティング（Troubleshooting）

よくある問題と解決策を記載します。

```markdown
## 🔧 Troubleshooting

### Common Issues

#### Installation fails with "permission denied"

**Solution:**
```bash
# Use sudo (macOS/Linux)
sudo npm install -g package-name

# Or use npx
npx package-name
```

#### Port already in use

**Solution:**
Change the port in `config.json`:
```json
{
  "port": 3001
}
```
```

#### 7. FAQ（よくある質問）

```markdown
## ❓ FAQ

### Q: このプロジェクトと〇〇の違いは？

A: 〇〇は△△に特化していますが、このプロジェクトは□□を重視しています。

### Q: 商用利用は可能ですか？

A: はい、MITライセンスのため商用利用可能です。

### Q: サポートはありますか？

A: GitHub Issuesで質問を受け付けています。
```

---

## 読者の目的別構成

READMEは、読者の目的に応じて情報を整理します。

### パターン1: ライブラリ・フレームワーク

**対象**: 開発者がコードで使用するもの

**推奨構成:**

```markdown
# プロジェクト名

> 1文での説明

## Badges

## Features

## Installation

## Quick Start

## API Documentation

## Examples

## Performance

## Contributing

## License
```

**重点:**
- インストール・使い方が最優先
- サンプルコードが豊富
- API仕様書へのリンク

### パターン2: CLIツール

**対象**: コマンドラインで使用するツール

**推奨構成:**

```markdown
# プロジェクト名

> 1文での説明

## Badges

## Installation

## Usage

```bash
# Basic usage
command-name [options]

# Examples
command-name --input file.txt --output result.txt
```

## Options

## Examples

## Troubleshooting

## Contributing

## License
```

**重点:**
- コマンドの使い方を具体的に
- オプション一覧
- 実行例が豊富

### パターン3: Webアプリケーション

**対象**: デプロイして使うアプリケーション

**推奨構成:**

```markdown
# プロジェクト名

> 1文での説明

## Badges

## Features

## Demo

## Screenshots

## Installation

## Configuration

## Deployment

## Architecture

## Contributing

## License
```

**重点:**
- スクリーンショット・デモ
- デプロイ方法
- 設定方法

### パターン4: 研究・学術プロジェクト

**対象**: 論文、証明、実験など

**推奨構成:**

```markdown
# プロジェクト名

> 1文での説明

## Badges

## Abstract

## Key Results

## Methodology

## Experiments

## Citation

## License
```

**実例（claude-code-skills）:**

```markdown
# Claude Code Skills

> A comprehensive collection of mathematically rigorous algorithm proofs

## 🎯 Project Overview

（概要）

## 📊 Quality Metrics

| Metric | Score | Status |
|--------|-------|--------|
| **Theoretical Rigor** | 20/20 | ✅ Perfect |
| **Reproducibility** | 20/20 | ✅ Perfect |

## 🌟 Key Features

### 1. Mathematical Rigor
（数学的厳密性の説明）

### 2. Distributed Systems Theory
（分散システム理論の説明）

## 📈 Highlighted Results

（主要な結果）

## 🔬 Methodology

（方法論）

## 📚 Referenced Papers (255+)

（引用論文）
```

**重点:**
- 研究結果・品質指標
- 方法論の明確化
- 引用文献

---

## Markdownの効果的な使い方

READMEはMarkdownで記述します。効果的な使い方を学びましょう。

### 見出しの階層

見出しは、情報の階層を示します。

```markdown
# h1: プロジェクト名（1つのみ）

## h2: 主要セクション

### h3: サブセクション

#### h4: 詳細セクション
```

**原則:**
- h1はプロジェクト名のみ
- h2が主要セクション（Installation、Usage、Licenseなど）
- h3以降は階層的に使用
- h5、h6は避ける（階層が深すぎる）

### 箇条書き

情報を整理するのに最適です。

```markdown
## 箇条書き

### 順序なしリスト

- Item 1
- Item 2
  - Nested item 2.1
  - Nested item 2.2
- Item 3

### 順序付きリスト

1. First step
2. Second step
3. Third step

### チェックリスト

- [x] Completed task
- [ ] Pending task
- [ ] Future task
```

### コードブロック

コードは言語を指定してシンタックスハイライトを有効化します。

````markdown
```typescript
// TypeScript
function greet(name: string): string {
  return `Hello, ${name}!`;
}
```

```bash
# Shell
npm install package-name
```

```json
{
  "name": "package-name",
  "version": "1.0.0"
}
```
````

**対応言語:**
- `typescript`, `javascript`, `python`, `ruby`, `go`, `rust`, `swift`
- `bash`, `shell`, `sh`
- `json`, `yaml`, `toml`
- `markdown`, `html`, `css`
- `sql`

### 表（Table）

データを整理して表示します。

```markdown
| Header 1 | Header 2 | Header 3 |
|----------|----------|----------|
| Cell 1   | Cell 2   | Cell 3   |
| Cell 4   | Cell 5   | Cell 6   |

## 左寄せ、中央揃え、右寄せ

| Left | Center | Right |
|:-----|:------:|------:|
| A    | B      | C     |
| D    | E      | F     |
```

**実例（性能比較）:**

```markdown
| Algorithm | Speedup | p-value | Effect Size | R² |
|-----------|---------|---------|-------------|-----|
| FFT | **852×** | <0.001 | d=30.9 | 0.9997 |
| Binary Search | **4,027×** | <0.001 | d=67.3 | 0.9997 |
```

### リンク

```markdown
## 内部リンク（セクションへのジャンプ）

[Installation](#installation)

## 外部リンク

[OpenAI](https://openai.com/)

## 画像

![Alt text](./path/to/image.png)

## 画像にリンク

[![Alt text](./badge.png)](https://example.com/)
```

### 引用

```markdown
> **注意:** 重要な注意事項はこのように表示されます。

> 複数行にわたる引用も
> このように記述できます。
```

### 水平線

セクションの区切りに使用します。

```markdown
---
```

### インラインコード

文章中のコードや技術用語を強調します。

```markdown
`npm install` コマンドを実行してください。

この関数は `Promise<User>` を返します。
```

### 絵文字

視覚的な区別に使用します（適度に）。

```markdown
## 🚀 Quick Start
## 📖 Documentation
## 🤝 Contributing
## 📜 License
## ⚠️ Warning
```

**注意:**
- 絵文字は控えめに（各セクション見出しに1つ程度）
- 意味のある絵文字を選ぶ
- 絵文字だけで情報を伝えない

---

## バッジの活用

バッジは、プロジェクトの状態を視覚的に伝えます。

### shields.ioの使い方

[shields.io](https://shields.io/) は、カスタムバッジを作成できるサービスです。

**基本的な形式:**

```
https://img.shields.io/badge/<LABEL>-<MESSAGE>-<COLOR>
```

**例:**

```markdown
![MIT License](https://img.shields.io/badge/License-MIT-yellow.svg)
![Version](https://img.shields.io/badge/Version-1.0.0-blue)
![Status](https://img.shields.io/badge/Status-Stable-green)
```

### GitHub Actionsのバッジ

CIステータスを表示します。

```markdown
![Build](https://github.com/user/repo/actions/workflows/ci.yml/badge.svg)
```

### npm/PyPIのバッジ

パッケージのバージョンやダウンロード数を表示します。

```markdown
![npm version](https://img.shields.io/npm/v/package-name)
![npm downloads](https://img.shields.io/npm/dm/package-name)
![PyPI version](https://img.shields.io/pypi/v/package-name)
```

### カバレッジバッジ

テストカバレッジを表示します。

```markdown
![Coverage](https://img.shields.io/codecov/c/github/user/repo)
```

### カスタムバッジ

プロジェクト固有の指標を表示します。

**実例（claude-code-skills）:**

```markdown
![Progress](https://img.shields.io/badge/Progress-100%25-green)
![Skills](https://img.shields.io/badge/Skills-25%2F25-blue)
![Characters](https://img.shields.io/badge/Characters-3485K-informational)
```

---

## 目次の作成

長いREADMEには目次が必須です。

### 手動での作成

```markdown
## 📚 Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Features](#features)
  - [Feature 1](#feature-1)
  - [Feature 2](#feature-2)
- [API Documentation](#api-documentation)
- [Contributing](#contributing)
- [License](#license)
```

**ポイント:**
- アンカーリンクはセクション名を小文字化し、スペースをハイフンに変換
- 例: `## Quick Start` → `#quick-start`
- 特殊文字は削除: `## API Documentation` → `#api-documentation`

### 自動生成ツール

#### markdown-toc（Node.js）

```bash
npm install -g markdown-toc

# README.mdに目次を自動挿入
markdown-toc -i README.md
```

**使い方:**

README.mdに以下を記述：

```markdown
<!-- toc -->
<!-- tocstop -->
```

コマンド実行後、自動的に目次が生成されます。

#### VS Code拡張機能

**Markdown All in One**

- 自動目次生成
- セクションナンバリング
- リンク検証

インストール後、コマンドパレット（Cmd+Shift+P）から：
```
Markdown All in One: Create Table of Contents
```

---

## 実践的な例

実際のREADMEを分析して、良い構成を学びましょう。

### 例1: claude-code-skills

**特徴:**
- 研究・学術プロジェクト
- 品質指標（Quality Metrics）を前面に
- ナビゲーション重視（INDEX.md、NAVIGATION.mdへのリンク）

**優れている点:**

1. **明確な価値提案**

```markdown
> A comprehensive collection of mathematically rigorous algorithm proofs,
> distributed systems theory, and formal verification, achieving MIT
> master's thesis level standards.
```

→ 1文で「何を」「どのレベルか」が明確

2. **品質指標の可視化**

```markdown
## 📊 Quality Metrics

| Metric | Score | Status |
|--------|-------|--------|
| **Theoretical Rigor** | 20/20 | ✅ Perfect |
| **Reproducibility** | 20/20 | ✅ Perfect |
| **Originality** | 17/20 | ✅ Excellent |
| **Practicality** | 33/40 | ✅ Strong |
| **Total** | **90/100** | **🎓 MIT+ Level** |
```

→ 具体的な評価基準と点数（実際のプロジェクトの自己評価）

3. **ナビゲーションの充実**

```markdown
### Quick Links

- **[INDEX.md](INDEX.md)** - 🔍 **Searchable index with official links**
- **[SKILLS-MAP.md](SKILLS-MAP.md)** - 🗺️ **Skills Relationship Map**
- **[NAVIGATION.md](NAVIGATION.md)** - 🧭 **Quick navigation guide**
- **[MAINTENANCE.md](MAINTENANCE.md)** - 🔄 **Maintenance guide**
```

→ 目的別のドキュメントへの導線

4. **実測値の明記**

```markdown
| Algorithm | Speedup | p-value | Effect Size | R² |
|-----------|---------|---------|-------------|-----|
| FFT | **852×** | <0.001 | d=30.9 | 0.9997 |
| Binary Search | **4,027×** | <0.001 | d=67.3 | 0.9997 |
```

→ 実際の実験結果（サンプルサイズ、統計検定など明記）

### 例2: zenn-articles

**特徴:**
- シンプルで実用的
- 重要な注意事項を前面に
- 運用ワークフローを重視

**優れている点:**

1. **警告を最初に配置**

```markdown
## 📚 本の執筆について

**⚠️ 重要: 本を執筆する前に必ず以下のファイルを読んでください**

### [`.zenn-deployment-notes.md`](./.zenn-deployment-notes.md)
```

→ トラブルを防ぐための注意喚起

2. **クイックスタート**

```markdown
## 🚀 クイックスタート

### 新しい本を執筆する場合

1. **デプロイ注意事項を確認**:
   ```bash
   cat .zenn-deployment-notes.md
   ```

2. **スラッシュコマンドを使用**:
   ```
   /start-book-writing
   ```
```

→ 段階的な手順

3. **ディレクトリ構成**

```markdown
## 📝 ディレクトリ構成

```
zenn-articles/
├── .zenn-deployment-notes.md  # ⚠️ 執筆前に必読
├── .claude/
│   └── commands/
├── articles/                  # Zenn記事
└── books/                     # Zenn本
```
```

→ コメント付きで構造を説明

---

## 段階的な情報開示

READMEは「玄関」ですが、すべての情報を詰め込む必要はありません。

### 原則: 概要 → 詳細

```markdown
## Installation

```bash
npm install package-name
```

詳しいインストール方法は [docs/installation.md](docs/installation.md) を参照してください。
```

### 詳細ドキュメントへのリンク

```markdown
## Architecture

このプロジェクトは、以下のような構成になっています：

```
User → API Gateway → Microservices → Database
```

詳細なアーキテクチャについては、以下のドキュメントを参照してください：

- [Architecture Overview](docs/architecture/overview.md)
- [API Design](docs/architecture/api-design.md)
- [Database Schema](docs/architecture/database.md)
```

### 実例へのリンク

```markdown
## Examples

基本的な使い方：

```typescript
import { createApp } from 'package-name';
const app = createApp();
```

より詳細な例は [examples/](examples/) を参照してください：

- [Basic Example](examples/01-basic.ts)
- [Advanced Example](examples/02-advanced.ts)
- [Custom Configuration](examples/03-custom-config.ts)
```

---

## 保守性を高める工夫

READMEは、プロジェクトと共に進化します。保守しやすい構成を心がけましょう。

### 1. バージョン情報を一元化

バージョン番号をハードコードしない：

```markdown
❌ 悪い例:

## Installation

Node.js 18.0.0以上が必要です。

npm install package-name@1.2.3
```

```markdown
✅ 良い例:

## Installation

Node.jsの推奨バージョンは、[.node-version](.node-version) を参照してください。

```bash
npm install package-name
# または
npm install package-name@latest
```
```

### 2. 動的バッジの使用

静的な情報をバッジで自動更新：

```markdown
❌ 悪い例:

現在のバージョン: 1.2.3

✅ 良い例:

![Version](https://img.shields.io/npm/v/package-name)
```

### 3. 自動生成ツールの活用

**目次の自動生成:**

```markdown
<!-- toc -->
<!-- tocstop -->
```

→ `markdown-toc -i README.md` で自動更新

**バッジの自動生成:**

GitHub Actionsで自動更新：

```yaml
name: Update README Badges
on:
  push:
    branches: [main]
jobs:
  update-badges:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: ./scripts/update-badges.sh
```

### 4. ドキュメントのモジュール化

READMEが長くなりすぎたら、分割します：

```
project/
├── README.md              # メインREADME
├── docs/
│   ├── installation.md    # インストール詳細
│   ├── configuration.md   # 設定詳細
│   ├── api.md             # API仕様
│   └── contributing.md    # コントリビューションガイド
└── CONTRIBUTING.md        # コントリビューション概要
```

READMEには概要とリンクのみ：

```markdown
## Installation

```bash
npm install package-name
```

詳細なインストール手順は [docs/installation.md](docs/installation.md) を参照してください。
```

---

## よくある間違い

### 1. 情報が多すぎる

```markdown
❌ 悪い例:

## Installation

### macOS

#### Homebrewを使う場合

1. Homebrewをインストール
   - ターミナルを開く
   - 以下のコマンドを実行:
     ```bash
     /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
     ```
   - パスワードを入力
   - （以下、詳細な説明が続く...3000文字）

#### Homebrewを使わない場合
（さらに詳細な説明...2000文字）

### Windows
（さらに詳細な説明...3000文字）

### Linux
（さらに詳細な説明...2500文字）
```

**問題点:**
- READMEが長すぎる
- 基本的な情報が埋もれる
- 保守が大変

**改善案:**

```markdown
✅ 良い例:

## Installation

### Quick Install

```bash
# macOS/Linux
brew install package-name

# Windows
choco install package-name

# npm (全OS)
npm install -g package-name
```

詳細なインストール手順は [docs/installation.md](docs/installation.md) を参照してください。
```

### 2. 情報が少なすぎる

```markdown
❌ 悪い例:

# MyProject

便利なツールです。

## Install

```bash
npm install
```
```

**問題点:**
- 何をするプロジェクトか不明
- 使い方がわからない
- ライセンス不明

**改善案:**

```markdown
✅ 良い例:

# MyProject

> タスク管理を効率化するCLIツール

## 🚀 Quick Start

```bash
# Install
npm install -g myproject

# Create a task
myproject add "Buy groceries"

# List tasks
myproject list
```

## 📜 License

MIT License - See [LICENSE](LICENSE) for details.
```

### 3. コマンドが実行できない

```markdown
❌ 悪い例:

## Installation

```bash
npm install package-name
cd package-name
npm start
```
```

**問題点:**
- `npm install package-name` はグローバルインストールではない
- インストール後に `cd package-name` はできない

**改善案:**

```markdown
✅ 良い例:

## Installation

```bash
# グローバルインストール
npm install -g package-name

# または、プロジェクトローカル
npm install package-name
```

## Usage

```bash
# グローバルインストールの場合
package-name [command]

# プロジェクトローカルの場合
npx package-name [command]
```
```

### 4. 古い情報

```markdown
❌ 悪い例:

## Installation

Node.js 12以上が必要です。
```

**問題点:**
- Node.js 12は2022年にEOL（End of Life）
- 古い情報が放置されている

**改善案:**

```markdown
✅ 良い例:

## Installation

### Prerequisites

- Node.js 18.0.0 or higher ([Download](https://nodejs.org/))

推奨バージョンは [.node-version](.node-version) を参照してください。
```

### 5. 誠実性の欠如

```markdown
❌ 悪い例:

## Performance

このツールを使うと、処理時間が3.2秒から0.8秒に短縮されます（75%改善）。
メモリ使用量も1.2GBから200MBに削減されました（83%削減）。
```

**問題点:**
- 実測していないのに具体的な数値
- 読者を誤解させる

**改善案:**

```markdown
✅ 良い例:

## Performance

このツールは、以下のような最適化手法を使用しています：

- Stream処理によるメモリ効率の向上
- 並列処理による処理時間の短縮
- キャッシュ機構による重複処理の削減

理論的には、大容量データ（10万件以上）の処理において、
従来の一括読み込み方式と比較して、以下のような効果が期待できます：

- メモリ使用量の削減（データ全体をメモリに載せない）
- 処理時間の短縮（並列処理による）

実際の効果は、データの特性や実行環境により異なります。
```

---

## チェックリスト

あなたのREADMEを評価してみましょう。

### 必須項目

- [ ] **プロジェクト名**がh1見出しで明記されている
- [ ] **1文での説明**がある
- [ ] **ライセンス**が明記されている（LICENSEファイルへのリンク）
- [ ] **インストール方法**が記載されている
- [ ] **基本的な使い方**の例がある

### 推奨項目

- [ ] **バッジ**で品質やステータスを示している
- [ ] **目次**がある（READMEが長い場合）
- [ ] **主要な機能**が箇条書きで示されている
- [ ] **前提条件**（Node.jsバージョン、OSなど）が明記されている
- [ ] **サンプルコード**が動作する（コピペで試せる）

### 品質項目

- [ ] **情報が最新**である（古い情報が放置されていない）
- [ ] **誠実性**を保っている（実測していないデータを事実のように書いていない）
- [ ] **段階的開示**ができている（詳細は別ドキュメントにリンク）
- [ ] **読者ファースト**である（検討者、導入者、コントリビューターそれぞれが求める情報がある）
- [ ] **一貫性**がある（スタイル、トーン、フォーマットが統一されている）

### アクセシビリティ項目

- [ ] **画像に代替テキスト**がある
- [ ] **リンクが明確**である（「こちら」ではなく、リンク先が明確）
- [ ] **見出しの階層**が適切である（h1→h2→h3の順）
- [ ] **コードブロックに言語指定**がある（シンタックスハイライトのため）

### OSSプロジェクト向け

- [ ] **Contributing**セクションがある
- [ ] **Code of Conduct**へのリンクがある
- [ ] **Issue/PRのテンプレート**へのリンクがある
- [ ] **開発環境のセットアップ**手順がある
- [ ] **テストの実行方法**が記載されている

---

## まとめ

この章では、READMEの基本構成について学びました。

### 重要なポイント

1. **読者ファースト**: 検討者、導入者、コントリビューター、それぞれの目的を理解する
2. **必須セクション**: タイトル、概要、インストール、使い方、ライセンスは必ず含める
3. **段階的開示**: READMEは概要、詳細は別ドキュメントに
4. **誠実性**: 実測していないデータを事実のように書かない
5. **保守性**: バージョン情報の一元化、動的バッジの活用

### 次のステップ

次の章では、「プロジェクト説明の書き方」について学びます：

- プロジェクトの価値を1分で伝える方法
- 主要な機能の説明
- スクリーンショット・図解の活用
- ユースケースの提示

READMEの「概要」セクションをより魅力的にする方法を、具体的に学びます。

---

## 参考リソース

### 公式ガイド

- [GitHub - About READMEs](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-readmes)
- [Markdown Guide](https://www.markdownguide.org/)
- [Shields.io](https://shields.io/) - バッジ作成サービス

### 優れたREADMEの例

- [claude-code-skills](https://github.com/Gaku52/claude-code-skills) - 研究・学術プロジェクト
- [zenn-articles](https://github.com/Gaku52/zenn-articles) - シンプルで実用的
- [React](https://github.com/facebook/react) - ライブラリの例
- [Next.js](https://github.com/vercel/next.js) - フレームワークの例

### ツール

- [markdown-toc](https://github.com/jonschlinkert/markdown-toc) - 目次自動生成
- [Markdown All in One (VS Code)](https://marketplace.visualstudio.com/items?itemName=yzhang.markdown-all-in-one) - Markdown編集支援

---

**次の章**: [05-project-description.md](./05-project-description.md) - プロジェクト説明の書き方
