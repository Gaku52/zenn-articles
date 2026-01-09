---
title: "Chapter 11: Core Data完全マスター"
---

# Chapter 11: Core Data完全マスター

## 11.1 概要

Core DataはAppleが提供する、オブジェクトグラフ管理とデータ永続化のためのフレームワークです。単なるデータベースラッパーではなく、オブジェクト指向のデータモデル管理、メモリ管理、クエリの最適化、変更の追跡など、多くの高度な機能を提供します。

### 本章で学べること

- Core Dataの基本概念とアーキテクチャ
- データモデルの設計とベストプラクティス
- CRUD操作の実装
- NSFetchedResultsControllerを使ったデータ管理
- リレーションシップとカスケード削除
- マイグレーションとバージョン管理
- パフォーマンス最適化
- SwiftUIとの統合

### Core Dataのアーキテクチャ

```swift
/*
Core Data Stack:

NSManagedObjectModel
    ↓
NSPersistentStoreCoordinator
    ↓
NSManagedObjectContext (Main)
NSManagedObjectContext (Background)
    ↓
NSManagedObject (Entity instances)
*/
```

## 11.2 Core Data Stack のセットアップ

### 11.2.1 基本的なCore Data Stack

```swift
import CoreData

class CoreDataManager {
    static let shared = CoreDataManager()

    private init() {}

    // MARK: - Core Data Stack
    lazy var persistentContainer: NSPersistentContainer = {
        let container = NSPersistentContainer(name: "AppDataModel")

        container.loadPersistentStores { storeDescription, error in
            if let error = error as NSError? {
                fatalError("Unresolved error \(error), \(error.userInfo)")
            }
        }

        // Merge Policy設定
        container.viewContext.automaticallyMergesChangesFromParent = true
        container.viewContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy

        return container
    }()

    // Main context (UI thread)
    var viewContext: NSManagedObjectContext {
        return persistentContainer.viewContext
    }

    // Background context for heavy operations
    func newBackgroundContext() -> NSManagedObjectContext {
        return persistentContainer.newBackgroundContext()
    }

    // MARK: - Save Context
    func save(context: NSManagedObjectContext? = nil) {
        let context = context ?? viewContext

        guard context.hasChanges else { return }

        do {
            try context.save()
        } catch {
            let nsError = error as NSError
            fatalError("Unresolved error \(nsError), \(nsError.userInfo)")
        }
    }

    // MARK: - Background Task
    func performBackgroundTask<T>(_ block: @escaping (NSManagedObjectContext) throws -> T) async throws -> T {
        try await withCheckedThrowingContinuation { continuation in
            persistentContainer.performBackgroundTask { context in
                do {
                    let result = try block(context)
                    if context.hasChanges {
                        try context.save()
                    }
                    continuation.resume(returning: result)
                } catch {
                    continuation.resume(throwing: error)
                }
            }
        }
    }
}
```

### 11.2.2 高度なセットアップ

```swift
class AdvancedCoreDataManager {
    static let shared = AdvancedCoreDataManager()

    private init() {
        setupNotifications()
    }

    // MARK: - Core Data Stack
    lazy var persistentContainer: NSPersistentContainer = {
        let container = NSPersistentContainer(name: "AppDataModel")

        // Store Description のカスタマイズ
        guard let description = container.persistentStoreDescriptions.first else {
            fatalError("Failed to retrieve a persistent store description.")
        }

        // マイグレーション設定
        description.shouldMigrateStoreAutomatically = true
        description.shouldInferMappingModelAutomatically = true

        // Write-Ahead Logging有効化（パフォーマンス向上）
        description.setOption(true as NSNumber, forKey: NSSQLitePragmasOption)

        // iCloud同期（必要な場合）
        // description.cloudKitContainerOptions = NSPersistentCloudKitContainerOptions(
        //     containerIdentifier: "iCloud.com.example.app"
        // )

        container.loadPersistentStores { storeDescription, error in
            if let error = error as NSError? {
                fatalError("Unresolved error \(error), \(error.userInfo)")
            }

            print("Successfully loaded persistent store: \(storeDescription)")
        }

        // View Context設定
        container.viewContext.automaticallyMergesChangesFromParent = true
        container.viewContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
        container.viewContext.undoManager = nil
        container.viewContext.shouldDeleteInaccessibleFaults = true

        return container
    }()

    var viewContext: NSManagedObjectContext {
        return persistentContainer.viewContext
    }

    // MARK: - Context Creation
    func newBackgroundContext() -> NSManagedObjectContext {
        let context = persistentContainer.newBackgroundContext()
        context.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
        context.undoManager = nil
        return context
    }

    func newPrivateContext() -> NSManagedObjectContext {
        let context = NSManagedObjectContext(concurrencyType: .privateQueueConcurrencyType)
        context.parent = viewContext
        context.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
        context.undoManager = nil
        return context
    }

    // MARK: - Notifications
    private func setupNotifications() {
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(contextDidSave),
            name: .NSManagedObjectContextDidSave,
            object: nil
        )
    }

    @objc private func contextDidSave(_ notification: Notification) {
        guard let context = notification.object as? NSManagedObjectContext else { return }

        // Main contextの場合はスキップ
        if context == viewContext {
            return
        }

        // Background contextの変更をview contextにマージ
        viewContext.perform {
            self.viewContext.mergeChanges(fromContextDidSave: notification)
        }
    }

    // MARK: - Save Operations
    func save(context: NSManagedObjectContext? = nil) throws {
        let context = context ?? viewContext

        guard context.hasChanges else { return }

        try context.save()
    }

    func saveViewContext() throws {
        try save(context: viewContext)
    }

    // MARK: - Async Operations
    func performBackgroundTask<T>(_ block: @escaping (NSManagedObjectContext) throws -> T) async throws -> T {
        try await withCheckedThrowingContinuation { continuation in
            persistentContainer.performBackgroundTask { context in
                do {
                    let result = try block(context)
                    if context.hasChanges {
                        try context.save()
                    }
                    continuation.resume(returning: result)
                } catch {
                    continuation.resume(throwing: error)
                }
            }
        }
    }

    // MARK: - Reset
    func resetPersistentStore() throws {
        let coordinator = persistentContainer.persistentStoreCoordinator

        for store in coordinator.persistentStores {
            if let storeURL = store.url {
                try coordinator.destroyPersistentStore(
                    at: storeURL,
                    ofType: NSSQLiteStoreType,
                    options: nil
                )
            }
        }

        // 再読み込み
        persistentContainer.loadPersistentStores { _, error in
            if let error = error {
                fatalError("Failed to reload persistent store: \(error)")
            }
        }
    }
}
```

