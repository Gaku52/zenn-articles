---
title: "Chapter 12: Realm実装 - モダンなモバイルデータベース"
---

# Chapter 12: Realm実装 - モダンなモバイルデータベース

## 12.1 概要

Realmは、モバイルアプリケーション向けに設計された高速でシンプルなデータベースです。Core Dataと比較して、より直感的なAPIと優れたパフォーマンスを提供します。

### 本章で学べること

- Realmの基本概念とセットアップ
- モデル定義とリレーションシップ
- CRUD操作の実装
- クエリと検索
- Realm Observationによるリアルタイム更新
- スレッド管理とベストプラクティス
- マイグレーション戦略
- SwiftUIとの統合
- パフォーマンス最適化

### RealmとCore Dataの比較

```swift
/*
Realm vs Core Data:

【Realm の利点】
✅ シンプルなAPI
✅ 高速なパフォーマンス
✅ リアルタイム同期（Realm Cloud）
✅ クロスプラットフォーム（iOS, Android, JS）
✅ より少ないボイラープレート

【Core Data の利点】
✅ Appleの公式フレームワーク
✅ より深いiOS統合
✅ iCloud同期の標準サポート
✅ 長い歴史と安定性

【使い分け】
- 新規プロジェクト、シンプルさ重視 → Realm
- Apple純正、複雑な統合が必要 → Core Data
- リアルタイム同期が必要 → Realm
*/
```

## 12.2 セットアップ

### 12.2.1 Realmのインストール

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/realm/realm-swift.git", from: "10.0.0")
]

// または CocoaPods
// pod 'RealmSwift'

// または Swift Package Manager (Xcode)
// File > Add Packages...
// https://github.com/realm/realm-swift.git
```

### 12.2.2 基本的な設定

```swift
import RealmSwift
import Foundation

class RealmManager {
    static let shared = RealmManager()

    private init() {
        configureRealm()
    }

    private func configureRealm() {
        var config = Realm.Configuration.defaultConfiguration

        // データベースファイルのパス
        let fileURL = FileManager.default
            .urls(for: .documentDirectory, in: .userDomainMask)[0]
            .appendingPathComponent("default.realm")

        config.fileURL = fileURL

        // スキーマバージョン
        config.schemaVersion = 1

        // マイグレーション
        config.migrationBlock = { migration, oldSchemaVersion in
            // マイグレーション処理
        }

        // デフォルト設定として保存
        Realm.Configuration.defaultConfiguration = config

        // 初期化を確認
        do {
            let realm = try Realm()
            print("Realm initialized successfully at: \(realm.configuration.fileURL!)")
        } catch {
            fatalError("Failed to initialize Realm: \(error)")
        }
    }

    // MARK: - Realm Instance
    var realm: Realm {
        do {
            return try Realm()
        } catch {
            fatalError("Failed to get Realm instance: \(error)")
        }
    }

    // バックグラウンド用の新しいインスタンス
    func backgroundRealm() throws -> Realm {
        return try Realm()
    }

    // MARK: - Write Transaction
    func write(_ block: (Realm) throws -> Void) throws {
        let realm = self.realm
        try realm.write {
            try block(realm)
        }
    }

    // Async write
    func write(_ block: @escaping (Realm) throws -> Void) async throws {
        try await Task {
            let realm = try Realm()
            try realm.write {
                try block(realm)
            }
        }.value
    }
}
```

### 12.2.3 高度な設定

```swift
class AdvancedRealmManager {
    static let shared = AdvancedRealmManager()

    private init() {
        configureRealm()
    }

    private func configureRealm() {
        var config = Realm.Configuration.defaultConfiguration

        // カスタムファイルパス
        config.fileURL = getRealmFileURL()

        // スキーマバージョン
        config.schemaVersion = 2

        // マイグレーション
        config.migrationBlock = { migration, oldSchemaVersion in
            self.performMigration(migration: migration, oldVersion: oldSchemaVersion)
        }

        // 暗号化（必要な場合）
        if let encryptionKey = getEncryptionKey() {
            config.encryptionKey = encryptionKey
        }

        // Delete if migration needed（開発時のみ）
        #if DEBUG
        config.deleteRealmIfMigrationNeeded = true
        #endif

        // Read-only mode
        // config.readOnly = true

        // In-memory Realm（テスト用）
        // config.inMemoryIdentifier = "test"

        Realm.Configuration.defaultConfiguration = config

        // 初期データ投入
        createDefaultDataIfNeeded()
    }

    private func getRealmFileURL() -> URL {
        let documentsPath = FileManager.default.urls(
            for: .documentDirectory,
            in: .userDomainMask
        )[0]

        return documentsPath.appendingPathComponent("myapp.realm")
    }

    private func getEncryptionKey() -> Data? {
        // Keychainから暗号化キーを取得
        // 実装は省略
        return nil
    }

    private func performMigration(migration: Migration, oldVersion: UInt64) {
        if oldVersion < 1 {
            // バージョン0から1へのマイグレーション
            migration.enumerateObjects(ofType: User.className()) { oldObject, newObject in
                // マイグレーション処理
            }
        }

        if oldVersion < 2 {
            // バージョン1から2へのマイグレーション
        }
    }

    private func createDefaultDataIfNeeded() {
        let realm = try! Realm()

        guard realm.isEmpty else { return }

        // 初期データを作成
        try! realm.write {
            // デフォルトデータの作成
        }
    }
}
```

## 12.3 モデル定義

### 12.3.1 基本的なモデル

```swift
import RealmSwift

// MARK: - User Model
class User: Object {
    @Persisted(primaryKey: true) var id: UUID
    @Persisted var username: String
    @Persisted var email: String
    @Persisted var firstName: String?
    @Persisted var lastName: String?
    @Persisted var bio: String?
    @Persisted var avatarURL: String?
    @Persisted var createdAt: Date
    @Persisted var updatedAt: Date
    @Persisted var isActive: Bool

    // Relationships
    @Persisted var posts: List<Post>
    @Persisted var comments: List<Comment>
    @Persisted var followers: List<User>
    @Persisted var following: List<User>

    // Convenience initializer
    convenience init(username: String, email: String) {
        self.init()
        self.id = UUID()
        self.username = username
        self.email = email
        self.createdAt = Date()
        self.updatedAt = Date()
        self.isActive = true
    }

    // Computed Properties
    var fullName: String {
        if let firstName = firstName, let lastName = lastName {
            return "\(firstName) \(lastName)"
        }
        return username
    }

    var postCount: Int {
        return posts.count
    }

    var followerCount: Int {
        return followers.count
    }

    var followingCount: Int {
        return following.count
    }
}

// MARK: - Post Model
class Post: Object {
    @Persisted(primaryKey: true) var id: UUID
    @Persisted var title: String
    @Persisted var content: String
    @Persisted var imageURL: String?
    @Persisted var viewCount: Int
    @Persisted var likeCount: Int
    @Persisted var createdAt: Date
    @Persisted var updatedAt: Date
    @Persisted var isPublished: Bool

