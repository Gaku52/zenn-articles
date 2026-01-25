---
title: "ネットワーク層アーキテクチャ"
---

# ネットワーク層アーキテクチャ

スケーラブルで保守性の高いアプリケーションには、適切に設計されたネットワーク層が不可欠です。本章では、レイヤー分離、プロトコル指向設計、依存性注入を活用した、テスト容易性の高いネットワーク層アーキテクチャを解説します。

## ネットワーク層の設計原則

効果的なネットワーク層は、以下の原則に基づいて設計されるべきです：

1. **関心の分離（Separation of Concerns）**: 各レイヤーは単一の責任を持つ
2. **依存性逆転の原則（Dependency Inversion Principle）**: 具象ではなく抽象に依存する
3. **テスタビリティ**: 各コンポーネントが独立してテスト可能
4. **再利用性**: コードの重複を避け、再利用可能なコンポーネントを作成
5. **拡張性**: 新しい機能の追加が容易

## レイヤー構造

推奨されるネットワーク層のアーキテクチャは以下の通りです：

```plaintext
┌─────────────────────────────────┐
│  Presentation Layer (ViewModel) │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│      Use Case Layer             │
│  (ビジネスロジック)              │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│    Repository Interface         │
│  (プロトコル定義)                │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│  Repository Implementation      │
│  (データソースの抽象化)          │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│       API Client                │
│  (HTTP通信の実装)                │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│    Network Manager              │
│  (URLSession / Alamofire)       │
└─────────────────────────────────┘
```

## Endpointパターン

Endpointパターンを使うことで、APIエンドポイントを型安全に定義できます。

### Endpoint定義

```swift
enum HTTPMethod: String {
    case get = "GET"
    case post = "POST"
    case put = "PUT"
    case delete = "DELETE"
    case patch = "PATCH"
}

protocol Endpoint {
    var baseURL: String { get }
    var path: String { get }
    var method: HTTPMethod { get }
    var headers: [String: String]? { get }
    var parameters: [String: Any]? { get }
    var body: Data? { get }
}

extension Endpoint {
    var baseURL: String {
        "https://api.example.com"
    }

    var headers: [String: String]? {
        [
            "Content-Type": "application/json",
            "Accept": "application/json"
        ]
    }

    var parameters: [String: Any]? {
        nil
    }

    var body: Data? {
        nil
    }

    func asURLRequest() throws -> URLRequest {
        guard let url = URL(string: baseURL + path) else {
            throw NetworkError.invalidURL
        }

        var request = URLRequest(url: url)
        request.httpMethod = method.rawValue
        request.allHTTPHeaderFields = headers
        request.httpBody = body

        if let parameters = parameters {
            var components = URLComponents(url: url, resolvingAgainstBaseURL: false)
            components?.queryItems = parameters.map { URLQueryItem(name: $0.key, value: "\($0.value)") }
            request.url = components?.url
        }

        return request
    }
}
```

### 具体的なEndpoint実装

```swift
struct CreateUserRequest: Codable {
    let name: String
    let email: String
    let age: Int
}

struct UpdateUserRequest: Codable {
    let name: String?
    let email: String?
    let age: Int?
}

enum UserEndpoint: Endpoint {
    case getUser(id: Int)
    case getUsers(page: Int, limit: Int)
    case createUser(CreateUserRequest)
    case updateUser(id: Int, UpdateUserRequest)
    case deleteUser(id: Int)

    var path: String {
        switch self {
        case .getUser(let id):
            return "/users/\(id)"
        case .getUsers:
            return "/users"
        case .createUser:
            return "/users"
        case .updateUser(let id, _):
            return "/users/\(id)"
        case .deleteUser(let id):
            return "/users/\(id)"
        }
    }

    var method: HTTPMethod {
        switch self {
        case .getUser, .getUsers:
            return .get
        case .createUser:
            return .post
        case .updateUser:
            return .put
        case .deleteUser:
            return .delete
        }
    }

    var parameters: [String: Any]? {
        switch self {
        case .getUsers(let page, let limit):
            return ["page": page, "limit": limit]
        default:
            return nil
        }
    }

    var body: Data? {
        switch self {
        case .createUser(let request):
            return try? JSONEncoder().encode(request)
        case .updateUser(_, let request):
            return try? JSONEncoder().encode(request)
        default:
            return nil
        }
    }
}
```