## 11.3 データモデルの設計

### 11.3.1 エンティティの定義

```swift
// User Entity
@objc(User)
public class User: NSManagedObject {
    @NSManaged public var id: UUID
    @NSManaged public var username: String
    @NSManaged public var email: String
    @NSManaged public var firstName: String?
    @NSManaged public var lastName: String?
    @NSManaged public var avatarURL: URL?
    @NSManaged public var bio: String?
    @NSManaged public var createdAt: Date
    @NSManaged public var updatedAt: Date
    @NSManaged public var isActive: Bool

    // Relationships
    @NSManaged public var posts: NSSet?
    @NSManaged public var comments: NSSet?
    @NSManaged public var followers: NSSet?
    @NSManaged public var following: NSSet?
}

// Post Entity
@objc(Post)
public class Post: NSManagedObject {
    @NSManaged public var id: UUID
    @NSManaged public var title: String
    @NSManaged public var content: String
    @NSManaged public var imageURL: URL?
    @NSManaged public var viewCount: Int64
    @NSManaged public var likeCount: Int64
    @NSManaged public var createdAt: Date
    @NSManaged public var updatedAt: Date
    @NSManaged public var isPublished: Bool

    // Relationships
    @NSManaged public var author: User?
    @NSManaged public var comments: NSSet?
    @NSManaged public var tags: NSSet?
}

// Comment Entity
@objc(Comment)
public class Comment: NSManagedObject {
    @NSManaged public var id: UUID
    @NSManaged public var content: String
    @NSManaged public var createdAt: Date
    @NSManaged public var updatedAt: Date

    // Relationships
    @NSManaged public var author: User?
    @NSManaged public var post: Post?
    @NSManaged public var parentComment: Comment?
    @NSManaged public var replies: NSSet?
}

// Tag Entity
@objc(Tag)
public class Tag: NSManagedObject {
    @NSManaged public var id: UUID
    @NSManaged public var name: String
    @NSManaged public var createdAt: Date

    // Relationships
    @NSManaged public var posts: NSSet?
}
```

### 11.3.2 エンティティ拡張

```swift
// MARK: - User Extensions
extension User {
    @nonobjc public class func fetchRequest() -> NSFetchRequest<User> {
        return NSFetchRequest<User>(entityName: "User")
    }

    // Computed Properties
    var fullName: String {
        if let firstName = firstName, let lastName = lastName {
            return "\(firstName) \(lastName)"
        }
        return username
    }

    var postsArray: [Post] {
        let set = posts as? Set<Post> ?? []
        return set.sorted { $0.createdAt > $1.createdAt }
    }

    var commentsArray: [Comment] {
        let set = comments as? Set<Comment> ?? []
        return set.sorted { $0.createdAt > $1.createdAt }
    }

    var followersArray: [User] {
        let set = followers as? Set<User> ?? []
        return set.sorted { $0.username < $1.username }
    }

    var followingArray: [User] {
        let set = following as? Set<User> ?? []
        return set.sorted { $0.username < $1.username }
    }

    // MARK: - Relationships Management
    func addPost(_ post: Post) {
        let posts = mutableSetValue(forKey: "posts")
        posts.add(post)
    }

    func removePost(_ post: Post) {
        let posts = mutableSetValue(forKey: "posts")
        posts.remove(post)
    }

    func follow(_ user: User) {
        let following = mutableSetValue(forKey: "following")
        following.add(user)
    }

    func unfollow(_ user: User) {
        let following = mutableSetValue(forKey: "following")
        following.remove(user)
    }

    func isFollowing(_ user: User) -> Bool {
        return followingArray.contains(user)
    }
}

// MARK: - Post Extensions
extension Post {
    @nonobjc public class func fetchRequest() -> NSFetchRequest<Post> {
        return NSFetchRequest<Post>(entityName: "Post")
    }

    var commentsArray: [Comment] {
        let set = comments as? Set<Comment> ?? []
        return set.sorted { $0.createdAt > $1.createdAt }
    }

    var tagsArray: [Tag] {
        let set = tags as? Set<Tag> ?? []
        return set.sorted { $0.name < $1.name }
    }

    func addComment(_ comment: Comment) {
        let comments = mutableSetValue(forKey: "comments")
        comments.add(comment)
    }

    func addTag(_ tag: Tag) {
        let tags = mutableSetValue(forKey: "tags")
        tags.add(tag)
    }

    func removeTag(_ tag: Tag) {
        let tags = mutableSetValue(forKey: "tags")
        tags.remove(tag)
    }

    func incrementViewCount() {
        viewCount += 1
    }

    func incrementLikeCount() {
        likeCount += 1
    }

    func decrementLikeCount() {
        likeCount = max(0, likeCount - 1)
    }
}

// MARK: - Comment Extensions
extension Comment {
    @nonobjc public class func fetchRequest() -> NSFetchRequest<Comment> {
        return NSFetchRequest<Comment>(entityName: "Comment")
    }

    var repliesArray: [Comment] {
        let set = replies as? Set<Comment> ?? []
        return set.sorted { $0.createdAt < $1.createdAt }
    }

    func addReply(_ comment: Comment) {
        let replies = mutableSetValue(forKey: "replies")
        replies.add(comment)
        comment.parentComment = self
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

// MARK: - Tag Extensions
extension Tag {
    @nonobjc public class func fetchRequest() -> NSFetchRequest<Tag> {
        return NSFetchRequest<Tag>(entityName: "Tag")
    }

    var postsArray: [Post] {
        let set = posts as? Set<Post> ?? []
        return set.sorted { $0.createdAt > $1.createdAt }
    }

    var postCount: Int {
        return posts?.count ?? 0
    }
}

// MARK: - Identifiable Conformance
extension User: Identifiable {}
extension Post: Identifiable {}
extension Comment: Identifiable {}
extension Tag: Identifiable {}
```

## 11.4 CRUD操作

### 11.4.1 Create - データの作成

