---
title: "Chapter 16: Core Data完全マスター Part 2 - マイグレーションとパフォーマンス最適化"
---

# Chapter 16: Core Data完全マスター Part 2 - マイグレーションとパフォーマンス最適化

## 16.1 概要

この章では、Core Dataの高度なトピックであるマイグレーション、パフォーマンス最適化、SwiftUIとの統合について学びます。これらは本番環境でCore Dataを運用する上で必須の知識です。

### 本章で学べること

- データモデルのマイグレーション戦略
- カスタムマイグレーションの実装
- フェッチリクエストの最適化
- インデックスとメモリ管理
- SwiftUIとの効率的な統合
- プロダクション環境でのベストプラクティス

## 16.2 マイグレーション

### 16.2.1 軽量マイグレーション

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

### 16.2.2 カスタムマイグレーション

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

### 16.2.3 段階的マイグレーション

```swift
class ProgressiveMigrationManager {
    private let sourceModel: NSManagedObjectModel
    private let destinationModel: NSManagedObjectModel
    private let storeURL: URL

    init(sourceModel: NSManagedObjectModel, destinationModel: NSManagedObjectModel, storeURL: URL) {
        self.sourceModel = sourceModel
        self.destinationModel = destinationModel
        self.storeURL = storeURL
    }

    func performMigration() throws {
        // マイグレーションパスを取得
        let migrationSteps = try getMigrationSteps()

        // 各ステップを順次実行
        for step in migrationSteps {
            print("Migrating from \(step.source.versionIdentifiers) to \(step.destination.versionIdentifiers)")
            try migrateStep(step)
        }
    }

    private func getMigrationSteps() throws -> [MigrationStep] {
        // モデルバージョンの検出
        let allModelVersions = try getAllModelVersions()

        // 現在のストアのバージョンを特定
        let metadata = try NSPersistentStoreCoordinator.metadataForPersistentStore(
            ofType: NSSQLiteStoreType,
            at: storeURL
        )

        guard let currentModelVersion = allModelVersions.first(where: {
            $0.isConfiguration(withName: nil, compatibleWithStoreMetadata: metadata)
        }) else {
            throw MigrationError.sourceModelNotFound
        }

        // マイグレーションパスを構築
        return buildMigrationPath(from: currentModelVersion, to: destinationModel, allVersions: allModelVersions)
    }

    private func getAllModelVersions() throws -> [NSManagedObjectModel] {
        guard let modelURL = Bundle.main.url(forResource: "AppDataModel", withExtension: "momd") else {
            throw MigrationError.modelNotFound
        }

        guard let versionURLs = FileManager.default.subpaths(atPath: modelURL.path)?
            .filter({ $0.hasSuffix(".mom") })
            .map({ modelURL.appendingPathComponent($0) }) else {
            throw MigrationError.modelVersionsNotFound
        }

        return versionURLs.compactMap { NSManagedObjectModel(contentsOf: $0) }
    }

    private func buildMigrationPath(
        from source: NSManagedObjectModel,
        to destination: NSManagedObjectModel,
        allVersions: [NSManagedObjectModel]
    ) -> [MigrationStep] {
        // マイグレーションステップを構築
        var steps: [MigrationStep] = []
        var currentModel = source

        while currentModel != destination {
            // 次のバージョンを見つける
            if let nextModel = findNextModel(from: currentModel, to: destination, in: allVersions) {
                steps.append(MigrationStep(source: currentModel, destination: nextModel))
                currentModel = nextModel
            } else {
                break
            }
        }

        return steps
    }

    private func findNextModel(
        from source: NSManagedObjectModel,
        to destination: NSManagedObjectModel,
        in allVersions: [NSManagedObjectModel]
    ) -> NSManagedObjectModel? {
        // 直接のマッピングモデルが存在するバージョンを探す
        return allVersions.first { model in
            NSMappingModel(from: [Bundle.main], forSourceModel: source, destinationModel: model) != nil
        }
    }

    private func migrateStep(_ step: MigrationStep) throws {
        let manager = CustomMigrationManager()
        try manager.migrateStore(at: storeURL, from: step.source, to: step.destination)
    }

    struct MigrationStep {
        let source: NSManagedObjectModel
        let destination: NSManagedObjectModel
    }

    enum MigrationError: Error {
        case modelNotFound
        case modelVersionsNotFound
        case sourceModelNotFound
        case migrationFailed
    }
}
```

## 16.3 パフォーマンス最適化

