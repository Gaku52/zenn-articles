---
title: "iOS依存関係管理ツール比較"
---

# iOS依存関係管理ツール比較

## パッケージマネージャー概要

iOS開発には、主に3つのパッケージマネージャーが使用されています。それぞれ異なる哲学とアプローチを持ち、プロジェクトの要件に応じて適切な選択が求められます。

### Swift Package Manager (SPM)

Apple公式のパッケージマネージャーで、Xcode 11以降にネイティブ統合されています。

**リリース**: 2015年
**開発元**: Apple
**言語**: Swift
**対応プラットフォーム**: iOS、macOS、watchOS、tvOS、Linux

### CocoaPods

最も成熟したサードパーティパッケージマネージャーです。

**リリース**: 2011年
**開発元**: コミュニティ
**言語**: Ruby
**対応プラットフォーム**: iOS、macOS、watchOS、tvOS

### Carthage

シンプルで分散型のパッケージマネージャーです。

**リリース**: 2014年
**開発元**: コミュニティ
**言語**: Swift
**対応プラットフォーム**: iOS、macOS、watchOS、tvOS

## 詳細比較表

### 基本機能

| 機能 | SPM | CocoaPods | Carthage |
|------|-----|-----------|----------|
| **公式サポート** | ✅ Apple公式 | ❌ コミュニティ | ❌ コミュニティ |
| **Xcode統合** | ✅ ネイティブ | ⚠️ ワークスペース | ❌ 手動 |
| **設定ファイル** | Package.swift | Podfile | Cartfile |
| **ロックファイル** | Package.resolved | Podfile.lock | Cartfile.resolved |
| **中央リポジトリ** | なし（分散型） | CocoaPods Trunk | なし（分散型） |
| **学習曲線** | 低 | 中 | 高 |

### ライブラリエコシステム

| 項目 | SPM | CocoaPods | Carthage |
|------|-----|-----------|----------|
| **登録ライブラリ数** | 中（増加中） | 多（9万以上） | 少 |
| **Swift専用ライブラリ** | ✅ 多い | ⚠️ 中程度 | ⚠️ 中程度 |
| **Objective-Cサポート** | ⚠️ 限定的 | ✅ 完全対応 | ✅ 完全対応 |
| **バイナリ配布** | ✅ XCFramework | ✅ Framework | ✅ Framework/XCFramework |
| **プライベートパッケージ** | ✅ 容易 | ✅ Spec Repo必要 | ✅ 容易 |

### ビルド・パフォーマンス

| 項目 | SPM | CocoaPods | Carthage |
|------|-----|-----------|----------|
| **初回ビルド時間** | 中 | 中 | 長（事前ビルド） |
| **増分ビルド時間** | 速い | 中 | 速い（ビルド済み） |
| **並列ビルド** | ✅ サポート | ⚠️ 限定的 | ✅ サポート |
| **ビルドキャッシュ** | ✅ あり | ⚠️ 限定的 | ✅ あり |
| **Xcodeプロジェクト変更** | なし | あり | なし |

### 開発体験

| 項目 | SPM | CocoaPods | Carthage |
|------|-----|-----------|----------|
| **セットアップの容易さ** | ✅ Xcode統合 | ⚠️ 別途インストール | ⚠️ 別途インストール |
| **依存関係の追加** | GUIで簡単 | Podfile編集 | Cartfile編集 |
| **バージョン管理** | セマンティック | セマンティック | セマンティック |
| **エラーメッセージ** | わかりやすい | わかりやすい | わかりにくい |
| **ドキュメント** | 豊富 | 非常に豊富 | 少ない |

## 実装例比較

### Package.swift (SPM)

```swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
    name: "MyLibrary",
    platforms: [
        .iOS(.v15),
        .macOS(.v12)
    ],
    products: [
        .library(
            name: "MyLibrary",
            targets: ["MyLibrary"]
        )
    ],
    dependencies: [
        .package(url: "https://github.com/Alamofire/Alamofire.git", from: "5.8.0"),
        .package(url: "https://github.com/onevcat/Kingfisher.git", from: "7.10.0")
    ],
    targets: [
        .target(
            name: "MyLibrary",
            dependencies: [
                "Alamofire",
                "Kingfisher"
            ]
        ),
        .testTarget(
            name: "MyLibraryTests",
            dependencies: ["MyLibrary"]
        )
    ]
)
```