```swift
class UserRepository {
    private let coreData = CoreDataManager.shared

    // MARK: - Create
    func createUser(
        username: String,
        email: String,
        firstName: String? = nil,
        lastName: String? = nil
    ) async throws -> User {
        try await coreData.performBackgroundTask { context in
            let user = User(context: context)
            user.id = UUID()
            user.username = username
            user.email = email
            user.firstName = firstName
            user.lastName = lastName
            user.createdAt = Date()
            user.updatedAt = Date()
            user.isActive = true

            return user
        }
    }

    func createUser(from dto: CreateUserDTO) async throws -> User {
        try await createUser(
            username: dto.username,
            email: dto.email,
            firstName: dto.firstName,
            lastName: dto.lastName
        )
    }
}

class PostRepository {
    private let coreData = CoreDataManager.shared

    func createPost(
        title: String,
        content: String,
        author: User,
        tags: [Tag] = []
    ) async throws -> Post {
        try await coreData.performBackgroundTask { context in
            // Authorを現在のcontextに取得
            guard let authorInContext = try context.existingObject(with: author.objectID) as? User else {
                throw CoreDataError.objectNotFound
            }

            let post = Post(context: context)
            post.id = UUID()
            post.title = title
            post.content = content
            post.author = authorInContext
            post.viewCount = 0
            post.likeCount = 0
            post.createdAt = Date()
            post.updatedAt = Date()
            post.isPublished = true

            // Tagsを追加
            for tag in tags {
                if let tagInContext = try? context.existingObject(with: tag.objectID) as? Tag {
                    post.addTag(tagInContext)
                }
            }

            return post
        }
    }
}

// DTOs
struct CreateUserDTO {
    let username: String
    let email: String
    let firstName: String?
    let lastName: String?
}

struct CreatePostDTO {
    let title: String
    let content: String
    let authorID: UUID
    let tagNames: [String]
}

enum CoreDataError: Error {
    case objectNotFound
    case saveFailed
    case deleteFailed
    case invalidData
}
```

### 11.4.2 Read - データの取得

```swift
extension UserRepository {
    // MARK: - Fetch Single
    func fetchUser(byID id: UUID) async throws -> User? {
        try await coreData.performBackgroundTask { context in
            let request = User.fetchRequest()
            request.predicate = NSPredicate(format: "id == %@", id as CVarArg)
            request.fetchLimit = 1

            return try context.fetch(request).first
        }
    }

    func fetchUser(byUsername username: String) async throws -> User? {
        try await coreData.performBackgroundTask { context in
            let request = User.fetchRequest()
            request.predicate = NSPredicate(format: "username == %@", username)
            request.fetchLimit = 1

            return try context.fetch(request).first
        }
    }

    func fetchUser(byEmail email: String) async throws -> User? {
        try await coreData.performBackgroundTask { context in
            let request = User.fetchRequest()
            request.predicate = NSPredicate(format: "email == %@", email)
            request.fetchLimit = 1

            return try context.fetch(request).first
        }
    }

    // MARK: - Fetch Multiple
    func fetchAllUsers() async throws -> [User] {
        try await coreData.performBackgroundTask { context in
            let request = User.fetchRequest()
            request.sortDescriptors = [
                NSSortDescriptor(key: "createdAt", ascending: false)
            ]

            return try context.fetch(request)
        }
    }

    func fetchActiveUsers() async throws -> [User] {
        try await coreData.performBackgroundTask { context in
            let request = User.fetchRequest()
            request.predicate = NSPredicate(format: "isActive == true")
            request.sortDescriptors = [
                NSSortDescriptor(key: "username", ascending: true)
            ]

            return try context.fetch(request)
        }
    }

    // MARK: - Search
    func searchUsers(query: String) async throws -> [User] {
        try await coreData.performBackgroundTask { context in
            let request = User.fetchRequest()

            // 複合検索（username, firstName, lastName, email）
            let predicates = [
                NSPredicate(format: "username CONTAINS[cd] %@", query),
                NSPredicate(format: "firstName CONTAINS[cd] %@", query),
                NSPredicate(format: "lastName CONTAINS[cd] %@", query),
                NSPredicate(format: "email CONTAINS[cd] %@", query)
            ]

            request.predicate = NSCompoundPredicate(orPredicateWithSubpredicates: predicates)
            request.sortDescriptors = [
                NSSortDescriptor(key: "username", ascending: true)
            ]

            return try context.fetch(request)
        }
    }

    // MARK: - Pagination
    func fetchUsers(offset: Int, limit: Int) async throws -> [User] {
        try await coreData.performBackgroundTask { context in
            let request = User.fetchRequest()
            request.fetchOffset = offset
            request.fetchLimit = limit
            request.sortDescriptors = [
                NSSortDescriptor(key: "createdAt", ascending: false)
            ]

            return try context.fetch(request)
        }
    }

    // MARK: - Count
    func countUsers() async throws -> Int {
        try await coreData.performBackgroundTask { context in
            let request = User.fetchRequest()
            return try context.count(for: request)
        }
    }

    func countActiveUsers() async throws -> Int {
        try await coreData.performBackgroundTask { context in
            let request = User.fetchRequest()
            request.predicate = NSPredicate(format: "isActive == true")
            return try context.count(for: request)
        }
    }
}

extension PostRepository {
    // MARK: - Fetch Posts
    func fetchPost(byID id: UUID) async throws -> Post? {
        try await coreData.performBackgroundTask { context in
            let request = Post.fetchRequest()
            request.predicate = NSPredicate(format: "id == %@", id as CVarArg)
            request.fetchLimit = 1

            return try context.fetch(request).first
        }
    }

    func fetchAllPosts() async throws -> [Post] {
        try await coreData.performBackgroundTask { context in
            let request = Post.fetchRequest()
            request.sortDescriptors = [
                NSSortDescriptor(key: "createdAt", ascending: false)
            ]

            return try context.fetch(request)
        }
    }

    func fetchPublishedPosts() async throws -> [Post] {
        try await coreData.performBackgroundTask { context in
            let request = Post.fetchRequest()
            request.predicate = NSPredicate(format: "isPublished == true")
            request.sortDescriptors = [
                NSSortDescriptor(key: "createdAt", ascending: false)
            ]

            return try context.fetch(request)
        }
    }

    func fetchPosts(byAuthor author: User) async throws -> [Post] {
        try await coreData.performBackgroundTask { context in
            let request = Post.fetchRequest()
            request.predicate = NSPredicate(format: "author == %@", author)
            request.sortDescriptors = [
                NSSortDescriptor(key: "createdAt", ascending: false)
            ]

            return try context.fetch(request)
        }
    }

    func fetchPosts(withTag tag: Tag) async throws -> [Post] {
        try await coreData.performBackgroundTask { context in
            let request = Post.fetchRequest()
            request.predicate = NSPredicate(format: "%@ IN tags", tag)
            request.sortDescriptors = [
                NSSortDescriptor(key: "createdAt", ascending: false)
            ]

            return try context.fetch(request)
        }
    }

    // MARK: - Advanced Queries
    func fetchPopularPosts(minLikes: Int64, limit: Int = 10) async throws -> [Post] {
        try await coreData.performBackgroundTask { context in
            let request = Post.fetchRequest()
            request.predicate = NSPredicate(format: "likeCount >= %lld", minLikes)
            request.sortDescriptors = [
                NSSortDescriptor(key: "likeCount", ascending: false),
                NSSortDescriptor(key: "createdAt", ascending: false)
            ]
            request.fetchLimit = limit

            return try context.fetch(request)
        }
    }

    func fetchRecentPosts(days: Int, limit: Int = 20) async throws -> [Post] {
        try await coreData.performBackgroundTask { context in
            let startDate = Calendar.current.date(byAdding: .day, value: -days, to: Date())!

            let request = Post.fetchRequest()
            request.predicate = NSPredicate(format: "createdAt >= %@", startDate as NSDate)
            request.sortDescriptors = [
                NSSortDescriptor(key: "createdAt", ascending: false)
            ]
            request.fetchLimit = limit

            return try context.fetch(request)
        }
    }

    // MARK: - Search
    func searchPosts(query: String) async throws -> [Post] {
        try await coreData.performBackgroundTask { context in
            let request = Post.fetchRequest()

            let predicates = [
                NSPredicate(format: "title CONTAINS[cd] %@", query),
                NSPredicate(format: "content CONTAINS[cd] %@", query)
            ]

            request.predicate = NSCompoundPredicate(orPredicateWithSubpredicates: predicates)
            request.sortDescriptors = [
                NSSortDescriptor(key: "createdAt", ascending: false)
            ]

            return try context.fetch(request)
        }
    }
}
```

