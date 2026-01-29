---
title: "エラーハンドリングとリトライ戦略"
---

# エラーハンドリングとリトライ戦略

ネットワーク通信では、様々なエラーが発生する可能性があります。この章では、エラーを適切に処理し、ユーザー体験を向上させるためのテクニックを学びます。

## 包括的なエラー定義

まず、発生しうる全てのエラーを網羅的に定義します。

```swift
enum NetworkError: Error, LocalizedError {
    case invalidURL
    case invalidResponse
    case httpError(Int, Data?)
    case decodingError(Error)
    case encodingError(Error)
    case noData
    case networkUnavailable
    case timeout
    case cancelled
    case unauthorized
    case forbidden
    case notFound
    case serverError(Int)
    case apiError(ErrorResponse?)
    case unknown(Error)

    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "無効な URL です"
        case .invalidResponse:
            return "サーバーからの応答が無効です"
        case .httpError(let code, _):
            return "HTTP エラー (\(code))"
        case .decodingError(let error):
            return "データの解析に失敗しました: \(error.localizedDescription)"
        case .encodingError(let error):
            return "データのエンコードに失敗しました: \(error.localizedDescription)"
        case .noData:
            return "サーバーからデータが返されませんでした"
        case .networkUnavailable:
            return "ネットワーク接続がありません"
        case .timeout:
            return "リクエストがタイムアウトしました"
        case .cancelled:
            return "リクエストがキャンセルされました"
        case .unauthorized:
            return "認証が必要です。再度ログインしてください"
        case .forbidden:
            return "このリソースへのアクセスが拒否されました"
        case .notFound:
            return "リソースが見つかりませんでした"
        case .serverError(let code):
            return "サーバーエラーが発生しました (\(code))"
        case .apiError(let response):
            return response?.message ?? "API エラーが発生しました"
        case .unknown(let error):
            return "予期しないエラーが発生しました: \(error.localizedDescription)"
        }
    }

    var recoverySuggestion: String? {
        switch self {
        case .networkUnavailable:
            return "インターネット接続を確認して、もう一度お試しください"
        case .timeout:
            return "リクエストに時間がかかりすぎました。もう一度お試しください"
        case .unauthorized:
            return "再度ログインしてください"
        case .serverError:
            return "しばらく待ってから、もう一度お試しください"
        default:
            return nil
        }
    }
}
```

## ステータスコードからのエラー生成

HTTP ステータスコードに応じて適切なエラーを生成します:

```swift
extension NetworkError {
    static func from(statusCode: Int, data: Data? = nil) -> NetworkError {
        switch statusCode {
        case 400:
            return .httpError(400, data)
        case 401:
            return .unauthorized
        case 403:
            return .forbidden
        case 404:
            return .notFound
        case 408:
            return .timeout
        case 500...599:
            return .serverError(statusCode)
        default:
            return .httpError(statusCode, data)
        }
    }
}
```

## URLError のマッピング

URLError を独自のエラーにマッピングします:

```swift
extension NetworkError {
    static func from(urlError: URLError) -> NetworkError {
        switch urlError.code {
        case .notConnectedToInternet, .networkConnectionLost:
            return .networkUnavailable
        case .timedOut:
            return .timeout
        case .cancelled:
            return .cancelled
        case .badURL:
            return .invalidURL
        case .cannotDecodeContentData, .cannotDecodeRawData:
            return .decodingError(urlError)
        default:
            return .unknown(urlError)
        }
    }
}
```

## 堅牢な APIService の実装

エラーハンドリングを強化した APIService を実装します:

```swift
class RobustAPIService: APIService {
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

        do {
            let (data, response) = try await session.data(for: request)

            // レスポンスの検証
            guard let httpResponse = response as? HTTPURLResponse else {
                throw NetworkError.invalidResponse
            }

            // ステータスコードのチェック
            guard (200...299).contains(httpResponse.statusCode) else {
                throw NetworkError.from(statusCode: httpResponse.statusCode, data: data)
            }

            // データの存在チェック
            guard !data.isEmpty else {
                throw NetworkError.noData
            }

            // デコード
            do {
                return try decoder.decode(T.self, from: data)
            } catch {
                throw NetworkError.decodingError(error)
            }

        } catch let error as NetworkError {
            throw error
        } catch let error as URLError {
            throw NetworkError.from(urlError: error)
        } catch {
            throw NetworkError.unknown(error)
        }
    }
}
```

## リトライロジック

一時的なエラーの場合、リトライすることでリクエストが成功する可能性があります。

```swift
func requestWithRetry<T: Decodable>(
    _ endpoint: Endpoint,
    maxRetries: Int = 3,
    retryDelay: TimeInterval = 2.0,
    apiService: APIService
) async throws -> T {
    var lastError: Error?
    var attempt = 0

    while attempt < maxRetries {
        do {
            return try await apiService.request(endpoint)
        } catch let error as NetworkError {
            lastError = error

            // リトライすべきエラーかチェック
            guard shouldRetry(error: error, attempt: attempt, maxRetries: maxRetries) else {
                throw error
            }

            // 指数バックオフで待機
            let delay = retryDelay * pow(2.0, Double(attempt))
            try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))

            attempt += 1

        } catch {
            throw error
        }
    }

    throw lastError ?? NetworkError.unknown(NSError(domain: "Unknown", code: -1))
}

private func shouldRetry(
    error: NetworkError,
    attempt: Int,
    maxRetries: Int
) -> Bool {
    guard attempt < maxRetries - 1 else { return false }

    switch error {
    case .timeout, .networkUnavailable:
        return true
    case .serverError(let code) where code >= 500:
        return true
    default:
        return false
    }
}
```

