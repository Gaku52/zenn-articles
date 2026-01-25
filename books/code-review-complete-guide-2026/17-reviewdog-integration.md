---
title: "第17章 ReviewDog統合"
---

# 第17章 ReviewDog統合

本章ではReviewDogを使った静的解析ツールの統合とGitHub Actionsでの自動化について、具体的なセットアップ方法と実践的な設定を詳しく解説します。ReviewDogは、各種Lintツールの結果をPull Requestのコメントとして自動投稿する強力なツールです。

## ReviewDogとは

ReviewDogは、静的解析ツール（Linter）の実行結果をPull Requestに自動的にコメントするツールです。ESLint、Pylint、SwiftLintなど、様々なLintツールと統合できます。

### ReviewDogの利点

- Lintエラーを該当行に直接コメント
- 複数のLintツールを一元管理
- GitHubのSuggestion機能で修正案を提示
- レビューの自動化と効率化
- CI/CDパイプラインとの統合が容易

## ReviewDogのセットアップ

ReviewDogをプロジェクトに導入する手順を解説します。

### インストール

```bash
# macOS (Homebrew)
brew install reviewdog

# Linux
curl -sfL https://raw.githubusercontent.com/reviewdog/reviewdog/master/install.sh | sh -s

# Docker
docker pull reviewdog/reviewdog:latest
```

### GitHub Actions での使用

最も一般的な使用方法は、GitHub Actionsでの自動実行です。

```yaml
# .github/workflows/reviewdog.yml
name: ReviewDog

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  reviewdog:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup ReviewDog
        uses: reviewdog/action-setup@v1
        with:
          reviewdog_version: latest
```

## ESLint との統合

JavaScriptやTypeScriptプロジェクトでの ESLint 統合例です。

### 設定ファイル

```yaml
# .github/workflows/eslint.yml
name: ESLint ReviewDog

on:
  pull_request:

jobs:
  eslint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint with ReviewDog
        uses: reviewdog/action-eslint@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          reporter: github-pr-review
          eslint_flags: 'src/**/*.{ts,tsx,js,jsx}'
          fail_on_error: true
          filter_mode: added
```

### カスタム ESLint 設定

より詳細な制御が必要な場合は、reviewdog コマンドを直接使用します。

```yaml
# .github/workflows/eslint-custom.yml
name: ESLint Custom ReviewDog

on:
  pull_request:

jobs:
  eslint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Setup ReviewDog
        uses: reviewdog/action-setup@v1

      - name: Run ESLint
        env:
          REVIEWDOG_GITHUB_API_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          npm run lint -- --format=json | \
          reviewdog -f=eslint \
            -name="ESLint" \
            -reporter=github-pr-review \
            -filter-mode=added \
            -fail-on-error
```

### .eslintrc.js 設定例

```javascript
// .eslintrc.js
module.exports = {
  env: {
    browser: true,
    es2021: true,
    node: true,
  },
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
    'plugin:react/recommended',
    'plugin:react-hooks/recommended',
    'prettier',
  ],
  parser: '@typescript-eslint/parser',
  parserOptions: {
    ecmaFeatures: {
      jsx: true,
    },
    ecmaVersion: 'latest',
    sourceType: 'module',
  },
  plugins: ['@typescript-eslint', 'react', 'react-hooks'],
  rules: {
    // エラーレベル
    'no-console': 'error',
    'no-debugger': 'error',
    '@typescript-eslint/no-unused-vars': 'error',
    '@typescript-eslint/explicit-function-return-type': 'warn',

    // React specific
    'react/react-in-jsx-scope': 'off',
    'react/prop-types': 'off',
    'react-hooks/rules-of-hooks': 'error',
    'react-hooks/exhaustive-deps': 'warn',
  },
  settings: {
    react: {
      version: 'detect',
    },
  },
};
```

## Pylint との統合

Pythonプロジェクトでの Pylint 統合例です。

```yaml
# .github/workflows/pylint.yml
name: Pylint ReviewDog

on:
  pull_request:

jobs:
  pylint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pylint

      - name: Setup ReviewDog
        uses: reviewdog/action-setup@v1

      - name: Run Pylint with ReviewDog
        env:
          REVIEWDOG_GITHUB_API_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          pylint --output-format=json src/ | \
          reviewdog -f=pylint \
            -name="Pylint" \
            -reporter=github-pr-review \
            -filter-mode=added \
            -fail-on-error
```

