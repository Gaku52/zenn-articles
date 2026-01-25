---
title: "APIエラーハンドリング"
---

# APIエラーハンドリング

API通信では様々なエラーが発生します。ネットワークの不安定性、サーバー側の障害、データフォーマットの不一致など、多岐にわたるエラーケースに対して適切に対処することで、ユーザー体験の向上とアプリケーションの安定性を確保できます。本章では、堅牢なエラーハンドリングの実装方法を解説します。

## エラーの分類と設計

### カスタムエラー型の定義

エラーを体系的に管理するために、カスタムエラー型を定義します。

```swift
import Foundation

enum APIError: Error {
    case networkError(underlying: Error)
    case httpError(statusCode: Int, data: Data?)
    case decodingError(underlying: Error)
    case invalidResponse
    case serverError(message: String, code: String?)
    case unauthorized
    case forbidden
    case notFound
    case timeout
    case noConnection
    case cancelled
}

extension APIError: LocalizedError {
    var errorDescription: String? {
        switch self {
        case .networkError(let error):
            return "ネットワークエラーが発生しました: \(error.localizedDescription)"
        case .httpError(let statusCode, _):
            return "HTTPエラー: \(statusCode)"
        case .decodingError(let error):
            return "データの解析に失敗しました: \(error.localizedDescription)"
        case .invalidResponse:
            return "無効なレスポンスを受信しました"
        case .serverError(let message, let code):
            if let code = code {
                return "サーバーエラー [\(code)]: \(message)"
            }
            return "サーバーエラー: \(message)"
        case .unauthorized:
            return "認証が必要です"
        case .forbidden:
            return "アクセスが拒否されました"
        case .notFound:
            return "リソースが見つかりません"
        case .timeout:
            return "通信がタイムアウトしました"
        case .noConnection:
            return "インターネット接続がありません"
        case .cancelled:
            return "リクエストがキャンセルされました"
        }
    }
}
```

### エラーレスポンスの定義

サーバーから返されるエラーレスポンスの構造を定義します。

```swift
struct ErrorResponse: Decodable {
    let code: String
    let message: String
    let details: [ErrorDetail]?
    let timestamp: Date?

    struct ErrorDetail: Decodable {
        let field: String?
        let message: String
    }
}
```

## HTTPステータスコード別処理

各HTTPステータスコードに応じた適切な処理を実装します。

```swift
import Foundation

class HTTPErrorHandler {
    static func handleHTTPError(statusCode: Int, data: Data?) -> APIError {
        // サーバーエラーレスポンスの解析を試行
        let errorResponse = data.flatMap { data in
            try? JSONDecoder().decode(ErrorResponse.self, from: data)
        }

        switch statusCode {
        case 400:
            return .serverError(
                message: errorResponse?.message ?? "不正なリクエストです",
                code: errorResponse?.code
            )
        case 401:
            return .unauthorized
        case 403:
            return .forbidden
        case 404:
            return .notFound
        case 408:
            return .timeout
        case 422:
            return .serverError(
                message: errorResponse?.message ?? "バリデーションエラーが発生しました",
                code: errorResponse?.code
            )
        case 429:
            return .serverError(
                message: "リクエスト数が上限に達しました。しばらくしてから再試行してください。",
                code: "RATE_LIMIT_EXCEEDED"
            )
        case 500...599:
            return .serverError(
                message: errorResponse?.message ?? "サーバーエラーが発生しました",
                code: errorResponse?.code
            )
        default:
            return .httpError(statusCode: statusCode, data: data)
        }
    }

    static func isRetryable(error: APIError) -> Bool {
        switch error {
        case .networkError, .timeout, .noConnection:
            return true
        case .httpError(let statusCode, _):
            return statusCode >= 500 || statusCode == 408 || statusCode == 429
        default:
            return false
        }
    }
}
```

## リトライ機構の実装

### 指数バックオフを使用したリトライ

エラー発生時に適切な間隔で再試行する機構を実装します。

