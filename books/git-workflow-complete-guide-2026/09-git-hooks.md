---
title: "Git Hooks活用ガイド - コードの品質を自動化"
---

# Git Hooks活用ガイド

## Git Hooksとは

Git Hooksは、Gitの特定のアクション（コミット、プッシュ等）が実行される前後に自動的にスクリプトを実行する仕組みです。コードの品質チェック、テスト実行、メッセージ検証などを自動化できます。

**想定される効果:**
- コミットメッセージエラー削減: **-95%**（pre-commit hook導入後）
- テストし忘れ防止: **-100%**（pre-push hook）
- コードスタイル違反削減: **-88%**（自動フォーマット）
- リリース前の問題発見: **+400%**（各種hook活用）

**想定ROI（月間200コミット想定）:**
```
導入コスト: 初回2時間 + メンテ月30分 = 2.5時間

削減時間:
  コミットメッセージ修正: 5分 × 20回 = 100分
  テスト実行忘れ: 30分 × 4回 = 120分
  コードスタイル修正: 10分 × 15回 = 150分
  合計: 370分（6.2時間）

→ 月間3.7時間の節約（初月でも黒字）
```

## Git Hooksの種類

### クライアントサイドHooks

| Hook | タイミング | 用途 | 使用頻度 |
|------|----------|------|---------|
| **pre-commit** | コミット前 | Lint、フォーマット、テスト | 最頻 |
| **prepare-commit-msg** | メッセージ編集前 | テンプレート挿入 | 中 |
| **commit-msg** | メッセージ保存時 | メッセージ検証 | 高 |
| **post-commit** | コミット後 | 通知、ログ | 低 |
| **pre-push** | プッシュ前 | 全テスト実行 | 高 |
| **pre-rebase** | rebase前 | 保護 | 低 |

### サーバーサイドHooks

| Hook | タイミング | 用途 |
|------|----------|------|
| **pre-receive** | プッシュ受信前 | ブランチ保護 |
| **update** | 各ブランチ更新前 | 権限チェック |
| **post-receive** | プッシュ受信後 | デプロイ、通知 |

## 実装パターン

### パターン1: Lint & Format（pre-commit）

最も一般的なパターン。コミット前にコードスタイルをチェック・修正します。

```bash
#!/bin/sh
# .git/hooks/pre-commit

echo "🔍 Running pre-commit checks..."

# ステージングされたファイルのみチェック
FILES=$(git diff --cached --name-only --diff-filter=ACMR | grep -E '\.(ts|tsx|js|jsx)$')

if [ -z "$FILES" ]; then
  exit 0
fi

# ESLint実行
echo "📝 Running ESLint..."
npx eslint $FILES

if [ $? -ne 0 ]; then
  echo "❌ ESLint failed. Please fix the errors."
  exit 1
fi

# Prettier実行
echo "✨ Running Prettier..."
npx prettier --write $FILES

if [ $? -ne 0 ]; then
  echo "❌ Prettier failed."
  exit 1
fi

# フォーマット後のファイルをステージング
git add $FILES

echo "✅ Pre-commit checks passed!"
exit 0
```

**想定効果:**
- コードスタイル違反: **100件/月 → 12件/月** (-88%)
- レビュー時のスタイル指摘: **-95%**

### パターン2: コミットメッセージ検証（commit-msg）

Conventional Commits形式を強制します。

```bash
#!/bin/sh
# .git/hooks/commit-msg

commit_msg_file=$1
commit_msg=$(cat "$commit_msg_file")

# Conventional Commits正規表現
pattern="^(feat|fix|docs|style|refactor|perf|test|chore|ci|revert)(\(.+\))?: .+$"

if ! echo "$commit_msg" | grep -qE "$pattern"; then
  echo "❌ Invalid commit message format"
  echo ""
  echo "Format: <type>(<scope>): <subject>"
  echo ""
  echo "Types: feat, fix, docs, style, refactor, perf, test, chore, ci, revert"
  echo ""
  echo "Example:"
  echo "  feat(auth): add Google OAuth login"
  echo "  fix(ui): resolve button alignment issue"
  echo ""
  exit 1
fi

# subjectの長さチェック（50文字以内推奨）
subject=$(echo "$commit_msg" | head -n 1)
subject_length=${#subject}

if [ $subject_length -gt 72 ]; then
  echo "⚠️  Warning: Subject is too long ($subject_length chars)"
  echo "   Recommended: 50 chars, max: 72 chars"
  echo ""
fi

echo "✅ Commit message is valid"
exit 0
```

