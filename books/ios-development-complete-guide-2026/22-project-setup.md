---
title: "Chapter 15: プロジェクトセットアップとベストプラクティス"
---

# Chapter 15: プロジェクトセットアップとベストプラクティス

## 15.1 プロジェクト初期化の概要

### 15.1.1 適切な初期設定の重要性

iOSプロジェクトの初期設定は、長期的な開発効率と保守性に大きな影響を与えます。

**初期設定のメリット:**

```swift
/*
適切な初期設定がもたらす効果:

1. 開発効率の向上
   - 統一された構造により、コードの配置場所が明確
   - ボイラープレートコードの再利用
   - セットアップ時間の短縮

2. 保守性の向上
   - 拡張しやすい構造で機能追加がスムーズ
   - 一貫性のある設計で理解しやすい
   - リファクタリングコストの削減

3. チーム協業の円滑化
   - 統一されたルールでレビューやマージが容易
   - オンボーディングの簡素化
   - コミュニケーションコストの削減

4. CI/CDの導入
   - 自動化しやすい構造
   - デプロイの安全性向上
   - 品質保証の自動化

5. 技術的負債の削減
   - 初期から適切な設計
   - スケーラブルなアーキテクチャ
   - 将来の変更に柔軟に対応
*/
```

### 15.1.2 プロジェクト作成前のチェックリスト

```bash
# プロジェクト作成前の確認事項

## ビジネス要件
- [ ] アプリ名の決定
- [ ] ターゲットユーザーの明確化
- [ ] 主要機能のリストアップ
- [ ] リリース時期の目標設定
- [ ] 予算とリソースの確認
- [ ] 競合分析

## 技術要件
- [ ] 対応iOSバージョンの決定（推奨: iOS 15.0+）
- [ ] UI Framework選定（SwiftUI / UIKit / 混合）
- [ ] アーキテクチャの選定（MVVM / Clean Architecture）
- [ ] 外部サービスの選定（Firebase / AWS）
- [ ] 必須ライブラリのリストアップ
- [ ] データベース選定（Core Data / Realm）

## チーム体制
- [ ] 開発メンバーの確認
- [ ] ロールと責任の明確化
- [ ] コミュニケーションツールの選定
- [ ] コードレビュープロセスの決定
- [ ] ブランチ戦略の決定（Git Flow / GitHub Flow）

## 環境準備
- [ ] Xcode最新版のインストール
- [ ] Apple Developer Programへの加入
- [ ] Gitリポジトリの作成
- [ ] プロジェクト管理ツールのセットアップ
- [ ] デザインツールへのアクセス（Figma）
```

### 15.1.3 プロジェクト命名規則

```swift
// プロジェクト命名のベストプラクティス

/*
プロジェクト名:
✅ PascalCase を使用: MyAwesomeApp
❌ スペース不可: My Awesome App
❌ 特殊文字不可: My-App, My_App
✅ 簡潔で覚えやすい名前
✅ 検索可能な名前（一般的すぎない）

Bundle Identifier:
✅ リバースドメイン形式: com.company.appname
✅ 小文字推奨: com.company.myawesomeapp
❌ ハイフン不可: com.my-company.app
✅ アンダースコア可: com.company.my_app
✅ サブドメイン活用: com.company.ios.appname

環境別 Bundle Identifier:
- Production:  com.company.myapp
- Staging:     com.company.myapp.staging
- Development: com.company.myapp.dev
- Beta:        com.company.myapp.beta

Target名:
- Main Target:     MyApp
- Unit Tests:      MyAppTests
- UI Tests:        MyAppUITests
- Shared Framework: MyAppKit
- Extensions:      MyAppWidgetExtension
*/
```

## 15.2 Xcodeプロジェクト作成

### 15.2.1 ステップバイステップガイド

**Step 1: Xcodeプロジェクトの新規作成**

```bash
# Xcode起動
1. Xcode.app を起動
2. "Create a new Xcode project" を選択
3. テンプレート選択画面へ

# テンプレート選択
Platform: iOS
Template: App

# 推奨テンプレート:
- シンプルなアプリ: App
- ゲーム: Game
- ARアプリ: Augmented Reality App
```

**Step 2: プロジェクト設定**

