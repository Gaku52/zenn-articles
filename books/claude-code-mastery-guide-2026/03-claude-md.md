---
title: "CLAUDE.md -- プロジェクトの頭脳"
---

# CLAUDE.md -- プロジェクトの頭脳

Claude Codeは、ファイルを読み、コードを分析し、タスクを実行します。しかし、コードだけでは推測できない情報があります。「このプロジェクトでは2スペースインデントを使う」「コミットメッセージはConventional Commits形式で書く」「テストは必ず通してからコミットする」――こうしたルールや規約を毎回口頭で伝えるのは非効率です。

この問題を解決するのが **CLAUDE.md** です。この特別なファイルは、セッション開始時にClaudeが自動的に読み込み、プロジェクトのコンテキストとして保持します。本章では、CLAUDE.mdの役割、配置場所、書き方のベストプラクティス、そして関連機能について詳しく解説します。

## CLAUDE.mdとは

CLAUDE.mdは、Claudeに対してプロジェクト固有のコンテキストを提供するMarkdown形式のファイルです。セッション開始時に自動的に読み込まれ、以下のような情報をClaudeに伝えます。

- **プロジェクトの概要**: 技術スタック、アーキテクチャ、主要な依存関係
- **開発ルール**: コーディング規約、命名規則、インデントスタイル
- **ワークフロー**: テストの実行方法、コミットルール、デプロイ手順
- **プロジェクト構造**: 主要ディレクトリの役割、特殊なファイルの意味
- **注意事項**: 触ってはいけないファイル、特別な設定が必要な箇所

コードそのものからは読み取れない「人間の意図」や「チームの決め事」を明示的に伝えることで、Claudeはより正確で一貫性のある提案を行えるようになります。

## ファイルの配置場所とスコープ

CLAUDE.mdは、配置場所によって適用範囲（スコープ）が異なります。Claude Codeは、優先度の高い順に以下の場所からCLAUDE.mdを探します。

### 1. プロジェクトレベル（最優先）

```
./CLAUDE.md
または
./.claude/CLAUDE.md
```

プロジェクトルートに配置するCLAUDE.mdです。このファイルは通常、Gitリポジトリにコミットされ、チーム全体で共有されます。

**適用範囲**: そのプロジェクト内でのみ有効

**使用例**:
- プロジェクト固有のコーディング規約
- アーキテクチャの説明
- チーム全員が守るべきルール

**メリット**:
- チーム全員が同じコンテキストを共有できる
- バージョン管理によって変更履歴を追跡できる
- 新メンバーのオンボーディングが容易になる

### 2. ユーザーレベル

```
~/.claude/CLAUDE.md
```

個人のホームディレクトリに配置するCLAUDE.mdです。すべてのプロジェクトに共通して適用される個人的な設定を記述します。

**適用範囲**: そのユーザーが開くすべてのプロジェクト

**使用例**:
- 個人的なコーディングスタイルの好み
- よく使うツールの設定
- プライベートなメモやリマインダー

**注意点**:
- プロジェクトレベルのCLAUDE.mdがある場合、そちらが優先されます
- チームメンバーには共有されないため、個人的な設定のみを記述してください

### 3. 組織レベル（macOS）

```
/Library/Application Support/ClaudeCode/CLAUDE.md
```

macOSのシステムレベルで配置するCLAUDE.mdです。組織全体で共通のポリシーやガイドラインを適用する場合に使用します。

**適用範囲**: そのマシンで開くすべてのプロジェクト（すべてのユーザー）

**使用例**:
- 組織全体のセキュリティポリシー
- 全社的なコーディング規約
- コンプライアンス要件

**優先順位**:
プロジェクトレベル > ユーザーレベル > 組織レベルの順に優先されます。

## 書き方のベストプラクティス

CLAUDE.mdは「Claudeに読ませる」ためのファイルですが、人間も読めるように書くことが重要です。以下のベストプラクティスに従うことで、効果的なCLAUDE.mdを作成できます。

### 1. 200行以下を推奨

CLAUDE.mdが長すぎると、Claudeが重要な情報を見落としたり、遵守率が低下したりします。目安として **200行以下** に抑えることを推奨します。

