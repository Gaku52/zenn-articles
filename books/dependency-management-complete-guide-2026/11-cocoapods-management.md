---
title: "CocoaPods管理"
---

# CocoaPods管理

## CocoaPodsとは

CocoaPodsは、2011年にリリースされたiOS/macOS向けの依存関係管理ツールです。Ruby製で、9万以上のライブラリが登録されており、iOSエコシステムで広く使用されています。

### CocoaPodsの特徴

#### 成熟したエコシステム

- 9万以上の登録ライブラリ
- 活発なコミュニティサポート
- 豊富なドキュメント

#### 柔軟な設定

- カスタマイズ可能なPodfile
- post_installフックによる細かい制御
- プライベートPodsのサポート

#### Xcode統合

- ワークスペースの自動生成
- ビルド設定の自動管理
- フレームワークの自動リンク

## インストール

### システム要件

- macOS
- Ruby 2.7以降
- Xcode 14以降

### インストール手順

```bash
# RubyGemsでインストール
sudo gem install cocoapods

# バージョン確認
pod --version
# => 1.15.2

# リポジトリのセットアップ（初回のみ）
pod setup
```

### Bundlerでの管理（推奨）

チーム開発では、CocoaPodsのバージョンを固定することが推奨されます。

```ruby
# Gemfile
source 'https://rubygems.org'

gem 'cocoapods', '~> 1.15'
gem 'cocoapods-binary', '~> 0.4'
```

```bash
# Bundlerでインストール
bundle install

# Bundler経由で実行
bundle exec pod install
```

## Podfileの基本

### 基本構成

```ruby
# Podfile
platform :ios, '15.0'
use_frameworks!
inhibit_all_warnings!

target 'MyApp' do
  # ネットワーク
  pod 'Alamofire', '~> 5.8'

  # データベース
  pod 'Realm', '~> 10.45'

  # UI
  pod 'SnapKit', '~> 5.6'
  pod 'Kingfisher', '~> 7.10'

  # リアクティブプログラミング
  pod 'RxSwift', '~> 6.6'
  pod 'RxCocoa', '~> 6.6'

  # ユーティリティ
  pod 'SwiftyJSON', '~> 5.0'

  # テストターゲット
  target 'MyAppTests' do
    inherit! :search_paths
    pod 'Quick', '~> 7.0'
    pod 'Nimble', '~> 12.0'
  end

  # UIテストターゲット
  target 'MyAppUITests' do
    inherit! :search_paths
  end
end
```

### バージョン指定

```ruby
# Podfile
target 'MyApp' do
  # 固定バージョン
  pod 'Alamofire', '5.8.1'

  # 最低バージョン
  pod 'Realm', '>= 10.45.0'

  # 範囲指定
  pod 'SnapKit', '> 5.0', '< 6.0'

  # Optimistic operator（推奨）
  pod 'Kingfisher', '~> 7.10'  # >= 7.10.0, < 8.0.0

  # 最新バージョン（非推奨）
  pod 'SwiftyJSON'

  # Gitリポジトリ
  pod 'MyPod', :git => 'https://github.com/user/MyPod.git'
  pod 'MyPod', :git => 'https://github.com/user/MyPod.git', :branch => 'develop'
  pod 'MyPod', :git => 'https://github.com/user/MyPod.git', :tag => '1.0.0'
  pod 'MyPod', :git => 'https://github.com/user/MyPod.git', :commit => 'abc123'

  # ローカルパス
  pod 'MyLocalPod', :path => '../MyLocalPod'
end
```

### サブスペック

```ruby
# Podfile
target 'MyApp' do
  # Firebaseの特定機能のみ
  pod 'Firebase/Analytics'
  pod 'Firebase/Crashlytics'
  pod 'Firebase/RemoteConfig'

  # RxSwiftとRxCocoa
  pod 'RxSwift'
  pod 'RxCocoa'

  # または統合して指定
  pod 'RxSwift', '~> 6.6', :subspecs => ['RxCocoa']
end
```

