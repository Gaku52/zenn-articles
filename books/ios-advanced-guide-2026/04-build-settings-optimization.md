---
title: "ビルド設定の最適化"
---

# ビルド設定の最適化

ビルド時間の短縮とアプリのパフォーマンス向上は、開発効率とユーザー体験の両面で重要です。本章では、Build Settingsの最適化手法を解説します。

## ビルド時間の最適化

開発中のビルド時間を短縮することで、開発者の生産性が向上することが期待されます。

### Swift Compiler - Code Generation

**Debug ビルドでの推奨設定:**

```plaintext
Optimization Level: No Optimization [-Onone]
  理由: コンパイル時間を優先

Compilation Mode: Incremental
  理由: 変更されたファイルのみを再コンパイル

Enable Testability: Yes
  理由: Unit Test でのアクセス性確保
```

**Release ビルドでの推奨設定:**

```plaintext
Optimization Level: Optimize for Speed [-O]
  理由: 実行速度を最大化

Compilation Mode: Whole Module
  理由: モジュール全体を最適化

Enable Testability: No
  理由: バイナリサイズの削減
```

### Compilation Mode の詳細

#### Incremental（増分コンパイル）

```plaintext
メリット:
- ビルド時間が大幅に短縮される
- 変更したファイルのみを再コンパイル

デメリット:
- 最適化の機会が限定的

推奨用途: Debug ビルド
```

#### Whole Module（全モジュールコンパイル）

```plaintext
メリット:
- モジュール全体を最適化できる
- パフォーマンスが向上する
- バイナリサイズが小さくなる

デメリット:
- ビルド時間が長くなる

推奨用途: Release ビルド
```

### xcconfig での設定

```plaintext
// Debug.xcconfig
SWIFT_OPTIMIZATION_LEVEL = -Onone
SWIFT_COMPILATION_MODE = incremental
ENABLE_TESTABILITY = YES

// Release.xcconfig
SWIFT_OPTIMIZATION_LEVEL = -O
SWIFT_COMPILATION_MODE = wholemodule
ENABLE_TESTABILITY = NO
```

## ビルド時間の計測

どの部分が時間を要しているか把握することが最適化の第一歩です。

### ビルド時間の表示

ターミナルから以下のコマンドを実行します：

```bash
defaults write com.apple.dt.Xcode ShowBuildOperationDuration -bool YES
```

これにより、Xcodeのステータスバーにビルド時間が表示されます。

### 関数単位のコンパイル時間計測

Build Settings で以下を追加：

```plaintext
Other Swift Flags: -Xfrontend -debug-time-function-bodies -Xfrontend -debug-time-compilation
```

ビルド後、Build Logに各関数のコンパイル時間が表示されます。

### ビルド時間が長い関数の特定

```bash
# ビルドログから時間のかかる関数を抽出
xcodebuild -workspace MyApp.xcworkspace \
  -scheme MyApp \
  clean build \
  | grep -E "^[0-9]+\.[0-9]+ms" \
  | sort -nr \
  | head -20
```

## 並列ビルドの最適化

### Build Active Architecture Only

```plaintext
Debug: Yes
  理由: 開発中は現在のアーキテクチャのみビルド

Release: No
  理由: 全アーキテクチャをサポート
```

### Parallelization の有効化

```plaintext
Build Settings > Build Options
Enable Parallel Build: Yes
```

これにより、複数のターゲットを並列でビルドすることが期待されます。

## Derived Data の管理

Derived Data の肥大化はビルド時間に影響を与えます。

### 定期的なクリーンアップ

```bash
# Derived Data の完全削除
rm -rf ~/Library/Developer/Xcode/DerivedData

# プロジェクト固有の Derived Data のみ削除
rm -rf ~/Library/Developer/Xcode/DerivedData/MyApp-*
```

### Xcode からのクリーン

```plaintext
Product > Clean Build Folder (Shift + Command + K)
```

## 依存関係の最適化

### Swift Package Manager のキャッシュ

SPM はパッケージをキャッシュすることで、2回目以降のビルドが高速化されます。

```bash
# パッケージキャッシュの場所
~/Library/Caches/org.swift.swiftpm

# キャッシュのクリア（問題がある場合）
rm -rf ~/Library/Caches/org.swift.swiftpm
```

### CocoaPods の最適化

```ruby
# Podfile
use_frameworks!
use_modular_headers!

# ビルド設定の統一
post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      config.build_settings['SWIFT_VERSION'] = '5.9'
      config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '17.0'
    end
  end
end
```

## Code Signing の最適化

Code Signing の検証もビルド時間に影響します。

### 開発時の自動署名

```plaintext
Automatically manage signing: Yes
Team: 開発チームを選択
Signing Certificate: Apple Development
```

### Manual Signing（CI/CD 環境）

```plaintext
Automatically manage signing: No
Provisioning Profile: （手動で指定）
Signing Certificate: （手動で指定）
```

## アプリサイズの最適化

バイナリサイズの削減により、ダウンロード時間とインストール容量が改善されることが期待されます。

