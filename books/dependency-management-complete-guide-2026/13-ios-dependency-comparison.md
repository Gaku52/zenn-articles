---
title: "iOS依存関係管理ツール比較"
---

# iOS依存関係管理ツール比較

## パッケージマネージャー比較表

| 機能 | SPM | CocoaPods | Carthage |
|------|-----|-----------|----------|
| 公式サポート | ✅ Apple公式 | ❌ | ❌ |
| Xcode統合 | ✅ ネイティブ | ⚠️ ワークスペース | ❌ 手動 |
| 学習曲線 | 低 | 中 | 高 |
| ライブラリ数 | 中（増加中） | 多（9万+） | 少 |
| ビルド時間 | 速い | 中 | 遅い |
| バイナリ配布 | ✅ | ✅ | ✅ |
| プライベートパッケージ | ✅ | ✅ | ✅ |

## 選択基準

### Swift Package Manager（SPM）推奨

- 新規プロジェクト
- Swift専用
- Apple公式の最新機能を使いたい

### CocoaPods推奨

- 既存プロジェクト（移行コストが高い）
- Objective-Cライブラリが必要
- Firebase等のSPM未対応ライブラリを使用

### Carthage推奨

- 最小限の依存関係管理
- ビルド済みフレームワークを使用したい

## 移行戦略

### CocoaPods → SPM移行

```ruby
# Podfile（段階的移行）
platform :ios, '15.0'

target 'MyApp' do
  # SPMに移行済み
  # - Alamofire（Xcodeで管理）
  
  # まだCocoaPods
  pod 'Firebase/Analytics'
  pod 'Firebase/Crashlytics'
end
```

## まとめ

新規プロジェクトではSPMが推奨されますが、既存プロジェクトではCocoaPodsの継続使用も合理的です。

次章では、セキュリティ脆弱性スキャンについて解説します。
