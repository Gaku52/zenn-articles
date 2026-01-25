---
title: "第18章 GitHub Actionsワークフロー"
---

# 第18章 GitHub Actionsワークフロー

本章では、Danger.jsとReviewDogを統合した完全なGitHub Actionsワークフローの構築方法について詳しく解説します。コードレビューの自動化を実現する包括的なCI/CDパイプラインを作成します。

## GitHub Actionsワークフローの全体像

コードレビュー自動化のための完全なワークフローには、以下の要素が含まれます。

### ワークフローの構成要素

- 静的解析（Linter）の自動実行
- テストの自動実行とカバレッジ測定
- Danger.jsによるPRメタデータチェック
- ReviewDogによるLint結果の自動コメント
- ビルドの検証
- セキュリティスキャン

## 基本的なワークフロー構成

まずは基本的なワークフロー構造を理解しましょう。

### ディレクトリ構成

```:
.github/
  workflows/
    code-review.yml        # メインのレビューワークフロー
    danger.yml             # Danger.js専用
    test.yml              # テスト実行
    lint.yml              # Lint実行
    build.yml             # ビルド検証
```

### トリガーの設定

```yaml
# .github/workflows/code-review.yml
name: Code Review Automation

on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches:
      - main
      - develop

  # 手動実行も可能に
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: write
  checks: write
```

## 完全なコードレビューワークフロー

Danger.js、ReviewDog、テスト、ビルドを統合した完全なワークフロー例です。

```yaml
# .github/workflows/code-review.yml
name: Code Review Automation

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write
  checks: write
  statuses: write

jobs:
  # ===== Danger.js によるPRチェック =====
  danger:
    name: Danger PR Check
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run Danger
        run: npm run danger:pr
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # ===== ESLint + ReviewDog =====
  eslint:
    name: ESLint Review
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
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

  # ===== TypeScript型チェック =====
  typecheck:
    name: TypeScript Type Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run TypeScript compiler
        run: npx tsc --noEmit

  # ===== テスト実行 =====
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests with coverage
        run: npm run test:coverage

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/coverage-final.json
          fail_ci_if_error: false

  # ===== ビルド検証 =====
  build:
    name: Build Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build project
        run: npm run build

      - name: Check bundle size
        run: |
          npm run build:analyze
          # バンドルサイズが大きすぎる場合は警告
          if [ -f dist/stats.json ]; then
            SIZE=$(jq '.assets[0].size' dist/stats.json)
            if [ $SIZE -gt 5000000 ]; then
              echo "::warning::Bundle size is larger than 5MB"
            fi
          fi

  # ===== セキュリティチェック =====
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run npm audit
        run: npm audit --audit-level=moderate
        continue-on-error: true

      - name: Run Snyk security scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
```

## マトリックスビルドの活用

複数のバージョンやプラットフォームでテストを実行する方法です。

```yaml
# .github/workflows/matrix-test.yml
name: Matrix Tests

on:
  pull_request:

jobs:
  test:
    name: Test on Node ${{ matrix.node }} and ${{ matrix.os }}
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node: ['18', '20', '21']
        exclude:
          # Windows + Node 18 の組み合わせを除外
          - os: windows-latest
            node: '18'
      fail-fast: false

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results-${{ matrix.os }}-${{ matrix.node }}
          path: test-results/
```

## キャッシュ戦略

CI実行時間を短縮するための効果的なキャッシュ戦略です。

### 依存関係のキャッシュ

```yaml
# Node.js の依存関係キャッシュ
- name: Setup Node.js with cache
  uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'

# より詳細なキャッシュ制御
- name: Cache node modules
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

### ビルド成果物のキャッシュ

```yaml
# ビルド成果物のキャッシュ
- name: Cache build output
  uses: actions/cache@v4
  with:
    path: |
      .next/cache
      dist/
      build/
    key: ${{ runner.os }}-build-${{ github.sha }}
    restore-keys: |
      ${{ runner.os }}-build-
```

### テストキャッシュ

```yaml
# Jestのキャッシュ
- name: Cache Jest
  uses: actions/cache@v4
  with:
    path: .jest-cache
    key: ${{ runner.os }}-jest-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-jest-
```

## 条件付き実行

特定の条件下でのみジョブを実行する設定です。

```yaml
# .github/workflows/conditional.yml
name: Conditional Workflows

on:
  pull_request:
    paths:
      - 'src/**'
      - 'package.json'

jobs:
  # ドキュメント変更時のみ実行
  docs:
    name: Documentation Check
    if: contains(github.event.pull_request.labels.*.name, 'documentation')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check documentation
        run: npm run docs:check

  # 特定ファイル変更時のみ実行
  api-test:
    name: API Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Check if API files changed
        id: api-files
        run: |
          if git diff --name-only HEAD^ HEAD | grep -q '^src/api/'; then
            echo "changed=true" >> $GITHUB_OUTPUT
          else
            echo "changed=false" >> $GITHUB_OUTPUT
          fi

      - name: Run API tests
        if: steps.api-files.outputs.changed == 'true'
        run: npm run test:api

  # PRサイズに応じた実行
  large-pr-check:
    name: Large PR Additional Checks
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Check PR size
        id: pr-size
        run: |
          ADDITIONS=${{ github.event.pull_request.additions }}
          DELETIONS=${{ github.event.pull_request.deletions }}
          TOTAL=$((ADDITIONS + DELETIONS))

          if [ $TOTAL -gt 500 ]; then
            echo "is_large=true" >> $GITHUB_OUTPUT
          else
            echo "is_large=false" >> $GITHUB_OUTPUT
          fi

      - name: Run comprehensive checks for large PR
        if: steps.pr-size.outputs.is_large == 'true'
        run: |
          npm run lint:strict
          npm run test:integration
          npm run security:scan