## 詳細設定

### 複数ターゲット

```ruby
# Podfile
platform :ios, '15.0'
use_frameworks!

# 共通のPods
def shared_pods
  pod 'Alamofire', '~> 5.8'
  pod 'SwiftyJSON', '~> 5.0'
end

target 'MyApp' do
  shared_pods
  pod 'Firebase/Analytics'

  target 'MyAppTests' do
    inherit! :search_paths
    pod 'Quick'
    pod 'Nimble'
  end
end

target 'MyAppDev' do
  shared_pods
  pod 'FLEX', '~> 5.0'  # デバッグ用
end
```

### 設定の抑制

```ruby
# Podfile
target 'MyApp' do
  # 特定のPodの警告を抑制
  pod 'Realm', '~> 10.45', :inhibit_warnings => true

  # すべてのPodの警告を抑制（ルートに記述）
  inhibit_all_warnings!

  # モジュールマップを使用しない
  pod 'SomePod', :modular_headers => false
end
```

### post_install hooks

```ruby
# Podfile
post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      # 最小デプロイメントターゲットを設定
      config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '15.0'

      # Bitcodeを無効化
      config.build_settings['ENABLE_BITCODE'] = 'NO'

      # デバッグ情報の形式を設定
      config.build_settings['DEBUG_INFORMATION_FORMAT'] = 'dwarf'

      # Code Signingを無効化
      config.build_settings['CODE_SIGNING_ALLOWED'] = 'NO'
    end
  end

  # 特定のターゲットのみ設定変更
  installer.pods_project.targets.each do |target|
    if target.name == 'Alamofire'
      target.build_configurations.each do |config|
        config.build_settings['SWIFT_VERSION'] = '5.9'
      end
    end
  end
end
```

## 基本コマンド

### 初期化

```bash
# Podfileを生成
pod init

# 生成されたPodfileの内容
# Uncomment the next line to define a global platform for your project
# platform :ios, '9.0'
#
# target 'MyApp' do
#   # Comment the next line if you don't want to use dynamic frameworks
#   use_frameworks!
#
#   # Pods for MyApp
#
# end
```

### インストール

```bash
# 初回インストール
pod install

# 詳細ログを表示
pod install --verbose

# リポジトリを更新せずにインストール
pod install --no-repo-update

# クリーンインストール
pod install --clean-install

# 特定のPodのみインストール
pod install --deployment
```

### アップデート

```bash
# すべてのPodを更新
pod update

# 特定のPodのみ更新
pod update Alamofire
pod update Alamofire Realm

# リポジトリのみ更新
pod repo update
```

### 検索・情報

```bash
# Podを検索
pod search Alamofire

# Podの詳細情報
pod spec which Alamofire
pod spec cat Alamofire

# 古いPodを確認
pod outdated

# インストール済みPodの一覧
pod list
```

## Podfile.lock

Podfile.lockは、インストールされた正確なバージョンを記録するファイルです。

### Podfile.lockの内容