**判断基準**:
各行について「これを削除するとClaudeがミスするか？」と自問してください。答えが「No」なら削除を検討しましょう。

### 2. 具体的な指示を書く

曖昧な指示ではなく、具体的で測定可能な指示を書きましょう。

**悪い例**:
```markdown
- コードは適切にフォーマットしてください
```

**良い例**:
```markdown
- 2スペースインデント、セミコロンなし
- Prettier設定: .prettierrc に従う
```

**悪い例**:
```markdown
- テストを書いてください
```

**良い例**:
```markdown
- 新機能には必ずテストを追加: `npm run test`
- カバレッジ80%以上を維持
```

### 3. Markdownヘッダーと箇条書きで構造化

Claudeはマークダウン形式の構造を理解します。見出しと箇条書きを活用して、情報を整理しましょう。

```markdown
# プロジェクト概要
TypeScript + React + Next.js のWebアプリケーション

# 開発ルール
- テストは vitest を使用: `npm run test`
- コミットメッセージは Conventional Commits 形式
- 2スペースインデント、セミコロンなし

# アーキテクチャ
- src/app/ — Next.js App Router
- src/components/ — 共通コンポーネント
- src/lib/ — ユーティリティ・APIクライアント

# 注意事項
- src/legacy/ は触らない（レガシーコード、削除予定）
- .env.example は常に最新に保つ
```

### 4. コードスニペットではなくファイル参照を使う

CLAUDE.mdにコードスニペットを大量に貼り付けるのは避けましょう。代わりに、ファイルパスと行番号を参照します。

**悪い例**:
```markdown
# ESLint設定
以下の設定を使用します：
```json
{
  "extends": ["next/core-web-vitals"],
  "rules": {
    "no-console": "warn"
  }
}
```
（20行のJSONが続く...）
```

**良い例**:
```markdown
# ESLint設定
.eslintrc.json に従う
特に注意: no-console はwarnレベル
```

### 5. 「なぜ」を説明する

ルールだけでなく、その理由を簡潔に説明すると、Claudeはより適切な判断ができます。

```markdown
# テストルール
- E2Eテストは Playwright を使用
  理由: Cypressから移行中、新規テストはPlaywrightで書く
- ユニットテストは vitest を使用
  理由: Jestより高速、ESM対応
```

## @インポート機能

CLAUDE.mdから他のファイルを参照することで、情報の重複を避け、保守性を高めることができます。

### 基本的な@インポート

```markdown
# プロジェクト概要
@README

# 依存関係
@package.json

# API仕様
@docs/api-spec.md
```

`@ファイル名` の形式で記述すると、Claudeはそのファイルの内容を読み込んで参照します。

### 外部ファイルの参照

ホームディレクトリや絶対パスのファイルも参照できます。

```markdown
# 個人的なコーディングルール
@~/.claude/my-coding-style.md

# チーム共通のセキュリティガイドライン
@/Users/Shared/team-docs/security-guidelines.md
```

### @インポートのメリット

- **情報の一元管理**: README.mdに書かれた情報を再記述する必要がない
- **保守性の向上**: 元ファイルを更新すれば、CLAUDE.mdも自動的に最新になる
- **チーム共有**: 複数プロジェクトで共通のルールファイルを参照できる

### 注意点

- 参照先ファイルが大きすぎると、CLAUDE.mdの実質的な長さが増えます
- 循環参照（AがBを参照し、BがAを参照）は避けてください

## `.claude/rules/` による構造化

大規模プロジェクトでは、CLAUDE.mdが肥大化しがちです。そのような場合、`.claude/rules/` ディレクトリを使ってルールを複数ファイルに分割できます。

### ディレクトリ構造の例

```
.claude/
├── CLAUDE.md                # メインファイル（概要と全体ルール）
└── rules/
    ├── code-style.md        # コーディングスタイル
    ├── testing.md           # テストルール
    ├── security.md          # セキュリティガイドライン
    ├── api-development.md   # API開発ルール
    └── deployment.md        # デプロイメントルール
```

### パス固有ルール