### 16.3.1 フェッチリクエストの最適化

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

### 16.3.2 インデックスの活用

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

複合インデックス:
- Post: [isPublished, createdAt] - 公開済み投稿を日付順で取得する場合
- User: [isActive, createdAt] - アクティブユーザーを日付順で取得する場合
*/
```

### 16.3.3 メモリ管理

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

    // 大量データの一括処理（メモリ効率的）
    func efficientBatchProcessing() async throws {
        try await coreData.performBackgroundTask { context in
            context.shouldDeleteInaccessibleFaults = true

            let request = Post.fetchRequest()
            request.fetchBatchSize = 100
            request.returnsObjectsAsFaults = true

            let posts = try context.fetch(request)

            var processedCount = 0
            for post in posts {
                // 処理
                post.viewCount += 1

                processedCount += 1

                // 100件ごとに保存してメモリをクリア
                if processedCount % 100 == 0 {
                    try context.save()
                    context.reset() // すべてのオブジェクトをフォールト化
                }
            }

            // 残りを保存
            if context.hasChanges {
                try context.save()
            }
        }
    }
}
```

### 16.3.4 Predicateの最適化

```swift
class PredicateOptimization {
    private let coreData = CoreDataManager.shared

    // ❌ 非効率なPredicate（大文字小文字を区別しない検索）
    func inefficientSearch(query: String) async throws -> [Post] {
        try await coreData.performBackgroundTask { context in
            let request = Post.fetchRequest()
            request.predicate = NSPredicate(format: "title CONTAINS[cd] %@", query)
            return try context.fetch(request)
        }
    }

    // ✅ 効率的なPredicate（完全一致）
    func efficientExactMatch(title: String) async throws -> [Post] {
        try await coreData.performBackgroundTask { context in
            let request = Post.fetchRequest()
            request.predicate = NSPredicate(format: "title == %@", title)
            return try context.fetch(request)
        }
    }

    // ✅ インデックスを活用したPredicate
    func efficientIndexedSearch(isPublished: Bool) async throws -> [Post] {
        try await coreData.performBackgroundTask { context in
            let request = Post.fetchRequest()
            request.predicate = NSPredicate(format: "isPublished == %@", NSNumber(value: isPublished))
            request.sortDescriptors = [
                NSSortDescriptor(key: "createdAt", ascending: false)
            ]
            return try context.fetch(request)
        }
    }

    // 複合Predicateの最適化
    func optimizedCompoundPredicate() async throws -> [Post] {
        try await coreData.performBackgroundTask { context in
            let request = Post.fetchRequest()

            // インデックス化された属性を優先
            let publishedPredicate = NSPredicate(format: "isPublished == true")
            let datePredicate = NSPredicate(
                format: "createdAt >= %@",
                Calendar.current.date(byAdding: .day, value: -7, to: Date())! as NSDate
            )

            request.predicate = NSCompoundPredicate(andPredicateWithSubpredicates: [
                publishedPredicate,
                datePredicate
            ])

            return try context.fetch(request)
        }
    }
}
```

## 16.4 SwiftUIとの統合

### 16.4.1 @FetchRequestの使用

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

### 16.4.2 動的フィルタリング

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

### 16.4.3 SectionedFetchRequest

```swift
import SwiftUI

struct SectionedPostListView: View {
    @SectionedFetchRequest<String, Post>(
        sectionIdentifier: \.sectionIdentifier,
        sortDescriptors: [
            SortDescriptor(\.createdAt, order: .reverse)
        ],
        predicate: NSPredicate(format: "isPublished == true")
    )
    private var sections: SectionedFetchResults<String, Post>

    @Environment(\.managedObjectContext) private var viewContext

    var body: some View {
        List {
            ForEach(sections) { section in
                Section(header: Text(section.id)) {
                    ForEach(section) { post in
                        PostRow(post: post)
                    }
                }
            }
        }
        .navigationTitle("Posts by Month")
    }
}
```

### 16.4.4 カスタムFetchedResultsPublisher

