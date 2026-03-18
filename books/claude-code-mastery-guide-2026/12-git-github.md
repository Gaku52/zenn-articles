---
title: "Git・GitHub連携"
---

# Git・GitHub連携

Claude Codeは、Gitとの深い統合により、バージョン管理とコードレビューのワークフローを大幅に効率化します。この章では、コミット操作からPR作成、コードレビュー、GitHub Actions統合まで、Claude CodeとGit・GitHubを連携させる実践的な方法を学びます。

## コミット操作

### Claude Codeにコミットを依頼する方法

Claude Codeは、変更内容を自動的に分析し、適切なコミットメッセージを生成してコミットを実行できます。

```bash
# セッション中に変更を加えた後
claude -p "これまでの変更をコミットしてください"
```

Claude Codeは以下の手順でコミットを実行します。

1. `git status` で変更ファイルを確認
2. `git diff` で変更内容を分析
3. `git log` で既存のコミットメッセージスタイルを学習
4. コミットメッセージを生成
5. 関連ファイルを `git add` でステージング
6. `git commit` でコミット実行

### コミットメッセージの自動生成

Claude Codeは、変更内容とプロジェクトのコミット履歴を分析して、一貫性のあるコミットメッセージを生成します。

```bash
# 例: 認証機能を追加した場合のコミットメッセージ
Add user authentication with JWT tokens

Implements login/logout endpoints with bcrypt password hashing
and JWT-based session management. Adds middleware for protected
routes and user context injection.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
```

コミットメッセージは以下の原則で生成されます。

- 1行目に簡潔な要約（50文字以内）
- 2行目は空行
- 3行目以降に詳細説明
- 変更の「why」（理由）を重視
- プロジェクトの既存スタイルに合わせる

### attribution設定でCo-Authored-By表記

`~/.config/claude/config.json`で`attribution`を設定すると、コミットメッセージに自動的にClaude Codeの貢献を記録できます。

```json
{
  "attribution": {
    "commit": "🤖 Generated with Claude Code",
    "pr": ""
  }
}
```

この設定により、コミットメッセージの末尾に以下が追加されます。

```
Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
```

GitHubでは、Co-Authored-Byタグにより、コミットに複数の貢献者が表示されます。これは、AI支援による開発を透明化し、チームメンバーに変更の背景を明確に伝える上で有効です。

## PR操作

### gh pr createを使ったPR作成

Claude Codeは、GitHubの公式CLI `gh` を使ってプルリクエストを作成できます。

```bash
# PR作成を依頼
claude -p "この変更でPRを作成してください"
```

Claude Codeは以下の手順でPRを作成します。

1. `git status` と `git diff` で変更を確認
2. `git log [base-branch]...HEAD` でブランチの全コミット履歴を確認
3. PRタイトルと説明を生成
4. 必要に応じて `git push -u origin <branch>` でプッシュ
5. `gh pr create` でPR作成

```bash
# 実際の実行例
gh pr create --title "Add user authentication" --body "$(cat <<'EOF'
## Summary
- Implement JWT-based authentication system
- Add bcrypt password hashing
- Create login/logout endpoints

## Test plan
- [ ] Test login with valid credentials
- [ ] Test login with invalid credentials
- [ ] Test token expiration
- [ ] Test protected route access

🤖 Generated with Claude Code
EOF
)"
```

### PRの要約・レビュー依頼

既存のPRの内容を要約したり、レビューポイントを整理することもできます。

```bash
# PRの要約を依頼
claude -p "PR #123の変更内容を要約してください"

# レビュー依頼コメントの作成
claude -p "このPRのレビュー依頼コメントを書いてください。特にセキュリティ面を重点的に見てほしいです"
```

### --from-pr オプションでPRベースのセッション開始

特定のPRをベースにセッションを開始することで、そのPRに関連する作業に集中できます。

```bash
# PR #123をベースにセッション開始
claude --from-pr 123

# セッション内でPRの変更を分析
claude -p "このPRでどのファイルが変更されていますか？"
```

このオプションにより、Claude CodeはPRのdiffを自動的に読み込み、コンテキストを理解した状態で作業を開始します。

