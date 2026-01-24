---
title: "認証・認可の実装"
---

# 認証・認可の実装

セキュアなアプリケーションには、適切な認証・認可の実装が不可欠です。本章では、OAuth 2.0とOpenID Connectを使った現代的な認証フローの実装方法を解説します。

## 認証と認可の違い

### 認証（Authentication）

ユーザーが本人であることを確認するプロセスです。

```plaintext
認証の例:
- ユーザー名とパスワードでログイン
- 生体認証（Face ID / Touch ID）
- OAuthによるソーシャルログイン
```

### 認可（Authorization）

認証されたユーザーが、特定のリソースにアクセスする権限を持つかを確認するプロセスです。

```plaintext
認可の例:
- 管理者のみが削除操作を実行できる
- プレミアムユーザーのみが特定機能を使用できる
- APIトークンのスコープによるアクセス制御
```

## OAuth 2.0 / OpenID Connect

OAuth 2.0は認可の標準規格、OpenID Connect（OIDC）はその上に構築された認証層です。

### 認証フローの概要

```plaintext
1. ユーザーが「ログイン」をタップ
2. アプリが認証サーバーの認証画面を表示
3. ユーザーが認証情報を入力
4. 認証サーバーが認可コードを発行
5. アプリが認可コードをアクセストークンと交換
6. アプリがアクセストークンでAPIにアクセス
```

## ASWebAuthenticationSession の実装

iOS標準のWeb認証APIを使用した実装です。

```swift
// Core/Authentication/AuthManager.swift
import AuthenticationServices

final class AuthManager: NSObject, ObservableObject {
    @Published var isAuthenticated = false
    @Published var user: User?

    private let authURL = URL(string: "https://auth.example.com/oauth/authorize")!
    private let tokenURL = URL(string: "https://auth.example.com/oauth/token")!
    private let clientID = "your-client-id"
    private let redirectURI = "myapp://callback"

    // MARK: - Sign In

    func signIn(presenting viewController: UIViewController) {
        guard var components = URLComponents(url: authURL, resolvingAgainstBaseURL: false) else {
            return
        }

        // 認証リクエストのパラメータを設定
        components.queryItems = [
            URLQueryItem(name: "client_id", value: clientID),
            URLQueryItem(name: "redirect_uri", value: redirectURI),
            URLQueryItem(name: "response_type", value: "code"),
            URLQueryItem(name: "scope", value: "openid profile email"),
            URLQueryItem(name: "state", value: generateState())
        ]

        guard let url = components.url else { return }

        let session = ASWebAuthenticationSession(
            url: url,
            callbackURLScheme: "myapp"
        ) { [weak self] callbackURL, error in
            if let error = error {
                self?.handleAuthError(error)
                return
            }

            guard let callbackURL = callbackURL else { return }
            self?.handleCallback(callbackURL)
        }

        session.presentationContextProvider = self
        session.prefersEphemeralWebBrowserSession = true
        session.start()
    }

    // MARK: - Callback Processing

    private func handleCallback(_ url: URL) {
        guard let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
              let code = components.queryItems?.first(where: { $0.name == "code" })?.value else {
            return
        }

        Task {
            await exchangeCodeForToken(code)
        }
    }

    // MARK: - Token Exchange

    private func exchangeCodeForToken(_ code: String) async {
        var request = URLRequest(url: tokenURL)
        request.httpMethod = "POST"
        request.setValue("application/x-www-form-urlencoded", forHTTPHeaderField: "Content-Type")

        let body = [
            "grant_type": "authorization_code",
            "code": code,
            "client_id": clientID,
            "redirect_uri": redirectURI
        ]
        request.httpBody = body.percentEncoded()

        do {
            let (data, response) = try await URLSession.shared.data(for: request)

            guard let httpResponse = response as? HTTPURLResponse,
                  (200...299).contains(httpResponse.statusCode) else {
                throw AuthError.invalidResponse
            }

            let tokenResponse = try JSONDecoder().decode(TokenResponse.self, from: data)

            // トークンをKeychainに保存
            try await saveTokens(tokenResponse)

            // ユーザー情報を取得
            await fetchUserInfo()

            await MainActor.run {
                self.isAuthenticated = true
            }
        } catch {
            handleAuthError(error)
        }
    }

    // MARK: - Token Management

    private func saveTokens(_ response: TokenResponse) async throws {
        try KeychainManager.shared.save(
            response.accessToken.data(using: .utf8)!,
            forKey: "accessToken"
        )

        try KeychainManager.shared.save(
            response.refreshToken.data(using: .utf8)!,
            forKey: "refreshToken"
        )

        // トークンの有効期限を保存
        let expiresAt = Date().addingTimeInterval(TimeInterval(response.expiresIn))
        UserDefaults.standard.set(expiresAt, forKey: "tokenExpiresAt")
    }

    // MARK: - User Info

    private func fetchUserInfo() async {
        guard let token = try? KeychainManager.shared.load(forKey: "accessToken"),
              let tokenString = String(data: token, encoding: .utf8) else {
            return
        }

        let userInfoURL = URL(string: "https://auth.example.com/userinfo")!
        var request = URLRequest(url: userInfoURL)
        request.setValue("Bearer \(tokenString)", forHTTPHeaderField: "Authorization")

        do {
            let (data, _) = try await URLSession.shared.data(for: request)
            let user = try JSONDecoder().decode(User.self, from: data)

            await MainActor.run {
                self.user = user
            }
        } catch {
            print("Failed to fetch user info: \(error)")
        }
    }

    // MARK: - Sign Out

    func signOut() {
        try? KeychainManager.shared.delete(forKey: "accessToken")
        try? KeychainManager.shared.delete(forKey: "refreshToken")
        UserDefaults.standard.removeObject(forKey: "tokenExpiresAt")

        isAuthenticated = false
        user = nil
    }

    // MARK: - Helper Methods

    private func generateState() -> String {
        UUID().uuidString
    }

    private func handleAuthError(_ error: Error) {
        print("Authentication error: \(error)")
        // エラーハンドリングの実装
    }
}

// MARK: - ASWebAuthenticationPresentationContextProviding

extension AuthManager: ASWebAuthenticationPresentationContextProviding {
    func presentationAnchor(for session: ASWebAuthenticationSession) -> ASPresentationAnchor {
        ASPresentationAnchor()
    }
}

// MARK: - Models

struct TokenResponse: Codable {
    let accessToken: String
    let refreshToken: String
    let expiresIn: Int
    let tokenType: String

    enum CodingKeys: String, CodingKey {
        case accessToken = "access_token"
        case refreshToken = "refresh_token"
        case expiresIn = "expires_in"
        case tokenType = "token_type"
    }
}

struct User: Codable {
    let id: String
    let email: String
    let name: String
    let picture: String?
}

enum AuthError: Error {
    case invalidResponse
    case tokenExpired
    case invalidToken
}
```

