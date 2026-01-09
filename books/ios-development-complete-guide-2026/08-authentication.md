---
title: "認証とトークン管理"
---

# 認証とトークン管理

## この章で学ぶこと

- JWT認証の実装とベストプラクティス
- OAuth 2.0フローの完全実装
- セキュアなトークン保存方法
- トークンリフレッシュ戦略
- 生体認証の統合
- セキュリティベストプラクティス

## 認証の基礎

### 認証方式の種類

iOSアプリで一般的に使用される認証方式:

1. **トークンベース認証（JWT）**: 最も一般的。ステートレスで拡張性が高い
2. **OAuth 2.0**: サードパーティ認証（Google、Appleなど）に使用
3. **Basic認証**: シンプルだがセキュリティが低い（非推奨）
4. **APIキー認証**: サーバー間通信に適している

### 認証フロー

典型的な認証フロー:

```
1. ユーザーがログイン情報を入力
   ↓
2. アプリがサーバーに認証情報を送信
   ↓
3. サーバーが認証し、アクセストークンとリフレッシュトークンを返す
   ↓
4. アプリがトークンをKeychainに保存
   ↓
5. 以降のAPIリクエストにアクセストークンを含める
   ↓
6. アクセストークンの期限切れ時、リフレッシュトークンで更新
```

## JWT認証の実装

### JWTトークンの構造

JWTは3つの部分で構成されます:

```
Header.Payload.Signature

例:
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### トークンモデルの定義

```swift
struct AuthToken: Codable {
    let accessToken: String
    let refreshToken: String
    let expiresIn: TimeInterval
    let tokenType: String

    enum CodingKeys: String, CodingKey {
        case accessToken = "access_token"
        case refreshToken = "refresh_token"
        case expiresIn = "expires_in"
        case tokenType = "token_type"
    }
}

struct JWTPayload: Codable {
    let sub: String // Subject (user ID)
    let exp: TimeInterval // Expiration time
    let iat: TimeInterval // Issued at
    let name: String?
    let email: String?
    let roles: [String]?
}

extension AuthToken {
    var expirationDate: Date {
        Date().addingTimeInterval(expiresIn)
    }

    var isExpired: Bool {
        Date() >= expirationDate
    }

    var isExpiringSoon: Bool {
        let threshold: TimeInterval = 300 // 5分前
        return Date().addingTimeInterval(threshold) >= expirationDate
    }
}
```

### JWTのデコード

```swift
enum JWTError: Error {
    case invalidFormat
    case invalidBase64
    case decodingFailed
}

struct JWTDecoder {
    static func decode(_ token: String) throws -> JWTPayload {
        let segments = token.components(separatedBy: ".")
        guard segments.count == 3 else {
            throw JWTError.invalidFormat
        }

        // Payload部分を取得
        var base64String = segments[1]

        // Base64URLをBase64に変換
        base64String = base64String
            .replacingOccurrences(of: "-", with: "+")
            .replacingOccurrences(of: "_", with: "/")

        // パディング追加
        let paddingLength = 4 - base64String.count % 4
        if paddingLength < 4 {
            base64String.append(String(repeating: "=", count: paddingLength))
        }

        // Base64デコード
        guard let data = Data(base64Encoded: base64String) else {
            throw JWTError.invalidBase64
        }

        // JSONデコード
        do {
            return try JSONDecoder().decode(JWTPayload.self, from: data)
        } catch {
            throw JWTError.decodingFailed
        }
    }

    static func isExpired(_ token: String) -> Bool {
        guard let payload = try? decode(token) else {
            return true
        }
        return Date().timeIntervalSince1970 >= payload.exp
    }
}
```

### 認証サービスの実装

```swift
protocol AuthenticationService {
    func login(email: String, password: String) async throws -> AuthToken
    func logout() async throws
    func refreshToken() async throws -> AuthToken
    func isAuthenticated() -> Bool
}

class AuthenticationServiceImpl: AuthenticationService {
    private let apiService: APIService
    private let keychainManager: KeychainManager
    private let tokenSubject = CurrentValueSubject<AuthToken?, Never>(nil)

    var currentToken: AnyPublisher<AuthToken?, Never> {
        tokenSubject.eraseToAnyPublisher()
    }

    init(
        apiService: APIService,
        keychainManager: KeychainManager = .shared
    ) {
        self.apiService = apiService
        self.keychainManager = keychainManager

        // 保存されているトークンを読み込む
        if let token = try? keychainManager.loadAuthToken() {
            tokenSubject.send(token)
        }
    }

