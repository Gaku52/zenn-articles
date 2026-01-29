---
title: "API 通信アーキテクチャ: 型安全で保守性の高い設計"
---

# API 通信アーキテクチャ: 型安全で保守性の高い設計

前章では URLSession の基本的な使い方を学びました。この章では、それらの知識を基に、より保守性が高く、テストしやすい API 通信層のアーキテクチャを構築します。

## レイヤー構造の設計

まず、API 通信を以下の3つのレイヤーに分離します:

1. **Presentation Layer**: ViewModel や View
2. **Domain Layer**: ビジネスロジックとモデル
3. **Data Layer**: API 通信とデータ永続化

この設計により、関心の分離と依存性の逆転を実現できます。

## Endpoint パターン

API エンドポイントを型安全に表現するために、Endpoint プロトコルを定義します。

```swift
protocol Endpoint {
    var baseURL: String { get }
    var path: String { get }
    var method: HTTPMethod { get }
    var headers: [String: String]? { get }
    var queryItems: [URLQueryItem]? { get }
    var body: Data? { get }

    func makeRequest() throws -> URLRequest
}

enum HTTPMethod: String {
    case get = "GET"
    case post = "POST"
    case put = "PUT"
    case delete = "DELETE"
    case patch = "PATCH"
}
```

デフォルト実装を提供します:

```swift
extension Endpoint {
    var baseURL: String {
        "https://api.example.com"
    }

    var headers: [String: String]? {
        ["Content-Type": "application/json"]
    }

    var queryItems: [URLQueryItem]? {
        nil
    }

    var body: Data? {
        nil
    }

    func makeRequest() throws -> URLRequest {
        var components = URLComponents(string: baseURL + path)
        components?.queryItems = queryItems

        guard let url = components?.url else {
            throw NetworkError.invalidURL
        }

        var request = URLRequest(url: url)
        request.httpMethod = method.rawValue
        request.httpBody = body

        headers?.forEach { key, value in
            request.setValue(value, forHTTPHeaderField: key)
        }

        return request
    }
}
```

## 具体的な Endpoint の実装

User API のエンドポイントを定義します:

```swift
enum UserEndpoint: Endpoint {
    case getAllUsers
    case getUser(id: Int)
    case createUser(CreateUserRequest)
    case updateUser(id: Int, UpdateUserRequest)
    case deleteUser(id: Int)
    case searchUsers(query: String, page: Int)

    var path: String {
        switch self {
        case .getAllUsers:
            return "/users"
        case .getUser(let id), .updateUser(let id, _), .deleteUser(let id):
            return "/users/\(id)"
        case .createUser:
            return "/users"
        case .searchUsers:
            return "/users/search"
        }
    }

    var method: HTTPMethod {
        switch self {
        case .getAllUsers, .getUser, .searchUsers:
            return .get
        case .createUser:
            return .post
        case .updateUser:
            return .put
        case .deleteUser:
            return .delete
        }
    }

    var queryItems: [URLQueryItem]? {
        switch self {
        case .searchUsers(let query, let page):
            return [
                URLQueryItem(name: "q", value: query),
                URLQueryItem(name: "page", value: "\(page)")
            ]
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

## APIService レイヤー

API 通信を抽象化する APIService プロトコルを定義します:

```swift
protocol APIService {
    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T
}

class APIServiceImpl: APIService {
    private let session: URLSession
    private let decoder: JSONDecoder

    init(
        session: URLSession = .shared,
        decoder: JSONDecoder = .default
    ) {
        self.session = session
        self.decoder = decoder
    }

    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        let request = try endpoint.makeRequest()
        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }

        guard (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.from(statusCode: httpResponse.statusCode, data: data)
        }

        do {
            return try decoder.decode(T.self, from: data)
        } catch {
            throw NetworkError.decodingError(error)
        }
    }
}
```

## Repository パターン

ビジネスロジックから API 実装の詳細を隠蔽するために、Repository パターンを使用します。

```swift
protocol UserRepository {
    func fetchUser(id: Int) async throws -> User
    func fetchAllUsers() async throws -> [User]
    func createUser(_ request: CreateUserRequest) async throws -> User
    func updateUser(id: Int, _ request: UpdateUserRequest) async throws -> User
    func deleteUser(id: Int) async throws
}

class UserRepositoryImpl: UserRepository {
    private let apiService: APIService

    init(apiService: APIService) {
        self.apiService = apiService
    }

    func fetchUser(id: Int) async throws -> User {
        try await apiService.request(UserEndpoint.getUser(id: id))
    }

    func fetchAllUsers() async throws -> [User] {
        try await apiService.request(UserEndpoint.getAllUsers)
    }

    func createUser(_ request: CreateUserRequest) async throws -> User {
        try await apiService.request(UserEndpoint.createUser(request))
    }

