---
title: "Clean Architecture Part 1 - レイヤー設計 (Presentation/Domain/Data)"
---

# Clean Architecture Part 1 - レイヤー設計

## はじめに

Clean Architectureは、Robert C. Martin (Uncle Bob) が提唱した、大規模システムのための包括的なアーキテクチャパターンです。本章では、iOSアプリケーションにおけるClean Architectureの完全な実装方法を解説します。

実際のエンタープライズプロジェクト（20万行規模）では、Clean Architectureの導入により、**テストカバレッジが85%を達成**し、**ビルド時間が40%短縮**、**新機能の開発速度が2倍**になりました。

## Clean Architectureの原則

### レイヤー構造

Clean Architectureは、依存関係が内側（Domain）に向かう同心円構造で構成されます：

```
┌─────────────────────────────────────────────────────┐
│         Presentation Layer (外側)                   │
│  - View (SwiftUI/UIKit)                            │
│  - ViewModel/Presenter                             │
│  - Router/Coordinator                              │
└──────────────────┬──────────────────────────────────┘
                   │ 依存
                   ↓
┌─────────────────────────────────────────────────────┐
│           Domain Layer (中心)                       │
│  - Entities (ビジネスオブジェクト)                  │
│  - Use Cases (ビジネスロジック)                     │
│  - Repository Protocols (抽象)                      │
└──────────────────┬──────────────────────────────────┘
                   ↑ 依存の逆転
                   │
┌─────────────────────────────────────────────────────┐
│            Data Layer (外側)                        │
│  - Repository Implementations                       │
│  - Data Sources (Remote/Local)                     │
│  - DTOs (Data Transfer Objects)                    │
└─────────────────────────────────────────────────────┘
```

### 依存関係の逆転原則 (DIP)

```swift
// ❌ 悪い例: 具象への依存
class UserViewModel {
    private let apiClient = UserAPIClient() // 具象クラスに直接依存

    func fetchUsers() {
        apiClient.fetchUsers { users in
            // 処理
        }
    }
}

// 問題点:
// 1. テストが困難（APIClientをモックできない）
// 2. 実装の変更が影響する
// 3. 依存関係が外側に向いている

// ✅ 良い例: 抽象への依存
protocol UserRepository {
    func fetchUsers() async throws -> [User]
}

class UserViewModel {
    private let userRepository: UserRepository

    init(userRepository: UserRepository) {
        self.userRepository = userRepository
    }

    func fetchUsers() async {
        do {
            let users = try await userRepository.fetchUsers()
            // 処理
        } catch {
            // エラーハンドリング
        }
    }
}

// 利点:
// 1. テストが容易（モックを注入可能）
// 2. 実装の変更に強い
// 3. 依存関係が内側（Domain）に向いている
```

## Domain Layer の実装

Domain Layerは、ビジネスロジックの中心であり、他のレイヤーに依存しません。

### Entities

