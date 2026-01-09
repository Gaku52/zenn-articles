---
title: "URLSessionとAPI通信"
---

# URLSessionとAPI通信

## この章で学ぶこと

- URLSessionの基礎から実践的な使い方まで
- 型安全なAPI通信の設計パターン
- async/awaitを活用したモダンな非同期処理
- エラーハンドリングとリトライ戦略
- テスタブルなネットワーク層の構築
- パフォーマンス最適化テクニック

## URLSession基礎

### URLSessionとは

URLSessionは、iOS/macOSにおけるHTTP/HTTPS通信の標準フレームワークです。iOS 7で導入され、iOS 15でasync/await対応が追加されました。

#### URLSessionの主要な特徴

- **HTTP/2サポート**: 自動的にHTTP/2を使用し、マルチプレクシングによる効率的な通信
- **バックグラウンド転送**: アプリがバックグラウンドでも継続可能な転送処理
- **認証サポート**: Basic、Digest、OAuth等の各種認証方式に対応
- **キャッシュ機能**: HTTPキャッシュの自動管理
- **Cookie管理**: Cookieの自動的な保存と送信

### シンプルなGETリクエスト

最も基本的なGETリクエストから始めましょう。

```swift
// 最小限の実装
func fetchUser(id: Int) async throws -> User {
    let url = URL(string: "https://api.example.com/users/\(id)")!
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(User.self, from: data)
}

struct User: Codable {
    let id: Int
    let name: String
    let email: String
}
```

この実装は簡潔ですが、以下の問題があります:

1. エラーハンドリングが不十分
2. HTTPステータスコードを確認していない
3. URLが無効な場合の処理がない
4. デコードエラーの詳細が不明

### 堅牢なGETリクエスト

本番環境で使用できるレベルに改善します:

```swift
enum NetworkError: Error {
    case invalidURL
    case invalidResponse
    case httpError(statusCode: Int, data: Data?)
    case decodingError(Error)
    case noData
}

func fetchUserRobust(id: Int) async throws -> User {
    // 1. URL検証
    guard let url = URL(string: "https://api.example.com/users/\(id)") else {
        throw NetworkError.invalidURL
    }

    // 2. リクエスト実行
    let (data, response) = try await URLSession.shared.data(from: url)

    // 3. レスポンス検証
    guard let httpResponse = response as? HTTPURLResponse else {
        throw NetworkError.invalidResponse
    }

    // 4. ステータスコード確認
    guard (200...299).contains(httpResponse.statusCode) else {
        throw NetworkError.httpError(
            statusCode: httpResponse.statusCode,
            data: data
        )
    }

    // 5. データ検証
    guard !data.isEmpty else {
        throw NetworkError.noData
    }

    // 6. デコード
    do {
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601
        return try decoder.decode(User.self, from: data)
    } catch {
        throw NetworkError.decodingError(error)
    }
}
```

### POSTリクエスト

データを送信するPOSTリクエストの実装:

```swift
struct CreateUserRequest: Codable {
    let name: String
    let email: String
    let password: String
}

struct CreateUserResponse: Codable {
    let id: Int
    let name: String
    let email: String
    let createdAt: Date
}

func createUser(request: CreateUserRequest) async throws -> CreateUserResponse {
    guard let url = URL(string: "https://api.example.com/users") else {
        throw NetworkError.invalidURL
    }

    // URLRequestの作成
    var urlRequest = URLRequest(url: url)
    urlRequest.httpMethod = "POST"
    urlRequest.setValue("application/json", forHTTPHeaderField: "Content-Type")
    urlRequest.setValue("Bearer \(accessToken)", forHTTPHeaderField: "Authorization")

    // リクエストボディのエンコード
    do {
        let encoder = JSONEncoder()
        encoder.dateEncodingStrategy = .iso8601
        urlRequest.httpBody = try encoder.encode(request)
    } catch {
        throw NetworkError.encodingError(error)
    }

    // リクエスト実行
    let (data, response) = try await URLSession.shared.data(for: urlRequest)

    // レスポンス検証
    guard let httpResponse = response as? HTTPURLResponse else {
        throw NetworkError.invalidResponse
    }

    guard (200...299).contains(httpResponse.statusCode) else {
        throw NetworkError.httpError(
            statusCode: httpResponse.statusCode,
            data: data
        )
    }

    // デコード
    do {
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601
        return try decoder.decode(CreateUserResponse.self, from: data)
    } catch {
        throw NetworkError.decodingError(error)
    }
}
```

