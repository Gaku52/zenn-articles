---
title: "ベストプラクティスとよくある問題"
---

# ベストプラクティスとよくある問題

この最終章では、実践的なベストプラクティスと、開発現場でよく遭遇する問題の解決策をまとめます。

## アーキテクチャのベストプラクティス

### レイヤーの分離

適切なレイヤー分離により、保守性とテスタビリティが向上します:

```swift
// ❌ すべてが混在
class BadViewController: UIViewController {
    func loadData() {
        let url = URL(string: "https://api.example.com/users")!
        URLSession.shared.dataTask(with: url) { data, _, _ in
            if let data = data {
                let users = try? JSONDecoder().decode([User].self, from: data)
                DispatchQueue.main.async {
                    self.users = users ?? []
                }
            }
        }.resume()
    }
}

// ✅ レイヤーに分離
// Presentation Layer
@MainActor
class GoodViewModel: ObservableObject {
    @Published var users: [User] = []

    private let repository: UserRepository

    init(repository: UserRepository) {
        self.repository = repository
    }

    func loadUsers() async {
        do {
            users = try await repository.fetchAllUsers()
        } catch {
            handleError(error)
        }
    }
}

// Domain Layer
protocol UserRepository {
    func fetchAllUsers() async throws -> [User]
}

// Data Layer
class UserRepositoryImpl: UserRepository {
    private let apiService: APIService

    init(apiService: APIService) {
        self.apiService = apiService
    }

    func fetchAllUsers() async throws -> [User] {
        try await apiService.request(UserEndpoint.getAllUsers)
    }
}
```

### 依存性注入

依存性注入により、テストが容易になります:

```swift
// ❌ 直接インスタンス化
class BadViewModel {
    private let repository = UserRepositoryImpl(apiService: APIServiceImpl())
}

// ✅ 依存性注入
class GoodViewModel {
    private let repository: UserRepository

    init(repository: UserRepository) {
        self.repository = repository
    }
}

// テスト時
class MockUserRepository: UserRepository {
    var users: [User] = []

    func fetchAllUsers() async throws -> [User] {
        users
    }
}

let testViewModel = GoodViewModel(repository: MockUserRepository())
```

## エラーハンドリングのベストプラクティス

### 包括的なエラー処理

すべてのエラーケースを考慮します:

```swift
enum DataError: Error, LocalizedError {
    case networkError(NetworkError)
    case databaseError(CoreDataError)
    case validationError(String)
    case unknown(Error)

    var errorDescription: String? {
        switch self {
        case .networkError(let error):
            return "ネットワークエラー: \(error.localizedDescription)"
        case .databaseError(let error):
            return "データベースエラー: \(error.localizedDescription)"
        case .validationError(let message):
            return "検証エラー: \(message)"
        case .unknown(let error):
            return "予期しないエラー: \(error.localizedDescription)"
        }
    }

    var recoverySuggestion: String? {
        switch self {
        case .networkError:
            return "ネットワーク接続を確認してください"
        case .databaseError:
            return "アプリを再起動してください"
        case .validationError:
            return "入力内容を確認してください"
        case .unknown:
            return "もう一度お試しください"
        }
    }
}
```

### ユーザーフレンドリーなエラーメッセージ

技術的な詳細を隠し、ユーザーに分かりやすいメッセージを表示します:

```swift
func displayUserFriendlyError(_ error: Error) -> String {
    if let dataError = error as? DataError {
        return dataError.errorDescription ?? "エラーが発生しました"
    }

    if let networkError = error as? NetworkError {
        switch networkError {
        case .networkUnavailable:
            return "インターネット接続がありません"
        case .timeout:
            return "リクエストがタイムアウトしました"
        case .unauthorized:
            return "ログインが必要です"
        default:
            return "通信エラーが発生しました"
        }
    }

    return "予期しないエラーが発生しました"
}
```

## パフォーマンスの最適化

### メモリ管理

メモリリークを防ぎます:

```swift
// ❌ 強参照サイクル
class BadImageCache {
    var images: [URL: UIImage] = [:]  // メモリリーク!

    func insert(_ image: UIImage, for url: URL) {
        images[url] = image
    }
}

// ✅ NSCache を使用
class GoodImageCache {
    private let cache = NSCache<NSString, UIImage>()

    init() {
        cache.countLimit = 100
        cache.totalCostLimit = 50 * 1024 * 1024

        // メモリ警告時にクリア
        NotificationCenter.default.addObserver(
            forName: UIApplication.didReceiveMemoryWarningNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            self?.cache.removeAllObjects()
        }
    }

    func insert(_ image: UIImage, for url: URL) {
        cache.setObject(image, forKey: url.absoluteString as NSString)
    }
}
```

### バッチ処理

大量のデータを効率的に処理します:

```swift
// ❌ 個別に保存
func badBatchInsert(users: [User]) async {
    for user in users {
        try? await localStore.create(name: user.name, email: user.email)
    }
}

// ✅ バッチで保存
func goodBatchInsert(users: [User]) async throws {
    try await CoreDataManager.shared.performBackgroundTask { context in
        for user in users {
            _ = User.create(in: context, name: user.name, email: user.email)
        }
        // コンテキストを一度だけ保存
    }
}
```

## セキュリティのベストプラクティス

### 機密情報の保護

機密情報は必ず Keychain に保存します:

```swift
// ❌ UserDefaults に保存
UserDefaults.standard.set("secret-token", forKey: "accessToken")

// ✅ Keychain に保存
try KeychainManager.shared.save(
    "secret-token",
    service: "com.example.app",
    account: "accessToken"
)
```

### HTTPS の強制

App Transport Security (ATS) を適切に設定します:

```xml
<!-- Info.plist -->
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <false/>
    <!-- 開発環境のみ例外を許可 -->
    <key>NSExceptionDomains</key>
    <dict>
        <key>localhost</key>
        <dict>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/>
        </dict>
    </dict>
</dict>
```

### 証明書ピンニング

中間者攻撃を防ぎます:

```swift
class CertificatePinningDelegate: NSObject, URLSessionDelegate {
    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        guard let serverTrust = challenge.protectionSpace.serverTrust,
              let certificate = SecTrustGetCertificateAtIndex(serverTrust, 0) else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        let serverCertificateData = SecCertificateCopyData(certificate) as Data

        // ピン留めされた証明書と比較
        if serverCertificateData == pinnedCertificateData {
            let credential = URLCredential(trust: serverTrust)
            completionHandler(.useCredential, credential)
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }

    private var pinnedCertificateData: Data {
        // アプリにバンドルされた証明書を取得
        guard let certPath = Bundle.main.path(forResource: "certificate", ofType: "cer"),
              let certData = try? Data(contentsOf: URL(fileURLWithPath: certPath)) else {
            fatalError("証明書が見つかりません")
        }
        return certData
    }
}
```

## テストのベストプラクティス

### モックの作成

テスト可能な設計を心がけます:

```swift
// プロトコルを使用
protocol HTTPClient {
    func data(for request: URLRequest) async throws -> (Data, URLResponse)
}

extension URLSession: HTTPClient {}

// モック実装
class MockHTTPClient: HTTPClient {
    var mockData: Data?
    var mockResponse: URLResponse?
    var mockError: Error?

    func data(for request: URLRequest) async throws -> (Data, URLResponse) {
        if let error = mockError {
            throw error
        }

        guard let data = mockData, let response = mockResponse else {
            throw NetworkError.noData
        }

        return (data, response)
    }
}

// テスト
func testSuccessfulRequest() async throws {
    let mockClient = MockHTTPClient()
    let user = User(id: 1, name: "Test", email: "test@example.com", createdAt: Date())
    mockClient.mockData = try JSONEncoder().encode(user)
    mockClient.mockResponse = HTTPURLResponse(
        url: URL(string: "https://api.example.com")!,
        statusCode: 200,
        httpVersion: nil,
        headerFields: nil
    )

    let service = APIServiceImpl(httpClient: mockClient)
    let result: User = try await service.request(UserEndpoint.getUser(id: 1))

    XCTAssertEqual(result.id, 1)
}
```

