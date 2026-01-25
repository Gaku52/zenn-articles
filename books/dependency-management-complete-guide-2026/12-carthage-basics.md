---
title: "Carthage基礎"
---

# Carthage基礎

## Carthageとは

Carthageは、2014年にリリースされたiOS/macOS向けの分散型パッケージマネージャーです。「できるだけシンプルに」という哲学のもと設計されており、CocoaPodsとは異なるアプローチを採用しています。

### Carthageの哲学

#### 最小限の介入

Carthageは、Xcodeプロジェクトファイルを変更しません。ビルド済みフレームワークを提供するのみで、統合は開発者が手動で行います。

#### 分散型

中央集権的なリポジトリを持たず、GitHubなどのGitリポジトリから直接依存関係を取得します。

#### ビルド済みフレームワーク

依存関係をビルドし、バイナリフレームワークとして提供します。これによりビルド時間が短縮されますが、初回ビルドに時間がかかります。

### CocoaPodsとの違い

| 特徴 | Carthage | CocoaPods |
|------|----------|-----------|
| プロジェクト変更 | なし | あり（ワークスペース生成） |
| 中央リポジトリ | 不要 | 必要 |
| ビルド方式 | ビルド済みフレームワーク | ソースから毎回ビルド |
| 統合方法 | 手動 | 自動 |
| 学習曲線 | 高い | 低い |

## インストール

### Homebrewでのインストール

```bash
# Homebrewでインストール
brew install carthage

# バージョン確認
carthage version
# => 0.39.1
```

### 手動インストール

```bash
# GitHubからダウンロード
curl -OL https://github.com/Carthage/Carthage/releases/download/0.39.1/Carthage.pkg
sudo installer -pkg Carthage.pkg -target /

# インストール確認
which carthage
# => /usr/local/bin/carthage
```

## Cartfileの作成

### 基本構成

```
# Cartfile

# GitHubリポジトリ（推奨）
github "Alamofire/Alamofire" ~> 5.8
github "onevcat/Kingfisher" ~> 7.10
github "SnapKit/SnapKit" ~> 5.6

# バージョン指定のバリエーション
github "ReactiveCocoa/ReactiveCocoa" >= 11.0
github "Realm/realm-swift" == 10.45.2

# Gitリポジトリ
git "https://enterprise.local/repo/MyFramework.git" ~> 1.0

# ブランチ指定
git "https://github.com/user/MyFramework.git" "develop"

# コミットハッシュ指定
git "https://github.com/user/MyFramework.git" "abc123def456"

# バイナリのみ（XCFramework）
binary "https://my.domain.com/release/MyFramework.json" ~> 1.0
```

### バージョン指定

```
# Cartfile

# 固定バージョン
github "Alamofire/Alamofire" == 5.8.1

# 最低バージョン
github "Kingfisher" >= 7.10.0

# Optimistic operator（推奨）
github "SnapKit/SnapKit" ~> 5.6  # >= 5.6.0, < 6.0.0

# 最新バージョン（非推奨）
github "Alamofire/Alamofire"
```

### プライベートリポジトリ

```
# Cartfile

# SSH認証
git "git@github.com:mycompany/MyPrivateFramework.git" ~> 1.0

# HTTPS認証（GitHub Personal Access Tokenを使用）
git "https://github.com/mycompany/MyPrivateFramework.git" ~> 1.0
```

認証設定（~/.netrc）:

```
machine github.com
  login YOUR_GITHUB_USERNAME
  password YOUR_PERSONAL_ACCESS_TOKEN
```

## 基本コマンド

### 依存関係の解決とビルド

```bash
# すべての依存関係をビルド
carthage update

# 特定のフレームワークのみ
carthage update Alamofire
carthage update Alamofire Kingfisher

# プラットフォーム指定
carthage update --platform iOS
carthage update --platform iOS,macOS

# ビルドせずに依存関係のみ解決
carthage bootstrap

# 既存のCartfile.resolvedを使用してビルド
carthage bootstrap --platform iOS

# 詳細ログを表示
carthage update --verbose

# キャッシュを使用
carthage update --cache-builds

# 並列ビルド
carthage update --use-xcframeworks
```

### ビルドのみ

```bash
# すべてをビルド
carthage build

# 特定のフレームワークのみ
carthage build --platform iOS Alamofire

# XCFrameworkとしてビルド（推奨）
carthage build --use-xcframeworks
```

## Xcodeプロジェクトへの統合

### フレームワークの追加

#### 1. フレームワークのリンク

Xcodeプロジェクトで以下の手順を実行:

1. プロジェクトナビゲーターでプロジェクトファイルを選択
2. Targetを選択
3. "General" タブを開く
4. "Frameworks, Libraries, and Embedded Content" セクションに移動
5. "+" ボタンをクリック
6. "Add Other..." → "Add Files..." を選択
7. `Carthage/Build/iOS/`配下のフレームワークを選択
8. "Embed & Sign" を選択

#### 2. Run Script Phaseの追加

1. Targetの "Build Phases" タブを開く
2. "+" → "New Run Script Phase" をクリック
3. スクリプトを追加:

```bash
/usr/local/bin/carthage copy-frameworks
```

または、XCFrameworkを使用する場合:

```bash
# XCFrameworkの場合は不要（Embed & Signで自動処理）
```

#### 3. Input Filesの追加

Run Script Phaseの "Input Files" に以下を追加:

```
$(SRCROOT)/Carthage/Build/iOS/Alamofire.framework
$(SRCROOT)/Carthage/Build/iOS/Kingfisher.framework
$(SRCROOT)/Carthage/Build/iOS/SnapKit.framework
```

XCFrameworkの場合:

```
$(SRCROOT)/Carthage/Build/Alamofire.xcframework
$(SRCROOT)/Carthage/Build/Kingfisher.xcframework
$(SRCROOT)/Carthage/Build/SnapKit.xcframework
```

#### 4. Output Filesの追加

```
$(BUILT_PRODUCTS_DIR)/$(FRAMEWORKS_FOLDER_PATH)/Alamofire.framework
$(BUILT_PRODUCTS_DIR)/$(FRAMEWORKS_FOLDER_PATH)/Kingfisher.framework
$(BUILT_PRODUCTS_DIR)/$(FRAMEWORKS_FOLDER_PATH)/SnapKit.framework
```

### XCFrameworkの使用（推奨）

XCFrameworkは、複数のプラットフォーム・アーキテクチャをサポートする新しいフレームワーク形式です。

```bash
# XCFrameworkとしてビルド
carthage update --use-xcframeworks

# 生成されたXCFramework
# Carthage/Build/
#   ├── Alamofire.xcframework
#   ├── Kingfisher.xcframework
#   └── SnapKit.xcframework
```

XCFrameworkの利点:
- App Store提出時にシンボルストリップが不要
- 複数アーキテクチャの統合
- シンプルな統合プロセス

## Cartfile.resolved

Cartfile.resolvedは、ビルドされた正確なバージョンを記録します。

```
github "Alamofire/Alamofire" "5.8.1"
github "onevcat/Kingfisher" "7.10.1"
github "SnapKit/SnapKit" "5.6.0"
github "ReactiveCocoa/ReactiveCocoa" "11.2.2"
```

### ロックファイル管理

```bash
# Gitにコミット（必須）
git add Cartfile.resolved
git commit -m "chore: update dependencies"

# CIでの使用
carthage bootstrap --platform iOS --cache-builds
```

## CI/CD統合

### GitHub Actions

```yaml
# .github/workflows/ios.yml
name: iOS CI

on: [push, pull_request]

jobs:
  build:
    runs-on: macos-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install Carthage
        run: brew install carthage

      - name: Cache Carthage
        uses: actions/cache@v3
        with:
          path: Carthage
          key: ${{ runner.os }}-carthage-${{ hashFiles('**/Cartfile.resolved') }}
          restore-keys: |
            ${{ runner.os }}-carthage-

      - name: Build dependencies
        run: carthage bootstrap --platform iOS --cache-builds --use-xcframeworks

      - name: Build project
        run: xcodebuild -scheme MyApp -sdk iphonesimulator -destination 'platform=iOS Simulator,name=iPhone 15' build test
```

### Fastlane統合

```ruby
# Fastfile
lane :build do
  # Carthage依存関係のビルド
  carthage(
    command: "bootstrap",
    platform: "iOS",
    cache_builds: true,
    use_xcframeworks: true
  )

  # プロジェクトのビルド
  build_app(
    scheme: "MyApp",
    export_method: "app-store"
  )
end
```

## プライベートフレームワークの作成

### フレームワークプロジェクトの作成

1. Xcodeで新規プロジェクト作成
2. "Framework" テンプレートを選択
3. プロジェクト名を入力（例: MyFramework）

### Carthage対応

#### 1. Shared Schemeの作成

1. "Product" → "Scheme" → "Manage Schemes..."
2. フレームワークのSchemeを選択
3. "Shared" チェックボックスをオン

#### 2. .gitignoreの設定

```gitignore
# .gitignore
Carthage/
.build/
*.xcodeproj/xcuserdata/
*.xcworkspace/xcuserdata/
```

