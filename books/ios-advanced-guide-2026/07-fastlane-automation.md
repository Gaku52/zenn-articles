---
title: "Fastlaneによる自動化"
---

# Fastlaneによる自動化

Fastlaneは、iOS開発における繰り返し作業を自動化するツールです。本章では、Fastlaneを使った包括的な自動化の実装方法を解説します。

## Fastlane のセットアップ

### インストール

Bundlerを使用した管理が推奨されます。

**Gemfile:**

```ruby
source "https://rubygems.org"

gem "fastlane"
gem "cocoapods"

# プラグイン
plugins_path = File.join(File.dirname(__FILE__), 'fastlane', 'Pluginfile')
eval_gemfile(plugins_path) if File.exist?(plugins_path)
```

**インストール:**

```bash
bundle install
```

### 初期化

```bash
bundle exec fastlane init
```

対話式のセットアップで以下を選択します：

```plaintext
1. 📸 Automate screenshots
2. 👩‍✈️ Automate beta distribution to TestFlight
3. 🚀 Automate App Store distribution
4. 🛠 Manual setup
```

今回は「4. Manual setup」を選択します。

## Fastfile の構成

プロジェクト全体の自動化ロジックを Fastfile に記述します。

**fastlane/Fastfile:**

```ruby
# Fastfile

default_platform(:ios)

# 定数定義
WORKSPACE = "MyAwesomeApp.xcworkspace"
SCHEME_DEBUG = "MyApp (Debug)"
SCHEME_STAGING = "MyApp (Staging)"
SCHEME_RELEASE = "MyApp (Release)"

platform :ios do
  # MARK: - 環境設定

  before_all do
    ensure_bundle_exec
  end

  # MARK: - テスト

  desc "Run unit and UI tests"
  lane :test do
    run_tests(
      workspace: WORKSPACE,
      scheme: SCHEME_DEBUG,
      devices: ["iPhone 15 Pro", "iPhone 15"],
      code_coverage: true,
      output_directory: "./fastlane/test_output",
      output_types: "html,junit"
    )
  end

  desc "Run tests with coverage report"
  lane :test_with_coverage do
    test

    # カバレッジレポートの生成
    slather(
      workspace: WORKSPACE,
      scheme: SCHEME_DEBUG,
      proj: "MyAwesomeApp.xcodeproj",
      html: true,
      output_directory: "./fastlane/coverage"
    )
  end

  # MARK: - ビルド

  desc "Build Debug"
  lane :build_debug do
    build_app(
      workspace: WORKSPACE,
      scheme: SCHEME_DEBUG,
      configuration: "Debug",
      skip_archive: true
    )
  end

  desc "Build Staging"
  lane :build_staging do
    build_app(
      workspace: WORKSPACE,
      scheme: SCHEME_STAGING,
      configuration: "Staging",
      export_method: "ad-hoc"
    )
  end

  # MARK: - 証明書管理

  desc "Sync code signing certificates and profiles"
  lane :sync_certificates do |options|
    type = options[:type] || "development"

    match(
      type: type,
      readonly: true,
      app_identifier: ["com.company.myapp", "com.company.myapp.staging"]
    )
  end

  desc "Register new device"
  lane :register_device do |options|
    device_name = options[:name]
    device_udid = options[:udid]

    register_devices(
      devices: {
        device_name => device_udid
      }
    )

    # プロビジョニングプロファイルの更新
    match(
      type: "adhoc",
      force_for_new_devices: true
    )
  end

  # MARK: - TestFlight

  desc "Deploy to TestFlight (Staging)"
  lane :testflight_staging do
    # 証明書の同期
    sync_certificates(type: "appstore")

    # ビルド番号のインクリメント
    increment_build_number(
      xcodeproj: "MyAwesomeApp.xcodeproj"
    )

    # ビルド
    build_app(
      workspace: WORKSPACE,
      scheme: SCHEME_STAGING,
      configuration: "Staging",
      export_method: "app-store",
      export_options: {
        provisioningProfiles: {
          "com.company.myapp.staging" => "match AppStore com.company.myapp.staging"
        }
      }
    )

    # TestFlightへアップロード
    upload_to_testflight(
      skip_waiting_for_build_processing: true,
      notify_external_testers: false,
      changelog: "新機能とバグ修正",
      groups: ["Internal Testers"]
    )

    # Slackに通知
    notify_slack(
      message: "Staging版をTestFlightにアップロードしました",
      success: true
    )
  end

  desc "Deploy to TestFlight (Production)"
  lane :testflight_production do
    sync_certificates(type: "appstore")

    increment_build_number(
      xcodeproj: "MyAwesomeApp.xcodeproj"
    )

    build_app(
      workspace: WORKSPACE,
      scheme: SCHEME_RELEASE,
      configuration: "Release",
      export_method: "app-store"
    )

    upload_to_testflight(
      skip_waiting_for_build_processing: true,
      notify_external_testers: true,
      changelog: change_log,
      groups: ["External Testers"]
    )

    notify_slack(
      message: "Production版をTestFlightにアップロードしました",
      success: true
    )
  end

  # MARK: - App Store

  desc "Submit to App Store"
  lane :app_store do
    # Git の状態確認
    ensure_git_status_clean
    ensure_git_branch(branch: "main")

    # 証明書の同期
    sync_certificates(type: "appstore")

    # バージョン番号の確認
    version = get_version_number(xcodeproj: "MyAwesomeApp.xcodeproj")

    # ビルド
    build_app(
      workspace: WORKSPACE,
      scheme: SCHEME_RELEASE,
      configuration: "Release",
      export_method: "app-store"
    )

    # App Storeへアップロード
    upload_to_app_store(
      submit_for_review: false,
      automatic_release: false,
      skip_metadata: false,
      skip_screenshots: false,
      precheck_include_in_app_purchases: false
    )

    # Gitタグの作成
    add_git_tag(
      tag: "v#{version}",
      message: "Release version #{version}"
    )

    push_git_tags

    # Slackに通知
    notify_slack(
      message: "App Store v#{version} の申請が完了しました",
      success: true
    )
  end

  # MARK: - スクリーンショット

  desc "Generate screenshots"
  lane :screenshots do
    capture_screenshots(
      workspace: WORKSPACE,
      scheme: "MyAppUITests",
      devices: [
        "iPhone 15 Pro Max",
        "iPhone 15 Pro",
        "iPhone 15",
        "iPhone SE (3rd generation)",
        "iPad Pro (12.9-inch) (6th generation)"
      ],
      languages: ["ja-JP", "en-US"],
      output_directory: "./fastlane/screenshots",
      clear_previous_screenshots: true
    )

    frame_screenshots(
      white: true,
      path: "./fastlane/screenshots"
    )
  end

  # MARK: - バージョン管理

  desc "Bump version (major, minor, patch)"
  lane :bump_version do |options|
    type = options[:type] || "patch"

    increment_version_number(
      bump_type: type,
      xcodeproj: "MyAwesomeApp.xcodeproj"
    )

    version = get_version_number(xcodeproj: "MyAwesomeApp.xcodeproj")

    UI.success("Version bumped to #{version}")
  end

  # MARK: - コード品質

  desc "Run SwiftLint"
  lane :lint do
    swiftlint(
      mode: :lint,
      strict: true,
      reporter: "html",
      output_file: "./fastlane/swiftlint-report.html"
    )
  end

  desc "Run SwiftFormat"
  lane :format do
    sh("swiftformat ..")
  end

  # MARK: - ヘルパーメソッド

  private_lane :notify_slack do |options|
    next unless is_ci

    slack(
      message: options[:message],
      success: options[:success],
      slack_url: ENV["SLACK_WEBHOOK_URL"],
      payload: {
        "Version" => get_version_number(xcodeproj: "MyAwesomeApp.xcodeproj"),
        "Build" => get_build_number(xcodeproj: "MyAwesomeApp.xcodeproj"),
        "Git Branch" => git_branch
      }
    )
  end

  private_lane :change_log do
    changelog_from_git_commits(
      commits_count: 10,
      pretty: "- %s",
      merge_commit_filtering: "exclude_merges"
    )
  end

  # MARK: - エラーハンドリング

  error do |lane, exception|
    slack(
      message: "エラーが発生しました: #{exception.message}",
      success: false,
      slack_url: ENV["SLACK_WEBHOOK_URL"]
    ) if is_ci
  end
end
```

