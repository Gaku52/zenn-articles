---
title: "チーム運用・CI統合"
---

# チーム運用・CI統合

Claude Codeは個人の開発生産性を高めるだけでなく、チーム全体のワークフローを統一し、CI/CDパイプラインに組み込むことで、組織全体の開発効率を向上させることができます。本章では、チームでClaude Codeを運用するためのベストプラクティスと、CI統合の具体的な方法を解説します。

## チームでのCLAUDE.md運用

プロジェクトルートに配置する`CLAUDE.md`は、チーム全員が同じルールでClaude Codeを使うための重要なファイルです。これをgit管理することで、以下のメリットが得られます。

- チーム全員が同じコーディング規約・命名規則を遵守
- プロジェクト固有の制約事項を自動的に適用
- 新メンバーのオンボーディングが容易に

### CLAUDE.mdの基本構成

```markdown
# プロジェクト: EコマースAPI

## コーディング規約
- TypeScript strictモード必須
- ESLint/Prettierの設定に従う
- テストカバレッジ80%以上を維持

## 命名規則
- API関数: `handle{Action}`（例: handleCreateOrder）
- DBモデル: PascalCase（例: UserAccount）
- 環境変数: SCREAMING_SNAKE_CASE

## 制約事項
- データベース直接操作禁止（必ずORMを使用）
- 個人情報は必ずログマスキング
- 外部API呼び出しは必ずリトライ処理を実装

## セキュリティ
- パスワードは必ずbcryptでハッシュ化
- JWTトークンの有効期限は15分
- CORS設定は本番環境のドメインのみ許可
```

### ドメインごとのルール分割

大規模プロジェクトでは、`.claude/rules/`ディレクトリを使ってルールを分割すると管理しやすくなります。

```
.claude/
├── rules/
│   ├── backend.md      # バックエンド固有のルール
│   ├── frontend.md     # フロントエンド固有のルール
│   ├── database.md     # データベース操作のルール
│   └── security.md     # セキュリティポリシー
└── settings.json
```

`.claude/rules/backend.md`の例：

```markdown
# バックエンド開発ルール

## APIエンドポイント設計
- RESTful設計に従う
- エラーレスポンスは必ず`{ error: string, code: string }`形式
- ページネーションは`limit`/`offset`パラメータを使用

## エラーハンドリング
- 予期しないエラーは500、業務エラーは400番台
- ログには必ずrequestIdを含める
- スタックトレースは本番環境で非表示

## パフォーマンス
- N+1クエリを避ける
- データベースクエリは必ずINDEXを確認
- 重い処理はバックグラウンドジョブに移譲
```

### コードレビューでのCLAUDE.md活用

プルリクエスト時には、コードと一緒に`CLAUDE.md`の変更もレビューします。これにより、ルール変更がチーム全体に共有され、一貫性が保たれます。

```yaml
# .github/pull_request_template.md
## チェックリスト
- [ ] CLAUDE.mdのルールに準拠している
- [ ] 新しい制約事項がある場合、CLAUDE.mdに追記した
- [ ] テストカバレッジが80%以上
```

## チーム設定の共有

Claude Codeの設定ファイルを適切に管理することで、チーム全体で統一された開発環境を構築できます。

### プロジェクト設定とローカル設定の使い分け

```
.claude/
├── settings.json         # チーム共有（git管理）
└── settings.local.json   # 個人設定（gitignore）
```

`.claude/settings.json`（チーム共有）:

```json
{
  "permissions": {
    "deny": [
      ".env",
      "*.key",
      "secrets/**",
      "node_modules/**"
    ],
    "sandbox": {
      "enabled": true,
      "allowedCommands": ["npm", "git", "docker"]
    }
  },
  "preferredModel": "claude-opus-4-6",
  "hooks": {
    "beforeCommit": "npm run lint && npm test"
  }
}
```

`.claude/settings.local.json`（個人設定、gitignore）:

```json
{
  "editor": "cursor",
  "theme": "dark",
  "notifications": {
    "desktop": true
  }
}
```

`.gitignore`に追加:

```
.claude/settings.local.json
.claude/sessions/
```

### 権限設定でチーム全体のセキュリティポリシーを統一

`permissions`設定をプロジェクト全体で統一することで、誤って機密情報にアクセスするリスクを軽減できます。