### 11.4.3 Update - データの更新

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
        avatarURL: URL? = nil
    ) async throws {
        try await coreData.performBackgroundTask { context in
            guard let userInContext = try context.existingObject(with: user.objectID) as? User else {
                throw CoreDataError.objectNotFound
            }

            if let username = username {
                userInContext.username = username
            }
            if let email = email {
                userInContext.email = email
            }
            if let firstName = firstName {
                userInContext.firstName = firstName
            }
            if let lastName = lastName {
                userInContext.lastName = lastName
            }
            if let bio = bio {
                userInContext.bio = bio
            }
            if let avatarURL = avatarURL {
                userInContext.avatarURL = avatarURL
            }

            userInContext.updatedAt = Date()
        }
    }

    func deactivateUser(_ user: User) async throws {
        try await coreData.performBackgroundTask { context in
            guard let userInContext = try context.existingObject(with: user.objectID) as? User else {
                throw CoreDataError.objectNotFound
            }

            userInContext.isActive = false
            userInContext.updatedAt = Date()
        }
    }

    func reactivateUser(_ user: User) async throws {
        try await coreData.performBackgroundTask { context in
            guard let userInContext = try context.existingObject(with: user.objectID) as? User else {
                throw CoreDataError.objectNotFound
            }

            userInContext.isActive = true
            userInContext.updatedAt = Date()
        }
    }
}

extension PostRepository {
    func updatePost(
        _ post: Post,
        title: String? = nil,
        content: String? = nil,
        imageURL: URL? = nil
    ) async throws {
        try await coreData.performBackgroundTask { context in
            guard let postInContext = try context.existingObject(with: post.objectID) as? Post else {
                throw CoreDataError.objectNotFound
            }

            if let title = title {
                postInContext.title = title
            }
            if let content = content {
                postInContext.content = content
            }
            if let imageURL = imageURL {
                postInContext.imageURL = imageURL
            }

            postInContext.updatedAt = Date()
        }
    }

    func publishPost(_ post: Post) async throws {
        try await coreData.performBackgroundTask { context in
            guard let postInContext = try context.existingObject(with: post.objectID) as? Post else {
                throw CoreDataError.objectNotFound
            }

            postInContext.isPublished = true
            postInContext.updatedAt = Date()
        }
    }

    func unpublishPost(_ post: Post) async throws {
        try await coreData.performBackgroundTask { context in
            guard let postInContext = try context.existingObject(with: post.objectID) as? Post else {
                throw CoreDataError.objectNotFound
            }

            postInContext.isPublished = false
            postInContext.updatedAt = Date()
        }
    }

    func incrementPostViews(_ post: Post) async throws {
        try await coreData.performBackgroundTask { context in
            guard let postInContext = try context.existingObject(with: post.objectID) as? Post else {
                throw CoreDataError.objectNotFound
            }

            postInContext.incrementViewCount()
        }
    }

    func likePost(_ post: Post) async throws {
        try await coreData.performBackgroundTask { context in
            guard let postInContext = try context.existingObject(with: post.objectID) as? Post else {
                throw CoreDataError.objectNotFound
            }

            postInContext.incrementLikeCount()
        }
    }

    func unlikePost(_ post: Post) async throws {
        try await coreData.performBackgroundTask { context in
            guard let postInContext = try context.existingObject(with: post.objectID) as? Post else {
                throw CoreDataError.objectNotFound
            }

            postInContext.decrementLikeCount()
        }
    }
}
```

### 11.4.4 Delete - データの削除

```swift
extension UserRepository {
    // MARK: - Delete
    func deleteUser(_ user: User) async throws {
        try await coreData.performBackgroundTask { context in
            guard let userInContext = try context.existingObject(with: user.objectID) as? User else {
                throw CoreDataError.objectNotFound
            }

            context.delete(userInContext)
        }
    }