### .pylintrc 設定例

```ini
# .pylintrc
[MASTER]
jobs=4
suggestion-mode=yes
unsafe-load-any-extension=no

[MESSAGES CONTROL]
disable=
    missing-docstring,
    too-few-public-methods,
    too-many-arguments,
    too-many-locals,
    too-many-branches,
    too-many-statements

[REPORTS]
output-format=json
reports=no
score=yes

[BASIC]
good-names=i,j,k,x,y,z,ex,_,id,pk
bad-names=foo,bar,baz,toto,tutu,tata

[FORMAT]
max-line-length=100
indent-string='    '

[DESIGN]
max-args=7
max-locals=15
max-returns=6
max-branches=12
max-statements=50
```

## SwiftLint との統合

iOSプロジェクトでの SwiftLint 統合例です。

```yaml
# .github/workflows/swiftlint.yml
name: SwiftLint ReviewDog

on:
  pull_request:

jobs:
  swiftlint:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install SwiftLint
        run: brew install swiftlint

      - name: Setup ReviewDog
        uses: reviewdog/action-setup@v1

      - name: Run SwiftLint with ReviewDog
        env:
          REVIEWDOG_GITHUB_API_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          swiftlint lint --reporter json | \
          reviewdog -f=swiftlint \
            -name="SwiftLint" \
            -reporter=github-pr-review \
            -filter-mode=added \
            -fail-on-error
```

### .swiftlint.yml 設定例

```yaml
# .swiftlint.yml
included:
  - Sources
  - Tests

excluded:
  - Pods
  - .build
  - Generated

analyzer_rules:
  - unused_declaration
  - unused_import

opt_in_rules:
  - array_init
  - closure_end_indentation
  - closure_spacing
  - empty_count
  - explicit_init
  - fatal_error_message
  - first_where
  - force_unwrapping
  - implicitly_unwrapped_optional
  - sorted_first_last

disabled_rules:
  - trailing_whitespace

line_length:
  warning: 120
  error: 200
  ignores_comments: true

type_body_length:
  warning: 300
  error: 500

function_body_length:
  warning: 40
  error: 100

file_length:
  warning: 500
  error: 1000

cyclomatic_complexity:
  warning: 10
  error: 20

identifier_name:
  min_length:
    warning: 2
  max_length:
    warning: 40
    error: 50
```

## カスタムルール作成

ReviewDogで独自のカスタムルールを作成する方法です。

### カスタムLinterスクリプト

```bash
#!/bin/bash
# scripts/custom-linter.sh

# TODO コメントのチェック
grep -n "TODO" src/**/*.ts | while read -r line; do
  file=$(echo "$line" | cut -d: -f1)
  lineno=$(echo "$line" | cut -d: -f2)
  message=$(echo "$line" | cut -d: -f3-)

  echo "$file:$lineno:warning: TODO コメントが残っています: $message"
done

# FIXME コメントのチェック
grep -n "FIXME" src/**/*.ts | while read -r line; do
  file=$(echo "$line" | cut -d: -f1)
  lineno=$(echo "$line" | cut -d: -f2)
  message=$(echo "$line" | cut -d: -f3-)

  echo "$file:$lineno:error: FIXME コメントが残っています: $message"
done

# console.log のチェック
grep -n "console\.log" src/**/*.ts | while read -r line; do
  file=$(echo "$line" | cut -d: -f1)
  lineno=$(echo "$line" | cut -d: -f2)

  echo "$file:$lineno:warning: console.log が残っています"
done
```

### カスタムルールの実行

```yaml
# .github/workflows/custom-lint.yml
name: Custom Lint ReviewDog

on:
  pull_request:

jobs:
  custom-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup ReviewDog
        uses: reviewdog/action-setup@v1

      - name: Run Custom Linter
        env:
          REVIEWDOG_GITHUB_API_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          bash scripts/custom-linter.sh | \
          reviewdog -efm="%f:%l:%t%*[^:]: %m" \
            -name="CustomLint" \
            -reporter=github-pr-review \
            -filter-mode=added
```

## 複数Linterの統合

複数のLintツールを一つのワークフローで実行する例です。

