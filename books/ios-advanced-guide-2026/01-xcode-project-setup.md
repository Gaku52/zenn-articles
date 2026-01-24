---
title: "Xcodeプロジェクトの初期設定"
---

# Xcodeプロジェクトの初期設定

プロジェクトの初期設定は、その後の開発効率と保守性に大きな影響を与えます。本章では、Xcodeプロジェクトの作成から基本設定まで、プロフェッショナルなプロジェクトセットアップの手順を解説します。

## プロジェクト作成前のチェックリスト

新規プロジェクトを作成する前に、以下の項目を明確にしておくことが推奨されます。

### プロジェクト情報の決定

- **プロジェクト名**: アプリケーションの正式名称
- **Bundle Identifier**: 逆DNS形式（例: `com.company.appname`）
- **Organization Name**: 組織名または個人名
- **Organization Identifier**: 組織のドメイン（例: `com.company`）

### 技術選定

- **UIフレームワーク**: SwiftUI / UIKit
- **ライフサイクル**: SwiftUI App / UIKit App Delegate
- **言語**: Swift（推奨）
- **最小デプロイメントターゲット**: iOS 17.0以降を推奨

## プロジェクトの作成

### Step 1: 新規プロジェクトの作成

Xcodeを起動し、新規プロジェクトを作成します。

```plaintext
File > New > Project > iOS > App
```

### Step 2: プロジェクト情報の入力

以下の情報を入力します：

```plaintext
Product Name: MyAwesomeApp
Team: （開発チームを選択）
Organization Identifier: com.company
Bundle Identifier: com.company.myawesomeapp
Interface: SwiftUI
Language: Swift
Storage: None（初期段階ではCore Dataは選択しない）
```

### Step 3: 保存場所の選択

プロジェクトを保存するディレクトリを選択し、「Create Git repository on my Mac」にチェックを入れます。これにより、初期状態からバージョン管理が有効になります。

## プロジェクト設定の最適化

プロジェクト作成後、Generalタブで以下の設定を確認・調整します。

### Identity設定

```plaintext
Display Name: My Awesome App
Bundle Identifier: com.company.myawesomeapp
Version: 1.0.0
Build: 1
```

**バージョン管理のベストプラクティス:**
- Version: セマンティックバージョニング（例: 1.0.0）
- Build: 連番で管理（リリース毎にインクリメント）

### Deployment Info

```plaintext
Minimum Deployments: iOS 17.0
Supported Destinations: iPhone, iPad
Device Orientation: Portrait（必要に応じて追加）
Status Bar Style: Default
```

**デプロイメントターゲットの選択:**
- iOS 17.0以降を推奨（最新機能の活用）
- サポート範囲を広げたい場合はiOS 16.0以降

### App Icons and Launch Screen

```plaintext
App Icons Source: AppIcon（Assets.xcassetsで管理）
Launch Screen: LaunchScreen.storyboard
```

## Signing & Capabilities設定

適切なCode Signingの設定は、開発とリリースの円滑化に不可欠です。

### Automatically manage signing（推奨）

```plaintext
☑ Automatically manage signing
Team: （開発チームを選択）
Provisioning Profile: Xcode Managed Profile
Signing Certificate: Apple Development
```

**自動署名のメリット:**
- 証明書とプロビジョニングプロファイルを自動管理
- チーム開発での環境構築が容易
- 期限切れの自動更新

### Capabilitiesの追加

必要な機能に応じてCapabilitiesを追加します：

```plaintext
+ Capability
- Push Notifications（プッシュ通知）
- Background Modes（バックグラウンド処理）
- Keychain Sharing（Keychain共有）
- App Groups（アプリグループ）
```

## Build Settings初期設定

Build Settingsタブで、プロジェクト全体に適用される設定を行います。

### Swift Compiler設定

```plaintext
Swift Language Version: Swift 5
Swift Compiler - Code Generation
  - Optimization Level (Debug): No Optimization [-Onone]
  - Optimization Level (Release): Optimize for Speed [-O]
  - Compilation Mode (Debug): Incremental
  - Compilation Mode (Release): Whole Module
```

**最適化レベルの推奨設定:**
- Debug: 最適化なし（ビルド時間優先）
- Release: 速度優先（パフォーマンス最大化）

### Linking設定

```plaintext
Dead Code Stripping (Release): Yes
Strip Debug Symbols During Copy (Release): Yes
Strip Swift Symbols (Release): Yes
```

これらの設定により、リリースビルドのバイナリサイズが削減され、パフォーマンスが向上することが期待されます。

## Info.plistの設定

Info.plistには、アプリの動作に必要な設定と権限を記述します。

### 基本設定

```xml
<key>CFBundleDisplayName</key>
<string>My Awesome App</string>

<key>CFBundleShortVersionString</key>
<string>$(MARKETING_VERSION)</string>

<key>CFBundleVersion</key>
<string>$(CURRENT_PROJECT_VERSION)</string>
```