    func deleteUsers(_ users: [User]) async throws {
        try await coreData.performBackgroundTask { context in
            for user in users {
                if let userInContext = try? context.existingObject(with: user.objectID) {
                    context.delete(userInContext)
                }
            }
        }
    }

    func deleteAllInactiveUsers() async throws -> Int {
        try await coreData.performBackgroundTask { context in
            let request = User.fetchRequest()
            request.predicate = NSPredicate(format: "isActive == false")

            let users = try context.fetch(request)
            let count = users.count

            for user in users {
                context.delete(user)
            }

            return count
        }
    }
}

extension PostRepository {
    func deletePost(_ post: Post) async throws {
        try await coreData.performBackgroundTask { context in
            guard let postInContext = try context.existingObject(with: post.objectID) as? Post else {
                throw CoreDataError.objectNotFound
            }

            context.delete(postInContext)
        }
    }

    func deletePosts(_ posts: [Post]) async throws {
        try await coreData.performBackgroundTask { context in
            for post in posts {
                if let postInContext = try? context.existingObject(with: post.objectID) {
                    context.delete(postInContext)
                }
            }
        }
    }

    func deleteAllUnpublishedPosts() async throws -> Int {
        try await coreData.performBackgroundTask { context in
            let request = Post.fetchRequest()
            request.predicate = NSPredicate(format: "isPublished == false")

            let posts = try context.fetch(request)
            let count = posts.count

            for post in posts {
                context.delete(post)
            }

            return count
        }
    }
}
```

## 11.5 NSFetchedResultsController

### 11.5.1 基本的な実装

```swift
import Combine

class PostListViewModel: NSObject, ObservableObject {
    @Published var posts: [Post] = []
    @Published var isLoading = false
    @Published var error: Error?

    private var fetchedResultsController: NSFetchedResultsController<Post>!
    private let context = CoreDataManager.shared.viewContext

    override init() {
        super.init()
        setupFetchedResultsController()
        fetchPosts()
    }

    private func setupFetchedResultsController() {
        let request = Post.fetchRequest()
        request.sortDescriptors = [
            NSSortDescriptor(key: "createdAt", ascending: false)
        ]
        request.predicate = NSPredicate(format: "isPublished == true")

        fetchedResultsController = NSFetchedResultsController(
            fetchRequest: request,
            managedObjectContext: context,
            sectionNameKeyPath: nil,
            cacheName: "PostListCache"
        )

        fetchedResultsController.delegate = self
    }

    func fetchPosts() {
        isLoading = true

        do {
            try fetchedResultsController.performFetch()
            posts = fetchedResultsController.fetchedObjects ?? []
            isLoading = false
        } catch {
            self.error = error
            isLoading = false
        }
    }

    func refresh() {
        fetchPosts()
    }
}

extension PostListViewModel: NSFetchedResultsControllerDelegate {
    func controllerDidChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        posts = controller.fetchedObjects as? [Post] ?? []
    }

    func controller(
        _ controller: NSFetchedResultsController<NSFetchRequestResult>,
        didChange anObject: Any,
        at indexPath: IndexPath?,
        for type: NSFetchedResultsChangeType,
        newIndexPath: IndexPath?
    ) {
        // UITableView用の細かい変更通知が必要な場合
        switch type {
        case .insert:
            print("Inserted at \(String(describing: newIndexPath))")
        case .delete:
            print("Deleted at \(String(describing: indexPath))")
        case .move:
            print("Moved from \(String(describing: indexPath)) to \(String(describing: newIndexPath))")
        case .update:
            print("Updated at \(String(describing: indexPath))")
        @unknown default:
            break
        }
    }
}
```

### 11.5.2 セクション付きFetchedResultsController

```swift
class GroupedPostListViewModel: NSObject, ObservableObject {
    @Published var sections: [PostSection] = []

    private var fetchedResultsController: NSFetchedResultsController<Post>!
    private let context = CoreDataManager.shared.viewContext

    struct PostSection: Identifiable {
        let id = UUID()
        let name: String
        let posts: [Post]
    }

    override init() {
        super.init()
        setupFetchedResultsController()
        fetchPosts()
    }

    private func setupFetchedResultsController() {
        let request = Post.fetchRequest()
        request.sortDescriptors = [
            NSSortDescriptor(key: "createdAt", ascending: false)
        ]

        fetchedResultsController = NSFetchedResultsController(
            fetchRequest: request,
            managedObjectContext: context,
            sectionNameKeyPath: "sectionIdentifier", // カスタムプロパティ
            cacheName: "GroupedPostListCache"
        )

        fetchedResultsController.delegate = self
    }

    private func fetchPosts() {
        do {
            try fetchedResultsController.performFetch()
            updateSections()
        } catch {
            print("Failed to fetch posts: \(error)")
        }
    }

    private func updateSections() {
        guard let sectionInfos = fetchedResultsController.sections else {
            sections = []
            return
        }

        sections = sectionInfos.map { sectionInfo in
            PostSection(
                name: sectionInfo.name,
                posts: sectionInfo.objects as? [Post] ?? []
            )
        }
    }
}

extension GroupedPostListViewModel: NSFetchedResultsControllerDelegate {
    func controllerDidChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        updateSections()
    }
}

// Post+Extensions.swift に追加
extension Post {
    @objc var sectionIdentifier: String {
        let calendar = Calendar.current
        let components = calendar.dateComponents([.year, .month], from: createdAt)

        if let year = components.year, let month = components.month {
            let formatter = DateFormatter()
            formatter.dateFormat = "MMMM yyyy"
            let date = calendar.date(from: components)!
            return formatter.string(from: date)
        }

        return "Unknown"
    }
}
```

## 11.6 リレーションシップの管理

### 11.6.1 1対多のリレーションシップ

```swift
class PostService {
    private let coreData = CoreDataManager.shared
    private let postRepository = PostRepository()
    private let commentRepository = CommentRepository()

    // Commentを追加
    func addComment(to post: Post, content: String, author: User) async throws -> Comment {
        try await coreData.performBackgroundTask { context in
            guard let postInContext = try context.existingObject(with: post.objectID) as? Post,
                  let authorInContext = try context.existingObject(with: author.objectID) as? User else {
                throw CoreDataError.objectNotFound
            }

            let comment = Comment(context: context)
            comment.id = UUID()
            comment.content = content
            comment.author = authorInContext
            comment.post = postInContext
            comment.createdAt = Date()
            comment.updatedAt = Date()

            // リレーションシップは自動的に設定される
            // postInContext.addComment(comment) // これは不要

            return comment
        }
    }