```yaml
# .github/workflows/lint-all.yml
name: All Linters

on:
  pull_request:

jobs:
  lint:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        linter:
          - name: ESLint
            command: npm run lint -- --format=json
            format: eslint
          - name: Prettier
            command: npm run format:check
            format: diff
          - name: TypeScript
            command: npx tsc --noEmit
            format: tsc
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Setup ReviewDog
        uses: reviewdog/action-setup@v1

      - name: Run ${{ matrix.linter.name }}
        env:
          REVIEWDOG_GITHUB_API_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          ${{ matrix.linter.command }} | \
          reviewdog -f=${{ matrix.linter.format }} \
            -name="${{ matrix.linter.name }}" \
            -reporter=github-pr-review \
            -filter-mode=added \
            -fail-on-error=false
```

## ReviewDog設定のベストプラクティス

効果的なReviewDog運用のためのベストプラクティスです。

### reporter オプションの使い分け

```yaml
# github-pr-review: PR全体にレビューコメント（推奨）
reporter: github-pr-review

# github-pr-check: GitHub Checks API を使用
reporter: github-pr-check

# github-check: Deprecated、github-pr-check を使用
reporter: github-check
```

### filter-mode の設定

```yaml
# added: 追加・変更された行のみ（推奨）
filter-mode: added

# diff_context: 変更された行とその周辺
filter-mode: diff_context

# file: ファイル全体
filter-mode: file

# nofilter: フィルタリングなし
filter-mode: nofilter
```

### fail-on-error の使い方

```yaml
# エラー時にCIを失敗させる（本番向け）
fail-on-error: true

# エラーがあってもCIを継続（開発中）
fail-on-error: false
```

## ReviewDogチェックリスト

ReviewDogを導入・運用する際の確認事項です。

### セットアップ

- [ ] ReviewDogがインストールされているか
- [ ] GitHub Tokenが適切に設定されているか
- [ ] 各Lintツールが正しく設定されているか
- [ ] ワークフローファイルが作成されているか

### Linter設定

- [ ] ESLint、Pylint、SwiftLintなどが適切に設定されているか
- [ ] ルールが厳しすぎず、緩すぎないか
- [ ] チーム全体でルールが合意されているか
- [ ] カスタムルールが必要な場合は実装されているか

### 運用

- [ ] false positive（誤検知）が少ないか
- [ ] コメントが該当行に正しく投稿されているか
- [ ] filter-mode が適切に設定されているか
- [ ] CI実行時間が許容範囲内か

### パフォーマンス

- [ ] Linter実行時間が長すぎないか
- [ ] キャッシュが有効活用されているか
- [ ] 並列実行が適切に設定されているか

## トラブルシューティング

よくある問題と解決方法です。

### コメントが投稿されない

```yaml
# GITHUB_TOKEN の権限を確認
env:
  REVIEWDOG_GITHUB_API_TOKEN: ${{ secrets.GITHUB_TOKEN }}

# または Personal Access Token を使用
env:
  REVIEWDOG_GITHUB_API_TOKEN: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
```

### Linterの出力形式が合わない

```bash
# カスタムフォーマットを定義
reviewdog -efm="%f:%l:%c: %m" \
  -name="CustomLinter" \
  -reporter=github-pr-review
```

### 実行時間が長い

```yaml
# キャッシュを活用
- uses: actions/setup-node@v4
  with:
    cache: 'npm'

# 変更ファイルのみをチェック
filter-mode: added
```

## まとめ

本章ではReviewDogによる静的解析ツールの統合について解説しました。

主なポイント:
- ReviewDogで各種Lintツールを統合できる
- ESLint、Pylint、SwiftLintなどと連携可能
- GitHub Actionsで自動化が容易
- カスタムルールも作成可能
- filter-modeやreporterの適切な設定が重要

次章では、GitHub Actionsワークフローの完全な構築方法について学びます。

## 参考文献

- [ReviewDog Official Documentation](https://github.com/reviewdog/reviewdog)
- [reviewdog/action-eslint](https://github.com/reviewdog/action-eslint)
- [reviewdog/action-setup](https://github.com/reviewdog/action-setup)
- [GitHub Actions - Understanding GitHub Actions](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions)
- [ESLint Documentation](https://eslint.org/docs/latest/)
