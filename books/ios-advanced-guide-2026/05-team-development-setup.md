---
title: "チーム開発環境の構築"
---

# チーム開発環境の構築

チーム開発では、環境の統一とワークフローの標準化が開発効率に直結します。本章では、チームメンバー全員が同じ環境で開発できるセットアップ手法を解説します。

## 開発環境の統一

### 必要なツールのバージョン管理

チーム全体で使用するツールのバージョンを明示します。

**.tool-versions ファイル:**

```plaintext
# .tool-versions
xcode 15.0
swift 5.9
ruby 3.2.0
nodejs 20.0.0
```

### Homebrew による依存関係管理

Brewfile を使用して、必要なツールを一元管理します。

**Brewfile:**

```ruby
# Brewfile

# Xcode Command Line Tools
# xcode-selectでインストール済み前提

# Ruby環境（Fastlane用）
brew "rbenv"
brew "ruby-build"

# Node.js環境（ツール用）
brew "nodenv"

# iOS開発ツール
brew "swiftlint"
brew "swiftformat"
brew "xcbeautify"

# Git関連
brew "git-lfs"
brew "gh"

# その他
brew "cocoapods"
brew "carthage"
```

インストール：

```bash
brew bundle install
```

## セットアップスクリプトの作成

新メンバーが簡単に環境構築できるスクリプトを用意します。

**scripts/setup.sh:**

```bash
#!/bin/bash

set -e

echo "🚀 iOS開発環境のセットアップを開始します"

# Homebrewのインストール確認
if ! command -v brew &> /dev/null; then
    echo "📦 Homebrewをインストールしています..."
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
fi

# Homebrewの更新
echo "📦 Homebrewを更新しています..."
brew update

# Brewfileから依存関係をインストール
echo "📦 開発ツールをインストールしています..."
brew bundle install

# Xcodeのバージョン確認
echo "🔍 Xcodeのバージョンを確認しています..."
xcodebuild -version

# Ruby環境のセットアップ
if [ -f ".ruby-version" ]; then
    echo "💎 Ruby環境をセットアップしています..."
    rbenv install --skip-existing
    rbenv rehash
fi

# Bundlerのインストール
if ! command -v bundle &> /dev/null; then
    echo "💎 Bundlerをインストールしています..."
    gem install bundler
fi

# Ruby依存関係のインストール
if [ -f "Gemfile" ]; then
    echo "💎 Ruby依存関係をインストールしています..."
    bundle install
fi

# CocoaPodsのセットアップ
if [ -f "Podfile" ]; then
    echo "📦 CocoaPodsの依存関係をインストールしています..."
    bundle exec pod install
fi

# Git hooksのセットアップ
echo "🪝 Git hooksをセットアップしています..."
cp scripts/pre-commit .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit

# .envファイルのコピー
if [ ! -f ".env" ] && [ -f ".env.example" ]; then
    echo "📝 .envファイルを作成しています..."
    cp .env.example .env
    echo "⚠️  .envファイルを編集して、必要な環境変数を設定してください"
fi

echo "✅ セットアップが完了しました！"
echo ""
echo "次のステップ:"
echo "1. .envファイルを編集"
echo "2. Xcodeでプロジェクトを開く"
echo "3. Scheme を選択してビルド"
```

実行方法：

```bash
chmod +x scripts/setup.sh
./scripts/setup.sh
```

## Git ワークフローの標準化

### ブランチ戦略

Git Flow を採用した例です。

```plaintext
main        : 本番環境にデプロイされるブランチ
develop     : 開発の中心となるブランチ
feature/*   : 新機能開発用
hotfix/*    : 緊急修正用
release/*   : リリース準備用
```

### ブランチ命名規則

```plaintext
feature/機能名
例: feature/user-authentication

hotfix/問題の説明
例: hotfix/login-crash

release/バージョン
例: release/1.2.0
```

### Gitコミットメッセージ規約

Conventional Commits を採用します。