```swift
import Combine
import CoreData

class FetchedResultsPublisher<T: NSManagedObject>: NSObject, NSFetchedResultsControllerDelegate {
    private let fetchedResultsController: NSFetchedResultsController<T>
    private let subject = PassthroughSubject<[T], Never>()

    var publisher: AnyPublisher<[T], Never> {
        subject.eraseToAnyPublisher()
    }

    init(fetchRequest: NSFetchRequest<T>, context: NSManagedObjectContext) {
        self.fetchedResultsController = NSFetchedResultsController(
            fetchRequest: fetchRequest,
            managedObjectContext: context,
            sectionNameKeyPath: nil,
            cacheName: nil
        )

        super.init()

        fetchedResultsController.delegate = self
        try? fetchedResultsController.performFetch()

        // 初期値を送信
        subject.send(fetchedResultsController.fetchedObjects ?? [])
    }

    func controllerDidChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        subject.send(controller.fetchedObjects as? [T] ?? [])
    }
}

// 使用例
class PostViewModel: ObservableObject {
    @Published var posts: [Post] = []

    private var cancellables = Set<AnyCancellable>()
    private let publisher: FetchedResultsPublisher<Post>

    init(context: NSManagedObjectContext) {
        let request = Post.fetchRequest()
        request.sortDescriptors = [
            NSSortDescriptor(key: "createdAt", ascending: false)
        ]

        self.publisher = FetchedResultsPublisher(fetchRequest: request, context: context)

        publisher.publisher
            .sink { [weak self] posts in
                self?.posts = posts
            }
            .store(in: &cancellables)
    }
}
```

## 16.5 テストとデバッグ

### 16.5.1 インメモリストアを使ったテスト

```swift
class CoreDataTestStack {
    static func createInMemoryStack() -> NSPersistentContainer {
        let container = NSPersistentContainer(name: "AppDataModel")

        let description = NSPersistentStoreDescription()
        description.type = NSInMemoryStoreType

        container.persistentStoreDescriptions = [description]

        container.loadPersistentStores { description, error in
            if let error = error {
                fatalError("Failed to create in-memory store: \(error)")
            }
        }

        return container
    }
}

// テスト例
import XCTest

class UserRepositoryTests: XCTestCase {
    var container: NSPersistentContainer!
    var repository: UserRepository!

    override func setUp() {
        super.setUp()
        container = CoreDataTestStack.createInMemoryStack()
        repository = UserRepository()
    }

    override func tearDown() {
        container = nil
        repository = nil
        super.tearDown()
    }

    func testCreateUser() async throws {
        let user = try await repository.createUser(
            username: "testuser",
            email: "test@example.com"
        )

        XCTAssertEqual(user.username, "testuser")
        XCTAssertEqual(user.email, "test@example.com")
        XCTAssertTrue(user.isActive)
    }

    func testFetchUser() async throws {
        let createdUser = try await repository.createUser(
            username: "testuser",
            email: "test@example.com"
        )

        let fetchedUser = try await repository.fetchUser(byUsername: "testuser")

        XCTAssertNotNil(fetchedUser)
        XCTAssertEqual(fetchedUser?.id, createdUser.id)
    }
}
```

### 16.5.2 デバッグとロギング

```swift
class CoreDataDebugManager {
    static func enableSQLLogging() {
        // Launch Arguments に追加:
        // -com.apple.CoreData.SQLDebug 1
        // -com.apple.CoreData.Logging.stderr 1

        // またはコードで設定:
        UserDefaults.standard.set(["com.apple.CoreData.SQLDebug": "1"], forKey: "SQLDebug")
    }

    static func printStatistics(for context: NSManagedObjectContext) {
        print("=== Core Data Statistics ===")
        print("Inserted objects: \(context.insertedObjects.count)")
        print("Updated objects: \(context.updatedObjects.count)")
        print("Deleted objects: \(context.deletedObjects.count)")
        print("Has changes: \(context.hasChanges)")
    }

    static func validateManagedObject(_ object: NSManagedObject) throws {
        try object.validateForInsert()
        print("\(object.entity.name ?? "Unknown") validation succeeded")
    }
}
```

## 16.6 ベストプラクティス

### 16.6.1 エラーハンドリング