## API Client実装

API Clientは、HTTPリクエストの送信とレスポンスのデコードを担当します。

```swift
enum NetworkError: Error {
    case invalidURL
    case requestFailed(Error)
    case invalidResponse
    case statusCode(Int)
    case decodingFailed(Error)
    case noData
}

protocol APIClientProtocol {
    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T
}

class APIClient: APIClientProtocol {
    static let shared = APIClient()

    private let session: URLSession
    private let decoder: JSONDecoder

    init(session: URLSession = .shared, decoder: JSONDecoder = JSONDecoder()) {
        self.session = session
        self.decoder = decoder
        self.decoder.keyDecodingStrategy = .convertFromSnakeCase
        self.decoder.dateDecodingStrategy = .iso8601
    }

    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        let urlRequest = try endpoint.asURLRequest()

        do {
            let (data, response) = try await session.data(for: urlRequest)

            guard let httpResponse = response as? HTTPURLResponse else {
                throw NetworkError.invalidResponse
            }

            guard (200...299).contains(httpResponse.statusCode) else {
                throw NetworkError.statusCode(httpResponse.statusCode)
            }

            do {
                let decoded = try decoder.decode(T.self, from: data)
                return decoded
            } catch {
                throw NetworkError.decodingFailed(error)
            }
        } catch let error as NetworkError {
            throw error
        } catch {
            throw NetworkError.requestFailed(error)
        }
    }
}
```

## Repositoryパターン

Repositoryパターンにより、データソース（API、データベース、キャッシュ）を抽象化し、ビジネスロジックから分離します。

### Repositoryプロトコル

```swift
struct User: Codable, Identifiable {
    let id: Int
    let name: String
    let email: String
    let age: Int
    let createdAt: Date
}

protocol UserRepository {
    func fetchUser(id: Int) async throws -> User
    func fetchUsers(page: Int, limit: Int) async throws -> [User]
    func createUser(_ request: CreateUserRequest) async throws -> User
    func updateUser(id: Int, _ request: UpdateUserRequest) async throws -> User
    func deleteUser(id: Int) async throws
}
```

### Repository実装

```swift
class UserRepositoryImpl: UserRepository {
    private let apiClient: APIClientProtocol

    init(apiClient: APIClientProtocol = APIClient.shared) {
        self.apiClient = apiClient
    }

    func fetchUser(id: Int) async throws -> User {
        try await apiClient.request(UserEndpoint.getUser(id: id))
    }

    func fetchUsers(page: Int, limit: Int) async throws -> [User] {
        let response: UsersResponse = try await apiClient.request(
            UserEndpoint.getUsers(page: page, limit: limit)
        )
        return response.users
    }

    func createUser(_ request: CreateUserRequest) async throws -> User {
        try await apiClient.request(UserEndpoint.createUser(request))
    }

    func updateUser(id: Int, _ request: UpdateUserRequest) async throws -> User {
        try await apiClient.request(UserEndpoint.updateUser(id: id, request))
    }

    func deleteUser(id: Int) async throws {
        let _: EmptyResponse = try await apiClient.request(UserEndpoint.deleteUser(id: id))
    }
}

struct UsersResponse: Codable {
    let users: [User]
    let total: Int
    let page: Int
    let limit: Int
}

struct EmptyResponse: Codable {}
```

## UseCaseパターン

UseCaseは、ビジネスロジックをカプセル化し、ViewModelから複雑なロジックを分離します。

### UseCaseプロトコル

```swift
protocol GetUserUseCase {
    func execute(id: Int) async throws -> User
}

protocol GetUsersUseCase {
    func execute(page: Int, limit: Int) async throws -> [User]
}

protocol CreateUserUseCase {
    func execute(name: String, email: String, age: Int) async throws -> User
}
```