```plaintext
<type>(<scope>): <subject>

<body>

<footer>
```

**Type の種類:**

```plaintext
feat     : 新機能
fix      : バグ修正
docs     : ドキュメントのみの変更
style    : コードの意味に影響しない変更（空白、フォーマットなど）
refactor : リファクタリング
perf     : パフォーマンス改善
test     : テストの追加・修正
chore    : ビルドプロセスやツールの変更
```

**コミットメッセージの例:**

```plaintext
feat(auth): add OAuth 2.0 authentication

- Implement OAuth 2.0 authorization code flow
- Add token refresh mechanism
- Store tokens in Keychain

Closes #123
```

### Git Hooks

コミット前に自動チェックを実行します。

**scripts/pre-commit:**

```bash
#!/bin/bash

echo "🔍 Pre-commit checks..."

# SwiftLint の実行
if which swiftlint >/dev/null; then
    swiftlint --strict
    if [ $? -ne 0 ]; then
        echo "❌ SwiftLint failed. Please fix the issues before committing."
        exit 1
    fi
fi

# SwiftFormat の実行
if which swiftformat >/dev/null; then
    swiftformat . --lint
    if [ $? -ne 0 ]; then
        echo "❌ SwiftFormat failed. Run 'swiftformat .' to fix formatting."
        exit 1
    fi
fi

# ステージングされたファイルのみをチェック
SWIFT_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep ".swift$")

if [ -n "$SWIFT_FILES" ]; then
    echo "✅ Swift files are properly formatted"
fi

echo "✅ All pre-commit checks passed!"
```

## コーディング規約の統一

### SwiftLint 設定

**.swiftlint.yml:**

```yaml
# SwiftLint Configuration

disabled_rules:
  - trailing_whitespace

opt_in_rules:
  - empty_count
  - explicit_init
  - first_where
  - modifier_order
  - redundant_nil_coalescing
  - sorted_first_last
  - closure_spacing
  - explicit_type_interface

excluded:
  - Pods
  - .build
  - DerivedData
  - fastlane

# Rule Configuration
line_length:
  warning: 120
  error: 200
  ignores_comments: true

type_body_length:
  warning: 300
  error: 500

file_length:
  warning: 500
  error: 1000

identifier_name:
  min_length:
    warning: 2
  max_length:
    warning: 40
    error: 50
  excluded:
    - id
    - URL
    - url

force_cast: warning
force_try: warning

custom_rules:
  no_print:
    name: "No Print Statements"
    regex: "print\\("
    message: "Logger.debug() を使用してください"
    severity: warning

  no_force_unwrap:
    name: "No Force Unwrapping"
    regex: "!(?![\\s}\\]\\)])"
    message: "Force unwrapping は避けてください"
    severity: warning
```

### SwiftFormat 設定

**.swiftformat:**

```plaintext
# SwiftFormat Configuration

--swiftversion 5.9

# Indentation
--indent 4
--indentcase false
--tabwidth 4
--xcodeindentation enabled

# Wrapping
--maxwidth 120
--wraparguments before-first
--wrapparameters before-first
--wrapcollections before-first

# Spacing
--trimwhitespace always
--commas inline
--semicolons inline

# Options
--self remove
--importgrouping testable-bottom
--stripunusedargs closure-only

# Rules
--enable isEmpty
--enable sortedImports
--enable redundantSelf
--disable redundantReturn
```

## コードレビューのガイドライン

### Pull Request テンプレート

**.github/pull_request_template.md:**

