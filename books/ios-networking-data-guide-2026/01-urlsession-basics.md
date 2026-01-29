---
title: "URLSession 基礎: HTTP 通信の実装"
---

# URLSession 基礎: HTTP 通信の実装

URLSession は iOS アプリでネットワーク通信を行うための標準的な API です。この章では、URLSession を使った基本的な HTTP 通信の実装方法を学びます。

## URLSession とは

URLSession は、HTTP/HTTPS プロトコルを使用してデータをダウンロード・アップロードするための Apple 公式の API です。以下の特徴があります:

- **async/await 対応**: 非同期処理を直感的に記述できる
- **バックグラウンド転送**: アプリが終了しても転送を継続できる
- **認証サポート**: 基本認証、証明書ベース認証などに対応
- **キャッシュ管理**: HTTP キャッシュを自動的に処理

## シンプルな GET リクエスト

最も基本的な GET リクエストの実装から始めましょう。

```swift
struct User: Codable {
    let id: Int
    let name: String
    let email: String
}

func fetchUser(id: Int) async throws -> User {
    // URL の作成
    let url = URL(string: "https://api.example.com/users/\(id)")!

    // データの取得
    let (data, _) = try await URLSession.shared.data(from: url)

    // JSON のデコード
    return try JSONDecoder().decode(User.self, from: data)
}
```

この実装はシンプルですが、いくつかの問題があります:

- URL の生成が失敗する可能性を無視している
- HTTP ステータスコードをチェックしていない
- エラーハンドリングが不十分

## エラーハンドリングを追加

より堅牢な実装に改善しましょう。

```swift
enum NetworkError: Error, LocalizedError {
    case invalidURL
    case invalidResponse
    case httpError(Int)
    case decodingError(Error)

    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "無効な URL です"
        case .invalidResponse:
            return "サーバーからの応答が無効です"
        case .httpError(let code):
            return "HTTP エラー: \(code)"
        case .decodingError(let error):
            return "データの解析に失敗しました: \(error.localizedDescription)"
        }
    }
}

func fetchUserWithValidation(id: Int) async throws -> User {
    // URL の生成とバリデーション
    guard let url = URL(string: "https://api.example.com/users/\(id)") else {
        throw NetworkError.invalidURL
    }

    // データの取得
    let (data, response) = try await URLSession.shared.data(from: url)

    // レスポンスの検証
    guard let httpResponse = response as? HTTPURLResponse else {
        throw NetworkError.invalidResponse
    }

    // ステータスコードのチェック
    guard (200...299).contains(httpResponse.statusCode) else {
        throw NetworkError.httpError(httpResponse.statusCode)
    }

    // JSON のデコード
    do {
        return try JSONDecoder().decode(User.self, from: data)
    } catch {
        throw NetworkError.decodingError(error)
    }
}
```

## POST リクエスト

次に、データを送信する POST リクエストを実装します。

```swift
struct CreateUserRequest: Codable {
    let name: String
    let email: String
    let password: String
}

func createUser(request: CreateUserRequest) async throws -> User {
    guard let url = URL(string: "https://api.example.com/users") else {
        throw NetworkError.invalidURL
    }

    // URLRequest の作成
    var urlRequest = URLRequest(url: url)
    urlRequest.httpMethod = "POST"
    urlRequest.setValue("application/json", forHTTPHeaderField: "Content-Type")

    // リクエストボディのエンコード
    do {
        urlRequest.httpBody = try JSONEncoder().encode(request)
    } catch {
        throw NetworkError.encodingError(error)
    }

    // リクエストの送信
    let (data, response) = try await URLSession.shared.data(for: urlRequest)

    // レスポンスの検証
    guard let httpResponse = response as? HTTPURLResponse,
          (200...299).contains(httpResponse.statusCode) else {
        throw NetworkError.invalidResponse
    }

    // レスポンスのデコード
    return try JSONDecoder().decode(User.self, from: data)
}
```

## URLSessionConfiguration のカスタマイズ

URLSession の動作は、URLSessionConfiguration を使ってカスタマイズできます。

```swift
class NetworkManager {
    static let shared = NetworkManager()

    private let session: URLSession

    private init() {
        let configuration = URLSessionConfiguration.default

        // タイムアウト設定
        configuration.timeoutIntervalForRequest = 30  // リクエストタイムアウト: 30秒
        configuration.timeoutIntervalForResource = 300  // リソース全体のタイムアウト: 5分

        // 接続性の設定
        configuration.waitsForConnectivity = true  // 接続を待つ
        configuration.allowsCellularAccess = true  // セルラー接続を許可

        // キャッシュ設定
        configuration.requestCachePolicy = .returnCacheDataElseLoad

        // HTTP ヘッダー設定
        configuration.httpAdditionalHeaders = [
            "Accept": "application/json",
            "User-Agent": "MyApp/1.0"
        ]

        // Cookie 設定
        configuration.httpCookieAcceptPolicy = .always

        // HTTP/2 設定
        configuration.httpMaximumConnectionsPerHost = 5

        self.session = URLSession(configuration: configuration)
    }

    func request<T: Decodable>(url: URL) async throws -> T {
        let (data, response) = try await session.data(from: url)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.invalidResponse
        }

        return try JSONDecoder().decode(T.self, from: data)
    }
}
```