```json
{
  "permissions": {
    "deny": [
      ".env*",
      "*.pem",
      "*.key",
      "secrets/**",
      "credentials/**",
      ".aws/**",
      ".ssh/**"
    ],
    "sandbox": {
      "enabled": true,
      "allowedCommands": [
        "npm",
        "pnpm",
        "yarn",
        "git",
        "docker",
        "kubectl"
      ],
      "deniedCommands": [
        "rm -rf /",
        "sudo",
        "chmod 777"
      ]
    }
  }
}
```

## Skills（カスタムコマンド）の共有

プロジェクト固有のワークフローをSkillsとして標準化し、チーム全体で共有できます。

### プロジェクトSkillsの配置

```
.claude/
├── skills/
│   ├── deploy.skill
│   ├── review.skill
│   └── generate-migration.skill
└── settings.json
```

`.claude/skills/deploy.skill`:

```yaml
name: deploy
description: アプリケーションをステージング環境にデプロイ
instructions: |
  1. `npm run build` でビルド
  2. `npm test` でテスト実行
  3. テストが成功したら `./scripts/deploy.sh staging` を実行
  4. デプロイ完了後、ヘルスチェックURLを確認
  5. 結果をSlackの#deploymentsチャンネルに報告
```

`.claude/skills/review.skill`:

```yaml
name: review
description: プルリクエストのコードレビュー
instructions: |
  1. 変更ファイルを確認し、CLAUDE.mdのルールに準拠しているか確認
  2. セキュリティ上の問題がないかチェック（SQLインジェクション、XSS、認証バイパスなど）
  3. パフォーマンス上の問題がないか確認（N+1クエリ、無限ループなど）
  4. テストが適切に書かれているか確認
  5. レビューコメントをMarkdown形式で出力
```

`.claude/skills/generate-migration.skill`:

```yaml
name: generate-migration
description: データベースマイグレーションファイルを生成
instructions: |
  1. ユーザーから変更内容をヒアリング（例: "usersテーブルにemailカラム追加"）
  2. `migrations/YYYYMMDDHHMMSS_description.sql`形式でファイル生成
  3. UP/DOWNマイグレーションを記述
  4. INDEXが必要な場合は自動追加
  5. ロールバック手順も含める
```

チーム全員が`/deploy`、`/review`、`/generate-migration`コマンドを使うことで、ワークフローが統一されます。

## MCPサーバーの共有

プロジェクト固有のMCPサーバーをチーム全体で共有することで、全員が同じツールセットを利用できます。

### プロジェクトスコープでのMCPサーバー管理

```bash
# プロジェクトスコープでMCPサーバーをインストール
claude mcp install @anthropic/mcp-server-postgres --scope project

# プロジェクトスコープでのサーバー一覧
claude mcp list --scope project
```

これにより、`.mcp.json`がプロジェクトルートに作成されます。

`.mcp.json`:

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-postgres"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-slack"],
      "env": {
        "SLACK_TOKEN": "${SLACK_TOKEN}"
      }
    }
  }
}
```

環境変数は`.env`で管理し、`.env`自体はgitignoreに追加します。

`.env.example`（gitに含める）:

```
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
SLACK_TOKEN=xoxb-your-token-here
```

## CI/CDでのClaude Code活用

GitHub ActionsなどのCIパイプラインでClaude Codeを実行することで、自動コードレビューやテスト生成が可能になります。

### GitHub ActionsでのClaude Code実行

`.github/workflows/claude-review.yml`:

```yaml
name: Claude Code Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 全履歴を取得（diffに必要）

      - name: Install Claude Code
        run: |
          curl -fsSL https://claude.ai/install.sh | bash
          echo "$HOME/.claude/bin" >> $GITHUB_PATH

      - name: Review PR
        run: |
          claude -p "以下のプルリクエストをレビューしてください。CLAUDE.mdのルールに準拠しているか、セキュリティ上の問題がないかを確認してください。" \
            --output-format json \
            --context "$(git diff origin/main...HEAD)" \
            > review.json
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

      - name: Post review comment
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const review = JSON.parse(fs.readFileSync('review.json', 'utf8'));
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## Claude Code Review\n\n${review.output}`
            });