```swift
// Project Options の設定

/*
Product Name: MyApp
- アプリのプロジェクト名
- PascalCase推奨
- スペースなし

Team: Your Apple Developer Team
- Apple Developerアカウント
- 個人開発者 or 組織

Organization Identifier: com.company
- リバースドメイン形式
- Bundle IDのプレフィックスになる

Bundle Identifier: com.company.MyApp
- 自動生成される
- 世界で一意である必要がある

Language: Swift
- Swift推奨（Objective-Cとの混在も可能）

User Interface: SwiftUI
- SwiftUI: モダンな宣言的UI
- Storyboard: 従来型のUI（UIKit）

Include Tests: ✅
- Unit Test Targetを自動生成
- 必須: チェック推奨

Include UI Tests: ✅
- UI Test Targetを自動生成
- 推奨: 初期からテスト基盤を用意

Storage: None / Core Data / SwiftData
- None: 手動でデータベース選択
- Core Data: Apple純正ORM
- SwiftData: iOS 17+ の新しいデータ管理
*/
```

**Step 3: 保存場所の選択**

```bash
# プロジェクト保存場所の決定

推奨ディレクトリ構造:
~/Development/
├── Personal/
│   └── MyApp/
├── Company/
│   ├── ProductA/
│   └── ProductB/
└── OpenSource/
    └── MyLibrary/

# Git リポジトリの初期化
☑️ Create Git repository on my Mac
- チェック推奨: 自動的にGit初期化

# 保存
1. 適切なディレクトリを選択
2. "Create" をクリック
3. プロジェクトが開く
```

### 15.2.2 プロジェクト作成直後の設定

**General タブの設定:**

```swift
// TARGETS > MyApp > General

/*
Identity:
- Display Name: MyApp
  → ホーム画面に表示される名前
  → 環境ごとに変更可能: "MyApp Dev", "MyApp Staging"

- Bundle Identifier: com.company.myapp
  → App Storeで一意
  → 変更不可（一度公開したら）

- Version: 1.0.0
  → セマンティックバージョニング
  → マーケティングバージョン（ユーザー向け）

- Build: 1
  → ビルド番号（内部管理用）
  → リリースごとにインクリメント
  → TestFlightでは必須

Deployment Info:
- iOS Deployment Target: 15.0
  → 対応する最小iOSバージョン
  → 推奨: iOS 15.0+（2024年時点）
  → トレードオフ: 新機能 vs ユーザーカバレッジ

- iPhone, iPad
  → 対応デバイス
  → iPhoneのみ / iPadのみ / Universal

- Device Orientation
  → Portrait: 縦向き（推奨: ON）
  → Landscape Left/Right: 横向き
  → Upside Down: 上下逆（iPhone: OFF推奨）

App Icons and Launch Screen:
- App Icon Source: AppIcon
  → Assets.xcassets内のアイコンセット
  → 必須サイズ: 1024x1024 (@1x)

- Launch Screen: LaunchScreen
  → 起動画面
  → SwiftUI or Storyboard
*/
```

**Signing & Capabilities タブの設定:**

```swift
// TARGETS > MyApp > Signing & Capabilities

/*
Signing:

Automatically manage signing: ✅ (推奨: 開発初期)
- Xcodeが自動的に証明書とプロビジョニングプロファイルを管理
- メリット: 設定が簡単、エラーが少ない
- デメリット: CI/CDでの制御が難しい

Team: Your Team Name
- Apple Developerアカウント
- Personal Team（無料アカウント）も選択可能
  → ただし実機テストは制限あり

Bundle Identifier: com.company.myapp
- Generalタブと同期

Provisioning Profile: Xcode Managed Profile
- 自動生成される
- 開発用 / AdHoc / App Store用が自動選択

Capabilities: （必要に応じて追加）

+ Capability から追加:
  - Push Notifications: プッシュ通知
  - Background Modes: バックグラウンド実行
  - Sign in with Apple: Apple IDログイン
  - App Groups: アプリ間データ共有
  - Associated Domains: Universal Links
  - iCloud: クラウドストレージ
*/
```

### 15.2.3 Build Settings の重要設定