YAMLフロントマターの `paths` フィールドを使うと、特定のディレクトリやファイルパターンにのみ適用されるルールを定義できます。

**例: API開発ルール** (`.claude/rules/api-development.md`)

```yaml
---
paths:
  - "src/api/**/*.ts"
  - "src/routes/**/*.ts"
---

# API開発ルール

## エンドポイント命名
- RESTful原則に従う
- 複数形を使用: `/users`, `/posts`
- バージョニング: `/v1/users`

## エラーハンドリング
- すべてのエンドポイントで共通のエラー形式を返す
- HTTP ステータスコードを適切に使用
  - 200: 成功
  - 400: クライアントエラー
  - 500: サーバーエラー

## 認証
- すべてのAPIエンドポイントでJWT認証を実装
- `src/lib/auth.ts` のミドルウェアを使用
```

**例: テストルール** (`.claude/rules/testing.md`)

```yaml
---
paths:
  - "**/*.test.ts"
  - "**/*.spec.ts"
  - "tests/**/*"
---

# テストルール

## ユニットテスト
- vitest を使用
- AAA パターン（Arrange, Act, Assert）に従う
- テストファイル名: `*.test.ts`

## E2Eテスト
- Playwright を使用
- テストファイル名: `*.spec.ts`
- `tests/e2e/` に配置

## カバレッジ
- 最低80%を維持
- `npm run test:coverage` で確認
```

### メインCLAUDE.mdからの参照

メインのCLAUDE.mdでは、分割したルールファイルを参照します。

```markdown
# プロジェクト概要
TypeScript + Next.js のWebアプリケーション

# 詳細ルール
以下のルールファイルを参照してください：
- @.claude/rules/code-style.md
- @.claude/rules/testing.md
- @.claude/rules/security.md
- @.claude/rules/api-development.md
- @.claude/rules/deployment.md
```

### 構造化のメリット

- **保守性**: 関心ごとに分割され、編集が容易
- **パフォーマンス**: パス固有ルールにより、関係ないファイル編集時にルールが読み込まれない
- **チーム協業**: 担当者ごとにルールファイルを分担できる

## Auto Memory（自動記憶）

CLAUDE.mdはプロジェクト全体に共通するルールを記述するファイルですが、**Auto Memory** は作業の進行に応じてClaudeが自動的に学習・記録するメモリシステムです。

### Auto Memoryの仕組み

Claudeは作業を進める中で、以下のような情報を自動的にMEMORY.mdに記録します。

- プロジェクトの進捗状況
- 発見した問題や解決策
- 重要な決定事項
- 繰り返し言及される概念やパターン

### 保存場所

```
~/.claude/projects/<project-hash>/memory/MEMORY.md
```

プロジェクトごとに一意のハッシュ値が割り当てられ、そのディレクトリにMEMORY.mdが保存されます。

### MEMORY.mdの最初の200行が自動ロード

セッション開始時、ClaudeはMEMORY.mdの最初の200行を自動的に読み込みます。これにより、前回のセッションで得た知識を引き継ぐことができます。

### `/memory` コマンド

`/memory` コマンドを使うと、Auto Memoryを管理できます。

```
/memory                # メモリの内容を表示
/memory clear          # メモリをクリア
/memory add "情報"     # 手動でメモリに追加
```

### CLAUDE.mdとAuto Memoryの使い分け

| 項目 | CLAUDE.md | Auto Memory |
|------|-----------|-------------|
| 目的 | 不変のルール・規約 | 進行中の作業コンテキスト |
| 更新頻度 | 低（プロジェクト設定変更時） | 高（セッションごと） |
| 共有 | チーム全体（Git管理） | 個人ローカル |
| 内容 | コーディング規約、アーキテクチャ | 進捗メモ、発見事項、TODO |

**例**:

CLAUDE.md（不変のルール）:
```markdown
# テストルール
- すべての新機能にテストを追加
- カバレッジ80%以上を維持
```

MEMORY.md（進行中の作業）:
```markdown
# 現在の作業
- ユーザー認証機能を実装中
- JWT トークンの有効期限を24時間に設定した
- 次回: リフレッシュトークンの実装

# 発見した問題
- bcrypt のバージョン競合 → 5.1.0 に固定で解決
```

