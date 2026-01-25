---
title: "Realmデータベース"
---

# Realmデータベース

Realmは、モバイル向けに最適化された高速なデータベースです。Core Dataと比較して、よりシンプルなAPIと高いパフォーマンスを提供します。本章では、Realmの基本的な使い方から高度な機能まで解説します。

## Realmの特徴

Realmは以下の特徴を持つオブジェクト指向データベースです：

- シンプルで直感的なAPI
- 高速なクエリパフォーマンス
- リアクティブなデータ通知
- マルチスレッド対応
- 暗号化サポート
- クロスプラットフォーム対応

## Realmの導入

### Swift Package Managerによるインストール

Xcodeプロジェクトにて、File > Add Packages...から以下のURLを指定してRealmを追加できます。

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/realm/realm-swift.git", from: "10.40.0")
]
```

### 基本設定

```swift
import RealmSwift

class RealmConfiguration {
    static let shared = RealmConfiguration()

    private init() {}

    // デフォルト設定
    func setupDefaultConfiguration() {
        var config = Realm.Configuration.defaultConfiguration

        // スキーマバージョンの設定
        config.schemaVersion = 1

        // マイグレーション処理
        config.migrationBlock = { migration, oldSchemaVersion in
            if oldSchemaVersion < 1 {
                // マイグレーション処理
            }
        }

        // 削除ポリシー（オプション）
        config.deleteRealmIfMigrationNeeded = false

        Realm.Configuration.defaultConfiguration = config
    }

    // カスタム設定（暗号化付き）
    func setupEncryptedConfiguration() throws -> Realm.Configuration {
        guard let key = getEncryptionKey() else {
            throw RealmError.encryptionKeyNotFound
        }

        var config = Realm.Configuration.defaultConfiguration
        config.encryptionKey = key
        config.schemaVersion = 1

        return config
    }

    private func getEncryptionKey() -> Data? {
        // Keychainから暗号化キーを取得
        // 実装は省略
        return Data(count: 64)
    }
}

enum RealmError: Error {
    case encryptionKeyNotFound
}
```

## モデル定義

Realmのモデルは`Object`クラスを継承して定義します。

```swift
import RealmSwift
import Foundation

class User: Object {
    @Persisted(primaryKey: true) var id: UUID = UUID()
    @Persisted var name: String = ""
    @Persisted var email: String = ""
    @Persisted var age: Int = 0
    @Persisted var isActive: Bool = true
    @Persisted var createdAt: Date = Date()
    @Persisted var updatedAt: Date = Date()

    // リレーションシップ（One-to-Many）
    @Persisted var posts: List<Post> = List<Post>()

    // 計算プロパティ（永続化されない）
    var displayName: String {
        return name.isEmpty ? "Unknown" : name
    }

    // 便利なイニシャライザ
    convenience init(name: String, email: String, age: Int) {
        self.init()
        self.name = name
        self.email = email
        self.age = age
    }
}

class Post: Object {
    @Persisted(primaryKey: true) var id: UUID = UUID()
    @Persisted var title: String = ""
    @Persisted var content: String = ""
    @Persisted var createdAt: Date = Date()
    @Persisted var updatedAt: Date = Date()

    // リレーションシップ（Many-to-One）
    @Persisted(originProperty: "posts") var author: LinkingObjects<User>

    // リレーションシップ（Many-to-Many）
    @Persisted var tags: List<Tag> = List<Tag>()

    convenience init(title: String, content: String) {
        self.init()
        self.title = title
        self.content = content
    }
}

class Tag: Object {
    @Persisted(primaryKey: true) var id: UUID = UUID()
    @Persisted var name: String = ""

    // 逆リレーションシップ
    @Persisted(originProperty: "tags") var posts: LinkingObjects<Post>

    convenience init(name: String) {
        self.init()
        self.name = name
    }
}
```

## CRUD操作

### Create（作成）

```swift
import RealmSwift