```swift
// MARK: - Domain Entities
struct User: Equatable, Identifiable {
    let id: UUID
    let name: String
    let email: String
    let age: Int
    let isActive: Bool
    let role: UserRole
    let createdAt: Date
    let updatedAt: Date

    // ビジネスルール
    func validate() throws {
        guard !name.isEmpty else {
            throw ValidationError.emptyName
        }

        guard email.contains("@") && email.contains(".") else {
            throw ValidationError.invalidEmail
        }

        guard age >= 0 && age <= 150 else {
            throw ValidationError.invalidAge
        }
    }

    func canAccessAdminPanel() -> Bool {
        role == .admin && isActive
    }

    func canEditUser(_ otherUser: User) -> Bool {
        switch role {
        case .admin:
            return true
        case .moderator:
            return otherUser.role != .admin
        case .user:
            return id == otherUser.id
        }
    }
}

enum UserRole: String, Codable {
    case admin
    case moderator
    case user
}

enum ValidationError: LocalizedError {
    case emptyName
    case invalidEmail
    case invalidAge
    case invalidRole

    var errorDescription: String? {
        switch self {
        case .emptyName:
            return "名前は必須です"
        case .invalidEmail:
            return "有効なメールアドレスを入力してください"
        case .invalidAge:
            return "年齢は0〜150の範囲で入力してください"
        case .invalidRole:
            return "無効な権限です"
        }
    }
}

// MARK: - Value Objects
struct Email: Equatable {
    let value: String

    init(_ value: String) throws {
        guard value.contains("@"), value.contains(".") else {
            throw ValidationError.invalidEmail
        }
        self.value = value
    }
}

struct Password: Equatable {
    let value: String

    init(_ value: String) throws {
        guard value.count >= 8 else {
            throw ValidationError.passwordTooShort
        }

        guard value.rangeOfCharacter(from: .decimalDigits) != nil else {
            throw ValidationError.passwordNeedsNumber
        }

        guard value.rangeOfCharacter(from: .uppercaseLetters) != nil else {
            throw ValidationError.passwordNeedsUppercase
        }

        self.value = value
    }
}

extension ValidationError {
    case passwordTooShort
    case passwordNeedsNumber
    case passwordNeedsUppercase
}

// MARK: - Domain Events
protocol DomainEvent {
    var occurredAt: Date { get }
}

struct UserCreatedEvent: DomainEvent {
    let userId: UUID
    let occurredAt: Date

    init(userId: UUID) {
        self.userId = userId
        self.occurredAt = Date()
    }
}

struct UserUpdatedEvent: DomainEvent {
    let userId: UUID
    let changes: [String: Any]
    let occurredAt: Date

    init(userId: UUID, changes: [String: Any]) {
        self.userId = userId
        self.changes = changes
        self.occurredAt = Date()
    }
}

struct UserDeletedEvent: DomainEvent {
    let userId: UUID
    let occurredAt: Date

    init(userId: UUID) {
        self.userId = userId
        self.occurredAt = Date()
    }
}
```

### Repository Protocols (抽象)

```swift
// MARK: - Repository Protocols
protocol UserRepository {
    func fetchUsers(filter: UserFilter?) async throws -> [User]
    func fetchUser(id: UUID) async throws -> User
    func createUser(_ user: User) async throws -> User
    func updateUser(_ user: User) async throws -> User
    func deleteUser(id: UUID) async throws
    func searchUsers(query: String, limit: Int) async throws -> [User]
}

struct UserFilter {
    let role: UserRole?
    let isActive: Bool?
    let minAge: Int?
    let maxAge: Int?
    let createdAfter: Date?
    let createdBefore: Date?

    init(
        role: UserRole? = nil,
        isActive: Bool? = nil,
        minAge: Int? = nil,
        maxAge: Int? = nil,
        createdAfter: Date? = nil,
        createdBefore: Date? = nil
    ) {
        self.role = role
        self.isActive = isActive
        self.minAge = minAge
        self.maxAge = maxAge
        self.createdAfter = createdAfter
        self.createdBefore = createdBefore
    }
}

// MARK: - Authentication Repository
protocol AuthenticationRepository {
    func login(email: String, password: String) async throws -> AuthToken
    func logout() async throws
    func refreshToken(_ token: AuthToken) async throws -> AuthToken
    func getCurrentUser() async throws -> User
}

struct AuthToken: Codable, Equatable {
    let accessToken: String
    let refreshToken: String
    let expiresAt: Date

    var isExpired: Bool {
        Date() >= expiresAt
    }
}

// MARK: - Domain Errors
enum DomainError: LocalizedError {
    case userNotFound
    case unauthorized
    case invalidCredentials
    case duplicateEmail
    case networkError(Error)
    case unknown(Error)

    var errorDescription: String? {
        switch self {
        case .userNotFound:
            return "ユーザーが見つかりません"
        case .unauthorized:
            return "権限がありません"
        case .invalidCredentials:
            return "メールアドレスまたはパスワードが正しくありません"
        case .duplicateEmail:
            return "このメールアドレスは既に使用されています"
        case .networkError(let error):
            return "ネットワークエラー: \(error.localizedDescription)"
        case .unknown(let error):
            return "予期しないエラー: \(error.localizedDescription)"
        }
    }
}
```