    // Relationships
    @Persisted(originProperty: "posts") var author: LinkingObjects<User>
    @Persisted var comments: List<Comment>
    @Persisted var tags: List<Tag>

    convenience init(title: String, content: String) {
        self.init()
        self.id = UUID()
        self.title = title
        self.content = content
        self.viewCount = 0
        self.likeCount = 0
        self.createdAt = Date()
        self.updatedAt = Date()
        self.isPublished = false
    }

    var commentCount: Int {
        return comments.count
    }

    var authorName: String {
        return author.first?.username ?? "Unknown"
    }
}

// MARK: - Comment Model
class Comment: Object {
    @Persisted(primaryKey: true) var id: UUID
    @Persisted var content: String
    @Persisted var createdAt: Date
    @Persisted var updatedAt: Date

    // Relationships
    @Persisted(originProperty: "comments") var author: LinkingObjects<User>
    @Persisted(originProperty: "comments") var post: LinkingObjects<Post>
    @Persisted var parentComment: Comment?
    @Persisted var replies: List<Comment>

    convenience init(content: String) {
        self.init()
        self.id = UUID()
        self.content = content
        self.createdAt = Date()
        self.updatedAt = Date()
    }

    var replyCount: Int {
        return replies.count
    }

    var depth: Int {
        var depth = 0
        var current = parentComment
        while current != nil {
            depth += 1
            current = current?.parentComment
        }
        return depth
    }
}

// MARK: - Tag Model
class Tag: Object {
    @Persisted(primaryKey: true) var id: UUID
    @Persisted var name: String
    @Persisted var createdAt: Date

    @Persisted(originProperty: "tags") var posts: LinkingObjects<Post>

    convenience init(name: String) {
        self.init()
        self.id = UUID()
        self.name = name
        self.createdAt = Date()
    }

    var postCount: Int {
        return posts.count
    }
}

// MARK: - Identifiable Conformance
extension User: Identifiable {}
extension Post: Identifiable {}
extension Comment: Identifiable {}
extension Tag: Identifiable {}
```

### 12.3.2 埋め込みオブジェクト

```swift
// 埋め込みオブジェクト（親オブジェクトと共に保存される）
class Address: EmbeddedObject {
    @Persisted var street: String?
    @Persisted var city: String?
    @Persisted var state: String?
    @Persisted var zipCode: String?
    @Persisted var country: String?

    var fullAddress: String {
        let components = [street, city, state, zipCode, country].compactMap { $0 }
        return components.joined(separator: ", ")
    }
}

class UserProfile: Object {
    @Persisted(primaryKey: true) var id: UUID
    @Persisted var user: User?
    @Persisted var address: Address?
    @Persisted var phoneNumber: String?
    @Persisted var dateOfBirth: Date?

    convenience init(user: User) {
        self.init()
        self.id = UUID()
        self.user = user
    }
}
```

### 12.3.3 カスタム型

```swift
// Enum
enum UserRole: String, PersistableEnum {
    case admin
    case moderator
    case user
    case guest
}

// 複雑なEnum
enum PostStatus: String, PersistableEnum, CaseIterable {
    case draft
    case published
    case archived
    case deleted

    var displayName: String {
        switch self {
        case .draft: return "Draft"
        case .published: return "Published"
        case .archived: return "Archived"
        case .deleted: return "Deleted"
        }
    }
}

class AdvancedUser: Object {
    @Persisted(primaryKey: true) var id: UUID
    @Persisted var username: String
    @Persisted var role: UserRole
    @Persisted var createdAt: Date

    // Array of primitives
    @Persisted var favoriteColors: List<String>
    @Persisted var scores: List<Int>

    // Dictionary (Realm 10.8+)
    @Persisted var metadata: Map<String, String>

    convenience init(username: String, role: UserRole = .user) {
        self.init()
        self.id = UUID()
        self.username = username
        self.role = role
        self.createdAt = Date()
    }
}

class AdvancedPost: Object {
    @Persisted(primaryKey: true) var id: UUID
    @Persisted var title: String
    @Persisted var status: PostStatus
    @Persisted var tags: List<String>
    @Persisted var metadata: Map<String, String>

    convenience init(title: String) {
        self.init()
        self.id = UUID()
        self.title = title
        self.status = .draft
    }
}
```

## 12.4 CRUD操作

### 12.4.1 Create - データの作成

```swift
class UserRepository {
    private let realm: Realm

    init(realm: Realm = try! Realm()) {
        self.realm = realm
    }

    // MARK: - Create
    func createUser(username: String, email: String) throws -> User {
        let user = User(username: username, email: email)

        try realm.write {
            realm.add(user)
        }

        return user
    }

    func createUser(from dto: CreateUserDTO) throws -> User {
        return try createUser(username: dto.username, email: dto.email)
    }

    // 複数作成
    func createUsers(_ dtos: [CreateUserDTO]) throws -> [User] {
        var users: [User] = []

        try realm.write {
            for dto in dtos {
                let user = User(username: dto.username, email: dto.email)
                realm.add(user)
                users.append(user)
            }
        }

        return users
    }

    // Update or Insert
    func upsertUser(id: UUID, username: String, email: String) throws -> User {
        if let existingUser = realm.object(ofType: User.self, forPrimaryKey: id) {
            try realm.write {
                existingUser.username = username
                existingUser.email = email
                existingUser.updatedAt = Date()
            }
            return existingUser
        } else {
            let user = User(username: username, email: email)
            user.id = id
            try realm.write {
                realm.add(user)
            }
            return user
        }
    }

    // Async create
    func createUserAsync(username: String, email: String) async throws -> User {
        return try await Task {
            let realm = try Realm()
            let user = User(username: username, email: email)

            try realm.write {
                realm.add(user)
            }

            return user
        }.value
    }
}

class PostRepository {
    private let realm: Realm

    init(realm: Realm = try! Realm()) {
        self.realm = realm
    }

    func createPost(title: String, content: String, author: User) throws -> Post {
        let post = Post(title: title, content: content)

        try realm.write {
            author.posts.append(post)
            realm.add(post)
        }

        return post
    }

    func createPost(title: String, content: String, authorID: UUID) throws -> Post {
        guard let author = realm.object(ofType: User.self, forPrimaryKey: authorID) else {
            throw RealmRepositoryError.objectNotFound
        }

        return try createPost(title: title, content: content, author: author)
    }

    func createPostWithTags(
        title: String,
        content: String,
        author: User,
        tagNames: [String]
    ) throws -> Post {
        let post = Post(title: title, content: content)

        try realm.write {
            author.posts.append(post)

            // Tagsを追加
            for tagName in tagNames {
                let tag = findOrCreateTag(name: tagName, in: realm)
                post.tags.append(tag)
            }

            realm.add(post)
        }

        return post
    }