    // Postのすべてのコメントを取得
    func fetchComments(for post: Post) async throws -> [Comment] {
        try await coreData.performBackgroundTask { context in
            guard let postInContext = try context.existingObject(with: post.objectID) as? Post else {
                throw CoreDataError.objectNotFound
            }

            return postInContext.commentsArray
        }
    }
}
```

### 11.6.2 多対多のリレーションシップ

```swift
class TagService {
    private let coreData = CoreDataManager.shared

    // Tagを作成または取得
    func findOrCreateTag(name: String) async throws -> Tag {
        try await coreData.performBackgroundTask { context in
            // 既存のTagを検索
            let request = Tag.fetchRequest()
            request.predicate = NSPredicate(format: "name ==[cd] %@", name)
            request.fetchLimit = 1

            if let existingTag = try context.fetch(request).first {
                return existingTag
            }

            // 新規作成
            let tag = Tag(context: context)
            tag.id = UUID()
            tag.name = name
            tag.createdAt = Date()

            return tag
        }
    }

    // PostにTagを追加
    func addTag(to post: Post, tagName: String) async throws {
        let tag = try await findOrCreateTag(name: tagName)

        try await coreData.performBackgroundTask { context in
            guard let postInContext = try context.existingObject(with: post.objectID) as? Post,
                  let tagInContext = try context.existingObject(with: tag.objectID) as? Tag else {
                throw CoreDataError.objectNotFound
            }

            postInContext.addTag(tagInContext)
        }
    }

    // PostからTagを削除
    func removeTag(from post: Post, tag: Tag) async throws {
        try await coreData.performBackgroundTask { context in
            guard let postInContext = try context.existingObject(with: post.objectID) as? Post,
                  let tagInContext = try context.existingObject(with: tag.objectID) as? Tag else {
                throw CoreDataError.objectNotFound
            }

            postInContext.removeTag(tagInContext)
        }
    }

    // 特定のTagを持つPostを取得
    func fetchPosts(withTag tag: Tag) async throws -> [Post] {
        try await coreData.performBackgroundTask { context in
            guard let tagInContext = try context.existingObject(with: tag.objectID) as? Tag else {
                throw CoreDataError.objectNotFound
            }

            return tagInContext.postsArray
        }
    }
}
```

### 11.6.3 自己参照リレーションシップ

```swift
class CommentService {
    private let coreData = CoreDataManager.shared

    // 返信を追加
    func addReply(to parentComment: Comment, content: String, author: User) async throws -> Comment {
        try await coreData.performBackgroundTask { context in
            guard let parentInContext = try context.existingObject(with: parentComment.objectID) as? Comment,
                  let authorInContext = try context.existingObject(with: author.objectID) as? User else {
                throw CoreDataError.objectNotFound
            }

            let reply = Comment(context: context)
            reply.id = UUID()
            reply.content = content
            reply.author = authorInContext
            reply.parentComment = parentInContext
            reply.createdAt = Date()
            reply.updatedAt = Date()

            // 親のPostも設定
            reply.post = parentInContext.post

            return reply
        }
    }

    // コメントツリーを取得（階層構造）
    func fetchCommentTree(for post: Post) async throws -> [CommentNode] {
        let comments = try await coreData.performBackgroundTask { context in
            guard let postInContext = try context.existingObject(with: post.objectID) as? Post else {
                throw CoreDataError.objectNotFound
            }

            return postInContext.commentsArray
        }

        // ルートコメント（親がないもの）を抽出
        let rootComments = comments.filter { $0.parentComment == nil }

        return rootComments.map { buildCommentTree(from: $0, allComments: comments) }
    }

    private func buildCommentTree(from comment: Comment, allComments: [Comment]) -> CommentNode {
        let replies = allComments
            .filter { $0.parentComment == comment }
            .map { buildCommentTree(from: $0, allComments: allComments) }

        return CommentNode(comment: comment, replies: replies)
    }
}

struct CommentNode: Identifiable {
    let id = UUID()
    let comment: Comment
    let replies: [CommentNode]

    var depth: Int {
        comment.depth
    }
}
```

## 11.7 バッチ操作

### 11.7.1 バッチ挿入

```swift
class BatchInsertService {
    private let coreData = CoreDataManager.shared

    func batchInsertUsers(_ dtos: [CreateUserDTO]) async throws {
        try await coreData.performBackgroundTask { context in
            for dto in dtos {
                let user = User(context: context)
                user.id = UUID()
                user.username = dto.username
                user.email = dto.email
                user.firstName = dto.firstName
                user.lastName = dto.lastName
                user.createdAt = Date()
                user.updatedAt = Date()
                user.isActive = true
            }

            // 一度だけ保存
            if context.hasChanges {
                try context.save()
            }
        }
    }

    // より効率的なバッチ挿入（iOS 13+）
    func efficientBatchInsert(_ dtos: [CreateUserDTO]) async throws {
        let context = coreData.newBackgroundContext()

        try await context.perform {
            let batchInsert = NSBatchInsertRequest(
                entity: User.entity(),
                objects: dtos.map { dto in
                    [
                        "id": UUID(),
                        "username": dto.username,
                        "email": dto.email,
                        "firstName": dto.firstName as Any,
                        "lastName": dto.lastName as Any,
                        "createdAt": Date(),
                        "updatedAt": Date(),
                        "isActive": true
                    ] as [String: Any]
                }
            )

            try context.execute(batchInsert)
        }
    }
}
```

### 11.7.2 バッチ更新

```swift
extension UserRepository {
    // すべての非アクティブユーザーを削除
    func deactivateOldUsers(olderThan days: Int) async throws -> Int {
        try await coreData.performBackgroundTask { context in
            let cutoffDate = Calendar.current.date(byAdding: .day, value: -days, to: Date())!

            let batchUpdate = NSBatchUpdateRequest(entity: User.entity())
            batchUpdate.predicate = NSPredicate(format: "updatedAt < %@", cutoffDate as NSDate)
            batchUpdate.propertiesToUpdate = [
                "isActive": false,
                "updatedAt": Date()
            ]
            batchUpdate.resultType = .updatedObjectsCountResultType

            let result = try context.execute(batchUpdate) as! NSBatchUpdateResult
            return result.result as? Int ?? 0
        }
    }
}