```swift
// TARGETS > MyApp > Build Settings

/*
Swift Compiler - Language:
- Swift Language Version: Swift 5
  → 最新のSwiftバージョンを使用

Swift Compiler - Code Generation:
- Optimization Level:
  → Debug: No Optimization [-Onone]
  → Release: Optimize for Speed [-O]

- Compilation Mode:
  → Debug: Incremental（差分ビルド）
  → Release: Whole Module（最適化優先）

Deployment:
- iOS Deployment Target: 15.0
  → サポート最小バージョン

- Strip Debug Symbols During Copy:
  → Debug: No
  → Release: Yes（アプリサイズ削減）

Architectures:
- Build Active Architecture Only:
  → Debug: Yes（ビルド時間短縮）
  → Release: No（全アーキテクチャ対応）

Build Options:
- Enable Bitcode: No
  → iOS 15+では非推奨

- Debug Information Format:
  → Debug: DWARF
  → Release: DWARF with dSYM File
  → クラッシュレポート解析に必須
*/
```

## 15.3 プロジェクト構造設計

### 15.3.1 MVVM アーキテクチャによるフォルダ構成

```
MyApp/
├── App/
│   ├── MyApp.swift                    # @main App Entry Point (SwiftUI)
│   ├── AppDelegate.swift              # UIKit Lifecycle (Optional)
│   └── SceneDelegate.swift            # Scene Management (Optional)
│
├── Features/                          # Feature-Based Organization
│   ├── Authentication/
│   │   ├── Views/
│   │   │   ├── LoginView.swift
│   │   │   ├── SignUpView.swift
│   │   │   └── ForgotPasswordView.swift
│   │   ├── ViewModels/
│   │   │   ├── LoginViewModel.swift
│   │   │   └── SignUpViewModel.swift
│   │   ├── Models/
│   │   │   ├── User.swift
│   │   │   └── AuthCredentials.swift
│   │   └── Services/
│   │       └── AuthenticationService.swift
│   │
│   ├── Home/
│   │   ├── Views/
│   │   │   ├── HomeView.swift
│   │   │   ├── FeedView.swift
│   │   │   └── Components/
│   │   │       ├── FeedItemView.swift
│   │   │       └── EmptyStateView.swift
│   │   ├── ViewModels/
│   │   │   └── HomeViewModel.swift
│   │   └── Models/
│   │       └── FeedItem.swift
│   │
│   └── Profile/
│       ├── Views/
│       │   ├── ProfileView.swift
│       │   └── EditProfileView.swift
│       ├── ViewModels/
│       │   └── ProfileViewModel.swift
│       └── Models/
│           └── UserProfile.swift
│
├── Core/                              # Shared Core Components
│   ├── Networking/
│   │   ├── HTTPClient.swift
│   │   ├── APIClient.swift
│   │   ├── Endpoint.swift
│   │   └── NetworkError.swift
│   │
│   ├── Database/
│   │   ├── CoreDataStack.swift
│   │   ├── DatabaseManager.swift
│   │   └── Entities/
│   │       └── MyAppModel.xcdatamodeld
│   │
│   ├── Services/
│   │   ├── LocationService.swift
│   │   ├── NotificationService.swift
│   │   ├── AnalyticsService.swift
│   │   └── CrashReportingService.swift
│   │
│   └── Storage/
│       ├── UserDefaults+Extension.swift
│       ├── KeychainManager.swift
│       └── FileManager+Extension.swift
│
├── Common/                            # Common/Shared Components
│   ├── Extensions/
│   │   ├── Foundation/
│   │   │   ├── String+Extensions.swift
│   │   │   ├── Date+Extensions.swift
│   │   │   └── URL+Extensions.swift
│   │   ├── UIKit/
│   │   │   ├── UIView+Extensions.swift
│   │   │   └── UIColor+Extensions.swift
│   │   └── SwiftUI/
│   │       ├── View+Extensions.swift
│   │       └── Color+Extensions.swift
│   │
│   ├── Utilities/
│   │   ├── Logger.swift
│   │   ├── Validator.swift
│   │   ├── DateFormatter.swift
│   │   └── ImageLoader.swift
│   │
│   ├── Constants/
│   │   ├── AppConstants.swift
│   │   ├── APIConstants.swift
│   │   └── ColorPalette.swift
│   │
│   └── Helpers/
│       ├── KeyboardHelper.swift
│       └── HapticHelper.swift
│
├── UI/                                # UI Components
│   ├── Components/
│   │   ├── Buttons/
│   │   │   ├── PrimaryButton.swift
│   │   │   └── SecondaryButton.swift
│   │   ├── TextFields/
│   │   │   ├── CustomTextField.swift
│   │   │   └── SearchTextField.swift
│   │   ├── Cards/
│   │   │   └── ContentCard.swift
│   │   └── LoadingViews/
│   │       └── LoadingSpinner.swift
│   │
│   ├── Modifiers/
│   │   ├── CardModifier.swift
│   │   └── ShimmerModifier.swift
│   │
│   └── Styles/
│       ├── ButtonStyles.swift
│       └── TextFieldStyles.swift
│
├── Resources/                         # Resource Files
│   ├── Assets.xcassets/
│   │   ├── AppIcon.appiconset
│   │   ├── Colors/
│   │   ├── Images/
│   │   └── Icons/
│   │
│   ├── Fonts/
│   │   ├── CustomFont-Regular.ttf
│   │   └── CustomFont-Bold.ttf
│   │
│   └── Localizable/
│       ├── en.lproj/
│       │   └── Localizable.strings
│       └── ja.lproj/
│           └── Localizable.strings
│
├── Config/                            # Build Configurations
│   ├── Base.xcconfig
│   ├── Debug.xcconfig
│   ├── Staging.xcconfig
│   └── Release.xcconfig
│
└── Supporting Files/
    ├── Info.plist
    └── MyApp.entitlements
```

