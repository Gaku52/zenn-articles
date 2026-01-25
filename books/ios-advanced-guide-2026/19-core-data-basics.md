---
title: "Core Data基礎"
---

# Core Data基礎

Core DataはApple公式のデータ永続化フレームワークです。オブジェクトグラフ管理とデータ永続化を組み合わせた強力なフレームワークで、iOSアプリケーションにおける複雑なデータモデルの管理に適しています。本章では、Core Dataの基本的な使い方を解説します。

## Core Dataとは

Core Dataは単なるデータベースラッパーではなく、オブジェクトグラフ管理フレームワークです。主な機能として以下があります：

- オブジェクトのライフサイクル管理
- データの永続化（SQLite、バイナリ、インメモリ）
- データのバリデーション
- リレーションシップの管理
- 変更の追跡
- Undo/Redo機能

## Core Data Stackのセットアップ

Core Dataを使用するには、Core Data Stackを構築する必要があります。Core Data Stackは、以下のコンポーネントで構成されます：

- NSManagedObjectModel: データモデルの定義
- NSPersistentStoreCoordinator: ストアの管理
- NSManagedObjectContext: オブジェクトの操作
- NSPersistentContainer: スタック全体の管理（iOS 10以降）

```swift
import CoreData

class CoreDataManager {
    static let shared = CoreDataManager()

    // シングルトンパターン
    private init() {}

    // NSPersistentContainerの作成
    lazy var persistentContainer: NSPersistentContainer = {
        let container = NSPersistentContainer(name: "AppModel")

        // ストアの読み込み
        container.loadPersistentStores { description, error in
            if let error = error {
                // エラーハンドリング
                fatalError("Unable to load persistent stores: \(error)")
            }

            // 自動マージポリシーの設定
            description.setOption(true as NSNumber, forKey: NSPersistentHistoryTrackingKey)
            description.setOption(true as NSNumber, forKey: NSPersistentStoreRemoteChangeNotificationPostOptionKey)
        }

        // ビューコンテキストの設定
        container.viewContext.automaticallyMergesChangesFromParent = true
        container.viewContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy

        return container
    }()

    // メインスレッド用のコンテキスト
    var viewContext: NSManagedObjectContext {
        persistentContainer.viewContext
    }

    // バックグラウンドコンテキストの作成
    func newBackgroundContext() -> NSManagedObjectContext {
        let context = persistentContainer.newBackgroundContext()
        context.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
        return context
    }

    // コンテキストの保存
    func saveContext(_ context: NSManagedObjectContext) {
        guard context.hasChanges else { return }

        do {
            try context.save()
        } catch {
            let nsError = error as NSError
            print("Error saving context: \(nsError), \(nsError.userInfo)")
        }
    }
}
```

## データモデルの定義

Xcodeのデータモデルエディタを使用して、エンティティと属性を定義します。

### プログラムによるNSManagedObjectサブクラスの定義

```swift
import CoreData

@objc(User)
public class User: NSManagedObject {
    @NSManaged public var id: UUID
    @NSManaged public var name: String
    @NSManaged public var email: String
    @NSManaged public var age: Int16
    @NSManaged public var createdAt: Date
    @NSManaged public var updatedAt: Date
}

extension User {
    @nonobjc public class func fetchRequest() -> NSFetchRequest<User> {
        return NSFetchRequest<User>(entityName: "User")
    }

    // 便利なイニシャライザ
    convenience init(context: NSManagedObjectContext, name: String, email: String, age: Int16) {
        self.init(context: context)
        self.id = UUID()
        self.name = name
        self.email = email
        self.age = age
        self.createdAt = Date()
        self.updatedAt = Date()
    }
}
```

## CRUD操作

### Create（作成）