## コードレビュー

### Claude Codeにレビューを依頼

Claude Codeは、コードの品質、バグ、セキュリティ問題、パフォーマンス問題などを多角的にレビューします。

```bash
# 特定のPRをレビュー
claude -p "PR #123をレビューしてください。特にセキュリティとパフォーマンスの観点でチェックしてください"

# 特定のファイルをレビュー
claude -p "src/auth/login.tsをレビューしてください"
```

レビュー結果の例：

```markdown
## セキュリティ
- ❌ パスワードの最小文字数制限がない（推奨: 8文字以上）
- ⚠️  レート制限がなく、ブルートフォース攻撃に脆弱
- ✅ bcryptでハッシュ化済み（ソルトラウンド10は適切）

## パフォーマンス
- ⚠️  JWTトークンの検証が毎リクエストで実行されている
  → Redisでキャッシュすることを推奨

## コード品質
- ✅ TypeScriptの型定義が適切
- ⚠️  エラーハンドリングが不十分（ログに機密情報が含まれる可能性）
```

### /simplifyで3並列レビューエージェント実行

`/simplify` スキルを使うと、複数のエージェントが並列でコードを分析し、リファクタリング提案を行います。

```bash
# セッション中に
/simplify src/auth/
```

これにより、3つの独立したエージェントが以下の観点でコードを分析します。

1. **構造の単純化**: 複雑な関数を分割、ネストを削減
2. **重複コードの削減**: DRY原則に基づく共通化
3. **可読性の向上**: 変数名の改善、コメントの追加

各エージェントの提案は並列で実行され、最終的に統合されたリファクタリング案が提示されます。

### PRコメントの分析・対応

GitHub上のPRコメントを読み込んで、指摘事項に対応することもできます。

```bash
# PRコメントを取得して分析
claude -p "PR #123のコメントを確認して、指摘事項に対応してください"
```

Claude Codeは `gh api` コマンドでPRコメントを取得し、各指摘に対して適切な修正を行います。

```bash
# 実際の実行例
gh api repos/owner/repo/pulls/123/comments
# コメント内容を分析して修正を適用
```

## GitHub Actions統合

### CI/CDでClaude Codeを実行

GitHub Actionsのワークフロー内でClaude Codeを実行することで、自動化されたコードレビューやテスト生成を実現できます。

```yaml
# .github/workflows/code-review.yml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Claude Code
        run: |
          npm install -g @anthropic/claude-code
          echo "${{ secrets.ANTHROPIC_API_KEY }}" > ~/.anthropic_api_key

      - name: Code Review
        run: |
          claude -p "Review the changes in this PR for security, performance, and code quality issues" \
            --output-format json > review.json

      - name: Post Review Comment
        uses: actions/github-script@v7
        with:
          script: |
            const review = require('./review.json');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: review.output
            });
```

### 自動テスト生成

PRの変更内容に基づいて、自動的にテストケースを生成することもできます。

```yaml
# .github/workflows/auto-test.yml
name: Auto Test Generation

on:
  pull_request:
    types: [opened]

jobs:
  generate-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Claude Code
        run: |
          npm install -g @anthropic/claude-code
          echo "${{ secrets.ANTHROPIC_API_KEY }}" > ~/.anthropic_api_key

      - name: Generate Tests
        run: |
          claude -p "Generate unit tests for the changes in this PR" \
            --output-format json > tests.json

      - name: Create Test Files
        run: |
          # tests.jsonから生成されたテストコードを抽出してファイル作成
          node scripts/create-tests-from-json.js

      - name: Commit Tests
        run: |
          git config user.name "Claude Code Bot"
          git config user.email "bot@claudecode.ai"
          git add tests/
          git commit -m "Add auto-generated tests"
          git push
```

### PRに自動コメント

レビュー結果やテスト結果を自動的にPRにコメントすることで、開発者へのフィードバックループを高速化できます。

