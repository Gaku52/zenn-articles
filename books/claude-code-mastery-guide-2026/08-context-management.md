---
title: "コンテキスト管理"
---

# コンテキスト管理

Claude Codeを効果的に使いこなすには、コンテキスト管理の理解が欠かせません。この章では、LLMの「作業メモリ」であるコンテキストウィンドウの仕組みと、大規模なプロジェクトでも効率的に作業を進めるための戦略を学びます。

## コンテキストウィンドウとは

コンテキストウィンドウは、LLMの「作業メモリ」のようなものです。Claude Codeでの作業中、以下のすべての情報がコンテキストに蓄積されていきます。

- **会話履歴**: あなたとClaudeのやり取り
- **ファイル内容**: 読み込んだソースコード、設定ファイル、ドキュメント
- **ツール実行結果**: Bash、Grep、Globなどの出力
- **プロジェクトコンテキスト**: CLAUDE.mdやスコープ内のファイル

このコンテキストは、Claudeがあなたの意図を理解し、適切な判断を下すための重要な情報源です。しかし、コンテキストウィンドウには上限があり、それを超えると古い情報から順に失われていきます。

コンテキストが失われると、以下のような問題が発生します。

- 過去の会話内容を忘れる
- プロジェクトの構造や設計方針を見失う
- 既に読み込んだファイルを再度読む必要が生じる

そのため、コンテキストを効率的に管理することが、長時間の作業セッションでは特に重要になります。

## 1Mトークンコンテキストウィンドウ（2026年）

2026年、Claudeのコンテキストウィンドウは大幅に拡張されました。Opus 4.6とSonnet 4.6では、**1M（100万）トークン**のコンテキストウィンドウが一般提供されています。

### 主な特徴

**従来の5倍の拡張**

従来の200Kトークンから1Mトークンへと、5倍に拡張されました。これにより、より多くのファイルを同時に扱ったり、長い会話を継続したりできるようになりました。

**追加料金なし**

1Mトークンのコンテキストウィンドウは、追加料金なしで提供されます。トークン単価は従来と同じで、より大きなコンテキストを使っても費用は変わりません。

**プラン別の提供状況**

- **Max/Team/Enterpriseプラン**: 自動的に有効化されています
- **Proプラン**: `/extra-usage` コマンドでオプトインすることで利用可能です

### 実用例

1Mトークンのコンテキストウィンドウがあれば、以下のような作業が可能になります。

```bash
# 大規模なコードベースを一度にスコープに追加
claude code --add-dir src --add-dir tests --add-dir docs

# 複数の長いログファイルを同時に分析
claude code --add logs/error.log --add logs/access.log --add logs/system.log
```

以前は頻繁にコンテキストが溢れていた状況でも、1Mトークンがあれば余裕を持って作業できるようになりました。

## Compaction（圧縮）

コンテキストウィンドウが上限に近づいたとき、Claude Codeは**Compaction（圧縮）**という機能を使って、重要な情報を保持しながらコンテキストサイズを削減します。

### 自動Compaction

Claude Codeは、コンテキスト使用量を常に監視しています。上限に近づくと、自動的にCompactionが実行され、以下のような処理が行われます。

- 冗長な会話履歴を要約
- 不要になった古いツール実行結果を削除
- 重要なコード変更や意思決定は保持

1Mトークンコンテキストの導入後、自動Compactionの発生頻度は約15%減少しました。より長い作業セッションを、中断なく続けられるようになったのです。

### 手動Compaction

自動Compactionに加えて、任意のタイミングで手動でCompactionを実行することもできます。

```bash
# 基本的な手動Compaction
/compact

# カスタム指示を付けてCompaction
/compact Focus on the API changes and keep all database schema discussions

# 特定のトピックに絞ってCompaction
/compact Preserve only React component refactoring context
```

手動Compactionは、以下のような場面で有効です。

- 長いデバッグセッションの後、解決策が見つかったとき
- 設計議論が終わり、実装フェーズに移るとき
- コンテキストが雑多になってきたと感じたとき

カスタム指示を使うことで、あなたにとって重要な情報を優先的に保持できます。

### Compactionのベストプラクティス

**適切なタイミング**

- タスクの区切りで実行する（設計→実装、調査→修正など）
- エラー解決後、不要なログやスタックトレースを削除する
- 話題が変わる前に実行して、新しいトピックに集中する

**カスタム指示の活用**

```bash
# データベース移行作業の後
/compact Keep the migration plan and final schema, remove debugging steps

# API設計の後
/compact Preserve the agreed API contract and security requirements

# パフォーマンス改善の後
/compact Keep the optimization results and metrics, summarize the investigation process
```

