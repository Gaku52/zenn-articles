---
title: "実践的なREADMEパターン"
---

# 実践的なREADMEパターン

## この章で学ぶこと

READMEは、プロジェクトの種類によって構成や内容が大きく異なります。本章では、実際のプロジェクトを参照しながら、以下の4つのパターンを学びます：

1. **OSSライブラリ型** - 再利用可能なコードパッケージ
2. **Webアプリケーション型** - エンドユーザー向けアプリケーション
3. **CLIツール型** - コマンドラインツール
4. **モノレポ型** - 複数プロジェクトの統合管理

それぞれのパターンについて、実際のREADMEを参照しながら、必要なセクション、構成、書き方を具体的に解説します。

---

## なぜパターンを学ぶのか

### プロジェクトタイプによる違い

同じREADMEでも、プロジェクトの種類によって読者が求める情報は大きく異なります：

| プロジェクトタイプ | 主な読者 | 求める情報 |
|------------------|---------|----------|
| OSSライブラリ | 開発者（利用者） | API、インストール方法、使用例 |
| Webアプリ | エンドユーザー、開発者 | 機能、デモ、セットアップ手順 |
| CLIツール | エンドユーザー | コマンド一覧、使い方、インストール |
| モノレポ | チームメンバー | プロジェクト構造、開発フロー |

**間違ったパターンを使うと：**
- 必要な情報が見つからない
- 不要な情報が多すぎる
- 読者が混乱する

**正しいパターンを使うと：**
- 読者が求める情報に素早くアクセスできる
- プロジェクトの第一印象が良くなる
- コントリビューターが増える

---

## パターン1: OSSライブラリ型README

### 特徴と目的

OSSライブラリのREADMEは、**開発者が他のプロジェクトに組み込むことを前提**としています。

**主な読者：**
- ライブラリを検討している開発者
- すでに使っている開発者
- コントリビューター

**必須の情報：**
- インストール方法
- クイックスタートの例
- API仕様・使い方
- ライセンス情報

### 実例：claude-code-skills