## 各種 HTTP メソッドの実装

PUT、DELETE、PATCH などのメソッドも同様に実装できます。

```swift
extension NetworkManager {
    // PUT リクエスト
    func put<T: Encodable, U: Decodable>(
        url: URL,
        body: T
    ) async throws -> U {
        var request = URLRequest(url: url)
        request.httpMethod = "PUT"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = try JSONEncoder().encode(body)

        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.invalidResponse
        }

        return try JSONDecoder().decode(U.self, from: data)
    }

    // DELETE リクエスト
    func delete(url: URL) async throws {
        var request = URLRequest(url: url)
        request.httpMethod = "DELETE"

        let (_, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.invalidResponse
        }
    }

    // PATCH リクエスト
    func patch<T: Encodable, U: Decodable>(
        url: URL,
        body: T
    ) async throws -> U {
        var request = URLRequest(url: url)
        request.httpMethod = "PATCH"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = try JSONEncoder().encode(body)

        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.invalidResponse
        }

        return try JSONDecoder().decode(U.self, from: data)
    }
}
```

## クエリパラメータの扱い

URL にクエリパラメータを追加する方法を見てみましょう。

```swift
extension URL {
    func appending(queryItems: [URLQueryItem]) -> URL? {
        var components = URLComponents(url: self, resolvingAgainstBaseURL: false)
        components?.queryItems = queryItems
        return components?.url
    }
}

// 使用例
func searchUsers(query: String, page: Int) async throws -> [User] {
    let baseURL = URL(string: "https://api.example.com/users/search")!

    let queryItems = [
        URLQueryItem(name: "q", value: query),
        URLQueryItem(name: "page", value: "\(page)"),
        URLQueryItem(name: "limit", value: "20")
    ]

    guard let url = baseURL.appending(queryItems: queryItems) else {
        throw NetworkError.invalidURL
    }

    let (data, response) = try await URLSession.shared.data(from: url)

    guard let httpResponse = response as? HTTPURLResponse,
          (200...299).contains(httpResponse.statusCode) else {
        throw NetworkError.invalidResponse
    }

    return try JSONDecoder().decode([User].self, from: data)
}
```

## JSONDecoder のカスタマイズ

日付やキー名の変換など、JSONDecoder をカスタマイズする方法を学びましょう。

```swift
extension JSONDecoder {
    static var `default`: JSONDecoder {
        let decoder = JSONDecoder()

        // 日付のデコード戦略
        decoder.dateDecodingStrategy = .iso8601

        // キー名の変換戦略（snake_case → camelCase）
        decoder.keyDecodingStrategy = .convertFromSnakeCase

        return decoder
    }
}

// カスタム日付フォーマット
extension JSONDecoder {
    static var customDate: JSONDecoder {
        let decoder = JSONDecoder()
        let formatter = DateFormatter()
        formatter.dateFormat = "yyyy-MM-dd HH:mm:ss"
        formatter.timeZone = TimeZone(identifier: "Asia/Tokyo")
        decoder.dateDecodingStrategy = .formatted(formatter)
        return decoder
    }
}

// 使用例
struct Post: Codable {
    let id: Int
    let title: String
    let content: String
    let createdAt: Date  // ISO8601 形式でデコード
    let userId: Int      // user_id から自動変換
}

func fetchPost(id: Int) async throws -> Post {
    let url = URL(string: "https://api.example.com/posts/\(id)")!
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder.default.decode(Post.self, from: data)
}
```

## まとめ

この章では、URLSession を使った基本的な HTTP 通信の実装方法を学びました:

- GET、POST、PUT、DELETE などの HTTP メソッドの実装
- エラーハンドリングの重要性
- URLSessionConfiguration によるカスタマイズ
- クエリパラメータとリクエストボディの扱い
- JSONDecoder のカスタマイズ

次の章では、これらの知識を基に、より保守性の高い API クライアントのアーキテクチャを構築していきます。

## 参考リソース

- [URLSession - Apple Developer](https://developer.apple.com/documentation/foundation/urlsession)
- [URLSessionConfiguration - Apple Developer](https://developer.apple.com/documentation/foundation/urlsessionconfiguration)
- [JSONDecoder - Apple Developer](https://developer.apple.com/documentation/foundation/jsondecoder)