```swift
import CoreData

class UserRepository {
    private let context: NSManagedObjectContext

    init(context: NSManagedObjectContext = CoreDataManager.shared.viewContext) {
        self.context = context
    }

    // ユーザーの作成
    func createUser(name: String, email: String, age: Int) throws -> User {
        let user = User(context: context, name: name, email: email, age: Int16(age))

        try context.save()

        return user
    }

    // 複数ユーザーの一括作成
    func createUsers(_ users: [(name: String, email: String, age: Int)]) throws {
        for userData in users {
            _ = User(
                context: context,
                name: userData.name,
                email: userData.email,
                age: Int16(userData.age)
            )
        }

        try context.save()
    }
}
```

### Read（読み取り）

```swift
extension UserRepository {
    // 全ユーザーの取得
    func fetchAllUsers() throws -> [User] {
        let fetchRequest = User.fetchRequest()
        return try context.fetch(fetchRequest)
    }

    // IDによるユーザーの取得
    func fetchUser(by id: UUID) throws -> User? {
        let fetchRequest = User.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "id == %@", id as CVarArg)
        fetchRequest.fetchLimit = 1

        return try context.fetch(fetchRequest).first
    }

    // メールアドレスによるユーザーの取得
    func fetchUser(by email: String) throws -> User? {
        let fetchRequest = User.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "email == %@", email)
        fetchRequest.fetchLimit = 1

        return try context.fetch(fetchRequest).first
    }

    // 年齢範囲によるユーザーの取得
    func fetchUsers(ageRange: ClosedRange<Int>) throws -> [User] {
        let fetchRequest = User.fetchRequest()
        fetchRequest.predicate = NSPredicate(
            format: "age >= %d AND age <= %d",
            ageRange.lowerBound,
            ageRange.upperBound
        )

        return try context.fetch(fetchRequest)
    }
}
```

### Update（更新）

```swift
extension UserRepository {
    // ユーザー情報の更新
    func updateUser(_ user: User, name: String? = nil, email: String? = nil, age: Int? = nil) throws {
        if let name = name {
            user.name = name
        }

        if let email = email {
            user.email = email
        }

        if let age = age {
            user.age = Int16(age)
        }

        user.updatedAt = Date()

        try context.save()
    }

    // 条件に一致するユーザーの一括更新
    func updateUsers(matching predicate: NSPredicate, updates: (User) -> Void) throws {
        let fetchRequest = User.fetchRequest()
        fetchRequest.predicate = predicate

        let users = try context.fetch(fetchRequest)

        for user in users {
            updates(user)
            user.updatedAt = Date()
        }

        try context.save()
    }
}
```

### Delete（削除）

```swift
extension UserRepository {
    // ユーザーの削除
    func deleteUser(_ user: User) throws {
        context.delete(user)
        try context.save()
    }

    // IDによるユーザーの削除
    func deleteUser(by id: UUID) throws {
        guard let user = try fetchUser(by: id) else {
            throw RepositoryError.userNotFound
        }

        context.delete(user)
        try context.save()
    }

    // 条件に一致するユーザーの一括削除
    func deleteUsers(matching predicate: NSPredicate) throws {
        let fetchRequest = User.fetchRequest()
        fetchRequest.predicate = predicate

        let users = try context.fetch(fetchRequest)

        for user in users {
            context.delete(user)
        }

        try context.save()
    }

    // 全ユーザーの削除
    func deleteAllUsers() throws {
        let fetchRequest = User.fetchRequest()
        let users = try context.fetch(fetchRequest)

        for user in users {
            context.delete(user)
        }

        try context.save()
    }
}

enum RepositoryError: Error {
    case userNotFound
}
```

## NSFetchRequest

NSFetchRequestは、Core Dataからデータを取得するためのクエリを構築するオブジェクトです。