### 15.3.2 ファイル配置の実装例

```swift
// Features/Authentication/Views/LoginView.swift

import SwiftUI

struct LoginView: View {
    @StateObject private var viewModel: LoginViewModel

    init(viewModel: LoginViewModel = LoginViewModel()) {
        _viewModel = StateObject(wrappedValue: viewModel)
    }

    var body: some View {
        VStack(spacing: 20) {
            Text("Login")
                .font(.largeTitle)
                .bold()

            CustomTextField(
                placeholder: "Email",
                text: $viewModel.email
            )

            SecureField("Password", text: $viewModel.password)
                .textFieldStyle(RoundedBorderTextFieldStyle())

            if let errorMessage = viewModel.errorMessage {
                Text(errorMessage)
                    .foregroundColor(.red)
                    .font(.caption)
            }

            PrimaryButton(title: "Login") {
                Task {
                    await viewModel.login()
                }
            }
            .disabled(viewModel.isLoading)
        }
        .padding()
    }
}

// Features/Authentication/ViewModels/LoginViewModel.swift

import Foundation
import Combine

@MainActor
final class LoginViewModel: ObservableObject {
    @Published var email: String = ""
    @Published var password: String = ""
    @Published var isLoading: Bool = false
    @Published var errorMessage: String?

    private let authService: AuthenticationService
    private var cancellables = Set<AnyCancellable>()

    init(authService: AuthenticationService = AuthenticationService()) {
        self.authService = authService
    }

    func login() async {
        isLoading = true
        errorMessage = nil
        defer { isLoading = false }

        do {
            try await authService.login(email: email, password: password)
            // Navigate to home
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}
```

## 15.4 Swift Package Manager (SPM)

### 15.4.1 SPMによるパッケージ追加

**Xcode GUIからの追加:**

```swift
/*
1. File > Add Packages...
2. 検索バーにURLを入力:
   https://github.com/Alamofire/Alamofire.git
3. Dependency Ruleを選択:
   - Up to Next Major Version: 5.0.0 < 6.0.0
   - Up to Next Minor Version: 5.8.0 < 5.9.0
   - Exact Version: 5.8.0
   - Branch: main
   - Commit: abc123def
4. Add Packageをクリック
5. Targetに追加するライブラリを選択
*/
```

**Package.swiftによる管理（推奨）:**