```

## 並列実行と依存関係

ジョブの並列実行と依存関係の管理です。

```yaml
# .github/workflows/parallel-jobs.yml
name: Parallel Jobs with Dependencies

on:
  pull_request:

jobs:
  # 準備ジョブ（最初に実行）
  setup:
    name: Setup
    runs-on: ubuntu-latest
    outputs:
      cache-key: ${{ steps.cache-key.outputs.key }}
    steps:
      - uses: actions/checkout@v4

      - name: Generate cache key
        id: cache-key
        run: echo "key=${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}" >> $GITHUB_OUTPUT

  # 並列実行されるジョブ群
  lint:
    name: Lint
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint

  test-unit:
    name: Unit Tests
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:unit

  test-integration:
    name: Integration Tests
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:integration

  # すべてのテストが成功した後に実行
  build:
    name: Build
    needs: [lint, test-unit, test-integration]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build

  # 最終確認（すべて成功後に実行）
  all-checks-passed:
    name: All Checks Passed
    needs: [lint, test-unit, test-integration, build]
    runs-on: ubuntu-latest
    steps:
      - name: Success message
        run: echo "All checks passed successfully!"
```

## エラーハンドリングと通知

エラー発生時の適切な処理と通知の設定です。

```yaml
# .github/workflows/error-handling.yml
name: Error Handling

on:
  pull_request:

jobs:
  test-with-retry:
    name: Tests with Retry
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      # リトライ機能付きテスト実行
      - name: Run tests
        uses: nick-fields/retry@v3
        with:
          timeout_minutes: 10
          max_attempts: 3
          command: npm test

      # エラー時にアーティファクトをアップロード
      - name: Upload error logs
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: error-logs
          path: |
            npm-debug.log
            test-results/
            screenshots/

      # エラー時にSlack通知
      - name: Notify on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Test failed for PR #${{ github.event.pull_request.number }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "Test failed for PR <${{ github.event.pull_request.html_url }}|#${{ github.event.pull_request.number }}>"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## ワークフローチェックリスト

効果的なGitHub Actionsワークフローを構築するための確認事項です。

### 基本設定

- [ ] 適切なトリガーが設定されているか
- [ ] 必要な権限が付与されているか
- [ ] ブランチ保護ルールと連携しているか
- [ ] タイムアウト設定が適切か

### パフォーマンス

- [ ] キャッシュが効果的に使用されているか
- [ ] 並列実行が適切に設定されているか
- [ ] 不要なステップが削除されているか
- [ ] ジョブの依存関係が最適化されているか

### セキュリティ

- [ ] シークレット情報が適切に管理されているか
- [ ] 最小権限の原則が守られているか
- [ ] サードパーティアクションのバージョンが固定されているか
- [ ] セキュリティスキャンが実行されているか

### 保守性

- [ ] ワークフローが適切に分割されているか
- [ ] コメントで説明が記載されているか
- [ ] 再利用可能なワークフローが活用されているか
- [ ] エラーハンドリングが実装されているか

## パフォーマンス最適化

CI実行時間を短縮するためのテクニックです。

### 早期失敗戦略

```yaml
jobs:
  quick-checks:
    name: Quick Checks
    runs-on: ubuntu-latest
    steps:
      # 最も速いチェックから実行
      - uses: actions/checkout@v4

      - name: Check PR title
        run: |
          if [[ ! "${{ github.event.pull_request.title }}" =~ ^(feat|fix|docs|style|refactor|test|chore): ]]; then
            echo "::error::PR title must follow Conventional Commits format"
            exit 1
          fi

      - name: Check file count
        run: |
          COUNT=$(git diff --name-only ${{ github.event.pull_request.base.sha }} ${{ github.event.pull_request.head.sha }} | wc -l)
          if [ $COUNT -gt 100 ]; then
            echo "::warning::This PR modifies $COUNT files"
          fi

  # 上記のチェックが成功した場合のみ実行
  comprehensive-checks:
    needs: quick-checks
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # ... より時間のかかるチェック
```

## まとめ

本章ではGitHub Actionsワークフローの完全な構築方法について解説しました。

主なポイント:
- Danger.jsとReviewDogを統合した包括的なワークフロー
- マトリックスビルドで複数環境をテスト
- キャッシュ戦略でCI実行時間を短縮
- 条件付き実行で効率化
- エラーハンドリングと通知で品質を担保

次章では、大規模PRのケーススタディについて学びます。

## 参考文献

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Actions - Workflow syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [GitHub Actions - Caching dependencies](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows)
- [GitHub Actions - Using matrices](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs)
- [Codecov Documentation](https://docs.codecov.com/docs)