### Podfile (CocoaPods)

```ruby
# Podfile
platform :ios, '15.0'
use_frameworks!

target 'MyApp' do
  pod 'Alamofire', '~> 5.8'
  pod 'Kingfisher', '~> 7.10'
  pod 'SnapKit', '~> 5.6'

  target 'MyAppTests' do
    inherit! :search_paths
    pod 'Quick', '~> 7.0'
    pod 'Nimble', '~> 12.0'
  end
end

post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '15.0'
    end
  end
end
```

### Cartfile (Carthage)

```
# Cartfile
github "Alamofire/Alamofire" ~> 5.8
github "onevcat/Kingfisher" ~> 7.10
github "SnapKit/SnapKit" ~> 5.6
```

## 選択基準

### Swift Package Manager推奨

以下の場合にSPMが推奨されます。

#### 新規プロジェクト

- Xcode 11以降を使用
- Swift中心の開発
- Apple公式のサポートを重視

#### 利点

```swift
// Package.swiftの例
// 1. Xcode統合でGUIから簡単に追加
// File → Add Package Dependencies...

// 2. 設定ファイルが不要（Xcode管理）

// 3. クリーンなプロジェクト構造
MyApp.xcodeproj/
├── MyApp/
│   └── Sources/
└── Packages/  # Xcode管理（自動）
```

#### 制約事項

- Objective-C専用ライブラリのサポートが限定的
- 一部の古いライブラリはSPM未対応
- Firebase SDKは一部のみSPM対応

### CocoaPods推奨

以下の場合にCocoaPodsが推奨されます。

#### 既存プロジェクト

- CocoaPodsで長期運用中
- 移行コストが高い
- Objective-Cライブラリが必要

#### 利点

```ruby
# Podfile
# 1. 豊富なライブラリエコシステム（9万以上）
pod 'Firebase/Analytics'  # SPM未対応の機能も利用可能
pod 'Realm', '~> 10.45'

# 2. post_installで細かい制御
post_install do |installer|
  # ビルド設定のカスタマイズが容易
end

# 3. サブスペック対応
pod 'Firebase/Analytics'
pod 'Firebase/Crashlytics'
```

#### 制約事項

- Ruby環境が必要
- Xcodeワークスペースを生成
- ビルド時間がやや長い

### Carthage推奨

以下の場合にCarthageが推奨されます。

#### 特殊なケース

- Xcodeプロジェクトを変更したくない
- ビルド済みフレームワークを使用したい
- 最小限の依存関係管理

#### 利点

```
# Cartfile
# 1. シンプルな設定
github "Alamofire/Alamofire" ~> 5.8

# 2. Xcodeプロジェクト非変更
# プロジェクトファイルは一切変更されない

# 3. ビルド済みフレームワーク
# 初回ビルド後は高速
```

#### 制約事項

- 手動統合が必要（Run Script Phase）
- 初回ビルドに時間がかかる
- ライブラリ数が少ない
- メンテナンスが停滞気味

## 移行戦略

### CocoaPods → SPM

段階的な移行が推奨されます。

#### Phase 1: SPM対応ライブラリを移行

```ruby
# Podfile（移行途中）
platform :ios, '15.0'

target 'MyApp' do
  # SPMに移行済み（Xcodeで管理）
  # - Alamofire
  # - Kingfisher
  # - SnapKit

  # まだCocoaPods
  pod 'Firebase/Analytics'
  pod 'Firebase/Crashlytics'
  pod 'Realm', '~> 10.45'
end
```

#### Phase 2: 代替ライブラリを検討

```swift
// Firebase Crashlytics → OSLog + Sentry等
import OSLog
import Sentry

// Realm → SwiftData（iOS 17+）
import SwiftData
```

#### Phase 3: 完全移行

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/Alamofire/Alamofire.git", from: "5.8.0"),
    .package(url: "https://github.com/onevcat/Kingfisher.git", from: "7.10.0"),
    .package(url: "https://github.com/getsentry/sentry-cocoa.git", from: "8.0.0")
]
```

### Carthage → SPM

Carthageは比較的移行が容易です。

```bash
# 1. Cartfileから依存関係を確認
cat Cartfile