**想定効果:**
- 不正なメッセージ: **38件/月 → 2件/月** (-95%)
- CHANGELOG自動生成精度: **+100%**

### パターン3: 包括的テスト（pre-push）

プッシュ前に全テストを実行し、品質を保証します。

```bash
#!/bin/sh
# .git/hooks/pre-push

echo "🧪 Running tests before push..."

# ブランチ名取得
branch=$(git rev-parse --abbrev-ref HEAD)

# mainブランチへのpushは特に厳格
if [ "$branch" = "main" ] || [ "$branch" = "master" ]; then
  echo "⚠️  Pushing to $branch - running full test suite"

  # 全テスト実行
  npm run test:all

  if [ $? -ne 0 ]; then
    echo "❌ Tests failed. Cannot push to $branch"
    exit 1
  fi

  # ビルドチェック
  npm run build

  if [ $? -ne 0 ]; then
    echo "❌ Build failed. Cannot push to $branch"
    exit 1
  fi
else
  # featureブランチは軽量チェック
  npm run test:changed

  if [ $? -ne 0 ]; then
    echo "❌ Tests failed"
    echo "💡 Tip: Use --no-verify to skip (not recommended)"
    exit 1
  fi
fi

echo "✅ All checks passed. Pushing..."
exit 0
```

**想定効果:**
- 壊れたコードのpush: **15件/月 → 0件/月** (-100%)
- CI/CD失敗率: **22% → 3%** (-86%)

### パターン4: 自動チケット番号挿入（prepare-commit-msg）

ブランチ名からチケット番号を抽出し、コミットメッセージに自動挿入します。

```bash
#!/bin/sh
# .git/hooks/prepare-commit-msg

commit_msg_file=$1
commit_source=$2

# マージコミット等はスキップ
if [ "$commit_source" = "merge" ] || [ "$commit_source" = "squash" ]; then
  exit 0
fi

# ブランチ名からチケット番号を抽出
# 例: feature/PROJ-123-add-login → PROJ-123
branch=$(git rev-parse --abbrev-ref HEAD)
ticket=$(echo "$branch" | grep -oE '[A-Z]+-[0-9]+')

if [ -n "$ticket" ]; then
  # 既にチケット番号が含まれていない場合のみ追加
  if ! grep -q "$ticket" "$commit_msg_file"; then
    # メッセージの末尾に追加
    echo "" >> "$commit_msg_file"
    echo "Refs: $ticket" >> "$commit_msg_file"
    echo "✅ Added ticket number: $ticket"
  fi
fi

exit 0
```

**想定効果:**
- チケット番号の記載漏れ: **45% → 0%** (-100%)
- コミット-チケット追跡性: **+100%**

## Huskyによる管理

### Huskyとは

Huskyは、Git Hooksを簡単に管理・共有するためのツールです。`.git/hooks`に直接書く代わりに、プロジェクトに含められます。

### セットアップ

```bash
# インストール
npm install --save-dev husky

# 初期化
npx husky install

# package.jsonに自動セットアップスクリプト追加
npm pkg set scripts.prepare="husky install"
```

### Hook追加

```bash
# pre-commitフック追加
npx husky add .husky/pre-commit "npm run lint"

# commit-msgフック追加
npx husky add .husky/commit-msg 'npx --no -- commitlint --edit $1'

# pre-pushフック追加
npx husky add .husky/pre-push "npm test"
```

**ファイル構成:**
```
.husky/
├── _/
│   └── husky.sh
├── pre-commit
├── commit-msg
└── pre-push
```

### 実装例: .husky/pre-commit

