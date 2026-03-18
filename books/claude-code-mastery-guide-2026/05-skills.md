---
title: "Skills -- カスタムワークフロー"
---

# Skills -- カスタムワークフロー

Claude Codeの真の力は、繰り返し行う作業をSkillsとしてパッケージ化し、いつでも呼び出せるようにできることです。この章では、カスタムワークフローを作成し、チーム全体で共有する方法を学びます。

## Skillsとは

Skillsは、Claude Codeで再利用可能なワークフローをパッケージ化し、`/skill-name`という形式で呼び出せる機能です。頻繁に行う作業パターンを一度定義しておけば、チームメンバー全員が同じ品質でタスクを実行できます。

### スラッシュコマンドとの違い

Claude Codeには、2種類のコマンド体系があります。

**組み込みスラッシュコマンド**は、Claude Codeに最初から搭載されている機能です。例えば`/commit`、`/review-pr`、`/help`などがこれに該当します。これらは開発チームによってメンテナンスされ、アップデートで改善されます。

**Skills**は、ユーザーが自由に定義できるカスタムワークフローです。プロジェクト固有のコーディング規約チェック、デプロイ前の確認手順、ドキュメント生成など、あなたのチーム特有のニーズに合わせて作成できます。

### 自動ロード機能

Skillsの強力な点は、Claudeが会話の文脈から自動的に適切なSkillをロードできることです。例えば、ユーザーが「コードを図解付きで説明して」と言った場合、`description`フィールドに「Explains code with visual diagrams」と書かれたSkillが自動的に読み込まれます。

明示的に`/explain-code`と入力する必要はありません。Claudeがユーザーの意図を理解し、最適なSkillを選択します。

## Skillの作成方法

実際にSkillを作成してみましょう。ここでは、コードを初心者にも分かりやすく説明する`explain-code`というSkillを例にします。

### ステップ1: ディレクトリ作成

まず、Skillを保存するディレクトリを作成します。

```bash
mkdir -p ~/.claude/skills/explain-code
```

### ステップ2: SKILL.mdファイル作成

次に、`SKILL.md`ファイルを作成します。このファイルがSkillの本体です。

```bash
cd ~/.claude/skills/explain-code
touch SKILL.md
```

### ステップ3: Skill定義の記述

`SKILL.md`に以下のような内容を記述します。

```markdown
---
name: explain-code
description: Explains code with visual diagrams and analogies
---

# Code Explanation Workflow

When explaining code, always include:

1. **Analogy**: Compare the code to everyday life concepts
   - Use familiar scenarios (cooking, driving, organizing a party)
   - Make technical concepts relatable

2. **Diagram**: ASCII art showing the flow
   - Use boxes and arrows to illustrate structure
   - Show data flow with clear labels

3. **Step-by-Step Breakdown**: Line-by-line explanation
   - What each section does
   - Why it's written this way
   - Common pitfalls to avoid

4. **Use Cases**: When to use this pattern
   - Appropriate scenarios
   - Alternatives and trade-offs

## Example Format

```
## Analogy
[Everyday comparison here]

## Visual Flow
[ASCII diagram here]

## Detailed Explanation
[Line-by-line breakdown]

## When to Use
[Scenarios and alternatives]
```
```

このSkillを保存すると、`/explain-code path/to/file.js`という形式で呼び出せるようになります。

## 保存場所とスコープ

Skillsには3つのスコープがあり、それぞれ保存場所が異なります。

### Personal Skills（個人用）

```
~/.claude/skills/<skill-name>/SKILL.md
```

あなたの全プロジェクトで使用できるSkillです。個人的なコーディングスタイルやよく使うテンプレートを保存するのに適しています。

**使用例**:
- 個人的なコードレビューチェックリスト
- 頻繁に使うコード生成パターン
- 学習用のドキュメント生成ツール

### Project Skills（プロジェクト固有）

```
.claude/skills/<skill-name>/SKILL.md
```

特定のプロジェクト内でのみ使用できるSkillです。プロジェクトのリポジトリにコミットすることで、チーム全体で共有できます。

**使用例**:
- プロジェクト固有のコーディング規約チェック
- 特定のフレームワークに最適化された実装パターン
- プロジェクト固有のドキュメントフォーマット

### Enterprise Skills（組織全体）

Enterprise版のClaude Codeでは、managed settings経由で組織全体にSkillsを配布できます。すべてのチームメンバーが同じSkillを使用できるため、組織全体で一貫した開発プロセスを維持できます。