```yaml
- name: Post Review Comment
  uses: actions/github-script@v7
  with:
    script: |
      const fs = require('fs');
      const review = JSON.parse(fs.readFileSync('review.json', 'utf8'));

      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: `## 🤖 Claude Code Review\n\n${review.output}\n\n---\n*Generated by Claude Code*`
      });
```

## ブランチ操作

### ブランチ作成・切り替え

Claude Codeは、機能開発やバグ修正のために新しいブランチを作成し、適切な名前を提案します。

```bash
# ブランチ作成を依頼
claude -p "ユーザー認証機能を実装するためのブランチを作成してください"
```

Claude Codeは以下のように実行します。

```bash
git checkout -b feature/user-authentication
```

ブランチ名は、プロジェクトの命名規則（`feature/`, `fix/`, `refactor/`など）に従って自動的に決定されます。

### マージコンフリクトの解消

マージコンフリクトが発生した場合、Claude Codeは両方の変更内容を理解した上で、適切な解消方法を提案します。

```bash
# コンフリクト解消を依頼
claude -p "mainブランチとのマージコンフリクトを解消してください"
```

Claude Codeは以下の手順で対応します。

1. `git status` でコンフリクトファイルを特定
2. コンフリクトマーカー（`<<<<<<<`, `=======`, `>>>>>>>`）を含むファイルを読み込み
3. 両方の変更の意図を理解
4. 適切な統合方法を決定
5. ファイルを編集してコンフリクトを解消
6. `git add` でマーク

```bash
# 実際の実行例
# コンフリクトしたファイルを編集
git add src/config.ts
git commit -m "Resolve merge conflict in config.ts"
```

### リベース操作

コミット履歴を整理するためのリベース操作もサポートしています。

```bash
# リベースを依頼
claude -p "mainブランチの最新変更をリベースしてください"
```

```bash
# 実際の実行例
git fetch origin
git rebase origin/main
```

注意: Claude Codeは `-i` フラグを使った対話的なリベースは実行できません（対話的入力が必要なため）。単純なリベースのみサポートしています。

## git worktree統合

### claude -w feature-authで隔離セッション

`git worktree` を使うと、同じリポジトリの複数のブランチを同時に操作できます。Claude Codeは `-w` オプションでworktree内のセッションを開始できます。

```bash
# worktreeを作成
git worktree add ../myproject-feature-auth feature-auth

# worktree内でセッション開始
claude -w feature-auth
```

この機能により、メインブランチでの作業を中断せずに、別のブランチで緊急のバグ修正を行うことができます。

### 並列作業時のコンフリクト防止

複数のworktreeで同時に作業する場合、Claude Codeは各worktreeを独立したセッションとして扱います。

```bash
# ターミナル1: メイン機能開発
cd ~/myproject
claude -r "main-feature"