```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

echo "🔍 Running pre-commit checks..."

# lint-stagedで変更ファイルのみチェック
npx lint-staged

# TypeScript型チェック
echo "📘 Type checking..."
npx tsc --noEmit

if [ $? -ne 0 ]; then
  echo "❌ Type check failed"
  exit 1
fi

echo "✅ Pre-commit checks passed!"
```

### lint-staged連携

変更されたファイルのみに処理を実行します。

**package.json:**
```json
{
  "lint-staged": {
    "*.{ts,tsx,js,jsx}": [
      "eslint --fix",
      "prettier --write",
      "git add"
    ],
    "*.{css,scss}": [
      "stylelint --fix",
      "prettier --write",
      "git add"
    ],
    "*.md": [
      "prettier --write",
      "git add"
    ]
  }
}
```

**想定される効果:**
- Hook実行時間: **30秒 → 3秒** (-90%)（変更ファイルのみ処理）
- 開発者満足度: **+85%**（待ち時間短縮）

## 高度なパターン

### パターン5: 段階的チェック（pre-commit）

コミットサイズに応じて実行するチェックを変更します。

```bash
#!/bin/sh
# .git/hooks/pre-commit

# 変更ファイル数取得
files_changed=$(git diff --cached --name-only | wc -l)

echo "📊 Files changed: $files_changed"

if [ $files_changed -le 5 ]; then
  # 小規模コミット: 全チェック
  echo "🔍 Small commit - running full checks"
  npm run lint
  npm run test:unit
  npm run test:e2e
elif [ $files_changed -le 20 ]; then
  # 中規模コミット: 軽量チェック
  echo "⚡ Medium commit - running quick checks"
  npm run lint
  npm run test:unit
else
  # 大規模コミット: 最低限のチェック
  echo "⚠️  Large commit detected ($files_changed files)"
  echo "💡 Consider splitting into smaller commits"
  npm run lint
fi
```

### パターン6: ブランチ保護（pre-push）

特定ブランチへの直接pushを防ぎます。

```bash
#!/bin/sh
# .git/hooks/pre-push

protected_branches="main master develop"
current_branch=$(git rev-parse --abbrev-ref HEAD)

for branch in $protected_branches; do
  if [ "$current_branch" = "$branch" ]; then
    echo "❌ Direct push to $branch is not allowed"
    echo "💡 Please create a feature branch and submit a PR"
    echo ""
    echo "To create a feature branch:"
    echo "  git checkout -b feature/your-feature-name"
    echo ""
    exit 1
  fi
done

exit 0
```

### パターン7: セキュリティスキャン（pre-commit）

機密情報の混入を防ぎます。

```bash
#!/bin/sh
# .git/hooks/pre-commit

echo "🔒 Checking for sensitive data..."

# 秘密鍵パターン
patterns=(
  "BEGIN RSA PRIVATE KEY"
  "BEGIN DSA PRIVATE KEY"
  "BEGIN EC PRIVATE KEY"
  "AWS_SECRET_ACCESS_KEY"
  "password\s*=\s*['\"][^'\"]+['\"]"
  "api_key\s*=\s*['\"][^'\"]+['\"]"
)

files=$(git diff --cached --name-only)

for file in $files; do
  for pattern in "${patterns[@]}"; do
    if grep -qE "$pattern" "$file" 2>/dev/null; then
      echo "❌ Potential secret found in $file"
      echo "   Pattern: $pattern"
      echo ""
      echo "💡 Remove sensitive data before committing"
      exit 1
    fi
  done
done

# .envファイルチェック
if echo "$files" | grep -q "\.env$"; then
  echo "⚠️  Warning: .env file detected"
  echo "   Make sure it's in .gitignore"

  if ! grep -q "\.env" .gitignore 2>/dev/null; then
    echo "❌ .env is not in .gitignore!"
    exit 1
  fi
fi

echo "✅ No sensitive data detected"
exit 0
```

**想定効果:**
- 秘密鍵流出: **3件/年 → 0件/年** (-100%)
- セキュリティインシデント: **-67%**

## トラブルシューティング

### 問題1: Hookが実行されない

**原因:**
- Hookファイルに実行権限がない

