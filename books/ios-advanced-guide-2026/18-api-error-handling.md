---
title: "APIエラーハンドリング"
---

# APIエラーハンドリング

API通信では様々なエラーが発生します。本章では、堅牢なエラーハンドリングの実装方法を解説します。

## エラーの分類

```swift
enum APIError: Error {
    case networkError(Error)
    case httpError(statusCode: Int, data: Data?)
    case decodingError(Error)
    case invalidResponse
    case serverError(message: String)
}
```

## エラーレスポンスの処理

```swift
struct ErrorResponse: Decodable {
    let code: String
    let message: String
    let details: [String]?
}

func handleAPIError(_ error: Error, data: Data?) -> APIError {
    if let data = data,
       let errorResponse = try? JSONDecoder().decode(ErrorResponse.self, from: data) {
        return .serverError(message: errorResponse.message)
    }
    return .networkError(error)
}
```

## リトライロジック

```swift
func fetchWithRetry<T>(maxRetries: Int = 3, operation: @escaping () async throws -> T) async throws -> T {
    var lastError: Error?

    for attempt in 0..<maxRetries {
        do {
            return try await operation()
        } catch {
            lastError = error
            if attempt < maxRetries - 1 {
                try await Task.sleep(nanoseconds: UInt64(pow(2.0, Double(attempt)) * 1_000_000_000))
            }
        }
    }

    throw lastError!
}
```

## まとめ

適切なエラーハンドリングにより、ユーザー体験が向上し、デバッグが容易になります。次章では、Core Dataの基礎について解説します。