### URLSessionConfiguration

URLSessionの動作をカスタマイズするための設定:

```swift
class NetworkManager {
    static let shared = NetworkManager()

    let session: URLSession

    private init() {
        let configuration = URLSessionConfiguration.default

        // タイムアウト設定
        configuration.timeoutIntervalForRequest = 30 // リクエストタイムアウト
        configuration.timeoutIntervalForResource = 300 // リソース全体のタイムアウト

        // 接続性の設定
        configuration.waitsForConnectivity = true // 接続を待つ
        configuration.allowsCellularAccess = true // セルラーアクセスを許可
        configuration.allowsExpensiveNetworkAccess = true // 高価な接続を許可
        configuration.allowsConstrainedNetworkAccess = true // 制限された接続を許可

        // キャッシュ設定
        let cachesURL = FileManager.default.urls(
            for: .cachesDirectory,
            in: .userDomainMask
        ).first!
        let diskPath = cachesURL.appendingPathComponent("NetworkCache").path

        let cache = URLCache(
            memoryCapacity: 50 * 1024 * 1024, // 50MB
            diskCapacity: 100 * 1024 * 1024, // 100MB
            diskPath: diskPath
        )
        configuration.urlCache = cache
        configuration.requestCachePolicy = .returnCacheDataElseLoad

        // HTTPヘッダー設定
        configuration.httpAdditionalHeaders = [
            "Accept": "application/json",
            "User-Agent": "MyApp/1.0 (iOS)",
            "Accept-Language": Locale.current.languageCode ?? "en"
        ]

        // Cookie設定
        configuration.httpCookieAcceptPolicy = .always
        configuration.httpShouldSetCookies = true

        // HTTP/2設定
        configuration.httpMaximumConnectionsPerHost = 5

        // TLS設定
        configuration.tlsMinimumSupportedProtocolVersion = .TLSv12

        self.session = URLSession(configuration: configuration)
    }
}
```

## 型安全なAPI通信の設計

### Endpointパターン

APIエンドポイントを型安全に定義するパターン:

```swift
// HTTPメソッド
enum HTTPMethod: String {
    case get = "GET"
    case post = "POST"
    case put = "PUT"
    case delete = "DELETE"
    case patch = "PATCH"
}

// Endpointプロトコル
protocol Endpoint {
    associatedtype Response: Decodable

    var baseURL: String { get }
    var path: String { get }
    var method: HTTPMethod { get }
    var headers: [String: String]? { get }
    var queryItems: [URLQueryItem]? { get }
    var body: Data? { get }

    func makeRequest() throws -> URLRequest
}

// デフォルト実装
extension Endpoint {
    var baseURL: String {
        "https://api.example.com/v1"
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

        // デフォルトヘッダーを追加
        headers?.forEach { key, value in
            request.setValue(value, forHTTPHeaderField: key)
        }

        return request
    }
}
```

### 具体的なEndpointの実装

ユーザーAPIのエンドポイント定義:

```swift
enum UserEndpoint {
    case list(page: Int, limit: Int)
    case get(id: Int)
    case create(CreateUserRequest)
    case update(id: Int, UpdateUserRequest)
    case delete(id: Int)
    case search(query: String)
}

extension UserEndpoint: Endpoint {
    typealias Response = UserResponse

    var path: String {
        switch self {
        case .list:
            return "/users"
        case .get(let id), .update(let id, _), .delete(let id):
            return "/users/\(id)"
        case .create:
            return "/users"
        case .search:
            return "/users/search"
        }
    }

    var method: HTTPMethod {
        switch self {
        case .list, .get, .search:
            return .get
        case .create:
            return .post
        case .update:
            return .put
        case .delete:
            return .delete
        }
    }

    var queryItems: [URLQueryItem]? {
        switch self {
        case .list(let page, let limit):
            return [
                URLQueryItem(name: "page", value: "\(page)"),
                URLQueryItem(name: "limit", value: "\(limit)")
            ]
        case .search(let query):
            return [URLQueryItem(name: "q", value: query)]
        default:
            return nil
        }
    }

    var body: Data? {
        let encoder = JSONEncoder()
        encoder.dateEncodingStrategy = .iso8601

        switch self {
        case .create(let request):
            return try? encoder.encode(request)
        case .update(_, let request):
            return try? encoder.encode(request)
        default:
            return nil
        }
    }
}

// レスポンス型
enum UserResponse: Decodable {
    case single(User)
    case list([User])
    case created(User)
    case updated(User)
    case deleted

    init(from decoder: Decoder) throws {
        if let user = try? User(from: decoder) {
            self = .single(user)
        } else if let users = try? [User](from: decoder) {
            self = .list(users)
        } else {
            throw DecodingError.dataCorrupted(
                DecodingError.Context(
                    codingPath: decoder.codingPath,
                    debugDescription: "Unable to decode UserResponse"
                )
            )
        }
    }
}
```

