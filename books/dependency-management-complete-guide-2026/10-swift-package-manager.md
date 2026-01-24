---
title: "Swift Package Manager実践"
---

# Swift Package Manager実践

## Swift Package Managerとは

Swift Package Manager（SPM）は、Appleが提供する公式のSwift用パッケージマネージャーです。Xcode 11以降でネイティブサポートされています。

### SPMの特徴

- Swift公式（Apple製）
- Xcodeに統合
- 依存関係の自動解決
- マルチプラットフォーム対応

## Package.swiftの基本

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "MyLibrary",
    
    platforms: [
        .iOS(.v16),
        .macOS(.v13)
    ],
    
    products: [
        .library(
            name: "MyLibrary",
            targets: ["MyLibrary"]
        )
    ],
    
    dependencies: [
        .package(
            url: "https://github.com/Alamofire/Alamofire.git",
            from: "5.8.0"
        )
    ],
    
    targets: [
        .target(
            name: "MyLibrary",
            dependencies: ["Alamofire"]
        ),
        .testTarget(
            name: "MyLibraryTests",
            dependencies: ["MyLibrary"]
        )
    ]
)
```

## 依存関係の追加方法

### Xcode GUI

1. プロジェクトを選択
2. Package Dependencies タブ
3. "+" ボタンをクリック
4. GitHubのURLを入力
5. バージョンを選択

### コマンドライン

```bash
# パッケージを追加
swift package add-dependency \
  --url https://github.com/Alamofire/Alamofire.git \
  --from 5.8.0

# 依存関係を解決
swift package resolve

# パッケージを更新
swift package update

# ビルド
swift build

# テスト実行
swift test
```

## バージョン指定

```swift
dependencies: [
    // 特定バージョン
    .package(url: "...", exact: "5.8.0"),
    
    // 最小バージョン指定
    .package(url: "...", from: "5.8.0"),
    
    // 範囲指定
    .package(url: "...", "5.0.0"..<"6.0.0"),
    
    // メジャーバージョン固定
    .package(url: "...", .upToNextMajor(from: "5.8.0")),
    
    // マイナーバージョン固定
    .package(url: "...", .upToNextMinor(from: "5.8.0")),
    
    // ブランチ指定（開発用）
    .package(url: "...", branch: "develop"),
    
    // コミット指定
    .package(url: "...", revision: "abc123")
]
```

## Package.resolved

Package.resolvedは、SPMのロックファイルです。

```json
{
  "pins": [
    {
      "identity": "alamofire",
      "kind": "remoteSourceControl",
      "location": "https://github.com/Alamofire/Alamofire.git",
      "state": {
        "revision": "bc268c28fb170f494de9e9927c371b8342979ece",
        "version": "5.8.1"
      }
    }
  ],
  "version": 2
}
```

### ロックファイルの管理

```bash
# Gitにコミット（推奨）
git add Package.resolved
git commit -m "chore: update dependencies"

# 依存関係を完全に再解決
rm -rf .build
rm Package.resolved
swift package resolve
```

## まとめ

SPMは、Swift開発における標準的なパッケージマネージャーです。

次章では、CocoaPodsの管理について解説します。