**使用例**:
- セキュリティチェックリスト
- コンプライアンス確認手順
- 組織標準のドキュメント生成

## Frontmatterリファレンス

`SKILL.md`の先頭にあるYAML frontmatterで、Skillの動作を細かく制御できます。

### 基本フィールド

**name**（必須）

```yaml
name: explain-code
```

Skillの識別名です。`/explain-code`という形式で呼び出します。英数字とハイフンのみ使用できます。

**description**（推奨）

```yaml
description: Explains code with visual diagrams and analogies
```

Skillの機能を説明します。Claudeがこの説明を読み、ユーザーの要求に合致するか判断します。自動ロード機能の鍵となるフィールドです。

### 動作制御フィールド

**disable-model-invocation**

```yaml
disable-model-invocation: true
```

Skillの自動呼び出しを無効化します。ユーザーが明示的に`/skill-name`と入力した場合のみ実行されます。

**使用例**: デバッグ用Skill、破壊的な操作を行うSkillなど

**user-invocable**

```yaml
user-invocable: false
```

`false`に設定すると、Skillメニューに表示されなくなります。他のSkillから呼び出される内部Skillや、実験的なSkillに使用します。

### セキュリティと制限

**allowed-tools**

```yaml
allowed-tools: [Read, Grep, Glob]
```

Skillが使用できるツールを制限します。読み取り専用の操作に限定したい場合などに有効です。

```yaml
allowed-tools: [Bash, Read, Write]
```

**使用例**:
- 読み取り専用レビューSkill: `[Read, Grep, Glob]`
- ファイル操作Skill: `[Read, Write, Edit]`
- デプロイSkill: `[Bash]`

### 実行環境の指定

**model**

```yaml
model: claude-sonnet-4-6
```

特定のモデルでSkillを実行します。高速な処理にはSonnetを、複雑な推論が必要な場合はOpusを指定できます。

**context**

```yaml
context: fork
```

`fork`を指定すると、Skillをサブエージェントとして独立したコンテキストで実行します。メインの会話履歴を汚さずに大量の調査を行う場合に便利です。

**agent**

```yaml
agent: research
```

エージェントタイプを指定します。`research`、`coding`、`review`などが指定できます。

## 引数と変数

Skillsには強力な変数展開機能があり、動的なワークフローを作成できます。

### コマンドライン引数

ユーザーが`/skill-name arg1 arg2 arg3`と入力した場合、以下の変数が使用できます。

**$ARGUMENTS** — 全引数の文字列

```markdown
# Task: $ARGUMENTS
```

`/review-pr fix login bug`と呼び出すと、`fix login bug`が展開されます。

**$ARGUMENTS[N]** — N番目の引数（0始まり）

```markdown
# Reviewing file: $ARGUMENTS[0]
# Focus area: $ARGUMENTS[1]
```

`/review-pr src/auth.ts security`と呼び出すと、
- `$ARGUMENTS[0]` → `src/auth.ts`
- `$ARGUMENTS[1]` → `security`

**$N** — 省略形（$1, $2, ...）

```markdown
# File: $1
# Mode: $2
```

より簡潔に書けます。

### 実践例

```markdown
---
name: test-file
description: Run tests for a specific file
---

# Running Tests

Testing file: **$1**

Steps:
1. Run unit tests: `npm test $1`
2. Check coverage for $1
3. Verify all assertions pass

!`npm test $1 -- --coverage`
```

`/test-file src/utils.test.ts`と呼び出すと、`src/utils.test.ts`のテストを実行し、カバレッジを確認します。

### システム変数

**${CLAUDE_SESSION_ID}**

現在のセッションIDです。ログファイル名や一時ファイル名に使用できます。

```markdown
# Session: ${CLAUDE_SESSION_ID}
Logs: /tmp/claude-${CLAUDE_SESSION_ID}.log
```

**${CLAUDE_SKILL_DIR}**

現在実行中のSkillのディレクトリパスです。Skill内の補助ファイルを参照する際に使用します。

```markdown
# Loading templates from: ${CLAUDE_SKILL_DIR}/templates/
```

## 動的コンテキスト注入

Skillsの最も強力な機能の1つが、動的コンテキスト注入です。コマンド実行結果をSkillの実行時に注入できます。

### 基本構文

```markdown
!`command`
```

バッククォートで囲まれたコマンドが実行され、結果がSkillに注入されます。

### GitHubとの連携例

Pull Requestレビュー用のSkillを作成してみましょう。