# 2. Xcodeで同じライブラリをSPMで追加
# File → Add Package Dependencies...

# 3. Cartfileを削除
rm Cartfile Cartfile.resolved
rm -rf Carthage
```

## ハイブリッド構成

複数のパッケージマネージャーを併用することも可能です。

### SPM + CocoaPods

```ruby
# Podfile
platform :ios, '15.0'
use_frameworks!

target 'MyApp' do
  # CocoaPods経由（SPM未対応）
  pod 'Firebase/Analytics'
  pod 'Firebase/Crashlytics'

  # SPMで管理（Xcode）
  # - Alamofire
  # - Kingfisher
end
```

### SPM + Carthage

```
# Cartfile
# Carthage経由（特殊なビルド設定が必要）
github "ReactiveCocoa/ReactiveCocoa" ~> 11.0

# SPMで管理
# - Alamofire
# - Kingfisher
```

### 推奨される組み合わせ

| 組み合わせ | 推奨度 | 理由 |
|-----------|--------|------|
| SPM + CocoaPods | ⭐⭐⭐⭐⭐ | 最も一般的で安定 |
| SPM + Carthage | ⭐⭐⭐ | 可能だが複雑 |
| CocoaPods + Carthage | ⭐⭐ | 非推奨（複雑） |
| SPM + CocoaPods + Carthage | ⭐ | 避けるべき |

## パフォーマンス比較

### 想定されるビルド時間

中規模プロジェクト（依存関係15個）の想定値:

| 項目 | SPM | CocoaPods | Carthage |
|------|-----|-----------|----------|
| **初回ビルド** | 3-5分 | 4-6分 | 8-12分 |
| **クリーンビルド** | 3-5分 | 4-6分 | 1-2分（ビルド済み） |
| **増分ビルド** | 10-30秒 | 30-60秒 | 10-30秒 |
| **依存関係更新** | 1-3分 | 2-4分 | 5-10分 |

注意: これらは想定値であり、プロジェクト規模や環境により大きく異なります。

### ディスク容量

| 項目 | SPM | CocoaPods | Carthage |
|------|-----|-----------|----------|
| **ソースコード** | 中 | 中 | 中 |
| **ビルド成果物** | 中 | 大 | 小（バイナリ） |
| **キャッシュ** | 中 | 大 | 中 |

## 最終推奨

### 新規プロジェクト

**推奨**: Swift Package Manager

理由:
- Apple公式サポート
- Xcode統合で設定不要
- 将来性が高い

### 既存プロジェクト（CocoaPods使用中）

**推奨**: CocoaPods継続 + 段階的SPM移行

理由:
- 移行コストが高い
- 安定稼働中なら無理に移行不要
- 新しい依存関係はSPMで追加

### 既存プロジェクト（Carthage使用中）

**推奨**: SPMへ完全移行

理由:
- Carthageのメンテナンスが停滞
- SPMへの移行が比較的容易
- 長期的な保守性向上

## まとめ

iOS依存関係管理ツールの選択は、プロジェクトの状況に応じて適切に判断すべきです。新規プロジェクトではSPMが推奨されますが、既存プロジェクトでは現在のツールを継続しつつ段階的に移行することが現実的です。

選択基準のまとめ:
- **新規・Swift中心**: Swift Package Manager
- **既存・多機能**: CocoaPods継続 + SPM併用
- **シンプル・制御重視**: Carthage（ただし非推奨傾向）

次章では、セキュリティ脆弱性スキャンについて解説します。

## 参考文献

- [Swift Package Manager Documentation](https://www.swift.org/package-manager/)
- [CocoaPods Official Guide](https://guides.cocoapods.org/)
- [Carthage Documentation](https://github.com/Carthage/Carthage)
- [Apple Developer - Distributing Binary Frameworks as Swift Packages](https://developer.apple.com/documentation/xcode/distributing-binary-frameworks-as-swift-packages)
- [CocoaPods vs Carthage vs SPM - NSHipster](https://nshipster.com/swift-package-manager/)