```

### セキュリティスキャンの自動化

`.github/workflows/security-scan.yml`:

```yaml
name: Security Scan
on:
  push:
    branches: [main, develop]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Claude Code
        run: curl -fsSL https://claude.ai/install.sh | bash

      - name: Security scan
        run: |
          claude -p "このコードベースをスキャンし、以下のセキュリティ問題を検出してください:
            - SQLインジェクション
            - XSS脆弱性
            - 認証・認可のバイパス
            - 機密情報のハードコーディング
            - 安全でない暗号化
            問題が見つかった場合、ファイルパス・行番号・深刻度を報告してください。" \
            --output-format json > security-report.json
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

      - name: Upload report
        uses: actions/upload-artifact@v4
        with:
          name: security-report
          path: security-report.json

      - name: Fail if critical issues found
        run: |
          if jq -e '.issues[] | select(.severity == "critical")' security-report.json > /dev/null; then
            echo "Critical security issues found!"
            exit 1
          fi
```

### テスト生成の自動化

```yaml
name: Generate Missing Tests
on:
  schedule:
    - cron: '0 0 * * 0'  # 毎週日曜日

jobs:
  generate-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Claude Code
        run: curl -fsSL https://claude.ai/install.sh | bash

      - name: Generate tests
        run: |
          claude -p "カバレッジレポートを確認し、テストが不足しているファイルに対してテストを生成してください。" \
            --skill generate-tests
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

      - name: Create PR
        uses: peter-evans/create-pull-request@v5
        with:
          title: "自動生成: 不足しているテストを追加"
          branch: auto-generate-tests
          commit-message: "Add missing tests"
```

## Managed Settings（エンタープライズ）

エンタープライズ環境では、組織管理者が全ユーザーに適用する設定を一元管理できます。

### managed-settings.jsonの配置

組織管理者が`managed-settings.json`を配布し、ユーザーの`~/.claude/managed-settings.json`に配置します。

`managed-settings.json`:

```json
{
  "forceLoginMethod": "sso",
  "allowedMcpServers": [
    "@anthropic/mcp-server-*",
    "@company/mcp-server-*"
  ],
  "deniedMcpServers": [
    "*"
  ],
  "permissions": {
    "deny": [
      ".env*",
      "*.key",
      "*.pem",
      "secrets/**",
      ".ssh/**",
      ".aws/**"
    ],
    "sandbox": {
      "enabled": true,
      "allowedCommands": ["npm", "git", "docker", "kubectl"]
    }
  },
  "companyAnnouncements": [
    "社内ポリシー: Claude Codeで生成したコードは必ずレビューを受けてください。",
    "新機能: /review スキルでセキュリティチェックが可能になりました。"
  ],
  "hooks": {
    "beforeCommit": "npm run lint && npm test",
    "beforePush": "npm run build"
  }
}
```

### 設定項目の詳細

| 設定項目 | 説明 | 例 |
|---------|------|-----|
| `forceLoginMethod` | ログイン方法の制限 | `"sso"`（SSO強制） |
| `allowedMcpServers` | 許可するMCPサーバー（ワイルドカード対応） | `["@anthropic/mcp-server-*"]` |
| `deniedMcpServers` | 拒否するMCPサーバー | `["*"]`（デフォルト拒否） |
| `companyAnnouncements` | 起動時に表示するメッセージ | 社内ポリシー、新機能案内など |
| `permissions` | アクセス制御設定 | deny、sandbox設定 |
| `hooks` | 自動実行フック | beforeCommit、beforePushなど |

この設定により、組織全体で統一されたセキュリティポリシーを適用できます。

## リモートセッション

リモートセッション機能を使うと、デバイス間でClaude Codeのセッションを移動できます。

### Webセッションの作成

```bash
# ローカルからWebセッションを作成
claude --remote "Fix the login bug in auth.ts"
# → Web URLが生成される: https://claude.ai/session/abc123
```

ブラウザで上記URLを開くと、セッションの続きを確認・編集できます。

### リモートコントロール

```bash
# ローカルセッションをWeb上でリモートコントロール
claude --remote-control
# → Web URLが生成され、ブラウザで操作可能
```

デスクトップで作業していたセッションを、会議室のディスプレイに映したり、スマホで確認したりできます。

### Webセッションをローカルに移行

```bash
# WebセッションをローカルCLIに移行
claude --teleport https://claude.ai/session/abc123
```

外出先でブラウザで始めた作業を、帰宅後にローカルで続けることができます。

### チーム活用例

- コードレビュー時、レビュアーがWebセッションを開いて対話的にレビュー
- ペアプログラミングで、同じセッションを2人で共有
- 障害対応時、担当者のセッションを別メンバーが引き継ぐ

## セキュリティポリシーの策定

チーム運用では、セキュリティポリシーを明確に定義し、技術的に強制することが重要です。

### deny設定で機密ファイルへのアクセスを制限

`.claude/settings.json`:

```json
{
  "permissions": {
    "deny": [
      ".env*",
      "*.key",
      "*.pem",
      "*.crt",
      "secrets/**",
      "credentials/**",
      ".ssh/**",
      ".aws/**",
      ".gcp/**",
      "terraform.tfstate"
    ]
  }
}
```

これにより、Claude Codeは上記ファイルにアクセスできません。誤って機密情報を含むコードを生成するリスクを軽減できます。

### サンドボックスで実行環境を隔離

```json
{
  "permissions": {
    "sandbox": {
      "enabled": true,
      "allowedCommands": [
        "npm",
        "pnpm",
        "yarn",
        "git",
        "docker",
        "kubectl",
        "terraform"
      ],
      "deniedCommands": [
        "rm -rf /",
        "sudo",
        "chmod 777",
        "curl | bash",
        "wget -O- | sh"
      ],
      "allowedPaths": [
        "/workspace",
        "/tmp"
      ],
      "deniedPaths": [
        "/etc",
        "/root",
        "/var"
      ]
    }
  }
}
```

### フックで危険な操作をブロック

```json
{
  "hooks": {
    "beforeCommit": "npm run lint && npm test",
    "beforePush": "npm run build && npm run test:e2e",
    "beforeDeploy": "./scripts/security-check.sh"
  }
}
```

`scripts/security-check.sh`:

```bash
#!/bin/bash
set -e