```swift
import Foundation

class RetryHandler {
    private let maxRetries: Int
    private let baseDelay: TimeInterval
    private let maxDelay: TimeInterval

    init(maxRetries: Int = 3, baseDelay: TimeInterval = 1.0, maxDelay: TimeInterval = 30.0) {
        self.maxRetries = maxRetries
        self.baseDelay = baseDelay
        self.maxDelay = maxDelay
    }

    func execute<T>(
        operation: @escaping () async throws -> T,
        shouldRetry: @escaping (Error) -> Bool = { _ in true }
    ) async throws -> T {
        var lastError: Error?
        var attempt = 0

        while attempt < maxRetries {
            do {
                return try await operation()
            } catch {
                lastError = error

                // リトライ可能かチェック
                guard shouldRetry(error) else {
                    throw error
                }

                attempt += 1

                // 最大リトライ回数に達した場合は例外をスロー
                guard attempt < maxRetries else {
                    throw error
                }

                // 指数バックオフ計算
                let delay = calculateDelay(attempt: attempt)
                print("リトライ \(attempt)/\(maxRetries) - \(delay)秒後に再試行します")

                try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
            }
        }

        throw lastError ?? APIError.invalidResponse
    }

    private func calculateDelay(attempt: Int) -> TimeInterval {
        // 指数バックオフ: baseDelay * 2^(attempt - 1)
        let exponentialDelay = baseDelay * pow(2.0, Double(attempt - 1))

        // ジッター（±25%のランダム性）を追加
        let jitter = exponentialDelay * Double.random(in: 0.75...1.25)

        // 最大遅延時間を超えないようにする
        return min(jitter, maxDelay)
    }
}

// 使用例
class APIClient {
    private let retryHandler = RetryHandler(maxRetries: 3, baseDelay: 1.0)

    func fetchUsers() async throws -> [User] {
        try await retryHandler.execute(
            operation: {
                try await performFetchUsers()
            },
            shouldRetry: { error in
                guard let apiError = error as? APIError else {
                    return false
                }
                return HTTPErrorHandler.isRetryable(error: apiError)
            }
        )
    }

    private func performFetchUsers() async throws -> [User] {
        // 実際のAPI呼び出し
        // ...
        return []
    }
}
```

## サーキットブレーカーパターン

連続的なエラーが発生した場合に、一時的にリクエストを遮断するサーキットブレーカーパターンを実装します。

```swift
import Foundation

class CircuitBreaker {
    enum State {
        case closed      // 正常状態
        case open        // エラーが多発、リクエスト遮断
        case halfOpen    // 回復テスト中
    }

    private let failureThreshold: Int
    private let successThreshold: Int
    private let timeout: TimeInterval

    private var state: State = .closed
    private var failureCount = 0
    private var successCount = 0
    private var lastFailureTime: Date?

    private let queue = DispatchQueue(label: "com.app.circuitbreaker")

    init(failureThreshold: Int = 5, successThreshold: Int = 2, timeout: TimeInterval = 60) {
        self.failureThreshold = failureThreshold
        self.successThreshold = successThreshold
        self.timeout = timeout
    }

    func execute<T>(operation: @escaping () async throws -> T) async throws -> T {
        // 現在の状態をチェック
        let currentState = await checkAndUpdateState()

        switch currentState {
        case .open:
            throw APIError.serverError(
                message: "サービスが一時的に利用できません。しばらくしてから再試行してください。",
                code: "CIRCUIT_BREAKER_OPEN"
            )

        case .halfOpen, .closed:
            do {
                let result = try await operation()
                await recordSuccess()
                return result
            } catch {
                await recordFailure()
                throw error
            }
        }
    }

    private func checkAndUpdateState() async -> State {
        await queue.sync {
            // Open状態でタイムアウト経過した場合、HalfOpen状態に移行
            if state == .open,
               let lastFailure = lastFailureTime,
               Date().timeIntervalSince(lastFailure) >= timeout {
                state = .halfOpen
                successCount = 0
                failureCount = 0
            }

            return state
        }
    }

    private func recordSuccess() async {
        await queue.sync {
            successCount += 1
            failureCount = 0

            // HalfOpen状態で十分な成功があればClosed状態に戻る
            if state == .halfOpen && successCount >= successThreshold {
                state = .closed
                successCount = 0
            }
        }
    }

    private func recordFailure() async {
        await queue.sync {
            failureCount += 1
            successCount = 0
            lastFailureTime = Date()

            // 失敗が閾値を超えたらOpen状態に移行
            if failureCount >= failureThreshold {
                state = .open
            }
        }
    }

    func reset() {
        queue.sync {
            state = .closed
            failureCount = 0
            successCount = 0
            lastFailureTime = nil
        }
    }
}
```

## ユーザーへのエラー通知

### Alertによる通知

```swift
import SwiftUI

struct ErrorAlert: ViewModifier {
    @Binding var error: APIError?

    func body(content: Content) -> some View {
        content
            .alert("エラー", isPresented: .constant(error != nil)) {
                Button("OK") {
                    error = nil
                }
                if shouldShowRetryButton() {
                    Button("再試行") {
                        // リトライ処理
                    }
                }
            } message: {
                if let error = error {
                    Text(error.localizedDescription)
                }
            }
    }

    private func shouldShowRetryButton() -> Bool {
        guard let error = error else { return false }
        return HTTPErrorHandler.isRetryable(error: error)
    }
}

extension View {
    func errorAlert(error: Binding<APIError?>) -> some View {
        modifier(ErrorAlert(error: error))
    }
}
```

### Toastによる通知