### UseCase実装

```swift
class GetUserUseCaseImpl: GetUserUseCase {
    private let repository: UserRepository

    init(repository: UserRepository) {
        self.repository = repository
    }

    func execute(id: Int) async throws -> User {
        try await repository.fetchUser(id: id)
    }
}

class CreateUserUseCaseImpl: CreateUserUseCase {
    private let repository: UserRepository
    private let validator: UserValidator

    init(repository: UserRepository, validator: UserValidator = UserValidator()) {
        self.repository = repository
        self.validator = validator
    }

    func execute(name: String, email: String, age: Int) async throws -> User {
        // バリデーション
        try validator.validate(name: name, email: email, age: age)

        // リクエスト作成
        let request = CreateUserRequest(name: name, email: email, age: age)

        // API呼び出し
        return try await repository.createUser(request)
    }
}

struct UserValidator {
    func validate(name: String, email: String, age: Int) throws {
        guard !name.isEmpty else {
            throw ValidationError.emptyName
        }

        guard email.contains("@") else {
            throw ValidationError.invalidEmail
        }

        guard age >= 0 && age <= 150 else {
            throw ValidationError.invalidAge
        }
    }
}

enum ValidationError: Error {
    case emptyName
    case invalidEmail
    case invalidAge
}
```

## 依存性注入（DI）

DIコンテナを使うことで、依存関係を一元管理し、テストを容易にします。

### DIコンテナ

```swift
class DIContainer {
    static let shared = DIContainer()

    // MARK: - API Client
    lazy var apiClient: APIClientProtocol = {
        APIClient()
    }()

    // MARK: - Repositories
    lazy var userRepository: UserRepository = {
        UserRepositoryImpl(apiClient: apiClient)
    }()

    // MARK: - Use Cases
    lazy var getUserUseCase: GetUserUseCase = {
        GetUserUseCaseImpl(repository: userRepository)
    }()

    lazy var getUsersUseCase: GetUsersUseCase = {
        GetUsersUseCaseImpl(repository: userRepository)
    }()

    lazy var createUserUseCase: CreateUserUseCase = {
        CreateUserUseCaseImpl(repository: userRepository)
    }()

    // テスト用のコンテナ
    static func mock() -> DIContainer {
        let container = DIContainer()
        container.apiClient = MockAPIClient()
        container.userRepository = MockUserRepository()
        return container
    }
}
```

## モック実装によるテスト

プロトコルベースの設計により、簡単にモックを作成できます。

### MockRepository

```swift
class MockUserRepository: UserRepository {
    var fetchUserResult: Result<User, Error>?
    var fetchUsersResult: Result<[User], Error>?
    var createUserResult: Result<User, Error>?

    func fetchUser(id: Int) async throws -> User {
        guard let result = fetchUserResult else {
            fatalError("fetchUserResult not set")
        }
        return try result.get()
    }

    func fetchUsers(page: Int, limit: Int) async throws -> [User] {
        guard let result = fetchUsersResult else {
            fatalError("fetchUsersResult not set")
        }
        return try result.get()
    }

    func createUser(_ request: CreateUserRequest) async throws -> User {
        guard let result = createUserResult else {
            fatalError("createUserResult not set")
        }
        return try result.get()
    }

    func updateUser(id: Int, _ request: UpdateUserRequest) async throws -> User {
        fatalError("Not implemented")
    }

    func deleteUser(id: Int) async throws {
        // No-op
    }
}
```

### UseCaseのテスト

