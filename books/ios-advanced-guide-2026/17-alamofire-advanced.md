---
title: "Alamofire高度な活用"
---

# Alamofire高度な活用

Alamofireは、SwiftでHTTP通信を簡潔に記述できる人気のライブラリです。URLSessionをラップし、より直感的で強力なAPIを提供します。本章では、Alamofireの高度な機能と実践的な使い方を解説します。

## Alamofireの導入

### Swift Package Managerによるインストール

Xcodeプロジェクトにて、File > Add Packages...から以下のURLを指定してAlamofireを追加できます。

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/Alamofire/Alamofire.git", from: "5.8.0")
]
```

Swift Package Managerを使用することで、依存関係の管理が容易になり、プロジェクトのセットアップが簡素化されます。

### 基本設定

Alamofireを使用する際は、まずインポートステートメントを追加します。

```swift
import Alamofire
```

## Sessionの設定

Alamofireでは、カスタムSessionを作成することで、タイムアウト、キャッシュポリシー、証明書検証などの詳細な設定が可能です。

```swift
import Alamofire
import Foundation

class NetworkConfiguration {
    static let shared = NetworkConfiguration()

    let session: Session

    private init() {
        // URLSessionConfiguration のカスタマイズ
        let configuration = URLSessionConfiguration.default
        configuration.timeoutIntervalForRequest = 30
        configuration.timeoutIntervalForResource = 60
        configuration.requestCachePolicy = .reloadIgnoringLocalCacheData

        // 証明書ピンニングの設定（本番環境推奨）
        let evaluators: [String: ServerTrustEvaluating] = [
            "api.example.com": PinnedCertificatesTrustEvaluator()
        ]

        let serverTrustManager = ServerTrustManager(evaluators: evaluators)

        // Sessionの作成
        session = Session(
            configuration: configuration,
            serverTrustManager: serverTrustManager,
            eventMonitors: [NetworkLogger()]
        )
    }
}

// ネットワークログ用のEventMonitor
class NetworkLogger: EventMonitor {
    func requestDidResume(_ request: Request) {
        let body = request.request.flatMap { $0.httpBody.map { String(decoding: $0, as: UTF8.self) } } ?? "None"
        print("Request Started: \(request)")
        print("Body: \(body)")
    }

    func request<Value>(_ request: DataRequest, didParseResponse response: DataResponse<Value, AFError>) {
        print("Response Received: \(response.debugDescription)")
    }
}
```

## Requestインターセプター

全てのリクエストに共通処理を適用するために、RequestInterceptorを使用します。認証トークンの自動付与やトークンリフレッシュなどに活用できます。

```swift
import Alamofire

class AuthInterceptor: RequestInterceptor {

    // リクエストの前処理（認証ヘッダーの追加など）
    func adapt(_ urlRequest: URLRequest, for session: Session, completion: @escaping (Result<URLRequest, Error>) -> Void) {
        var urlRequest = urlRequest

        // Keychainからトークンを取得
        if let token = KeychainManager.shared.loadToken() {
            urlRequest.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }

        // 共通ヘッダーの追加
        urlRequest.setValue("application/json", forHTTPHeaderField: "Content-Type")
        urlRequest.setValue("iOS", forHTTPHeaderField: "X-Platform")

        completion(.success(urlRequest))
    }

    // リトライロジック（401エラー時のトークンリフレッシュなど）
    func retry(_ request: Request, for session: Session, dueTo error: Error, completion: @escaping (RetryResult) -> Void) {
        guard let response = request.task?.response as? HTTPURLResponse else {
            completion(.doNotRetryWithError(error))
            return
        }

        // 401エラーの場合、トークンをリフレッシュしてリトライ
        if response.statusCode == 401 {
            refreshToken { success in
                if success {
                    completion(.retry)
                } else {
                    completion(.doNotRetryWithError(error))
                }
            }
        } else if response.statusCode >= 500 {
            // サーバーエラーの場合、指数バックオフでリトライ
            completion(.retryWithDelay(2.0))
        } else {
            completion(.doNotRetryWithError(error))
        }
    }

    private func refreshToken(completion: @escaping (Bool) -> Void) {
        // トークンリフレッシュの実装
        // 実際の実装では、リフレッシュトークンを使用して新しいアクセストークンを取得
        completion(false)
    }
}
```

## レスポンスバリデーション

Alamofireでは、レスポンスの妥当性を検証するためのバリデーション機能が用意されています。

```swift
import Alamofire

class APIClient {
    private let session: Session

    init() {
        let interceptor = AuthInterceptor()
        session = NetworkConfiguration.shared.session
    }

    func fetchUsers() async throws -> [User] {
        try await session.request("https://api.example.com/users")
            .validate(statusCode: 200..<300)
            .validate(contentType: ["application/json"])
            .serializingDecodable([User].self)
            .value
    }

    // カスタムバリデーション
    func fetchWithCustomValidation() async throws -> Data {
        try await session.request("https://api.example.com/data")
            .validate { request, response, data in
                // カスタム検証ロジック
                guard response.statusCode == 200 else {
                    return .failure(AFError.responseValidationFailed(reason: .unacceptableStatusCode(code: response.statusCode)))
                }

                guard let data = data, !data.isEmpty else {
                    return .failure(AFError.responseValidationFailed(reason: .dataFileNil))
                }

                return .success(())
            }
            .serializingData()
            .value
    }
}
```

## マルチパートアップロード

画像やファイルをアップロードする際は、マルチパートフォームデータを使用します。

```swift
import Alamofire
import UIKit

class FileUploadService {
    private let session: Session

    init() {
        session = NetworkConfiguration.shared.session
    }