    func login(email: String, password: String) async throws -> AuthToken {
        struct LoginRequest: Codable {
            let email: String
            let password: String
        }

        let endpoint = AuthEndpoint.login(
            LoginRequest(email: email, password: password)
        )

        let token: AuthToken = try await apiService.request(endpoint)

        // トークンを保存
        try keychainManager.saveAuthToken(token)

        // 状態を更新
        tokenSubject.send(token)

        return token
    }

    func logout() async throws {
        // サーバーにログアウトを通知（オプション）
        if let token = tokenSubject.value {
            let endpoint = AuthEndpoint.logout(token.accessToken)
            try? await apiService.request(endpoint)
        }

        // トークンを削除
        try keychainManager.deleteAuthToken()

        // 状態をクリア
        tokenSubject.send(nil)
    }

    func refreshToken() async throws -> AuthToken {
        guard let currentToken = tokenSubject.value else {
            throw AuthError.notAuthenticated
        }

        struct RefreshRequest: Codable {
            let refreshToken: String

            enum CodingKeys: String, CodingKey {
                case refreshToken = "refresh_token"
            }
        }

        let endpoint = AuthEndpoint.refresh(
            RefreshRequest(refreshToken: currentToken.refreshToken)
        )

        let newToken: AuthToken = try await apiService.request(endpoint)

        // 新しいトークンを保存
        try keychainManager.saveAuthToken(newToken)

        // 状態を更新
        tokenSubject.send(newToken)

        return newToken
    }

    func isAuthenticated() -> Bool {
        guard let token = tokenSubject.value else {
            return false
        }
        return !token.isExpired
    }
}

enum AuthError: Error, LocalizedError {
    case notAuthenticated
    case invalidCredentials
    case tokenExpired
    case refreshFailed

    var errorDescription: String? {
        switch self {
        case .notAuthenticated:
            return "User is not authenticated"
        case .invalidCredentials:
            return "Invalid email or password"
        case .tokenExpired:
            return "Authentication token has expired"
        case .refreshFailed:
            return "Failed to refresh authentication token"
        }
    }
}
```

### 認証付きAPIService

トークンを自動的に追加し、リフレッシュするAPIService:

```swift
class AuthenticatedAPIService: APIService {
    private let baseService: APIService
    private let authService: AuthenticationService
    private let refreshLock = NSLock()
    private var isRefreshing = false
    private var refreshContinuations: [CheckedContinuation<AuthToken, Error>] = []

    init(
        baseService: APIService,
        authService: AuthenticationService
    ) {
        self.baseService = baseService
        self.authService = authService
    }

    func request<E: Endpoint>(_ endpoint: E) async throws -> E.Response {
        // 認証が必要なエンドポイントかチェック
        guard endpoint.requiresAuthentication else {
            return try await baseService.request(endpoint)
        }

        // トークンを取得
        let token = try await getValidToken()

        // リクエストを作成
        var request = try endpoint.makeRequest()
        request.setValue(
            "\(token.tokenType) \(token.accessToken)",
            forHTTPHeaderField: "Authorization"
        )

        // リクエスト実行
        do {
            return try await executeRequest(request)
        } catch let error as NetworkError {
            // 401エラーの場合、トークンをリフレッシュして再試行
            if case .unauthorized = error {
                let newToken = try await refreshTokenIfNeeded()
                request.setValue(
                    "\(newToken.tokenType) \(newToken.accessToken)",
                    forHTTPHeaderField: "Authorization"
                )
                return try await executeRequest(request)
            }
            throw error
        }
    }

    private func getValidToken() async throws -> AuthToken {
        guard let token = try? keychainManager.loadAuthToken() else {
            throw AuthError.notAuthenticated
        }

        // トークンの期限が近い場合は事前にリフレッシュ
        if token.isExpiringSoon {
            return try await refreshTokenIfNeeded()
        }

        return token
    }

    private func refreshTokenIfNeeded() async throws -> AuthToken {
        // リフレッシュ中の場合は待機
        return try await withCheckedThrowingContinuation { continuation in
            refreshLock.lock()

            if isRefreshing {
                refreshContinuations.append(continuation)
                refreshLock.unlock()
                return
            }

            isRefreshing = true
            refreshLock.unlock()

            Task {
                do {
                    let newToken = try await authService.refreshToken()

                    // 待機中のリクエストを再開
                    refreshLock.lock()
                    let continuations = refreshContinuations
                    refreshContinuations.removeAll()
                    isRefreshing = false
                    refreshLock.unlock()

                    continuation.resume(returning: newToken)
                    continuations.forEach { $0.resume(returning: newToken) }

                } catch {
                    refreshLock.lock()
                    let continuations = refreshContinuations
                    refreshContinuations.removeAll()
                    isRefreshing = false
                    refreshLock.unlock()

                    continuation.resume(throwing: error)
                    continuations.forEach { $0.resume(throwing: error) }
                }
            }
        }
    }

