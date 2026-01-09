---
title: "Chapter 17: Realm実装 Part 1 - 基本とオブジェクトマッピング"
---

# Chapter 17: Realm実装 Part 1 - 基本とオブジェクトマッピング

## 17.1 概要

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

## 17.2 セットアップ

### 17.2.1 Realmのインストール

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

### 17.2.2 基本的な設定

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

### 17.2.3 高度な設定

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

## 17.3 モデル定義

### 17.3.1 基本的なモデル

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

### 17.3.2 埋め込みオブジェクト

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

### 17.3.3 カスタム型

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

## 17.4 CRUD操作

### 17.4.1 Create - データの作成

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

### 17.4.2 Read - データの取得

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

### 17.4.3 Update - データの更新

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

### 17.4.4 Delete - データの削除

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

## 17.5 Realm Observation - リアルタイム更新

### 17.5.1 オブジェクトの監視

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

### 17.5.2 ViewModelでの使用

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

### 17.5.3 Combineとの統合

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

## 17.6 リレーションシップの管理

### 17.6.1 1対多のリレーションシップ

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

### 17.6.2 多対多のリレーションシップ

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

### 17.6.3 自己参照リレーションシップ

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

## 17.7 まとめ

この章では、Realmの基本とオブジェクトマッピングについて学びました。

### 重要なポイント

1. **シンプルなセットアップ**
   - Realmの初期化と設定
   - In-memoryテスト用設定

2. **柔軟なモデル定義**
   - 基本的なObject
   - EmbeddedObject
   - Enum、List、Map

3. **完全なCRUD操作**
   - Create、Read、Update、Delete
   - 同期/非同期処理

4. **リアルタイム更新**
   - Realm Observation
   - Combineとの統合

5. **リレーションシップ**
   - 1対多、多対多
   - 自己参照

次のPart 2では、パフォーマンス最適化、マイグレーション、SwiftUI統合、テストなどの高度な機能について学びます。
