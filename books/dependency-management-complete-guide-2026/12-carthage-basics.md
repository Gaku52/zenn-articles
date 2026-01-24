---
title: "Carthage基礎"
---

# Carthage基礎

## Carthageとは

Carthageは、シンプルで分散型のiOS/macOS向けパッケージマネージャーです。

### Carthageの哲学

- 最小限の介入
- ビルド済みフレームワークの使用
- Xcodeプロジェクトを変更しない

## インストール

```bash
# Homebrewでインストール
brew install carthage

# バージョン確認
carthage version
```

## Cartfileの作成

```
# Cartfile

# GitHubリポジトリ
github "Alamofire/Alamofire" ~> 5.8
github "ReactiveCocoa/ReactiveCocoa" >= 11.0

# Gitリポジトリ
git "https://enterprise.local/repo.git" >= 1.0

# バイナリ
binary "https://my.domain.com/release/MyFramework.json"
```

## 基本コマンド

```bash
# 依存関係を解決してビルド
carthage update

# 特定のフレームワークのみ
carthage update Alamofire

# プラットフォーム指定
carthage update --platform iOS

# ビルドせずに依存関係のみ解決
carthage bootstrap

# ビルドのみ（依存関係解決なし）
carthage build
```

## Xcodeプロジェクトへの追加

1. Carthage/Build/iOS/*.frameworkをプロジェクトにドラッグ
2. Build Phases → New Run Script Phase
3. スクリプトを追加:

```bash
/usr/local/bin/carthage copy-frameworks
```

4. Input Filesに追加:

```
$(SRCROOT)/Carthage/Build/iOS/Alamofire.framework
```

## Cartfile.resolved

```
github "Alamofire/Alamofire" "5.8.1"
github "ReactiveCocoa/ReactiveCocoa" "11.2.2"
```

### ロックファイル管理

```bash
# Gitにコミット
git add Cartfile.resolved
git commit -m "chore: update dependencies"
```

## まとめ

Carthageはシンプルですが、SPMやCocoaPodsに比べて手動作業が多くなります。

次章では、iOS向けパッケージマネージャーの比較を行います。