```swift
class UserQueryBuilder {
    private let context: NSManagedObjectContext

    init(context: NSManagedObjectContext = CoreDataManager.shared.viewContext) {
        self.context = context
    }

    // 基本的なフェッチリクエスト
    func basicFetchRequest() throws -> [User] {
        let fetchRequest = User.fetchRequest()
        return try context.fetch(fetchRequest)
    }

    // フェッチリクエストのカスタマイズ
    func customFetchRequest() throws -> [User] {
        let fetchRequest = User.fetchRequest()

        // フェッチ件数の制限
        fetchRequest.fetchLimit = 10

        // オフセットの設定
        fetchRequest.fetchOffset = 0

        // バッチサイズの設定（パフォーマンス最適化）
        fetchRequest.fetchBatchSize = 20

        return try context.fetch(fetchRequest)
    }

    // 特定プロパティのみを取得
    func fetchUserNames() throws -> [String] {
        let fetchRequest = User.fetchRequest()

        // プロパティのみを取得
        fetchRequest.resultType = .dictionaryResultType
        fetchRequest.propertiesToFetch = ["name"]

        let results = try context.fetch(fetchRequest)

        return results.compactMap { user in
            user.name
        }
    }

    // 件数のカウント
    func countUsers() throws -> Int {
        let fetchRequest = User.fetchRequest()
        return try context.count(for: fetchRequest)
    }
}
```

## Predicateによるフィルタリング

NSPredicateを使用して、データをフィルタリングします。

```swift
extension UserQueryBuilder {
    // 完全一致
    func fetchUsers(name: String) throws -> [User] {
        let fetchRequest = User.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "name == %@", name)
        return try context.fetch(fetchRequest)
    }

    // 部分一致（CONTAINS）
    func fetchUsers(nameContains: String) throws -> [User] {
        let fetchRequest = User.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "name CONTAINS[cd] %@", nameContains)
        return try context.fetch(fetchRequest)
    }

    // 前方一致（BEGINSWITH）
    func fetchUsers(nameStartsWith: String) throws -> [User] {
        let fetchRequest = User.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "name BEGINSWITH[cd] %@", nameStartsWith)
        return try context.fetch(fetchRequest)
    }

    // 範囲指定
    func fetchUsers(ageBetween min: Int, and max: Int) throws -> [User] {
        let fetchRequest = User.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "age BETWEEN {%d, %d}", min, max)
        return try context.fetch(fetchRequest)
    }

    // IN条件
    func fetchUsers(ids: [UUID]) throws -> [User] {
        let fetchRequest = User.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "id IN %@", ids)
        return try context.fetch(fetchRequest)
    }

    // 複合条件（AND）
    func fetchUsers(minAge: Int, nameContains: String) throws -> [User] {
        let fetchRequest = User.fetchRequest()

        let agePredicate = NSPredicate(format: "age >= %d", minAge)
        let namePredicate = NSPredicate(format: "name CONTAINS[cd] %@", nameContains)

        fetchRequest.predicate = NSCompoundPredicate(andPredicateWithSubpredicates: [
            agePredicate,
            namePredicate
        ])

        return try context.fetch(fetchRequest)
    }

    // 複合条件（OR）
    func fetchUsers(name: String, orEmail email: String) throws -> [User] {
        let fetchRequest = User.fetchRequest()

        let namePredicate = NSPredicate(format: "name == %@", name)
        let emailPredicate = NSPredicate(format: "email == %@", email)

        fetchRequest.predicate = NSCompoundPredicate(orPredicateWithSubpredicates: [
            namePredicate,
            emailPredicate
        ])

        return try context.fetch(fetchRequest)
    }
}
```

## Sort Descriptorによるソート

NSSortDescriptorを使用して、取得結果をソートします。

