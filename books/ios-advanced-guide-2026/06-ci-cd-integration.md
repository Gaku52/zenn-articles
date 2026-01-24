---
title: "CI/CD統合"
---

# CI/CD統合

継続的インテグレーション（CI）と継続的デリバリー（CD）の導入により、開発プロセスの自動化と品質向上が期待されます。本章では、GitHub Actionsを使ったCI/CDパイプラインの構築方法を解説します。

## CI/CDの概要

### 継続的インテグレーション（CI）

コードの変更を頻繁にメインブランチに統合し、自動テストを実行することで、問題の早期発見が可能になります。

**CIで実行すること:**
- コードのビルド
- Unit Testの実行
- SwiftLintによる静的解析
- コードカバレッジの測定

### 継続的デリバリー（CD）

ビルドとテストが成功したコードを、自動的にTestFlightやApp Storeにデプロイします。

**CDで実行すること:**
- アーカイブの作成
- 署名とプロビジョニング
- TestFlightへのアップロード
- App Store申請（承認後）

## GitHub Actionsの基本

GitHub Actionsは、GitHubリポジトリに統合されたCI/CDプラットフォームです。

### ワークフローファイルの配置

```plaintext
.github/
└── workflows/
    ├── ci.yml
    ├── release.yml
    └── testflight.yml
```

## CI ワークフローの実装

プルリクエスト時に自動テストを実行するワークフローです。

**.github/workflows/ci.yml:**

```yaml
name: CI

on:
  pull_request:
    branches:
      - main
      - develop
  push:
    branches:
      - main
      - develop

env:
  DEVELOPER_DIR: /Applications/Xcode_15.0.app/Contents/Developer

jobs:
  test:
    name: Build and Test
    runs-on: macos-14

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Xcode
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: '15.0'

      - name: Cache SPM
        uses: actions/cache@v3
        with:
          path: .build
          key: ${{ runner.os }}-spm-${{ hashFiles('**/Package.resolved') }}
          restore-keys: |
            ${{ runner.os }}-spm-

      - name: Cache DerivedData
        uses: actions/cache@v3
        with:
          path: ~/Library/Developer/Xcode/DerivedData
          key: ${{ runner.os }}-derived-data-${{ hashFiles('**/*.swift') }}
          restore-keys: |
            ${{ runner.os }}-derived-data-

      - name: Install Dependencies
        run: |
          brew install swiftlint

      - name: SwiftLint
        run: swiftlint --strict

      - name: Build
        run: |
          xcodebuild build \
            -workspace MyAwesomeApp.xcworkspace \
            -scheme "MyApp (Debug)" \
            -destination 'platform=iOS Simulator,name=iPhone 15 Pro' \
            | xcbeautify

      - name: Run Tests
        run: |
          xcodebuild test \
            -workspace MyAwesomeApp.xcworkspace \
            -scheme "MyApp (Debug)" \
            -destination 'platform=iOS Simulator,name=iPhone 15 Pro' \
            -enableCodeCoverage YES \
            -resultBundlePath TestResults \
            | xcbeautify

      - name: Upload Test Results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: test-results
          path: TestResults.xcresult

      - name: Code Coverage
        run: |
          xcrun xccov view --report --json TestResults.xcresult > coverage.json

      - name: Upload Coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.json
          fail_ci_if_error: true
```

## TestFlight デプロイワークフロー

Staging環境用のTestFlightデプロイを自動化します。

**.github/workflows/testflight.yml:**

```yaml
name: Deploy to TestFlight

on:
  push:
    branches:
      - release/*

env:
  DEVELOPER_DIR: /Applications/Xcode_15.0.app/Contents/Developer

jobs:
  deploy:
    name: Build and Deploy to TestFlight
    runs-on: macos-14

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true

      - name: Setup Xcode
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: '15.0'

      - name: Install Dependencies
        run: bundle install

      - name: Import Certificates
        env:
          CERTIFICATE_BASE64: ${{ secrets.BUILD_CERTIFICATE_BASE64 }}
          P12_PASSWORD: ${{ secrets.P12_PASSWORD }}
          KEYCHAIN_PASSWORD: ${{ secrets.KEYCHAIN_PASSWORD }}
        run: |
          # Keychainの作成
          security create-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
          security default-keychain -s build.keychain
          security unlock-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
          security set-keychain-settings -t 3600 -u build.keychain

          # 証明書のインポート
          echo -n "$CERTIFICATE_BASE64" | base64 --decode -o certificate.p12
          security import certificate.p12 \
            -k build.keychain \
            -P "$P12_PASSWORD" \
            -T /usr/bin/codesign \
            -T /usr/bin/security
          security set-key-partition-list -S apple-tool:,apple: -s -k "$KEYCHAIN_PASSWORD" build.keychain

      - name: Download Provisioning Profiles
        env:
          PROVISIONING_PROFILE_BASE64: ${{ secrets.PROVISIONING_PROFILE_BASE64 }}
        run: |
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          echo -n "$PROVISIONING_PROFILE_BASE64" | base64 --decode -o ~/Library/MobileDevice/Provisioning\ Profiles/profile.mobileprovision

      - name: Increment Build Number
        run: |
          BUILD_NUMBER=${{ github.run_number }}
          agvtool new-version -all $BUILD_NUMBER

      - name: Build and Upload to TestFlight
        env:
          FASTLANE_USER: ${{ secrets.FASTLANE_USER }}
          FASTLANE_PASSWORD: ${{ secrets.FASTLANE_PASSWORD }}
          FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD: ${{ secrets.FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD }}
        run: |
          bundle exec fastlane testflight

      - name: Clean up
        if: always()
        run: |
          security delete-keychain build.keychain
          rm -f ~/Library/MobileDevice/Provisioning\ Profiles/profile.mobileprovision
          rm -f certificate.p12
```