    func uploadImage(_ image: UIImage, withMetadata metadata: [String: String]) async throws -> UploadResponse {
        guard let imageData = image.jpegData(compressionQuality: 0.8) else {
            throw UploadError.invalidImageData
        }

        return try await session.upload(
            multipartFormData: { multipartFormData in
                // 画像データの追加
                multipartFormData.append(
                    imageData,
                    withName: "file",
                    fileName: "image.jpg",
                    mimeType: "image/jpeg"
                )

                // メタデータの追加
                for (key, value) in metadata {
                    if let data = value.data(using: .utf8) {
                        multipartFormData.append(data, withName: key)
                    }
                }
            },
            to: "https://api.example.com/upload"
        )
        .validate()
        .serializingDecodable(UploadResponse.self)
        .value
    }

    // 進捗監視付きアップロード
    func uploadWithProgress(_ image: UIImage, progress: @escaping (Double) -> Void) async throws -> UploadResponse {
        guard let imageData = image.jpegData(compressionQuality: 0.8) else {
            throw UploadError.invalidImageData
        }

        let upload = session.upload(
            multipartFormData: { multipartFormData in
                multipartFormData.append(
                    imageData,
                    withName: "file",
                    fileName: "image.jpg",
                    mimeType: "image/jpeg"
                )
            },
            to: "https://api.example.com/upload"
        )

        // 進捗の監視
        upload.uploadProgress { progressData in
            progress(progressData.fractionCompleted)
        }

        return try await upload
            .validate()
            .serializingDecodable(UploadResponse.self)
            .value
    }
}

enum UploadError: Error {
    case invalidImageData
}

struct UploadResponse: Decodable {
    let fileId: String
    let url: String
}
```

## ダウンロード進捗監視

大きなファイルをダウンロードする際は、進捗を監視してユーザーにフィードバックを提供することが推奨されます。

```swift
import Alamofire

class DownloadService {
    private let session: Session

    init() {
        session = NetworkConfiguration.shared.session
    }

    func downloadFile(from url: String, progress: @escaping (Double) -> Void) async throws -> URL {
        let destination: DownloadRequest.Destination = { _, _ in
            let documentsURL = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
            let fileURL = documentsURL.appendingPathComponent("downloaded_file.zip")

            return (fileURL, [.removePreviousFile, .createIntermediateDirectories])
        }

        let download = session.download(url, to: destination)

        // ダウンロード進捗の監視
        download.downloadProgress { progressData in
            progress(progressData.fractionCompleted)
        }

        return try await download
            .validate()
            .serializingDownloadedFileURL()
            .value
    }

    // レジューム可能なダウンロード
    func resumableDownload(from url: String, resumeData: Data? = nil) async throws -> URL {
        let destination: DownloadRequest.Destination = { _, _ in
            let documentsURL = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
            let fileURL = documentsURL.appendingPathComponent("downloaded_file.zip")

            return (fileURL, [.removePreviousFile, .createIntermediateDirectories])
        }

        let download: DownloadRequest

        if let resumeData = resumeData {
            download = session.download(resumingWith: resumeData, to: destination)
        } else {
            download = session.download(url, to: destination)
        }

        return try await download
            .validate()
            .serializingDownloadedFileURL()
            .value
    }
}
```

## パフォーマンス最適化

### リクエストの並列実行

複数のAPIリクエストを並列で実行することで、全体の処理時間を短縮できます。

```swift
import Alamofire

class ParallelRequestService {
    func fetchMultipleResources() async throws -> (users: [User], posts: [Post], comments: [Comment]) {
        async let users = fetchUsers()
        async let posts = fetchPosts()
        async let comments = fetchComments()

        return try await (users, posts, comments)
    }

    private func fetchUsers() async throws -> [User] {
        try await AF.request("https://api.example.com/users")
            .validate()
            .serializingDecodable([User].self)
            .value
    }

    private func fetchPosts() async throws -> [Post] {
        try await AF.request("https://api.example.com/posts")
            .validate()
            .serializingDecodable([Post].self)
            .value
    }

    private func fetchComments() async throws -> [Comment] {
        try await AF.request("https://api.example.com/comments")
            .validate()
            .serializingDecodable([Comment].self)
            .value
    }
}
```

## チェックリスト

Alamofireを使用する際のベストプラクティスチェックリスト：

- [ ] Swift Package Managerでの依存関係管理を設定済み
- [ ] カスタムSessionで適切なタイムアウト値を設定済み
- [ ] RequestInterceptorで認証トークンの自動付与を実装済み
- [ ] レスポンスバリデーションを全てのリクエストに適用済み
- [ ] 本番環境で証明書ピンニングを有効化済み
- [ ] EventMonitorでネットワークログを記録済み
- [ ] マルチパートアップロードで進捗監視を実装済み
- [ ] ダウンロード処理でレジューム機能を実装済み
- [ ] エラーハンドリングで適切なリトライロジックを実装済み
- [ ] 並列リクエストで処理時間を最適化済み

## まとめ

Alamofireを活用することで、ネットワーク通信のコードが簡潔になり、開発効率が向上することが期待されます。RequestInterceptorによる共通処理の一元管理、マルチパートアップロード、ダウンロード進捗監視など、Alamofireの高度な機能を適切に使用することで、堅牢なネットワーク層を構築できます。

次章では、APIエラーハンドリングについて詳しく解説します。

## 参考文献

- [Alamofire GitHub Repository](https://github.com/Alamofire/Alamofire)
- [Alamofire Documentation](https://alamofire.github.io/Alamofire/)
- [Apple Developer Documentation - URLSession](https://developer.apple.com/documentation/foundation/urlsession)
