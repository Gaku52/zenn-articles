---
title: "ビルド設定と環境管理"
---

# ビルド設定と環境管理

開発、ステージング、本番など、複数の環境を適切に管理することは、プロフェッショナルなアプリ開発において不可欠です。本章では、Build ConfigurationとSchemeを使った環境管理の手法を解説します。

## Build Configuration の基礎

Xcodeは標準でDebugとReleaseの2つのConfigurationを提供しますが、実務ではより細かい環境分けが推奨されます。

### 推奨される Configuration 構成

```plaintext
- Debug（開発環境）
- Staging（検証環境）
- Release（本番環境）
```

## Custom Configuration の作成

### Step 1: Configuration の複製

プロジェクト設定から新しいConfigurationを作成します。

```plaintext
1. Project > Info タブを開く
2. Configurations セクションで「+」をクリック
3. 「Duplicate "Release" Configuration」を選択
4. 名前を「Staging」に変更
```

### Step 2: xcconfig ファイルの作成

各環境用のxconfigファイルを作成します。

**Config/Debug.xcconfig:**

```plaintext
#include "Base.xcconfig"

// Configuration Name
CONFIGURATION_NAME = Debug

// App Information
PRODUCT_BUNDLE_IDENTIFIER = com.company.myapp.dev
PRODUCT_NAME = $(APP_NAME) Dev
ASSETCATALOG_COMPILER_APPICON_NAME = AppIcon-Dev

// Build Settings
SWIFT_OPTIMIZATION_LEVEL = -Onone
SWIFT_COMPILATION_MODE = incremental
SWIFT_ACTIVE_COMPILATION_CONDITIONS = DEBUG
GCC_OPTIMIZATION_LEVEL = 0
DEBUG_INFORMATION_FORMAT = dwarf

// API Configuration
API_BASE_URL = https:/$()/api-dev.example.com
API_KEY = dev_api_key_12345

// Feature Flags
ENABLE_ANALYTICS = NO
ENABLE_CRASH_REPORTING = NO
DEBUG_MENU_ENABLED = YES
```

**Config/Staging.xcconfig:**

```plaintext
#include "Base.xcconfig"

// Configuration Name
CONFIGURATION_NAME = Staging

// App Information
PRODUCT_BUNDLE_IDENTIFIER = com.company.myapp.staging
PRODUCT_NAME = $(APP_NAME) Staging
ASSETCATALOG_COMPILER_APPICON_NAME = AppIcon-Staging

// Build Settings
SWIFT_OPTIMIZATION_LEVEL = -O
SWIFT_COMPILATION_MODE = wholemodule
GCC_OPTIMIZATION_LEVEL = s
DEBUG_INFORMATION_FORMAT = dwarf-with-dsym

// API Configuration
API_BASE_URL = https:/$()/api-staging.example.com
API_KEY = staging_api_key_67890

// Feature Flags
ENABLE_ANALYTICS = YES
ENABLE_CRASH_REPORTING = YES
DEBUG_MENU_ENABLED = YES
```

**Config/Release.xcconfig:**

```plaintext
#include "Base.xcconfig"

// Configuration Name
CONFIGURATION_NAME = Release

// App Information
PRODUCT_BUNDLE_IDENTIFIER = com.company.myapp
PRODUCT_NAME = $(APP_NAME)
ASSETCATALOG_COMPILER_APPICON_NAME = AppIcon

// Build Settings
SWIFT_OPTIMIZATION_LEVEL = -O
SWIFT_COMPILATION_MODE = wholemodule
GCC_OPTIMIZATION_LEVEL = s
DEBUG_INFORMATION_FORMAT = dwarf-with-dsym

// Code Stripping
DEAD_CODE_STRIPPING = YES
COPY_PHASE_STRIP = YES
STRIP_INSTALLED_PRODUCT = YES

// API Configuration
API_BASE_URL = https:/$()/api.example.com
API_KEY = prod_api_key_abcde

// Feature Flags
ENABLE_ANALYTICS = YES
ENABLE_CRASH_REPORTING = YES
DEBUG_MENU_ENABLED = NO
```

**Config/Base.xcconfig:**