```swift
import SwiftUI

struct Toast: View {
    let message: String
    let type: ToastType

    enum ToastType {
        case error
        case warning
        case success

        var backgroundColor: Color {
            switch self {
            case .error: return .red
            case .warning: return .orange
            case .success: return .green
            }
        }

        var icon: String {
            switch self {
            case .error: return "xmark.circle.fill"
            case .warning: return "exclamationmark.triangle.fill"
            case .success: return "checkmark.circle.fill"
            }
        }
    }

    var body: some View {
        HStack(spacing: 12) {
            Image(systemName: type.icon)
                .foregroundColor(.white)

            Text(message)
                .font(.subheadline)
                .foregroundColor(.white)

            Spacer()
        }
        .padding()
        .background(type.backgroundColor)
        .cornerRadius(8)
        .shadow(radius: 4)
        .padding(.horizontal)
    }
}

struct ToastModifier: ViewModifier {
    @Binding var toast: Toast?
    @State private var workItem: DispatchWorkItem?

    func body(content: Content) -> some View {
        content
            .overlay(
                ZStack {
                    if let toast = toast {
                        VStack {
                            toast
                            Spacer()
                        }
                        .transition(.move(edge: .top).combined(with: .opacity))
                    }
                }
                .animation(.spring(), value: toast)
            )
            .onChange(of: toast) { newValue in
                showToast()
            }
    }

    private func showToast() {
        guard toast != nil else { return }

        workItem?.cancel()

        let task = DispatchWorkItem {
            withAnimation {
                toast = nil
            }
        }

        workItem = task
        DispatchQueue.main.asyncAfter(deadline: .now() + 3, execute: task)
    }
}

extension View {
    func toast(toast: Binding<Toast?>) -> some View {
        modifier(ToastModifier(toast: toast))
    }
}
```

## エラーハンドリングの統合例

```swift
import SwiftUI

@MainActor
class UserListViewModel: ObservableObject {
    @Published var users: [User] = []
    @Published var isLoading = false
    @Published var error: APIError?
    @Published var toast: Toast?

    private let apiClient: APIClient
    private let retryHandler: RetryHandler
    private let circuitBreaker: CircuitBreaker

    init(
        apiClient: APIClient = APIClient(),
        retryHandler: RetryHandler = RetryHandler(),
        circuitBreaker: CircuitBreaker = CircuitBreaker()
    ) {
        self.apiClient = apiClient
        self.retryHandler = retryHandler
        self.circuitBreaker = circuitBreaker
    }

    func loadUsers() async {
        isLoading = true
        error = nil

        do {
            users = try await circuitBreaker.execute {
                try await self.retryHandler.execute {
                    try await self.apiClient.fetchUsers()
                }
            }
            toast = Toast(message: "ユーザー情報を取得しました", type: .success)
        } catch let apiError as APIError {
            error = apiError
            handleError(apiError)
        } catch {
            let apiError = APIError.networkError(underlying: error)
            self.error = apiError
            handleError(apiError)
        }

        isLoading = false
    }

    private func handleError(_ error: APIError) {
        // エラーの種類に応じた処理
        switch error {
        case .unauthorized:
            // 認証エラーの場合はログイン画面へ遷移
            NotificationCenter.default.post(name: .userDidLogout, object: nil)

        case .noConnection:
            // ネットワーク接続エラーの場合はToastで通知
            toast = Toast(message: "インターネット接続を確認してください", type: .warning)

        default:
            // その他のエラーはアラートで表示
            break
        }
    }
}
```

## チェックリスト

APIエラーハンドリングのベストプラクティスチェックリスト：

- [ ] カスタムエラー型を定義し、エラーを分類済み
- [ ] HTTPステータスコード別の処理を実装済み
- [ ] 指数バックオフを使用したリトライ機構を実装済み
- [ ] サーキットブレーカーパターンを実装済み
- [ ] ユーザーへの適切なエラー通知を実装済み
- [ ] ネットワーク接続エラーを検出する仕組みを実装済み
- [ ] リトライ可能なエラーと不可能なエラーを区別済み
- [ ] タイムアウト処理を適切に設定済み
- [ ] エラーログを記録する仕組みを実装済み
- [ ] 認証エラー時の自動ログアウト処理を実装済み

## まとめ

適切なエラーハンドリングにより、ユーザー体験が向上し、デバッグが容易になります。カスタムエラー型による体系的なエラー管理、指数バックオフを使用したリトライ機構、サーキットブレーカーパターンによる過負荷防止、そしてユーザーへの適切なフィードバックを組み合わせることで、堅牢なAPIクライアントを構築できます。

次章では、Core Dataの基礎について解説します。

## 参考文献

- [Apple Developer Documentation - Error Handling](https://developer.apple.com/documentation/swift/error-handling)
- [Apple Developer Documentation - URLSession](https://developer.apple.com/documentation/foundation/urlsession)
- [Martin Fowler - Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)