### URL Helper Extension

```swift
// Core/Extensions/Dictionary+URLEncoding.swift
extension Dictionary where Key == String, Value == String {
    func percentEncoded() -> Data? {
        map { key, value in
            let escapedKey = key.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? ""
            let escapedValue = value.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? ""
            return "\(escapedKey)=\(escapedValue)"
        }
        .joined(separator: "&")
        .data(using: .utf8)
    }
}
```

## トークンリフレッシュの実装

アクセストークンの有効期限が切れた場合、リフレッシュトークンを使って新しいトークンを取得します。

```swift
// Core/Authentication/TokenRefreshManager.swift
actor TokenRefreshManager {
    static let shared = TokenRefreshManager()

    private var refreshTask: Task<String, Error>?

    func getValidAccessToken() async throws -> String {
        // 既存のリフレッシュタスクがあれば待機
        if let task = refreshTask {
            return try await task.value
        }

        // 現在のトークンをチェック
        if let tokenData = try? KeychainManager.shared.load(forKey: "accessToken"),
           let token = String(data: tokenData, encoding: .utf8),
           !isTokenExpired() {
            return token
        }

        // 新しいリフレッシュタスクを開始
        let task = Task { () -> String in
            defer { self.refreshTask = nil }
            return try await self.refreshAccessToken()
        }

        refreshTask = task
        return try await task.value
    }

    private func refreshAccessToken() async throws -> String {
        guard let refreshTokenData = try? KeychainManager.shared.load(forKey: "refreshToken"),
              let refreshToken = String(data: refreshTokenData, encoding: .utf8) else {
            throw AuthError.invalidToken
        }

        let url = URL(string: "https://auth.example.com/oauth/token")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/x-www-form-urlencoded", forHTTPHeaderField: "Content-Type")

        let body = [
            "grant_type": "refresh_token",
            "refresh_token": refreshToken,
            "client_id": "your-client-id"
        ]
        request.httpBody = body.percentEncoded()

        let (data, response) = try await URLSession.shared.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw AuthError.invalidResponse
        }

        let tokenResponse = try JSONDecoder().decode(TokenResponse.self, from: data)

        // 新しいトークンを保存
        try KeychainManager.shared.save(
            tokenResponse.accessToken.data(using: .utf8)!,
            forKey: "accessToken"
        )

        let expiresAt = Date().addingTimeInterval(TimeInterval(tokenResponse.expiresIn))
        UserDefaults.standard.set(expiresAt, forKey: "tokenExpiresAt")

        return tokenResponse.accessToken
    }

    private func isTokenExpired() -> Bool {
        guard let expiresAt = UserDefaults.standard.object(forKey: "tokenExpiresAt") as? Date else {
            return true
        }

        return Date() >= expiresAt
    }
}
```

