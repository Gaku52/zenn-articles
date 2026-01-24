---
title: "データ永続化戦略"
---

# データ永続化戦略

プロジェクトに最適なデータ永続化手法を選択することが重要です。本章では、各手法の特徴と選択基準を解説します。

## データ保存手法の比較

```plaintext
UserDefaults:
✅ 軽量な設定値
✅ 簡単に使える
❌ 機密情報は不可
❌ 大量データは不可

Keychain:
✅ 機密情報（パスワード、トークン）
✅ セキュア
❌ 読み書きがやや遅い
❌ 大量データは不可

File System:
✅ 画像、動画などのバイナリ
✅ 大きなファイル
❌ 検索が困難
❌ リレーションなし

Core Data:
✅ 複雑なデータ構造
✅ リレーション
✅ 高度なクエリ
❌ 学習コストが高い

Realm:
✅ シンプルなAPI
✅ 高速
✅ リアルタイム更新
❌ ファイルサイズが大きい
```

## データ保存戦略の選択

```swift
enum DataPersistenceStrategy {
    case userDefaults
    case keychain
    case fileSystem
    case coreData
    case realm

    static func recommend(for dataType: DataType) -> DataPersistenceStrategy {
        switch dataType {
        case .userSettings:
            return .userDefaults
        case .authToken:
            return .keychain
        case .image, .video:
            return .fileSystem
        case .complexData:
            return .coreData
        case .realtimeData:
            return .realm
        }
    }
}

enum DataType {
    case userSettings
    case authToken
    case image
    case video
    case complexData
    case realtimeData
}
```

## ハイブリッド戦略

```swift
class DataManager {
    func saveUser(_ user: User) async throws {
        // トークンはKeychain
        try KeychainManager.shared.saveToken(user.authToken)

        // ユーザー情報はCore Data
        try await CoreDataManager.shared.saveUser(user)

        // 設定はUserDefaults
        UserDefaults.standard.set(user.preferences, forKey: "userPreferences")

        // プロフィール画像はFile System
        if let imageData = user.profileImage {
            try FileManager.default.saveImage(imageData, filename: "profile_\(user.id).jpg")
        }
    }
}
```

## まとめ

データの特性に応じて最適な保存手法を選択することで、パフォーマンスと保守性が向上します。最終章では、セキュリティチェックリストをまとめます。