echo "セキュリティチェック開始..."

# 機密情報のハードコーディングチェック
if grep -r "password\s*=\s*['\"]" src/; then
  echo "エラー: パスワードがハードコーディングされています"
  exit 1
fi

# 環境変数の検証
if [ ! -f .env ]; then
  echo "エラー: .envファイルが見つかりません"
  exit 1
fi

# 依存関係の脆弱性スキャン
npm audit --audit-level=high

echo "セキュリティチェック完了"
```

## まとめ

| 項目 | 内容 | 管理方法 |
|------|------|---------|
| **CLAUDE.md** | チーム全体のルール・制約事項 | git管理、コードレビューで確認 |
| **プロジェクト設定** | `.claude/settings.json` | git管理、チーム全体で統一 |
| **個人設定** | `.claude/settings.local.json` | gitignore、各自でカスタマイズ |
| **Skills** | `.claude/skills/` | git管理、チーム共通ワークフロー |
| **MCPサーバー** | `.mcp.json` | git管理、環境変数は`.env`で |
| **CI/CD** | GitHub Actions、自動レビュー | ANTHROPIC_API_KEYをSecrets管理 |
| **Managed Settings** | 組織管理者が配布 | エンタープライズ向け |
| **リモートセッション** | デバイス間でセッション移動 | URLベースで共有 |
| **セキュリティ** | deny、sandbox、hooks | プロジェクト全体で統一 |

## やってみよう！

1. **プロジェクトにCLAUDE.mdを追加しよう**
   - コーディング規約、命名規則、制約事項を記述
   - チームメンバーにレビューしてもらう

2. **`.claude/settings.json`を作成しよう**
   - permissions設定で機密ファイルをdeny
   - sandbox設定で安全なコマンドのみ許可

3. **チーム共通のSkillを作ろう**
   - `/deploy`、`/review`、`/generate-migration`などを定義
   - チーム全員が使えるようにドキュメント化

4. **GitHub ActionsでClaude Codeを実行しよう**
   - プルリクエストの自動レビューを実装
   - ANTHROPIC_API_KEYをSecretsに登録

5. **リモートセッションを試そう**
   - `claude --remote "簡単なタスク"`でWebセッション作成
   - ブラウザで開いて、デバイス間での移動を体験

次の章では、Claude Codeのコスト管理とエンタープライズでの運用最適化について解説します。大規模なチームでClaude Codeを導入する際のコスト見積もり、使用量の監視、予算管理の方法を学びましょう。