### Dead Code Stripping

```plaintext
Dead Code Stripping: Yes（Release のみ）
```

使用されていないコードを削除します。

### Strip Debug Symbols

```plaintext
Strip Debug Symbols During Copy: Yes（Release のみ）
Strip Swift Symbols: Yes（Release のみ）
```

デバッグシンボルを削除し、バイナリサイズを削減します。

### Asset Catalog Optimization

```plaintext
Assets.xcassets の最適化:
- App Thinning を有効化
- 画像を適切な解像度で提供（@1x, @2x, @3x）
- 未使用のアセットを削除
```

### xcconfig での設定

```plaintext
// Release.xcconfig
DEAD_CODE_STRIPPING = YES
COPY_PHASE_STRIP = YES
STRIP_INSTALLED_PRODUCT = YES
STRIP_SWIFT_SYMBOLS = YES
ENABLE_BITCODE = NO
```

## リンカーの最適化

### Link-Time Optimization (LTO)

```plaintext
Link-Time Optimization: Incremental (Release)
```

リンク時に追加の最適化を行います。ビルド時間は増加しますが、実行速度が向上することが期待されます。

### Other Linker Flags

```plaintext
Debug:
-Xlinker -no_application_extension

Release:
-Xlinker -no_application_extension
-Xlinker -export_dynamic
```

## ビルドログの分析

Xcode Build Timeline を使用してビルドプロセスを可視化します。

### Build Timeline の表示

```plaintext
1. ビルドを実行
2. Report Navigator (Command + 9) を開く
3. 最新のビルドログを選択
4. Editor > Assistant を開く
```

タイムラインから以下を確認できます：

- 各ターゲットのビルド時間
- 依存関係の解決時間
- コンパイル時間
- リンク時間

## ビルド設定の比較

Debug と Release の設定を比較し、最適化の効果を確認します。

```swift
// Core/Build/BuildConfiguration.swift
struct BuildConfiguration {
    static func printSettings() {
        #if DEBUG
        print("=== Build Settings (Debug) ===")
        print("Optimization: -Onone")
        print("Compilation Mode: incremental")
        print("Testability: Enabled")
        print("Dead Code Stripping: Disabled")
        #else
        print("=== Build Settings (Release) ===")
        print("Optimization: -O")
        print("Compilation Mode: wholemodule")
        print("Testability: Disabled")
        print("Dead Code Stripping: Enabled")
        #endif
        print("Bundle Size: \(bundleSize)")
        print("========================")
    }

    private static var bundleSize: String {
        guard let path = Bundle.main.bundlePath else { return "Unknown" }
        let url = URL(fileURLWithPath: path)

        do {
            let resources = try url.resourceValues(forKeys: [.fileSizeKey])
            if let size = resources.fileSize {
                return ByteCountFormatter.string(fromByteCount: Int64(size), countStyle: .file)
            }
        } catch {
            print("Error calculating bundle size: \(error)")
        }

        return "Unknown"
    }
}
```

## 最適化のチェックリスト

開発効率とアプリパフォーマンスのバランスを取るためのチェックリストです。

### Debug ビルド

- [ ] Optimization Level: -Onone
- [ ] Compilation Mode: incremental
- [ ] Enable Testability: Yes
- [ ] Build Active Architecture Only: Yes
- [ ] Dead Code Stripping: No
- [ ] Debug Information Format: dwarf

### Release ビルド

- [ ] Optimization Level: -O
- [ ] Compilation Mode: wholemodule
- [ ] Enable Testability: No
- [ ] Build Active Architecture Only: No
- [ ] Dead Code Stripping: Yes
- [ ] Strip Debug Symbols: Yes
- [ ] Strip Swift Symbols: Yes
- [ ] Debug Information Format: dwarf-with-dsym

## ビルド時間改善の実践例

大規模プロジェクトにおけるビルド時間短縮の施策例です。

### モジュール化

大きなターゲットを小さなモジュールに分割することで、変更時の再ビルド範囲が限定されます。

```plaintext
プロジェクト構成:
- CoreModule (Framework)
- NetworkingModule (Framework)
- UIComponentsModule (Framework)
- MainApp (App)
```

### Precompiled Headers の活用

頻繁に使用されるヘッダーをプリコンパイルします。

```plaintext
Prefix Header: Yes
Precompile Prefix Header: Yes
```

### Build Phases の最適化

不要な Run Script Phase を削除し、必要なものだけを実行します。

```bash
# SwiftLint を Releaseビルドでは実行しない
if [ "${CONFIGURATION}" = "Debug" ]; then
    "${PODS_ROOT}/SwiftLint/swiftlint"
fi
```

## まとめ

本章では、ビルド設定の最適化手法を解説しました。適切な最適化により、以下の効果が期待されます：

- 開発中のビルド時間短縮
- リリースビルドのパフォーマンス向上
- バイナリサイズの削減
- 開発者の生産性向上

プロジェクトの特性に応じて、ビルド時間とパフォーマンスのバランスを調整しましょう。次章では、チーム開発環境のセットアップについて解説します。