extension PostRepository {
    // 古いPostを非公開にする
    func unpublishOldPosts(olderThan days: Int) async throws -> Int {
        try await coreData.performBackgroundTask { context in
            let cutoffDate = Calendar.current.date(byAdding: .day, value: -days, to: Date())!

            let batchUpdate = NSBatchUpdateRequest(entity: Post.entity())
            batchUpdate.predicate = NSPredicate(
                format: "createdAt < %@ AND isPublished == true",
                cutoffDate as NSDate
            )
            batchUpdate.propertiesToUpdate = [
                "isPublished": false,
                "updatedAt": Date()
            ]
            batchUpdate.resultType = .updatedObjectsCountResultType

            let result = try context.execute(batchUpdate) as! NSBatchUpdateResult
            return result.result as? Int ?? 0
        }
    }
}
```

### 11.7.3 バッチ削除

```swift
extension UserRepository {
    // 非アクティブユーザーを一括削除
    func batchDeleteInactiveUsers() async throws -> Int {
        try await coreData.performBackgroundTask { context in
            let fetchRequest: NSFetchRequest<NSFetchRequestResult> = User.fetchRequest()
            fetchRequest.predicate = NSPredicate(format: "isActive == false")

            let batchDelete = NSBatchDeleteRequest(fetchRequest: fetchRequest)
            batchDelete.resultType = .resultTypeCount

            let result = try context.execute(batchDelete) as! NSBatchDeleteResult
            return result.result as? Int ?? 0
        }
    }
}

extension PostRepository {
    // 古い非公開Postを削除
    func batchDeleteOldUnpublishedPosts(olderThan days: Int) async throws -> Int {
        try await coreData.performBackgroundTask { context in
            let cutoffDate = Calendar.current.date(byAdding: .day, value: -days, to: Date())!

            let fetchRequest: NSFetchRequest<NSFetchRequestResult> = Post.fetchRequest()
            fetchRequest.predicate = NSPredicate(
                format: "createdAt < %@ AND isPublished == false",
                cutoffDate as NSDate
            )

            let batchDelete = NSBatchDeleteRequest(fetchRequest: fetchRequest)
            batchDelete.resultType = .resultTypeCount

            let result = try context.execute(batchDelete) as! NSBatchDeleteResult
            return result.result as? Int ?? 0
        }
    }
}
```

## 11.8 マイグレーション

### 11.8.1 軽量マイグレーション

```swift
class MigrationManager {
    static let shared = MigrationManager()

    private init() {}

    func setupPersistentStoreWithMigration(container: NSPersistentContainer) {
        guard let storeDescription = container.persistentStoreDescriptions.first else {
            fatalError("No store description found")
        }

        // 軽量マイグレーションを有効化
        storeDescription.shouldMigrateStoreAutomatically = true
        storeDescription.shouldInferMappingModelAutomatically = true

        container.loadPersistentStores { description, error in
            if let error = error {
                fatalError("Failed to load Core Data stack: \(error)")
            }

            print("Successfully loaded persistent store")
        }
    }

    // マイグレーションが必要かチェック
    func needsMigration(at storeURL: URL, for model: NSManagedObjectModel) -> Bool {
        do {
            let metadata = try NSPersistentStoreCoordinator.metadataForPersistentStore(
                ofType: NSSQLiteStoreType,
                at: storeURL
            )

            return !model.isConfiguration(withName: nil, compatibleWithStoreMetadata: metadata)
        } catch {
            print("Error checking migration: \(error)")
            return false
        }
    }
}
```

### 11.8.2 カスタムマイグレーション

```swift
class CustomMigrationManager {
    func migrateStore(
        at sourceURL: URL,
        from sourceModel: NSManagedObjectModel,
        to destinationModel: NSManagedObjectModel
    ) throws {
        // マッピングモデルを取得
        guard let mappingModel = NSMappingModel(
            from: [Bundle.main],
            forSourceModel: sourceModel,
            destinationModel: destinationModel
        ) else {
            throw MigrationError.mappingModelNotFound
        }

        // マイグレーションマネージャーを作成
        let migrationManager = NSMigrationManager(
            sourceModel: sourceModel,
            destinationModel: destinationModel
        )

        // 一時的な宛先URLを作成
        let destinationURL = sourceURL.deletingLastPathComponent()
            .appendingPathComponent("temp_migration.sqlite")

        // マイグレーションを実行
        try migrationManager.migrateStore(
            from: sourceURL,
            sourceType: NSSQLiteStoreType,
            options: nil,
            with: mappingModel,
            toDestinationURL: destinationURL,
            destinationType: NSSQLiteStoreType,
            destinationOptions: nil
        )

        // 古いストアを削除
        try FileManager.default.removeItem(at: sourceURL)

        // 新しいストアを移動
        try FileManager.default.moveItem(at: destinationURL, to: sourceURL)
    }

    enum MigrationError: Error {
        case mappingModelNotFound
        case migrationFailed
    }
}
```

## 11.9 パフォーマンス最適化

### 11.9.1 フェッチリクエストの最適化

```swift
class OptimizedRepository {
    private let coreData = CoreDataManager.shared

    // ❌ 非効率な実装
    func inefficientFetch() async throws -> [Post] {
        try await coreData.performBackgroundTask { context in
            let request = Post.fetchRequest()
            let posts = try context.fetch(request)

            // すべてのリレーションシップがフォールトになっている
            return posts
        }
    }

    // ✅ 効率的な実装（プリフェッチ）
    func efficientFetch() async throws -> [Post] {
        try await coreData.performBackgroundTask { context in
            let request = Post.fetchRequest()

            // リレーションシップをプリフェッチ
            request.relationshipKeyPathsForPrefetching = ["author", "tags", "comments"]

            return try context.fetch(request)
        }
    }

    // フェッチリミットとオフセット
    func fetchWithPagination(page: Int, pageSize: Int = 20) async throws -> [Post] {
        try await coreData.performBackgroundTask { context in
            let request = Post.fetchRequest()
            request.fetchLimit = pageSize
            request.fetchOffset = page * pageSize
            request.sortDescriptors = [
                NSSortDescriptor(key: "createdAt", ascending: false)
            ]

            return try context.fetch(request)
        }
    }