## `/init` コマンド

CLAUDE.mdを一から書くのは大変です。`/init` コマンドを使うと、Claudeがプロジェクトを分析し、適切なCLAUDE.mdを自動生成してくれます。

### 基本的な使い方

```
/init
```

プロジェクトルートで `/init` コマンドを実行すると、Claudeは以下を行います。

1. package.json、README.md、tsconfig.json などの主要ファイルを読み込む
2. プロジェクトの技術スタックや構造を分析
3. 適切なCLAUDE.mdを生成（または改善提案）

### 既存のCLAUDE.mdがある場合

既にCLAUDE.mdが存在する場合、Claudeは削除せず、改善提案を行います。

```
既存のCLAUDE.mdを分析しました。以下の改善を提案します：

1. TypeScript設定について言及がありません
2. テストコマンドが古くなっています（jest → vitest）
3. 新しく追加されたAPIディレクトリの説明がありません

これらの改善を適用しますか？
```

### `/init` の実行例

実際に `/init` コマンドを実行すると、以下のようなCLAUDE.mdが生成されます。

```markdown
# プロジェクト概要
TypeScript + React + Next.js 14 (App Router) のWebアプリケーション

技術スタック:
- Next.js 14.2.3
- React 18.3.1
- TypeScript 5.4.5
- Tailwind CSS 3.4.1
- Prisma (PostgreSQL)

# 開発ルール
- Node.js 20.x を使用
- パッケージマネージャー: pnpm
- 2スペースインデント、セミコロンなし
- Prettier設定: .prettierrc に従う
- ESLint設定: .eslintrc.json に従う

# スクリプト
- `pnpm dev` — 開発サーバー起動（http://localhost:3000）
- `pnpm build` — 本番ビルド
- `pnpm test` — テスト実行（vitest）
- `pnpm lint` — Lint実行

# アーキテクチャ
- src/app/ — Next.js App Router（ページとレイアウト）
- src/components/ — 共通Reactコンポーネント
- src/lib/ — ユーティリティ・APIクライアント
- prisma/ — データベーススキーマとマイグレーション

# データベース
- PostgreSQL (Prisma ORM)
- マイグレーション: `pnpm prisma migrate dev`
- Prisma Studio: `pnpm prisma studio`

# コミットルール
- Conventional Commits 形式
  - feat: 新機能
  - fix: バグ修正
  - docs: ドキュメント更新
  - refactor: リファクタリング

# 注意事項
- .env.local は Git管理外（.env.example を参照）
- src/legacy/ は触らない（削除予定）
```

### `/init` のメリット

- **時短**: 手動で書くより圧倒的に速い
- **正確**: 実際のファイルを分析するため、正確な情報が得られる
- **最新**: プロジェクトの変更に応じて再実行できる

## CLAUDE.mdの実例

実際のプロジェクトで使われているCLAUDE.mdの例をいくつか紹介します。

### 例1: シンプルなNext.jsプロジェクト

```markdown
# プロジェクト概要
TypeScript + Next.js 14 のブログアプリケーション

# 開発ルール
- 2スペースインデント、セミコロンなし
- コミット前に `npm run lint` を実行
- コミットメッセージは Conventional Commits 形式

# アーキテクチャ
- src/app/ — Next.js App Router
- src/components/ — 共通コンポーネント
- src/lib/ — ユーティリティ

# テスト
- `npm run test` — vitest でユニットテスト
- カバレッジ80%以上を維持

# 注意事項
- .env.local は Git管理外
```

### 例2: マイクロサービス構成のプロジェクト

```markdown
# プロジェクト概要
Node.js マイクロサービス（Express + TypeScript + MongoDB）

# アーキテクチャ
- services/auth/ — 認証サービス
- services/api/ — メインAPIサービス
- services/notification/ — 通知サービス
- shared/ — 共通ライブラリ

# 開発ルール
- 各サービスは独立してビルド・テスト可能
- 共通コードは shared/ に配置
- Docker Compose で全サービスを起動: `docker-compose up`

# テスト
- ユニットテスト: `npm run test`
- 統合テスト: `npm run test:integration`
- E2Eテスト: `npm run test:e2e`

# デプロイ
- Kubernetes (GKE)
- CI/CD: GitHub Actions
- 本番デプロイは main ブランチへのマージで自動実行
```