### APIServiceの実装

汎用的なAPIサービスレイヤー:

```swift
protocol APIService {
    func request<E: Endpoint>(_ endpoint: E) async throws -> E.Response
}

class APIServiceImpl: APIService {
    private let session: URLSession
    private let decoder: JSONDecoder

    init(session: URLSession = .shared) {
        self.session = session

        self.decoder = JSONDecoder()
        self.decoder.dateDecodingStrategy = .iso8601
        self.decoder.keyDecodingStrategy = .convertFromSnakeCase
    }

    func request<E: Endpoint>(_ endpoint: E) async throws -> E.Response {
        // リクエスト作成
        let request = try endpoint.makeRequest()

        // ログ出力
        logRequest(request)

        // リクエスト実行
        let (data, response) = try await session.data(for: request)

        // レスポンスログ
        logResponse(response, data: data)

        // HTTPレスポンス検証
        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }

        // ステータスコード検証
        guard (200...299).contains(httpResponse.statusCode) else {
            throw try parseError(statusCode: httpResponse.statusCode, data: data)
        }

        // デコード
        do {
            return try decoder.decode(E.Response.self, from: data)
        } catch {
            throw NetworkError.decodingError(error)
        }
    }

    private func logRequest(_ request: URLRequest) {
        #if DEBUG
        print("🌐 Request: \(request.httpMethod ?? "GET") \(request.url?.absoluteString ?? "")")
        if let headers = request.allHTTPHeaderFields {
            print("📋 Headers: \(headers)")
        }
        if let body = request.httpBody,
           let bodyString = String(data: body, encoding: .utf8) {
            print("📦 Body: \(bodyString)")
        }
        #endif
    }

    private func logResponse(_ response: URLResponse, data: Data) {
        #if DEBUG
        if let httpResponse = response as? HTTPURLResponse {
            print("📡 Response: \(httpResponse.statusCode)")
            if let responseString = String(data: data, encoding: .utf8) {
                print("📥 Data: \(responseString)")
            }
        }
        #endif
    }

    private func parseError(statusCode: Int, data: Data) throws -> NetworkError {
        // APIエラーレスポンスをパース
        if let errorResponse = try? decoder.decode(APIErrorResponse.self, from: data) {
            return .apiError(errorResponse)
        }

        return NetworkError.httpError(statusCode: statusCode, data: data)
    }
}

struct APIErrorResponse: Codable {
    let code: String
    let message: String
    let details: [String: String]?
}
```

## エラーハンドリング

### 包括的なエラー定義

実運用に耐えるエラー定義:

```swift
enum NetworkError: Error {
    case invalidURL
    case invalidResponse
    case httpError(statusCode: Int, data: Data?)
    case apiError(APIErrorResponse)
    case encodingError(Error)
    case decodingError(Error)
    case noData
    case networkUnavailable
    case timeout
    case cancelled
    case unauthorized
    case forbidden
    case notFound
    case serverError(Int)
    case rateLimitExceeded(retryAfter: TimeInterval?)
    case unknown(Error)
}

extension NetworkError: LocalizedError {
    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "The URL is invalid."
        case .invalidResponse:
            return "Invalid response from server."
        case .httpError(let statusCode, _):
            return "HTTP error with status code \(statusCode)."
        case .apiError(let response):
            return response.message
        case .encodingError(let error):
            return "Failed to encode request: \(error.localizedDescription)"
        case .decodingError(let error):
            return "Failed to decode response: \(error.localizedDescription)"
        case .noData:
            return "No data received from server."
        case .networkUnavailable:
            return "Network connection is unavailable."
        case .timeout:
            return "Request timed out."
        case .cancelled:
            return "Request was cancelled."
        case .unauthorized:
            return "You are not authorized. Please login again."
        case .forbidden:
            return "Access to this resource is forbidden."
        case .notFound:
            return "The requested resource was not found."
        case .serverError(let code):
            return "Server error occurred (code: \(code))."
        case .rateLimitExceeded(let retryAfter):
            if let retryAfter = retryAfter {
                return "Rate limit exceeded. Please try again in \(Int(retryAfter)) seconds."
            }
            return "Rate limit exceeded. Please try again later."
        case .unknown(let error):
            return "An unknown error occurred: \(error.localizedDescription)"
        }
    }

    var recoverySuggestion: String? {
        switch self {
        case .networkUnavailable:
            return "Please check your internet connection and try again."
        case .timeout:
            return "The request took too long. Please try again."
        case .unauthorized:
            return "Please login again to continue."
        case .serverError:
            return "The server is experiencing issues. Please try again later."
        case .rateLimitExceeded:
            return "You have made too many requests. Please wait before trying again."
        default:
            return nil
        }
    }

    var isRetryable: Bool {
        switch self {
        case .timeout, .networkUnavailable, .serverError:
            return true
        default:
            return false
        }
    }
}

// HTTPステータスコードからエラーを生成
extension NetworkError {
    static func from(statusCode: Int, data: Data? = nil, headers: [String: String]? = nil) -> NetworkError {
        switch statusCode {
        case 401:
            return .unauthorized
        case 403:
            return .forbidden
        case 404:
            return .notFound
        case 408:
            return .timeout
        case 429:
            let retryAfter = headers?["Retry-After"].flatMap(TimeInterval.init)
            return .rateLimitExceeded(retryAfter: retryAfter)
        case 500...599:
            return .serverError(statusCode)
        default:
            return .httpError(statusCode: statusCode, data: data)
        }
    }
}

// URLErrorからNetworkErrorへのマッピング
extension NetworkError {
    static func from(urlError: URLError) -> NetworkError {
        switch urlError.code {
        case .notConnectedToInternet, .networkConnectionLost, .dataNotAllowed:
            return .networkUnavailable
        case .timedOut:
            return .timeout
        case .cancelled:
            return .cancelled
        default:
            return .unknown(urlError)
        }
    }
}
```

### エラーハンドリングの実装

エラーを適切に処理するサービス実装:

```swift
class RobustAPIService: APIService {
    private let session: URLSession
    private let decoder: JSONDecoder

    init(session: URLSession = .shared) {
        self.session = session

        self.decoder = JSONDecoder()
        self.decoder.dateDecodingStrategy = .iso8601
        self.decoder.keyDecodingStrategy = .convertFromSnakeCase
    }

    func request<E: Endpoint>(_ endpoint: E) async throws -> E.Response {
        // リクエスト作成
        let request = try endpoint.makeRequest()

        do {
            // リクエスト実行
            let (data, response) = try await session.data(for: request)

            // HTTPレスポンス検証
            guard let httpResponse = response as? HTTPURLResponse else {
                throw NetworkError.invalidResponse
            }

            // ステータスコードに基づいてエラーを処理
            if !(200...299).contains(httpResponse.statusCode) {
                throw handleErrorResponse(
                    statusCode: httpResponse.statusCode,
                    data: data,
                    headers: httpResponse.allHeaderFields as? [String: String]
                )
            }

            // 空データチェック
            guard !data.isEmpty else {
                throw NetworkError.noData
            }

            // デコード
            return try decodeResponse(data: data)

        } catch let error as NetworkError {
            throw error
        } catch let error as URLError {
            throw NetworkError.from(urlError: error)
        } catch {
            throw NetworkError.unknown(error)
        }
    }

    private func decodeResponse<T: Decodable>(data: Data) throws -> T {
        do {
            return try decoder.decode(T.self, from: data)
        } catch {
            // デコードエラーの詳細ログ
            #if DEBUG
            print("❌ Decoding error: \(error)")
            if let dataString = String(data: data, encoding: .utf8) {
                print("📄 Response data: \(dataString)")
            }
            #endif

            throw NetworkError.decodingError(error)
        }
    }

    private func handleErrorResponse(
        statusCode: Int,
        data: Data,
        headers: [String: String]?
    ) -> NetworkError {
        // APIエラーレスポンスをパース
        if let errorResponse = try? decoder.decode(APIErrorResponse.self, from: data) {
            return .apiError(errorResponse)
        }

        return NetworkError.from(
            statusCode: statusCode,
            data: data,
            headers: headers
        )
    }
}
```