#### 3. バージョンタグの作成

```bash
git tag -a 1.0.0 -m "Release 1.0.0"
git push origin 1.0.0
```

### 他プロジェクトでの使用

```
# Cartfile
github "mycompany/MyFramework" ~> 1.0
```

## パフォーマンス最適化

### キャッシュの活用

```bash
# ビルドキャッシュを使用
carthage update --cache-builds

# キャッシュ場所
~/Library/Caches/org.carthage.CarthageKit/
```

### 並列ビルド

```bash
# デフォルトで並列ビルド
carthage update

# 並列数を指定（環境変数）
export CARTHAGE_BUILD_PARALLELISM=4
carthage update
```

### ビルド済みバイナリの使用

```
# Cartfile
binary "https://my-cdn.com/releases/MyFramework.json" ~> 1.0
```

バイナリJSON仕様:

```json
{
  "1.0.0": "https://my-cdn.com/releases/MyFramework-1.0.0.zip"
}
```

## トラブルシューティング

### よくある問題

#### ビルドエラー: Module not found

```bash
# 解決方法1: キャッシュをクリア
rm -rf ~/Library/Caches/org.carthage.CarthageKit
rm -rf Carthage

# 解決方法2: クリーンビルド
carthage update --platform iOS --no-use-binaries
```

#### Code Signingエラー

```bash
# 解決方法: CODE_SIGNING_ALLOWEDを無効化
carthage update --platform iOS --no-use-binaries

# または、Xcodeプロジェクトで設定
# Build Settings → CODE_SIGNING_ALLOWED = NO
```

#### XCFramework変換エラー

```bash
# 解決方法: 最新版にアップデート
brew upgrade carthage

# または、--use-xcframeworksなしでビルド
carthage update --platform iOS
```

## Carthage管理のベストプラクティス

### チェックリスト

- [ ] Cartfile.resolvedをGitにコミットする
- [ ] XCFrameworkを使用する（`--use-xcframeworks`）
- [ ] CIでキャッシュを活用する
- [ ] プライベートフレームワークはShared Schemeを有効にする
- [ ] バージョンは`~>`で指定する（固定バージョンは避ける）
- [ ] ビルド済みフレームワークはGitにコミットしない（.gitignoreに追加）
- [ ] 定期的に依存関係を更新する
- [ ] Run Script Phaseを正しく設定する

### 推奨設定

```
# Cartfile
github "Alamofire/Alamofire" ~> 5.8
github "onevcat/Kingfisher" ~> 7.10
github "SnapKit/SnapKit" ~> 5.6

# .gitignore
Carthage/Build/
Carthage/Checkouts/
```

```bash
# ビルドコマンド
carthage update --platform iOS --use-xcframeworks --cache-builds
```

## CarthageからSPMへの移行

### 移行手順

1. **依存関係の確認**

```bash
# Cartfileから依存関係を確認
cat Cartfile
```

2. **SPMで追加**

Xcodeで:
1. File → Add Package Dependencies...
2. GitHubリポジトリURLを入力
3. バージョンを選択
4. Add Package

3. **Carthageを削除**

```bash
# Cartfileを削除
rm Cartfile Cartfile.resolved

# Carthageディレクトリを削除
rm -rf Carthage

# Xcodeプロジェクトからフレームワークを削除
# - General → Frameworks, Libraries, and Embedded Content
# - Build Phases → Run Script（carthage copy-frameworks）
```

## まとめ

Carthageは、シンプルで分散型のパッケージマネージャーです。Xcodeプロジェクトを変更しないため、細かい制御が可能ですが、手動統合が必要です。

主要なポイント:
- **シンプル**: Xcodeプロジェクトを変更しない
- **分散型**: 中央リポジトリ不要
- **ビルド済みフレームワーク**: 初回ビルドは遅いが、以降は高速
- **手動統合**: Run Script Phaseの設定が必要

新規プロジェクトでは、Apple公式のSwift Package Managerの使用が推奨されます。

次章では、iOS向けパッケージマネージャーの比較を行います。

## 参考文献

- [Carthage Official Documentation](https://github.com/Carthage/Carthage)
- [Carthage README](https://github.com/Carthage/Carthage/blob/master/README.md)
- [Adding Frameworks to an App](https://github.com/Carthage/Carthage#adding-frameworks-to-an-application)
- [Building platform-independent XCFrameworks](https://github.com/Carthage/Carthage#building-platform-independent-xcframeworks)
- [Carthage Migration Guide](https://github.com/Carthage/Carthage/blob/master/Documentation/Xcode12Workaround.md)