    private func findOrCreateTag(name: String, in realm: Realm) -> Tag {
        // 既存のTagを検索
        if let tag = realm.objects(Tag.self).filter("name ==[cd] %@", name).first {
            return tag
        }

        // 新規作成
        let tag = Tag(name: name)
        realm.add(tag)
        return tag
    }
}

// DTOs
struct CreateUserDTO {
    let username: String
    let email: String
}

struct CreatePostDTO {
    let title: String
    let content: String
    let authorID: UUID
    let tagNames: [String]
}

enum RealmRepositoryError: Error {
    case objectNotFound
    case saveFailed
    case deleteFailed
    case invalidData
}
```

### 12.4.2 Read - データの取得

```swift
extension UserRepository {
    // MARK: - Fetch Single
    func fetchUser(byID id: UUID) -> User? {
        return realm.object(ofType: User.self, forPrimaryKey: id)
    }

    func fetchUser(byUsername username: String) -> User? {
        return realm.objects(User.self)
            .filter("username == %@", username)
            .first
    }

    func fetchUser(byEmail email: String) -> User? {
        return realm.objects(User.self)
            .filter("email == %@", email)
            .first
    }

    // MARK: - Fetch Multiple
    func fetchAllUsers() -> Results<User> {
        return realm.objects(User.self)
            .sorted(byKeyPath: "createdAt", ascending: false)
    }

    func fetchActiveUsers() -> Results<User> {
        return realm.objects(User.self)
            .filter("isActive == true")
            .sorted(byKeyPath: "username", ascending: true)
    }

    // MARK: - Search
    func searchUsers(query: String) -> Results<User> {
        return realm.objects(User.self)
            .filter("username CONTAINS[cd] %@ OR firstName CONTAINS[cd] %@ OR lastName CONTAINS[cd] %@ OR email CONTAINS[cd] %@",
                   query, query, query, query)
            .sorted(byKeyPath: "username", ascending: true)
    }

    // MARK: - Advanced Queries
    func fetchUsers(createdAfter date: Date) -> Results<User> {
        return realm.objects(User.self)
            .filter("createdAt > %@", date)
            .sorted(byKeyPath: "createdAt", ascending: false)
    }

    func fetchUsers(withMinimumPosts count: Int) -> Results<User> {
        return realm.objects(User.self)
            .filter("posts.@count >= %d", count)
            .sorted(byKeyPath: "posts.@count", ascending: false)
    }

    // MARK: - Pagination
    func fetchUsers(offset: Int, limit: Int) -> [User] {
        let results = realm.objects(User.self)
            .sorted(byKeyPath: "createdAt", ascending: false)

        let startIndex = offset
        let endIndex = min(offset + limit, results.count)

        guard startIndex < results.count else { return [] }

        return Array(results[startIndex..<endIndex])
    }

    // MARK: - Count
    func countUsers() -> Int {
        return realm.objects(User.self).count
    }

    func countActiveUsers() -> Int {
        return realm.objects(User.self)
            .filter("isActive == true")
            .count
    }

    // MARK: - Array Conversion
    func fetchAllUsersAsArray() -> [User] {
        return Array(fetchAllUsers())
    }
}

extension PostRepository {
    // MARK: - Fetch Posts
    func fetchPost(byID id: UUID) -> Post? {
        return realm.object(ofType: Post.self, forPrimaryKey: id)
    }

    func fetchAllPosts() -> Results<Post> {
        return realm.objects(Post.self)
            .sorted(byKeyPath: "createdAt", ascending: false)
    }

    func fetchPublishedPosts() -> Results<Post> {
        return realm.objects(Post.self)
            .filter("isPublished == true")
            .sorted(byKeyPath: "createdAt", ascending: false)
    }

    func fetchPosts(byAuthor author: User) -> Results<Post> {
        return author.posts
            .sorted(byKeyPath: "createdAt", ascending: false)
    }

    func fetchPosts(byAuthorID authorID: UUID) -> Results<Post>? {
        guard let author = realm.object(ofType: User.self, forPrimaryKey: authorID) else {
            return nil
        }
        return fetchPosts(byAuthor: author)
    }

    func fetchPosts(withTag tag: Tag) -> Results<Post> {
        return tag.posts
            .sorted(byKeyPath: "createdAt", ascending: false)
    }

    func fetchPosts(withTagName tagName: String) -> Results<Post> {
        return realm.objects(Post.self)
            .filter("ANY tags.name ==[cd] %@", tagName)
            .sorted(byKeyPath: "createdAt", ascending: false)
    }

    // MARK: - Advanced Queries
    func fetchPopularPosts(minLikes: Int, limit: Int = 10) -> [Post] {
        let results = realm.objects(Post.self)
            .filter("likeCount >= %d", minLikes)
            .sorted(byKeyPath: "likeCount", ascending: false)

        return Array(results.prefix(limit))
    }

    func fetchRecentPosts(days: Int, limit: Int = 20) -> [Post] {
        let startDate = Calendar.current.date(byAdding: .day, value: -days, to: Date())!

        let results = realm.objects(Post.self)
            .filter("createdAt >= %@", startDate)
            .sorted(byKeyPath: "createdAt", ascending: false)

        return Array(results.prefix(limit))
    }

    func fetchTrendingPosts(minViews: Int, minLikes: Int, limit: Int = 10) -> [Post] {
        let results = realm.objects(Post.self)
            .filter("viewCount >= %d AND likeCount >= %d", minViews, minLikes)
            .sorted(byKeyPath: "viewCount", ascending: false)

        return Array(results.prefix(limit))
    }

    // MARK: - Search
    func searchPosts(query: String) -> Results<Post> {
        return realm.objects(Post.self)
            .filter("title CONTAINS[cd] %@ OR content CONTAINS[cd] %@", query, query)
            .sorted(byKeyPath: "createdAt", ascending: false)
    }

    // MARK: - Complex Queries
    func fetchPosts(
        byAuthorID authorID: UUID?,
        withStatus isPublished: Bool?,
        createdAfter: Date?,
        tagNames: [String]?
    ) -> Results<Post> {
        var predicates: [NSPredicate] = []

        if let authorID = authorID {
            if let author = realm.object(ofType: User.self, forPrimaryKey: authorID) {
                predicates.append(NSPredicate(format: "ANY author.id == %@", authorID as CVarArg))
            }
        }

        if let isPublished = isPublished {
            predicates.append(NSPredicate(format: "isPublished == %@", NSNumber(value: isPublished)))
        }

        if let createdAfter = createdAfter {
            predicates.append(NSPredicate(format: "createdAt >= %@", createdAfter as NSDate))
        }

        if let tagNames = tagNames, !tagNames.isEmpty {
            let tagPredicates = tagNames.map { NSPredicate(format: "ANY tags.name ==[cd] %@", $0) }
            predicates.append(NSCompoundPredicate(orPredicateWithSubpredicates: tagPredicates))
        }

        let compoundPredicate = NSCompoundPredicate(andPredicateWithSubpredicates: predicates)

        return realm.objects(Post.self)
            .filter(compoundPredicate)
            .sorted(byKeyPath: "createdAt", ascending: false)
    }
}
```

### 12.4.3 Update - データの更新

```swift
extension UserRepository {
    // MARK: - Update
    func updateUser(
        _ user: User,
        username: String? = nil,
        email: String? = nil,
        firstName: String? = nil,
        lastName: String? = nil,
        bio: String? = nil,
        avatarURL: String? = nil
    ) throws {
        try realm.write {
            if let username = username {
                user.username = username
            }
            if let email = email {
                user.email = email
            }
            if let firstName = firstName {
                user.firstName = firstName
            }
            if let lastName = lastName {
                user.lastName = lastName
            }
            if let bio = bio {
                user.bio = bio
            }
            if let avatarURL = avatarURL {
                user.avatarURL = avatarURL
            }

            user.updatedAt = Date()
        }
    }

