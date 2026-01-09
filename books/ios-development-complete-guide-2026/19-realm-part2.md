---
title: "Chapter 18: Realm実装 Part 2 - パフォーマンス最適化と高度な機能"
---

# Chapter 18: Realm実装 Part 2 - パフォーマンス最適化と高度な機能

## 18.1 マイグレーション

### 18.1.1 基本的なマイグレーション

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

### 18.1.2 データ変換を伴うマイグレーション

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

## 18.2 スレッド管理

### 18.2.1 バックグラウンド処理

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

### 18.2.2 バッチ処理

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

## 18.3 SwiftUIとの統合

### 18.3.1 @ObservedRealmObjectの使用

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

### 18.3.2 @ObservedResultsの使用

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

### 18.3.3 フィルタリングとソート

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

### 18.3.4 セクション分け

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

## 18.4 パフォーマンス最適化

### 18.4.1 クエリの最適化

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

### 18.4.2 メモリ管理

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

### 18.4.3 書き込みの最適化

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

## 18.5 テスト

### 18.5.1 ユニットテスト

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

### 18.5.2 モックとスタブ

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

## 18.6 ベストプラクティス

### 18.6.1 設計原則

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

### 18.6.2 エラーハンドリング

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

### 18.6.3 インデックスの活用

```swift
class IndexedUser: Object {
    @Persisted(primaryKey: true) var id: UUID
    @Persisted(indexed: true) var email: String  // インデックスを設定
    @Persisted(indexed: true) var username: String
    @Persisted var firstName: String?
    @Persisted var lastName: String?

    convenience init(username: String, email: String) {
        self.init()
        self.id = UUID()
        self.username = username
        self.email = email
    }
}

// インデックスを活用した高速検索
class IndexedQueryManager {
    private let realm: Realm

    init(realm: Realm = try! Realm()) {
        self.realm = realm
    }

    // インデックスされたプロパティでの検索は高速
    func findByEmail(_ email: String) -> IndexedUser? {
        return realm.objects(IndexedUser.self)
            .filter("email == %@", email)
            .first
    }

    func findByUsername(_ username: String) -> IndexedUser? {
        return realm.objects(IndexedUser.self)
            .filter("username == %@", username)
            .first
    }
}
```

### 18.6.4 データの整合性

```swift
class DataIntegrityManager {
    private let realm: Realm

    init(realm: Realm = try! Realm()) {
        self.realm = realm
    }

    // トランザクション内で関連データをまとめて更新
    func deleteUserWithRelatedData(_ user: User) throws {
        try realm.write {
            // ユーザーの投稿を削除
            realm.delete(user.posts)

            // ユーザーのコメントを削除
            realm.delete(user.comments)

            // フォロー関係を解除
            for follower in user.followers {
                if let index = follower.following.index(of: user) {
                    follower.following.remove(at: index)
                }
            }

            for following in user.following {
                if let index = following.followers.index(of: user) {
                    following.followers.remove(at: index)
                }
            }

            // ユーザーを削除
            realm.delete(user)
        }
    }

    // カスケード削除
    func deletePostWithComments(_ post: Post) throws {
        try realm.write {
            // コメントの返信を再帰的に削除
            for comment in post.comments {
                deleteCommentRecursively(comment)
            }

            // 投稿を削除
            realm.delete(post)
        }
    }

    private func deleteCommentRecursively(_ comment: Comment) {
        for reply in comment.replies {
            deleteCommentRecursively(reply)
        }
        realm.delete(comment)
    }
}
```

## 18.7 高度なテクニック

### 18.7.1 計算プロパティとインデックス

```swift
class OptimizedPost: Object {
    @Persisted(primaryKey: true) var id: UUID
    @Persisted var title: String
    @Persisted var content: String
    @Persisted var likeCount: Int
    @Persisted var viewCount: Int
    @Persisted var commentCount: Int  // キャッシュ

    // 計算プロパティよりキャッシュを使用
    func updateCommentCount() {
        // コメント数を定期的に更新
        commentCount = comments.count
    }
}
```

### 18.7.2 部分的なデータ読み込み

```swift
class LazyLoadingManager {
    private let realm: Realm

    init(realm: Realm = try! Realm()) {
        self.realm = realm
    }

    // 必要なプロパティのみを読み込む
    func fetchUserSummaries() -> [(id: UUID, username: String, postCount: Int)] {
        let users = realm.objects(User.self)

        return users.map { user in
            (
                id: user.id,
                username: user.username,
                postCount: user.posts.count
            )
        }
    }

    // ページネーションとの組み合わせ
    func fetchPostsPaged(page: Int, pageSize: Int) -> [Post] {
        let results = realm.objects(Post.self)
            .sorted(byKeyPath: "createdAt", ascending: false)

        let start = page * pageSize
        let end = min(start + pageSize, results.count)

        guard start < results.count else { return [] }

        return Array(results[start..<end])
    }
}
```

### 18.7.3 複雑な集計クエリ