    func updateUser(id: Int, _ request: UpdateUserRequest) async throws -> User {
        try await apiService.request(UserEndpoint.updateUser(id: id, request))
    }

    func deleteUser(id: Int) async throws {
        let _: EmptyResponse = try await apiService.request(UserEndpoint.deleteUser(id: id))
    }
}

struct EmptyResponse: Codable {}
```

## ViewModel での使用

Repository を ViewModel で使用します:

```swift
@MainActor
class UserListViewModel: ObservableObject {
    @Published var users: [User] = []
    @Published var isLoading = false
    @Published var error: Error?

    private let repository: UserRepository

    init(repository: UserRepository) {
        self.repository = repository
    }

    func loadUsers() async {
        isLoading = true
        defer { isLoading = false }

        do {
            users = try await repository.fetchAllUsers()
            error = nil
        } catch {
            self.error = error
            print("Failed to load users: \(error)")
        }
    }

    func createUser(name: String, email: String, password: String) async {
        isLoading = true
        defer { isLoading = false }

        do {
            let request = CreateUserRequest(
                name: name,
                email: email,
                password: password
            )
            let newUser = try await repository.createUser(request)
            users.insert(newUser, at: 0)
            error = nil
        } catch {
            self.error = error
            print("Failed to create user: \(error)")
        }
    }

    func deleteUser(_ user: User) async {
        isLoading = true
        defer { isLoading = false }

        do {
            try await repository.deleteUser(id: user.id)
            users.removeAll { $0.id == user.id }
            error = nil
        } catch {
            self.error = error
            print("Failed to delete user: \(error)")
        }
    }
}
```

## 依存性注入

テストしやすくするために、依存性注入を使用します:

```swift
// プロダクション環境
let productionRepository = UserRepositoryImpl(
    apiService: APIServiceImpl()
)
let viewModel = UserListViewModel(repository: productionRepository)

// テスト環境
class MockUserRepository: UserRepository {
    var users: [User] = []

    func fetchAllUsers() async throws -> [User] {
        users
    }

    func createUser(_ request: CreateUserRequest) async throws -> User {
        let user = User(
            id: users.count + 1,
            name: request.name,
            email: request.email,
            createdAt: Date()
        )
        users.append(user)
        return user
    }

    // その他のメソッド...
}

let testRepository = MockUserRepository()
let testViewModel = UserListViewModel(repository: testRepository)
```

## レスポンスの型安全な処理

API レスポンスを型安全に扱うためのパターンを紹介します:

```swift
// 成功レスポンス
struct APIResponse<T: Codable>: Codable {
    let data: T
    let message: String?
}

// ページネーション対応
struct PaginatedResponse<T: Codable>: Codable {
    let data: [T]
    let page: Int
    let totalPages: Int
    let totalCount: Int
}

// エラーレスポンス
struct ErrorResponse: Codable {
    let code: String
    let message: String
    let details: [String]?
}

extension APIServiceImpl {
    func requestWithWrapper<T: Decodable>(
        _ endpoint: Endpoint
    ) async throws -> T {
        let request = try endpoint.makeRequest()
        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }

        if (200...299).contains(httpResponse.statusCode) {
            let wrapper = try decoder.decode(APIResponse<T>.self, from: data)
            return wrapper.data
        } else {
            let errorResponse = try? decoder.decode(ErrorResponse.self, from: data)
            throw NetworkError.apiError(errorResponse)
        }
    }
}
```

## 環境別の設定

開発、ステージング、本番環境で異なる API エンドポイントを使用できるようにします:

```swift
enum Environment {
    case development
    case staging
    case production

    var baseURL: String {
        switch self {
        case .development:
            return "http://localhost:3000"
        case .staging:
            return "https://staging-api.example.com"
        case .production:
            return "https://api.example.com"
        }
    }
}

class AppConfig {
    static let shared = AppConfig()

    #if DEBUG
    let environment: Environment = .development
    #else
    let environment: Environment = .production
    #endif

    var baseURL: String {
        environment.baseURL
    }

    private init() {}
}

// Endpoint での使用
extension Endpoint {
    var baseURL: String {
        AppConfig.shared.baseURL
    }
}
```

## まとめ

この章では、保守性とテスタビリティを重視した API 通信アーキテクチャを構築しました:

- Endpoint パターンによる型安全なエンドポイント定義
- APIService による通信層の抽象化
- Repository パターンによるビジネスロジックの分離
- 依存性注入によるテスタビリティの向上
- 環境別設定の管理

次の章では、エラーハンドリングとリトライ戦略について詳しく学びます。

## 参考リソース

- [Protocol-Oriented Programming - Swift.org](https://docs.swift.org/swift-book/LanguageGuide/Protocols.html)
- [Dependency Injection in Swift](https://www.swiftbysundell.com/articles/dependency-injection-using-factories-in-swift/)