```swift
enum CoreDataRepositoryError: Error, LocalizedError {
    case objectNotFound
    case saveFailed(underlying: Error)
    case fetchFailed(underlying: Error)
    case deleteFailed(underlying: Error)
    case invalidData(reason: String)
    case contextNotAvailable

    var errorDescription: String? {
        switch self {
        case .objectNotFound:
            return "The requested object was not found in the database."
        case .saveFailed(let error):
            return "Failed to save changes: \(error.localizedDescription)"
        case .fetchFailed(let error):
            return "Failed to fetch data: \(error.localizedDescription)"
        case .deleteFailed(let error):
            return "Failed to delete object: \(error.localizedDescription)"
        case .invalidData(let reason):
            return "Invalid data: \(reason)"
        case .contextNotAvailable:
            return "Managed object context is not available."
        }
    }
}

class SafeUserRepository {
    private let coreData = CoreDataManager.shared

    func createUser(username: String, email: String) async throws -> User {
        guard !username.isEmpty, !email.isEmpty else {
            throw CoreDataRepositoryError.invalidData(reason: "Username and email are required")
        }

        do {
            return try await coreData.performBackgroundTask { context in
                let user = User(context: context)
                user.id = UUID()
                user.username = username
                user.email = email
                user.createdAt = Date()
                user.updatedAt = Date()
                user.isActive = true

                return user
            }
        } catch {
            throw CoreDataRepositoryError.saveFailed(underlying: error)
        }
    }

    func fetchUser(byID id: UUID) async throws -> User {
        do {
            guard let user = try await coreData.performBackgroundTask({ context in
                let request = User.fetchRequest()
                request.predicate = NSPredicate(format: "id == %@", id as CVarArg)
                request.fetchLimit = 1

                return try context.fetch(request).first
            }) else {
                throw CoreDataRepositoryError.objectNotFound
            }

            return user
        } catch let error as CoreDataRepositoryError {
            throw error
        } catch {
            throw CoreDataRepositoryError.fetchFailed(underlying: error)
        }
    }
}
```

### 16.6.2 コンカレンシー管理

```swift
class ConcurrencySafeManager {
    private let coreData = CoreDataManager.shared

    // 並行処理の安全な実装
    func processMultipleOperations() async throws {
        // 複数の独立した操作を並行実行
        try await withThrowingTaskGroup(of: Void.self) { group in
            group.addTask {
                try await self.updatePosts()
            }

            group.addTask {
                try await self.updateUsers()
            }

            group.addTask {
                try await self.cleanupOldData()
            }

            try await group.waitForAll()
        }
    }

    private func updatePosts() async throws {
        try await coreData.performBackgroundTask { context in
            // Post更新処理
            let request = Post.fetchRequest()
            let posts = try context.fetch(request)

            for post in posts {
                post.updatedAt = Date()
            }
        }
    }

    private func updateUsers() async throws {
        try await coreData.performBackgroundTask { context in
            // User更新処理
            let request = User.fetchRequest()
            let users = try context.fetch(request)

            for user in users {
                user.updatedAt = Date()
            }
        }
    }

    private func cleanupOldData() async throws {
        try await coreData.performBackgroundTask { context in
            // 古いデータの削除
            let cutoffDate = Calendar.current.date(byAdding: .month, value: -6, to: Date())!

            let request = Post.fetchRequest()
            request.predicate = NSPredicate(format: "createdAt < %@", cutoffDate as NSDate)

            let oldPosts = try context.fetch(request)
            for post in oldPosts {
                context.delete(post)
            }
        }
    }
}
```

## 16.7 まとめ

この章では、Core Dataの高度なトピックについて詳しく学びました。

### 重要なポイント

1. **マイグレーション**
   - 軽量マイグレーションの活用
   - カスタムマイグレーションの実装
   - 段階的マイグレーション戦略

2. **パフォーマンス最適化**
   - フェッチリクエストの最適化
   - プリフェッチとバッチサイズ
   - インデックスの活用
   - メモリ管理

3. **SwiftUI統合**
   - @FetchRequestの効果的な使用
   - 動的フィルタリング
   - SectionedFetchRequest
   - カスタムPublisher

4. **テストとデバッグ**
   - インメモリストアを使ったテスト
   - SQLロギング
   - 統計情報の収集

5. **ベストプラクティス**
   - 適切なエラーハンドリング
   - コンカレンシー管理
   - 型安全な実装

### プロダクション環境での考慮事項

1. **データの整合性**
   - トランザクション管理
   - Merge Policy の適切な設定
   - バックアップ戦略

2. **スケーラビリティ**
   - バッチ処理の活用
   - ページネーション
   - 段階的データロード

3. **監視とメンテナンス**
   - パフォーマンスモニタリング
   - クラッシュレポート分析
   - データマイグレーションの検証

Core Dataは強力なフレームワークですが、適切に使用しないとパフォーマンス問題やメモリリークを引き起こす可能性があります。本章で学んだベストプラクティスを実践することで、堅牢で効率的なデータ永続化層を構築できます。

次の章では、より軽量で高速なデータベースであるRealmについて学びます。