## リトライ戦略

### 基本的なリトライロジック

指数バックオフを使用したリトライ実装:

```swift
class RetryableAPIService: APIService {
    private let baseService: APIService
    private let maxRetries: Int
    private let baseDelay: TimeInterval

    init(
        baseService: APIService,
        maxRetries: Int = 3,
        baseDelay: TimeInterval = 1.0
    ) {
        self.baseService = baseService
        self.maxRetries = maxRetries
        self.baseDelay = baseDelay
    }

    func request<E: Endpoint>(_ endpoint: E) async throws -> E.Response {
        var lastError: Error?

        for attempt in 0..<maxRetries {
            do {
                return try await baseService.request(endpoint)
            } catch let error as NetworkError {
                lastError = error

                // リトライすべきかチェック
                guard shouldRetry(error: error, attempt: attempt) else {
                    throw error
                }

                // 指数バックオフで待機
                let delay = calculateDelay(attempt: attempt)
                try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))

                // キャンセルチェック
                try Task.checkCancellation()

                print("🔄 Retrying request (attempt \(attempt + 2)/\(maxRetries))...")

            } catch {
                throw error
            }
        }

        throw lastError ?? NetworkError.unknown(NSError(domain: "Unknown", code: -1))
    }

    private func shouldRetry(error: NetworkError, attempt: Int) -> Bool {
        // 最大リトライ回数チェック
        guard attempt < maxRetries - 1 else { return false }

        // リトライ可能なエラーかチェック
        return error.isRetryable
    }

    private func calculateDelay(attempt: Int) -> TimeInterval {
        // 指数バックオフ: baseDelay * 2^attempt
        let exponentialDelay = baseDelay * pow(2.0, Double(attempt))

        // ジッター追加 (±25%)
        let jitter = Double.random(in: 0.75...1.25)

        return exponentialDelay * jitter
    }
}
```

### 高度なリトライ戦略

より洗練されたリトライロジック:

```swift
struct RetryPolicy {
    let maxRetries: Int
    let baseDelay: TimeInterval
    let maxDelay: TimeInterval
    let retryableStatusCodes: Set<Int>

    static let `default` = RetryPolicy(
        maxRetries: 3,
        baseDelay: 1.0,
        maxDelay: 60.0,
        retryableStatusCodes: [408, 429, 500, 502, 503, 504]
    )

    static let aggressive = RetryPolicy(
        maxRetries: 5,
        baseDelay: 0.5,
        maxDelay: 30.0,
        retryableStatusCodes: [408, 429, 500, 502, 503, 504]
    )

    static let conservative = RetryPolicy(
        maxRetries: 2,
        baseDelay: 2.0,
        maxDelay: 120.0,
        retryableStatusCodes: [503, 504]
    )
}

actor RetryCoordinator {
    private var retryCount: [String: Int] = [:]

    func shouldRetry(for key: String, policy: RetryPolicy) -> Bool {
        let count = retryCount[key, default: 0]
        return count < policy.maxRetries
    }

    func recordRetry(for key: String) {
        retryCount[key, default: 0] += 1
    }

    func reset(for key: String) {
        retryCount.removeValue(forKey: key)
    }
}

class AdvancedRetryableAPIService: APIService {
    private let baseService: APIService
    private let policy: RetryPolicy
    private let coordinator = RetryCoordinator()

    init(baseService: APIService, policy: RetryPolicy = .default) {
        self.baseService = baseService
        self.policy = policy
    }

    func request<E: Endpoint>(_ endpoint: E) async throws -> E.Response {
        let requestKey = makeRequestKey(endpoint)
        var lastError: Error?

        while await coordinator.shouldRetry(for: requestKey, policy: policy) {
            do {
                let response = try await baseService.request(endpoint)
                await coordinator.reset(for: requestKey)
                return response

            } catch let error as NetworkError {
                lastError = error

                guard shouldRetry(error: error) else {
                    throw error
                }

                await coordinator.recordRetry(for: requestKey)

                let delay = await calculateDelay(for: requestKey)

                print("🔄 Retrying after \(delay)s...")
                try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))

                try Task.checkCancellation()

            } catch {
                throw error
            }
        }

        await coordinator.reset(for: requestKey)
        throw lastError ?? NetworkError.unknown(NSError(domain: "Unknown", code: -1))
    }

    private func makeRequestKey<E: Endpoint>(_ endpoint: E) -> String {
        "\(endpoint.method.rawValue):\(endpoint.path)"
    }

    private func shouldRetry(error: NetworkError) -> Bool {
        switch error {
        case .timeout, .networkUnavailable:
            return true
        case .serverError(let code):
            return policy.retryableStatusCodes.contains(code)
        case .httpError(let statusCode, _):
            return policy.retryableStatusCodes.contains(statusCode)
        case .rateLimitExceeded:
            return true
        default:
            return false
        }
    }

    private func calculateDelay(for key: String) async -> TimeInterval {
        let attempt = await coordinator.retryCount[key, default: 0]
        let exponentialDelay = policy.baseDelay * pow(2.0, Double(attempt))
        let cappedDelay = min(exponentialDelay, policy.maxDelay)

        // ジッター追加
        let jitter = Double.random(in: 0.8...1.2)
        return cappedDelay * jitter
    }
}
```

