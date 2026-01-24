---
title: "Alamofire高度な活用"
---

# Alamofire高度な活用

Alamofireは、SwiftでHTTP通信を簡潔に記述できる人気のライブラリです。本章では、Alamofireの高度な機能と実践的な使い方を解説します。

## Alamofireの導入

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/Alamofire/Alamofire.git", from: "5.8.0")
]
```

## 基本的な使用方法

```swift
import Alamofire

class APIClient {
    func fetchUsers() async throws -> [User] {
        try await AF.request("https://api.example.com/users")
            .validate()
            .serializingDecodable([User].self)
            .value
    }
}
```

## Requestインターセプター

全てのリクエストに共通処理を追加します。

```swift
class AuthInterceptor: RequestInterceptor {
    func adapt(_ urlRequest: URLRequest, for session: Session, completion: @escaping (Result<URLRequest, Error>) -> Void) {
        var urlRequest = urlRequest

        if let token = try? KeychainManager.shared.loadToken() {
            urlRequest.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }

        completion(.success(urlRequest))
    }

    func retry(_ request: Request, for session: Session, dueTo error: Error, completion: @escaping (RetryResult) -> Void) {
        guard let response = request.task?.response as? HTTPURLResponse, response.statusCode == 401 else {
            completion(.doNotRetryWithError(error))
            return
        }

        // トークンリフレッシュ後にリトライ
        completion(.retryWithDelay(1.0))
    }
}
```

## まとめ

Alamofireを活用することで、ネットワーク通信のコードが簡潔になり、開発効率が向上することが期待されます。次章では、APIエラーハンドリングについて解説します。
