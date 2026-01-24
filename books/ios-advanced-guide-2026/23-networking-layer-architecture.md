---
title: "ネットワーク層アーキテクチャ"
---

# ネットワーク層アーキテクチャ

スケーラブルなアプリケーションには、適切に設計されたネットワーク層が不可欠です。本章では、保守性の高いネットワーク層アーキテクチャを解説します。

## レイヤー構造

```plaintext
Presentation Layer (ViewModel)
       ↓
Use Case Layer
       ↓
Repository Interface
       ↓
Repository Implementation
       ↓
API Client
       ↓
Network Manager (URLSession / Alamofire)
```

## Endpointパターン

```swift
protocol Endpoint {
    var baseURL: String { get }
    var path: String { get }
    var method: HTTPMethod { get }
    var headers: [String: String]? { get }
    var parameters: [String: Any]? { get }
}

enum UserEndpoint: Endpoint {
    case getUser(id: Int)
    case createUser(CreateUserRequest)
    case updateUser(id: Int, UpdateUserRequest)

    var path: String {
        switch self {
        case .getUser(let id):
            return "/users/\(id)"
        case .createUser, .updateUser:
            return "/users"
        }
    }

    var method: HTTPMethod {
        switch self {
        case .getUser: return .get
        case .createUser: return .post
        case .updateUser: return .put
        }
    }
}
```

## Repositoryパターン

```swift
protocol UserRepository {
    func fetchUser(id: Int) async throws -> User
    func createUser(_ request: CreateUserRequest) async throws -> User
}

class UserRepositoryImpl: UserRepository {
    private let apiClient: APIClient

    init(apiClient: APIClient = .shared) {
        self.apiClient = apiClient
    }

    func fetchUser(id: Int) async throws -> User {
        try await apiClient.request(UserEndpoint.getUser(id: id))
    }

    func createUser(_ request: CreateUserRequest) async throws -> User {
        try await apiClient.request(UserEndpoint.createUser(request))
    }
}
```

## UseCaseパターン

```swift
protocol GetUserUseCase {
    func execute(id: Int) async throws -> User
}

class GetUserUseCaseImpl: GetUserUseCase {
    private let repository: UserRepository

    init(repository: UserRepository) {
        self.repository = repository
    }

    func execute(id: Int) async throws -> User {
        try await repository.fetchUser(id: id)
    }
}
```

## まとめ

適切なアーキテクチャにより、テストが容易で保守性の高いコードが実現できます。次章では、データ永続化戦略について解説します。