    private func executeRequest<T: Decodable>(_ request: URLRequest) async throws -> T {
        let (data, response) = try await URLSession.shared.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }

        guard (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.from(
                statusCode: httpResponse.statusCode,
                data: data
            )
        }

        return try JSONDecoder().decode(T.self, from: data)
    }
}

extension Endpoint {
    var requiresAuthentication: Bool {
        // デフォルトは認証必要
        true
    }
}
```

## Keychainへのトークン保存

### KeychainManagerの実装

セキュアにトークンを保存するマネージャー:

```swift
enum KeychainError: Error {
    case duplicateItem
    case itemNotFound
    case invalidData
    case unexpectedStatus(OSStatus)
}

class KeychainManager {
    static let shared = KeychainManager()

    private let service: String
    private let accessGroup: String?

    init(
        service: String = Bundle.main.bundleIdentifier ?? "com.example.app",
        accessGroup: String? = nil
    ) {
        self.service = service
        self.accessGroup = accessGroup
    }

    // MARK: - AuthToken

    func saveAuthToken(_ token: AuthToken) throws {
        let data = try JSONEncoder().encode(token)
        try save(data, for: "auth_token")
    }

    func loadAuthToken() throws -> AuthToken {
        let data = try load(for: "auth_token")
        return try JSONDecoder().decode(AuthToken.self, from: data)
    }

    func deleteAuthToken() throws {
        try delete(for: "auth_token")
    }

    // MARK: - Generic Methods

    func save(_ data: Data, for key: String) throws {
        var query = baseQuery(for: key)
        query[kSecValueData as String] = data
        query[kSecAttrAccessible as String] = kSecAttrAccessibleAfterFirstUnlock

        let status = SecItemAdd(query as CFDictionary, nil)

        if status == errSecDuplicateItem {
            // 既存のアイテムを更新
            try update(data, for: key)
        } else if status != errSecSuccess {
            throw KeychainError.unexpectedStatus(status)
        }
    }

    func load(for key: String) throws -> Data {
        var query = baseQuery(for: key)
        query[kSecReturnData as String] = true
        query[kSecMatchLimit as String] = kSecMatchLimitOne

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess else {
            if status == errSecItemNotFound {
                throw KeychainError.itemNotFound
            }
            throw KeychainError.unexpectedStatus(status)
        }

        guard let data = result as? Data else {
            throw KeychainError.invalidData
        }

        return data
    }

    func update(_ data: Data, for key: String) throws {
        let query = baseQuery(for: key)
        let attributes: [String: Any] = [
            kSecValueData as String: data
        ]

        let status = SecItemUpdate(
            query as CFDictionary,
            attributes as CFDictionary
        )

        guard status == errSecSuccess else {
            throw KeychainError.unexpectedStatus(status)
        }
    }

    func delete(for key: String) throws {
        let query = baseQuery(for: key)
        let status = SecItemDelete(query as CFDictionary)

        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.unexpectedStatus(status)
        }
    }

    func deleteAll() throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service
        ]

        let status = SecItemDelete(query as CFDictionary)

        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.unexpectedStatus(status)
        }
    }

    private func baseQuery(for key: String) -> [String: Any] {
        var query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key
        ]

        if let accessGroup = accessGroup {
            query[kSecAttrAccessGroup as String] = accessGroup
        }

        return query
    }
}
```

### 生体認証保護

Face ID/Touch IDでトークンを保護:

```swift
extension KeychainManager {
    func saveAuthTokenWithBiometrics(_ token: AuthToken) throws {
        let data = try JSONEncoder().encode(token)

        var query = baseQuery(for: "auth_token_biometric")
        query[kSecValueData as String] = data

        // 生体認証を要求
        let accessControl = SecAccessControlCreateWithFlags(
            nil,
            kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
            .biometryCurrentSet,
            nil
        )
        query[kSecAttrAccessControl as String] = accessControl

        let status = SecItemAdd(query as CFDictionary, nil)

        if status == errSecDuplicateItem {
            try updateWithBiometrics(data)
        } else if status != errSecSuccess {
            throw KeychainError.unexpectedStatus(status)
        }
    }

