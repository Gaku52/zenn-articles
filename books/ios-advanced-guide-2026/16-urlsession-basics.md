---
title: "URLSession高度な活用"
---

# URLSession高度な活用

URLSessionは、iOSでHTTP通信を行うための標準APIです。本章では、URLSessionの基本から高度な活用方法まで、実践的なパターンを解説します。

## URLSessionの概要

URLSessionは、以下の機能を提供します。

- データタスク: JSON APIのリクエスト・レスポンス
- ダウンロードタスク: 大きなファイルのダウンロード
- アップロードタスク: 画像や動画のアップロード
- WebSocketタスク: リアルタイム双方向通信
- バックグラウンド転送: アプリが非アクティブでもダウンロード継続

Appleの公式ドキュメントでは、ネットワーク通信の第一選択としてURLSessionを推奨しています。

https://developer.apple.com/documentation/foundation/urlsession

## 基本的なGETリクエスト

まずは最もシンプルなGETリクエストから見ていきましょう。

```swift
import Foundation

// 最もシンプルな実装（async/await使用）
func fetchData() async throws -> Data {
    let url = URL(string: "https://api.example.com/users")!
    let (data, _) = try await URLSession.shared.data(from: url)
    return data
}

// Codableと組み合わせた実装
struct User: Codable {
    let id: Int
    let name: String
    let email: String
}

func fetchUsers() async throws -> [User] {
    let url = URL(string: "https://api.example.com/users")!
    let (data, _) = try await URLSession.shared.data(from: url)
    let decoder = JSONDecoder()
    decoder.keyDecodingStrategy = .convertFromSnakeCase
    return try decoder.decode([User].self, from: data)
}
```

### レスポンスの検証

HTTPステータスコードを検証し、適切なエラーハンドリングを行います。

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

func fetchUser(id: Int) async throws -> User {
    guard let url = URL(string: "https://api.example.com/users/\(id)") else {
        throw NetworkError.invalidURL
    }

    let (data, response) = try await URLSession.shared.data(from: url)

    guard let httpResponse = response as? HTTPURLResponse else {
        throw NetworkError.invalidResponse
    }

    guard (200...299).contains(httpResponse.statusCode) else {
        throw NetworkError.httpError(httpResponse.statusCode)
    }

    do {
        return try JSONDecoder().decode(User.self, from: data)
    } catch {
        throw NetworkError.decodingError(error)
    }
}
```

## POSTリクエスト

データを送信するPOSTリクエストの実装パターンです。

```swift
struct CreateUserRequest: Codable {
    let name: String
    let email: String
    let age: Int
}

func createUser(_ userData: CreateUserRequest) async throws -> User {
    guard let url = URL(string: "https://api.example.com/users") else {
        throw NetworkError.invalidURL
    }

    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")

    let encoder = JSONEncoder()
    encoder.keyEncodingStrategy = .convertToSnakeCase
    request.httpBody = try encoder.encode(userData)

    let (data, response) = try await URLSession.shared.data(for: request)

    guard let httpResponse = response as? HTTPURLResponse,
          (200...299).contains(httpResponse.statusCode) else {
        throw NetworkError.invalidResponse
    }

    return try JSONDecoder().decode(User.self, from: data)
}
```

### PATCH/PUTリクエスト

既存リソースの更新パターンです。

```swift
struct UpdateUserRequest: Codable {
    let name: String?
    let email: String?
    let age: Int?
}

func updateUser(id: Int, updates: UpdateUserRequest) async throws -> User {
    guard let url = URL(string: "https://api.example.com/users/\(id)") else {
        throw NetworkError.invalidURL
    }

    var request = URLRequest(url: url)
    request.httpMethod = "PATCH" // 部分更新にはPATCH、全体更新にはPUT
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.httpBody = try JSONEncoder().encode(updates)

    let (data, response) = try await URLSession.shared.data(for: request)

    guard let httpResponse = response as? HTTPURLResponse,
          (200...299).contains(httpResponse.statusCode) else {
        throw NetworkError.invalidResponse
    }

    return try JSONDecoder().decode(User.self, from: data)
}
```

## URLSessionConfigurationの活用

カスタムURLSessionを作成し、タイムアウトやキャッシュポリシーを設定します。

```swift
class NetworkManager {
    static let shared = NetworkManager()