**解決策:**
```bash
# 実行権限付与
chmod +x .git/hooks/pre-commit

# または全Hook
chmod +x .git/hooks/*

# Huskyの場合
chmod +x .husky/pre-commit
```

### 問題2: Hookが遅すぎる

**症状:**
```
コミットに30秒以上かかる
開発者がストレス
```

**対策:**
```bash
# 並列実行
npm run lint & npm run test &
wait

# 変更ファイルのみ処理（lint-staged）
npx lint-staged

# キャッシュ活用
npx eslint --cache $FILES
```

**想定改善:**
- 実行時間: **28秒 → 4秒** (-86%)

### 問題3: Hookをスキップしたい（緊急時）

```bash
# 一時的にスキップ
git commit --no-verify -m "hotfix: urgent fix"

# または
git commit -n -m "hotfix: urgent fix"

# pushでもスキップ可能
git push --no-verify
```

**注意:** 緊急時のみ使用し、後で必ず修正してください。

### 問題4: チーム全員で共有できない

**問題:** `.git/hooks`はバージョン管理されない

**解決策:**

**方法1: Husky使用（推奨）**
```bash
npm install --save-dev husky
npx husky install
# .huskyディレクトリがGitで共有される
```

**方法2: スクリプトディレクトリ作成**
```bash
# プロジェクトルート
mkdir -p scripts/git-hooks

# Hook作成
cat > scripts/git-hooks/pre-commit << 'EOF'
#!/bin/sh
npm run lint
EOF

# インストールスクリプト
cat > scripts/install-hooks.sh << 'EOF'
#!/bin/sh
cp scripts/git-hooks/* .git/hooks/
chmod +x .git/hooks/*
EOF

# 実行
sh scripts/install-hooks.sh
```

**package.json:**
```json
{
  "scripts": {
    "postinstall": "sh scripts/install-hooks.sh"
  }
}
```

## 実践的な設定例

### スタートアップ向け（高速重視）

```json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged",
      "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
    }
  },
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": ["eslint --fix", "git add"]
  }
}
```

**実行時間:** 平均2-5秒

### エンタープライズ向け（品質重視）

```json
{
  "husky": {
    "hooks": {
      "pre-commit": "npm run pre-commit:full",
      "commit-msg": "commitlint -E HUSKY_GIT_PARAMS",
      "pre-push": "npm run pre-push:full"
    }
  },
  "scripts": {
    "pre-commit:full": "lint-staged && npm run type-check && npm run test:unit",
    "pre-push:full": "npm run test:all && npm run build"
  }
}
```

**実行時間:** pre-commit 10-15秒、pre-push 30-60秒

## まとめ

### Git Hooks導入の想定効果

| 項目 | 改善率 | 具体的な数値 |
|------|--------|------------|
| コミットメッセージエラー削減 | -95% | 38件/月 → 2件/月 |
| テスト実行忘れ防止 | -100% | 15件/月 → 0件/月 |
| コードスタイル違反削減 | -88% | 100件/月 → 12件/月 |
| CI/CD失敗率削減 | -86% | 22% → 3% |
| セキュリティインシデント削減 | -67% | リスク大幅減 |

### 推奨される最小限のHook構成

```
必須:
├── pre-commit: Lint + Format
└── commit-msg: メッセージ検証

推奨:
└── pre-push: テスト実行

オプション:
├── prepare-commit-msg: チケット番号自動挿入
└── pre-commit: セキュリティスキャン
```

### ベストプラクティス

1. **軽量に保つ** - 実行時間は5秒以内目標
2. **明確なエラーメッセージ** - 修正方法を示す
3. **スキップ可能にする** - 緊急時は`--no-verify`
4. **チーム全員で共有** - Huskyを使用
5. **段階的に導入** - まずpre-commitから

Git Hooksを活用することで、コードの品質が自動的に保証され、レビューの負担が大幅に軽減されます。初期投資は小さく、ROIは非常に高いため、全てのプロジェクトで導入を推奨します。

次の章では、**チーム開発ワークフロー**として、複数人での効率的な開発手法を学びます。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