## リクエストのキャンセル

### Taskベースのキャンセル

async/awaitでのキャンセル処理:

```swift
@MainActor
class SearchViewModel: ObservableObject {
    @Published var searchResults: [User] = []
    @Published var isSearching = false
    @Published var error: NetworkError?

    private var searchTask: Task<Void, Never>?
    private let apiService: APIService

    init(apiService: APIService) {
        self.apiService = apiService
    }

    func search(query: String) {
        // 既存の検索をキャンセル
        searchTask?.cancel()

        guard !query.isEmpty else {
            searchResults = []
            return
        }

        searchTask = Task { @MainActor in
            isSearching = true
            error = nil

            do {
                // デバウンス
                try await Task.sleep(nanoseconds: 300_000_000) // 300ms

                // キャンセルチェック
                try Task.checkCancellation()

                // API呼び出し
                let results: [User] = try await apiService.request(
                    UserEndpoint.search(query: query)
                )

                // キャンセルチェック
                guard !Task.isCancelled else { return }

                // 結果を更新
                searchResults = results

            } catch is CancellationError {
                // キャンセルは無視
            } catch let error as NetworkError {
                self.error = error
            } catch {
                self.error = .unknown(error)
            }

            isSearching = false
        }
    }

    func cancelSearch() {
        searchTask?.cancel()
        searchTask = nil
        isSearching = false
    }

    deinit {
        searchTask?.cancel()
    }
}
```

### URLSessionTaskのキャンセル

URLSessionTaskを直接キャンセル:

```swift
class CancellableNetworkManager {
    private var activeTasks: [String: URLSessionTask] = [:]
    private let lock = NSLock()

    func request<T: Decodable>(
        url: URL,
        taskIdentifier: String
    ) async throws -> T {
        let task = URLSession.shared.dataTask(with: url)

        // タスクを記録
        lock.lock()
        activeTasks[taskIdentifier] = task
        lock.unlock()

        defer {
            lock.lock()
            activeTasks.removeValue(forKey: taskIdentifier)
            lock.unlock()
        }

        return try await withTaskCancellationHandler {
            try await withCheckedThrowingContinuation { continuation in
                task.completionHandler = { data, response, error in
                    if let error = error {
                        continuation.resume(throwing: error)
                        return
                    }

                    guard let data = data,
                          let httpResponse = response as? HTTPURLResponse,
                          (200...299).contains(httpResponse.statusCode) else {
                        continuation.resume(throwing: NetworkError.invalidResponse)
                        return
                    }

                    do {
                        let decoded = try JSONDecoder().decode(T.self, from: data)
                        continuation.resume(returning: decoded)
                    } catch {
                        continuation.resume(throwing: NetworkError.decodingError(error))
                    }
                }
                task.resume()
            }
        } onCancel: {
            task.cancel()
        }
    }

    func cancel(taskIdentifier: String) {
        lock.lock()
        defer { lock.unlock()  }

        activeTasks[taskIdentifier]?.cancel()
        activeTasks.removeValue(forKey: taskIdentifier)
    }

    func cancelAll() {
        lock.lock()
        defer { lock.unlock() }

        activeTasks.values.forEach { $0.cancel() }
        activeTasks.removeAll()
    }
}
```