```plaintext
// Base Configuration
SWIFT_VERSION = 5.9
IPHONEOS_DEPLOYMENT_TARGET = 17.0
TARGETED_DEVICE_FAMILY = 1,2

// App Name
APP_NAME = MyAwesomeApp

// Warnings
CLANG_WARN_BLOCK_CAPTURE_AUTORELEASING = YES
CLANG_WARN_BOOL_CONVERSION = YES
CLANG_WARN_COMMA = YES
CLANG_WARN_CONSTANT_CONVERSION = YES
CLANG_WARN_DEPRECATED_OBJC_IMPLEMENTATIONS = YES
CLANG_WARN_EMPTY_BODY = YES
CLANG_WARN_ENUM_CONVERSION = YES
CLANG_WARN_INFINITE_RECURSION = YES
CLANG_WARN_INT_CONVERSION = YES
CLANG_WARN_NON_LITERAL_NULL_CONVERSION = YES
CLANG_WARN_OBJC_LITERAL_CONVERSION = YES
CLANG_WARN_RANGE_LOOP_ANALYSIS = YES
CLANG_WARN_STRICT_PROTOTYPES = YES
CLANG_WARN_SUSPICIOUS_MOVE = YES
CLANG_WARN_UNREACHABLE_CODE = YES
CLANG_WARN__DUPLICATE_METHOD_MATCH = YES

// Swift Warnings
SWIFT_TREAT_WARNINGS_AS_ERRORS = NO
GCC_TREAT_WARNINGS_AS_ERRORS = NO
```

## Scheme の設定

環境ごとにSchemeを作成することで、簡単に切り替えられます。

### Scheme の作成

```plaintext
1. Product > Scheme > Manage Schemes
2. 「+」をクリック
3. Target を選択
4. Scheme 名を「MyApp (Staging)」に設定
```

### Scheme の詳細設定

**Debug Scheme:**

```plaintext
Product > Scheme > Edit Scheme > MyApp (Debug)

Run:
  Build Configuration: Debug
  Executable: MyApp.app
  Debugger: LLDB

Test:
  Build Configuration: Debug
  Code Coverage: ✓

Profile:
  Build Configuration: Debug

Analyze:
  Build Configuration: Debug

Archive:
  Build Configuration: Debug
```

**Staging Scheme:**

```plaintext
Product > Scheme > Edit Scheme > MyApp (Staging)

Run:
  Build Configuration: Staging

Archive:
  Build Configuration: Staging
```

**Release Scheme:**

```plaintext
Product > Scheme > Edit Scheme > MyApp (Release)

Run:
  Build Configuration: Release

Archive:
  Build Configuration: Release
```

## Info.plist での環境変数の利用

xcconfig で定義した変数を Info.plist で参照できます。

```xml
<key>CFBundleDisplayName</key>
<string>$(PRODUCT_NAME)</string>

<key>CFBundleIdentifier</key>
<string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>

<key>APIBaseURL</key>
<string>$(API_BASE_URL)</string>

<key>APIKey</key>
<string>$(API_KEY)</string>
```

## Swift コードでの環境変数の取得

Info.plist に設定した値を Swift コードから取得します。

```swift
// Core/Configuration/AppConfiguration.swift
import Foundation

enum AppConfiguration {
    // MARK: - API Settings

    static var apiBaseURL: String {
        guard let urlString = Bundle.main.object(forInfoDictionaryKey: "APIBaseURL") as? String else {
            fatalError("APIBaseURL not found in Info.plist")
        }
        return urlString
    }

    static var apiKey: String {
        guard let key = Bundle.main.object(forInfoDictionaryKey: "APIKey") as? String else {
            fatalError("APIKey not found in Info.plist")
        }
        return key
    }

    // MARK: - Build Configuration

    static var isDebug: Bool {
        #if DEBUG
        return true
        #else
        return false
        #endif
    }

    static var isStaging: Bool {
        Bundle.main.bundleIdentifier?.contains(".staging") ?? false
    }

    static var isProduction: Bool {
        !isDebug && !isStaging
    }

    // MARK: - Feature Flags

    static var analyticsEnabled: Bool {
        #if DEBUG
        return false
        #else
        return true
        #endif
    }

    static var debugMenuEnabled: Bool {
        #if DEBUG
        return true
        #else
        return isStaging
        #endif
    }
}
```

### 使用例

```swift
// API通信での利用
class NetworkManager {
    private let baseURL: String

    init() {
        self.baseURL = AppConfiguration.apiBaseURL
    }

    func request(_ endpoint: String) async throws -> Data {
        guard let url = URL(string: baseURL + endpoint) else {
            throw NetworkError.invalidURL
        }

        var request = URLRequest(url: url)
        request.setValue(AppConfiguration.apiKey, forHTTPHeaderField: "X-API-Key")

        let (data, _) = try await URLSession.shared.data(for: request)
        return data
    }
}
```