```swift
extension UserQueryBuilder {
    // 名前の昇順でソート
    func fetchUsersSortedByName() throws -> [User] {
        let fetchRequest = User.fetchRequest()
        fetchRequest.sortDescriptors = [NSSortDescriptor(key: "name", ascending: true)]
        return try context.fetch(fetchRequest)
    }

    // 年齢の降順でソート
    func fetchUsersSortedByAge() throws -> [User] {
        let fetchRequest = User.fetchRequest()
        fetchRequest.sortDescriptors = [NSSortDescriptor(key: "age", ascending: false)]
        return try context.fetch(fetchRequest)
    }

    // 複数条件でソート（年齢降順 → 名前昇順）
    func fetchUsersSortedByAgeAndName() throws -> [User] {
        let fetchRequest = User.fetchRequest()
        fetchRequest.sortDescriptors = [
            NSSortDescriptor(key: "age", ascending: false),
            NSSortDescriptor(key: "name", ascending: true)
        ]
        return try context.fetch(fetchRequest)
    }

    // 大文字小文字を区別しないソート
    func fetchUsersSortedByNameCaseInsensitive() throws -> [User] {
        let fetchRequest = User.fetchRequest()
        fetchRequest.sortDescriptors = [
            NSSortDescriptor(key: "name", ascending: true, selector: #selector(NSString.caseInsensitiveCompare(_:)))
        ]
        return try context.fetch(fetchRequest)
    }
}
```

## SwiftUIとの統合

SwiftUIでCore Dataを使用する方法を示します。

```swift
import SwiftUI
import CoreData

struct UserListView: View {
    @StateObject private var viewModel = UserListViewModel()

    var body: some View {
        NavigationView {
            List {
                ForEach(viewModel.users) { user in
                    VStack(alignment: .leading) {
                        Text(user.name)
                            .font(.headline)
                        Text(user.email)
                            .font(.subheadline)
                            .foregroundColor(.secondary)
                    }
                }
                .onDelete(perform: viewModel.deleteUsers)
            }
            .navigationTitle("ユーザー一覧")
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    Button("追加") {
                        viewModel.addUser()
                    }
                }
            }
        }
        .onAppear {
            viewModel.fetchUsers()
        }
    }
}

@MainActor
class UserListViewModel: ObservableObject {
    @Published var users: [User] = []

    private let repository: UserRepository

    init(repository: UserRepository = UserRepository()) {
        self.repository = repository
    }

    func fetchUsers() {
        do {
            users = try repository.fetchAllUsers()
        } catch {
            print("Error fetching users: \(error)")
        }
    }

    func addUser() {
        do {
            let user = try repository.createUser(
                name: "New User",
                email: "user@example.com",
                age: 25
            )
            users.append(user)
        } catch {
            print("Error creating user: \(error)")
        }
    }

    func deleteUsers(at offsets: IndexSet) {
        do {
            for index in offsets {
                let user = users[index]
                try repository.deleteUser(user)
            }
            users.remove(atOffsets: offsets)
        } catch {
            print("Error deleting user: \(error)")
        }
    }
}
```

## チェックリスト

Core Data基礎実装のベストプラクティスチェックリスト：

- [ ] Core Data Stackを適切にセットアップ済み
- [ ] シングルトンパターンでCoreDataManagerを実装済み
- [ ] エンティティとNSManagedObjectサブクラスを定義済み
- [ ] CRUD操作を実装するRepositoryパターンを採用済み
- [ ] NSFetchRequestでクエリを構築済み
- [ ] NSPredicateで適切なフィルタリングを実装済み
- [ ] NSSortDescriptorでソート機能を実装済み
- [ ] エラーハンドリングを適切に実装済み
- [ ] メモリリークを防ぐためのコンテキスト管理を実装済み
- [ ] SwiftUIまたはUIKitとの統合を実装済み

## まとめ

Core Dataを使うことで、大量のデータを効率的に管理できます。NSPersistentContainerによる簡潔なスタック構築、NSFetchRequestによる柔軟なクエリ、NSPredicateとNSSortDescriptorによる強力なフィルタリングとソート機能を活用することで、堅牢なデータ永続化層を構築できます。

次章では、Core Dataの高度な活用方法について解説します。

## 参考文献

- [Apple Developer Documentation - Core Data](https://developer.apple.com/documentation/coredata)
- [Apple Developer Documentation - NSPersistentContainer](https://developer.apple.com/documentation/coredata/nspersistentcontainer)
- [Apple Developer Documentation - NSFetchRequest](https://developer.apple.com/documentation/coredata/nsfetchrequest)
