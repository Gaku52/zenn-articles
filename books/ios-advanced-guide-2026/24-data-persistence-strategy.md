---
title: "データ永続化戦略"
---

# データ永続化戦略

プロジェクトに最適なデータ永続化手法を選択することは、アプリケーションのパフォーマンス、セキュリティ、保守性に大きく影響します。本章では、各手法の特徴と選択基準、実装パターンを詳しく解説します。

## データ保存手法の比較

iOSアプリケーションで利用可能な主要なデータ保存手法とその特性を理解することが重要です。

### UserDefaults

軽量な設定値の保存に最適です。

**適している用途**:
- アプリケーション設定（テーマ、言語、通知設定など）
- ユーザー設定（フォントサイズ、表示オプションなど）
- 簡単なフラグやカウンター

**利点**:
- APIがシンプルで使いやすい
- 読み書きが高速（メモリキャッシュ）
- Property Wrapperで型安全にアクセス可能

**制約**:
- 機密情報の保存は不可（暗号化されていない）
- 大量データの保存は不適切（起動時間に影響）
- データサイズの上限は推奨で1MB程度

```swift
// 使用例
UserDefaults.standard.set(true, forKey: "darkModeEnabled")
UserDefaults.standard.set("ja", forKey: "appLanguage")

let darkMode = UserDefaults.standard.bool(forKey: "darkModeEnabled")
let language = UserDefaults.standard.string(forKey: "appLanguage")
```

### Keychain

機密情報の安全な保存に最適です。

**適している用途**:
- パスワード
- 認証トークン（OAuth Token、JWT）
- APIキー
- 暗号化キー
- クレジットカード情報

**利点**:
- ハードウェアレベルで暗号化される（Secure Enclave）
- アプリ削除後もデータが残る（オプション）
- iCloudで同期可能（オプション）
- 脱獄デバイスでも保護される

**制約**:
- APIが複雑（C言語ベース）
- 読み書きがUserDefaultsより遅い
- 大量データの保存は不適切

```swift
// 使用例（簡略化版）
let token = "secret_auth_token"
let tokenData = token.data(using: .utf8)!

// 保存
let query: [String: Any] = [
    kSecClass as String: kSecClassGenericPassword,
    kSecAttrAccount as String: "authToken",
    kSecValueData as String: tokenData
]
SecItemAdd(query as CFDictionary, nil)

// 取得
let searchQuery: [String: Any] = [
    kSecClass as String: kSecClassGenericPassword,
    kSecAttrAccount as String: "authToken",
    kSecReturnData as String: true
]
var result: AnyObject?
SecItemCopyMatching(searchQuery as CFDictionary, &result)
```

### File System

バイナリデータやドキュメントの保存に最適です。

**適している用途**:
- 画像、動画、音声ファイル
- PDFやOfficeドキュメント
- ダウンロードしたファイル
- キャッシュデータ

**利点**:
- 大きなファイルも保存可能
- ファイル単位での管理が容易
- FileManager APIで柔軟な操作が可能

**制約**:
- 構造化されたデータの検索が困難
- リレーションシップの管理が難しい
- データの整合性は自分で管理する必要がある

```swift
// 使用例
let fileManager = FileManager.default
let documentsURL = fileManager.urls(for: .documentDirectory, in: .userDomainMask)[0]
let imageURL = documentsURL.appendingPathComponent("profile.jpg")

// 保存
if let imageData = image.jpegData(compressionQuality: 0.8) {
    try? imageData.write(to: imageURL)
}

// 読み込み
if let imageData = try? Data(contentsOf: imageURL),
   let image = UIImage(data: imageData) {
    // 画像を使用
}
```

### Core Data

複雑なデータ構造の保存に最適です。

**適している用途**:
- エンティティ間にリレーションシップがあるデータ
- 大量のデータ（数千〜数百万レコード）
- 高度なクエリが必要なデータ
- オフラインファースト設計のアプリ

**利点**:
- リレーションシップの管理が容易
- 高度なクエリ（NSPredicate、NSSortDescriptor）
- マイグレーション機能が充実
- CloudKitとの統合が容易

**制約**:
- 学習コストが高い
- 設定が複雑
- スレッド管理に注意が必要

```swift
// 使用例
// Entity定義（Core Data Model Editorで作成）
// User: name (String), email (String), age (Int16), createdAt (Date)

// 保存
let context = persistentContainer.viewContext
let user = User(context: context)
user.name = "John Doe"
user.email = "john@example.com"
user.age = 30
user.createdAt = Date()

try? context.save()

// 取得
let fetchRequest: NSFetchRequest<User> = User.fetchRequest()
fetchRequest.predicate = NSPredicate(format: "age >= %d", 18)
fetchRequest.sortDescriptors = [NSSortDescriptor(key: "name", ascending: true)]

let users = try? context.fetch(fetchRequest)
```