```markdown
---
name: review-pr
description: Comprehensive PR review with context
---

# Pull Request Review

## PR Information

!`gh pr view --json title,author,body`

## Changed Files

!`gh pr diff --name-only`

## Full Diff

!`gh pr diff`

## Comments So Far

!`gh pr view --comments`

## Review Checklist

Based on the above context, review:
1. **Code Quality**: Clean, readable, follows conventions?
2. **Tests**: Are new features tested?
3. **Documentation**: Are changes documented?
4. **Breaking Changes**: Any API changes?
5. **Security**: Any security concerns?
```

このSkillを`/review-pr`で呼び出すと、現在のPRのすべての情報が自動的に取得され、包括的なレビューが実行されます。

### 実行時の動作

1. `!`command``部分が検出される
2. コマンドが実行される（例: `gh pr view --json title,author,body`）
3. 実行結果がSkillの該当箇所に挿入される
4. 完全に展開されたSkillがClaudeに渡される

### 複雑な例: デプロイ前チェック

```markdown
---
name: pre-deploy
description: Pre-deployment safety checks
---

# Pre-Deployment Checklist

## Current Branch Status

!`git status`

## Recent Commits

!`git log --oneline -5`

## Test Results

!`npm test 2>&1`

## Build Status

!`npm run build 2>&1`

## Environment Variables Check

!`grep -c "API_KEY" .env.production`

## Database Migration Status

!`npm run db:migrate:status`

---

**Review the above outputs and confirm:**
- [ ] All tests passing
- [ ] Build successful
- [ ] No uncommitted changes
- [ ] Environment variables configured
- [ ] Migrations up to date

Should we proceed with deployment?
```

このSkillは、デプロイ前に必要なすべてのチェックを自動実行し、結果を一覧表示します。

## バンドルスキル（組み込み）

Claude Codeには、すぐに使える強力なバンドルSkillsが含まれています。

### /batch — 大規模変更の並列実行

```bash
/batch "Add TypeScript type annotations to all utility functions"
```

**動作**:
1. プロジェクトを5〜30のユニットに分割
2. 各ユニットを独立したworktreeで並列処理
3. 変更を自動的にマージ

**使用例**:
- 大規模なリファクタリング（変数名変更、型追加など）
- コーディングスタイル統一
- ドキュメント一括生成

**注意点**: 各ユニットが独立して変更可能な場合に最適です。相互依存が強い変更には向きません。

### /simplify — 3並列レビューエージェント

```bash
/simplify auth-module
```

**動作**:
1. 3つのレビューエージェントを並列起動
2. 各エージェントが異なる観点でコードを分析
   - 可読性
   - パフォーマンス
   - ベストプラクティス準拠
3. 結果を統合し、改善提案を提示

**使用例**:
- 複雑なコードの簡素化
- レガシーコードの改善
- 新しいチームメンバーが理解しやすいコードへのリファクタリング

### /loop — 定期的なプロンプト実行

```bash
/loop 30 "Check if the build is complete and report status"
```

**動作**:
- 指定された間隔（秒）でプロンプトを繰り返し実行
- `Ctrl+C`で停止

**使用例**:
- 長時間実行されるビルドやテストの監視
- デプロイメントプロセスの追跡
- リソース使用状況の定期チェック

**実用例**:

```bash
/loop 60 "Check test coverage and alert if it drops below 80%"
```

### /claude-api — APIリファレンスロード

```bash
/claude-api
```

**動作**:
- Claude APIとAgent SDKの公式リファレンスをロード
- API呼び出しのサンプルコードを提供

**使用例**:
- Claude APIを使用するアプリケーション開発
- カスタムエージェントの実装
- API統合のトラブルシューティング

### /debug — デバッグログ読み取り

```bash
/debug "WebSocket connection keeps failing"
```

**動作**:
1. Claude Codeのデバッグログを読み取り
2. 指定された問題に関連するログエントリを抽出
3. 問題の原因を分析し、解決策を提案

**使用例**:
- Claude Code自体の動作不良調査
- パフォーマンス問題のデバッグ
- エラーメッセージの詳細確認

## 実践例: カスタムSkillの作成

実際に業務で使えるSkillをいくつか作成してみましょう。

### 例1: APIエンドポイント生成Skill

```markdown
---
name: new-api
description: Generate a new REST API endpoint with tests
---

# Creating New API Endpoint: $1

## Files to Create

1. **Route Handler**: `src/routes/$1.ts`
2. **Controller**: `src/controllers/$1.controller.ts`
3. **Service**: `src/services/$1.service.ts`
4. **Tests**: `tests/routes/$1.test.ts`

## Implementation Steps

### 1. Route Handler

```typescript
// src/routes/$1.ts
import express from 'express';
import { $1Controller } from '../controllers/$1.controller';

