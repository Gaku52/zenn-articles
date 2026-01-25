---
title: "Core Data応用"
---

# Core Data応用

本章では、Core Dataのより高度な機能と最適化手法を解説します。リレーションシップの管理、NSFetchedResultsControllerによるUIとの統合、バックグラウンド処理、マイグレーション、パフォーマンス最適化など、実践的なテクニックを学びます。

## リレーションシップの管理

Core Dataでは、エンティティ間の関係を定義し、効率的に管理できます。

### One-to-Manyリレーションシップ

1対多の関係を実装する例として、ユーザーと投稿の関係を定義します。

```swift
import CoreData

// User エンティティ
@objc(User)
public class User: NSManagedObject {
    @NSManaged public var id: UUID
    @NSManaged public var name: String
    @NSManaged public var email: String
    @NSManaged public var posts: NSSet?
}

extension User {
    @objc(addPostsObject:)
    @NSManaged public func addToPosts(_ value: Post)

    @objc(removePostsObject:)
    @NSManaged public func removeFromPosts(_ value: Post)

    @objc(addPosts:)
    @NSManaged public func addToPosts(_ values: NSSet)

    @objc(removePosts:)
    @NSManaged public func removeFromPosts(_ values: NSSet)

    var postsArray: [Post] {
        let set = posts as? Set<Post> ?? []
        return set.sorted { $0.createdAt > $1.createdAt }
    }
}

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
    @nonobjc public class func fetchRequest() -> NSFetchRequest<Post> {
        return NSFetchRequest<Post>(entityName: "Post")
    }

    convenience init(context: NSManagedObjectContext, title: String, content: String, author: User) {
        self.init(context: context)
        self.id = UUID()
        self.title = title
        self.content = content
        self.createdAt = Date()
        self.author = author
    }
}
```

### Many-to-Manyリレーションシップ

多対多の関係を実装する例として、投稿とタグの関係を定義します。

```swift
import CoreData

// Post エンティティ（拡張）
extension Post {
    @NSManaged public var tags: NSSet?

    @objc(addTagsObject:)
    @NSManaged public func addToTags(_ value: Tag)

    @objc(removeTagsObject:)
    @NSManaged public func removeFromTags(_ value: Tag)

    var tagsArray: [Tag] {
        let set = tags as? Set<Tag> ?? []
        return set.sorted { $0.name < $1.name }
    }
}

// Tag エンティティ
@objc(Tag)
public class Tag: NSManagedObject {
    @NSManaged public var id: UUID
    @NSManaged public var name: String
    @NSManaged public var posts: NSSet?
}

extension Tag {
    @nonobjc public class func fetchRequest() -> NSFetchRequest<Tag> {
        return NSFetchRequest<Tag>(entityName: "Tag")
    }

    var postsArray: [Post] {
        let set = posts as? Set<Post> ?? []
        return set.sorted { $0.createdAt > $1.createdAt }
    }

    convenience init(context: NSManagedObjectContext, name: String) {
        self.init(context: context)
        self.id = UUID()
        self.name = name
    }
}
```

### リレーションシップを使用したクエリ

```swift
class PostRepository {
    private let context: NSManagedObjectContext

    init(context: NSManagedObjectContext = CoreDataManager.shared.viewContext) {
        self.context = context
    }

    // 特定ユーザーの投稿を取得
    func fetchPosts(by user: User) throws -> [Post] {
        let fetchRequest = Post.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "author == %@", user)
        fetchRequest.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]
        return try context.fetch(fetchRequest)
    }

    // 特定タグを持つ投稿を取得
    func fetchPosts(withTag tag: Tag) throws -> [Post] {
        let fetchRequest = Post.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "ANY tags == %@", tag)
        fetchRequest.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]
        return try context.fetch(fetchRequest)
    }

    // 複数タグを全て持つ投稿を取得
    func fetchPosts(withAllTags tags: [Tag]) throws -> [Post] {
        let fetchRequest = Post.fetchRequest()

        var predicates: [NSPredicate] = []
        for tag in tags {
            predicates.append(NSPredicate(format: "ANY tags == %@", tag))
        }

        fetchRequest.predicate = NSCompoundPredicate(andPredicateWithSubpredicates: predicates)
        fetchRequest.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]

        return try context.fetch(fetchRequest)
    }
}
```

## NSFetchedResultsController

NSFetchedResultsControllerは、Core DataとUIを効率的に統合するためのコントローラーです。データの変更を自動的に監視し、UIを更新します。

### UIKitでの使用例