```swift
class GetUserUseCaseTests: XCTestCase {
    func testExecute_Success() async throws {
        // Arrange
        let mockRepository = MockUserRepository()
        let expectedUser = User(
            id: 1,
            name: "John Doe",
            email: "john@example.com",
            age: 30,
            createdAt: Date()
        )
        mockRepository.fetchUserResult = .success(expectedUser)

        let useCase = GetUserUseCaseImpl(repository: mockRepository)

        // Act
        let user = try await useCase.execute(id: 1)

        // Assert
        XCTAssertEqual(user.id, expectedUser.id)
        XCTAssertEqual(user.name, expectedUser.name)
    }

    func testExecute_Failure() async {
        // Arrange
        let mockRepository = MockUserRepository()
        mockRepository.fetchUserResult = .failure(NetworkError.noData)

        let useCase = GetUserUseCaseImpl(repository: mockRepository)

        // Act & Assert
        do {
            _ = try await useCase.execute(id: 1)
            XCTFail("Expected error to be thrown")
        } catch {
            XCTAssertTrue(error is NetworkError)
        }
    }
}
```

## SwiftUIとの統合

ViewModelでUseCaseを使用し、SwiftUIのViewにデータを提供します。

### ViewModel

```swift
@MainActor
class UserListViewModel: ObservableObject {
    @Published var users: [User] = []
    @Published var isLoading = false
    @Published var errorMessage: String?

    private let getUsersUseCase: GetUsersUseCase

    init(getUsersUseCase: GetUsersUseCase = DIContainer.shared.getUsersUseCase) {
        self.getUsersUseCase = getUsersUseCase
    }

    func loadUsers(page: Int = 1, limit: Int = 20) async {
        isLoading = true
        errorMessage = nil

        do {
            users = try await getUsersUseCase.execute(page: page, limit: limit)
        } catch {
            errorMessage = "ユーザーの読み込みに失敗しました: \(error.localizedDescription)"
        }

        isLoading = false
    }
}
```

### SwiftUI View

```swift
struct UserListView: View {
    @StateObject private var viewModel = UserListViewModel()

    var body: some View {
        NavigationView {
            Group {
                if viewModel.isLoading {
                    ProgressView()
                } else if let errorMessage = viewModel.errorMessage {
                    VStack {
                        Text(errorMessage)
                            .foregroundColor(.red)
                        Button("再試行") {
                            Task {
                                await viewModel.loadUsers()
                            }
                        }
                    }
                } else {
                    List(viewModel.users) { user in
                        VStack(alignment: .leading) {
                            Text(user.name)
                                .font(.headline)
                            Text(user.email)
                                .font(.subheadline)
                                .foregroundColor(.secondary)
                        }
                    }
                }
            }
            .navigationTitle("ユーザー一覧")
            .task {
                await viewModel.loadUsers()
            }
        }
    }
}
```

## チェックリスト

ネットワーク層アーキテクチャのベストプラクティス：

- [ ] **レイヤー分離**: Presentation、UseCase、Repository、API Clientが明確に分離されている
- [ ] **プロトコル指向**: すべての主要コンポーネントがプロトコルで定義されている
- [ ] **Endpointパターン**: APIエンドポイントが型安全に定義されている
- [ ] **エラーハンドリング**: 適切なエラー型が定義され、各レイヤーで処理されている
- [ ] **依存性注入**: DIコンテナを使って依存関係を管理している
- [ ] **テスタビリティ**: すべてのコンポーネントがモック可能
- [ ] **SwiftUI統合**: ViewModelでUseCaseを使用し、Viewから分離されている
- [ ] **非同期処理**: async/awaitを活用した現代的な非同期処理

## まとめ

適切なネットワーク層アーキテクチャにより、テストが容易で保守性の高いコードが実現できます。レイヤー分離、プロトコル指向設計、依存性注入を活用することで、変更に強く、拡張しやすいコードベースを構築できます。

次章では、データ永続化戦略について解説します。

## 参考文献

- [Apple Developer Documentation - URLSession](https://developer.apple.com/documentation/foundation/urlsession)
- [Apple Developer Documentation - Concurrency](https://developer.apple.com/documentation/swift/concurrency)
- [Swift by Sundell - Networking in Swift](https://www.swiftbysundell.com/basics/networking/)
- [Combine with Swift - Networking](https://www.raywenderlich.com/books/combine-asynchronous-programming-with-swift)
- [Clean Architecture - Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