class UserRepository {
    private let realm: Realm

    init() throws {
        realm = try Realm()
    }

    // ユーザーの作成
    func createUser(name: String, email: String, age: Int) throws -> User {
        let user = User(name: name, email: email, age: age)

        try realm.write {
            realm.add(user)
        }

        return user
    }

    // 複数ユーザーの一括作成
    func createUsers(_ users: [(name: String, email: String, age: Int)]) throws {
        try realm.write {
            for userData in users {
                let user = User(name: userData.name, email: userData.email, age: userData.age)
                realm.add(user)
            }
        }
    }

    // 既存オブジェクトの更新または作成
    func createOrUpdateUser(id: UUID, name: String, email: String, age: Int) throws {
        let user = User()
        user.id = id
        user.name = name
        user.email = email
        user.age = age
        user.updatedAt = Date()

        try realm.write {
            realm.add(user, update: .modified)
        }
    }
}
```

### Read（読み取り）

```swift
extension UserRepository {
    // 全ユーザーの取得
    func fetchAllUsers() -> Results<User> {
        return realm.objects(User.self)
    }

    // IDによるユーザーの取得
    func fetchUser(by id: UUID) -> User? {
        return realm.object(ofType: User.self, forPrimaryKey: id)
    }

    // メールアドレスによるユーザーの取得
    func fetchUser(by email: String) -> User? {
        return realm.objects(User.self)
            .filter("email == %@", email)
            .first
    }

    // アクティブなユーザーの取得
    func fetchActiveUsers() -> Results<User> {
        return realm.objects(User.self)
            .filter("isActive == true")
    }

    // 年齢範囲によるユーザーの取得
    func fetchUsers(ageRange: ClosedRange<Int>) -> Results<User> {
        return realm.objects(User.self)
            .filter("age >= %d AND age <= %d", ageRange.lowerBound, ageRange.upperBound)
    }

    // ソート付き取得
    func fetchUsersSortedByName() -> Results<User> {
        return realm.objects(User.self)
            .sorted(byKeyPath: "name", ascending: true)
    }

    // 複数条件でのソート
    func fetchUsersSortedByAgeAndName() -> Results<User> {
        return realm.objects(User.self)
            .sorted(by: [
                SortDescriptor(keyPath: "age", ascending: false),
                SortDescriptor(keyPath: "name", ascending: true)
            ])
    }
}
```

### Update（更新）

```swift
extension UserRepository {
    // ユーザー情報の更新
    func updateUser(id: UUID, name: String? = nil, email: String? = nil, age: Int? = nil) throws {
        guard let user = fetchUser(by: id) else {
            throw RepositoryError.userNotFound
        }

        try realm.write {
            if let name = name {
                user.name = name
            }

            if let email = email {
                user.email = email
            }

            if let age = age {
                user.age = age
            }

            user.updatedAt = Date()
        }
    }

    // 条件に一致するユーザーの一括更新
    func updateUsers(matching predicate: NSPredicate, updates: (User) -> Void) throws {
        let users = realm.objects(User.self).filter(predicate)

        try realm.write {
            for user in users {
                updates(user)
                user.updatedAt = Date()
            }
        }
    }

    // ユーザーの有効化/無効化
    func setUserActive(id: UUID, isActive: Bool) throws {
        guard let user = fetchUser(by: id) else {
            throw RepositoryError.userNotFound
        }

        try realm.write {
            user.isActive = isActive
            user.updatedAt = Date()
        }
    }
}

enum RepositoryError: Error {
    case userNotFound
}
```

### Delete（削除）

```swift
extension UserRepository {
    // ユーザーの削除
    func deleteUser(id: UUID) throws {
        guard let user = fetchUser(by: id) else {
            throw RepositoryError.userNotFound
        }

        try realm.write {
            realm.delete(user)
        }
    }

