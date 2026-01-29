---
title: "Core Data: 構造化データの管理"
---

# Core Data: 構造化データの管理

Core Data は、Apple が提供するオブジェクトグラフとデータ永続化のフレームワークです。この章では、Core Data の基本から実践的な使用方法まで学びます。

## Core Data の特徴

- **オブジェクトグラフ管理**: オブジェクト間の関係を管理
- **遅延読み込み**: 必要なときにデータを読み込む
- **変更追跡**: オブジェクトの変更を自動的に追跡
- **マイグレーション**: データモデルの変更に対応

## Core Data Stack の構築

まず、Core Data Stack を設定します:

```swift
class CoreDataManager {
    static let shared = CoreDataManager()

    lazy var persistentContainer: NSPersistentContainer = {
        let container = NSPersistentContainer(name: "AppModel")

        container.loadPersistentStores { description, error in
            if let error = error {
                fatalError("Core Data ストアの読み込みに失敗しました: \(error)")
            }
        }

        // 自動マージ設定
        container.viewContext.automaticallyMergesChangesFromParent = true
        container.viewContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy

        return container
    }()

    var viewContext: NSManagedObjectContext {
        persistentContainer.viewContext
    }

    func newBackgroundContext() -> NSManagedObjectContext {
        persistentContainer.newBackgroundContext()
    }

    func save(context: NSManagedObjectContext? = nil) {
        let context = context ?? viewContext

        guard context.hasChanges else { return }

        do {
            try context.save()
        } catch {
            let nsError = error as NSError
            fatalError("Core Data の保存に失敗しました: \(nsError), \(nsError.userInfo)")
        }
    }

    func performBackgroundTask<T>(_ block: @escaping (NSManagedObjectContext) throws -> T) async throws -> T {
        try await withCheckedThrowingContinuation { continuation in
            persistentContainer.performBackgroundTask { context in
                do {
                    let result = try block(context)
                    try context.save()
                    continuation.resume(returning: result)
                } catch {
                    continuation.resume(throwing: error)
                }
            }
        }
    }
}
```

## エンティティの定義と拡張

User エンティティを例に、Core Data の使い方を見ていきます:

```swift
// User+CoreDataClass.swift（自動生成）
@objc(User)
public class User: NSManagedObject {
    @NSManaged public var id: UUID
    @NSManaged public var name: String
    @NSManaged public var email: String
    @NSManaged public var createdAt: Date
    @NSManaged public var posts: NSSet?
}

// User+CoreDataProperties.swift（拡張）
extension User {
    @nonobjc public class func fetchRequest() -> NSFetchRequest<User> {
        return NSFetchRequest<User>(entityName: "User")
    }

    // ファクトリメソッド
    static func create(
        in context: NSManagedObjectContext,
        name: String,
        email: String
    ) -> User {
        let user = User(context: context)
        user.id = UUID()
        user.name = name
        user.email = email
        user.createdAt = Date()
        return user
    }

    // クエリメソッド
    static func fetchAll(in context: NSManagedObjectContext) throws -> [User] {
        let request = fetchRequest()
        request.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]
        return try context.fetch(request)
    }

    static func fetch(
        byId id: UUID,
        in context: NSManagedObjectContext
    ) throws -> User? {
        let request = fetchRequest()
        request.predicate = NSPredicate(format: "id == %@", id as CVarArg)
        request.fetchLimit = 1
        return try context.fetch(request).first
    }

    static func search(
        nameContains query: String,
        in context: NSManagedObjectContext
    ) throws -> [User] {
        let request = fetchRequest()
        request.predicate = NSPredicate(format: "name CONTAINS[cd] %@", query)
        request.sortDescriptors = [NSSortDescriptor(key: "name", ascending: true)]
        return try context.fetch(request)
    }
}

// 関連オブジェクトへのアクセス
extension User {
    var postsArray: [Post] {
        let set = posts as? Set<Post> ?? []
        return set.sorted { $0.createdAt > $1.createdAt }
    }

    func addPost(_ post: Post) {
        addToPosts(post)
    }

    func removePost(_ post: Post) {
        removeFromPosts(post)
    }
}
```