Compactionは、コンテキストを「整理整頓」する作業だと考えると理解しやすいでしょう。重要な書類はファイリングして、不要なメモは捨てるイメージです。

## `/clear` vs `/compact`

コンテキストをリセットする方法として、`/clear`と`/compact`の2つがあります。それぞれの特徴を理解して、適切に使い分けましょう。

### `/clear`

完全に会話をクリアして、新しいセッションを開始します。

```bash
/clear
```

**特徴**
- すべての会話履歴が削除される
- ファイル内容やツール実行結果も消える
- 新セッションの開始と同等

**使うべき場面**
- まったく別のタスクを開始するとき
- プロジェクトを切り替えるとき
- 不要な情報が多すぎて混乱しているとき

### `/compact`

重要な情報を要約して残しながら、コンテキストを圧縮します。

```bash
/compact
```

**特徴**
- 重要な決定事項やコード変更は保持される
- 冗長な会話や古いツール結果は削除される
- 作業の継続性が保たれる

**使うべき場面**
- 長いセッションの途中でリフレッシュしたいとき
- コンテキストが肥大化してきたが、作業は続けたいとき
- タスクの区切りでクリーンアップしたいとき

### 使い分けの例

```bash
# ケース1: バグ修正が完了して、次は新機能開発
/compact Keep the bug fix approach and final solution

# ケース2: リファクタリング完了、次はドキュメント作成
/clear
# まったく別の作業なので /clear でリセット

# ケース3: 長いAPI設計議論の後、実装フェーズへ
/compact Preserve the API contract and architectural decisions
```

原則として、**話題を変えるなら `/clear`、続けるなら `/compact`** と覚えておくとよいでしょう。

## 大規模コードベースでの戦略

大規模なプロジェクトでは、コンテキストを効率的に管理することが成功の鍵です。以下の戦略を活用しましょう。

### 1. CLAUDE.mdで毎回の説明を削減

プロジェクト固有のルールや背景情報を`CLAUDE.md`に記載しておくことで、セッションごとに同じ説明を繰り返す必要がなくなります。

```markdown
# CLAUDE.md

## プロジェクト構造
- src/components: React components (Atomic Design)
- src/hooks: Custom hooks
- src/api: API client modules
- src/types: TypeScript type definitions

## コーディング規約
- すべてのコンポーネントはTypeScript + React.FCで記述
- propsはinterfaceで定義（typeは使わない）
- データフェッチはSWRを使用
- スタイリングはTailwind CSS

## テスト方針
- コンポーネントはReact Testing Libraryでテスト
- API層はMSWでモック
- カバレッジ目標80%以上
```

この情報は自動的にすべてのセッションで読み込まれるため、コンテキストを無駄に消費せずに済みます。

### 2. 必要なファイルだけをスコープに入れる

`--add-dir`や`--add`オプションを使う際は、本当に必要なファイルだけを含めましょう。

```bash
# 悪い例: すべてを追加
claude code --add-dir .

# 良い例: 作業に必要な部分だけ追加
claude code --add-dir src/auth --add tests/auth.test.ts
```

不要なファイルを追加すると、コンテキストが圧迫されるだけでなく、Claudeが関係ない情報に気を取られる原因にもなります。

### 3. サブエージェント（Task）に調査を委任

大規模な調査や探索的なタスクは、Taskツールを使ってサブエージェントに委任しましょう。

```bash
# メインセッションで
"調査タスクを別のエージェントに任せて、結果だけ報告してください"
```

サブエージェントは独立したコンテキストを持つため、メインセッションのコンテキストを圧迫しません。調査が完了したら、要約だけをメインセッションに戻せばよいのです。

### 4. 長いタスクはセッションを分割

1つのセッションで大きすぎるタスクを扱うと、コンテキストが溢れるリスクが高まります。以下のように分割しましょう。

**セッション1: 計画フェーズ**
```bash
claude code -n "user-auth-planning"
"認証機能の設計を行い、実装計画をドキュメント化してください"
```

**セッション2: 実装フェーズ**
```bash
claude code -n "user-auth-implementation" --add docs/auth-plan.md
"auth-plan.mdに基づいて認証機能を実装してください"
```

このように分割することで、各セッションで扱う情報量を適切に保ちながら、作業の継続性も確保できます。

## セッション設計のベストプラクティス

効果的なセッション設計は、コンテキスト管理の基本です。以下のプラクティスを実践しましょう。

### 1セッション = 1タスクの原則

1つのセッションでは、1つの明確なタスクに集中しましょう。

```bash
# 良い例
claude code -n "fix-login-bug"
claude code -n "add-password-reset"
claude code -n "update-api-docs"

# 悪い例
claude code -n "various-fixes"
# 複数の無関係なタスクを詰め込むと、コンテキストが散らかる
```