    func loadAuthTokenWithBiometrics(
        reason: String = "Authenticate to access your account"
    ) async throws -> AuthToken {
        let context = LAContext()
        context.localizedReason = reason

        var query = baseQuery(for: "auth_token_biometric")
        query[kSecReturnData as String] = true
        query[kSecMatchLimit as String] = kSecMatchLimitOne
        query[kSecUseAuthenticationContext as String] = context

        return try await withCheckedThrowingContinuation { continuation in
            DispatchQueue.global().async {
                var result: AnyObject?
                let status = SecItemCopyMatching(query as CFDictionary, &result)

                if status == errSecSuccess, let data = result as? Data {
                    do {
                        let token = try JSONDecoder().decode(AuthToken.self, from: data)
                        continuation.resume(returning: token)
                    } catch {
                        continuation.resume(throwing: KeychainError.invalidData)
                    }
                } else if status == errSecItemNotFound {
                    continuation.resume(throwing: KeychainError.itemNotFound)
                } else {
                    continuation.resume(throwing: KeychainError.unexpectedStatus(status))
                }
            }
        }
    }

    private func updateWithBiometrics(_ data: Data) throws {
        let query = baseQuery(for: "auth_token_biometric")

        let accessControl = SecAccessControlCreateWithFlags(
            nil,
            kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
            .biometryCurrentSet,
            nil
        )

        let attributes: [String: Any] = [
            kSecValueData as String: data,
            kSecAttrAccessControl as String: accessControl as Any
        ]

        let status = SecItemUpdate(
            query as CFDictionary,
            attributes as CFDictionary
        )

        guard status == errSecSuccess else {
            throw KeychainError.unexpectedStatus(status)
        }
    }
}
```

## OAuth 2.0の実装

### OAuth 2.0フロー

Authorization Code Flowの実装:

```swift
enum OAuthError: Error {
    case invalidURL
    case authorizationFailed
    case tokenExchangeFailed
    case invalidState
    case userCancelled
}

class OAuthManager {
    private let clientId: String
    private let clientSecret: String
    private let redirectURI: String
    private let authorizationEndpoint: URL
    private let tokenEndpoint: URL
    private let scope: String

    init(
        clientId: String,
        clientSecret: String,
        redirectURI: String,
        authorizationEndpoint: URL,
        tokenEndpoint: URL,
        scope: String
    ) {
        self.clientId = clientId
        self.clientSecret = clientSecret
        self.redirectURI = redirectURI
        self.authorizationEndpoint = authorizationEndpoint
        self.tokenEndpoint = tokenEndpoint
        self.scope = scope
    }

    // MARK: - Authorization Code Flow

    func authorize(presentingViewController: UIViewController) async throws -> AuthToken {
        // 1. 認証URLを生成
        let state = generateState()
        let codeVerifier = generateCodeVerifier()
        let codeChallenge = generateCodeChallenge(from: codeVerifier)

        let authURL = try buildAuthorizationURL(
            state: state,
            codeChallenge: codeChallenge
        )

        // 2. ユーザーに認証を求める
        let authorizationCode = try await presentAuthorizationUI(
            url: authURL,
            expectedState: state,
            presentingViewController: presentingViewController
        )

        // 3. 認証コードをトークンと交換
        return try await exchangeCodeForToken(
            code: authorizationCode,
            codeVerifier: codeVerifier
        )
    }

    private func buildAuthorizationURL(
        state: String,
        codeChallenge: String
    ) throws -> URL {
        var components = URLComponents(
            url: authorizationEndpoint,
            resolvingAgainstBaseURL: false
        )

        components?.queryItems = [
            URLQueryItem(name: "response_type", value: "code"),
            URLQueryItem(name: "client_id", value: clientId),
            URLQueryItem(name: "redirect_uri", value: redirectURI),
            URLQueryItem(name: "scope", value: scope),
            URLQueryItem(name: "state", value: state),
            URLQueryItem(name: "code_challenge", value: codeChallenge),
            URLQueryItem(name: "code_challenge_method", value: "S256")
        ]

        guard let url = components?.url else {
            throw OAuthError.invalidURL
        }

        return url
    }

    private func presentAuthorizationUI(
        url: URL,
        expectedState: String,
        presentingViewController: UIViewController
    ) async throws -> String {
        try await withCheckedThrowingContinuation { continuation in
            let session = ASWebAuthenticationSession(
                url: url,
                callbackURLScheme: extractScheme(from: redirectURI)
            ) { callbackURL, error in
                if let error = error {
                    if case ASWebAuthenticationSessionError.canceledLogin = error {
                        continuation.resume(throwing: OAuthError.userCancelled)
                    } else {
                        continuation.resume(throwing: OAuthError.authorizationFailed)
                    }
                    return
                }

                guard let callbackURL = callbackURL else {
                    continuation.resume(throwing: OAuthError.authorizationFailed)
                    return
                }

                do {
                    let code = try self.extractAuthorizationCode(
                        from: callbackURL,
                        expectedState: expectedState
                    )
                    continuation.resume(returning: code)
                } catch {
                    continuation.resume(throwing: error)
                }
            }

            session.presentationContextProvider = self
            session.prefersEphemeralWebBrowserSession = true
            session.start()
        }
    }