const router = express.Router();

router.get('/', $1Controller.getAll);
router.get('/:id', $1Controller.getById);
router.post('/', $1Controller.create);
router.put('/:id', $1Controller.update);
router.delete('/:id', $1Controller.delete);

export default router;
```

### 2. Controller (with error handling)

[Controller implementation template...]

### 3. Service Layer

[Service implementation template...]

### 4. Unit Tests

[Test implementation template...]

## Checklist

- [ ] Route handler created
- [ ] Controller implemented
- [ ] Service layer added
- [ ] Tests written and passing
- [ ] API documented in OpenAPI spec
```

使用方法:

```bash
/new-api users
```

これで`users`エンドポイントに必要な全ファイルが生成されます。

### 例2: セキュリティチェックSkill

```markdown
---
name: security-check
description: Run security audit on code changes
allowed-tools: [Read, Grep, Bash]
---

# Security Audit

## 1. Dependency Vulnerabilities

!`npm audit --json`

## 2. Hardcoded Secrets Scan

!`git diff main... | grep -E "(password|api_key|secret|token)" -i`

## 3. SQL Injection Risks

!`grep -r "SELECT.*\${" src/ --include="*.ts"`

## 4. XSS Vulnerabilities

!`grep -r "innerHTML\|dangerouslySetInnerHTML" src/ --include="*.tsx"`

## 5. Authentication Checks

!`grep -r "auth\|authenticate" src/ --include="*.ts" | head -20`

---

## Review Results

Based on the above scans:
1. Are there any critical vulnerabilities?
2. Are secrets properly managed?
3. Is user input properly sanitized?
4. Are authentication checks in place?

**Action Items**: [List specific issues to fix]
```

### 例3: ドキュメント生成Skill

```markdown
---
name: gen-docs
description: Generate comprehensive documentation for a module
---

# Documentation Generation for: $1

## Source Code Analysis

!`cat $1`

## Generate Documentation

Create a markdown document with:

### 1. Overview
- What this module does
- When to use it

### 2. API Reference
- All exported functions/classes
- Parameters and return types
- Examples

### 3. Usage Examples
- Basic usage
- Advanced scenarios
- Edge cases

### 4. Implementation Details
- Key algorithms
- Performance considerations
- Dependencies

### 5. Testing
- How to test
- Example test cases

Save to: `docs/$1.md`
```

使用方法:

```bash
/gen-docs src/utils/encryption.ts
```

## まとめ

| 項目 | 説明 |
|------|------|
| **Skills** | 再利用可能なワークフローをパッケージ化し、`/skill-name`で呼び出せる |
| **スコープ** | Personal（`~/.claude/skills/`）、Project（`.claude/skills/`）、Enterprise（managed settings） |
| **自動ロード** | `description`フィールドに基づき、Claudeが自動的に適切なSkillを選択 |
| **引数** | `$ARGUMENTS`、`$1`〜`$N`で動的なワークフローを実現 |
| **動的注入** | `!`command``でコマンド実行結果をリアルタイム注入 |
| **バンドルSkill** | `/batch`（並列処理）、`/simplify`（コードレビュー）、`/loop`（定期実行） |
| **セキュリティ** | `allowed-tools`で使用可能なツールを制限可能 |

## やってみよう！

自分だけのSkillを作成してみましょう。以下の課題に挑戦してください。

### レベル1: 基本Skill

あなたがよく行うコードレビューのチェックリストをSkillにしてみましょう。

```markdown
---
name: my-review
description: My personal code review checklist
---

# Code Review Checklist

[あなたのチェックリストを書いてください]
```

### レベル2: 引数付きSkill

特定のファイルのコード品質をチェックするSkillを作成してください。`$1`でファイルパスを受け取ります。

### レベル3: 動的注入Skill

`git diff`の結果を注入し、変更内容を分析するSkillを作成してください。

### レベル4: チーム共有Skill

プロジェクトの`.claude/skills/`に、チーム全体で使えるSkillを作成し、リポジトリにコミットしてみましょう。

---

次の章では、Claude Codeを使った実践的な開発ワークフロー（テスト駆動開発、リファクタリング、デバッグ）を学びます。Skillsと組み合わせることで、さらに効率的な開発が可能になります。