## Repository パターンの実装

Core Data へのアクセスを抽象化します:

```swift
protocol UserDataStore {
    func create(name: String, email: String) async throws -> User
    func fetchAll() async throws -> [User]
    func fetch(byId id: UUID) async throws -> User?
    func update(_ user: User, name: String?, email: String?) async throws
    func delete(_ user: User) async throws
}

class CoreDataUserStore: UserDataStore {
    private let coreData = CoreDataManager.shared

    func create(name: String, email: String) async throws -> User {
        try await coreData.performBackgroundTask { context in
            User.create(in: context, name: name, email: email)
        }
    }

    func fetchAll() async throws -> [User] {
        try await coreData.performBackgroundTask { context in
            try User.fetchAll(in: context)
        }
    }

    func fetch(byId id: UUID) async throws -> User? {
        try await coreData.performBackgroundTask { context in
            try User.fetch(byId: id, in: context)
        }
    }

    func update(_ user: User, name: String?, email: String?) async throws {
        try await coreData.performBackgroundTask { context in
            guard let objectID = user.objectID as NSManagedObjectID?,
                  let user = try? context.existingObject(with: objectID) as? User else {
                throw CoreDataError.objectNotFound
            }

            if let name = name {
                user.name = name
            }
            if let email = email {
                user.email = email
            }
        }
    }

    func delete(_ user: User) async throws {
        try await coreData.performBackgroundTask { context in
            guard let objectID = user.objectID as NSManagedObjectID?,
                  let user = try? context.existingObject(with: objectID) else {
                throw CoreDataError.objectNotFound
            }

            context.delete(user)
        }
    }
}

enum CoreDataError: Error {
    case objectNotFound
    case saveFailed
}
```

## NSFetchedResultsController の使用

SwiftUI と組み合わせて、リアルタイムに UI を更新します:

```swift
class UserListViewModel: NSObject, ObservableObject {
    @Published var users: [User] = []

    private lazy var fetchedResultsController: NSFetchedResultsController<User> = {
        let request = User.fetchRequest()
        request.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]

        let controller = NSFetchedResultsController(
            fetchRequest: request,
            managedObjectContext: CoreDataManager.shared.viewContext,
            sectionNameKeyPath: nil,
            cacheName: nil
        )

        controller.delegate = self
        return controller
    }()

    func fetchUsers() {
        do {
            try fetchedResultsController.performFetch()
            users = fetchedResultsController.fetchedObjects ?? []
        } catch {
            print("ユーザーの取得に失敗しました: \(error)")
        }
    }
}

extension UserListViewModel: NSFetchedResultsControllerDelegate {
    func controllerDidChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        users = controller.fetchedObjects as? [User] ?? []
    }
}
```

## 複雑なクエリ

NSPredicate を使った高度なクエリを実装します:

```swift
extension User {
    // 複合条件でのクエリ
    static func searchAdvanced(
        nameContains name: String?,
        emailDomain: String?,
        createdAfter date: Date?,
        in context: NSManagedObjectContext
    ) throws -> [User] {
        var predicates: [NSPredicate] = []

        if let name = name, !name.isEmpty {
            predicates.append(NSPredicate(format: "name CONTAINS[cd] %@", name))
        }

        if let domain = emailDomain, !domain.isEmpty {
            predicates.append(NSPredicate(format: "email ENDSWITH[cd] %@", "@\(domain)"))
        }

        if let date = date {
            predicates.append(NSPredicate(format: "createdAt >= %@", date as NSDate))
        }

        let request = fetchRequest()

        if !predicates.isEmpty {
            request.predicate = NSCompoundPredicate(andPredicateWithSubpredicates: predicates)
        }

        request.sortDescriptors = [NSSortDescriptor(key: "name", ascending: true)]

        return try context.fetch(request)
    }

    // 集計クエリ
    static func count(in context: NSManagedObjectContext) throws -> Int {
        let request = fetchRequest()
        return try context.count(for: request)
    }

    static func countByDomain(in context: NSManagedObjectContext) throws -> [String: Int] {
        let request = fetchRequest()
        let users = try context.fetch(request)

        var domainCounts: [String: Int] = [:]
        for user in users {
            if let domain = user.email.split(separator: "@").last {
                let domainString = String(domain)
                domainCounts[domainString, default: 0] += 1
            }
        }

        return domainCounts
    }
}
```