    private func exchangeCodeForToken(
        code: String,
        codeVerifier: String
    ) async throws -> AuthToken {
        var request = URLRequest(url: tokenEndpoint)
        request.httpMethod = "POST"
        request.setValue("application/x-www-form-urlencoded", forHTTPHeaderField: "Content-Type")

        let parameters = [
            "grant_type": "authorization_code",
            "code": code,
            "redirect_uri": redirectURI,
            "client_id": clientId,
            "client_secret": clientSecret,
            "code_verifier": codeVerifier
        ]

        request.httpBody = parameters
            .map { "\($0.key)=\($0.value.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? "")" }
            .joined(separator: "&")
            .data(using: .utf8)

        let (data, response) = try await URLSession.shared.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw OAuthError.tokenExchangeFailed
        }

        return try JSONDecoder().decode(AuthToken.self, from: data)
    }

    // MARK: - PKCE Helpers

    private func generateState() -> String {
        let bytes = (0..<32).map { _ in UInt8.random(in: 0...255) }
        return Data(bytes).base64EncodedString()
            .replacingOccurrences(of: "+", with: "-")
            .replacingOccurrences(of: "/", with: "_")
            .replacingOccurrences(of: "=", with: "")
    }

    private func generateCodeVerifier() -> String {
        let bytes = (0..<32).map { _ in UInt8.random(in: 0...255) }
        return Data(bytes).base64EncodedString()
            .replacingOccurrences(of: "+", with: "-")
            .replacingOccurrences(of: "/", with: "_")
            .replacingOccurrences(of: "=", with: "")
    }

    private func generateCodeChallenge(from verifier: String) -> String {
        guard let data = verifier.data(using: .utf8) else { return "" }
        let hash = SHA256.hash(data: data)
        return Data(hash).base64EncodedString()
            .replacingOccurrences(of: "+", with: "-")
            .replacingOccurrences(of: "/", with: "_")
            .replacingOccurrences(of: "=", with: "")
    }

    private func extractScheme(from urlString: String) -> String? {
        guard let url = URL(string: urlString) else { return nil }
        return url.scheme
    }

    private func extractAuthorizationCode(
        from url: URL,
        expectedState: String
    ) throws -> String {
        guard let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
              let queryItems = components.queryItems else {
            throw OAuthError.authorizationFailed
        }

        // stateを検証
        let state = queryItems.first(where: { $0.name == "state" })?.value
        guard state == expectedState else {
            throw OAuthError.invalidState
        }

        // 認証コードを取得
        guard let code = queryItems.first(where: { $0.name == "code" })?.value else {
            throw OAuthError.authorizationFailed
        }

        return code
    }
}

extension OAuthManager: ASWebAuthenticationPresentationContextProviding {
    func presentationAnchor(for session: ASWebAuthenticationSession) -> ASPresentationAnchor {
        UIApplication.shared.windows.first { $0.isKeyWindow } ?? ASPresentationAnchor()
    }
}
```

### Sign in with Apple

Sign in with Appleの実装:

```swift
import AuthenticationServices

class AppleAuthManager: NSObject {
    private var continuation: CheckedContinuation<AuthToken, Error>?

    func signInWithApple() async throws -> AuthToken {
        try await withCheckedThrowingContinuation { continuation in
            self.continuation = continuation

            let request = ASAuthorizationAppleIDProvider().createRequest()
            request.requestedScopes = [.fullName, .email]

            let controller = ASAuthorizationController(authorizationRequests: [request])
            controller.delegate = self
            controller.presentationContextProvider = self
            controller.performRequests()
        }
    }
}

extension AppleAuthManager: ASAuthorizationControllerDelegate {
    func authorizationController(
        controller: ASAuthorizationController,
        didCompleteWithAuthorization authorization: ASAuthorization
    ) {
        guard let credential = authorization.credential as? ASAuthorizationAppleIDCredential,
              let identityToken = credential.identityToken,
              let tokenString = String(data: identityToken, encoding: .utf8) else {
            continuation?.resume(throwing: OAuthError.authorizationFailed)
            return
        }

        // サーバーにトークンを送信して認証
        Task {
            do {
                let token = try await exchangeAppleTokenForAuthToken(
                    identityToken: tokenString,
                    authorizationCode: credential.authorizationCode.flatMap {
                        String(data: $0, encoding: .utf8)
                    },
                    user: credential.user
                )
                continuation?.resume(returning: token)
            } catch {
                continuation?.resume(throwing: error)
            }
        }
    }