```swift
import UIKit
import CoreData

class PostListViewController: UITableViewController {
    private var fetchedResultsController: NSFetchedResultsController<Post>!

    override func viewDidLoad() {
        super.viewDidLoad()

        setupFetchedResultsController()
        performFetch()
    }

    private func setupFetchedResultsController() {
        let fetchRequest = Post.fetchRequest()
        fetchRequest.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]
        fetchRequest.fetchBatchSize = 20

        fetchedResultsController = NSFetchedResultsController(
            fetchRequest: fetchRequest,
            managedObjectContext: CoreDataManager.shared.viewContext,
            sectionNameKeyPath: nil,
            cacheName: "PostsCache"
        )

        fetchedResultsController.delegate = self
    }

    private func performFetch() {
        do {
            try fetchedResultsController.performFetch()
            tableView.reloadData()
        } catch {
            print("Error fetching: \(error)")
        }
    }

    // MARK: - UITableViewDataSource

    override func numberOfSections(in tableView: UITableView) -> Int {
        return fetchedResultsController.sections?.count ?? 0
    }

    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return fetchedResultsController.sections?[section].numberOfObjects ?? 0
    }

    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "PostCell", for: indexPath)
        let post = fetchedResultsController.object(at: indexPath)

        cell.textLabel?.text = post.title
        cell.detailTextLabel?.text = post.content

        return cell
    }
}

// MARK: - NSFetchedResultsControllerDelegate

extension PostListViewController: NSFetchedResultsControllerDelegate {
    func controllerWillChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        tableView.beginUpdates()
    }

    func controller(
        _ controller: NSFetchedResultsController<NSFetchRequestResult>,
        didChange anObject: Any,
        at indexPath: IndexPath?,
        for type: NSFetchedResultsChangeType,
        newIndexPath: IndexPath?
    ) {
        switch type {
        case .insert:
            if let newIndexPath = newIndexPath {
                tableView.insertRows(at: [newIndexPath], with: .automatic)
            }
        case .delete:
            if let indexPath = indexPath {
                tableView.deleteRows(at: [indexPath], with: .automatic)
            }
        case .update:
            if let indexPath = indexPath {
                tableView.reloadRows(at: [indexPath], with: .automatic)
            }
        case .move:
            if let indexPath = indexPath, let newIndexPath = newIndexPath {
                tableView.deleteRows(at: [indexPath], with: .automatic)
                tableView.insertRows(at: [newIndexPath], with: .automatic)
            }
        @unknown default:
            break
        }
    }

    func controllerDidChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        tableView.endUpdates()
    }
}
```

### SwiftUIでの使用例

```swift
import SwiftUI
import CoreData

struct PostListView: View {
    @StateObject private var viewModel = PostListViewModel()

    var body: some View {
        NavigationView {
            List {
                ForEach(viewModel.posts) { post in
                    VStack(alignment: .leading, spacing: 8) {
                        Text(post.title)
                            .font(.headline)
                        Text(post.content)
                            .font(.body)
                            .foregroundColor(.secondary)
                            .lineLimit(2)
                    }
                }
            }
            .navigationTitle("投稿一覧")
        }
    }
}

@MainActor
class PostListViewModel: NSObject, ObservableObject {
    @Published var posts: [Post] = []

    private let fetchedResultsController: NSFetchedResultsController<Post>

    override init() {
        let fetchRequest = Post.fetchRequest()
        fetchRequest.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]
        fetchRequest.fetchBatchSize = 20

        fetchedResultsController = NSFetchedResultsController(
            fetchRequest: fetchRequest,
            managedObjectContext: CoreDataManager.shared.viewContext,
            sectionNameKeyPath: nil,
            cacheName: nil
        )

        super.init()

        fetchedResultsController.delegate = self

        do {
            try fetchedResultsController.performFetch()
            posts = fetchedResultsController.fetchedObjects ?? []
        } catch {
            print("Error fetching: \(error)")
        }
    }
}

extension PostListViewModel: NSFetchedResultsControllerDelegate {
    func controllerDidChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        posts = fetchedResultsController.fetchedObjects ?? []
    }
}
```

## バックグラウンドコンテキスト

重い処理はバックグラウンドコンテキストで実行することが推奨されます。