    func updateUserByID(
        id: UUID,
        username: String? = nil,
        email: String? = nil
    ) throws {
        guard let user = fetchUser(byID: id) else {
            throw RealmRepositoryError.objectNotFound
        }

        try updateUser(user, username: username, email: email)
    }

    func deactivateUser(_ user: User) throws {
        try realm.write {
            user.isActive = false
            user.updatedAt = Date()
        }
    }

    func reactivateUser(_ user: User) throws {
        try realm.write {
            user.isActive = true
            user.updatedAt = Date()
        }
    }

    // Async update
    func updateUserAsync(
        id: UUID,
        username: String? = nil,
        email: String? = nil
    ) async throws {
        try await Task {
            let realm = try Realm()
            guard let user = realm.object(ofType: User.self, forPrimaryKey: id) else {
                throw RealmRepositoryError.objectNotFound
            }

            try realm.write {
                if let username = username {
                    user.username = username
                }
                if let email = email {
                    user.email = email
                }
                user.updatedAt = Date()
            }
        }.value
    }
}

extension PostRepository {
    func updatePost(
        _ post: Post,
        title: String? = nil,
        content: String? = nil,
        imageURL: String? = nil
    ) throws {
        try realm.write {
            if let title = title {
                post.title = title
            }
            if let content = content {
                post.content = content
            }
            if let imageURL = imageURL {
                post.imageURL = imageURL
            }

            post.updatedAt = Date()
        }
    }

    func publishPost(_ post: Post) throws {
        try realm.write {
            post.isPublished = true
            post.updatedAt = Date()
        }
    }

    func unpublishPost(_ post: Post) throws {
        try realm.write {
            post.isPublished = false
            post.updatedAt = Date()
        }
    }

    func incrementPostViews(_ post: Post) throws {
        try realm.write {
            post.viewCount += 1
        }
    }

    func likePost(_ post: Post) throws {
        try realm.write {
            post.likeCount += 1
        }
    }

    func unlikePost(_ post: Post) throws {
        try realm.write {
            post.likeCount = max(0, post.likeCount - 1)
        }
    }

    // Batch update
    func publishAllDrafts(by author: User) throws -> Int {
        let drafts = author.posts.filter("isPublished == false")
        let count = drafts.count

        try realm.write {
            for post in drafts {
                post.isPublished = true
                post.updatedAt = Date()
            }
        }

        return count
    }
}
```

### 12.4.4 Delete - データの削除

```swift
extension UserRepository {
    // MARK: - Delete
    func deleteUser(_ user: User) throws {
        try realm.write {
            realm.delete(user)
        }
    }

    func deleteUserByID(_ id: UUID) throws {
        guard let user = fetchUser(byID: id) else {
            throw RealmRepositoryError.objectNotFound
        }

        try deleteUser(user)
    }

    func deleteUsers(_ users: [User]) throws {
        try realm.write {
            realm.delete(users)
        }
    }

    func deleteAllInactiveUsers() throws -> Int {
        let inactiveUsers = realm.objects(User.self)
            .filter("isActive == false")

        let count = inactiveUsers.count

        try realm.write {
            realm.delete(inactiveUsers)
        }

        return count
    }

    func deleteAllUsers() throws -> Int {
        let users = realm.objects(User.self)
        let count = users.count

        try realm.write {
            realm.delete(users)
        }

        return count
    }
}

extension PostRepository {
    func deletePost(_ post: Post) throws {
        try realm.write {
            realm.delete(post)
        }
    }

    func deletePostByID(_ id: UUID) throws {
        guard let post = fetchPost(byID: id) else {
            throw RealmRepositoryError.objectNotFound
        }

        try deletePost(post)
    }

    func deletePosts(_ posts: [Post]) throws {
        try realm.write {
            realm.delete(posts)
        }
    }

    func deleteAllUnpublishedPosts() throws -> Int {
        let unpublished = realm.objects(Post.self)
            .filter("isPublished == false")

        let count = unpublished.count

        try realm.write {
            realm.delete(unpublished)
        }

        return count
    }

    func deleteOldPosts(olderThan days: Int) throws -> Int {
        let cutoffDate = Calendar.current.date(byAdding: .day, value: -days, to: Date())!
        let oldPosts = realm.objects(Post.self)
            .filter("createdAt < %@", cutoffDate)

        let count = oldPosts.count

        try realm.write {
            realm.delete(oldPosts)
        }

        return count
    }
}
```

## 12.5 Realm Observation - リアルタイム更新

### 12.5.1 オブジェクトの監視

```swift
import Combine

class UserObserver {
    private var cancellables = Set<AnyCancellable>()
    private let realm: Realm

    init(realm: Realm = try! Realm()) {
        self.realm = realm
    }

    // 単一オブジェクトの監視
    func observeUser(_ user: User, onChange: @escaping (User) -> Void) {
        user.observe { change in
            switch change {
            case .change(let properties):
                print("User changed: \(properties)")
                onChange(user)
            case .error(let error):
                print("Error observing user: \(error)")
            case .deleted:
                print("User was deleted")
            }
        }
        .store(in: &cancellables)
    }

    // Resultsの監視
    func observeUsers(_ results: Results<User>, onChange: @escaping ([User]) -> Void) {
        results.observe { changes in
            switch changes {
            case .initial(let users):
                print("Initial users: \(users.count)")
                onChange(Array(users))

            case .update(let users, let deletions, let insertions, let modifications):
                print("Updated users:")
                print("  Deletions: \(deletions)")
                print("  Insertions: \(insertions)")
                print("  Modifications: \(modifications)")
                onChange(Array(users))

            case .error(let error):
                print("Error observing users: \(error)")
            }
        }
        .store(in: &cancellables)
    }

    // Collectionの監視
    func observeUserPosts(_ user: User, onChange: @escaping ([Post]) -> Void) {
        user.posts.observe { changes in
            switch changes {
            case .initial(let posts):
                onChange(Array(posts))

            case .update(let posts, _, _, _):
                onChange(Array(posts))

            case .error(let error):
                print("Error observing posts: \(error)")
            }
        }
        .store(in: &cancellables)
    }