## バッチ処理

大量のデータを効率的に処理します:

```swift
extension CoreDataUserStore {
    func batchInsert(users: [CreateUserRequest]) async throws {
        try await CoreDataManager.shared.performBackgroundTask { context in
            for request in users {
                _ = User.create(in: context, name: request.name, email: request.email)
            }
            // コンテキストを一度だけ保存
        }
    }

    func batchUpdate(namePrefix: String, newName: String) async throws {
        let fetchRequest = User.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "name BEGINSWITH %@", namePrefix)

        let batchUpdateRequest = NSBatchUpdateRequest(entity: User.entity())
        batchUpdateRequest.predicate = fetchRequest.predicate
        batchUpdateRequest.propertiesToUpdate = ["name": newName]
        batchUpdateRequest.resultType = .updatedObjectsCountResultType

        try await CoreDataManager.shared.performBackgroundTask { context in
            try context.execute(batchUpdateRequest)
        }
    }

    func batchDelete(predicate: NSPredicate) async throws {
        let request = NSBatchDeleteRequest(fetchRequest: User.fetchRequest())
        request.predicate = predicate

        try await CoreDataManager.shared.performBackgroundTask { context in
            try context.execute(request)
        }
    }
}
```

## リレーションシップの管理

エンティティ間の関連を適切に管理します:

```swift
// Post エンティティ
@objc(Post)
public class Post: NSManagedObject {
    @NSManaged public var id: UUID
    @NSManaged public var title: String
    @NSManaged public var content: String
    @NSManaged public var createdAt: Date
    @NSManaged public var author: User?
}

extension Post {
    static func create(
        in context: NSManagedObjectContext,
        title: String,
        content: String,
        author: User
    ) -> Post {
        let post = Post(context: context)
        post.id = UUID()
        post.title = title
        post.content = content
        post.createdAt = Date()
        post.author = author

        // 逆方向のリレーションも自動的に設定される
        return post
    }

    static func fetchByAuthor(
        _ author: User,
        in context: NSManagedObjectContext
    ) throws -> [Post] {
        let request = fetchRequest()
        request.predicate = NSPredicate(format: "author == %@", author)
        request.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]
        return try context.fetch(request)
    }
}
```

## マイグレーション

データモデルの変更に対応します:

```swift
class CoreDataMigrationManager {
    static let shared = CoreDataMigrationManager()

    func migrateIfNeeded() {
        guard needsMigration() else { return }

        performMigration()
    }

    private func needsMigration() -> Bool {
        let storeURL = CoreDataManager.shared.persistentContainer
            .persistentStoreDescriptions.first?.url

        guard let url = storeURL,
              let metadata = try? NSPersistentStoreCoordinator.metadataForPersistentStore(
                ofType: NSSQLiteStoreType,
                at: url
              ) else {
            return false
        }

        let model = CoreDataManager.shared.persistentContainer.managedObjectModel
        return !model.isConfiguration(withName: nil, compatibleWithStoreMetadata: metadata)
    }

    private func performMigration() {
        // 軽量マイグレーション
        let description = NSPersistentStoreDescription()
        description.shouldMigrateStoreAutomatically = true
        description.shouldInferMappingModelAutomatically = true

        // 重量マイグレーションが必要な場合は、
        // カスタムマッピングモデルを作成
    }
}
```

## まとめ

この章では、Core Data の基本から実践的な使用方法まで学びました:

- Core Data Stack の構築
- エンティティの定義と拡張
- Repository パターンの実装
- NSFetchedResultsController の使用
- 複雑なクエリとバッチ処理
- リレーションシップの管理
- マイグレーション

次の章では、Realm と SQLite について学びます。

## 参考リソース

- [Core Data - Apple Developer](https://developer.apple.com/documentation/coredata)
- [NSPersistentContainer - Apple Developer](https://developer.apple.com/documentation/coredata/nspersistentcontainer)