```swift
class BackgroundDataService {
    private let persistentContainer: NSPersistentContainer

    init(persistentContainer: NSPersistentContainer = CoreDataManager.shared.persistentContainer) {
        self.persistentContainer = persistentContainer
    }

    // バックグラウンドでのデータインポート
    func importData(from jsonData: [[String: Any]]) async throws {
        let context = persistentContainer.newBackgroundContext()
        context.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy

        try await context.perform {
            for data in jsonData {
                guard let title = data["title"] as? String,
                      let content = data["content"] as? String else {
                    continue
                }

                let post = Post(context: context)
                post.id = UUID()
                post.title = title
                post.content = content
                post.createdAt = Date()
            }

            if context.hasChanges {
                try context.save()
            }
        }
    }

    // バックグラウンドでの一括更新
    func updateAllPosts(updateBlock: @escaping (Post) -> Void) async throws {
        let context = persistentContainer.newBackgroundContext()

        try await context.perform {
            let fetchRequest = Post.fetchRequest()
            let posts = try context.fetch(fetchRequest)

            for post in posts {
                updateBlock(post)
            }

            if context.hasChanges {
                try context.save()
            }
        }
    }

    // バックグラウンドでの一括削除
    func deleteOldPosts(olderThan days: Int) async throws {
        let context = persistentContainer.newBackgroundContext()

        try await context.perform {
            let calendar = Calendar.current
            guard let cutoffDate = calendar.date(byAdding: .day, value: -days, to: Date()) else {
                return
            }

            let fetchRequest = Post.fetchRequest()
            fetchRequest.predicate = NSPredicate(format: "createdAt < %@", cutoffDate as NSDate)

            let posts = try context.fetch(fetchRequest)

            for post in posts {
                context.delete(post)
            }

            if context.hasChanges {
                try context.save()
            }
        }
    }
}
```

## マイグレーション

データモデルの変更時には、マイグレーションが必要です。

### 軽量マイグレーション

単純な変更（属性の追加、削除、名前変更）は、軽量マイグレーションで対応できます。

```swift
import CoreData

class CoreDataMigrationManager {
    static let shared = CoreDataMigrationManager()

    private init() {}

    func setupPersistentContainer() -> NSPersistentContainer {
        let container = NSPersistentContainer(name: "AppModel")

        // 軽量マイグレーションの有効化
        let description = container.persistentStoreDescriptions.first
        description?.shouldInferMappingModelAutomatically = true
        description?.shouldMigrateStoreAutomatically = true

        container.loadPersistentStores { description, error in
            if let error = error {
                fatalError("Unable to load persistent stores: \(error)")
            }
        }

        return container
    }

    // マイグレーション履歴の確認
    func checkMigrationHistory() {
        let storeURL = NSPersistentContainer.defaultDirectoryURL()
            .appendingPathComponent("AppModel.sqlite")

        guard FileManager.default.fileExists(atPath: storeURL.path) else {
            print("Store does not exist yet")
            return
        }

        do {
            let metadata = try NSPersistentStoreCoordinator.metadataForPersistentStore(
                ofType: NSSQLiteStoreType,
                at: storeURL
            )

            print("Store metadata: \(metadata)")
        } catch {
            print("Error reading metadata: \(error)")
        }
    }
}
```

### カスタムマイグレーション

複雑な変更には、カスタムマイグレーションポリシーを実装します。

```swift
import CoreData

class CustomMigrationPolicy: NSEntityMigrationPolicy {
    override func createDestinationInstances(
        forSource sInstance: NSManagedObject,
        in mapping: NSEntityMapping,
        manager: NSMigrationManager
    ) throws {
        try super.createDestinationInstances(forSource: sInstance, in: mapping, manager: manager)

        // カスタムマイグレーションロジック
        guard let destinationInstance = manager.destinationInstances(
            forEntityMappingName: mapping.name,
            sourceInstances: [sInstance]
        ).first else {
            return
        }

        // 例: fullNameをfirstNameとlastNameに分割
        if let fullName = sInstance.value(forKey: "fullName") as? String {
            let components = fullName.components(separatedBy: " ")
            destinationInstance.setValue(components.first, forKey: "firstName")
            destinationInstance.setValue(components.last, forKey: "lastName")
        }
    }
}
```

## パフォーマンス最適化

### Faulting

Core Dataは、メモリ効率のためにフォルティング（遅延読み込み）を使用します。

```swift
class PerformanceOptimizer {
    private let context: NSManagedObjectContext

    init(context: NSManagedObjectContext = CoreDataManager.shared.viewContext) {
        self.context = context
    }

    // リレーションシップのプリフェッチ
    func fetchPostsWithAuthor() throws -> [Post] {
        let fetchRequest = Post.fetchRequest()

        // authorをプリフェッチ（N+1問題の回避）
        fetchRequest.relationshipKeyPathsForPrefetching = ["author", "tags"]

        return try context.fetch(fetchRequest)
    }

    // 必要なプロパティのみを取得
    func fetchPostTitles() throws -> [(UUID, String)] {
        let fetchRequest = Post.fetchRequest()

        // プロパティの制限
        fetchRequest.propertiesToFetch = ["id", "title"]
        fetchRequest.resultType = .dictionaryResultType

        let results = try context.fetch(fetchRequest) as? [[String: Any]] ?? []

        return results.compactMap { dict in
            guard let id = dict["id"] as? UUID,
                  let title = dict["title"] as? String else {
                return nil
            }
            return (id, title)
        }
    }
}
```