    func authorizationController(
        controller: ASAuthorizationController,
        didCompleteWithError error: Error
    ) {
        continuation?.resume(throwing: error)
    }

    private func exchangeAppleTokenForAuthToken(
        identityToken: String,
        authorizationCode: String?,
        user: String
    ) async throws -> AuthToken {
        struct AppleAuthRequest: Codable {
            let identityToken: String
            let authorizationCode: String?
            let user: String
        }

        let endpoint = AuthEndpoint.appleAuth(
            AppleAuthRequest(
                identityToken: identityToken,
                authorizationCode: authorizationCode,
                user: user
            )
        )

        return try await APIServiceImpl().request(endpoint)
    }
}

extension AppleAuthManager: ASAuthorizationControllerPresentationContextProviding {
    func presentationAnchor(for controller: ASAuthorizationController) -> ASPresentationAnchor {
        UIApplication.shared.windows.first { $0.isKeyWindow } ?? ASPresentationAnchor()
    }
}
```

## 生体認証の統合

### Local Authenticationの実装

Face ID/Touch IDを使用した認証:

```swift
import LocalAuthentication

enum BiometricError: Error {
    case notAvailable
    case notEnrolled
    case authenticationFailed
    case userCancel
    case userFallback
    case biometryLockout
}

class BiometricAuthManager {
    enum BiometricType {
        case none
        case touchID
        case faceID

        var description: String {
            switch self {
            case .none:
                return "None"
            case .touchID:
                return "Touch ID"
            case .faceID:
                return "Face ID"
            }
        }
    }

    static let shared = BiometricAuthManager()

    private init() {}

    func availableBiometricType() -> BiometricType {
        let context = LAContext()
        var error: NSError?

        guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) else {
            return .none
        }

        switch context.biometryType {
        case .none:
            return .none
        case .touchID:
            return .touchID
        case .faceID:
            return .faceID
        @unknown default:
            return .none
        }
    }

    func authenticateWithBiometrics(
        reason: String = "Authenticate to continue"
    ) async throws {
        let context = LAContext()
        context.localizedCancelTitle = "Cancel"
        context.localizedFallbackTitle = "Use Passcode"

        var error: NSError?
        guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) else {
            if let error = error {
                throw mapBiometricError(error)
            }
            throw BiometricError.notAvailable
        }

        do {
            let success = try await context.evaluatePolicy(
                .deviceOwnerAuthenticationWithBiometrics,
                localizedReason: reason
            )

            guard success else {
                throw BiometricError.authenticationFailed
            }
        } catch let error as LAError {
            throw mapLAError(error)
        }
    }

    func authenticateWithBiometricsOrPasscode(
        reason: String = "Authenticate to continue"
    ) async throws {
        let context = LAContext()
        context.localizedCancelTitle = "Cancel"

        var error: NSError?
        guard context.canEvaluatePolicy(.deviceOwnerAuthentication, error: &error) else {
            if let error = error {
                throw mapBiometricError(error)
            }
            throw BiometricError.notAvailable
        }

        do {
            let success = try await context.evaluatePolicy(
                .deviceOwnerAuthentication,
                localizedReason: reason
            )

            guard success else {
                throw BiometricError.authenticationFailed
            }
        } catch let error as LAError {
            throw mapLAError(error)
        }
    }

    private func mapBiometricError(_ error: NSError) -> BiometricError {
        guard let laError = error as? LAError else {
            return .authenticationFailed
        }
        return mapLAError(laError)
    }

    private func mapLAError(_ error: LAError) -> BiometricError {
        switch error.code {
        case .biometryNotAvailable:
            return .notAvailable
        case .biometryNotEnrolled:
            return .notEnrolled
        case .userCancel:
            return .userCancel
        case .userFallback:
            return .userFallback
        case .biometryLockout:
            return .biometryLockout
        default:
            return .authenticationFailed
        }
    }
}
```

### 生体認証統合ログイン

```swift
@MainActor
class BiometricLoginViewModel: ObservableObject {
    @Published var isBiometricEnabled = false
    @Published var biometricType: BiometricAuthManager.BiometricType = .none
    @Published var isAuthenticating = false
    @Published var error: Error?