    // ユーザーの一括削除
    func deleteUsers(_ users: Results<User>) throws {
        try realm.write {
            realm.delete(users)
        }
    }

    // 条件に一致するユーザーの削除
    func deleteInactiveUsers() throws {
        let inactiveUsers = realm.objects(User.self).filter("isActive == false")

        try realm.write {
            realm.delete(inactiveUsers)
        }
    }

    // 全ユーザーの削除
    func deleteAllUsers() throws {
        let users = realm.objects(User.self)

        try realm.write {
            realm.delete(users)
        }
    }
}
```

## クエリ（filter、sorted）

Realmは強力なクエリ機能を提供します。

```swift
class UserQueryBuilder {
    private let realm: Realm

    init() throws {
        realm = try Realm()
    }

    // 基本的なフィルター
    func queryByName(name: String) -> Results<User> {
        return realm.objects(User.self)
            .filter("name == %@", name)
    }

    // 部分一致検索（CONTAINS）
    func queryByNameContains(_ keyword: String) -> Results<User> {
        return realm.objects(User.self)
            .filter("name CONTAINS[c] %@", keyword)
    }

    // 前方一致検索（BEGINSWITH）
    func queryByNameStartsWith(_ prefix: String) -> Results<User> {
        return realm.objects(User.self)
            .filter("name BEGINSWITH[c] %@", prefix)
    }

    // 範囲検索
    func queryByAgeBetween(_ min: Int, and max: Int) -> Results<User> {
        return realm.objects(User.self)
            .filter("age BETWEEN {%d, %d}", min, max)
    }

    // IN条件
    func queryByIds(_ ids: [UUID]) -> Results<User> {
        return realm.objects(User.self)
            .filter("id IN %@", ids)
    }

    // 複合条件（AND）
    func queryActiveUsersOlderThan(_ age: Int) -> Results<User> {
        return realm.objects(User.self)
            .filter("isActive == true AND age >= %d", age)
    }

    // NSPredicateを使用したクエリ
    func queryWithPredicate(_ predicate: NSPredicate) -> Results<User> {
        return realm.objects(User.self)
            .filter(predicate)
    }

    // リレーションシップを使用したクエリ
    func queryUsersWithPosts() -> Results<User> {
        return realm.objects(User.self)
            .filter("posts.@count > 0")
    }

    // 集計クエリ
    func queryUsersWithManyPosts(minimumCount: Int) -> Results<User> {
        return realm.objects(User.self)
            .filter("posts.@count >= %d", minimumCount)
    }
}
```

## リレーションシップ

Realmは、List（一対多、多対多）とLinkingObjects（逆リレーション）を使用してリレーションシップを管理します。

```swift
class PostRepository {
    private let realm: Realm

    init() throws {
        realm = try Realm()
    }

    // ユーザーの投稿を取得
    func fetchPosts(by user: User) -> List<Post> {
        return user.posts
    }

    // 投稿を作成してユーザーに関連付け
    func createPost(title: String, content: String, author: User) throws {
        let post = Post(title: title, content: content)

        try realm.write {
            realm.add(post)
            author.posts.append(post)
        }
    }

    // 投稿にタグを追加
    func addTags(to post: Post, tags: [Tag]) throws {
        try realm.write {
            for tag in tags {
                if !post.tags.contains(tag) {
                    post.tags.append(tag)
                }
            }
        }
    }

    // タグから投稿を検索
    func fetchPosts(withTag tag: Tag) -> Results<LinkingObjects<Post>> {
        return tag.posts
    }

    // 複数タグを持つ投稿を検索
    func fetchPosts(withAllTags tags: [Tag]) -> Results<Post> {
        var results = realm.objects(Post.self)

        for tag in tags {
            results = results.filter("ANY tags == %@", tag)
        }

        return results
    }
}
```

## 通知機能（observe）

Realmは、データの変更を監視してリアクティブに更新を受け取る機能を提供します。

```swift
import SwiftUI
import RealmSwift
import Combine

