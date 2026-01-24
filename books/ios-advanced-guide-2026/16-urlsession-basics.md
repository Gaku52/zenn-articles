---
title: "URLSession基礎"
---

# URLSession基礎

URLSessionは、iOSでHTTP通信を行うための標準APIです。本章では、URLSessionの基本的な使い方と実践的なパターンを解説します。

## URLSessionの概要

URLSessionは、以下の機能を提供します：

- データタスク（Data Transfer）
- ダウンロードタスク（File Download）
- アップロードタスク（File Upload）
- WebSocketタスク（Bidirectional Communication）

## 基本的なGETリクエスト

```swift
// 最もシンプルな実装
func fetchData() async throws -> Data {
    let url = URL(string: "https://api.example.com/users")!
    let (data, _) = try await URLSession.shared.data(from: url)
    return data
}
```

### レスポンスの検証

```swift
func fetchUser(id: Int) async throws -> User {
    let url = URL(string: "https://api.example.com/users/\(id)")!
    let (data, response) = try await URLSession.shared.data(from: url)
    
    guard let httpResponse = response as? HTTPURLResponse,
          (200...299).contains(httpResponse.statusCode) else {
        throw NetworkError.invalidResponse
    }
    
    return try JSONDecoder().decode(User.self, from: data)
}
```

## POSTリクエスト

```swift
func createUser(_ user: CreateUserRequest) async throws -> User {
    let url = URL(string: "https://api.example.com/users")!
    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.httpBody = try JSONEncoder().encode(user)
    
    let (data, response) = try await URLSession.shared.data(for: request)
    
    guard let httpResponse = response as? HTTPURLResponse,
          (200...299).contains(httpResponse.statusCode) else {
        throw NetworkError.invalidResponse
    }
    
    return try JSONDecoder().decode(User.self, from: data)
}
```

## URLSessionConfigurationの設定

```swift
class NetworkManager {
    static let shared = NetworkManager()
    
    private let session: URLSession
    
    private init() {
        let configuration = URLSessionConfiguration.default
        configuration.timeoutIntervalForRequest = 30
        configuration.timeoutIntervalForResource = 300
        configuration.waitsForConnectivity = true
        configuration.requestCachePolicy = .reloadIgnoringLocalCacheData
        
        self.session = URLSession(configuration: configuration)
    }
    
    func request<T: Decodable>(_ url: URL) async throws -> T {
        let (data, _) = try await session.data(from: url)
        return try JSONDecoder().decode(T.self, from: data)
    }
}
```

## エラーハンドリング

```swift
enum NetworkError: Error, LocalizedError {
    case invalidURL
    case invalidResponse
    case httpError(Int)
    case decodingError(Error)
    case networkUnavailable
    
    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "無効なURLです"
        case .invalidResponse:
            return "サーバーからの応答が無効です"
        case .httpError(let statusCode):
            return "HTTPエラー: \(statusCode)"
        case .decodingError(let error):
            return "データの解析に失敗しました: \(error.localizedDescription)"
        case .networkUnavailable:
            return "ネットワークに接続できません"
        }
    }
}
```

## まとめ

URLSessionの基本操作を理解することで、シンプルで効果的なネットワーク通信が実装できます。次章では、Alamofireを使った高度なAPI通信について解説します。