    private let authService: AuthenticationService
    private let biometricManager: BiometricAuthManager
    private let keychainManager: KeychainManager

    init(
        authService: AuthenticationService,
        biometricManager: BiometricAuthManager = .shared,
        keychainManager: KeychainManager = .shared
    ) {
        self.authService = authService
        self.biometricManager = biometricManager
        self.keychainManager = keychainManager

        self.biometricType = biometricManager.availableBiometricType()
        self.isBiometricEnabled = UserDefaults.standard.bool(forKey: "biometric_login_enabled")
    }

    func enableBiometricLogin() async {
        do {
            // 生体認証をテスト
            try await biometricManager.authenticateWithBiometrics(
                reason: "Enable \(biometricType.description) for quick login"
            )

            // 設定を保存
            UserDefaults.standard.set(true, forKey: "biometric_login_enabled")
            isBiometricEnabled = true

        } catch {
            self.error = error
        }
    }

    func disableBiometricLogin() {
        UserDefaults.standard.set(false, forKey: "biometric_login_enabled")
        isBiometricEnabled = false

        // 生体認証で保護されたトークンを削除
        try? keychainManager.delete(for: "auth_token_biometric")
    }

    func authenticateWithBiometrics() async {
        guard isBiometricEnabled else { return }

        isAuthenticating = true
        error = nil

        do {
            // 生体認証
            try await biometricManager.authenticateWithBiometrics(
                reason: "Sign in to your account"
            )

            // トークンを取得
            let token = try await keychainManager.loadAuthTokenWithBiometrics()

            // トークンが有効か確認
            if token.isExpired {
                // 期限切れの場合はリフレッシュ
                _ = try await authService.refreshToken()
            }

        } catch {
            self.error = error
        }

        isAuthenticating = false
    }
}
```

## セキュリティベストプラクティス

### セキュアな通信

```swift
class SecureNetworkManager {
    static let shared = SecureNetworkManager()

    private let session: URLSession

    private init() {
        let configuration = URLSessionConfiguration.ephemeral
        configuration.tlsMinimumSupportedProtocolVersion = .TLSv13
        configuration.timeoutIntervalForRequest = 30
        configuration.waitsForConnectivity = true

        // Certificate pinningを設定
        self.session = URLSession(
            configuration: configuration,
            delegate: CertificatePinningDelegate(),
            delegateQueue: nil
        )
    }
}

class CertificatePinningDelegate: NSObject, URLSessionDelegate {
    private let pinnedCertificates: Set<Data>

    override init() {
        // アプリにバンドルされた証明書を読み込む
        var certificates: Set<Data> = []

        if let certPath = Bundle.main.path(forResource: "certificate", ofType: "cer"),
           let certData = try? Data(contentsOf: URL(fileURLWithPath: certPath)) {
            certificates.insert(certData)
        }

        self.pinnedCertificates = certificates
        super.init()
    }

    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        guard challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
              let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        // 証明書チェーンを検証
        var secResult = SecTrustResultType.invalid
        let status = SecTrustEvaluate(serverTrust, &secResult)

        guard status == errSecSuccess else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        // ピン留めされた証明書と比較
        guard let serverCertificate = SecTrustGetCertificateAtIndex(serverTrust, 0) else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        let serverCertificateData = SecCertificateCopyData(serverCertificate) as Data

        if pinnedCertificates.contains(serverCertificateData) {
            let credential = URLCredential(trust: serverTrust)
            completionHandler(.useCredential, credential)
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }
}
```

### トークンの安全な処理

```swift
class SecureTokenManager {
    private let keychainManager: KeychainManager
    private let encryptionKey: SymmetricKey

    init(keychainManager: KeychainManager = .shared) {
        self.keychainManager = keychainManager

        // 暗号化キーを生成または読み込み
        if let keyData = try? keychainManager.load(for: "encryption_key") {
            self.encryptionKey = SymmetricKey(data: keyData)
        } else {
            let key = SymmetricKey(size: .bits256)
            try? keychainManager.save(key.withUnsafeBytes { Data($0) }, for: "encryption_key")
            self.encryptionKey = key
        }
    }

    func saveToken(_ token: AuthToken) throws {
        // トークンをシリアライズ
        let data = try JSONEncoder().encode(token)

        // 暗号化
        let sealedBox = try AES.GCM.seal(data, using: encryptionKey)

        guard let encryptedData = sealedBox.combined else {
            throw EncryptionError.encryptionFailed
        }

        // Keychainに保存
        try keychainManager.save(encryptedData, for: "encrypted_auth_token")
    }