```swift
// Package.swift

import PackageDescription

let package = Package(
    name: "MyApp",
    platforms: [
        .iOS(.v15)
    ],
    products: [
        .library(
            name: "MyAppKit",
            targets: ["MyAppKit"]
        )
    ],
    dependencies: [
        // MARK: - Networking
        .package(
            url: "https://github.com/Alamofire/Alamofire.git",
            from: "5.8.0"
        ),
        .package(
            url: "https://github.com/Moya/Moya.git",
            from: "15.0.0"
        ),

        // MARK: - UI
        .package(
            url: "https://github.com/onevcat/Kingfisher.git",
            from: "7.10.0"
        ),
        .package(
            url: "https://github.com/kean/Nuke.git",
            from: "12.0.0"
        ),

        // MARK: - Reactive Programming
        .package(
            url: "https://github.com/CombineCommunity/CombineExt.git",
            from: "1.8.0"
        ),

        // MARK: - Database
        .package(
            url: "https://github.com/realm/realm-swift.git",
            from: "10.45.0"
        ),

        // MARK: - Firebase
        .package(
            url: "https://github.com/firebase/firebase-ios-sdk.git",
            from: "10.20.0"
        ),

        // MARK: - Utility
        .package(
            url: "https://github.com/SwiftyJSON/SwiftyJSON.git",
            from: "5.0.1"
        ),
        .package(
            url: "https://github.com/kishikawakatsumi/KeychainAccess.git",
            from: "4.2.2"
        ),

        // MARK: - Development Tools
        .package(
            url: "https://github.com/realm/SwiftLint.git",
            from: "0.54.0"
        )
    ],
    targets: [
        .target(
            name: "MyAppKit",
            dependencies: [
                "Alamofire",
                .product(name: "Moya", package: "Moya"),
                "Kingfisher",
                .product(name: "Nuke", package: "Nuke"),
                "CombineExt",
                .product(name: "RealmSwift", package: "realm-swift"),
                .product(name: "FirebaseAnalytics", package: "firebase-ios-sdk"),
                .product(name: "FirebaseCrashlytics", package: "firebase-ios-sdk"),
                "SwiftyJSON",
                "KeychainAccess"
            ]
        ),
        .testTarget(
            name: "MyAppKitTests",
            dependencies: ["MyAppKit"]
        )
    ]
)
```

### 15.4.2 SPMのワークフロー

```bash
# Packageの追加・更新

# 1. Packageを追加
# File > Add Packages... (GUI)
# またはPackage.swiftを編集

# 2. Packageを更新
# File > Packages > Update to Latest Package Versions
# または
xcodebuild -resolvePackageDependencies

# 3. Packageを削除
# Project Navigator > Package Dependencies > 右クリック > Remove

# 4. Package.resolvedの管理
# Package.resolved: 現在インストールされているバージョンを記録
# Gitで管理すべき: チーム全体で同じバージョンを使用

git add Package.resolved
git commit -m "chore: update package dependencies"

# 5. キャッシュのクリア
# File > Packages > Reset Package Caches

# コマンドライン:
rm -rf ~/Library/Caches/org.swift.swiftpm
rm -rf ~/Library/Developer/Xcode/DerivedData
```

## 15.5 xcconfig活用

### 15.5.1 xcconfigファイルの作成

**Development.xcconfig:**

```bash
// Config/Development.xcconfig

// App Configuration
APP_NAME = MyApp Dev
BUNDLE_ID_SUFFIX = .dev
APP_VERSION = 1.0.0
BUILD_NUMBER = 1

// Server Configuration
API_BASE_URL = https:/$()/dev.api.example.com
API_KEY = dev_api_key_here
WEB_SOCKET_URL = wss:/$()/dev.ws.example.com

// Feature Flags
ENABLE_ANALYTICS = NO
ENABLE_CRASH_REPORTING = NO
ENABLE_DEBUG_MENU = YES

// Build Settings
SWIFT_ACTIVE_COMPILATION_CONDITIONS = DEBUG DEV
GCC_PREPROCESSOR_DEFINITIONS = DEBUG=1 DEV=1

// Code Signing
CODE_SIGN_STYLE = Automatic
DEVELOPMENT_TEAM = ABCD123456
CODE_SIGN_IDENTITY = iPhone Developer

// Optimization
SWIFT_OPTIMIZATION_LEVEL = -Onone
GCC_OPTIMIZATION_LEVEL = 0
SWIFT_COMPILATION_MODE = singlefile
```

**Production.xcconfig:**

```bash
// Config/Production.xcconfig

// App Configuration
APP_NAME = MyApp
BUNDLE_ID_SUFFIX =
APP_VERSION = 1.0.0
BUILD_NUMBER = 1

// Server Configuration
API_BASE_URL = https:/$()/api.example.com
API_KEY = prod_api_key_here
WEB_SOCKET_URL = wss:/$()/ws.example.com

// Feature Flags
ENABLE_ANALYTICS = YES
ENABLE_CRASH_REPORTING = YES
ENABLE_DEBUG_MENU = NO

// Build Settings
SWIFT_ACTIVE_COMPILATION_CONDITIONS = RELEASE
GCC_PREPROCESSOR_DEFINITIONS =

// Code Signing
CODE_SIGN_STYLE = Manual
DEVELOPMENT_TEAM = ABCD123456
CODE_SIGN_IDENTITY = iPhone Distribution
PROVISIONING_PROFILE_SPECIFIER = MyApp Production

// Optimization
SWIFT_OPTIMIZATION_LEVEL = -O
GCC_OPTIMIZATION_LEVEL = s
SWIFT_COMPILATION_MODE = wholemodule
DEAD_CODE_STRIPPING = YES
STRIP_INSTALLED_PRODUCT = YES
```