    func stopObserving() {
        cancellables.removeAll()
    }
}
```

### 12.5.2 ViewModelでの使用

```swift
class PostListViewModel: ObservableObject {
    @Published var posts: [Post] = []
    @Published var isLoading = false
    @Published var error: Error?

    private var notificationToken: NotificationToken?
    private let realm: Realm
    private let repository: PostRepository

    init(realm: Realm = try! Realm()) {
        self.realm = realm
        self.repository = PostRepository(realm: realm)
        observePosts()
    }

    deinit {
        notificationToken?.invalidate()
    }

    private func observePosts() {
        let results = repository.fetchPublishedPosts()

        notificationToken = results.observe { [weak self] changes in
            guard let self = self else { return }

            switch changes {
            case .initial(let posts):
                self.posts = Array(posts)

            case .update(let posts, let deletions, let insertions, let modifications):
                // 変更を反映
                self.posts = Array(posts)

                // UIアニメーション用の詳細な変更情報
                print("Deletions: \(deletions)")
                print("Insertions: \(insertions)")
                print("Modifications: \(modifications)")

            case .error(let error):
                self.error = error
            }
        }
    }

    func refresh() {
        // Realmは自動的に更新されるため、特別な処理は不要
    }

    func createPost(title: String, content: String, authorID: UUID) {
        do {
            _ = try repository.createPost(title: title, content: content, authorID: authorID)
            // 自動的にobserveが発火して更新される
        } catch {
            self.error = error
        }
    }

    func deletePost(_ post: Post) {
        do {
            try repository.deletePost(post)
            // 自動的にobserveが発火して更新される
        } catch {
            self.error = error
        }
    }
}
```

### 12.5.3 Combineとの統合

```swift
extension Results: ObservableObject {
    // ResultsをCombineのPublisherとして使用
}

class CombinePostViewModel: ObservableObject {
    @Published var posts: [Post] = []

    private var cancellables = Set<AnyCancellable>()
    private let realm: Realm

    init(realm: Realm = try! Realm()) {
        self.realm = realm
        observePostsWithCombine()
    }

    private func observePostsWithCombine() {
        let results = realm.objects(Post.self)
            .filter("isPublished == true")
            .sorted(byKeyPath: "createdAt", ascending: false)

        // Resultsの変更をPublisherとして監視
        results.collectionPublisher
            .map { Array($0) }
            .receive(on: DispatchQueue.main)
            .sink(
                receiveCompletion: { completion in
                    if case .failure(let error) = completion {
                        print("Error: \(error)")
                    }
                },
                receiveValue: { [weak self] posts in
                    self?.posts = posts
                }
            )
            .store(in: &cancellables)
    }
}
```

## 12.6 リレーションシップの管理

### 12.6.1 1対多のリレーションシップ

```swift
class RelationshipManager {
    private let realm: Realm

    init(realm: Realm = try! Realm()) {
        self.realm = realm
    }

    // Postにコメントを追加
    func addComment(to post: Post, content: String, author: User) throws -> Comment {
        let comment = Comment(content: content)

        try realm.write {
            post.comments.append(comment)
            author.comments.append(comment)
        }

        return comment
    }

    // Postからコメントを削除
    func removeComment(_ comment: Comment, from post: Post) throws {
        try realm.write {
            if let index = post.comments.index(of: comment) {
                post.comments.remove(at: index)
            }
            realm.delete(comment)
        }
    }

    // ユーザーのすべてのPostを取得
    func fetchPosts(for user: User) -> Results<Post> {
        return user.posts.sorted(byKeyPath: "createdAt", ascending: false)
    }
}
```

### 12.6.2 多対多のリレーションシップ

```swift
class TagRelationshipManager {
    private let realm: Realm

    init(realm: Realm = try! Realm()) {
        self.realm = realm
    }

    // PostにTagを追加
    func addTag(to post: Post, tagName: String) throws {
        let tag = try findOrCreateTag(name: tagName)

        try realm.write {
            if !post.tags.contains(tag) {
                post.tags.append(tag)
            }
        }
    }

    // PostからTagを削除
    func removeTag(from post: Post, tag: Tag) throws {
        try realm.write {
            if let index = post.tags.index(of: tag) {
                post.tags.remove(at: index)
            }
        }
    }

    // 複数のTagを一度に追加
    func addTags(to post: Post, tagNames: [String]) throws {
        try realm.write {
            for tagName in tagNames {
                let tag = try findOrCreateTag(name: tagName)
                if !post.tags.contains(tag) {
                    post.tags.append(tag)
                }
            }
        }
    }

    // すべてのTagを削除
    func removeAllTags(from post: Post) throws {
        try realm.write {
            post.tags.removeAll()
        }
    }

    // Tagを持つすべてのPostを取得
    func fetchPosts(withTag tag: Tag) -> Results<Post> {
        return tag.posts.sorted(byKeyPath: "createdAt", ascending: false)
    }

    private func findOrCreateTag(name: String) throws -> Tag {
        if let tag = realm.objects(Tag.self)
            .filter("name ==[cd] %@", name)
            .first {
            return tag
        }

        let tag = Tag(name: name)
        try realm.write {
            realm.add(tag)
        }
        return tag
    }
}
```

### 12.6.3 自己参照リレーションシップ

```swift
class CommentRelationshipManager {
    private let realm: Realm

    init(realm: Realm = try! Realm()) {
        self.realm = realm
    }

    // コメントに返信を追加
    func addReply(to parentComment: Comment, content: String, author: User) throws -> Comment {
        let reply = Comment(content: content)

        try realm.write {
            reply.parentComment = parentComment
            parentComment.replies.append(reply)
            author.comments.append(reply)
        }

        return reply
    }

    // コメントツリーを取得
    func fetchCommentTree(for post: Post) -> [CommentNode] {
        let allComments = Array(post.comments)
        let rootComments = allComments.filter { $0.parentComment == nil }

        return rootComments.map { buildTree(from: $0, allComments: allComments) }
    }

    private func buildTree(from comment: Comment, allComments: [Comment]) -> CommentNode {
        let replies = allComments
            .filter { $0.parentComment == comment }
            .map { buildTree(from: $0, allComments: allComments) }

        return CommentNode(comment: comment, replies: replies)
    }

    // コメントの階層深度を取得
    func getDepth(of comment: Comment) -> Int {
        var depth = 0
        var current = comment.parentComment

        while current != nil {
            depth += 1
            current = current?.parentComment
        }

        return depth
    }
}

struct CommentNode {
    let comment: Comment
    let replies: [CommentNode]

    var depth: Int {
        comment.depth
    }
}
```

## 12.7 マイグレーション

### 12.7.1 基本的なマイグレーション

```swift
class RealmMigrationManager {
    static func performMigration() {
        let config = Realm.Configuration(
            schemaVersion: 3,
            migrationBlock: { migration, oldSchemaVersion in
                // Version 0 → 1
                if oldSchemaVersion < 1 {
                    migrateToVersion1(migration: migration)
                }

                // Version 1 → 2
                if oldSchemaVersion < 2 {
                    migrateToVersion2(migration: migration)
                }

                // Version 2 → 3
                if oldSchemaVersion < 3 {
                    migrateToVersion3(migration: migration)
                }
            }
        )

        Realm.Configuration.defaultConfiguration = config
    }