# ターミナル2: バグ修正
cd ~/myproject-hotfix
claude -r "hotfix-login"
```

各セッションは独立しているため、以下のメリットがあります。

- ファイルのロック競合が発生しない
- 各セッションのコンテキストが混ざらない
- 並列で異なるタスクを進行できる

## セッション管理とGit

### セッション名にブランチ名を使う

セッション名とブランチ名を一致させることで、作業の追跡が容易になります。

```bash
# ブランチ名でセッション開始
git checkout -b feature/oauth-integration
claude -r "feature/oauth-integration"
```

これにより、セッションログとGitのコミット履歴が紐づき、後から作業内容を振り返る際に便利です。

### -r "feature-name"で再開

中断した作業を再開する際、セッション名を指定することで、前回のコンテキストを引き継げます。

```bash
# 翌日、同じセッションを再開
claude -r "feature/oauth-integration" -p "昨日の続きから実装を進めてください"
```

セッション履歴により、Claude Codeは以下を記憶しています。

- 前回までの実装内容
- 未解決の課題
- 次に実装すべき機能
- テスト状況

### --fork-sessionでセッション分岐

既存のセッションを基に、新しいセッションを分岐させることができます。

```bash
# 既存のセッションを分岐
claude --fork-session "feature/oauth-integration" -r "feature/oauth-google"
```

これは、以下のようなケースで有効です。

- 複数のOAuthプロバイダを並列で実装
- 同じ機能の異なるアプローチを試す
- 実験的な変更を別セッションで試す

## 安全なGit操作のルール

Claude Codeは、危険なGit操作から開発者を保護するための複数の安全機構を持っています。

### force pushの防止

`git push --force` は履歴を書き換えるため、デフォルトでは実行されません。

```bash
# Claude Codeは以下を拒否します
git push --force origin main
```

ユーザーが明示的に依頼した場合のみ、警告を表示した上で実行します。

```bash
claude -p "force pushしてください（理由: ローカルでリベース済み）"
# → 警告メッセージを表示してから実行
```

### 本番ブランチへの直接コミット防止

`main` や `master` ブランチへの直接コミットは、プルリクエストフローを推奨するため、警告が表示されます。

```bash
# mainブランチで作業中
claude -p "この変更をコミットしてください"
# → 「mainブランチへの直接コミットは推奨されません。feature/ブランチを作成しますか？」
```

### .env等の機密ファイルのコミット防止

`.env`, `credentials.json`, `.pem` などの機密ファイルは、自動的にステージングから除外されます。

```bash
# Claude Codeは以下を検出して警告
git add .env
# → 「.envファイルは機密情報を含む可能性があります。コミットしますか？」
```

`.gitignore` に追加されていない機密ファイルがある場合、Claude Codeは `.gitignore` への追加を提案します。

### permissions設定でgit操作を制御

`~/.config/claude/config.json` で、Git操作の権限を細かく制御できます。

```json
{
  "permissions": {
    "git": {
      "commit": true,
      "push": "prompt",
      "force_push": false,
      "delete_branch": "prompt"
    }
  }
}
```

設定値：

- `true`: 自動的に許可
- `false`: 常に拒否
- `"prompt"`: 実行前に確認プロンプトを表示

## まとめ

| 機能 | コマンド例 | 用途 |
|------|-----------|------|
| **コミット** | `claude -p "コミットしてください"` | 変更を自動的にコミット |
| **PR作成** | `claude -p "PRを作成してください"` | `gh pr create`でPR作成 |
| **PRベースセッション** | `claude --from-pr 123` | 特定のPRをコンテキストに読み込み |
| **コードレビュー** | `claude -p "PR #123をレビューしてください"` | AI支援レビュー |
| **並列レビュー** | `/simplify src/` | 3エージェントで並列分析 |
| **GitHub Actions統合** | CI/CDで `claude -p "..."` | 自動レビュー・テスト生成 |
| **ブランチ作成** | `claude -p "機能ブランチを作成"` | 適切な名前でブランチ作成 |
| **コンフリクト解消** | `claude -p "コンフリクトを解消"` | マージコンフリクトの自動解決 |
| **worktree統合** | `claude -w feature-auth` | 並列作業の隔離 |
| **セッション再開** | `claude -r "feature/oauth"` | ブランチ名でセッション管理 |
| **安全な操作** | `permissions`設定 | force push等の制御 |

## やってみよう！

1. **自動コミットを試す**: 小さな変更を加えて、Claude Codeにコミットを依頼してください。生成されたコミットメッセージを確認し、プロジェクトのスタイルに合っているか確認しましょう。

2. **PRを作成する**: 新しいブランチで機能を実装し、Claude Codeに `gh pr create` でPRを作成させてください。PRの要約とテストプランが自動生成されることを確認しましょう。

3. **コードレビューを依頼**: 既存のPRをClaude Codeにレビューさせ、セキュリティやパフォーマンスの観点での指摘を受けてください。

4. **GitHub Actionsを設定**: `.github/workflows/code-review.yml` を作成し、PRが作成されたときに自動的にClaude Codeがレビューするように設定してください。

5. **セッション管理を実践**: ブランチ名でセッションを開始し、翌日に同じセッション名で再開してください。コンテキストが引き継がれることを確認しましょう。

次の章では、Claude Codeのセキュリティとプライバシー設定について詳しく学びます。機密情報の保護、アクセス制御、監査ログの管理など、安全に運用するための重要な設定を解説します。