### 15.5.2 xcconfigの継承

```bash
# Config/Base.xcconfig
// 共通設定

IPHONEOS_DEPLOYMENT_TARGET = 15.0
TARGETED_DEVICE_FAMILY = 1,2
SWIFT_VERSION = 5.9

ENABLE_BITCODE = NO
ALWAYS_EMBED_SWIFT_STANDARD_LIBRARIES = YES

CLANG_WARN_QUOTED_INCLUDE_IN_FRAMEWORK_HEADER = YES
CLANG_WARN_DOCUMENTATION_COMMENTS = YES
```

```bash
# Config/Debug.xcconfig
#include "Base.xcconfig"

APP_NAME = MyApp Dev
SWIFT_OPTIMIZATION_LEVEL = -Onone
ENABLE_TESTABILITY = YES
```

```bash
# Config/Release.xcconfig
#include "Base.xcconfig"

APP_NAME = MyApp
SWIFT_OPTIMIZATION_LEVEL = -O
ENABLE_TESTABILITY = NO
```

### 15.5.3 Swiftでの環境変数アクセス

```swift
// Environment.swift

import Foundation

enum Environment {
    // MARK: - API Configuration

    static var apiBaseURL: String {
        bundleValue(for: "API_BASE_URL") ?? "https://api.example.com"
    }

    static var apiKey: String {
        bundleValue(for: "API_KEY") ?? ""
    }

    // MARK: - Feature Flags

    static var isAnalyticsEnabled: Bool {
        bundleValue(for: "ENABLE_ANALYTICS") == "YES"
    }

    static var isCrashReportingEnabled: Bool {
        bundleValue(for: "ENABLE_CRASH_REPORTING") == "YES"
    }

    static var isDebugMenuEnabled: Bool {
        #if DEBUG
        return true
        #else
        return bundleValue(for: "ENABLE_DEBUG_MENU") == "YES"
        #endif
    }

    // MARK: - Helpers

    private static func bundleValue(for key: String) -> String? {
        Bundle.main.infoDictionary?[key] as? String
    }
}
```

## 15.6 Fastlane セットアップ

### 15.6.1 Fastlaneのインストール

```bash
# Homebrew でインストール（推奨）
brew install fastlane

# Bundler経由（推奨: バージョン固定）
# Gemfile
source "https://rubygems.org"

gem "fastlane", "~> 2.219"

# インストール
bundle install

# 以降はbundle execを使用
bundle exec fastlane
```

### 15.6.2 Fastfileの基本構成

```ruby
# fastlane/Fastfile

default_platform(:ios)

platform :ios do
  # 変数定義
  before_all do
    setup_ci if ENV['CI']
    ensure_git_status_clean unless ENV['CI']
  end

  # MARK: - Development

  desc "Development ビルド"
  lane :dev do
    build_app(
      scheme: "MyApp (Development)",
      configuration: "Debug",
      export_method: "development",
      output_directory: "./build",
      output_name: "MyApp-Dev.ipa",
      clean: true
    )
  end

  # MARK: - Testing

  desc "Unit Tests 実行"
  lane :test do
    scan(
      scheme: "MyApp",
      device: "iPhone 15 Pro",
      code_coverage: true,
      output_directory: "./test_output",
      clean: true
    )
  end

  # MARK: - Staging

  desc "Staging ビルドとTestFlight アップロード"
  lane :staging do
    ensure_git_branch(branch: "develop")

    match(
      type: "appstore",
      app_identifier: "com.company.myapp.staging",
      readonly: is_ci
    )

    increment_build_number(
      build_number: latest_testflight_build_number + 1
    )

    build_app(
      scheme: "MyApp (Staging)",
      configuration: "Staging",
      export_method: "app-store"
    )

    upload_to_testflight(
      skip_waiting_for_build_processing: true,
      changelog: git_changelog
    )
  end

  # MARK: - Production

  desc "Production ビルドとApp Store申請"
  lane :release do
    ensure_git_branch(branch: "main")
    ensure_git_status_clean

    version = prompt(text: "Enter version number (e.g. 1.2.0): ")
    increment_version_number(version_number: version)

    match(
      type: "appstore",
      app_identifier: "com.company.myapp",
      readonly: is_ci
    )

    increment_build_number(
      build_number: latest_testflight_build_number + 1
    )

    build_app(
      scheme: "MyApp (Production)",
      configuration: "Release",
      export_method: "app-store"
    )

    upload_to_testflight(
      skip_waiting_for_build_processing: false,
      changelog: git_changelog
    )

    add_git_tag(tag: "v#{version}")
    push_git_tags
  end

  # ヘルパーメソッド
  def git_changelog
    changelog_from_git_commits(
      between: [last_git_tag, "HEAD"],
      pretty: "- %s",
      merge_commit_filtering: "exclude_merges"
    )
  end
end
```