### Realm

リアルタイム性が求められるデータに最適です。

**適している用途**:
- リアルタイム更新が必要なデータ
- モバイルとWebでデータを同期する場合（Realm Sync）
- シンプルなAPIで構造化データを扱いたい場合

**利点**:
- APIがシンプルで直感的
- 高速な読み書き性能
- リアルタイム通知機能
- クロスプラットフォーム対応

**制約**:
- ファイルサイズが大きめ
- iOSプラットフォーム専用ではない（学習リソースがCore Dataより少ない）
- 複雑なマイグレーションはCore Dataに劣る

```swift
// 使用例
// Model定義
class User: Object {
    @Persisted(primaryKey: true) var id: Int
    @Persisted var name: String
    @Persisted var email: String
    @Persisted var age: Int
    @Persisted var createdAt: Date
}

// 保存
let realm = try! Realm()
let user = User()
user.id = 1
user.name = "John Doe"
user.email = "john@example.com"
user.age = 30
user.createdAt = Date()

try! realm.write {
    realm.add(user)
}

// 取得
let users = realm.objects(User.self).filter("age >= 18").sorted(byKeyPath: "name")
```

## データ保存戦略の選択フローチャート

データの特性に基づいて、最適なストレージを選択するための判断基準：

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
        case .authToken, .password, .apiKey:
            return .keychain
        case .image, .video, .audio, .document:
            return .fileSystem
        case .complexStructuredData:
            return .coreData
        case .realtimeData:
            return .realm
        }
    }

    static func recommendBySize(bytes: Int) -> DataPersistenceStrategy {
        switch bytes {
        case 0..<1024: // < 1KB
            return .userDefaults
        case 1024..<10_485_760: // 1KB - 10MB
            return .fileSystem
        default: // > 10MB
            return .fileSystem
        }
    }
}

enum DataType {
    case userSettings
    case authToken
    case password
    case apiKey
    case image
    case video
    case audio
    case document
    case complexStructuredData
    case realtimeData
}
```

## オフラインファースト戦略

ネットワーク接続が不安定な環境でも動作するアプリケーションを構築するためのパターンです。

### Repository with Cache

```swift
protocol UserRepository {
    func fetchUser(id: Int, forceRefresh: Bool) async throws -> User
}

class UserRepositoryImpl: UserRepository {
    private let apiClient: APIClient
    private let coreDataManager: CoreDataManager
    private let cachePolicy: CachePolicy

    init(
        apiClient: APIClient,
        coreDataManager: CoreDataManager,
        cachePolicy: CachePolicy = .cacheFirst
    ) {
        self.apiClient = apiClient
        self.coreDataManager = coreDataManager
        self.cachePolicy = cachePolicy
    }

    func fetchUser(id: Int, forceRefresh: Bool = false) async throws -> User {
        // forceRefreshがtrueの場合は必ずAPIから取得
        if forceRefresh {
            return try await fetchFromAPIAndCache(id: id)
        }

        // キャッシュポリシーに応じた処理
        switch cachePolicy {
        case .cacheFirst:
            // キャッシュ優先
            if let cachedUser = try? coreDataManager.fetchUser(id: id) {
                return cachedUser
            }
            return try await fetchFromAPIAndCache(id: id)

        case .networkFirst:
            // ネットワーク優先
            do {
                return try await fetchFromAPIAndCache(id: id)
            } catch {
                // ネットワークエラーの場合はキャッシュにフォールバック
                if let cachedUser = try? coreDataManager.fetchUser(id: id) {
                    return cachedUser
                }
                throw error
            }

        case .cacheOnly:
            // キャッシュのみ
            return try coreDataManager.fetchUser(id: id)

        case .networkOnly:
            // ネットワークのみ
            return try await fetchFromAPIAndCache(id: id)
        }
    }

    private func fetchFromAPIAndCache(id: Int) async throws -> User {
        let user = try await apiClient.fetchUser(id: id)
        try? coreDataManager.saveUser(user)
        return user
    }
}

enum CachePolicy {
    case cacheFirst   // キャッシュ優先、なければネットワーク
    case networkFirst // ネットワーク優先、失敗すればキャッシュ
    case cacheOnly    // キャッシュのみ
    case networkOnly  // ネットワークのみ
}
```

## キャッシュ戦略

効率的なキャッシュ管理により、パフォーマンスとデータの鮮度をバランスします。

### Time-To-Live (TTL) キャッシュ

```swift
struct CachedData<T: Codable>: Codable {
    let data: T
    let cachedAt: Date
    let ttl: TimeInterval