## Appfile の設定

**fastlane/Appfile:**

```ruby
app_identifier("com.company.myapp")
apple_id("developer@company.com")
itc_team_id("123456789")
team_id("ABCDEFGHIJ")

# Staging用の設定
for_platform :ios do
  for_lane :testflight_staging do
    app_identifier("com.company.myapp.staging")
  end
end
```

## Matchfile の設定

証明書とプロビジョニングプロファイルの管理を自動化します。

**fastlane/Matchfile:**

```ruby
git_url("https://github.com/company/certificates")

storage_mode("git")

type("development")

app_identifier(["com.company.myapp", "com.company.myapp.staging"])

username("developer@company.com")
team_id("ABCDEFGHIJ")

# 暗号化のためのパスフレーズ
# 実際の値は環境変数で管理
# export MATCH_PASSWORD=your_passphrase
```

### Matchの初期化

```bash
# 開発用証明書の作成
bundle exec fastlane match development

# App Store用証明書の作成
bundle exec fastlane match appstore

# Ad Hoc用証明書の作成
bundle exec fastlane match adhoc
```

## プラグインの活用

Fastlaneプラグインで機能を拡張します。

### プラグインのインストール

```bash
bundle exec fastlane add_plugin slack
bundle exec fastlane add_plugin versioning
bundle exec fastlane add_plugin badge
```