class UserListViewModel: ObservableObject {
    @Published var users: [User] = []

    private let realm: Realm
    private var notificationToken: NotificationToken?
    private var cancellables = Set<AnyCancellable>()

    init() throws {
        realm = try Realm()
        setupObservers()
    }

    deinit {
        notificationToken?.invalidate()
    }

    private func setupObservers() {
        let results = realm.objects(User.self).sorted(byKeyPath: "name", ascending: true)

        // Realmの通知を監視
        notificationToken = results.observe { [weak self] changes in
            switch changes {
            case .initial(let collection):
                self?.users = Array(collection)

            case .update(let collection, let deletions, let insertions, let modifications):
                // 効率的な更新処理
                self?.users = Array(collection)

                print("Deletions: \(deletions)")
                print("Insertions: \(insertions)")
                print("Modifications: \(modifications)")

            case .error(let error):
                print("Error observing: \(error)")
            }
        }
    }

    // ユーザーの追加
    func addUser(name: String, email: String, age: Int) {
        let user = User(name: name, email: email, age: age)

        do {
            try realm.write {
                realm.add(user)
            }
        } catch {
            print("Error adding user: \(error)")
        }
    }

    // ユーザーの削除
    func deleteUser(_ user: User) {
        do {
            try realm.write {
                realm.delete(user)
            }
        } catch {
            print("Error deleting user: \(error)")
        }
    }
}

// SwiftUIでの使用例
struct UserListView: View {
    @StateObject private var viewModel: UserListViewModel

    init() {
        do {
            _viewModel = StateObject(wrappedValue: try UserListViewModel())
        } catch {
            fatalError("Failed to initialize ViewModel: \(error)")
        }
    }

    var body: some View {
        NavigationView {
            List {
                ForEach(viewModel.users, id: \.id) { user in
                    VStack(alignment: .leading) {
                        Text(user.name)
                            .font(.headline)
                        Text(user.email)
                            .font(.subheadline)
                            .foregroundColor(.secondary)
                    }
                }
                .onDelete { indexSet in
                    for index in indexSet {
                        viewModel.deleteUser(viewModel.users[index])
                    }
                }
            }
            .navigationTitle("ユーザー一覧")
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    Button("追加") {
                        viewModel.addUser(
                            name: "New User",
                            email: "user@example.com",
                            age: 25
                        )
                    }
                }
            }
        }
    }
}
```

## マルチスレッド対応

Realmは各スレッドで独立したインスタンスを使用する必要があります。

```swift
import RealmSwift

class BackgroundTaskService {
    // バックグラウンドスレッドでのデータ処理
    func processDataInBackground(completion: @escaping (Result<Void, Error>) -> Void) {
        DispatchQueue.global(qos: .background).async {
            do {
                // 各スレッドで新しいRealmインスタンスを作成
                let realm = try Realm()

                try realm.write {
                    // 重い処理を実行
                    for i in 0..<1000 {
                        let user = User(name: "User \(i)", email: "user\(i)@example.com", age: 20 + i % 50)
                        realm.add(user)
                    }
                }

                DispatchQueue.main.async {
                    completion(.success(()))
                }
            } catch {
                DispatchQueue.main.async {
                    completion(.failure(error))
                }
            }
        }
    }

    // async/awaitを使用したバックグラウンド処理
    func processDataAsync() async throws {
        try await Task.detached {
            let realm = try Realm()

            try realm.write {
                // 処理を実行
                for i in 0..<1000 {
                    let user = User(name: "User \(i)", email: "user\(i)@example.com", age: 20 + i % 50)
                    realm.add(user)
                }
            }
        }.value
    }
}
```

## マイグレーション

スキーマ変更時にはマイグレーションが必要です。

```swift
import RealmSwift