    var isExpired: Bool {
        Date().timeIntervalSince(cachedAt) > ttl
    }
}

class TTLCache<T: Codable> {
    private let key: String
    private let ttl: TimeInterval

    init(key: String, ttl: TimeInterval = 3600) { // デフォルト1時間
        self.key = key
        self.ttl = ttl
    }

    func save(_ data: T) {
        let cachedData = CachedData(data: data, cachedAt: Date(), ttl: ttl)
        UserDefaults.standard.setCodable(cachedData, forKey: key)
    }

    func load() -> T? {
        guard let cachedData: CachedData<T> = UserDefaults.standard.codable(forKey: key),
              !cachedData.isExpired else {
            return nil
        }
        return cachedData.data
    }

    func invalidate() {
        UserDefaults.standard.removeObject(forKey: key)
    }
}

// 使用例
let userCache = TTLCache<User>(key: "user_cache", ttl: 1800) // 30分

// 保存
userCache.save(user)

// 取得（有効期限内であれば返す、切れていればnil）
if let cachedUser = userCache.load() {
    print("Cached user: \(cachedUser.name)")
}
```

### LRU (Least Recently Used) キャッシュ

```swift
class LRUCache<Key: Hashable, Value> {
    private var cache: [Key: (value: Value, timestamp: Date)] = [:]
    private let maxSize: Int
    private let queue = DispatchQueue(label: "com.example.lrucache", attributes: .concurrent)

    init(maxSize: Int = 100) {
        self.maxSize = maxSize
    }

    func get(_ key: Key) -> Value? {
        queue.sync {
            guard let cached = cache[key] else { return nil }
            // アクセス時刻を更新
            cache[key] = (cached.value, Date())
            return cached.value
        }
    }

    func set(_ key: Key, value: Value) {
        queue.async(flags: .barrier) {
            // キャッシュサイズ超過時は最も古いものを削除
            if self.cache.count >= self.maxSize, self.cache[key] == nil {
                self.evictOldest()
            }

            self.cache[key] = (value, Date())
        }
    }

    private func evictOldest() {
        guard let oldestKey = cache.min(by: { $0.value.timestamp < $1.value.timestamp })?.key else {
            return
        }
        cache.removeValue(forKey: oldestKey)
    }

    func clear() {
        queue.async(flags: .barrier) {
            self.cache.removeAll()
        }
    }
}

// 使用例
let imageCache = LRUCache<String, UIImage>(maxSize: 50)

// 保存
imageCache.set("profile_1", value: profileImage)

// 取得
if let cachedImage = imageCache.get("profile_1") {
    imageView.image = cachedImage
}
```

## データ同期パターン

ローカルとリモートのデータを同期する際のパターンです。

### Conflict Resolution Strategy

```swift
enum SyncStrategy {
    case serverWins      // サーバー優先
    case clientWins      // クライアント優先
    case lastWriteWins   // 最後の書き込みが優先
    case merge           // マージ（カスタムロジック）
}

protocol SyncManager {
    func sync<T: Syncable>(entity: T, strategy: SyncStrategy) async throws -> T
}

protocol Syncable: Codable {
    var id: String { get }
    var updatedAt: Date { get }
    var version: Int { get }
}

class DataSyncManager: SyncManager {
    private let apiClient: APIClient
    private let localDatabase: CoreDataManager

    init(apiClient: APIClient, localDatabase: CoreDataManager) {
        self.apiClient = apiClient
        self.localDatabase = localDatabase
    }

    func sync<T: Syncable>(entity: T, strategy: SyncStrategy) async throws -> T {
        // ローカルのデータを取得
        let localEntity = try? localDatabase.fetch(T.self, id: entity.id)

        // サーバーのデータを取得
        let serverEntity = try await apiClient.fetch(T.self, id: entity.id)

        // 競合解決
        let resolvedEntity = resolveConflict(
            local: localEntity,
            server: serverEntity,
            strategy: strategy
        )

        // ローカルに保存
        try localDatabase.save(resolvedEntity)

        // サーバーに送信
        try await apiClient.update(resolvedEntity)

        return resolvedEntity
    }

    private func resolveConflict<T: Syncable>(
        local: T?,
        server: T?,
        strategy: SyncStrategy
    ) -> T {
        guard let local = local, let server = server else {
            return local ?? server!
        }

        switch strategy {
        case .serverWins:
            return server

        case .clientWins:
            return local

        case .lastWriteWins:
            return local.updatedAt > server.updatedAt ? local : server

        case .merge:
            // カスタムマージロジック（ここでは単純化）
            return local.version > server.version ? local : server
        }
    }
}
```

## ハイブリッド戦略の実装

複数のストレージを組み合わせた実装例です。

```swift
class UserDataManager {
    private let keychainManager: KeychainManager
    private let coreDataManager: CoreDataManager
    private let fileManager: FileManager