## 15.7 コード品質ツール

### 15.7.1 SwiftLint設定

```yaml
# .swiftlint.yml

# 除外パス
excluded:
  - Pods
  - Carthage
  - build
  - .build
  - DerivedData

# 無効化するルール
disabled_rules:
  - trailing_whitespace
  - todo

# オプトインルール
opt_in_rules:
  - empty_count
  - closure_spacing
  - explicit_init
  - attributes
  - closure_end_indentation
  - contains_over_first_not_nil
  - empty_string
  - fatal_error_message
  - first_where
  - implicit_return
  - multiline_arguments
  - sorted_imports
  - toggle_bool
  - trailing_closure

# カスタム設定
line_length:
  warning: 120
  error: 200
  ignores_comments: true

file_length:
  warning: 500
  error: 1000

function_body_length:
  warning: 50
  error: 100

cyclomatic_complexity:
  warning: 10
  error: 20

identifier_name:
  min_length:
    warning: 3
  max_length:
    warning: 40
    error: 50
  excluded:
    - id
    - x
    - y
    - z

# カスタムルール
custom_rules:
  no_print:
    name: "No Print"
    regex: "\\bprint\\("
    message: "Use Logger instead of print()"
    severity: warning

  no_force_try:
    name: "No Force Try"
    regex: "try!"
    message: "Avoid using try!"
    severity: error
```

### 15.7.2 SwiftFormat設定

```bash
# .swiftformat

# Version
--swiftversion 5.9

# Indentation
--indent 4
--indentcase false

# Wrapping
--maxwidth 120
--wraparguments before-first
--wrapcollections before-first
--wrapparameters before-first

# Spacing
--trimwhitespace always
--commas inline

# Parentheses
--closingparen same-line
--elseposition same-line
--guardelse same-line

# Organization
--importgrouping testable-bottom
--organizetypes class,struct,enum,extension
--self remove
--stripunusedargs closure-only

# Enabled rules
--enable isEmpty
--enable sortedImports
--enable redundantReturn
--enable redundantSelf

# Disabled rules
--disable andOperator
```

## 15.8 CI/CD初期設定

### 15.8.1 GitHub Actions設定

```yaml
# .github/workflows/ci.yml

name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  lint:
    name: SwiftLint
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4

      - name: SwiftLint
        run: |
          brew install swiftlint
          swiftlint --strict

  test:
    name: Unit Tests
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_15.2.app

      - name: Cache SPM
        uses: actions/cache@v3
        with:
          path: .build
          key: ${{ runner.os }}-spm-${{ hashFiles('**/Package.resolved') }}

      - name: Build and Test
        run: |
          xcodebuild test \
            -scheme MyApp \
            -destination 'platform=iOS Simulator,name=iPhone 15 Pro' \
            -enableCodeCoverage YES \
            -resultBundlePath TestResults

      - name: Upload Coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./TestResults/Coverage.txt

  build:
    name: Build
    runs-on: macos-14
    needs: [lint, test]
    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: |
          xcodebuild build \
            -scheme MyApp \
            -destination 'platform=iOS Simulator,name=iPhone 15 Pro'
```

### 15.8.2 TestFlightへの自動デプロイ

```yaml
# .github/workflows/deploy.yml

name: Deploy to TestFlight

on:
  push:
    branches: [ main ]
    tags:
      - 'v*'

jobs:
  deploy:
    name: Deploy
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true

      - name: Setup Fastlane
        run: bundle install

      - name: Sync Certificates
        env:
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          MATCH_GIT_BASIC_AUTHORIZATION: ${{ secrets.MATCH_GIT_BASIC_AUTHORIZATION }}
        run: bundle exec fastlane sync_certificates

      - name: Build and Deploy
        env:
          FASTLANE_APPLE_ID: ${{ secrets.APPLE_ID }}
          FASTLANE_PASSWORD: ${{ secrets.APPLE_PASSWORD }}
        run: bundle exec fastlane release
```