## Exponential Backoff

リトライ間隔を指数的に増加させることで、サーバーへの負荷を軽減します:

```swift
struct RetryConfig {
    let maxRetries: Int
    let initialDelay: TimeInterval
    let maxDelay: TimeInterval
    let multiplier: Double

    static let `default` = RetryConfig(
        maxRetries: 3,
        initialDelay: 1.0,
        maxDelay: 30.0,
        multiplier: 2.0
    )

    static let aggressive = RetryConfig(
        maxRetries: 5,
        initialDelay: 0.5,
        maxDelay: 60.0,
        multiplier: 2.0
    )

    func delay(for attempt: Int) -> TimeInterval {
        let exponentialDelay = initialDelay * pow(multiplier, Double(attempt))
        return min(exponentialDelay, maxDelay)
    }
}

func requestWithExponentialBackoff<T: Decodable>(
    _ endpoint: Endpoint,
    config: RetryConfig = .default,
    apiService: APIService
) async throws -> T {
    var lastError: Error?

    for attempt in 0..<config.maxRetries {
        do {
            return try await apiService.request(endpoint)
        } catch let error as NetworkError {
            lastError = error

            guard shouldRetry(error: error, attempt: attempt, maxRetries: config.maxRetries) else {
                throw error
            }

            let delay = config.delay(for: attempt)
            try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
        }
    }

    throw lastError ?? NetworkError.unknown(NSError(domain: "Unknown", code: -1))
}
```

## ViewModel でのエラーハンドリング

ViewModel でエラーを適切に処理し、ユーザーに表示します:

```swift
@MainActor
class UserViewModel: ObservableObject {
    @Published var users: [User] = []
    @Published var isLoading = false
    @Published var errorMessage: String?
    @Published var showError = false

    private let repository: UserRepository

    init(repository: UserRepository) {
        self.repository = repository
    }

    func loadUsers() async {
        isLoading = true
        defer { isLoading = false }

        do {
            users = try await repository.fetchAllUsers()
            errorMessage = nil
            showError = false
        } catch let error as NetworkError {
            handleNetworkError(error)
        } catch {
            errorMessage = "予期しないエラーが発生しました"
            showError = true
        }
    }

    private func handleNetworkError(_ error: NetworkError) {
        errorMessage = error.errorDescription

        switch error {
        case .unauthorized:
            // ログイン画面に遷移
            NotificationCenter.default.post(name: .userUnauthorized, object: nil)

        case .networkUnavailable:
            // オフラインモードを有効化
            showError = true

        case .serverError:
            // リトライボタンを表示
            showError = true

        default:
            showError = true
        }
    }

    func retry() async {
        await loadUsers()
    }
}

extension Notification.Name {
    static let userUnauthorized = Notification.Name("userUnauthorized")
}
```

## SwiftUI での表示

エラーメッセージを SwiftUI で表示します:

```swift
struct UserListView: View {
    @StateObject private var viewModel: UserViewModel

    var body: some View {
        NavigationView {
            ZStack {
                if viewModel.isLoading {
                    ProgressView("読み込み中...")
                } else {
                    List(viewModel.users) { user in
                        UserRow(user: user)
                    }
                }
            }
            .navigationTitle("ユーザー一覧")
            .task {
                await viewModel.loadUsers()
            }
            .alert("エラー", isPresented: $viewModel.showError) {
                Button("再試行") {
                    Task {
                        await viewModel.retry()
                    }
                }
                Button("キャンセル", role: .cancel) {}
            } message: {
                Text(viewModel.errorMessage ?? "エラーが発生しました")
            }
        }
    }
}
```

## グローバルエラーハンドラー

アプリ全体で共通のエラー処理を行います:

```swift
class GlobalErrorHandler {
    static let shared = GlobalErrorHandler()

    private init() {}

    func handle(_ error: Error) {
        if let networkError = error as? NetworkError {
            handleNetworkError(networkError)
        } else {
            handleUnknownError(error)
        }
    }

    private func handleNetworkError(_ error: NetworkError) {
        switch error {
        case .unauthorized:
            // ログアウト処理
            NotificationCenter.default.post(name: .userUnauthorized, object: nil)

        case .networkUnavailable:
            // オフラインモードを有効化
            NotificationCenter.default.post(name: .networkUnavailable, object: nil)

        case .serverError:
            // エラーログを送信
            logError(error)

        default:
            break
        }
    }

    private func handleUnknownError(_ error: Error) {
        logError(error)
    }

    private func logError(_ error: Error) {
        // エラーログをサーバーに送信
        print("Error logged: \(error)")
    }
}

extension Notification.Name {
    static let networkUnavailable = Notification.Name("networkUnavailable")
}
```

## まとめ

この章では、堅牢なエラーハンドリングとリトライ戦略を学びました:

- 包括的なエラー定義
- ステータスコードに応じたエラー処理
- リトライロジックと Exponential Backoff
- ViewModel でのエラーハンドリング
- グローバルエラーハンドラー

次の章では、WebSocket を使ったリアルタイム通信について学びます。

## 参考リソース

- [Error Handling - Swift.org](https://docs.swift.org/swift-book/LanguageGuide/ErrorHandling.html)
- [URLError - Apple Developer](https://developer.apple.com/documentation/foundation/urlerror)