```yaml
PODS:
  - Alamofire (5.8.1)
  - Kingfisher (7.10.1)
  - Realm (10.45.2):
    - RealmSwift (= 10.45.2)
  - RealmSwift (10.45.2):
    - Realm (= 10.45.2)
  - RxCocoa (6.6.0):
    - RxRelay (= 6.6.0)
    - RxSwift (= 6.6.0)
  - RxRelay (6.6.0):
    - RxSwift (= 6.6.0)
  - RxSwift (6.6.0)
  - SnapKit (5.6.0)

DEPENDENCIES:
  - Alamofire (~> 5.8)
  - Kingfisher (~> 7.10)
  - Realm (~> 10.45)
  - RxCocoa (~> 6.6)
  - RxSwift (~> 6.6)
  - SnapKit (~> 5.6)

SPEC REPOS:
  trunk:
    - Alamofire
    - Kingfisher
    - Realm
    - RealmSwift
    - RxCocoa
    - RxRelay
    - RxSwift
    - SnapKit

SPEC CHECKSUMS:
  Alamofire: 3ca42e259043ee0dc5c0cdd76c4bc568b8e42af7
  Kingfisher: b9c985d864c43210bcca06a47c6c48e9c9ffe9bc
  Realm: 50ca5da096f0ccb83c176d88c0be0ee5e95b41f3
  RealmSwift: 1d0c8d8e5e5f6c7c8c4e5e9e5e5e5e5e5e5e5e5e
  RxCocoa: f5609b7e3e9f0d4d7e3e3e3e3e3e3e3e3e3e3e3e
  RxRelay: f5609b7e3e9f0d4d7e3e3e3e3e3e3e3e3e3e3e3e
  RxSwift: f5609b7e3e9f0d4d7e3e3e3e3e3e3e3e3e3e3e3e
  SnapKit: e0126861605c87d0e58591c9bb0d10f263d5b3e6

PODFILE CHECKSUM: 9e3e3e3e3e3e3e3e3e3e3e3e3e3e3e3e3e3e3e3e

COCOAPODS: 1.15.2
```

### ロックファイル管理

```bash
# Gitにコミット（必須）
git add Podfile.lock
git commit -m "chore: update pods"

# CIでの使用
pod install --deployment  # Podfile.lockを厳密に遵守
```

## プライベートPods

### プライベートリポジトリの作成

```bash
# プライベートSpec Repoを作成
pod repo add MyPrivateSpecs https://github.com/mycompany/MyPrivateSpecs.git

# Spec Repoの一覧
pod repo list
```

### Podspecの作成

```ruby
# MyLibrary.podspec
Pod::Spec.new do |s|
  s.name             = 'MyLibrary'
  s.version          = '1.0.0'
  s.summary          = 'My private library'
  s.description      = <<-DESC
    This is my private library for internal use.
  DESC

  s.homepage         = 'https://github.com/mycompany/MyLibrary'
  s.license          = { :type => 'MIT', :file => 'LICENSE' }
  s.author           = { 'My Name' => 'myemail@example.com' }
  s.source           = { :git => 'https://github.com/mycompany/MyLibrary.git', :tag => s.version.to_s }

  s.ios.deployment_target = '15.0'
  s.swift_version = '5.9'

  s.source_files = 'MyLibrary/Classes/**/*'
  s.resource_bundles = {
    'MyLibrary' => ['MyLibrary/Assets/*.png']
  }

  s.frameworks = 'UIKit', 'Foundation'
  s.dependency 'Alamofire', '~> 5.8'
end
```

### プライベートPodの公開

```bash
# Specを検証
pod spec lint MyLibrary.podspec

# プライベートリポジトリに公開
pod repo push MyPrivateSpecs MyLibrary.podspec
```

### Podfileでの使用

```ruby
# Podfile
source 'https://github.com/CocoaPods/Specs.git'
source 'https://github.com/mycompany/MyPrivateSpecs.git'

target 'MyApp' do
  pod 'MyLibrary', '~> 1.0'
end
```

## パフォーマンス最適化

### ビルド時間の短縮

```ruby
# Podfile
# フレームワークとしてビルド（推奨）
use_frameworks!

# モジュラーヘッダーを使用
use_modular_headers!

# 並列ビルド数を増やす
install! 'cocoapods',
         :generate_multiple_pod_projects => true,
         :incremental_installation => true
```

### キャッシュの活用

```bash
# Podsディレクトリをキャッシュ（CI）
# .github/workflows/ci.yml
- uses: actions/cache@v3
  with:
    path: Pods
    key: ${{ runner.os }}-pods-${{ hashFiles('**/Podfile.lock') }}
    restore-keys: |
      ${{ runner.os }}-pods-
```