## 環境別 App Icon の設定

各環境でアプリアイコンを変えることで、視覚的に区別できます。

### Assets.xcassets の設定

```plaintext
Assets.xcassets/
├── AppIcon.appiconset/       # 本番用
├── AppIcon-Dev.appiconset/   # 開発用（バッジ付き）
└── AppIcon-Staging.appiconset/ # Staging用（バッジ付き）
```

### xcconfig での指定

```plaintext
// Debug.xcconfig
ASSETCATALOG_COMPILER_APPICON_NAME = AppIcon-Dev

// Staging.xcconfig
ASSETCATALOG_COMPILER_APPICON_NAME = AppIcon-Staging

// Release.xcconfig
ASSETCATALOG_COMPILER_APPICON_NAME = AppIcon
```

## 環境切り替えの実装

開発中に環境を簡単に切り替えられる仕組みを実装します。

```swift
// Core/Configuration/Environment.swift
enum Environment: String, CaseIterable {
    case development
    case staging
    case production

    var displayName: String {
        switch self {
        case .development: return "Development"
        case .staging: return "Staging"
        case .production: return "Production"
        }
    }

    var baseURL: String {
        switch self {
        case .development: return "https://api-dev.example.com"
        case .staging: return "https://api-staging.example.com"
        case .production: return "https://api.example.com"
        }
    }

    var apiKey: String {
        switch self {
        case .development: return "dev_key"
        case .staging: return "staging_key"
        case .production: return "prod_key"
        }
    }

    static var current: Environment {
        #if DEBUG
        // 開発時は UserDefaults で切り替え可能
        if let savedEnv = UserDefaults.standard.string(forKey: "selectedEnvironment"),
           let environment = Environment(rawValue: savedEnv) {
            return environment
        }
        return .development
        #else
        if AppConfiguration.isStaging {
            return .staging
        }
        return .production
        #endif
    }
}
```

### 環境切り替え UI（Debug ビルドのみ）

```swift
// Features/Debug/EnvironmentSelectorView.swift
#if DEBUG
import SwiftUI

struct EnvironmentSelectorView: View {
    @State private var selectedEnvironment: Environment = .current

    var body: some View {
        List {
            Section("環境選択") {
                ForEach(Environment.allCases, id: \.self) { env in
                    HStack {
                        Text(env.displayName)
                        Spacer()
                        if env == selectedEnvironment {
                            Image(systemName: "checkmark")
                                .foregroundColor(.blue)
                        }
                    }
                    .contentShape(Rectangle())
                    .onTapGesture {
                        selectEnvironment(env)
                    }
                }
            }

            Section("現在の設定") {
                LabeledContent("Base URL", value: selectedEnvironment.baseURL)
                LabeledContent("API Key", value: selectedEnvironment.apiKey)
            }
        }
        .navigationTitle("環境設定")
    }

    private func selectEnvironment(_ env: Environment) {
        selectedEnvironment = env
        UserDefaults.standard.set(env.rawValue, forKey: "selectedEnvironment")

        // アプリを再起動する必要があることを通知
        showRestartAlert()
    }

    private func showRestartAlert() {
        // アラート表示の実装
    }
}
#endif
```

## ビルド設定の検証

環境ごとの設定が正しく適用されているか確認します。

```swift
// Core/Configuration/ConfigurationValidator.swift
struct ConfigurationValidator {
    static func validate() {
        print("=== Build Configuration ===")
        print("Environment: \(Environment.current.displayName)")
        print("Base URL: \(AppConfiguration.apiBaseURL)")
        print("Bundle ID: \(Bundle.main.bundleIdentifier ?? "Unknown")")
        print("Is Debug: \(AppConfiguration.isDebug)")
        print("Analytics Enabled: \(AppConfiguration.analyticsEnabled)")
        print("========================")
    }
}

// AppDelegate または App.swift で呼び出し
init() {
    #if DEBUG
    ConfigurationValidator.validate()
    #endif
}
```

## まとめ

本章では、Build ConfigurationとSchemeを使った環境管理の手法を解説しました。適切な環境管理により、以下の効果が期待されます：

- 環境ごとの設定を明確に分離
- 開発効率の向上
- 本番環境へのリリース時のミス防止
- チーム開発での設定統一

次章では、ビルド設定の最適化について詳しく解説します。