### Privacy設定（必須）

ユーザーのプライバシーデータにアクセスする場合、Usage Descriptionの記述が必須です：

```xml
<key>NSCameraUsageDescription</key>
<string>写真を撮影するためにカメラへのアクセスが必要です</string>

<key>NSPhotoLibraryUsageDescription</key>
<string>写真を保存するためにフォトライブラリへのアクセスが必要です</string>

<key>NSLocationWhenInUseUsageDescription</key>
<string>現在地を表示するために位置情報へのアクセスが必要です</string>
```

**SwiftUIでの記述方法:**

Xcode 14以降では、Info.plistの代わりにTargetのInfoタブから設定できます。

## Schemeの設定

Schemeは、ビルド、実行、テスト、アーカイブの各フェーズの設定を管理します。

### Scheme編集

```plaintext
Product > Scheme > Edit Scheme
```

### Run設定

```plaintext
Build Configuration: Debug
Debugger: LLDB
Environment Variables:
  - API_BASE_URL: https://api-dev.example.com
  - DEBUG_MODE: 1
```

環境変数を使用することで、環境ごとに異なる設定を簡単に切り替えられます。

### Test設定

```plaintext
Build Configuration: Debug
Code Coverage: ☑ Gather coverage data
Test Plans: Default
```

### Archive設定

```plaintext
Build Configuration: Release
Reveal Archive in Organizer: ☑
```

## xcconfig ファイルの導入

xconfigファイルを使用すると、ビルド設定を外部ファイルで管理でき、可読性と保守性が向上します。

### ファイル構成

```plaintext
Config/
├── Base.xcconfig
├── Debug.xcconfig
└── Release.xcconfig
```

### Base.xcconfig

```plaintext
// 全環境共通の設定
SWIFT_VERSION = 5.0
IPHONEOS_DEPLOYMENT_TARGET = 17.0
TARGETED_DEVICE_FAMILY = 1,2

// Warnings
CLANG_WARN_BLOCK_CAPTURE_AUTORELEASING = YES
CLANG_WARN_BOOL_CONVERSION = YES
CLANG_WARN_COMMA = YES
CLANG_WARN_CONSTANT_CONVERSION = YES
CLANG_WARN_DEPRECATED_OBJC_IMPLEMENTATIONS = YES
CLANG_WARN_EMPTY_BODY = YES
CLANG_WARN_ENUM_CONVERSION = YES
CLANG_WARN_INFINITE_RECURSION = YES
```

### Debug.xcconfig

```plaintext
#include "Base.xcconfig"

// 開発環境用設定
SWIFT_OPTIMIZATION_LEVEL = -Onone
SWIFT_COMPILATION_MODE = incremental
SWIFT_ACTIVE_COMPILATION_CONDITIONS = DEBUG
DEBUG_INFORMATION_FORMAT = dwarf

// API Endpoints
API_BASE_URL = https:/$()/api-dev.example.com
```

### Release.xcconfig

```plaintext
#include "Base.xcconfig"

// 本番環境用設定
SWIFT_OPTIMIZATION_LEVEL = -O
SWIFT_COMPILATION_MODE = wholemodule
DEBUG_INFORMATION_FORMAT = dwarf-with-dsym

// Code Stripping
DEAD_CODE_STRIPPING = YES
COPY_PHASE_STRIP = YES

// API Endpoints
API_BASE_URL = https:/$()/api.example.com
```

### xconfigファイルの適用

プロジェクト設定でxconfigファイルを指定します：

```plaintext
Project > Info > Configurations
Debug: Config/Debug.xcconfig
Release: Config/Release.xcconfig
```

## プロジェクトファイルの整理

.gitignoreファイルを作成し、不要なファイルをバージョン管理から除外します。

### .gitignore

```gitignore
# Xcode
*.xcuserstate
*.xcuserdatad/
DerivedData/
*.xcworkspace/xcuserdata/

# CocoaPods
Pods/
*.xcworkspace

# Swift Package Manager
.swiftpm/
.build/

# Fastlane
fastlane/report.xml
fastlane/Preview.html
fastlane/screenshots
fastlane/test_output

# macOS
.DS_Store

# Environment Variables
.env
.env.local
```

## 初回コミット

プロジェクトの初期設定が完了したら、Gitで初回コミットを作成します。

```bash
git add .
git commit -m "feat(init): initial project setup

- Create Xcode project with SwiftUI
- Configure build settings
- Add xcconfig files
- Setup .gitignore"
```

## まとめ

本章では、Xcodeプロジェクトの作成から初期設定、xconfigファイルの導入、Gitセットアップまでを解説しました。適切な初期設定により、以下のメリットが期待されます：

- ビルド設定の一元管理
- 環境ごとの設定切り替えの容易化
- チーム開発での設定統一
- 将来的な拡張性の確保

次章では、プロジェクトのフォルダ構成とベストプラクティスについて解説します。