### 例3: オープンソースプロジェクト

```markdown
# プロジェクト概要
オープンソースのCLIツール（TypeScript + Commander.js）

# 開発ルール
- Node.js 18.x 以上
- パッケージマネージャー: npm
- Prettier + ESLint で自動フォーマット
- コミット前に `npm run format && npm run lint` を実行

# コントリビューションガイドライン
- CONTRIBUTING.md を参照
- プルリクエストには必ずテストを追加
- Breaking Changes は CHANGELOG.md に記載

# テスト
- `npm run test` — Jest でユニットテスト
- `npm run test:watch` — ウォッチモード
- カバレッジ90%以上を目標

# リリース
- セマンティックバージョニング
- リリースノート: GitHub Releases で公開
```

## まとめ

| 項目 | 内容 |
|------|------|
| **CLAUDE.mdの役割** | セッション開始時にClaudeが読み込む、プロジェクトのコンテキストファイル |
| **配置場所** | プロジェクト（`./.claude/CLAUDE.md`）、ユーザー（`~/.claude/CLAUDE.md`）、組織（`/Library/Application Support/ClaudeCode/CLAUDE.md`） |
| **優先順位** | プロジェクト > ユーザー > 組織 |
| **推奨行数** | 200行以下（長すぎると遵守率が低下） |
| **書き方** | 具体的で測定可能な指示、Markdown構造化、ファイル参照を活用 |
| **@インポート** | `@README`, `@package.json`, `@~/.claude/my-rules.md` で他ファイルを参照 |
| **`.claude/rules/`** | 大規模プロジェクト向けにルールを複数ファイルに分割、パス固定ルールも可能 |
| **Auto Memory** | Claudeが自動的に学習・記録するメモリ（進行中の作業コンテキスト） |
| **`/memory` コマンド** | メモリの表示、クリア、手動追加 |
| **`/init` コマンド** | CLAUDE.mdの自動生成、既存ファイルの改善提案 |
| **CLAUDE.md vs MEMORY.md** | CLAUDE.md = 不変のルール、MEMORY.md = 進行中の作業 |

## やってみよう！

### 課題1: 既存プロジェクトにCLAUDE.mdを作成

あなたが開発中のプロジェクトで `/init` コマンドを実行し、CLAUDE.mdを生成してみましょう。生成されたCLAUDE.mdを確認し、以下の点をチェックしてください。

- プロジェクトの技術スタックが正しく記載されているか
- 開発ルールやコーディング規約が反映されているか
- 不要な情報や冗長な記述がないか

必要に応じて手動で編集し、チームメンバーと共有できる形に整えてみましょう。

### 課題2: `.claude/rules/` でルールを構造化

CLAUDE.mdが100行を超えているプロジェクトがあれば、`.claude/rules/` ディレクトリを作成し、以下のようにルールを分割してみましょう。

1. `code-style.md` — コーディングスタイルとフォーマットルール
2. `testing.md` — テスト関連のルール
3. `deployment.md` — デプロイメントとCI/CD関連のルール

それぞれのファイルにYAMLフロントマターで `paths` を設定し、特定のディレクトリにのみ適用されるようにしてください。

### 課題3: Auto Memoryを活用した作業管理

1週間の開発作業中、毎日 `/memory` コマンドでMEMORY.mdの内容を確認してみましょう。Claudeがどのような情報を自動的に記録しているかを観察し、以下を試してください。

- `/memory add "明日実装するタスク: ユーザー通知機能"` で手動メモを追加
- 1週間後、MEMORY.mdを見返して、どのような情報が蓄積されたかを振り返る
- 不要な情報があれば `/memory clear` でクリアし、新しいフェーズに備える

---

次の章では、Claude Codeの強力な機能である **タスク管理とプランニング** について解説します。複雑な開発タスクを構造化し、効率的に進める方法を学びましょう。