```swift
class AggregationManager {
    private let realm: Realm

    init(realm: Realm = try! Realm()) {
        self.realm = realm
    }

    // 統計情報の取得
    func getUserStatistics() -> UserStatistics {
        let users = realm.objects(User.self)

        let totalUsers = users.count
        let activeUsers = users.filter("isActive == true").count
        let totalPosts = users.sum(ofProperty: "posts.@count") as Int
        let averagePostsPerUser = totalUsers > 0 ? Double(totalPosts) / Double(totalUsers) : 0

        return UserStatistics(
            totalUsers: totalUsers,
            activeUsers: activeUsers,
            totalPosts: totalPosts,
            averagePostsPerUser: averagePostsPerUser
        )
    }

    // グループ別の集計
    func getPostsByMonth() -> [String: Int] {
        let posts = realm.objects(Post.self)
        var postsByMonth: [String: Int] = [:]

        let formatter = DateFormatter()
        formatter.dateFormat = "yyyy-MM"

        for post in posts {
            let month = formatter.string(from: post.createdAt)
            postsByMonth[month, default: 0] += 1
        }

        return postsByMonth
    }
}

struct UserStatistics {
    let totalUsers: Int
    let activeUsers: Int
    let totalPosts: Int
    let averagePostsPerUser: Double
}
```

## 18.8 トラブルシューティング

### 18.8.1 一般的な問題と解決策

```swift
class RealmTroubleshooting {
    // 問題: "Realm accessed from incorrect thread"
    // 解決: ThreadSafeReferenceを使用
    func fixThreadIssue() {
        let realm = try! Realm()
        let user = realm.objects(User.self).first!
        let userRef = ThreadSafeReference(to: user)

        DispatchQueue.global().async {
            let backgroundRealm = try! Realm()
            guard let user = backgroundRealm.resolve(userRef) else { return }
            print(user.username)
        }
    }

    // 問題: "Object has been deleted or invalidated"
    // 解決: isInvalidatedをチェック
    func checkInvalidated(user: User) {
        guard !user.isInvalidated else {
            print("User has been deleted")
            return
        }

        print(user.username)
    }

    // 問題: "Write transaction already in progress"
    // 解決: ネストしたwriteを避ける
    func avoidNestedWrites() throws {
        let realm = try! Realm()

        try realm.write {
            let user = User(username: "test", email: "test@example.com")
            realm.add(user)

            // ❌ 悪い例: ネストしたwrite
            // try realm.write { }

            // ✅ 良い例: 同じトランザクション内で処理
            let post = Post(title: "Post", content: "Content")
            user.posts.append(post)
        }
    }

    // 問題: メモリリーク
    // 解決: 通知トークンを適切に管理
    class ViewModel: ObservableObject {
        private var notificationToken: NotificationToken?

        deinit {
            notificationToken?.invalidate()
        }

        func observeChanges() {
            let realm = try! Realm()
            let results = realm.objects(User.self)

            notificationToken = results.observe { changes in
                // 処理
            }
        }
    }
}
```

### 18.8.2 デバッグとロギング

```swift
class RealmDebugger {
    static func enableDetailedLogging() {
        // Realmの詳細ログを有効化
        #if DEBUG
        var config = Realm.Configuration.defaultConfiguration
        config.shouldCompactOnLaunch = { totalBytes, usedBytes in
            let ratio = Double(usedBytes) / Double(totalBytes)
            print("Realm file size: \(totalBytes), used: \(usedBytes), ratio: \(ratio)")
            return ratio < 0.5
        }
        Realm.Configuration.defaultConfiguration = config
        #endif
    }

    static func printRealmInfo() {
        do {
            let realm = try Realm()
            print("Realm file path: \(realm.configuration.fileURL!)")
            print("Realm schema version: \(realm.configuration.schemaVersion)")
            print("Object types: \(realm.schema.objectSchema.map { $0.className })")
        } catch {
            print("Failed to access Realm: \(error)")
        }
    }

    static func printObjectCounts() {
        let realm = try! Realm()
        print("Users: \(realm.objects(User.self).count)")
        print("Posts: \(realm.objects(Post.self).count)")
        print("Comments: \(realm.objects(Comment.self).count)")
        print("Tags: \(realm.objects(Tag.self).count)")
    }
}
```

## 18.9 まとめ

この章では、Realmの高度な機能とパフォーマンス最適化について学びました。

### 重要なポイント

1. **マイグレーション**
   - スキーマバージョン管理
   - データ変換戦略

2. **スレッドセーフ**
   - ThreadSafeReference
   - バックグラウンド処理

3. **SwiftUI統合**
   - @ObservedRealmObject
   - @ObservedResults

4. **パフォーマンス最適化**
   - 効率的なクエリ
   - メモリ管理
   - バッチ処理

5. **テスト**
   - In-memory Realm
   - モックとスタブ

6. **ベストプラクティス**
   - Repositoryパターン
   - エラーハンドリング
   - データ整合性

### 次のステップ

Realmの実装を学んだ後は、以下のトピックに進むことをお勧めします：

- アーキテクチャパターン（Clean Architecture、MVVM）
- ネットワーキングとの統合
- オフラインファースト戦略
- データ同期とコンフリクト解決