    // プロパティの追加
    private static func migrateToVersion1(migration: Migration) {
        migration.enumerateObjects(ofType: User.className()) { oldObject, newObject in
            // 新しいプロパティにデフォルト値を設定
            newObject?["bio"] = ""
            newObject?["isActive"] = true
        }
    }

    // プロパティ名の変更
    private static func migrateToVersion2(migration: Migration) {
        migration.enumerateObjects(ofType: Post.className()) { oldObject, newObject in
            // 古いプロパティから新しいプロパティへ値をコピー
            if let oldTitle = oldObject?["oldTitle"] as? String {
                newObject?["title"] = oldTitle
            }
        }
    }

    // 複雑な変換
    private static func migrateToVersion3(migration: Migration) {
        migration.enumerateObjects(ofType: User.className()) { oldObject, newObject in
            // 複数のプロパティを結合
            let firstName = oldObject?["firstName"] as? String ?? ""
            let lastName = oldObject?["lastName"] as? String ?? ""
            newObject?["fullName"] = "\(firstName) \(lastName)".trimmingCharacters(in: .whitespaces)
        }
    }
}
```

### 12.7.2 データ変換を伴うマイグレーション

```swift
class AdvancedMigrationManager {
    static func setupWithMigration() {
        let config = Realm.Configuration(
            schemaVersion: 5,
            migrationBlock: { migration, oldSchemaVersion in
                if oldSchemaVersion < 4 {
                    migrateUserRoles(migration: migration)
                }

                if oldSchemaVersion < 5 {
                    migratePostStatus(migration: migration)
                }
            }
        )

        Realm.Configuration.defaultConfiguration = config
    }

    // Enum値の変換
    private static func migrateUserRoles(migration: Migration) {
        migration.enumerateObjects(ofType: User.className()) { oldObject, newObject in
            // 古い数値ベースのroleを文字列ベースに変換
            if let oldRole = oldObject?["roleValue"] as? Int {
                let newRole: String
                switch oldRole {
                case 0: newRole = "guest"
                case 1: newRole = "user"
                case 2: newRole = "moderator"
                case 3: newRole = "admin"
                default: newRole = "user"
                }
                newObject?["role"] = newRole
            }
        }
    }

    // 複数プロパティの統合
    private static func migratePostStatus(migration: Migration) {
        migration.enumerateObjects(ofType: Post.className()) { oldObject, newObject in
            let isDraft = oldObject?["isDraft"] as? Bool ?? false
            let isArchived = oldObject?["isArchived"] as? Bool ?? false
            let isDeleted = oldObject?["isDeleted"] as? Bool ?? false

            let status: String
            if isDeleted {
                status = "deleted"
            } else if isArchived {
                status = "archived"
            } else if isDraft {
                status = "draft"
            } else {
                status = "published"
            }

            newObject?["status"] = status
        }
    }

    // リレーションシップの変更
    private static func migrateRelationships(migration: Migration) {
        migration.enumerateObjects(ofType: Post.className()) { oldObject, newObject in
            // 1対1から1対多への変更など
            if let authorID = oldObject?["authorID"] as? String {
                // 新しいリレーションシップの設定
                newObject?["authorID"] = authorID
            }
        }
    }
}
```

## 12.8 スレッド管理

### 12.8.1 バックグラウンド処理

```swift
class ThreadSafeRealmManager {
    // ❌ 誤った実装（スレッドセーフではない）
    func badExample() {
        let realm = try! Realm()
        let user = realm.objects(User.self).first!

        DispatchQueue.global().async {
            // エラー！異なるスレッドからアクセス
            print(user.username)
        }
    }

    // ✅ 正しい実装
    func goodExample() {
        let realm = try! Realm()
        guard let user = realm.objects(User.self).first else { return }

        // ThreadSafeReferenceを使用
        let userRef = ThreadSafeReference(to: user)

        DispatchQueue.global().async {
            let backgroundRealm = try! Realm()
            guard let user = backgroundRealm.resolve(userRef) else { return }

            // 安全にアクセス
            print(user.username)
        }
    }

    // Async/Awaitでの実装
    func asyncExample() async throws {
        let realm = try await Task {
            try Realm()
        }.value

        let user = realm.objects(User.self).first!

        // 新しいタスクで処理
        try await Task {
            let backgroundRealm = try Realm()

            try backgroundRealm.write {
                if let user = backgroundRealm.objects(User.self)
                    .filter("id == %@", user.id).first {
                    user.username = "Updated"
                }
            }
        }.value
    }

    // ThreadSafeReferenceの活用
    func processInBackground(users: [User]) async throws {
        let userRefs = users.map { ThreadSafeReference(to: $0) }

        try await Task {
            let backgroundRealm = try Realm()

            try backgroundRealm.write {
                for userRef in userRefs {
                    if let user = backgroundRealm.resolve(userRef) {
                        // 処理
                        user.updatedAt = Date()
                    }
                }
            }
        }.value
    }
}
```

### 12.8.2 バッチ処理

```swift
class BatchProcessor {
    func processBatchInBackground(userIDs: [UUID]) async throws {
        try await Task {
            let realm = try Realm()

            try realm.write {
                for userID in userIDs {
                    if let user = realm.object(ofType: User.self, forPrimaryKey: userID) {
                        // 処理
                        user.isActive = true
                        user.updatedAt = Date()
                    }
                }
            }
        }.value
    }

    func processBatchWithProgress(
        userIDs: [UUID],
        onProgress: @escaping (Double) -> Void
    ) async throws {
        let total = Double(userIDs.count)

        try await Task {
            let realm = try Realm()

            for (index, userID) in userIDs.enumerated() {
                try realm.write {
                    if let user = realm.object(ofType: User.self, forPrimaryKey: userID) {
                        user.isActive = true
                        user.updatedAt = Date()
                    }
                }

                // 進捗を報告
                let progress = Double(index + 1) / total
                await MainActor.run {
                    onProgress(progress)
                }
            }
        }.value
    }
}
```

## 12.9 SwiftUIとの統合

### 12.9.1 @ObservedRealmObjectの使用

```swift
import SwiftUI
import RealmSwift

struct UserDetailView: View {
    @ObservedRealmObject var user: User

    var body: some View {
        Form {
            Section("Profile") {
                TextField("Username", text: $user.username)
                TextField("Email", text: $user.email)
                TextField("Bio", text: $user.bio.bound)
            }

            Section("Stats") {
                LabeledContent("Posts", value: "\(user.postCount)")
                LabeledContent("Followers", value: "\(user.followerCount)")
                LabeledContent("Following", value: "\(user.followingCount)")
            }

            Section {
                Toggle("Active", isOn: $user.isActive)
            }
        }
        .navigationTitle(user.username)
    }
}