## よくある問題と解決策

### 問題1: Core Data のスレッド安全性

```swift
// ❌ 異なるスレッドからアクセス
func badExample() {
    let user = User.create(in: viewContext, name: "Test", email: "test@example.com")

    DispatchQueue.global().async {
        user.name = "Updated"  // クラッシュ!
    }
}

// ✅ バックグラウンドコンテキストを使用
func goodExample() async {
    try await CoreDataManager.shared.performBackgroundTask { context in
        let user = User.create(in: context, name: "Test", email: "test@example.com")
        user.name = "Updated"
    }
}
```

### 問題2: Task のキャンセル忘れ

```swift
// ❌ Task がキャンセルされない
class BadViewModel {
    var task: Task<Void, Never>?

    func loadData() {
        task = Task {
            let data = try? await fetchData()
        }
    }
}

// ✅ deinit でキャンセル
class GoodViewModel {
    var task: Task<Void, Never>?

    func loadData() {
        task?.cancel()

        task = Task {
            let data = try? await fetchData()
            guard !Task.isCancelled else { return }
            // データを使用
        }
    }

    deinit {
        task?.cancel()
    }
}
```

### 問題3: JSONDecoder のエラーハンドリング

```swift
// ❌ エラーの詳細が不明
do {
    let user = try JSONDecoder().decode(User.self, from: data)
} catch {
    print("デコードに失敗しました")
}

// ✅ 詳細なエラー情報
do {
    let user = try JSONDecoder().decode(User.self, from: data)
} catch let DecodingError.keyNotFound(key, context) {
    print("キーが見つかりません: \(key.stringValue)")
    print("コンテキスト: \(context.debugDescription)")
} catch let DecodingError.typeMismatch(type, context) {
    print("型が一致しません: \(type)")
    print("コンテキスト: \(context.debugDescription)")
} catch let DecodingError.valueNotFound(type, context) {
    print("値が見つかりません: \(type)")
    print("コンテキスト: \(context.debugDescription)")
} catch {
    print("その他のエラー: \(error)")
}
```

## まとめ

本書を通じて、iOS のネットワーク通信とデータ永続化について学びました:

### 第1部: ネットワーク通信
- URLSession の基本
- 型安全な API 設計
- エラーハンドリングとリトライ
- WebSocket によるリアルタイム通信

### 第2部: データ永続化
- UserDefaults と FileManager
- Keychain によるセキュアな保存
- Core Data の実践的な使用
- Realm と SQLite の活用

### 第3部: キャッシュとオフライン対応
- 効率的なキャッシュ戦略
- オフラインファースト設計
- 同期とデータの整合性

### 第4部: ベストプラクティス
- アーキテクチャの設計
- エラーハンドリング
- パフォーマンス最適化
- セキュリティ対策

これらの知識を活用して、高品質な iOS アプリケーションを開発してください。

## 次のステップ

さらに学習を深めるために:

1. **プロジェクトで実践**: 本書で学んだパターンを実際のアプリに適用
2. **WWDC セッション**: Apple の最新技術を学ぶ
3. **オープンソース**: 人気ライブラリのコードを読む
4. **コミュニティ**: iOS 開発コミュニティに参加

## 参考リソース

### 公式ドキュメント
- [URLSession - Apple Developer](https://developer.apple.com/documentation/foundation/urlsession)
- [Core Data - Apple Developer](https://developer.apple.com/documentation/coredata)
- [Keychain Services - Apple Developer](https://developer.apple.com/documentation/security/keychain_services)

### WWDC セッション
- [What's new in Foundation](https://developer.apple.com/videos/)
- [Core Data Best Practices](https://developer.apple.com/videos/)
- [Networking in Swift](https://developer.apple.com/videos/)

### コミュニティ
- [Swift Forums](https://forums.swift.org/)
- [iOS Dev Weekly](https://iosdevweekly.com/)
- [Hacking with Swift](https://www.hackingwithswift.com/)

最後まで読んでいただき、ありがとうございました！