### Batch処理

大量のデータを処理する際は、バッチ処理を使用します。

```swift
class BatchProcessor {
    private let context: NSManagedObjectContext

    init(context: NSManagedObjectContext = CoreDataManager.shared.viewContext) {
        self.context = context
    }

    // バッチ更新
    func batchUpdatePostTitles(containing keyword: String, newTitle: String) throws {
        let batchUpdate = NSBatchUpdateRequest(entityName: "Post")
        batchUpdate.predicate = NSPredicate(format: "title CONTAINS %@", keyword)
        batchUpdate.propertiesToUpdate = ["title": newTitle]
        batchUpdate.resultType = .updatedObjectIDsResultType

        let result = try context.execute(batchUpdate) as? NSBatchUpdateResult
        let objectIDArray = result?.result as? [NSManagedObjectID] ?? []

        // 変更をコンテキストにマージ
        let changes = [NSUpdatedObjectsKey: objectIDArray]
        NSManagedObjectContext.mergeChanges(fromRemoteContextSave: changes, into: [context])
    }

    // バッチ削除
    func batchDeleteOldPosts(olderThan days: Int) throws {
        let calendar = Calendar.current
        guard let cutoffDate = calendar.date(byAdding: .day, value: -days, to: Date()) else {
            return
        }

        let fetchRequest = Post.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "createdAt < %@", cutoffDate as NSDate)

        let batchDelete = NSBatchDeleteRequest(fetchRequest: fetchRequest as! NSFetchRequest<NSFetchRequestResult>)
        batchDelete.resultType = .resultTypeObjectIDs

        let result = try context.execute(batchDelete) as? NSBatchDeleteResult
        let objectIDArray = result?.result as? [NSManagedObjectID] ?? []

        // 変更をコンテキストにマージ
        let changes = [NSDeletedObjectsKey: objectIDArray]
        NSManagedObjectContext.mergeChanges(fromRemoteContextSave: changes, into: [context])
    }
}
```

### メモリ管理

```swift
class MemoryOptimizer {
    private let context: NSManagedObjectContext

    init(context: NSManagedObjectContext = CoreDataManager.shared.viewContext) {
        self.context = context
    }

    // 大量データ処理時のメモリ管理
    func processLargeDataSet() async throws {
        let backgroundContext = CoreDataManager.shared.newBackgroundContext()

        try await backgroundContext.perform {
            let fetchRequest = Post.fetchRequest()
            fetchRequest.fetchBatchSize = 100

            let posts = try backgroundContext.fetch(fetchRequest)

            for (index, post) in posts.enumerated() {
                // 処理を実行
                post.updatedAt = Date()

                // 100件ごとに保存してメモリをリフレッシュ
                if index % 100 == 0 {
                    try backgroundContext.save()
                    backgroundContext.reset()
                }
            }

            // 最後の保存
            if backgroundContext.hasChanges {
                try backgroundContext.save()
            }
        }
    }
}
```

## チェックリスト

Core Data応用実装のベストプラクティスチェックリスト：

- [ ] One-to-ManyおよびMany-to-Manyリレーションシップを適切に定義済み
- [ ] NSFetchedResultsControllerでUIとデータを同期済み
- [ ] 重い処理はバックグラウンドコンテキストで実行済み
- [ ] 軽量マイグレーションを有効化済み
- [ ] 必要に応じてカスタムマイグレーションポリシーを実装済み
- [ ] リレーションシップのプリフェッチでN+1問題を回避済み
- [ ] バッチ更新・削除で大量データを効率的に処理済み
- [ ] fetchBatchSizeでメモリ使用量を最適化済み
- [ ] 定期的にコンテキストをresetしてメモリを解放済み
- [ ] マージポリシーを適切に設定済み

## まとめ

Core Dataの高度な機能を活用することで、パフォーマンスとユーザー体験が向上します。リレーションシップによる複雑なデータモデルの管理、NSFetchedResultsControllerによるUIとの効率的な統合、バックグラウンドコンテキストでの非同期処理、適切なマイグレーション戦略、そしてフォルティングとバッチ処理によるパフォーマンス最適化を組み合わせることで、スケーラブルなデータ層を構築できます。

次章では、Realmデータベースについて解説します。

## 参考文献

- [Apple Developer Documentation - Core Data](https://developer.apple.com/documentation/coredata)
- [Apple Developer Documentation - NSFetchedResultsController](https://developer.apple.com/documentation/coredata/nsfetchedresultscontroller)
- [Apple Developer Documentation - Core Data Migration](https://developer.apple.com/documentation/coredata/using_lightweight_migration)