    // バッチサイズの設定
    func fetchWithBatchSize() async throws -> [Post] {
        try await coreData.performBackgroundTask { context in
            let request = Post.fetchRequest()
            request.fetchBatchSize = 20 // 20件ずつフェッチ
            request.sortDescriptors = [
                NSSortDescriptor(key: "createdAt", ascending: false)
            ]

            return try context.fetch(request)
        }
    }

    // 特定のプロパティのみ取得
    func fetchPostTitlesOnly() async throws -> [[String: Any]] {
        try await coreData.performBackgroundTask { context in
            let request = NSFetchRequest<NSDictionary>(entityName: "Post")
            request.resultType = .dictionaryResultType
            request.propertiesToFetch = ["id", "title", "createdAt"]

            let results = try context.fetch(request)
            return results as? [[String: Any]] ?? []
        }
    }

    // カウントのみ取得
    func countPosts() async throws -> Int {
        try await coreData.performBackgroundTask { context in
            let request = Post.fetchRequest()
            request.resultType = .countResultType

            return try context.count(for: request)
        }
    }
}
```

### 11.9.2 インデックスの活用

```swift
/*
データモデルエディタで設定:
1. Entityを選択
2. Data Model Inspectorを開く
3. Indexesセクションで頻繁にクエリされる属性にインデックスを追加

例:
- User.username
- User.email
- Post.createdAt
- Post.isPublished
- Tag.name
*/
```

### 11.9.3 メモリ管理

```swift
class MemoryEfficientRepository {
    private let coreData = CoreDataManager.shared

    // 大量データの処理
    func processLargeDataset() async throws {
        try await coreData.performBackgroundTask { context in
            let request = Post.fetchRequest()
            request.fetchBatchSize = 50

            let posts = try context.fetch(request)

            for (index, post) in posts.enumerated() {
                // 処理
                self.process(post)

                // 定期的にメモリを解放
                if index % 50 == 0 {
                    context.refreshAllObjects()
                }
            }
        }
    }

    private func process(_ post: Post) {
        // 処理内容
    }

    // フォールト化
    func refreshObjects(_ objects: [NSManagedObject]) {
        let context = coreData.viewContext
        for object in objects {
            context.refresh(object, mergeChanges: false)
        }
    }
}
```

## 11.10 SwiftUIとの統合

### 11.10.1 @FetchRequestの使用

```swift
import SwiftUI

struct PostListView: View {
    @FetchRequest(
        entity: Post.entity(),
        sortDescriptors: [
            NSSortDescriptor(keyPath: \Post.createdAt, ascending: false)
        ],
        predicate: NSPredicate(format: "isPublished == true")
    ) var posts: FetchedResults<Post>

    @Environment(\.managedObjectContext) private var viewContext

    var body: some View {
        List(posts) { post in
            PostRow(post: post)
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
        withAnimation {
            let newPost = Post(context: viewContext)
            newPost.id = UUID()
            newPost.title = "New Post"
            newPost.content = "Content"
            newPost.createdAt = Date()
            newPost.updatedAt = Date()
            newPost.isPublished = true

            try? viewContext.save()
        }
    }
}

struct PostRow: View {
    @ObservedObject var post: Post

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text(post.title)
                .font(.headline)

            Text(post.content)
                .font(.body)
                .foregroundColor(.secondary)
                .lineLimit(2)

            HStack {
                Image(systemName: "heart.fill")
                Text("\(post.likeCount)")

                Image(systemName: "eye.fill")
                Text("\(post.viewCount)")
            }
            .font(.caption)
            .foregroundColor(.secondary)
        }
        .padding(.vertical, 4)
    }
}
```

### 11.10.2 動的フィルタリング

```swift
struct FilterablePostListView: View {
    @State private var searchText = ""
    @State private var showPublishedOnly = true

    var body: some View {
        PostListWithFilter(
            searchText: searchText,
            showPublishedOnly: showPublishedOnly
        )
        .searchable(text: $searchText)
        .toolbar {
            ToolbarItem(placement: .navigationBarTrailing) {
                Toggle("Published", isOn: $showPublishedOnly)
            }
        }
    }
}

struct PostListWithFilter: View {
    let searchText: String
    let showPublishedOnly: Bool

    @Environment(\.managedObjectContext) private var viewContext

    var fetchRequest: FetchRequest<Post>

    init(searchText: String, showPublishedOnly: Bool) {
        self.searchText = searchText
        self.showPublishedOnly = showPublishedOnly

        var predicates: [NSPredicate] = []

        if showPublishedOnly {
            predicates.append(NSPredicate(format: "isPublished == true"))
        }

        if !searchText.isEmpty {
            predicates.append(
                NSPredicate(format: "title CONTAINS[cd] %@ OR content CONTAINS[cd] %@",
                           searchText, searchText)
            )
        }

        let compound = NSCompoundPredicate(andPredicateWithSubpredicates: predicates)

        self.fetchRequest = FetchRequest<Post>(
            entity: Post.entity(),
            sortDescriptors: [
                NSSortDescriptor(keyPath: \Post.createdAt, ascending: false)
            ],
            predicate: predicates.isEmpty ? nil : compound
        )
    }

    var body: some View {
        List(fetchRequest.wrappedValue) { post in
            PostRow(post: post)
        }
    }
}
```

## 11.11 まとめ

この章では、Core Dataの基礎から高度な実装まで詳しく学びました。

### 重要なポイント

1. **Core Data Stack**
   - 適切なセットアップと設定
   - バックグラウンドコンテキストの活用

2. **CRUD操作**
   - 型安全なリポジトリパターン
   - async/awaitを使った非同期処理

3. **NSFetchedResultsController**
   - 効率的なデータ表示
   - 自動的な変更の監視

4. **リレーションシップ**
   - 1対多、多対多の実装
   - カスケード削除の設定

5. **パフォーマンス最適化**
   - プリフェッチ
   - バッチ操作
   - インデックスの活用

6. **SwiftUI統合**
   - @FetchRequestの活用
   - 動的フィルタリング

次の章では、より軽量で高速なデータベースであるRealmについて学びます。