    private let session: URLSession

    private init() {
        let configuration = URLSessionConfiguration.default

        // タイムアウト設定
        configuration.timeoutIntervalForRequest = 30 // リクエストタイムアウト
        configuration.timeoutIntervalForResource = 300 // リソース全体のタイムアウト

        // 接続待機
        configuration.waitsForConnectivity = true

        // キャッシュポリシー
        configuration.requestCachePolicy = .reloadIgnoringLocalCacheData
        configuration.urlCache = URLCache(
            memoryCapacity: 50 * 1024 * 1024, // 50MB
            diskCapacity: 100 * 1024 * 1024   // 100MB
        )

        // HTTPヘッダーのデフォルト値
        configuration.httpAdditionalHeaders = [
            "User-Agent": "MyApp/1.0",
            "Accept-Language": Locale.current.languageCode ?? "en"
        ]

        // 最大接続数
        configuration.httpMaximumConnectionsPerHost = 4

        self.session = URLSession(configuration: configuration)
    }

    func request<T: Decodable>(_ url: URL) async throws -> T {
        let (data, response) = try await session.data(from: url)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.invalidResponse
        }

        return try JSONDecoder().decode(T.self, from: data)
    }
}
```

### バックグラウンドセッション

アプリがバックグラウンドでもダウンロードを継続する設定です。

```swift
class BackgroundDownloadManager: NSObject {
    static let shared = BackgroundDownloadManager()

    private var session: URLSession!
    private var downloadTasks: [URLSessionDownloadTask] = []

    override private init() {
        super.init()

        let configuration = URLSessionConfiguration.background(
            withIdentifier: "com.example.app.background-download"
        )
        configuration.isDiscretionary = false // Wi-Fi待機しない
        configuration.sessionSendsLaunchEvents = true

        session = URLSession(
            configuration: configuration,
            delegate: self,
            delegateQueue: nil
        )
    }

    func startDownload(url: URL) {
        let task = session.downloadTask(with: url)
        task.resume()
        downloadTasks.append(task)
    }
}

extension BackgroundDownloadManager: URLSessionDownloadDelegate {
    func urlSession(
        _ session: URLSession,
        downloadTask: URLSessionDownloadTask,
        didFinishDownloadingTo location: URL
    ) {
        // ダウンロード完了時の処理
        guard let destinationURL = FileManager.default.urls(
            for: .documentDirectory,
            in: .userDomainMask
        ).first?.appendingPathComponent("downloaded_file.zip") else {
            return
        }

        try? FileManager.default.moveItem(at: location, to: destinationURL)
        print("Download completed: \(destinationURL)")
    }

    func urlSession(
        _ session: URLSession,
        downloadTask: URLSessionDownloadTask,
        didWriteData bytesWritten: Int64,
        totalBytesWritten: Int64,
        totalBytesExpectedToWrite: Int64
    ) {
        // 進捗状況の通知
        let progress = Double(totalBytesWritten) / Double(totalBytesExpectedToWrite)
        print("Download progress: \(progress * 100)%")
    }
}
```

## 認証ヘッダーの追加

Bearer トークンを使った認証の実装パターンです。

```swift
class AuthenticatedNetworkManager {
    private let session: URLSession
    private var accessToken: String?

    init() {
        let configuration = URLSessionConfiguration.default
        self.session = URLSession(configuration: configuration)
    }

    func setAccessToken(_ token: String) {
        self.accessToken = token
    }

    func request<T: Decodable>(
        url: URL,
        method: String = "GET",
        body: Data? = nil
    ) async throws -> T {
        var request = URLRequest(url: url)
        request.httpMethod = method
        request.httpBody = body

        // 認証ヘッダーの追加
        if let token = accessToken {
            request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }

        if body != nil {
            request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        }

        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }

        // 401エラー時の処理
        if httpResponse.statusCode == 401 {
            throw NetworkError.httpError(401)
        }

        guard (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.httpError(httpResponse.statusCode)
        }

        return try JSONDecoder().decode(T.self, from: data)
    }
}
```

## ファイルアップロード

マルチパートフォームデータでの画像アップロード実装です。

```swift
func uploadImage(_ imageData: Data, filename: String) async throws -> UploadResponse {
    guard let url = URL(string: "https://api.example.com/upload") else {
        throw NetworkError.invalidURL
    }

    let boundary = "Boundary-\(UUID().uuidString)"
    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("multipart/form-data; boundary=\(boundary)", forHTTPHeaderField: "Content-Type")

    var body = Data()

    // ファイルパート
    body.append("--\(boundary)\r\n".data(using: .utf8)!)
    body.append("Content-Disposition: form-data; name=\"file\"; filename=\"\(filename)\"\r\n".data(using: .utf8)!)
    body.append("Content-Type: image/jpeg\r\n\r\n".data(using: .utf8)!)
    body.append(imageData)
    body.append("\r\n".data(using: .utf8)!)

    // その他のフィールド
    body.append("--\(boundary)\r\n".data(using: .utf8)!)
    body.append("Content-Disposition: form-data; name=\"description\"\r\n\r\n".data(using: .utf8)!)
    body.append("Uploaded from iOS app\r\n".data(using: .utf8)!)

    body.append("--\(boundary)--\r\n".data(using: .utf8)!)

    request.httpBody = body

    let (data, response) = try await URLSession.shared.data(for: request)

    guard let httpResponse = response as? HTTPURLResponse,
          (200...299).contains(httpResponse.statusCode) else {
        throw NetworkError.invalidResponse
    }

    return try JSONDecoder().decode(UploadResponse.self, from: data)
}

struct UploadResponse: Codable {
    let fileUrl: String
    let fileId: String
}
```

## ダウンロード進捗の監視

大きなファイルのダウンロード進捗を監視する実装です。

```swift
class DownloadManager: NSObject, ObservableObject {
    @Published var downloadProgress: Double = 0.0
    @Published var isDownloading = false

    private var downloadTask: URLSessionDownloadTask?
    private lazy var session: URLSession = {
        let configuration = URLSessionConfiguration.default
        return URLSession(configuration: configuration, delegate: self, delegateQueue: nil)
    }()

    func startDownload(url: URL) {
        isDownloading = true
        downloadProgress = 0.0
        downloadTask = session.downloadTask(with: url)
        downloadTask?.resume()
    }

    func cancelDownload() {
        downloadTask?.cancel()
        isDownloading = false
        downloadProgress = 0.0
    }
}

extension DownloadManager: URLSessionDownloadDelegate {
    func urlSession(
        _ session: URLSession,
        downloadTask: URLSessionDownloadTask,
        didFinishDownloadingTo location: URL
    ) {
        DispatchQueue.main.async {
            self.isDownloading = false
            self.downloadProgress = 1.0
        }

        // ファイルを永続的な場所に移動
        guard let documentsPath = FileManager.default.urls(
            for: .documentDirectory,
            in: .userDomainMask
        ).first else { return }

        let destinationURL = documentsPath.appendingPathComponent("downloaded.pdf")

        try? FileManager.default.removeItem(at: destinationURL)
        try? FileManager.default.moveItem(at: location, to: destinationURL)
    }

    func urlSession(
        _ session: URLSession,
        downloadTask: URLSessionDownloadTask,
        didWriteData bytesWritten: Int64,
        totalBytesWritten: Int64,
        totalBytesExpectedToWrite: Int64
    ) {
        let progress = Double(totalBytesWritten) / Double(totalBytesExpectedToWrite)

        DispatchQueue.main.async {
            self.downloadProgress = progress
        }
    }