// Optional binding helper
extension Optional where Wrapped == String {
    var bound: String {
        get { self ?? "" }
        set { self = newValue.isEmpty ? nil : newValue }
    }
}
```

### 12.9.2 @ObservedResultsの使用

```swift
struct PostListView: View {
    @ObservedResults(Post.self) var posts

    var body: some View {
        List {
            ForEach(posts) { post in
                NavigationLink(destination: PostDetailView(post: post)) {
                    PostRow(post: post)
                }
            }
            .onDelete(perform: delete)
        }
        .navigationTitle("Posts")
        .toolbar {
            ToolbarItem(placement: .navigationBarTrailing) {
                Button("Add") {
                    addPost()
                }
            }
        }
    }

    private func addPost() {
        let post = Post(title: "New Post", content: "Content")
        $posts.append(post)
    }

    private func delete(at offsets: IndexSet) {
        $posts.remove(atOffsets: offsets)
    }
}

struct PostRow: View {
    @ObservedRealmObject var post: Post

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text(post.title)
                .font(.headline)

            Text(post.content)
                .font(.body)
                .foregroundColor(.secondary)
                .lineLimit(2)

            HStack {
                Label("\(post.likeCount)", systemImage: "heart.fill")
                Label("\(post.viewCount)", systemImage: "eye.fill")
                Label("\(post.commentCount)", systemImage: "bubble.left.fill")
            }
            .font(.caption)
            .foregroundColor(.secondary)
        }
        .padding(.vertical, 4)
    }
}
```

### 12.9.3 フィルタリングとソート

```swift
struct FilteredPostListView: View {
    @ObservedResults(
        Post.self,
        filter: NSPredicate(format: "isPublished == true"),
        sortDescriptor: SortDescriptor(keyPath: "createdAt", ascending: false)
    ) var posts

    @State private var searchText = ""
    @State private var showPublishedOnly = true

    var filteredPosts: [Post] {
        if searchText.isEmpty {
            return Array(posts)
        }

        return posts.filter {
            $0.title.localizedCaseInsensitiveContains(searchText) ||
            $0.content.localizedCaseInsensitiveContains(searchText)
        }
    }

    var body: some View {
        List(filteredPosts) { post in
            PostRow(post: post)
        }
        .searchable(text: $searchText)
        .toolbar {
            ToolbarItem(placement: .navigationBarTrailing) {
                Toggle("Published", isOn: $showPublishedOnly)
            }
        }
        .onChange(of: showPublishedOnly) { _ in
            updateFilter()
        }
    }

    private func updateFilter() {
        if showPublishedOnly {
            $posts.filter = NSPredicate(format: "isPublished == true")
        } else {
            $posts.filter = nil
        }
    }
}
```

### 12.9.4 セクション分け

```swift
struct GroupedPostListView: View {
    @ObservedResults(
        Post.self,
        sortDescriptor: SortDescriptor(keyPath: "createdAt", ascending: false)
    ) var posts

    var groupedPosts: [String: [Post]] {
        Dictionary(grouping: posts) { post in
            let formatter = DateFormatter()
            formatter.dateFormat = "MMMM yyyy"
            return formatter.string(from: post.createdAt)
        }
    }

    var sortedKeys: [String] {
        groupedPosts.keys.sorted { key1, key2 in
            let formatter = DateFormatter()
            formatter.dateFormat = "MMMM yyyy"
            guard let date1 = formatter.date(from: key1),
                  let date2 = formatter.date(from: key2) else {
                return false
            }
            return date1 > date2
        }
    }

    var body: some View {
        List {
            ForEach(sortedKeys, id: \.self) { key in
                Section(key) {
                    ForEach(groupedPosts[key] ?? []) { post in
                        PostRow(post: post)
                    }
                }
            }
        }
        .navigationTitle("Posts")
    }
}
```

## 12.10 パフォーマンス最適化

### 12.10.1 クエリの最適化

```swift
class OptimizedQueryManager {
    private let realm: Realm

    init(realm: Realm = try! Realm()) {
        self.realm = realm
    }

    // ❌ 非効率なクエリ
    func inefficientQuery() -> [User] {
        let allUsers = Array(realm.objects(User.self))
        return allUsers.filter { $0.isActive && $0.posts.count > 10 }
    }

    // ✅ 効率的なクエリ
    func efficientQuery() -> Results<User> {
        return realm.objects(User.self)
            .filter("isActive == true AND posts.@count > 10")
            .sorted(byKeyPath: "username", ascending: true)
    }

    // インデックスを活用
    func indexedQuery(email: String) -> User? {
        // emailプロパティにインデックスが設定されていると高速
        return realm.objects(User.self)
            .filter("email == %@", email)
            .first
    }

    // 複合クエリの最適化
    func complexQuery(
        isActive: Bool,
        minPostCount: Int,
        createdAfter: Date
    ) -> Results<User> {
        return realm.objects(User.self)
            .filter("isActive == %@ AND posts.@count >= %d AND createdAt >= %@",
                   isActive, minPostCount, createdAfter as NSDate)
            .sorted(byKeyPath: "createdAt", ascending: false)
    }
}
```

### 12.10.2 メモリ管理

```swift
class MemoryOptimizationManager {
    func processLargeDataset() {
        let realm = try! Realm()

        // ❌ 全データを配列に変換（メモリを大量消費）
        func inefficient() {
            let allUsers = Array(realm.objects(User.self))
            for user in allUsers {
                process(user)
            }
        }

        // ✅ Resultsのまま処理（効率的）
        func efficient() {
            let users = realm.objects(User.self)
            for user in users {
                process(user)
            }
        }
    }

    private func process(_ user: User) {
        // 処理
    }

    // ページネーション
    func fetchWithPagination(page: Int, pageSize: Int) -> [User] {
        let realm = try! Realm()
        let results = realm.objects(User.self)
            .sorted(byKeyPath: "createdAt", ascending: false)

        let start = page * pageSize
        let end = start + pageSize

        guard start < results.count else { return [] }

        let endIndex = min(end, results.count)
        return Array(results[start..<endIndex])
    }
}
```

### 12.10.3 書き込みの最適化

```swift
class WriteOptimizationManager {
    private let realm: Realm

    init(realm: Realm = try! Realm()) {
        self.realm = realm
    }

    // ❌ 非効率な書き込み（複数のトランザクション）
    func inefficientWrites(users: [CreateUserDTO]) throws {
        for dto in users {
            try realm.write {
                let user = User(username: dto.username, email: dto.email)
                realm.add(user)
            }
        }
    }

    // ✅ 効率的な書き込み（単一のトランザクション）
    func efficientWrites(users: [CreateUserDTO]) throws {
        try realm.write {
            for dto in users {
                let user = User(username: dto.username, email: dto.email)
                realm.add(user)
            }
        }
    }