## 15.9 ドキュメンテーション

### 15.9.1 README.md作成

```markdown
# MyApp

## 概要

[プロジェクトの説明をここに記載]

## 要件

- Xcode 15.0+
- iOS 15.0+
- Swift 5.9+

## セットアップ

\`\`\`bash
# 依存関係のインストール
./scripts/setup.sh

# プロジェクトを開く
open MyApp.xcodeproj
\`\`\`

## アーキテクチャ

- **UI Framework**: SwiftUI
- **Architecture**: MVVM
- **Dependency Injection**: Manual
- **Navigation**: Coordinator Pattern

## フォルダ構成

\`\`\`
MyApp/
├── App/                    # App Entry Point
├── Features/               # 機能ごとの実装
├── Core/                   # 共通コンポーネント
├── Common/                 # ユーティリティ
├── Resources/              # リソースファイル
└── Supporting Files/       # 設定ファイル
\`\`\`

## 依存関係

### Swift Package Manager

- [Alamofire](https://github.com/Alamofire/Alamofire) - ネットワーキング
- [Kingfisher](https://github.com/onevcat/Kingfisher) - 画像読み込み
- [Firebase](https://github.com/firebase/firebase-ios-sdk) - Analytics, Crashlytics

## ビルドと実行

\`\`\`bash
# Debugビルド
xcodebuild -scheme MyApp -configuration Debug build

# Releaseビルド
xcodebuild -scheme MyApp -configuration Release build

# テスト実行
xcodebuild test -scheme MyApp -destination 'platform=iOS Simulator,name=iPhone 15 Pro'
\`\`\`

## CI/CD

GitHub Actionsを使用したCI/CDパイプライン

- **Lint**: SwiftLintによる静的解析
- **Test**: Unit Tests + UI Tests
- **Build**: Debug/Releaseビルド確認
- **Deploy**: TestFlightへの自動デプロイ

## コントリビューション

[CONTRIBUTING.md](CONTRIBUTING.md)を参照してください。

## ライセンス

[LICENSE](LICENSE)を参照してください。
```

### 15.9.2 CONTRIBUTING.md作成

```markdown
# Contributing Guide

## 開発環境のセットアップ

1. リポジトリをクローン
2. `./scripts/setup.sh` を実行
3. Xcodeでプロジェクトを開く

## ブランチ戦略

- `main`: 本番環境
- `develop`: 開発環境
- `feature/*`: 新機能
- `bugfix/*`: バグ修正
- `hotfix/*`: 緊急修正

## コミットメッセージ規約

Conventional Commitsに従う:

\`\`\`
<type>(<scope>): <subject>

<body>

<footer>
\`\`\`

Types:
- `feat`: 新機能
- `fix`: バグ修正
- `docs`: ドキュメント
- `style`: コードスタイル
- `refactor`: リファクタリング
- `test`: テスト
- `chore`: ビルド・設定

Example:
\`\`\`
feat(auth): add login functionality

- Add login screen
- Implement authentication service
- Add unit tests

Closes #123
\`\`\`

## コードレビュー

1. Pull Requestを作成
2. CIが通ることを確認
3. レビュアーを指定
4. 承認後にマージ

## テスト

- Unit Tests: 必須
- UI Tests: 重要な機能のみ
- カバレッジ: 80%以上を目標

## スタイルガイド

- SwiftLintに従う
- SwiftFormatで自動整形
- コメントは英語または日本語
```

## まとめ

この章では、iOSプロジェクトのセットアップとベストプラクティスについて詳しく解説しました。

**重要なポイント:**

1. **適切な初期設定**: 長期的な開発効率と保守性に大きく影響
2. **統一されたフォルダ構成**: MVVM/Clean Architectureに基づく整理
3. **依存関係管理**: SPMによるモダンな管理
4. **xcconfig活用**: 環境ごとの設定を分離
5. **Fastlane**: ビルド・デプロイの自動化
6. **コード品質ツール**: SwiftLint/SwiftFormatによる品質担保
7. **CI/CD**: GitHub Actionsによる自動化
8. **ドキュメンテーション**: README/CONTRIBUTINGの充実

適切なプロジェクトセットアップにより、開発効率が向上し、チーム全体で統一された開発環境を構築できます。