    func urlSession(_ session: URLSession, task: URLSessionTask, didCompleteWithError error: Error?) {
        if let error = error {
            print("Download error: \(error.localizedDescription)")
            DispatchQueue.main.async {
                self.isDownloading = false
            }
        }
    }
}
```

## よくある間違いと解決策

### 間違い1: メインスレッドでの同期通信

```swift
// NG: メインスレッドをブロックする
func fetchDataSync() -> Data? {
    let url = URL(string: "https://api.example.com/data")!
    return try? Data(contentsOf: url) // これは使わない！
}

// OK: async/awaitで非同期処理
func fetchDataAsync() async throws -> Data {
    let url = URL(string: "https://api.example.com/data")!
    let (data, _) = try await URLSession.shared.data(from: url)
    return data
}
```

### 間違い2: エラーハンドリングの不足

```swift
// NG: エラーを無視
func fetchUser() async -> User? {
    let url = URL(string: "https://api.example.com/user")!
    let (data, _) = try? await URLSession.shared.data(from: url)
    return try? JSONDecoder().decode(User.self, from: data ?? Data())
}

// OK: 適切なエラーハンドリング
func fetchUser() async throws -> User {
    guard let url = URL(string: "https://api.example.com/user") else {
        throw NetworkError.invalidURL
    }

    let (data, response) = try await URLSession.shared.data(from: url)

    guard let httpResponse = response as? HTTPURLResponse,
          (200...299).contains(httpResponse.statusCode) else {
        throw NetworkError.invalidResponse
    }

    do {
        return try JSONDecoder().decode(User.self, from: data)
    } catch {
        throw NetworkError.decodingError(error)
    }
}
```

### 間違い3: URLSessionの過剰な生成

```swift
// NG: 毎回新しいセッションを作成
func fetchData1() async throws -> Data {
    let session = URLSession(configuration: .default)
    let (data, _) = try await session.data(from: URL(string: "https://api.example.com/1")!)
    return data
}

func fetchData2() async throws -> Data {
    let session = URLSession(configuration: .default) // 無駄
    let (data, _) = try await session.data(from: URL(string: "https://api.example.com/2")!)
    return data
}

// OK: シングルトンで再利用
class NetworkManager {
    static let shared = NetworkManager()
    private let session: URLSession

    private init() {
        session = URLSession(configuration: .default)
    }

    func fetchData(from url: URL) async throws -> Data {
        let (data, _) = try await session.data(from: url)
        return data
    }
}
```

## 実装チェックリスト

URLSessionを使ったネットワーク通信実装時に確認すべき項目です。

- [ ] URLの検証を行っている
- [ ] HTTPステータスコードをチェックしている
- [ ] エラーハンドリングを適切に実装している
- [ ] タイムアウト設定を適切に行っている
- [ ] メインスレッドをブロックしていない
- [ ] 認証トークンを安全に管理している
- [ ] レスポンスデータの検証を行っている
- [ ] キャッシュポリシーを適切に設定している
- [ ] バックグラウンドダウンロードが必要な場合は実装している
- [ ] ダウンロード/アップロードの進捗を表示している

## まとめ

本章では、URLSessionの高度な活用方法を学びました。

- 基本的なGET/POST/PATCH/DELETEリクエストの実装
- URLSessionConfigurationによるカスタマイズ
- バックグラウンドセッションでのダウンロード継続
- 認証ヘッダーの追加パターン
- ファイルアップロード・ダウンロードの実装
- 進捗監視の実装方法
- よくある間違いと対策

URLSessionは標準APIとして強力な機能を提供しており、適切に実装すればほとんどのユースケースに対応できます。次章では、Alamofireを使った高度なAPI通信について解説します。

## 参考リンク

- [URLSession - Apple Developer Documentation](https://developer.apple.com/documentation/foundation/urlsession)
- [URLSessionConfiguration - Apple Developer Documentation](https://developer.apple.com/documentation/foundation/urlsessionconfiguration)
- [URLSessionTask - Apple Developer Documentation](https://developer.apple.com/documentation/foundation/urlsessiontask)