**fastlane/Pluginfile:**

```ruby
# Autogenerated by fastlane

gem 'fastlane-plugin-slack'
gem 'fastlane-plugin-versioning'
gem 'fastlane-plugin-badge'
```

### バッジの追加

開発ビルドにバッジを追加します。

```ruby
desc "Add badge to app icon"
lane :add_badge do
  add_badge(
    shield: "Version-1.0.0-blue",
    shield_gravity: "South",
    shield_no_resize: true
  )
end
```

## 環境変数の管理

機密情報は環境変数で管理します。

**.env.default:**

```plaintext
# App Information
APP_IDENTIFIER=com.company.myapp
APPLE_ID=developer@company.com

# Team Information
TEAM_ID=ABCDEFGHIJ
ITC_TEAM_ID=123456789

# Slack
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

**.env.secret（Gitignore対象）:**

```plaintext
# Fastlane
FASTLANE_USER=developer@company.com
FASTLANE_PASSWORD=your_password
FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD=your_app_specific_password

# Match
MATCH_PASSWORD=your_match_passphrase

# Slack
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/ACTUAL/WEBHOOK
```

### 環境変数の読み込み

Fastfileで自動的に読み込まれます：

```ruby
before_all do
  # .envファイルから環境変数を読み込み
  Dotenv.load('.env.secret')
end
```

## カスタムアクション

繰り返し使用するロジックをカスタムアクションとして定義します。

**fastlane/actions/notify_completion.rb:**

```ruby
module Fastlane
  module Actions
    class NotifyCompletionAction < Action
      def self.run(params)
        message = params[:message]
        success = params[:success]

        # 複数の通知先に送信
        slack(
          message: message,
          success: success
        )

        # 他の通知サービスへも送信可能
        # notify_teams(message: message)
        # notify_discord(message: message)
      end

      def self.available_options
        [
          FastlaneCore::ConfigItem.new(
            key: :message,
            description: "Notification message",
            type: String
          ),
          FastlaneCore::ConfigItem.new(
            key: :success,
            description: "Success status",
            type: Boolean,
            default_value: true
          )
        ]
      end

      def self.is_supported?(platform)
        platform == :ios
      end
    end
  end
end
```

使用方法：

```ruby
notify_completion(
  message: "ビルドが完了しました",
  success: true
)
```

## よく使うコマンド

日常的に使用するFastlaneコマンドです。

### テスト実行

```bash
bundle exec fastlane test
```

### TestFlightへのデプロイ

```bash
# Staging
bundle exec fastlane testflight_staging

# Production
bundle exec fastlane testflight_production
```

### App Store申請

```bash
bundle exec fastlane app_store
```

### バージョン更新

```bash
# パッチバージョン（1.0.0 -> 1.0.1）
bundle exec fastlane bump_version type:patch

# マイナーバージョン（1.0.0 -> 1.1.0）
bundle exec fastlane bump_version type:minor

# メジャーバージョン（1.0.0 -> 2.0.0）
bundle exec fastlane bump_version type:major
```

### デバイス登録

```bash
bundle exec fastlane register_device name:"John's iPhone" udid:"00000000-0000000000000000"
```

## トラブルシューティング

### よくある問題と解決策

#### 1. 証明書が見つからない

```bash
# Matchのキャッシュをクリア
bundle exec fastlane match nuke development
bundle exec fastlane match development
```

#### 2. ビルドが失敗する

```bash
# Derived Dataをクリア
rm -rf ~/Library/Developer/Xcode/DerivedData

# 再度ビルド
bundle exec fastlane build_debug
```

#### 3. TestFlightアップロードがタイムアウト

```ruby
# Fastfile で timeout を延長
upload_to_testflight(
  skip_waiting_for_build_processing: true,
  timeout: 3600  # 1時間
)
```

## まとめ

本章では、Fastlaneによる包括的な自動化の実装方法を解説しました。Fastlane導入により、以下の効果が期待されます：

- リリースプロセスの標準化
- 手作業によるミスの削減
- デプロイ時間の短縮
- チーム全体の生産性向上

次章では、プロジェクトメンテナンスについて解説します。