## パフォーマンス最適化

### リクエストの重複排除

同じリクエストを複数回実行しないように最適化:

```swift
actor RequestDeduplicator {
    private var pendingRequests: [String: Task<Any, Error>] = [:]

    func deduplicate<T>(
        key: String,
        operation: @escaping () async throws -> T
    ) async throws -> T {
        // 既存のリクエストがあれば待機
        if let existingTask = pendingRequests[key] {
            return try await existingTask.value as! T
        }

        // 新しいリクエストを作成
        let task = Task<Any, Error> {
            try await operation()
        }

        pendingRequests[key] = task

        defer {
            pendingRequests.removeValue(forKey: key)
        }

        return try await task.value as! T
    }
}

class OptimizedAPIService: APIService {
    private let baseService: APIService
    private let deduplicator = RequestDeduplicator()

    init(baseService: APIService) {
        self.baseService = baseService
    }

    func request<E: Endpoint>(_ endpoint: E) async throws -> E.Response {
        let key = "\(endpoint.method.rawValue):\(endpoint.path)"

        return try await deduplicator.deduplicate(key: key) {
            try await baseService.request(endpoint)
        }
    }
}
```

### バッチリクエスト

複数のリクエストを効率的に処理:

```swift
struct BatchRequest {
    let id: String
    let endpoint: any Endpoint
}

class BatchRequestManager {
    private let apiService: APIService
    private let maxConcurrentRequests: Int

    init(apiService: APIService, maxConcurrentRequests: Int = 5) {
        self.apiService = apiService
        self.maxConcurrentRequests = maxConcurrentRequests
    }

    func execute<T: Decodable>(_ requests: [BatchRequest]) async throws -> [String: T] {
        var results: [String: T] = [:]

        // リクエストをチャンクに分割
        for chunk in requests.chunked(into: maxConcurrentRequests) {
            // 並列実行
            await withTaskGroup(of: (String, Result<T, Error>).self) { group in
                for request in chunk {
                    group.addTask {
                        do {
                            let result: T = try await self.apiService.request(request.endpoint)
                            return (request.id, .success(result))
                        } catch {
                            return (request.id, .failure(error))
                        }
                    }
                }

                // 結果を収集
                for await (id, result) in group {
                    switch result {
                    case .success(let value):
                        results[id] = value
                    case .failure(let error):
                        print("❌ Request \(id) failed: \(error)")
                    }
                }
            }
        }

        return results
    }
}

extension Array {
    func chunked(into size: Int) -> [[Element]] {
        stride(from: 0, to: count, by: size).map {
            Array(self[$0..<Swift.min($0 + size, count)])
        }
    }
}
```

### HTTP/2マルチプレクシング

HTTP/2の機能を最大限活用:

```swift
class HTTP2OptimizedNetworkManager {
    static let shared = HTTP2OptimizedNetworkManager()

    let session: URLSession

    private init() {
        let configuration = URLSessionConfiguration.default

        // HTTP/2最適化
        configuration.httpMaximumConnectionsPerHost = 1 // HTTP/2では1接続で複数リクエスト
        configuration.multipathServiceType = .handover // マルチパス TCP

        // パフォーマンス設定
        configuration.timeoutIntervalForRequest = 30
        configuration.timeoutIntervalForResource = 300
        configuration.waitsForConnectivity = true

        self.session = URLSession(configuration: configuration)
    }

    func parallelRequests<T: Decodable>(
        _ endpoints: [any Endpoint]
    ) async throws -> [T] {
        try await withThrowingTaskGroup(of: T.self) { group in
            for endpoint in endpoints {
                group.addTask {
                    let request = try endpoint.makeRequest()
                    let (data, _) = try await self.session.data(for: request)
                    return try JSONDecoder().decode(T.self, from: data)
                }
            }

            var results: [T] = []
            for try await result in group {
                results.append(result)
            }
            return results
        }
    }
}
```

## テスト

### モックAPIService

テスト用のモック実装:

```swift
class MockAPIService: APIService {
    var mockResponse: Any?
    var mockError: Error?
    var requestLog: [any Endpoint] = []

    func request<E: Endpoint>(_ endpoint: E) async throws -> E.Response {
        requestLog.append(endpoint)

        if let error = mockError {
            throw error
        }

        guard let response = mockResponse as? E.Response else {
            throw NetworkError.noData
        }

        return response
    }
}

// 使用例
final class UserServiceTests: XCTestCase {
    func testFetchUser() async throws {
        // Arrange
        let mockService = MockAPIService()
        let expectedUser = User(id: 1, name: "Test", email: "test@example.com")
        mockService.mockResponse = expectedUser

        let userService = UserService(apiService: mockService)

        // Act
        let user = try await userService.getUser(id: 1)

        // Assert
        XCTAssertEqual(user.id, expectedUser.id)
        XCTAssertEqual(user.name, expectedUser.name)
        XCTAssertEqual(mockService.requestLog.count, 1)
    }

    func testFetchUserError() async {
        // Arrange
        let mockService = MockAPIService()
        mockService.mockError = NetworkError.notFound

        let userService = UserService(apiService: mockService)

        // Act & Assert
        do {
            _ = try await userService.getUser(id: 999)
            XCTFail("Should throw error")
        } catch let error as NetworkError {
            if case .notFound = error {
                // Success
            } else {
                XCTFail("Wrong error type")
            }
        }
    }
}
```

### URLProtocolモック

URLSessionをモック:

```swift
class MockURLProtocol: URLProtocol {
    static var mockResponses: [URL: (Data, HTTPURLResponse)] = [:]
    static var mockError: Error?

    override class func canInit(with request: URLRequest) -> Bool {
        return true
    }

    override class func canonicalRequest(for request: URLRequest) -> URLRequest {
        return request
    }

    override func startLoading() {
        if let error = MockURLProtocol.mockError {
            client?.urlProtocol(self, didFailWithError: error)
            return
        }

        guard let url = request.url,
              let (data, response) = MockURLProtocol.mockResponses[url] else {
            client?.urlProtocol(
                self,
                didFailWithError: NetworkError.noData
            )
            return
        }

        client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)
        client?.urlProtocol(self, didLoad: data)
        client?.urlProtocolDidFinishLoading(self)
    }

    override func stopLoading() {
        // 何もしない
    }
}

// テストでの使用
final class APIServiceIntegrationTests: XCTestCase {
    var session: URLSession!
    var apiService: APIService!

    override func setUp() {
        super.setUp()

        let configuration = URLSessionConfiguration.ephemeral
        configuration.protocolClasses = [MockURLProtocol.self]
        session = URLSession(configuration: configuration)
        apiService = APIServiceImpl(session: session)
    }

    override func tearDown() {
        MockURLProtocol.mockResponses.removeAll()
        MockURLProtocol.mockError = nil
        super.tearDown()
    }

    func testFetchUserIntegration() async throws {
        // Arrange
        let url = URL(string: "https://api.example.com/v1/users/1")!
        let user = User(id: 1, name: "Test", email: "test@example.com")
        let data = try JSONEncoder().encode(user)
        let response = HTTPURLResponse(
            url: url,
            statusCode: 200,
            httpVersion: nil,
            headerFields: nil
        )!

        MockURLProtocol.mockResponses[url] = (data, response)

        // Act
        let result: User = try await apiService.request(UserEndpoint.get(id: 1))

        // Assert
        XCTAssertEqual(result.id, user.id)
        XCTAssertEqual(result.name, user.name)
    }
}
```

## まとめ

この章では、URLSessionを使った型安全で保守性の高いAPI通信の実装方法を学びました。

### 重要なポイント

1. **型安全性**: Endpointパターンで型安全なAPI定義
2. **エラーハンドリング**: 包括的なエラー処理とリトライ戦略
3. **パフォーマンス**: リクエストの重複排除とバッチ処理
4. **テスタビリティ**: モックを使った効率的なテスト
5. **保守性**: レイヤー分離による変更容易性

### 次のステップ

次章では、認証とトークン管理について詳しく学びます。JWTトークンの管理、OAuth 2.0フロー、セキュアなトークン保存方法などを解説します。

### 参考リソース

- [URLSession - Apple Developer Documentation](https://developer.apple.com/documentation/foundation/urlsession)
- [Swift Concurrency - Apple Developer Documentation](https://developer.apple.com/documentation/swift/swift-standard-library/concurrency)
- [HTTP/2 - RFC 7540](https://tools.ietf.org/html/rfc7540)
