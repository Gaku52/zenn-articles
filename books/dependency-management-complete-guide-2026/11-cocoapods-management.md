---
title: "CocoaPods管理"
---

# CocoaPods管理

## CocoaPodsとは

CocoaPodsは、iOSプロジェクトで最も広く使用されてきたパッケージマネージャーです。

### CocoaPodsの特徴

- 成熟したエコシステム
- 豊富なライブラリ（9万以上）
- Objective-C/Swift対応

## インストール

```bash
# RubyGemsでインストール
sudo gem install cocoapods

# バージョン確認
pod --version

# リポジトリのセットアップ
pod setup
```

## Podfileの基本

```ruby
# Podfile
platform :ios, '15.0'
use_frameworks!

target 'MyApp' do
  # ネットワーク
  pod 'Alamofire', '~> 5.8'
  
  # データベース
  pod 'Realm', '~> 10.45'
  
  # UI
  pod 'SnapKit', '~> 5.6'
  
  # 開発用
  pod 'SwiftLint', '~> 0.54', :configurations => ['Debug']
  
  target 'MyAppTests' do
    inherit! :search_paths
    pod 'Quick', '~> 7.0'
    pod 'Nimble', '~> 12.0'
  end
end

# Post install hooks
post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '15.0'
    end
  end
end
```

## 基本コマンド

```bash
# 初期化
pod init

# インストール
pod install

# 更新
pod update
pod update Alamofire  # 特定のPodのみ

# 削除
# Podfileから削除後
pod install

# 古いPodを確認
pod outdated

# リポジトリ検索
pod search Alamofire

# 仕様確認
pod spec which Alamofire
```

## Podfile.lock

Podfile.lockは、インストールされた正確なバージョンを記録します。

```yaml
PODS:
  - Alamofire (5.8.1)
  - Realm (10.45.2)

DEPENDENCIES:
  - Alamofire (~> 5.8)
  - Realm (~> 10.45)

SPEC CHECKSUMS:
  Alamofire: 3ca42e259043ee0dc5c0cdd76c4bc568b8e42af7
  Realm: 50ca5da096f0ccb83c176d88c0be0ee5e95b41f3
```

### ロックファイル管理

```bash
# 必ずGitにコミット
git add Podfile.lock
git commit -m "chore: update pods"

# CIでは
pod install --deployment  # Podfile.lockを厳密に遵守
```

## CocoaPodsとSPMの共存

```ruby
# Podfile
platform :ios, '15.0'
use_frameworks!

target 'MyApp' do
  # CocoaPods経由
  pod 'Firebase/Analytics'
  pod 'Firebase/Crashlytics'
  
  # SPMはXcodeで管理
  # - Alamofire
  # - Kingfisher
end
```

## まとめ

CocoaPodsは成熟した選択肢ですが、新規プロジェクトではSPMも検討すべきです。

次章では、Carthageの基礎を解説します。