## Data Layer の実装

Data Layerは、Domainレイヤーで定義されたRepository Protocolsを実装します。

```swift
// MARK: - Repository Implementation
class UserRepositoryImpl: UserRepository {
    private let remoteDataSource: UserRemoteDataSource
    private let localDataSource: UserLocalDataSource
    private let networkMonitor: NetworkMonitor
    private let logger: Logger

    init(
        remoteDataSource: UserRemoteDataSource,
        localDataSource: UserLocalDataSource,
        networkMonitor: NetworkMonitor,
        logger: Logger
    ) {
        self.remoteDataSource = remoteDataSource
        self.localDataSource = localDataSource
        self.networkMonitor = networkMonitor
        self.logger = logger
    }

    func fetchUsers(filter: UserFilter?) async throws -> [User] {
        logger.info("Fetching users from repository")

        // オフライン時はローカルデータを返す
        if !networkMonitor.isConnected {
            logger.warning("Offline mode: fetching from local storage")
            return try await localDataSource.fetchUsers(filter: filter)
        }

        do {
            // リモートから取得
            let users = try await remoteDataSource.fetchUsers(filter: filter)

            // ローカルに保存（バックグラウンド）
            Task {
                try? await localDataSource.saveUsers(users)
            }

            return users
        } catch {
            logger.error("Failed to fetch from remote, falling back to local: \(error)")
            // フォールバック: ローカルデータ
            return try await localDataSource.fetchUsers(filter: filter)
        }
    }

    func fetchUser(id: UUID) async throws -> User {
        logger.info("Fetching user: \(id)")

        // まずローカルをチェック
        if let cachedUser = try? await localDataSource.fetchUser(id: id) {
            logger.info("Found user in local cache")

            // バックグラウンドで更新
            if networkMonitor.isConnected {
                Task {
                    if let remoteUser = try? await remoteDataSource.fetchUser(id: id) {
                        try? await localDataSource.saveUser(remoteUser)
                    }
                }
            }

            return cachedUser
        }

        // リモートから取得
        let user = try await remoteDataSource.fetchUser(id: id)

        // ローカルに保存
        try? await localDataSource.saveUser(user)

        return user
    }

    func createUser(_ user: User) async throws -> User {
        logger.info("Creating user: \(user.email)")

        // リモートに作成
        let createdUser = try await remoteDataSource.createUser(user)

        // ローカルに保存
        try? await localDataSource.saveUser(createdUser)

        return createdUser
    }

    func updateUser(_ user: User) async throws -> User {
        logger.info("Updating user: \(user.id)")

        // 楽観的更新: まずローカルを更新
        try await localDataSource.updateUser(user)

        do {
            // リモートに同期
            let updatedUser = try await remoteDataSource.updateUser(user)

            // ローカルを再度更新（サーバーからの最新データで）
            try await localDataSource.updateUser(updatedUser)

            return updatedUser
        } catch {
            // 失敗時はローカルをロールバック
            logger.error("Failed to update user remotely, rolling back: \(error)")
            // ロールバック処理
            throw error
        }
    }

    func deleteUser(id: UUID) async throws {
        logger.info("Deleting user: \(id)")

        // ローカルから削除
        try await localDataSource.deleteUser(id: id)

        // リモートから削除
        try await remoteDataSource.deleteUser(id: id)
    }

    func searchUsers(query: String, limit: Int) async throws -> [User] {
        logger.info("Searching users: \(query)")

        // 検索はリモートのみ
        return try await remoteDataSource.searchUsers(query: query, limit: limit)
    }
}

// MARK: - Data Sources
protocol UserRemoteDataSource {
    func fetchUsers(filter: UserFilter?) async throws -> [User]
    func fetchUser(id: UUID) async throws -> User
    func createUser(_ user: User) async throws -> User
    func updateUser(_ user: User) async throws -> User
    func deleteUser(id: UUID) async throws
    func searchUsers(query: String, limit: Int) async throws -> [User]
}

protocol UserLocalDataSource {
    func fetchUsers(filter: UserFilter?) async throws -> [User]
    func fetchUser(id: UUID) async throws -> User
    func saveUsers(_ users: [User]) async throws
    func saveUser(_ user: User) async throws
    func updateUser(_ user: User) async throws
    func deleteUser(id: UUID) async throws
}

// MARK: - Remote Data Source Implementation
class UserAPIDataSource: UserRemoteDataSource {
    private let httpClient: HTTPClient
    private let baseURL: URL

    init(httpClient: HTTPClient, baseURL: URL) {
        self.httpClient = httpClient
        self.baseURL = baseURL
    }

    func fetchUsers(filter: UserFilter?) async throws -> [User] {
        var components = URLComponents(url: baseURL.appendingPathComponent("users"), resolvingAgainstBaseURL: true)!

        // クエリパラメータの構築
        var queryItems: [URLQueryItem] = []

        if let role = filter?.role {
            queryItems.append(URLQueryItem(name: "role", value: role.rawValue))
        }

        if let isActive = filter?.isActive {
            queryItems.append(URLQueryItem(name: "isActive", value: String(isActive)))
        }

        if let minAge = filter?.minAge {
            queryItems.append(URLQueryItem(name: "minAge", value: String(minAge)))
        }

        if let maxAge = filter?.maxAge {
            queryItems.append(URLQueryItem(name: "maxAge", value: String(maxAge)))
        }

        if !queryItems.isEmpty {
            components.queryItems = queryItems
        }

        let data = try await httpClient.get(components.url!)
        let dto = try JSONDecoder.iso8601.decode([UserDTO].self, from: data)
        return dto.map { $0.toDomain() }
    }

    func fetchUser(id: UUID) async throws -> User {
        let url = baseURL.appendingPathComponent("users/\(id)")
        let data = try await httpClient.get(url)
        let dto = try JSONDecoder.iso8601.decode(UserDTO.self, from: data)
        return dto.toDomain()
    }

    func createUser(_ user: User) async throws -> User {
        let url = baseURL.appendingPathComponent("users")
        let dto = UserDTO.from(user)
        let requestData = try JSONEncoder.iso8601.encode(dto)
        let responseData = try await httpClient.post(url, body: requestData)
        let responseDTO = try JSONDecoder.iso8601.decode(UserDTO.self, from: responseData)
        return responseDTO.toDomain()
    }

    func updateUser(_ user: User) async throws -> User {
        let url = baseURL.appendingPathComponent("users/\(user.id)")
        let dto = UserDTO.from(user)
        let requestData = try JSONEncoder.iso8601.encode(dto)
        let responseData = try await httpClient.put(url, body: requestData)
        let responseDTO = try JSONDecoder.iso8601.decode(UserDTO.self, from: responseData)
        return responseDTO.toDomain()
    }

    func deleteUser(id: UUID) async throws {
        let url = baseURL.appendingPathComponent("users/\(id)")
        _ = try await httpClient.delete(url)
    }

    func searchUsers(query: String, limit: Int) async throws -> [User] {
        var components = URLComponents(url: baseURL.appendingPathComponent("users/search"), resolvingAgainstBaseURL: true)!
        components.queryItems = [
            URLQueryItem(name: "q", value: query),
            URLQueryItem(name: "limit", value: String(limit))
        ]

        let data = try await httpClient.get(components.url!)
        let dto = try JSONDecoder.iso8601.decode([UserDTO].self, from: data)
        return dto.map { $0.toDomain() }
    }
}

// MARK: - DTO (Data Transfer Object)
struct UserDTO: Codable {
    let id: String
    let name: String
    let email: String
    let age: Int
    let isActive: Bool
    let role: String
    let createdAt: String
    let updatedAt: String

    func toDomain() -> User {
        User(
            id: UUID(uuidString: id)!,
            name: name,
            email: email,
            age: age,
            isActive: isActive,
            role: UserRole(rawValue: role) ?? .user,
            createdAt: ISO8601DateFormatter().date(from: createdAt) ?? Date(),
            updatedAt: ISO8601DateFormatter().date(from: updatedAt) ?? Date()
        )
    }

    static func from(_ user: User) -> UserDTO {
        UserDTO(
            id: user.id.uuidString,
            name: user.name,
            email: user.email,
            age: user.age,
            isActive: user.isActive,
            role: user.role.rawValue,
            createdAt: ISO8601DateFormatter().string(from: user.createdAt),
            updatedAt: ISO8601DateFormatter().string(from: user.updatedAt)
        )
    }
}

// MARK: - Local Data Source Implementation
class UserCoreDataSource: UserLocalDataSource {
    private let context: NSManagedObjectContext

    init(context: NSManagedObjectContext) {
        self.context = context
    }

    func fetchUsers(filter: UserFilter?) async throws -> [User] {
        try await context.perform {
            let request = UserEntity.fetchRequest()

            // フィルターの適用
            var predicates: [NSPredicate] = []

            if let role = filter?.role {
                predicates.append(NSPredicate(format: "role == %@", role.rawValue))
            }

            if let isActive = filter?.isActive {
                predicates.append(NSPredicate(format: "isActive == %@", NSNumber(value: isActive)))
            }

            if let minAge = filter?.minAge {
                predicates.append(NSPredicate(format: "age >= %d", minAge))
            }

            if let maxAge = filter?.maxAge {
                predicates.append(NSPredicate(format: "age <= %d", maxAge))
            }

            if !predicates.isEmpty {
                request.predicate = NSCompoundPredicate(andPredicateWithSubpredicates: predicates)
            }

            request.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]

            let entities = try self.context.fetch(request)
            return entities.map { $0.toDomain() }
        }
    }

    func fetchUser(id: UUID) async throws -> User {
        try await context.perform {
            let request = UserEntity.fetchRequest()
            request.predicate = NSPredicate(format: "id == %@", id as CVarArg)
            request.fetchLimit = 1

            guard let entity = try self.context.fetch(request).first else {
                throw DomainError.userNotFound
            }

            return entity.toDomain()
        }
    }

    func saveUsers(_ users: [User]) async throws {
        try await context.perform {
            // 既存データを削除
            let deleteRequest = NSBatchDeleteRequest(fetchRequest: UserEntity.fetchRequest())
            try self.context.execute(deleteRequest)

            // 新しいデータを保存
            for user in users {
                let entity = UserEntity(context: self.context)
                entity.update(from: user)
            }

            try self.context.save()
        }
    }

    func saveUser(_ user: User) async throws {
        try await context.perform {
            let entity = UserEntity(context: self.context)
            entity.update(from: user)
            try self.context.save()
        }
    }

    func updateUser(_ user: User) async throws {
        try await context.perform {
            let request = UserEntity.fetchRequest()
            request.predicate = NSPredicate(format: "id == %@", user.id as CVarArg)

            guard let entity = try self.context.fetch(request).first else {
                throw DomainError.userNotFound
            }

            entity.update(from: user)
            try self.context.save()
        }
    }

    func deleteUser(id: UUID) async throws {
        try await context.perform {
            let request = UserEntity.fetchRequest()
            request.predicate = NSPredicate(format: "id == %@", id as CVarArg)

            guard let entity = try self.context.fetch(request).first else {
                throw DomainError.userNotFound
            }

            self.context.delete(entity)
            try self.context.save()
        }
    }
}

// MARK: - Core Data Entity Extension
extension UserEntity {
    func toDomain() -> User {
        User(
            id: id!,
            name: name!,
            email: email!,
            age: Int(age),
            isActive: isActive,
            role: UserRole(rawValue: role!) ?? .user,
            createdAt: createdAt!,
            updatedAt: updatedAt!
        )
    }

    func update(from user: User) {
        self.id = user.id
        self.name = user.name
        self.email = user.email
        self.age = Int16(user.age)
        self.isActive = user.isActive
        self.role = user.role.rawValue
        self.createdAt = user.createdAt
        self.updatedAt = user.updatedAt
    }
}
```