実際のOSSプロジェクト [claude-code-skills](https://github.com/Gaku52/claude-code-skills) を参考に、構成を見ていきましょう。

#### 1. バッジセクション

```markdown
![Progress](https://img.shields.io/badge/Progress-100%25-green)
![Skills](https://img.shields.io/badge/Skills-25%2F25-blue)
![Characters](https://img.shields.io/badge/Characters-3485K-informational)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
```

**目的：**
- プロジェクトのステータスを一目で伝える
- 品質指標を視覚化する
- ライセンスを明示する

**よく使われるバッジ：**
- ビルドステータス（CI/CD）
- テストカバレッジ
- npmバージョン
- ダウンロード数
- ライセンス

#### 2. プロジェクト概要

```markdown
> **A comprehensive collection of mathematically rigorous algorithm proofs,
> distributed systems theory, and formal verification, achieving MIT
> master's thesis level standards.**
```

**ポイント：**
- 1-2文でプロジェクトの本質を伝える
- 専門用語は最小限に
- **何ができるか**を明確に

#### 3. 主要機能のハイライト

```markdown
## 🎯 Project Overview

- **25 Algorithm Proofs**: Data structures, sorting, graphs
- **5 Distributed Systems Proofs**: CAP theorem, Paxos, Raft
- **3 TLA+ Formal Specifications**: 152,500+ verified states
- **Statistical Rigor**: n≥30, p<0.001, R²>0.999
```

**ポイント：**
- 数値で具体性を持たせる（実測値の場合のみ）
- 箇条書きで読みやすく
- 絵文字で視認性を高める（適度に）

#### 4. クイックスタート

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
import { pairedTTest } from '@claude-code-skills/stats';

const before = [12.5, 13.2, 11.8, 14.1, 12.9];
const after = [4.8, 5.2, 4.5, 5.5, 4.9];
const result = pairedTTest(before, after);

console.log(`p-value: ${result.p < 0.001 ? '<0.001' : result.p.toFixed(3)}`);
```
```

**ポイント：**
- コピー&ペーストで動くコード
- インストールから使用までを5分以内に
- シンプルな例から始める

#### 5. ディレクトリ構成

```markdown
## 📚 Repository Structure

```
claude-code-skills/
├── backend-development/
│   └── guides/algorithms/           # 25 algorithm proofs
│
├── packages/                        # npm packages
│   ├── stats/                       # Statistical analysis library
│   └── crdt/                        # CRDT implementations
│
└── demos/                           # Interactive demos
    ├── stats-playground/
    └── crdt-demo/
```
```

**ポイント：**
- プロジェクトの全体像を把握しやすく
- 重要なディレクトリにコメントを追加
- 深すぎる階層は省略

#### 6. ドキュメントへのナビゲーション

```markdown
## 📖 Navigation & Documentation

### Quick Links

- **[INDEX.md](INDEX.md)** - 🔍 Searchable index with official links
- **[SKILLS-MAP.md](SKILLS-MAP.md)** - 🗺️ Skills Relationship Map
- **[NAVIGATION.md](NAVIGATION.md)** - 🧭 Quick navigation guide
- **[MAINTENANCE.md](MAINTENANCE.md)** - 🔄 Maintenance guide
```

**ポイント：**
- 重要なドキュメントへのリンク
- 各ドキュメントの目的を簡潔に説明
- 絵文字で視認性を高める

#### 7. コントリビューションガイド

```markdown
## 🤝 Contributing

This is a personal research project, but feedback and suggestions are welcome!

**For questions or discussions**:
- Open an issue on GitHub
- Reference specific proof files
- Cite relevant papers
```

**ポイント：**
- コントリビューションの受け入れ方針を明記
- プライベートプロジェクトでも、フィードバックは歓迎できる
- 質問の仕方を具体的に示す

#### 8. ライセンス

```markdown
## 📜 License

MIT License - See [LICENSE](LICENSE) file for details.
```

**ポイント：**
- ライセンスを必ず明記
- LICENSEファイルへのリンク
- 著作権情報（必要に応じて）

### OSSライブラリREADMEのチェックリスト

OSSライブラリのREADMEを書く際に確認すべき項目：

- [ ] **バッジ**: ビルドステータス、カバレッジ、ライセンス
- [ ] **概要**: 1-2文でプロジェクトの本質を伝える
- [ ] **インストール**: npmやyarnでのインストール方法
- [ ] **クイックスタート**: 5分で試せるコード例
- [ ] **API仕様**: 主要な関数やクラスの使い方
- [ ] **ディレクトリ構成**: プロジェクトの全体像
- [ ] **ドキュメント**: 詳細なドキュメントへのリンク
- [ ] **コントリビューション**: 貢献の仕方
- [ ] **ライセンス**: ライセンス情報の明記

---

## パターン2: Webアプリケーション型README

### 特徴と目的

WebアプリケーションのREADMEは、**エンドユーザーと開発者の両方を対象**とします。

**主な読者：**
- アプリケーションを使いたいユーザー
- ローカルで動かしたい開発者
- デプロイしたい運用担当者

**必須の情報：**
- アプリケーションの機能
- デモサイトやスクリーンショット
- ローカル開発環境のセットアップ
- デプロイ方法

### 実例：zenn-articles

実際のプロジェクト [zenn-articles](https://github.com/Gaku52/zenn-articles) を参考に構成を見ていきます。

#### 1. プロジェクト説明

```markdown
# Zenn Articles & Books

このリポジトリはZennの記事と本を管理しています。
```

**ポイント：**
- プロジェクト名とシンプルな説明
- 何を管理しているかを明確に

#### 2. 重要な注意事項

```markdown
## 📚 本の執筆について

**⚠️ 重要: 本を執筆する前に必ず以下のファイルを読んでください**

### [`.zenn-deployment-notes.md`](./.zenn-deployment-notes.md)

このファイルには、Zennのデプロイで問題が起きた際の
トラブルシューティングと、問題を未然に防ぐための
ベストプラクティスが記載されています。
```

**ポイント：**
- 重要な情報を目立たせる（⚠️や**太字**）
- 先に読むべきドキュメントへのリンク
- なぜ重要かを簡潔に説明

#### 3. クイックスタート

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

**ポイント：**
- ユースケース別に手順を分ける
- 番号付きリストで順序を明確に
- 実際に実行するコマンドを記載

#### 4. 既存コンテンツの紹介

```markdown
## 📖 既存の本

### React Development 2026 [Official]

**場所**: `books/react-development-2026-official/`

**内容**:
- 全10章（Chapter 00-09）
- React 18+、Next.js 14+、TypeScript 5+対応
- useState/useEffect → Hooks → TypeScript → Performance

**ステータス**: ✅ 完成
```

**ポイント：**
- 既存のコンテンツを紹介
- ステータスを明確に（作成中/完成）
- 対応バージョンを明記

#### 5. ディレクトリ構成

```markdown
## 📝 ディレクトリ構成

```
zenn-articles/
├── .zenn-deployment-notes.md  # ⚠️ 執筆前に必読
├── .claude/
│   └── commands/
├── articles/                  # Zenn記事
└── books/                     # Zenn本
    └── react-development-2026-official/
        ├── config.yaml
        ├── cover.png
        └── *.md              # チャプターファイル
```
```

**ポイント：**
- プロジェクトの全体構造を可視化
- 重要なファイル・ディレクトリにコメント
- 実際のディレクトリ構造を反映

#### 6. ベストプラクティス

```markdown
## 🎯 ベストプラクティス

### 執筆ワークフロー

1. **デプロイ注意事項を確認** (`.zenn-deployment-notes.md`)
2. チャプターを執筆
3. **関連チャプターを全て同じコミットに含める**
4. コミット・プッシュ
5. **3-5分待ってZennのデプロイログを確認**
6. 問題があれば`.zenn-deployment-notes.md`のトラブルシューティングを参照
```

**ポイント：**
- 推奨ワークフローを明記
- 重要な手順を**太字**で強調
- トラブル時の対処法を案内

#### 7. トラブルシューティング

```markdown
## 🔧 トラブルシューティング

問題が発生した場合は、必ず[`.zenn-deployment-notes.md`]を確認してください。

よくある問題:
- チャプターがZennに反映されない → デプロイログを確認
- 文字数が少ない（300-400字） → ファイルを少し編集して再プッシュ
- 「執筆中」のまま → 3-5分待つ、またはファイル編集して再プッシュ
```

**ポイント：**
- よくある問題と解決策を列挙
- 詳細ドキュメントへのリンク
- 具体的な対処法を記載

### Webアプリケーション型READMEのチェックリスト

- [ ] **プロジェクト説明**: 何をするアプリケーションか
- [ ] **デモ**: ライブデモのURLまたはスクリーンショット
- [ ] **主要機能**: 何ができるかを箇条書き
- [ ] **技術スタック**: 使用している技術（React、Node.jsなど）
- [ ] **セットアップ**: ローカル環境の構築手順
- [ ] **環境変数**: .env.exampleの説明
- [ ] **開発サーバー**: npm run devなどの起動方法
- [ ] **ビルド**: 本番ビルドの方法
- [ ] **デプロイ**: Vercel、Netlifyなどへのデプロイ手順
- [ ] **トラブルシューティング**: よくある問題と解決策

---

## パターン3: CLIツール型README

### 特徴と目的

CLIツールのREADMEは、**コマンドラインでの使い方を明確に伝える**ことが最優先です。

**主な読者：**
- ツールを使いたいエンドユーザー
- ツールを改良したい開発者

**必須の情報：**
- インストール方法
- 基本的な使い方
- コマンド一覧
- オプション・フラグの説明

### 構成例

#### 1. 簡潔な説明

```markdown
# awesome-cli

高速で軽量なファイル検索ツール

## 特徴

- 🚀 ripgrepベースの高速検索
- 📁 Git/Dockerfileの自動除外
- 🎨 シンタックスハイライト対応
- ⚡ インクリメンタルサーチ
```

**ポイント：**
- 1行で何のツールかを説明
- 特徴を箇条書きで列挙
- 絵文字で視認性を高める

#### 2. インストール

```markdown
## インストール

### Homebrew (macOS/Linux)

```bash
brew install awesome-cli
```

### Cargo (Rust)

```bash
cargo install awesome-cli
```

### バイナリダウンロード

[Releases](https://github.com/user/awesome-cli/releases)から
お使いのOSに合ったバイナリをダウンロードしてください。
```

**ポイント：**
- 複数のインストール方法を提供
- OS別に手順を分ける
- パッケージマネージャーごとに記載

#### 3. 基本的な使い方

```markdown
## 使い方

### 基本検索

```bash
# カレントディレクトリから検索
awesome search "pattern"

# 特定ディレクトリから検索
awesome search "pattern" /path/to/dir
```

### 高度な検索

```bash
# 正規表現で検索
awesome search -r "user_[0-9]+"

# ファイルタイプを指定
awesome search "TODO" --type ts

# 大文字小文字を区別しない
awesome search -i "error"
```
```

**ポイント：**
- シンプルな例から始める
- コメントで説明を追加
- よく使うパターンを網羅

#### 4. コマンド一覧

```markdown
## コマンド

### `search <pattern> [path]`

ファイルを検索します。

**引数:**
- `pattern` - 検索パターン（正規表現対応）
- `path` - 検索対象のパス（省略時はカレントディレクトリ）

**オプション:**
- `-r, --regex` - 正規表現として扱う
- `-i, --ignore-case` - 大文字小文字を区別しない
- `-t, --type <type>` - ファイルタイプで絞り込み
- `-e, --exclude <pattern>` - 除外パターン

**例:**
```bash
awesome search "error" src/
awesome search -r "user_[0-9]+" --type ts
```

### `config [options]`

設定を管理します。

**オプション:**
- `--list` - 現在の設定を表示
- `--set <key> <value>` - 設定を変更
- `--reset` - デフォルトに戻す

**例:**
```bash
awesome config --list
awesome config --set theme dark
```
```

**ポイント：**
- コマンドごとにセクションを分ける
- 引数とオプションを明確に区別
- 実際の使用例を必ず記載

#### 5. 設定ファイル

```markdown
## 設定

`~/.config/awesome/config.toml`で設定をカスタマイズできます。

```toml
# デフォルト設定
[search]
ignore_case = false
regex_mode = false
max_results = 1000

[display]
theme = "dark"
syntax_highlight = true

[exclude]
patterns = [
  "node_modules",
  ".git",
  "*.log"
]
```

**設定項目:**
- `search.ignore_case`: 大文字小文字の区別（デフォルト: false）
- `search.regex_mode`: 正規表現モード（デフォルト: false）
- `display.theme`: テーマ（dark/light）
```

**ポイント：**
- 設定ファイルの場所を明記
- デフォルト設定を例示
- 各設定項目の説明を追加

#### 6. トラブルシューティング

```markdown
## トラブルシューティング

### コマンドが見つからない

```bash
# PATHを確認
echo $PATH

# バイナリの場所を確認
which awesome
```

### 検索が遅い

- `.gitignore`や設定ファイルで不要なディレクトリを除外してください
- `--max-results`オプションで結果数を制限できます

### 文字化けが発生する

- ターミナルのエンコーディングをUTF-8に設定してください
- `LANG`環境変数を確認: `echo $LANG`
```

**ポイント：**
- よくある問題を列挙
- 診断コマンドを提示
- 解決策を具体的に記載

### CLIツール型READMEのチェックリスト

- [ ] **簡潔な説明**: 1行で何のツールかを説明
- [ ] **特徴**: 他のツールと何が違うか
- [ ] **インストール**: OS/パッケージマネージャー別の方法
- [ ] **基本的な使い方**: 最も一般的なユースケース
- [ ] **コマンド一覧**: 全コマンドの仕様
- [ ] **オプション**: フラグとその説明
- [ ] **設定ファイル**: カスタマイズ方法
- [ ] **使用例**: 実際のユースケース
- [ ] **トラブルシューティング**: よくある問題
- [ ] **アンインストール**: 削除方法

---

## パターン4: モノレポ型README

### 特徴と目的

モノレポのREADMEは、**複数のプロジェクトを統合管理する構造を説明**します。

**主な読者：**
- チームメンバー
- 新規参加者
- プロジェクト管理者

**必須の情報：**
- モノレポの構成
- プロジェクト一覧
- 開発ワークフロー
- 共通コマンド

### 構成例

#### 1. プロジェクト概要

```markdown
# My Monorepo

複数のフロントエンド・バックエンドプロジェクトを統合管理するモノレポです。

## 📦 含まれるプロジェクト

### Frontend

- **web** - メインWebアプリケーション（Next.js 14）
- **mobile** - モバイルアプリ（React Native）
- **admin** - 管理画面（React + Vite）

### Backend

- **api** - REST API（NestJS）
- **graphql** - GraphQL API（Apollo Server）
- **worker** - バックグラウンドジョブ（BullMQ）

### Shared

- **ui** - 共通UIコンポーネント（React）
- **utils** - ユーティリティ関数（TypeScript）
- **types** - 型定義（TypeScript）
```

**ポイント：**
- 全プロジェクトを一覧化
- 技術スタックを明記
- カテゴリ別に整理（Frontend/Backend/Shared）

#### 2. ディレクトリ構成

```markdown
## 📁 ディレクトリ構成

```
monorepo/
├── apps/                    # アプリケーション
│   ├── web/                # Next.js Webアプリ
│   ├── mobile/             # React Native
│   ├── admin/              # 管理画面
│   ├── api/                # REST API
│   ├── graphql/            # GraphQL API
│   └── worker/             # バックグラウンドジョブ
│
├── packages/               # 共有パッケージ
│   ├── ui/                 # UIコンポーネント
│   ├── utils/              # ユーティリティ
│   └── types/              # 型定義
│
├── tools/                  # 開発ツール
│   ├── scripts/            # ビルドスクリプト
│   └── config/             # 共通設定
│
├── package.json            # ルートpackage.json
├── turbo.json              # Turboレポ設定
└── tsconfig.base.json      # 共通TypeScript設定
```
```

**ポイント：**
- apps/とpackages/を明確に区別
- 各ディレクトリの役割をコメント
- 重要な設定ファイルも記載

#### 3. セットアップ

```markdown
## 🚀 セットアップ

### 前提条件

- Node.js 20.x以上
- pnpm 8.x以上
- Docker Desktop（ローカルDB用）

### 初回セットアップ

```bash
# 依存関係のインストール
pnpm install

# 環境変数の設定
cp .env.example .env
# .envファイルを編集してください

# データベースの起動
docker-compose up -d

# データベースのマイグレーション
pnpm db:migrate

# 開発サーバーの起動
pnpm dev
```

これで以下のサービスが起動します：
- Web: http://localhost:3000
- API: http://localhost:4000
- Admin: http://localhost:3001
```

**ポイント：**
- 前提条件を明記
- セットアップ手順を番号付きリストで
- 起動後のURLを記載

#### 4. 開発ワークフロー

```markdown
## 💻 開発ワークフロー

### 特定のアプリを起動

```bash
# Webアプリのみ起動
pnpm --filter web dev

# APIのみ起動
pnpm --filter api dev

# Webアプリ + APIを起動
pnpm --filter web --filter api dev
```

### ビルド

```bash
# 全プロジェクトをビルド
pnpm build

# 特定のプロジェクトをビルド
pnpm --filter web build
```

### テスト

```bash
# 全プロジェクトのテスト
pnpm test

# 特定のプロジェクトのテスト
pnpm --filter api test

# E2Eテスト
pnpm test:e2e
```

### Lint & Format

```bash
# Lint
pnpm lint

# Format
pnpm format

# 自動修正
pnpm lint:fix
pnpm format:fix
```
```

**ポイント：**
- よく使うコマンドを網羅
- フィルターオプションの使い方を説明
- カテゴリ別に整理（起動/ビルド/テスト）

#### 5. パッケージ間の依存関係

```markdown
## 🔗 パッケージ間の依存関係

### 依存関係マップ

```
apps/web
├── @repo/ui
├── @repo/utils
└── @repo/types

apps/mobile
├── @repo/ui
├── @repo/utils
└── @repo/types

apps/api
├── @repo/utils
└── @repo/types

packages/ui
├── @repo/utils
└── @repo/types
```

### 新しい依存関係の追加

```bash
# apps/webに外部パッケージを追加
pnpm --filter web add axios

# apps/webに内部パッケージを追加
pnpm --filter web add @repo/utils --workspace

# 全プロジェクトに追加（ルート）
pnpm add -w eslint
```
```

**ポイント：**
- 依存関係を可視化
- 内部パッケージと外部パッケージの区別
- パッケージ追加方法を具体的に記載

#### 6. CI/CD

```markdown
## 🚢 CI/CD

### GitHub Actions

プルリクエスト時に以下が実行されます：

1. **Lint & Type Check**: 全プロジェクト
2. **Unit Tests**: 全プロジェクト
3. **E2E Tests**: apps/web
4. **Build**: 全プロジェクト

### デプロイ

```bash
# ステージング環境へデプロイ
pnpm deploy:staging

# 本番環境へデプロイ（mainブランチのみ）
pnpm deploy:production
```

**デプロイ先:**
- Web: Vercel
- API: AWS ECS
- Worker: AWS ECS
```

**ポイント：**
- CI/CDの流れを説明
- デプロイコマンドを記載
- デプロイ先を明記

#### 7. コントリビューションガイド

```markdown
## 🤝 コントリビューション

### ブランチ戦略

- `main` - 本番環境
- `develop` - 開発環境
- `feature/*` - 新機能
- `fix/*` - バグ修正

### プルリクエストの作成

1. `develop`から新しいブランチを作成
   ```bash
   git checkout -b feature/new-feature develop
   ```

2. 変更をコミット
   ```bash
   git add .
   git commit -m "feat: Add new feature"
   ```

3. プッシュしてPRを作成
   ```bash
   git push origin feature/new-feature
   ```

4. PRテンプレートに従って説明を記載

### コミットメッセージ規約

[Conventional Commits](https://www.conventionalcommits.org/)に従ってください：

- `feat:` - 新機能
- `fix:` - バグ修正
- `docs:` - ドキュメント
- `style:` - コードスタイル
- `refactor:` - リファクタリング
- `test:` - テスト
- `chore:` - その他
```

**ポイント：**
- ブランチ戦略を明記
- PR作成手順を具体的に
- コミット規約を参照

### モノレポ型READMEのチェックリスト

- [ ] **プロジェクト一覧**: 含まれる全プロジェクト
- [ ] **ディレクトリ構成**: apps/とpackages/の区別
- [ ] **セットアップ**: 初回セットアップ手順
- [ ] **共通コマンド**: dev/build/testなど
- [ ] **フィルター**: 特定プロジェクトの操作方法
- [ ] **依存関係**: パッケージ間の関係
- [ ] **パッケージ追加**: 新しい依存関係の追加方法
- [ ] **CI/CD**: ビルド・デプロイの流れ
- [ ] **ブランチ戦略**: Git運用ルール
- [ ] **コントリビューション**: PR作成手順

---

## 共通要素と差別化

### すべてのREADMEに共通する要素

どのタイプのREADMEでも、以下の要素は**必須**です：

#### 1. プロジェクト名と説明

```markdown
# Project Name

1-2行でプロジェクトの本質を説明
```

#### 2. インストール/セットアップ

```markdown
## インストール

前提条件とセットアップ手順
```

#### 3. 基本的な使い方

```markdown
## 使い方

最も一般的なユースケース
```

#### 4. ライセンス

```markdown
## ライセンス

MIT License - 詳細は[LICENSE](LICENSE)を参照
```

### タイプ別の差別化ポイント

| 要素 | OSS | Webアプリ | CLI | モノレポ |
|------|-----|----------|-----|---------|
| APIリファレンス | ✅ 必須 | ❌ 不要 | ❌ 不要 | △ 共有パッケージのみ |
| デモ/スクリーンショット | △ あると良い | ✅ 必須 | △ あると良い | ❌ 不要 |
| コマンド一覧 | ❌ 不要 | △ npm scriptsのみ | ✅ 必須 | ✅ 必須 |
| プロジェクト構成 | △ あると良い | △ あると良い | ❌ 不要 | ✅ 必須 |
| 開発ワークフロー | △ コントリビューター向け | ✅ 必須 | △ 開発者向け | ✅ 必須 |

---

## 実践的なTips

### Tip 1: 読者の目的を考える

READMEを書く前に、読者が何を知りたいかを考えましょう：

**5秒で知りたいこと：**
- このプロジェクトは何か
- 何ができるか

**5分で知りたいこと：**
- インストール方法
- 基本的な使い方

**30分で知りたいこと：**
- 詳細なAPI仕様
- カスタマイズ方法
- トラブルシューティング

### Tip 2: 実際に動くコード例を使う

コード例は**必ずコピー&ペーストで動作する**ものにしてください：

```typescript
// ✅ 良い例: 完全なコード
import express from 'express';

const app = express();

app.get('/api/hello', (req, res) => {
  res.json({ message: 'Hello World' });
});

app.listen(3000, () => {
  console.log('Server running on http://localhost:3000');
});
```

```typescript
// ❌ 悪い例: 不完全なコード
app.get('/api/hello', (req, res) => {
  // ...
});
```

### Tip 3: バッジを適切に使う

バッジは有用ですが、多すぎると逆効果です：

```markdown
<!-- ✅ 適切な量 -->
![Build Status](...)
![Coverage](...)
![License](...)

<!-- ❌ 多すぎる -->
![Build](...)
![Coverage](...)
![License](...)
![Downloads](...)
![Stars](...)
![Forks](...)
![Issues](...)
![PRs](...)
```

**推奨バッジ（3-5個）：**
- ビルドステータス（CI/CD）
- テストカバレッジ
- ライセンス
- バージョン（npmなど）
- ダウンロード数（人気プロジェクトの場合）

### Tip 4: 目次を活用する

長いREADMEには目次を追加しましょう：

```markdown
## 目次

- [インストール](#インストール)
- [使い方](#使い方)
- [API仕様](#api仕様)
- [コントリビューション](#コントリビューション)
- [ライセンス](#ライセンス)
```

GitHubは自動的にアンカーリンクを生成します：
- `## インストール` → `#インストール`
- `## API仕様` → `#api仕様`

### Tip 5: スクリーンショットは必要最小限に

スクリーンショットは有用ですが、以下の点に注意：

**含めるべきスクリーンショット：**
- UIの全体像（Webアプリ）
- 特徴的な機能（差別化ポイント）
- CLIの出力例（CLIツール）

**含めないほうが良いもの：**
- 細かい設定画面
- 頻繁に変わるUI
- サイズの大きい画像（リポジトリ容量を圧迫）

**推奨サイズ：**
- 幅: 800-1200px
- フォーマット: PNG（スクリーンショット）、WebP（写真）
- 圧縮: TinyPNGなどで圧縮

### Tip 6: リンクは絶対パスと相対パスを使い分ける

```markdown
<!-- ✅ 同じリポジトリ内 → 相対パス -->
詳細は[CONTRIBUTING.md](./CONTRIBUTING.md)を参照してください。

<!-- ✅ 外部サイト → 絶対パス -->
[TypeScript公式ドキュメント](https://www.typescriptlang.org/docs/)

<!-- ✅ 他のリポジトリ → 絶対パス -->
[React公式](https://github.com/facebook/react)
```

### Tip 7: 更新日を記載する

READMEの最後に更新日を記載すると、情報の鮮度がわかります：

```markdown
---

**Last Updated**: 2026-01-28
**Version**: 2.0.0
```

---

## よくある間違い

### 間違い1: 情報が多すぎる

```markdown
❌ 悪い例:
## インストール

### macOS
#### Homebrew
##### Intel Mac
###### バージョン13以上の場合
...
```

**問題点：**
- 階層が深すぎて読みにくい
- 重要な情報が埋もれる

**改善策：**
- 重要な情報を最初に
- 詳細は別ドキュメントへ

### 間違い2: 前提知識が不明確

```markdown
❌ 悪い例:
## セットアップ

```bash
npm install
npm run dev
```
```

**問題点：**
- Node.jsのバージョンがわからない
- 環境変数の設定が不明

**改善策：**
```markdown
✅ 良い例:
## セットアップ

### 前提条件

- Node.js 20.x以上
- npm 10.x以上

### 手順

1. 依存関係のインストール
   ```bash
   npm install
   ```

2. 環境変数の設定
   ```bash
   cp .env.example .env
   # .envファイルを編集してください
   ```

3. 開発サーバーの起動
   ```bash
   npm run dev
   ```
```

### 間違い3: 古い情報が残っている

```markdown
❌ 悪い例:
## インストール

```bash
# Node.js 14以上が必要（2021年時点）
npm install
```
```

**問題点：**
- 情報が古い（Node.js 14はEOL）
- 年号が古いと信頼性が下がる

**改善策：**
- 定期的に更新する
- サポート対象バージョンを明記
- 更新日を記載

### 間違い4: コマンドが動かない

```markdown
❌ 悪い例:
```bash
cd src
npm start
```
```

**問題点：**
- `src`ディレクトリが存在しない可能性
- どのディレクトリで実行すべきか不明

**改善策：**
```markdown
✅ 良い例:
```bash
# プロジェクトルートで実行
npm start

# または特定のディレクトリで
cd apps/web && npm run dev
```
```

---

## まとめ

本章では、4つのREADMEパターンを学びました：

### パターン別の重要ポイント

**OSSライブラリ型：**
- クイックスタートを充実させる
- APIリファレンスを明確に
- インストール方法を複数提示

**Webアプリケーション型：**
- デモやスクリーンショットを活用
- セットアップ手順を詳細に
- 環境変数の設定を忘れずに

**CLIツール型：**
- コマンド一覧を網羅
- オプション・フラグを明確に
- 使用例を豊富に

**モノレポ型：**
- プロジェクト構成を可視化
- 共通コマンドを整理
- パッケージ間の関係を説明

### 共通の成功法則

どのタイプでも、以下を意識してください：

1. **読者ファースト**: 読者が求める情報を優先
2. **動くコード**: コピペで動作する例
3. **適切な粒度**: 詳細すぎず、簡潔すぎず
4. **最新性**: 定期的に更新
5. **ナビゲーション**: 長い場合は目次を

---

## チェックリスト

この章で学んだことを確認しましょう：

- [ ] プロジェクトタイプごとの違いを理解している
- [ ] 各パターンに必要な要素を把握している
- [ ] 実例を参考にできる
- [ ] 読者の目的に応じた構成を選べる
- [ ] 共通要素と差別化ポイントを理解している
- [ ] よくある間違いを避けられる

---

## 次のステップ

次の章では、**API仕様書の基本**を学びます。

RESTful APIの文書化方法、エンドポイント設計、リクエスト/レスポンス例の書き方など、APIドキュメント作成に必要な知識を体系的に学びます。

API仕様書は、バックエンド開発者だけでなく、フロントエンド開発者、QAエンジニア、プロダクトマネージャーなど、多くの関係者が参照する重要なドキュメントです。

明確で正確なAPI仕様書を書けるようになりましょう。

---

## 実践演習

### 演習1: 自分のプロジェクトの診断

以下の質問に答えて、プロジェクトに最適なパターンを見極めましょう：

**質問:**
1. プロジェクトの主な目的は何ですか？
   - [ ] 他のプロジェクトに組み込むライブラリ → **OSSライブラリ型**
   - [ ] エンドユーザーが使うWebアプリケーション → **Webアプリケーション型**
   - [ ] コマンドラインで使うツール → **CLIツール型**
   - [ ] 複数のプロジェクトを管理 → **モノレポ型**

2. 主な読者は誰ですか？
   - [ ] ライブラリを組み込む開発者 → **APIリファレンス重視**
   - [ ] アプリを使うエンドユーザー → **スクリーンショット重視**
   - [ ] コマンドを実行するユーザー → **コマンド一覧重視**
   - [ ] チームメンバー → **開発ワークフロー重視**

3. 最も重要な情報は何ですか？
   - [ ] インストール方法 → **クイックスタート充実**
   - [ ] 機能のデモ → **デモサイト・GIF追加**
   - [ ] コマンドの使い方 → **使用例を豊富に**
   - [ ] プロジェクト構成 → **ディレクトリ構成を詳細に**

### 演習2: READMEの改善

既存のREADMEを以下の観点で評価してください：

**評価項目:**

| 項目 | 現状 | 改善案 |
|------|------|--------|
| プロジェクト説明は明確か | ⭐⭐⭐☆☆ | 1行説明を追加 |
| インストール方法は具体的か | ⭐⭐☆☆☆ | OS別に手順を分ける |
| コード例は動作するか | ⭐⭐⭐⭐☆ | 実行結果も記載 |
| トラブルシューティングはあるか | ⭐☆☆☆☆ | よくある問題を追加 |
| 更新日は記載されているか | ⭐☆☆☆☆ | 最終更新日を追加 |

**改善の優先順位:**
1. 高優先度: プロジェクト説明、インストール、基本的な使い方
2. 中優先度: コード例、トラブルシューティング
3. 低優先度: バッジ、スクリーンショット、更新日

### 演習3: テンプレートの作成

自分のプロジェクト用のREADMEテンプレートを作成しましょう：

```markdown
# [Project Name]

[1行で説明]

## 特徴

- 特徴1
- 特徴2
- 特徴3

## インストール

```bash
# インストールコマンド
```

## 使い方

### 基本的な使い方

```[language]
// コード例
```

### 高度な使い方

```[language]
// 高度な例
```

## API仕様（ライブラリの場合）

### functionName(arg1, arg2)

説明

**パラメータ:**
- `arg1` (type) - 説明
- `arg2` (type) - 説明

**戻り値:**
- (type) - 説明

**例:**
```[language]
// 使用例
```

## コントリビューション

コントリビューションは歓迎します！

## ライセンス

MIT License
```

---

## パターン別テンプレート集

### OSSライブラリ型テンプレート

```markdown
# Library Name

[![npm version](https://img.shields.io/npm/v/library-name.svg)](https://www.npmjs.com/package/library-name)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

[1行の説明]

## 特徴

- 🚀 特徴1
- 📦 特徴2
- ⚡ 特徴3

## インストール

```bash
npm install library-name
# または
yarn add library-name
```

## クイックスタート

```typescript
import { functionName } from 'library-name';

const result = functionName('example');
console.log(result);
```

## API仕様

### functionName(param)

関数の説明

**パラメータ:**
- `param` (string) - パラメータの説明

**戻り値:**
- (string) - 戻り値の説明

**例:**
```typescript
const result = functionName('test');
// => 'result'
```

## ドキュメント

詳細なドキュメントは[こちら](./docs)を参照してください。

## コントリビューション

[CONTRIBUTING.md](./CONTRIBUTING.md)を参照してください。

## ライセンス

MIT License - 詳細は[LICENSE](./LICENSE)を参照してください。
```

### Webアプリケーション型テンプレート

```markdown
# App Name

[1行の説明]

## デモ

🔗 [Live Demo](https://example.com)

![Screenshot](./screenshots/main.png)

## 主な機能

- ✅ 機能1
- ✅ 機能2
- ✅ 機能3

## 技術スタック

- **Frontend**: React 18, TypeScript, Tailwind CSS
- **Backend**: Node.js, Express, PostgreSQL
- **Deployment**: Vercel

## セットアップ

### 前提条件

- Node.js 20.x以上
- PostgreSQL 15以上

### 手順

1. リポジトリのクローン
   ```bash
   git clone https://github.com/user/app-name.git
   cd app-name
   ```

2. 依存関係のインストール
   ```bash
   npm install
   ```

3. 環境変数の設定
   ```bash
   cp .env.example .env
   # .envファイルを編集してください
   ```

4. データベースのセットアップ
   ```bash
   npm run db:migrate
   npm run db:seed
   ```

5. 開発サーバーの起動
   ```bash
   npm run dev
   ```

   http://localhost:3000 でアプリケーションが起動します。

## デプロイ

### Vercel

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/user/app-name)

### 手動デプロイ

```bash
npm run build
npm start
```

## トラブルシューティング

### データベース接続エラー

`.env`ファイルの`DATABASE_URL`を確認してください。

### ポート3000が使用中

`.env`ファイルで`PORT`を変更できます：
```
PORT=3001
```

## ライセンス

MIT License
```

### CLIツール型テンプレート

```markdown
# CLI Tool Name

[1行の説明]

## インストール

### Homebrew

```bash
brew install cli-tool-name
```

### npm

```bash
npm install -g cli-tool-name
```

## 使い方

### 基本コマンド

```bash
# コマンド1
cli-tool command1 [options]

# コマンド2
cli-tool command2 [options]
```

### 例

```bash
# 例1: シンプルな使用例
cli-tool search "pattern"

# 例2: オプション付き
cli-tool search "pattern" --type ts --exclude node_modules

# 例3: 複数のフラグ
cli-tool process input.txt --output result.txt --verbose
```

## コマンド一覧

### `search <pattern> [path]`

ファイルを検索します。

**引数:**
- `pattern` - 検索パターン
- `path` - 検索パス（省略時はカレントディレクトリ）

**オプション:**
- `-t, --type <type>` - ファイルタイプで絞り込み
- `-e, --exclude <pattern>` - 除外パターン
- `-v, --verbose` - 詳細な出力

### `process <input> [options]`

ファイルを処理します。

**引数:**
- `input` - 入力ファイル

**オプション:**
- `-o, --output <file>` - 出力ファイル
- `-f, --format <format>` - 出力フォーマット
- `-v, --verbose` - 詳細な出力

## 設定

`~/.config/cli-tool/config.toml`で設定をカスタマイズできます。

```toml
[general]
verbose = false

[search]
exclude = ["node_modules", ".git"]
```

## トラブルシューティング

### コマンドが見つからない

PATHを確認してください：
```bash
echo $PATH
which cli-tool
```

### 権限エラー

管理者権限で実行してください：
```bash
sudo cli-tool command
```

## ライセンス

MIT License
```

### モノレポ型テンプレート

```markdown
# Monorepo Name

複数のプロジェクトを統合管理するモノレポです。

## 📦 プロジェクト一覧

### Apps

- **web** - メインWebアプリケーション
- **mobile** - モバイルアプリ
- **admin** - 管理画面

### Packages

- **ui** - 共通UIコンポーネント
- **utils** - ユーティリティ関数
- **types** - 型定義

## セットアップ

```bash
# 依存関係のインストール
pnpm install

# 環境変数の設定
cp .env.example .env

# 開発サーバーの起動
pnpm dev
```

## 開発

### 特定のアプリを起動

```bash
# webのみ
pnpm --filter web dev

# web + api
pnpm --filter web --filter api dev
```

### ビルド

```bash
# 全プロジェクト
pnpm build

# 特定のプロジェクト
pnpm --filter web build
```

### テスト

```bash
# 全プロジェクト
pnpm test

# 特定のプロジェクト
pnpm --filter api test
```

## プロジェクト構成

```
monorepo/
├── apps/
│   ├── web/
│   ├── mobile/
│   └── admin/
├── packages/
│   ├── ui/
│   ├── utils/
│   └── types/
└── turbo.json
```

## コントリビューション

[CONTRIBUTING.md](./CONTRIBUTING.md)を参照してください。

## ライセンス

MIT License
```

---

## README作成のワークフロー

### ステップ1: プロジェクトタイプを特定

まず、プロジェクトがどのタイプに該当するかを判断します：

```
質問: このプロジェクトは誰が使いますか？
↓
開発者（ライブラリとして） → OSSライブラリ型
エンドユーザー（アプリとして） → Webアプリケーション型
ユーザー（コマンドとして） → CLIツール型
チーム（複数プロジェクト） → モノレポ型
```

### ステップ2: テンプレートを選択

上記のテンプレートから適切なものを選びます。

### ステップ3: 必須セクションを埋める

最低限、以下のセクションを埋めます：

1. プロジェクト名と説明
2. インストール方法
3. 基本的な使い方
4. ライセンス

### ステップ4: 読者が求める情報を追加

プロジェクトタイプに応じて、追加情報を記載します：

- **OSSライブラリ**: APIリファレンス、使用例
- **Webアプリ**: デモ、スクリーンショット
- **CLIツール**: コマンド一覧、設定方法
- **モノレポ**: プロジェクト構成、ワークフロー

### ステップ5: レビューと改善

以下のチェックリストで確認します：

- [ ] プロジェクトの本質が1-2行で伝わるか
- [ ] インストール方法は明確か
- [ ] コード例は動作するか
- [ ] トラブルシューティングはあるか
- [ ] ライセンスは明記されているか
- [ ] リンク切れはないか
- [ ] 誤字脱字はないか

### ステップ6: 継続的な更新

READMEは一度書いたら終わりではありません：

**更新のタイミング:**
- 新機能を追加したとき
- APIを変更したとき
- セットアップ手順が変わったとき
- よくある質問が増えたとき

**更新のベストプラクティス:**
- PRでREADMEも同時に更新
- バージョンアップ時に見直す
- 3ヶ月に1回は全体をレビュー

---

## 高度なテクニック

### テクニック1: 動的バッジの活用

GitHub Actionsと連携して、動的に更新されるバッジを追加できます：

```markdown
<!-- ビルドステータス -->
![Build](https://github.com/user/repo/workflows/CI/badge.svg)

<!-- カバレッジ -->
![Coverage](https://codecov.io/gh/user/repo/branch/main/graph/badge.svg)

<!-- npmバージョン -->
![npm](https://img.shields.io/npm/v/package-name.svg)

<!-- ダウンロード数 -->
![Downloads](https://img.shields.io/npm/dm/package-name.svg)
```

### テクニック2: GIFアニメーションで機能を紹介

静止画像よりもGIFアニメーションの方が、機能を直感的に伝えられます：

```markdown
## デモ

![Demo](./demo.gif)
```

**GIF作成ツール:**
- [LICEcap](https://www.cockos.com/licecap/) (Windows/Mac)
- [Kap](https://getkap.co/) (Mac)
- [ScreenToGif](https://www.screentogif.com/) (Windows)

**推奨設定:**
- 解像度: 1280x720以下
- フレームレート: 10-15 fps
- ファイルサイズ: 5MB以下

### テクニック3: 折りたたみセクション

長い内容は折りたたんで読みやすくできます：

```markdown
<details>
<summary>高度な設定</summary>

## 詳細な設定方法

ここに詳細な内容...

</details>
```

**使いどころ:**
- トラブルシューティングの詳細
- 高度な設定方法
- 変更履歴（Changelog）

### テクニック4: 多言語対応

グローバルなプロジェクトでは、多言語READMEを用意します：

```markdown
# Project Name

[English](./README.md) | [日本語](./README.ja.md) | [中文](./README.zh.md)
```

**ディレクトリ構成:**
```
project/
├── README.md          # 英語（デフォルト）
├── README.ja.md       # 日本語
└── README.zh.md       # 中国語
```

### テクニック5: Contributors表示

コントリビューターを自動で表示できます：

```markdown
## Contributors

<!-- ALL-CONTRIBUTORS-LIST:START -->
<!-- ALL-CONTRIBUTORS-LIST:END -->
```

[all-contributors](https://github.com/all-contributors/all-contributors)を使うと、PRごとに自動更新されます。

---

## 参考リンク

- [GitHub READMEのベストプラクティス](https://docs.github.com/ja/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-readmes)
- [Markdown Guide](https://www.markdownguide.org/)
- [Shields.io](https://shields.io/) - バッジ作成サービス
- [Make a README](https://www.makeareadme.com/) - READMEテンプレート

---

**実例として参照したリポジトリ：**
- [claude-code-skills](https://github.com/Gaku52/claude-code-skills) - OSSライブラリ型
- [zenn-articles](https://github.com/Gaku52/zenn-articles) - Webアプリケーション型

これらは実在するリポジトリであり、本章の説明のために引用しています。