## Secretsの設定

GitHub Secretsで機密情報を管理します。

### 必要なSecrets

```plaintext
Settings > Secrets and variables > Actions

Secrets:
- BUILD_CERTIFICATE_BASE64: ビルド証明書（Base64エンコード）
- P12_PASSWORD: P12ファイルのパスワード
- KEYCHAIN_PASSWORD: Keychainのパスワード
- PROVISIONING_PROFILE_BASE64: プロビジョニングプロファイル（Base64エンコード）
- FASTLANE_USER: Apple IDのメールアドレス
- FASTLANE_PASSWORD: Apple IDのパスワード
- FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD: App用パスワード
```

### 証明書のBase64エンコード

```bash
# 証明書をBase64にエンコード
base64 -i Certificates.p12 | pbcopy

# プロビジョニングプロファイルをBase64にエンコード
base64 -i Profile.mobileprovision | pbcopy
```

## Fastlane との統合

Fastlaneでビルドとデプロイのロジックを管理します。

**fastlane/Fastfile:**

```ruby
# Fastfile

default_platform(:ios)

platform :ios do
  desc "Run tests"
  lane :test do
    run_tests(
      workspace: "MyAwesomeApp.xcworkspace",
      scheme: "MyApp (Debug)",
      devices: ["iPhone 15 Pro"],
      code_coverage: true
    )
  end

  desc "Build for TestFlight"
  lane :testflight do
    # 証明書とプロビジョニングプロファイルの同期
    sync_code_signing(
      type: "appstore",
      readonly: true
    )

    # ビルド番号のインクリメント
    increment_build_number(
      xcodeproj: "MyAwesomeApp.xcodeproj"
    )

    # ビルド
    build_app(
      workspace: "MyAwesomeApp.xcworkspace",
      scheme: "MyApp (Staging)",
      configuration: "Staging",
      export_method: "app-store",
      export_options: {
        provisioningProfiles: {
          "com.company.myapp.staging" => "MyApp Staging"
        }
      }
    )

    # TestFlightにアップロード
    upload_to_testflight(
      skip_waiting_for_build_processing: true,
      notify_external_testers: false
    )

    # Slackに通知（オプション）
    slack(
      message: "TestFlightへのアップロードが完了しました",
      success: true
    )
  end

  desc "Deploy to App Store"
  lane :release do
    # 証明書とプロビジョニングプロファイルの同期
    sync_code_signing(
      type: "appstore",
      readonly: true
    )

    # バージョン番号の確認
    ensure_git_status_clean

    # ビルド
    build_app(
      workspace: "MyAwesomeApp.xcworkspace",
      scheme: "MyApp (Release)",
      configuration: "Release",
      export_method: "app-store"
    )

    # App Storeに申請
    upload_to_app_store(
      submit_for_review: false,
      automatic_release: false,
      skip_metadata: false,
      skip_screenshots: false
    )

    # Gitタグの作成
    add_git_tag(
      tag: "v#{get_version_number}",
      message: "Release v#{get_version_number}"
    )

    push_git_tags

    # Slackに通知
    slack(
      message: "App Store申請が完了しました",
      success: true
    )
  end
end
```

## ビルド番号の自動管理

ビルド番号を自動でインクリメントします。

### agvtool の使用

```bash
# 現在のビルド番号を表示
agvtool what-version

# ビルド番号を設定
agvtool new-version -all 42

# バージョン番号を表示
agvtool what-marketing-version
```

### GitHub Actionsでの自動インクリメント

```yaml
- name: Increment Build Number
  run: |
    BUILD_NUMBER=${{ github.run_number }}
    agvtool new-version -all $BUILD_NUMBER
```

## コードカバレッジの可視化

テストカバレッジを計測し、可視化します。

### Codecovの統合

**.codecov.yml:**

```yaml
coverage:
  status:
    project:
      default:
        target: 80%
        threshold: 1%
    patch:
      default:
        target: 80%

comment:
  layout: "reach, diff, flags, files"
  behavior: default

ignore:
  - "**/*Tests.swift"
  - "**/Generated/**"
  - "**/*.generated.swift"
```

## Pull Request チェック

PRごとに実行するチェック項目を定義します。

**.github/workflows/pr-checks.yml:**

```yaml
name: PR Checks

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  lint:
    name: SwiftLint
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      - name: SwiftLint
        run: |
          brew install swiftlint
          swiftlint lint --strict --reporter github-actions-logging

  format:
    name: SwiftFormat
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      - name: SwiftFormat
        run: |
          brew install swiftformat
          swiftformat . --lint

  size:
    name: Check PR Size
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check PR Size
        run: |
          LINES=$(git diff --stat origin/${{ github.base_ref }}...HEAD | tail -1 | awk '{print $4}')
          if [ "$LINES" -gt 500 ]; then
            echo "::error::PR is too large ($LINES lines changed). Please split it into smaller PRs."
            exit 1
          fi
```

## 通知の設定

ビルド結果をSlackに通知します。

**Fastlane Slackプラグイン:**

```ruby
# Gemfile
gem 'fastlane'
gem 'fastlane-plugin-slack'
```

**Slackへの通知:**

```ruby
# Fastfile
slack(
  message: "ビルドが成功しました",
  channel: "#ios-releases",
  success: true,
  payload: {
    "Version" => get_version_number,
    "Build" => get_build_number
  }
)
```

## まとめ

本章では、GitHub ActionsとFastlaneを使ったCI/CDパイプラインの構築方法を解説しました。CI/CD導入により、以下の効果が期待されます：

- コードの品質向上
- リリースプロセスの自動化
- 問題の早期発見
- デプロイの効率化

次章では、Fastlaneによる自動化についてさらに詳しく解説します。