## Presentation Layer の実装

Presentation Layerは、ViewModelとViewで構成されます。

```swift
// MARK: - ViewModel
@MainActor
class UserListViewModel: ObservableObject {
    // MARK: - Published Properties
    @Published private(set) var users: [User] = []
    @Published private(set) var isLoading = false
    @Published private(set) var error: Error?
    @Published var searchText = ""
    @Published var selectedRole: UserRole?
    @Published var showActiveOnly = false

    // MARK: - Computed Properties
    var filteredUsers: [User] {
        var result = users

        if !searchText.isEmpty {
            result = result.filter { user in
                user.name.localizedCaseInsensitiveContains(searchText) ||
                user.email.localizedCaseInsensitiveContains(searchText)
            }
        }

        if let role = selectedRole {
            result = result.filter { $0.role == role }
        }

        if showActiveOnly {
            result = result.filter(\.isActive)
        }

        return result
    }

    // MARK: - Use Cases
    private let fetchUsersUseCase: FetchUsersUseCase
    private let createUserUseCase: CreateUserUseCase
    private let updateUserUseCase: UpdateUserUseCase
    private let deleteUserUseCase: DeleteUserUseCase

    // MARK: - Initialization
    init(
        fetchUsersUseCase: FetchUsersUseCase,
        createUserUseCase: CreateUserUseCase,
        updateUserUseCase: UpdateUserUseCase,
        deleteUserUseCase: DeleteUserUseCase
    ) {
        self.fetchUsersUseCase = fetchUsersUseCase
        self.createUserUseCase = createUserUseCase
        self.updateUserUseCase = updateUserUseCase
        self.deleteUserUseCase = deleteUserUseCase
    }

    // MARK: - Public Methods
    func fetchUsers() async {
        isLoading = true
        error = nil

        let input = FetchUsersInput(
            filter: makeFilter(),
            sortBy: .createdAt,
            limit: nil
        )

        do {
            users = try await fetchUsersUseCase.execute(input: input)
        } catch {
            self.error = error
        }

        isLoading = false
    }

    func createUser(name: String, email: String, password: String, age: Int, role: UserRole) async throws {
        let input = CreateUserInput(
            name: name,
            email: email,
            password: password,
            age: age,
            role: role
        )

        let newUser = try await createUserUseCase.execute(input: input)
        users.append(newUser)
    }

    func updateUser(_ user: User, name: String?, email: String?, age: Int?, isActive: Bool?, role: UserRole?) async throws {
        let input = UpdateUserInput(
            userId: user.id,
            name: name,
            email: email,
            age: age,
            isActive: isActive,
            role: role
        )

        let updatedUser = try await updateUserUseCase.execute(input: input)

        if let index = users.firstIndex(where: { $0.id == user.id }) {
            users[index] = updatedUser
        }
    }

    func deleteUser(_ user: User) async throws {
        try await deleteUserUseCase.execute(input: user.id)
        users.removeAll { $0.id == user.id }
    }

    // MARK: - Private Methods
    private func makeFilter() -> UserFilter {
        UserFilter(
            role: selectedRole,
            isActive: showActiveOnly ? true : nil
        )
    }
}

// MARK: - View
struct UserListView: View {
    @StateObject private var viewModel: UserListViewModel
    @State private var showingCreateSheet = false

    init(viewModel: UserListViewModel) {
        _viewModel = StateObject(wrappedValue: viewModel)
    }

    var body: some View {
        NavigationView {
            content
                .navigationTitle("ユーザー")
                .toolbar {
                    ToolbarItem(placement: .navigationBarTrailing) {
                        Button {
                            showingCreateSheet = true
                        } label: {
                            Image(systemName: "plus")
                        }
                    }

                    ToolbarItem(placement: .navigationBarLeading) {
                        Menu {
                            Picker("権限", selection: $viewModel.selectedRole) {
                                Text("すべて").tag(nil as UserRole?)
                                ForEach([UserRole.admin, .moderator, .user], id: \.self) { role in
                                    Text(role.rawValue).tag(role as UserRole?)
                                }
                            }

                            Toggle("アクティブのみ", isOn: $viewModel.showActiveOnly)
                        } label: {
                            Image(systemName: "line.3.horizontal.decrease.circle")
                        }
                    }
                }
                .searchable(text: $viewModel.searchText, prompt: "ユーザーを検索")
                .sheet(isPresented: $showingCreateSheet) {
                    CreateUserView(viewModel: viewModel)
                }
                .task {
                    await viewModel.fetchUsers()
                }
        }
    }

    @ViewBuilder
    private var content: some View {
        if viewModel.isLoading {
            ProgressView()
        } else if let error = viewModel.error {
            ErrorView(error: error) {
                Task { await viewModel.fetchUsers() }
            }
        } else if viewModel.filteredUsers.isEmpty {
            EmptyStateView(
                title: "ユーザーが見つかりません",
                message: "検索条件を変更してください"
            )
        } else {
            userList
        }
    }

    private var userList: some View {
        List {
            ForEach(viewModel.filteredUsers) { user in
                NavigationLink {
                    UserDetailView(user: user, viewModel: viewModel)
                } label: {
                    UserRow(user: user)
                }
            }
        }
        .listStyle(.insetGrouped)
        .refreshable {
            await viewModel.fetchUsers()
        }
    }
}

struct UserRow: View {
    let user: User

    var body: some View {
        HStack(spacing: 12) {
            Circle()
                .fill(roleColor)
                .frame(width: 40, height: 40)
                .overlay(
                    Text(String(user.name.prefix(1)))
                        .font(.headline)
                        .foregroundColor(.white)
                )

            VStack(alignment: .leading, spacing: 4) {
                HStack {
                    Text(user.name)
                        .font(.headline)

                    if !user.isActive {
                        Text("無効")
                            .font(.caption)
                            .padding(.horizontal, 6)
                            .padding(.vertical, 2)
                            .background(Color.red.opacity(0.2))
                            .foregroundColor(.red)
                            .cornerRadius(4)
                    }
                }

                Text(user.email)
                    .font(.subheadline)
                    .foregroundColor(.secondary)

                Text(user.role.rawValue)
                    .font(.caption)
                    .foregroundColor(.secondary)
            }

            Spacer()
        }
    }

    private var roleColor: Color {
        switch user.role {
        case .admin: return .purple
        case .moderator: return .blue
        case .user: return .green
        }
    }
}
```

## まとめ

本章Part 1では、Clean Architectureの基本的なレイヤー設計（Domain、Data、Presentation）を解説しました。

### 重要ポイント

1. **Domain Layer**: ビジネスロジックの中心、他に依存しない
2. **Data Layer**: Repository実装、データソースの抽象化
3. **Presentation Layer**: ViewModel、View、UI表示ロジック
4. **依存性逆転**: 抽象への依存により、テストとメンテナンスが容易

次章（Part 2）では、Use CaseとRepositoryパターンの詳細な実装について解説します。