### バイナリキャッシュ

```ruby
# Gemfile
gem 'cocoapods-binary'
```

```ruby
# Podfile
plugin 'cocoapods-binary'

use_frameworks!

target 'MyApp' do
  pod 'Alamofire', '~> 5.8', :binary => true
  pod 'Kingfisher', '~> 7.10', :binary => true
end
```

## CocoaPodsとSPMの共存

### 共存戦略

```ruby
# Podfile
platform :ios, '15.0'
use_frameworks!

target 'MyApp' do
  # CocoaPods経由（SPM未対応）
  pod 'Firebase/Analytics'
  pod 'Firebase/Crashlytics'
  pod 'Realm', '~> 10.45'

  # SPMはXcodeで管理
  # - Alamofire（SPMで追加）
  # - Kingfisher（SPMで追加）
  # - SnapKit（SPMで追加）
end
```

### 移行戦略

段階的にSPMに移行することが推奨されます。

```ruby
# Podfile（移行途中）
target 'MyApp' do
  # まだCocoaPods
  pod 'Firebase/Analytics'
  pod 'Firebase/Crashlytics'
  pod 'Realm', '~> 10.45'

  # 次回移行予定
  # pod 'Alamofire', '~> 5.8'  # → SPMに移行済み
  # pod 'Kingfisher', '~> 7.10'  # → SPMに移行済み
end
```

## トラブルシューティング

### よくある問題

#### Podsディレクトリが見つからない

```bash
# 解決方法
pod install --repo-update
```

#### ビルドエラー: Module not found

```bash
# キャッシュをクリア
rm -rf ~/Library/Caches/CocoaPods
rm -rf Pods
rm -rf ~/Library/Developer/Xcode/DerivedData/*

# 再インストール
pod install
```

#### バージョン競合

```bash
# 依存関係ツリーを確認
pod install --verbose

# 特定のバージョンに固定
pod 'Alamofire', '5.8.1'
```

## CocoaPods管理のベストプラクティス

### チェックリスト

- [ ] Podfile.lockをGitにコミットする
- [ ] BundlerでCocoaPodsのバージョンを固定する
- [ ] バージョンは`~>`で指定する（固定バージョンは避ける）
- [ ] 不要な警告は`inhibit_warnings`で抑制する
- [ ] post_installフックで最小デプロイメントターゲットを統一する
- [ ] CIでは`pod install --deployment`を使用する
- [ ] プライベートPodsは独自のSpec Repoで管理する
- [ ] 定期的に`pod outdated`でアップデートを確認する

### 推奨設定

```ruby
# Podfile
platform :ios, '15.0'
use_frameworks!
inhibit_all_warnings!

target 'MyApp' do
  # 依存関係
  pod 'Alamofire', '~> 5.8'
  pod 'Kingfisher', '~> 7.10'

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

## まとめ

CocoaPodsは成熟した依存関係管理ツールであり、豊富なライブラリエコシステムを持っています。ただし、新規プロジェクトではApple公式のSwift Package Managerも検討すべきです。

主要なポイント:
- **安定性**: Podfile.lockで再現可能なビルド環境を維持
- **柔軟性**: post_installフックで細かい制御が可能
- **エコシステム**: 9万以上の登録ライブラリ
- **移行**: SPMとの共存・段階的移行が可能

次章では、Carthageの基礎を解説します。

## 参考文献

- [CocoaPods Official Guide](https://guides.cocoapods.org/)
- [CocoaPods Podfile Syntax Reference](https://guides.cocoapods.org/syntax/podfile.html)
- [CocoaPods Podspec Syntax Reference](https://guides.cocoapods.org/syntax/podspec.html)
- [CocoaPods Private Pods](https://guides.cocoapods.org/making/private-cocoapods.html)
- [CocoaPods Best Practices](https://guides.cocoapods.org/using/using-cocoapods.html)