## API リクエストでの認証

全てのAPIリクエストに自動的にトークンを付与します。

```swift
// Core/Networking/AuthenticatedNetworkManager.swift
class AuthenticatedNetworkManager {
    static let shared = AuthenticatedNetworkManager()

    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        // 有効なアクセストークンを取得
        let token = try await TokenRefreshManager.shared.getValidAccessToken()

        var request = try endpoint.makeRequest()
        request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")

        let (data, response) = try await URLSession.shared.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }

        // 401エラー時はトークンが無効
        if httpResponse.statusCode == 401 {
            // 再認証を促す
            NotificationCenter.default.post(name: .unauthorized, object: nil)
            throw NetworkError.unauthorized
        }

        guard (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.httpError(httpResponse.statusCode)
        }

        return try JSONDecoder().decode(T.self, from: data)
    }
}

extension Notification.Name {
    static let unauthorized = Notification.Name("unauthorized")
}

enum NetworkError: Error {
    case invalidResponse
    case unauthorized
    case httpError(Int)
}
```

## SwiftUI での認証フロー

認証状態に応じて画面を切り替えます。

```swift
// App/MyAwesomeAppApp.swift
import SwiftUI

@main
struct MyAwesomeAppApp: App {
    @StateObject private var authManager = AuthManager()

    var body: some Scene {
        WindowGroup {
            Group {
                if authManager.isAuthenticated {
                    HomeView()
                } else {
                    LoginView()
                }
            }
            .environmentObject(authManager)
            .onReceive(NotificationCenter.default.publisher(for: .unauthorized)) { _ in
                authManager.signOut()
            }
        }
    }
}
```

### ログイン画面

```swift
// Features/Authentication/Views/LoginView.swift
import SwiftUI

struct LoginView: View {
    @EnvironmentObject var authManager: AuthManager

    var body: some View {
        VStack(spacing: 20) {
            Image("logo")
                .resizable()
                .scaledToFit()
                .frame(width: 120, height: 120)

            Text("MyAwesomeApp")
                .font(.largeTitle)
                .fontWeight(.bold)

            Text("ログインして始めましょう")
                .font(.body)
                .foregroundColor(.secondary)

            Spacer()

            Button {
                authManager.signIn(presenting: UIViewController())
            } label: {
                HStack {
                    Image(systemName: "person.circle.fill")
                    Text("ログイン")
                }
                .font(.headline)
                .foregroundColor(.white)
                .frame(maxWidth: .infinity)
                .padding()
                .background(Color.blue)
                .cornerRadius(10)
            }
        }
        .padding()
    }
}
```

## まとめ

本章では、OAuth 2.0とOpenID Connectを使った認証・認可の実装方法を解説しました。適切な認証実装により、以下の効果が期待されます：

- ユーザーの安全な認証
- トークンベースのセキュアなAPI通信
- シームレスな認証体験
- セッション管理の自動化

次章では、Keychainを使った安全なデータ保存について解説します。