class RealmMigrationManager {
    static func performMigration() {
        let config = Realm.Configuration(
            schemaVersion: 3,
            migrationBlock: { migration, oldSchemaVersion in
                // バージョン1へのマイグレーション
                if oldSchemaVersion < 1 {
                    migration.enumerateObjects(ofType: User.className()) { oldObject, newObject in
                        // 新しいプロパティのデフォルト値を設定
                        newObject!["isActive"] = true
                    }
                }

                // バージョン2へのマイグレーション
                if oldSchemaVersion < 2 {
                    migration.enumerateObjects(ofType: User.className()) { oldObject, newObject in
                        // プロパティ名の変更
                        if let fullName = oldObject!["fullName"] as? String {
                            let components = fullName.components(separatedBy: " ")
                            newObject!["firstName"] = components.first ?? ""
                            newObject!["lastName"] = components.last ?? ""
                        }
                    }
                }

                // バージョン3へのマイグレーション
                if oldSchemaVersion < 3 {
                    // 何もしない（新しいプロパティは自動的に追加される）
                }
            }
        )

        Realm.Configuration.defaultConfiguration = config

        // マイグレーションを実行
        do {
            _ = try Realm()
            print("Migration completed successfully")
        } catch {
            print("Migration failed: \(error)")
        }
    }
}
```

## パフォーマンス最適化

```swift
class PerformanceOptimizer {
    private let realm: Realm

    init() throws {
        realm = try Realm()
    }

    // 遅延評価を活用
    func fetchUsersLazily() -> Results<User> {
        // Resultsは遅延評価されるため、必要な時だけデータが読み込まれる
        return realm.objects(User.self)
            .filter("age >= 18")
            .sorted(byKeyPath: "name")
    }

    // バッチ処理
    func batchUpdateUsers() throws {
        let users = realm.objects(User.self)

        try realm.write {
            for user in users {
                user.updatedAt = Date()
            }
        }
    }

    // トランザクションのグループ化
    func groupedTransactions() throws {
        // 複数の書き込みを1つのトランザクションにまとめる
        try realm.write {
            for i in 0..<100 {
                let user = User(name: "User \(i)", email: "user\(i)@example.com", age: 20)
                realm.add(user)
            }
        }
    }

    // 不要なオブジェクトのコピーを避ける
    func efficientQuery() -> Int {
        // 件数だけが必要な場合、全オブジェクトを取得しない
        return realm.objects(User.self).filter("age >= 18").count
    }
}
```

## チェックリスト

Realm実装のベストプラクティスチェックリスト：

- [ ] Swift Package Managerでの依存関係管理を設定済み
- [ ] デフォルト設定でスキーマバージョンを指定済み
- [ ] 必要に応じて暗号化を有効化済み
- [ ] Objectサブクラスで適切なモデルを定義済み
- [ ] プライマリキーを設定済み
- [ ] CRUD操作を適切に実装済み
- [ ] filterとsortedで効率的なクエリを実装済み
- [ ] ListとLinkingObjectsでリレーションシップを管理済み
- [ ] 通知機能でリアクティブなUI更新を実装済み
- [ ] マルチスレッド処理で各スレッド独立したRealmインスタンスを使用済み

## まとめ

Realmは高速でシンプルなAPIを提供し、Core Dataの代替として活用できます。直感的なオブジェクト指向API、強力なクエリ機能、リアクティブな通知システム、そして効率的なマルチスレッド対応により、モダンなiOSアプリケーションのデータ層を構築できます。特に、通知機能を活用することで、データの変更を自動的にUIに反映させる仕組みを簡単に実装できます。

次章では、UserDefaultsのベストプラクティスについて解説します。

## 参考文献

- [Realm Swift Documentation](https://www.mongodb.com/docs/realm/sdk/swift/)
- [Realm GitHub Repository](https://github.com/realm/realm-swift)
- [Apple Developer Documentation - Concurrency](https://developer.apple.com/documentation/swift/concurrency)