### 大きなタスクは計画と実装に分ける

大規模な機能開発は、以下のように2つのフェーズに分割すると効率的です。

**フェーズ1: 計画**
```bash
claude code -n "payment-integration-planning"
"Stripe決済統合の設計と実装計画を作成してください"
```

**フェーズ2: 実装**
```bash
claude code -n "payment-integration-impl" --add docs/payment-plan.md
"計画に基づいてStripe決済を実装してください"
```

計画フェーズで作成したドキュメントを実装フェーズで参照することで、コンテキストを効率的に使えます。

### セッションに名前を付ける

`-n`オプションでセッションに名前を付けると、後から識別しやすくなります。

```bash
# セッション開始時に名前を付ける
claude code -n "refactor-user-service"

# 中断したセッションを再開
claude code -r "refactor-user-service"
```

名前は以下のような形式が推奨されます。

- `fix-[issue-number]`: `fix-1234`
- `add-[feature-name]`: `add-dark-mode`
- `refactor-[component-name]`: `refactor-auth-flow`
- `investigate-[problem]`: `investigate-memory-leak`

### セッション再開の活用

作業を中断する場合は、`-r`オプションで後から再開できます。

```bash
# 作業を中断（セッション名: api-redesign）
# 後日再開
claude code -r "api-redesign"
```

再開時には、前回のコンテキストが復元されるため、スムーズに作業を続けられます。

## コンテキスト使用量の確認

コンテキストの使用状況を定期的に確認することで、適切なタイミングでCompactionや分割を判断できます。

### `/cost` コマンド

トークン使用量と費用を確認できます。

```bash
/cost
```

**出力例**
```
Session Cost Summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Input tokens:    45,234  ($0.135)
Output tokens:   12,456  ($0.187)
Total cost:               $0.322
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

この情報から、現在のセッションでどれだけのコンテキストを消費しているか把握できます。

### `/context` コマンド

コンテキストウィンドウの使用率を確認できます。

```bash
/context
```

**出力例**
```
Context Window Usage
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Used:      234,567 / 1,000,000 tokens (23%)
Available: 765,433 tokens

Breakdown:
- Conversation: 45,234 tokens
- Files:       156,789 tokens
- Tools:        32,544 tokens
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

使用率が70%を超えたら、Compactionやセッション分割を検討するタイミングです。

## まとめ

| 項目 | 説明 |
|------|------|
| **コンテキストウィンドウ** | LLMの作業メモリ。会話、ファイル、ツール結果が蓄積される |
| **1Mトークン** | Opus/Sonnet 4.6で提供。従来の5倍、追加料金なし |
| **自動Compaction** | 上限に近づくと自動で圧縮。重要情報は保持される |
| **手動Compaction** | `/compact [instructions]` で任意のタイミングで実行可能 |
| **`/clear`** | 会話を完全にクリア。話題を変えるときに使用 |
| **`/compact`** | 重要情報を残して圧縮。作業を続けるときに使用 |
| **CLAUDE.md活用** | 毎回の説明を削減し、コンテキストを節約 |
| **スコープ最小化** | 必要なファイルだけを `--add` で追加 |
| **Task委任** | 大規模調査はサブエージェントに委任してコンテキストを守る |
| **セッション分割** | 1セッション=1タスク。大きなタスクは計画と実装に分ける |
| **`/cost`** | トークン使用量と費用を確認 |
| **`/context`** | コンテキスト使用率を確認 |

## やってみよう！

### 課題1: コンテキスト使用量の確認

現在のセッションで `/context` と `/cost` を実行して、使用状況を確認してください。どの部分（会話/ファイル/ツール）が最もコンテキストを消費していますか？

### 課題2: Compactionの実践

長い会話を続けた後、以下の2つのCompactionを試してみましょう。

```bash
# 1. 基本的なCompaction
/compact

# 2. カスタム指示付きCompaction
/compact Keep only the final implementation decisions and code changes
```

それぞれの実行後、どのような情報が残り、何が削除されたか確認してください。

### 課題3: セッション設計

次の大きなタスクを、計画フェーズと実装フェーズの2つのセッションに分割してください。各セッションに適切な名前を付け、どのファイルをスコープに含めるか考えましょう。

```bash
# 例: ダークモード機能の追加
# フェーズ1: 計画
claude code -n "dark-mode-planning"

# フェーズ2: 実装
claude code -n "dark-mode-impl" --add docs/dark-mode-plan.md --add src/theme
```

---

次の章では、Claude Codeの拡張機能であるMCP（Model Context Protocol）の基礎を学びます。MCPを使うことで、外部サービスとの連携やカスタムツールの追加が可能になり、Claude Codeの可能性がさらに広がります。