```markdown
## 概要
<!-- 変更内容の簡潔な説明 -->

## 変更の種類
- [ ] 新機能
- [ ] バグ修正
- [ ] リファクタリング
- [ ] ドキュメント更新
- [ ] その他

## 関連Issue
<!-- 関連するIssue番号 -->
Closes #

## 変更内容
<!-- 具体的な変更内容をリスト形式で記載 -->
-
-

## スクリーンショット
<!-- UI変更がある場合、Before/Afterのスクリーンショットを添付 -->

## テスト
<!-- 実施したテストについて記載 -->
- [ ] Unit Test を追加/更新
- [ ] UI Test を追加/更新
- [ ] 手動テストを実施

## チェックリスト
- [ ] コードは SwiftLint のルールに準拠している
- [ ] 適切なコメントを追加した
- [ ] ドキュメントを更新した（必要な場合）
- [ ] テストが全て通過する
- [ ] ビルドエラーがない
```

### レビューの観点

```plaintext
1. 機能要件
   - 仕様を満たしているか
   - エッジケースを考慮しているか

2. コード品質
   - 可読性は高いか
   - 適切に命名されているか
   - 重複コードはないか

3. アーキテクチャ
   - プロジェクトの構造に沿っているか
   - 責任の分離ができているか

4. パフォーマンス
   - 不要な処理はないか
   - メモリリークの可能性はないか

5. テスト
   - 十分なテストカバレッジがあるか
   - テストケースは適切か

6. セキュリティ
   - 機密情報が含まれていないか
   - 入力値の検証は適切か
```

## ドキュメント管理

### README.md

**README.md:**

```markdown
# MyAwesomeApp

## 概要
アプリケーションの簡潔な説明

## 開発環境
- Xcode 15.0+
- Swift 5.9+
- iOS 17.0+

## セットアップ

### 1. リポジトリのクローン
```bash
git clone https://github.com/company/myawesomeapp.git
cd myawesomeapp
```

### 2. 環境構築
```bash
./scripts/setup.sh
```

### 3. 環境変数の設定
`.env.example` をコピーして `.env` を作成し、必要な値を設定

### 4. Xcodeでプロジェクトを開く
```bash
open MyAwesomeApp.xcworkspace
```

## ビルド

### Debug ビルド
```bash
xcodebuild -workspace MyAwesomeApp.xcworkspace \
           -scheme "MyApp (Debug)" \
           -configuration Debug \
           build
```

### Release ビルド
```bash
bundle exec fastlane release
```

## テスト
```bash
xcodebuild test \
  -workspace MyAwesomeApp.xcworkspace \
  -scheme "MyApp (Debug)" \
  -destination 'platform=iOS Simulator,name=iPhone 15 Pro'
```

## コーディング規約
- [Swift API Design Guidelines](https://swift.org/documentation/api-design-guidelines/)
- プロジェクト固有の規約は `CONTRIBUTING.md` を参照

## ライセンス
MIT License
```

### CONTRIBUTING.md

チーム固有の開発ルールを記載します。

```markdown
# Contributing Guide

## ブランチ戦略
Git Flow を採用しています。詳細は `docs/git-workflow.md` を参照。

## コミットメッセージ
Conventional Commits に従ってください。

## Pull Request
- 小さく分割する（300行以内が望ましい）
- セルフレビューを実施する
- テストを必ず追加する

## コードレビュー
- レビューは24時間以内に対応
- 建設的なフィードバックを心がける
```

## Xcode プロジェクト設定の共有

### xcshareddata の管理

チーム全体で共有すべき設定は `xcshareddata` に配置します。

```plaintext
MyAwesomeApp.xcodeproj/
└── xcshareddata/
    └── xcschemes/
        ├── MyApp (Debug).xcscheme
        ├── MyApp (Staging).xcscheme
        └── MyApp (Release).xcscheme
```

これらのファイルは Git で管理します。

### xcuserdata の除外

個人設定は `.gitignore` で除外します。

```gitignore
*.xcuserstate
xcuserdata/
```

## まとめ

本章では、チーム開発環境の構築手法を解説しました。適切な環境統一により、以下の効果が期待されます：

- 新メンバーのオンボーディング時間の短縮
- コーディング規約の統一
- レビューの効率化
- プロジェクト全体の品質向上

次章では、CI/CD統合について詳しく解説します。
