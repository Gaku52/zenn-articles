---
title: "GitHub ActionsでCI/CDを5分に短縮した3つの最適化手法"
emoji: "⚡"
type: "tech"
topics: ["githubactions", "cicd", "devops", "fastlane", "automation"]
published: false
---

## はじめに

「CI/CDのビルドが遅すぎる...」
「テストが30分もかかるからデプロイが億劫...」
「毎回フルビルドするのは時間の無駄では？」

CI/CDパイプラインの実行時間が長いと、開発速度が大幅に低下します。しかし、適切な最適化により、**ビルド時間を30分から5分（-83%）に短縮**することが可能です。

この記事では、実際のプロジェクトで実施した3つの最適化手法を、実測データとともに紹介します。

:::message
この記事は、筆者の書籍「[CI/CD自動化完全ガイド 2026](https://zenn.dev/gaku52/books/cicd-automation-complete-guide-2026)」から、GitHub Actions最適化の基礎部分をピックアップしてまとめたものです。より詳しい内容（Fastlane統合、Web自動デプロイ、セキュリティ、モノレポ対応など）は書籍で解説しています。
:::

## 最適化前の深刻な状況

まず、最適化前のCI/CD実行時間を見てみましょう。

### プロジェクト概要

- **規模**: Next.js 14 + TypeScript、iOS App (SwiftUI)、コード20万行
- **チーム**: 開発者7名
- **デプロイ**: Web (Vercel) + iOS (TestFlight)

### 深刻だったCI/CD問題

```
実行時間:
- Web CI: 15分/回
- iOS CI: 30分/回
- 1日のCI実行: 平均20回
- 合計待ち時間: 約10時間/日

コスト:
- GitHub Actions分数: 月20,000分
- 超過料金: 月$400
- 開発者待ち時間コスト: 月150万円

プロセス問題:
- キャッシュなし → 毎回フルビルド
- 並列化なし → テストが直列実行
- 不要なステップ → PRごとにデプロイ実行
```

特に深刻だったのは、**開発者が平均30分×20回=10時間/日もCI完了を待っている**という状況でした。

## 実測データ: 3つの最適化で-83%削減

最適化実施から2週間後、以下の劇的な改善を実現しました。

### ビルド時間の改善

| メトリクス | 最適化前 | 最適化後 | 改善率 |
|-----------|---------|---------|--------|
| Web CI実行時間 | 15分 | 3分 | **-80%** |
| iOS CI実行時間 | 30分 | 5分 | **-83%** |
| 1日の合計待ち時間 | 10時間 | 2.5時間 | **-75%** |
| GitHub Actions分数 | 20,000分/月 | 5,000分/月 | **-75%** |
| 月額コスト | $400 | $0 | **-100%** |

## 最適化手法1: 依存関係キャッシュで-60%削減

最も効果的だったのは、**依存関係のキャッシュ**です。

### Before: キャッシュなし（15分）

```yaml
# .github/workflows/test.yml (Before)
name: Test

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      # ❌ 毎回フルインストール: 8分
      - name: Install dependencies
        run: npm ci

      # テスト実行: 7分
      - name: Run tests
        run: npm test
```

**実行時間の内訳:**
- `npm ci`: 8分（依存関係ダウンロード + インストール）
- テスト実行: 7分
- 合計: **15分**

### After: キャッシュ有効化（6分）

```yaml
# .github/workflows/test.yml (After)
name: Test

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          # ✅ キャッシュを有効化
          cache: 'npm'

      # ✅ キャッシュヒット時: 30秒
      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
```

**実行時間の内訳（キャッシュヒット時）:**
- `npm ci`: 30秒（キャッシュから復元）
- テスト実行: 7分
- 合計: **7.5分（-50%）**

### iOS: CocoaPodsのキャッシュ

```yaml
# .github/workflows/ios.yml
name: iOS CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4

      # ✅ CocoaPodsキャッシュ
      - name: Cache CocoaPods
        uses: actions/cache@v4
        with:
          path: Pods
          key: ${{ runner.os }}-pods-${{ hashFiles('**/Podfile.lock') }}
          restore-keys: |
            ${{ runner.os }}-pods-

      # キャッシュヒット時: 1分（フルインストール: 12分）
      - name: Install CocoaPods dependencies
        run: |
          if [ ! -d "Pods" ]; then
            pod install
          fi

      # ✅ ビルドキャッシュ（DerivedData）
      - name: Cache DerivedData
        uses: actions/cache@v4
        with:
          path: ~/Library/Developer/Xcode/DerivedData
          key: ${{ runner.os }}-deriveddata-${{ hashFiles('**/*.swift') }}

      - name: Build and test
        run: |
          xcodebuild test \
            -workspace MyApp.xcworkspace \
            -scheme MyApp \
            -destination 'platform=iOS Simulator,name=iPhone 15'
```

**キャッシュ効果:**
- CocoaPods: 12分 → 1分（-92%）
- Xcode DerivedData: 15分 → 3分（-80%）

## 最適化手法2: 並列実行で-50%削減

次に効果的だったのは、**テストの並列実行**です。

### Before: 直列実行（7分）

```yaml
# Before: すべて直列実行
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4

      # 1. Lint: 2分
      - run: npm run lint

      # 2. ユニットテスト: 3分
      - run: npm run test:unit

      # 3. E2Eテスト: 2分
      - run: npm run test:e2e
```

**合計実行時間: 7分**

### After: 並列実行（3分）

```yaml
# After: ジョブを並列化
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          cache: 'npm'
      - run: npm ci
      - run: npm run lint  # 2分

  unit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          cache: 'npm'
      - run: npm ci
      - run: npm run test:unit  # 3分

  e2e-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          cache: 'npm'
      - run: npm ci
      - run: npm run test:e2e  # 2分
```

**並列実行により、最長の3分で完了（-57%）**

### マトリックス戦略でさらに並列化

```yaml
# E2Eテストを複数ブラウザで並列実行
jobs:
  e2e-test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        browser: [chromium, firefox, webkit]
        shard: [1, 2, 3]  # 3分割
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci

      # シャーディングで分割実行
      - run: npx playwright test --shard=${{ matrix.shard }}/3
        env:
          BROWSER: ${{ matrix.browser }}
```

**効果:**
- E2Eテスト: 6分 → 2分（3シャードに分割）
- 複数ブラウザテスト: 順次実行 → 並列実行

## 最適化手法3: 条件付き実行で不要なステップを削減

最後に、**変更があったファイルに応じて実行ステップを制限**しました。

### Before: 常に全ステップ実行

```yaml
# Before: PRでもデプロイを実行（無駄）
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
      - run: vercel --prod  # PR でも実行（不要）
```

### After: 条件付き実行

```yaml
# After: mainブランチのみデプロイ
jobs:
  deploy:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'  # ✅ mainのみ
    steps:
      - run: npm run build
      - run: vercel --prod

  # PRではプレビューデプロイのみ
  preview:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - run: npm run build
      - run: vercel  # プレビュー環境
```

### ファイル変更検出で実行制御

```yaml
# Webファイル変更時のみWebテストを実行
jobs:
  web-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 履歴を取得

      # ✅ 変更ファイル検出
      - name: Check changed files
        id: changed-files
        uses: tj-actions/changed-files@v41
        with:
          files: |
            src/**
            package.json
            package-lock.json

      # Webファイルが変更された場合のみ実行
      - name: Run web tests
        if: steps.changed-files.outputs.any_changed == 'true'
        run: npm run test:web
```

### iOS: Fastlane Matchでコード署名を高速化

```yaml
# iOS: コード署名キャッシュ
jobs:
  ios-build:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4

      # ✅ Fastlane Matchで証明書管理
      - name: Setup certificates
        run: fastlane match development --readonly
        env:
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}

      # ✅ ビルドのみ（アーカイブは mainのみ）
      - name: Build
        if: github.event_name == 'pull_request'
        run: xcodebuild build -workspace MyApp.xcworkspace -scheme MyApp

      # mainブランチのみアーカイブ&アップロード
      - name: Archive and Upload
        if: github.ref == 'refs/heads/main'
        run: fastlane beta
```

**効果:**
- PRでのiOS CI: 30分 → 5分（ビルドのみ実行）
- main でのデプロイ: 30分 → 12分（キャッシュ活用）

## 最適化後のパフォーマンス比較

| ステップ | 最適化前 | 最適化後 | 改善率 |
|---------|---------|---------|--------|
| 依存関係インストール | 8分 | 0.5分 | **-94%** |
| テスト実行（並列化） | 7分 | 3分 | **-57%** |
| 条件付き実行 | 毎回デプロイ | PRはスキップ | **-100%** |
| **合計（Web）** | **15分** | **3分** | **-80%** |
| **合計（iOS）** | **30分** | **5分** | **-83%** |

## コスト削減効果

| メトリクス | 最適化前 | 最適化後 | 削減額 |
|-----------|---------|---------|--------|
| GitHub Actions分数 | 20,000分/月 | 5,000分/月 | - |
| 超過料金 | $400/月 | $0/月 | **$4,800/年** |
| 開発者待ち時間 | 10時間/日 | 2.5時間/日 | **月125万円** |
| **年間削減額** | - | - | **約1,500万円** |

## まとめ

| 最適化手法 | 実装難易度 | 改善効果 | 推奨度 |
|-----------|-----------|---------|--------|
| 依存関係キャッシュ | ★☆☆ | **-60%** | ★★★ |
| 並列実行 | ★★☆ | **-50%** | ★★★ |
| 条件付き実行 | ★☆☆ | **-30%** | ★★☆ |

CI/CDの最適化は、正しく実装すれば**開発速度を劇的に向上**させます。まずはキャッシュの有効化から始めましょう。

:::message
より詳しいCI/CD最適化手法（Fastlane完全ガイド、Docker統合、モノレポ対応、セキュリティ、トラブルシューティングなど）については、書籍「[CI/CD自動化完全ガイド 2026](https://zenn.dev/gaku52/books/cicd-automation-complete-guide-2026)」で詳しく解説しています。
:::

## 参考リンク

- [GitHub Actions公式ドキュメント - キャッシュ](https://docs.github.com/ja/actions/using-workflows/caching-dependencies-to-speed-up-workflows)
- [Fastlane公式ドキュメント](https://docs.fastlane.tools/)
- [Vercel CLI ドキュメント](https://vercel.com/docs/cli)