    // バッチ書き込み
    func batchWrite(users: [CreateUserDTO], batchSize: Int = 1000) throws {
        for batch in users.chunked(into: batchSize) {
            try realm.write {
                for dto in batch {
                    let user = User(username: dto.username, email: dto.email)
                    realm.add(user)
                }
            }
        }
    }
}

// Array extension for chunking
extension Array {
    func chunked(into size: Int) -> [[Element]] {
        stride(from: 0, to: count, by: size).map {
            Array(self[$0..<Swift.min($0 + size, count)])
        }
    }
}
```

## 12.11 テスト

### 12.11.1 ユニットテスト

```swift
import XCTest
import RealmSwift

class RealmUserRepositoryTests: XCTestCase {
    var realm: Realm!
    var repository: UserRepository!

    override func setUp() {
        super.setUp()

        // In-memory Realmを使用
        let config = Realm.Configuration(inMemoryIdentifier: "test")
        realm = try! Realm(configuration: config)
        repository = UserRepository(realm: realm)
    }

    override func tearDown() {
        // クリーンアップ
        try! realm.write {
            realm.deleteAll()
        }

        realm = nil
        repository = nil

        super.tearDown()
    }

    func testCreateUser() throws {
        // Given
        let username = "testuser"
        let email = "test@example.com"

        // When
        let user = try repository.createUser(username: username, email: email)

        // Then
        XCTAssertEqual(user.username, username)
        XCTAssertEqual(user.email, email)
        XCTAssertTrue(user.isActive)
        XCTAssertEqual(realm.objects(User.self).count, 1)
    }

    func testFetchUser() throws {
        // Given
        let user = try repository.createUser(username: "testuser", email: "test@example.com")

        // When
        let fetchedUser = repository.fetchUser(byID: user.id)

        // Then
        XCTAssertNotNil(fetchedUser)
        XCTAssertEqual(fetchedUser?.username, user.username)
    }

    func testUpdateUser() throws {
        // Given
        let user = try repository.createUser(username: "oldname", email: "old@example.com")
        let newUsername = "newname"
        let newEmail = "new@example.com"

        // When
        try repository.updateUser(user, username: newUsername, email: newEmail)

        // Then
        XCTAssertEqual(user.username, newUsername)
        XCTAssertEqual(user.email, newEmail)
    }

    func testDeleteUser() throws {
        // Given
        let user = try repository.createUser(username: "testuser", email: "test@example.com")
        XCTAssertEqual(realm.objects(User.self).count, 1)

        // When
        try repository.deleteUser(user)

        // Then
        XCTAssertEqual(realm.objects(User.self).count, 0)
    }

    func testSearchUsers() throws {
        // Given
        try repository.createUser(username: "alice", email: "alice@example.com")
        try repository.createUser(username: "bob", email: "bob@example.com")
        try repository.createUser(username: "charlie", email: "charlie@example.com")

        // When
        let results = repository.searchUsers(query: "ali")

        // Then
        XCTAssertEqual(results.count, 1)
        XCTAssertEqual(results.first?.username, "alice")
    }
}
```

### 12.11.2 モックとスタブ

```swift
protocol UserRepositoryProtocol {
    func createUser(username: String, email: String) throws -> User
    func fetchUser(byID id: UUID) -> User?
    func updateUser(_ user: User, username: String?, email: String?) throws
    func deleteUser(_ user: User) throws
}

class MockUserRepository: UserRepositoryProtocol {
    var users: [UUID: User] = [:]
    var createUserCalled = false
    var fetchUserCalled = false

    func createUser(username: String, email: String) throws -> User {
        createUserCalled = true
        let user = User(username: username, email: email)
        users[user.id] = user
        return user
    }

    func fetchUser(byID id: UUID) -> User? {
        fetchUserCalled = true
        return users[id]
    }

    func updateUser(_ user: User, username: String?, email: String?) throws {
        if let username = username {
            user.username = username
        }
        if let email = email {
            user.email = email
        }
    }

    func deleteUser(_ user: User) throws {
        users.removeValue(forKey: user.id)
    }
}

// テストでの使用
class ViewModelTests: XCTestCase {
    func testViewModel() {
        let mockRepository = MockUserRepository()
        let viewModel = UserListViewModel(repository: mockRepository)

        // テスト実行
        viewModel.createUser(username: "test", email: "test@example.com")

        XCTAssertTrue(mockRepository.createUserCalled)
        XCTAssertEqual(mockRepository.users.count, 1)
    }
}
```

## 12.12 ベストプラクティス

### 12.12.1 設計原則

```swift
// ✅ 良い例：Repositoryパターン
class GoodUserService {
    private let repository: UserRepository

    init(repository: UserRepository) {
        self.repository = repository
    }

    func registerUser(username: String, email: String) async throws -> User {
        // バリデーション
        guard !username.isEmpty, !email.isEmpty else {
            throw ValidationError.invalidInput
        }

        // 重複チェック
        if repository.fetchUser(byEmail: email) != nil {
            throw ValidationError.emailAlreadyExists
        }

        // 作成
        return try repository.createUser(username: username, email: email)
    }
}

// ❌ 悪い例：直接Realmを操作
class BadUserService {
    func registerUser(username: String, email: String) throws {
        let realm = try Realm()
        let user = User(username: username, email: email)
        try realm.write {
            realm.add(user)
        }
    }
}

enum ValidationError: Error {
    case invalidInput
    case emailAlreadyExists
}
```

### 12.12.2 エラーハンドリング

```swift
enum RealmError: Error, LocalizedError {
    case realmNotAvailable
    case writeTransactionFailed(Error)
    case objectNotFound
    case invalidData

    var errorDescription: String? {
        switch self {
        case .realmNotAvailable:
            return "Realm database is not available"
        case .writeTransactionFailed(let error):
            return "Failed to write to database: \(error.localizedDescription)"
        case .objectNotFound:
            return "Requested object not found"
        case .invalidData:
            return "Invalid data provided"
        }
    }
}

class RobustRepository {
    private let realm: Realm

    init() throws {
        do {
            self.realm = try Realm()
        } catch {
            throw RealmError.realmNotAvailable
        }
    }

    func createUser(username: String, email: String) -> Result<User, RealmError> {
        do {
            let user = User(username: username, email: email)
            try realm.write {
                realm.add(user)
            }
            return .success(user)
        } catch {
            return .failure(.writeTransactionFailed(error))
        }
    }
}
```

## 12.13 まとめ

この章では、Realmを使った効率的なデータ永続化について学びました。

### 重要なポイント

1. **シンプルなAPI**
   - 直感的なモデル定義
   - 簡潔なCRUD操作

2. **リアルタイム更新**
   - Realm Observationの活用
   - SwiftUIとのシームレスな統合

3. **パフォーマンス**
   - 効率的なクエリ
   - メモリ管理
   - バッチ処理

4. **スレッドセーフ**
   - ThreadSafeReferenceの使用
   - バックグラウンド処理

5. **マイグレーション**
   - スキーマバージョン管理
   - データ変換

次のChapterでは、これらの永続化技術を統合したアーキテクチャパターンについて学びます。