    init(
        keychainManager: KeychainManager = .shared,
        coreDataManager: CoreDataManager = .shared,
        fileManager: FileManager = .default
    ) {
        self.keychainManager = keychainManager
        self.coreDataManager = coreDataManager
        self.fileManager = fileManager
    }

    func saveUser(_ user: User) async throws {
        // 1. 認証トークンはKeychain（最も安全）
        if let authToken = user.authToken {
            try keychainManager.saveToken(authToken, forKey: "authToken")
        }

        // 2. ユーザー情報はCore Data（構造化データ）
        try await coreDataManager.saveUser(user)

        // 3. ユーザー設定はUserDefaults（軽量データ）
        UserDefaults.standard.set(user.preferences.theme, forKey: "theme")
        UserDefaults.standard.set(user.preferences.language, forKey: "language")

        // 4. プロフィール画像はFile System（バイナリデータ）
        if let imageData = user.profileImageData {
            let imagePath = getDocumentsDirectory().appendingPathComponent("profile_\(user.id).jpg")
            try imageData.write(to: imagePath)
        }
    }

    func loadUser(id: String) async throws -> User {
        // Core Dataからユーザー情報を取得
        guard var user = try await coreDataManager.fetchUser(id: id) else {
            throw DataError.userNotFound
        }

        // Keychainから認証トークンを取得
        if let token = try? keychainManager.loadToken(forKey: "authToken") {
            user.authToken = token
        }

        // UserDefaultsからユーザー設定を取得
        let theme = UserDefaults.standard.string(forKey: "theme") ?? "light"
        let language = UserDefaults.standard.string(forKey: "language") ?? "en"
        user.preferences = UserPreferences(theme: theme, language: language)

        // File Systemからプロフィール画像を取得
        let imagePath = getDocumentsDirectory().appendingPathComponent("profile_\(user.id).jpg")
        if let imageData = try? Data(contentsOf: imagePath) {
            user.profileImageData = imageData
        }

        return user
    }

    func deleteUser(id: String) async throws {
        // すべてのストレージから削除
        try keychainManager.deleteToken(forKey: "authToken")
        try await coreDataManager.deleteUser(id: id)
        UserDefaults.standard.removeObject(forKey: "theme")
        UserDefaults.standard.removeObject(forKey: "language")

        let imagePath = getDocumentsDirectory().appendingPathComponent("profile_\(id).jpg")
        try? fileManager.removeItem(at: imagePath)
    }

    private func getDocumentsDirectory() -> URL {
        fileManager.urls(for: .documentDirectory, in: .userDomainMask)[0]
    }
}

enum DataError: Error {
    case userNotFound
    case saveFailed
    case deleteFailed
}
```

## チェックリスト

データ永続化戦略のベストプラクティス：

- [ ] **適切なストレージ選択**: データの特性に応じて最適なストレージを選んでいる
- [ ] **セキュリティ**: 機密情報はKeychainに保存している
- [ ] **パフォーマンス**: 大量データはCore DataまたはRealmで管理している
- [ ] **オフライン対応**: ネットワーク不通時でも動作するキャッシュ機構がある
- [ ] **キャッシュ戦略**: TTLまたはLRUキャッシュを適切に実装している
- [ ] **データ同期**: ローカルとリモートの競合解決ロジックがある
- [ ] **マイグレーション**: データモデル変更時のマイグレーション計画がある
- [ ] **テスタビリティ**: データレイヤーがモック可能な設計になっている

## まとめ

データの特性に応じて最適な保存手法を選択することで、パフォーマンス、セキュリティ、保守性が向上します。UserDefaultsは軽量な設定、Keychainは機密情報、File Systemはバイナリデータ、Core DataやRealmは構造化された複雑なデータに適しています。

オフラインファースト戦略とキャッシュ戦略を組み合わせることで、ネットワーク状況に左右されない快適なユーザー体験を提供できます。

最終章では、セキュリティチェックリストをまとめます。

## 参考文献

- [Apple Developer Documentation - UserDefaults](https://developer.apple.com/documentation/foundation/userdefaults)
- [Apple Developer Documentation - Keychain Services](https://developer.apple.com/documentation/security/keychain_services)
- [Apple Developer Documentation - File System](https://developer.apple.com/documentation/foundation/file_system)
- [Apple Developer Documentation - Core Data](https://developer.apple.com/documentation/coredata)
- [Realm Documentation](https://docs.mongodb.com/realm/sdk/swift/)
- [Offline-First Apps with Swift](https://www.raywenderlich.com/books/advanced-ios-app-architecture)