    func loadToken() throws -> AuthToken {
        // Keychainから読み込み
        let encryptedData = try keychainManager.load(for: "encrypted_auth_token")

        // 復号化
        let sealedBox = try AES.GCM.SealedBox(combined: encryptedData)
        let decryptedData = try AES.GCM.open(sealedBox, using: encryptionKey)

        // デシリアライズ
        return try JSONDecoder().decode(AuthToken.self, from: decryptedData)
    }

    func deleteToken() throws {
        try keychainManager.delete(for: "encrypted_auth_token")
    }
}

enum EncryptionError: Error {
    case encryptionFailed
    case decryptionFailed
}
```

### セキュリティチェックリスト

認証実装時に確認すべきポイント:

```swift
struct SecurityAudit {
    static func performSecurityCheck() -> [SecurityIssue] {
        var issues: [SecurityIssue] = []

        // 1. JailbreakCheck
        if isJailbroken() {
            issues.append(.jailbreakDetected)
        }

        // 2. Debugger Check
        if isDebuggerAttached() {
            issues.append(.debuggerAttached)
        }

        // 3. SSL Pinning
        if !isSSLPinningEnabled() {
            issues.append(.sslPinningDisabled)
        }

        // 4. Keychain Protection
        if !isKeychainProperlyProtected() {
            issues.append(.weakKeychainProtection)
        }

        return issues
    }

    private static func isJailbroken() -> Bool {
        #if targetEnvironment(simulator)
        return false
        #else
        // Jailbreak検出ロジック
        let jailbreakPaths = [
            "/Applications/Cydia.app",
            "/Library/MobileSubstrate/MobileSubstrate.dylib",
            "/bin/bash",
            "/usr/sbin/sshd",
            "/etc/apt"
        ]

        for path in jailbreakPaths {
            if FileManager.default.fileExists(atPath: path) {
                return true
            }
        }

        // 書き込みテスト
        let testPath = "/private/test_jailbreak.txt"
        do {
            try "test".write(toFile: testPath, atomically: true, encoding: .utf8)
            try FileManager.default.removeItem(atPath: testPath)
            return true
        } catch {
            return false
        }
        #endif
    }

    private static func isDebuggerAttached() -> Bool {
        var info = kinfo_proc()
        var mib: [Int32] = [CTL_KERN, KERN_PROC, KERN_PROC_PID, getpid()]
        var size = MemoryLayout<kinfo_proc>.stride

        let result = sysctl(&mib, UInt32(mib.count), &info, &size, nil, 0)

        return (result == 0) && ((info.kp_proc.p_flag & P_TRACED) != 0)
    }

    private static func isSSLPinningEnabled() -> Bool {
        // SSL Pinningが有効かチェック
        // 実装に応じて調整
        return true
    }

    private static func isKeychainProperlyProtected() -> Bool {
        // Keychain保護レベルをチェック
        return true
    }
}

enum SecurityIssue {
    case jailbreakDetected
    case debuggerAttached
    case sslPinningDisabled
    case weakKeychainProtection

    var description: String {
        switch self {
        case .jailbreakDetected:
            return "Device appears to be jailbroken"
        case .debuggerAttached:
            return "Debugger is attached"
        case .sslPinningDisabled:
            return "SSL pinning is not enabled"
        case .weakKeychainProtection:
            return "Keychain protection is weak"
        }
    }

    var severity: Severity {
        switch self {
        case .jailbreakDetected, .debuggerAttached:
            return .critical
        case .sslPinningDisabled:
            return .high
        case .weakKeychainProtection:
            return .medium
        }
    }

    enum Severity {
        case critical
        case high
        case medium
        case low
    }
}
```

## まとめ

この章では、iOSアプリにおける認証とトークン管理の実装方法を学びました。

### 重要なポイント

1. **JWT認証**: トークンの検証とデコード
2. **OAuth 2.0**: Authorization Code FlowとPKCE
3. **Keychain**: セキュアなトークン保存
4. **生体認証**: Face ID/Touch IDの統合
5. **セキュリティ**: Certificate pinningと脅威検出

### 次のステップ

次章では、WebSocketを使ったリアルタイム通信について学びます。接続管理、再接続戦略、メッセージングパターンなどを解説します。

### 参考リソース

- [Authentication Services - Apple Developer](https://developer.apple.com/documentation/authenticationservices)
- [Keychain Services - Apple Developer](https://developer.apple.com/documentation/security/keychain_services)
- [OAuth 2.0 - RFC 6749](https://tools.ietf.org/html/rfc6749)
- [PKCE - RFC 7636](https://tools.ietf.org/html/rfc7636)
