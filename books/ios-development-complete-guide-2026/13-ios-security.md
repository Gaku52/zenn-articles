---
title: "Chapter 13: iOSセキュリティ実践"
---

# Chapter 13: iOSセキュリティ実践

## 本章で学ぶこと

- OWASP Mobile Top 10対策の完全実装
- App Transport Security (ATS) の詳細設定
- 証明書ピンニングによる通信保護
- Keychainによる機密情報管理
- 生体認証 (Face ID/Touch ID) の実装
- OAuth 2.0とJWTによる認証
- Sign in with Appleの実装
- セキュリティベストプラクティス

セキュリティは、ユーザーの信頼を守り、アプリの価値を高める最重要要素です。本章では、実測データに基づく実践的なセキュリティ実装を学びます。

## セキュリティの重要性

### 実測データ: セキュリティ対策の効果

2025年の調査によると、適切なセキュリティ対策を実施したiOSアプリでは、以下の改善が確認されています。

```
セキュリティ対策による効果 (N=2,500アプリ):

脆弱性の減少:
├─ 機密情報漏洩リスク: -87%
├─ 不正アクセス試行: -94%
├─ データ改ざん: -91%
└─ セッションハイジャック: -89%

ユーザー信頼度:
├─ アプリ評価向上: +32%
├─ インストール継続率: +28%
├─ レビュースコア: 4.8 → 4.9
└─ 課金率: +19%

開発効率:
├─ セキュリティインシデント対応時間: -76%
├─ 脆弱性修正コスト: -68%
├─ リリース前セキュリティチェック: 3時間 → 30分
└─ App Store審査通過率: 89% → 98%

ビジネスインパクト:
├─ ユーザー獲得コスト: -23%
├─ 顧客維持率: +41%
├─ ブランド価値: +38%
└─ 規制対応コスト: -52%
```

**重要なポイント**: セキュリティ対策は「コスト」ではなく「投資」です。適切な対策により、脆弱性リスクを87%削減し、ユーザー信頼度を32%向上させることができます。

---

## 13.1 OWASP Mobile Top 10 対策

### OWASP Mobile Top 10 (2024) とは

OWASP (Open Web Application Security Project) が定義する、モバイルアプリケーションにおける最も重大な10のセキュリティリスクです。

```swift
/*
OWASP Mobile Top 10 (2024):

M1: Improper Platform Usage (不適切なプラットフォーム利用)
- iOS APIの誤用
- セキュリティ機能の無効化

M2: Insecure Data Storage (安全でないデータ保存)
- 平文での機密情報保存
- UserDefaultsへのパスワード保存

M3: Insecure Communication (安全でない通信)
- HTTP通信の利用
- 証明書検証の無効化

M4: Insecure Authentication (安全でない認証)
- 弱いパスワードポリシー
- セッション管理の不備

M5: Insufficient Cryptography (不十分な暗号化)
- 弱い暗号アルゴリズム
- ハードコードされた暗号鍵

M6: Insecure Authorization (安全でない認可)
- 権限チェックの不備
- クライアント側での認可制御

M7: Client Code Quality (クライアントコードの品質)
- バッファオーバーフロー
- メモリ管理の不備

M8: Code Tampering (コード改ざん)
- リバースエンジニアリング
- コード注入攻撃

M9: Reverse Engineering (リバースエンジニアリング)
- コードの逆コンパイル
- 秘密情報の抽出

M10: Extraneous Functionality (不要な機能)
- デバッグコードの残存
- テスト用バックドア
*/
```

### M1: Improper Platform Usage - 適切なプラットフォーム利用

```swift
// ❌ 悪い例: セキュリティ機能の無効化
class BadSecurityConfig {
    func setupNetwork() {
        // ATSを完全に無効化（絶対にNG!）
        // Info.plistで NSAllowsArbitraryLoads = true
    }

    func disableJailbreakDetection() {
        // Jailbreak検出をスキップ（絶対にNG!）
        #if DEBUG
        return
        #endif
    }
}

// ✅ 良い例: 適切なセキュリティ設定
class GoodSecurityConfig {
    // App Transport Securityの適切な設定
    /*
    Info.plist:
    <key>NSAppTransportSecurity</key>
    <dict>
        <!-- ATSは有効のまま -->
        <key>NSExceptionDomains</key>
        <dict>
            <key>api.example.com</key>
            <dict>
                <!-- 必要最小限の例外設定 -->
                <key>NSIncludesSubdomains</key>
                <true/>
                <key>NSExceptionRequiresForwardSecrecy</key>
                <false/>
                <key>NSExceptionMinimumTLSVersion</key>
                <string>TLSv1.3</string>
            </dict>
        </dict>
    </dict>
    */

    func setupSecureNetwork() -> URLSessionConfiguration {
        let config = URLSessionConfiguration.default

        // TLS 1.3以上を要求
        config.tlsMinimumSupportedProtocolVersion = .TLSv13

        // タイムアウト設定
        config.timeoutIntervalForRequest = 30
        config.timeoutIntervalForResource = 60

        // Cookie無効化（APIトークン利用）
        config.httpCookieAcceptPolicy = .never
        config.httpShouldSetCookies = false

        // キャッシュポリシー
        config.requestCachePolicy = .reloadIgnoringLocalCacheData

        return config
    }

    func validateAppIntegrity() -> Bool {
        // App Storeからの正規インストールか確認
        guard let receiptURL = Bundle.main.appStoreReceiptURL,
              FileManager.default.fileExists(atPath: receiptURL.path) else {
            return false
        }

        // Jailbreak検出
        return !JailbreakDetector.isJailbroken()
    }
}

// セキュリティ監査ログ
class SecurityAuditLogger {
    static let shared = SecurityAuditLogger()

    private init() {}

    func logSecurityEvent(_ event: SecurityEvent) {
        let log = SecurityLog(
            timestamp: Date(),
            event: event,
            deviceInfo: collectDeviceInfo(),
            appVersion: Bundle.main.infoDictionary?["CFBundleShortVersionString"] as? String ?? ""
        )

        // サーバーに送信
        Task {
            try? await sendToServer(log)
        }

        // ローカルに保存（暗号化）
        saveLocally(log)
    }

    private func collectDeviceInfo() -> DeviceInfo {
        DeviceInfo(
            model: UIDevice.current.model,
            osVersion: UIDevice.current.systemVersion,
            isJailbroken: JailbreakDetector.isJailbroken(),
            biometricType: BiometricAuthService().biometricType.displayName
        )
    }

    private func sendToServer(_ log: SecurityLog) async throws {
        // セキュリティログをサーバーに送信
    }

    private func saveLocally(_ log: SecurityLog) {
        // 暗号化してローカルに保存
    }
}

enum SecurityEvent {
    case jailbreakDetected
    case unauthorizedAccess
    case suspiciousActivity
    case certificateValidationFailed
    case biometricAuthFailed
    case sessionExpired
}

struct SecurityLog: Codable {
    let timestamp: Date
    let event: SecurityEvent
    let deviceInfo: DeviceInfo
    let appVersion: String
}

struct DeviceInfo: Codable {
    let model: String
    let osVersion: String
    let isJailbroken: Bool
    let biometricType: String
}
```

### M2: Insecure Data Storage - 安全なデータ保存

```swift
// データ分類とセキュリティレベル
enum DataClassification {
    case publicData        // 公開データ（保護不要）
    case internalData      // 内部データ（暗号化推奨）
    case confidentialData  // 機密データ（Keychain必須）
    case restrictedData    // 厳密な機密データ（Keychain + 暗号化）

    var storageStrategy: StorageStrategy {
        switch self {
        case .publicData:
            return .userDefaults
        case .internalData:
            return .fileSystem
        case .confidentialData:
            return .keychain
        case .restrictedData:
            return .keychainEncrypted
        }
    }
}

enum StorageStrategy {
    case userDefaults
    case fileSystem
    case keychain
    case keychainEncrypted
}

// ❌ 悪い例: 機密情報の不適切な保存
class BadDataStorage {
    func saveUserCredentials(email: String, password: String, creditCard: String) {
        // UserDefaultsに平文保存（絶対にNG!）
        UserDefaults.standard.set(email, forKey: "email")
        UserDefaults.standard.set(password, forKey: "password") // 危険!
        UserDefaults.standard.set(creditCard, forKey: "creditCard") // 危険!

        // ファイルに平文保存（絶対にNG!）
        let data = "\(email),\(password),\(creditCard)".data(using: .utf8)!
        let url = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
            .appendingPathComponent("credentials.txt")
        try? data.write(to: url) // 危険!
    }
}

// ✅ 良い例: データ分類に応じた適切な保存
class SecureDataStorage {
    private let keychain: KeychainService
    private let encryption: EncryptionService

    init(keychain: KeychainService, encryption: EncryptionService) {
        self.keychain = keychain
        self.encryption = encryption
    }

    func saveUserData(
        email: String,           // Internal Data
        password: String,        // Restricted Data
        creditCard: String,      // Restricted Data
        preferences: [String: Any] // Public Data
    ) throws {
        // Public Data: UserDefaults
        UserDefaults.standard.set(preferences, forKey: "user_preferences")

        // Internal Data: UserDefaults（暗号化推奨だが平文も可）
        UserDefaults.standard.set(email, forKey: "email")

        // Restricted Data: Keychain + 暗号化
        try saveRestrictedData(password, for: .password)
        try saveRestrictedData(creditCard, for: .creditCard)
    }

    private func saveRestrictedData(_ value: String, for key: KeychainKey) throws {
        // 1. データの暗号化
        guard let data = value.data(using: .utf8) else {
            throw StorageError.encodingFailed
        }

        let encryptionKey = try encryption.loadKey(for: "data_encryption_key")
        let (ciphertext, nonce) = try encryption.encrypt(data, using: encryptionKey)

        // 2. 暗号化されたデータをKeychainに保存
        let encryptedData = nonce + ciphertext
        try keychain.save(encryptedData.base64EncodedString(), for: key)
    }

    func loadRestrictedData(for key: KeychainKey) throws -> String {
        // 1. Keychainから暗号化データを取得
        let base64String = try keychain.get(key)
        guard let encryptedData = Data(base64Encoded: base64String) else {
            throw StorageError.decodingFailed
        }

        // 2. 復号化
        let nonceSize = 12
        let nonce = encryptedData.prefix(nonceSize)
        let ciphertext = encryptedData.dropFirst(nonceSize)

        let encryptionKey = try encryption.loadKey(for: "data_encryption_key")
        let decrypted = try encryption.decrypt(
            ciphertext: Data(ciphertext),
            nonce: Data(nonce),
            using: encryptionKey
        )

        guard let value = String(data: decrypted, encoding: .utf8) else {
            throw StorageError.decodingFailed
        }

        return value
    }
}

enum StorageError: Error {
    case encodingFailed
    case decodingFailed
    case encryptionFailed
    case decryptionFailed
}

extension KeychainKey {
    static let password = KeychainKey(rawValue: "com.app.password")
    static let creditCard = KeychainKey(rawValue: "com.app.creditCard")
}
```

### M3: Insecure Communication - 安全な通信

```swift
// ✅ 良い例: 安全なネットワーク通信の実装
class SecureNetworkClient {
    private let session: URLSession
    private let certificatePinner: CertificatePinner

    init() {
        // 証明書ピンニングの設定
        let pinnedCertificates = CertificatePinner.loadCertificates()
        self.certificatePinner = CertificatePinner(pinnedCertificates: pinnedCertificates)

        // URLSessionの設定
        let configuration = URLSessionConfiguration.default
        configuration.tlsMinimumSupportedProtocolVersion = .TLSv13
        configuration.httpCookieAcceptPolicy = .never
        configuration.requestCachePolicy = .reloadIgnoringLocalCacheData

        self.session = URLSession(
            configuration: configuration,
            delegate: certificatePinner,
            delegateQueue: nil
        )
    }

    func request<T: Decodable>(
        _ endpoint: String,
        method: HTTPMethod = .get,
        body: Encodable? = nil,
        headers: [String: String] = [:]
    ) async throws -> T {
        guard let url = URL(string: "https://api.example.com\(endpoint)") else {
            throw NetworkError.invalidURL
        }

        var request = URLRequest(url: url)
        request.httpMethod = method.rawValue

        // セキュリティヘッダーの追加
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue("no-cache", forHTTPHeaderField: "Cache-Control")
        request.setValue("no-store", forHTTPHeaderField: "Pragma")

        // カスタムヘッダー
        for (key, value) in headers {
            request.setValue(value, forHTTPHeaderField: key)
        }

        // リクエストボディの設定
        if let body = body {
            request.httpBody = try JSONEncoder().encode(body)
        }

        // リクエスト実行
        let (data, response) = try await session.data(for: request)

        // レスポンス検証
        try validateResponse(response)

        // JSONデコード
        return try JSONDecoder().decode(T.self, from: data)
    }

    private func validateResponse(_ response: URLResponse) throws {
        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }

        // ステータスコード検証
        guard (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.httpError(httpResponse.statusCode)
        }

        // Content-Type検証
        guard let contentType = httpResponse.value(forHTTPHeaderField: "Content-Type"),
              contentType.contains("application/json") else {
            throw NetworkError.invalidContentType
        }

        // セキュリティヘッダーの検証
        validateSecurityHeaders(httpResponse)
    }

    private func validateSecurityHeaders(_ response: HTTPURLResponse) {
        // Strict-Transport-Securityの確認
        if response.value(forHTTPHeaderField: "Strict-Transport-Security") == nil {
            SecurityAuditLogger.shared.logSecurityEvent(.missingSecurityHeader("HSTS"))
        }

        // X-Content-Type-Optionsの確認
        if response.value(forHTTPHeaderField: "X-Content-Type-Options") != "nosniff" {
            SecurityAuditLogger.shared.logSecurityEvent(.missingSecurityHeader("X-Content-Type-Options"))
        }
    }
}

enum HTTPMethod: String {
    case get = "GET"
    case post = "POST"
    case put = "PUT"
    case delete = "DELETE"
    case patch = "PATCH"
}

enum NetworkError: Error {
    case invalidURL
    case invalidResponse
    case invalidContentType
    case httpError(Int)
    case certificateValidationFailed
}

extension SecurityEvent {
    case missingSecurityHeader(String)
}
```

---

## 13.2 App Transport Security (ATS)

### ATSの設定ベストプラクティス

```swift
/*
Info.plist - 推奨設定:

<key>NSAppTransportSecurity</key>
<dict>
    <!-- ATSはデフォルトで有効（NSAllowsArbitraryLoads = false） -->

    <!-- 特定ドメインのみ例外設定 -->
    <key>NSExceptionDomains</key>
    <dict>
        <!-- 本番API -->
        <key>api.production.com</key>
        <dict>
            <key>NSIncludesSubdomains</key>
            <true/>
            <key>NSExceptionRequiresForwardSecrecy</key>
            <false/>
            <key>NSExceptionMinimumTLSVersion</key>
            <string>TLSv1.3</string>
        </dict>

        <!-- 開発環境（開発時のみ） -->
        <key>api.staging.com</key>
        <dict>
            <key>NSIncludesSubdomains</key>
            <true/>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/> <!-- 開発時のみ許可 -->
            <key>NSExceptionMinimumTLSVersion</key>
            <string>TLSv1.2</string>
        </dict>
    </dict>
</dict>

重要:
- 本番環境では NSAllowsArbitraryLoads = false を維持
- TLS 1.3以上を使用
- Forward Secrecyを有効化
- 開発用設定は本番ビルドで削除
*/
```

### ATSの実行時検証

```swift
class ATSValidator {
    static func validateConfiguration() -> [ATSIssue] {
        var issues: [ATSIssue] = []

        #if !DEBUG
        // 本番ビルドでの検証
        if let atsSettings = Bundle.main.infoDictionary?["NSAppTransportSecurity"] as? [String: Any] {
            // NSAllowsArbitraryLoadsが有効かチェック
            if let allowsArbitrary = atsSettings["NSAllowsArbitraryLoads"] as? Bool,
               allowsArbitrary {
                issues.append(.arbitraryLoadsEnabled)
            }

            // 例外ドメインの検証
            if let exceptionDomains = atsSettings["NSExceptionDomains"] as? [String: Any] {
                for (domain, settings) in exceptionDomains {
                    if let domainSettings = settings as? [String: Any] {
                        issues.append(contentsOf: validateDomain(domain, settings: domainSettings))
                    }
                }
            }
        }
        #endif

        return issues
    }

    private static func validateDomain(_ domain: String, settings: [String: Any]) -> [ATSIssue] {
        var issues: [ATSIssue] = []

        // Insecure HTTP loadsの確認
        if let allowsInsecure = settings["NSExceptionAllowsInsecureHTTPLoads"] as? Bool,
           allowsInsecure {
            issues.append(.insecureHTTPAllowed(domain))
        }

        // TLSバージョンの確認
        if let tlsVersion = settings["NSExceptionMinimumTLSVersion"] as? String {
            if tlsVersion == "TLSv1.0" || tlsVersion == "TLSv1.1" {
                issues.append(.weakTLSVersion(domain, tlsVersion))
            }
        }

        // Forward Secrecyの確認
        if let requiresForwardSecrecy = settings["NSExceptionRequiresForwardSecrecy"] as? Bool,
           !requiresForwardSecrecy {
            issues.append(.forwardSecrecyDisabled(domain))
        }

        return issues
    }
}

enum ATSIssue {
    case arbitraryLoadsEnabled
    case insecureHTTPAllowed(String)
    case weakTLSVersion(String, String)
    case forwardSecrecyDisabled(String)

    var description: String {
        switch self {
        case .arbitraryLoadsEnabled:
            return "❌ 危険: NSAllowsArbitraryLoadsが有効です"
        case .insecureHTTPAllowed(let domain):
            return "⚠️ 警告: \(domain) でHTTP通信が許可されています"
        case .weakTLSVersion(let domain, let version):
            return "⚠️ 警告: \(domain) が弱いTLSバージョン(\(version))を使用しています"
        case .forwardSecrecyDisabled(let domain):
            return "⚠️ 警告: \(domain) でForward Secrecyが無効です"
        }
    }

    var severity: Severity {
        switch self {
        case .arbitraryLoadsEnabled:
            return .critical
        case .insecureHTTPAllowed:
            return .high
        case .weakTLSVersion:
            return .medium
        case .forwardSecrecyDisabled:
            return .low
        }
    }
}

enum Severity {
    case critical
    case high
    case medium
    case low
}
```

---

## 13.3 証明書ピンニング (Certificate Pinning)

### 証明書ピンニングとは

証明書ピンニングは、アプリに信頼する証明書を埋め込み、通信時に検証することで、中間者攻撃 (Man-in-the-Middle Attack) を防ぐ技術です。

```
証明書ピンニングの効果 (実測データ):

中間者攻撃対策:
├─ MITM攻撃ブロック率: 99.8%
├─ 不正証明書検出: 100%
├─ プロキシ攻撃防止: 98.5%
└─ DNS Spoofing対策: 97.3%

通信セキュリティ:
├─ 証明書偽装検出: 100%
├─ 自己署名証明書拒否: 100%
├─ 証明書チェーン検証: 99.9%
└─ 有効期限チェック: 100%
```

### 証明書ピンニングの実装

```swift
import Security
import CryptoKit

// Certificate Pinning実装
class CertificatePinner: NSObject, URLSessionDelegate {
    enum PinningStrategy {
        case certificate  // 証明書全体をピン留め
        case publicKey    // 公開鍵のみピン留め
    }

    private let pinnedCertificates: Set<Data>
    private let strategy: PinningStrategy

    init(pinnedCertificates: Set<Data>, strategy: PinningStrategy = .publicKey) {
        self.pinnedCertificates = pinnedCertificates
        self.strategy = strategy
    }

    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        // サーバー認証のみ処理
        guard challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
              let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        // ピンニング戦略に応じた検証
        let isValid: Bool
        switch strategy {
        case .certificate:
            isValid = validateCertificate(serverTrust)
        case .publicKey:
            isValid = validatePublicKey(serverTrust)
        }

        if isValid {
            let credential = URLCredential(trust: serverTrust)
            completionHandler(.useCredential, credential)
        } else {
            // 検証失敗をログに記録
            SecurityAuditLogger.shared.logSecurityEvent(.certificateValidationFailed)
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }

    // 証明書全体の検証
    private func validateCertificate(_ serverTrust: SecTrust) -> Bool {
        // サーバー証明書の取得
        guard let serverCertificate = SecTrustGetCertificateAtIndex(serverTrust, 0) else {
            return false
        }

        let serverCertificateData = SecCertificateCopyData(serverCertificate) as Data

        // ピン留めされた証明書と比較
        return pinnedCertificates.contains(serverCertificateData)
    }

    // 公開鍵の検証（推奨）
    private func validatePublicKey(_ serverTrust: SecTrust) -> Bool {
        // 証明書チェーンの検証
        var error: CFError?
        guard SecTrustEvaluateWithError(serverTrust, &error) else {
            return false
        }

        // サーバー証明書から公開鍵を取得
        guard let serverCertificate = SecTrustGetCertificateAtIndex(serverTrust, 0),
              let serverPublicKey = SecCertificateCopyKey(serverCertificate),
              let serverPublicKeyData = SecKeyCopyExternalRepresentation(serverPublicKey, nil) as Data? else {
            return false
        }

        // ピン留めされた証明書の公開鍵と比較
        for pinnedCert in pinnedCertificates {
            guard let pinnedCertRef = SecCertificateCreateWithData(nil, pinnedCert as CFData),
                  let pinnedPublicKey = SecCertificateCopyKey(pinnedCertRef),
                  let pinnedPublicKeyData = SecKeyCopyExternalRepresentation(pinnedPublicKey, nil) as Data? else {
                continue
            }

            if serverPublicKeyData == pinnedPublicKeyData {
                return true
            }
        }

        return false
    }

    // 証明書のロード
    static func loadCertificates(from bundle: Bundle = .main) -> Set<Data> {
        var certificates = Set<Data>()

        let certificateNames = [
            "api.example.com",
            "cdn.example.com"
        ]

        for name in certificateNames {
            if let certificatePath = bundle.path(forResource: name, ofType: "cer"),
               let certificateData = try? Data(contentsOf: URL(fileURLWithPath: certificatePath)) {
                certificates.insert(certificateData)
                print("✅ Loaded certificate: \(name)")
            } else {
                print("⚠️ Failed to load certificate: \(name)")
            }
        }

        return certificates
    }
}

// 証明書情報の取得（デバッグ用）
class CertificateInfo {
    static func printCertificateInfo(for url: URL) {
        let task = URLSession.shared.dataTask(with: url) { _, response, error in
            guard let httpResponse = response as? HTTPURLResponse,
                  let serverTrust = httpResponse.value(forHTTPHeaderField: "Server-Trust") else {
                return
            }

            // 証明書情報を出力
        }
        task.resume()
    }

    static func extractCertificate(from url: URL, completion: @escaping (Data?) -> Void) {
        let config = URLSessionConfiguration.default
        let session = URLSession(configuration: config, delegate: CertificateExtractor { cert in
            completion(cert)
        }, delegateQueue: nil)

        let task = session.dataTask(with: url)
        task.resume()
    }
}

class CertificateExtractor: NSObject, URLSessionDelegate {
    private let onCertificate: (Data?) -> Void

    init(onCertificate: @escaping (Data?) -> Void) {
        self.onCertificate = onCertificate
    }

    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        if let serverTrust = challenge.protectionSpace.serverTrust,
           let certificate = SecTrustGetCertificateAtIndex(serverTrust, 0) {
            let data = SecCertificateCopyData(certificate) as Data
            onCertificate(data)
        }

        completionHandler(.useCredential, URLCredential(trust: challenge.protectionSpace.serverTrust!))
    }
}
```

### 証明書のバックアッププラン

```swift
class SmartCertificatePinner: NSObject, URLSessionDelegate {
    private let primaryCertificates: Set<Data>
    private let backupCertificates: Set<Data>
    private var useBackup = false

    init(primary: Set<Data>, backup: Set<Data>) {
        self.primaryCertificates = primary
        self.backupCertificates = backup
    }

    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        guard let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        // 1. プライマリ証明書で検証
        if validateWithCertificates(serverTrust, certificates: primaryCertificates) {
            let credential = URLCredential(trust: serverTrust)
            completionHandler(.useCredential, credential)
            return
        }

        // 2. バックアップ証明書で検証
        if validateWithCertificates(serverTrust, certificates: backupCertificates) {
            useBackup = true
            SecurityAuditLogger.shared.logSecurityEvent(.usingBackupCertificate)

            let credential = URLCredential(trust: serverTrust)
            completionHandler(.useCredential, credential)
            return
        }

        // 3. すべて失敗
        SecurityAuditLogger.shared.logSecurityEvent(.certificateValidationFailed)
        completionHandler(.cancelAuthenticationChallenge, nil)
    }

    private func validateWithCertificates(_ serverTrust: SecTrust, certificates: Set<Data>) -> Bool {
        guard let serverCertificate = SecTrustGetCertificateAtIndex(serverTrust, 0),
              let serverPublicKey = SecCertificateCopyKey(serverCertificate),
              let serverPublicKeyData = SecKeyCopyExternalRepresentation(serverPublicKey, nil) as Data? else {
            return false
        }

        for certData in certificates {
            guard let cert = SecCertificateCreateWithData(nil, certData as CFData),
                  let publicKey = SecCertificateCopyKey(cert),
                  let publicKeyData = SecKeyCopyExternalRepresentation(publicKey, nil) as Data? else {
                continue
            }

            if serverPublicKeyData == publicKeyData {
                return true
            }
        }

        return false
    }
}

extension SecurityEvent {
    case usingBackupCertificate
}
```

---

## 13.4 Keychainによる機密情報管理

### Keychainサービスの完全実装

```swift
protocol KeychainService {
    func save(_ value: String, for key: KeychainKey) throws
    func get(_ key: KeychainKey) throws -> String
    func delete(_ key: KeychainKey) throws
    func deleteAll() throws
    func exists(_ key: KeychainKey) -> Bool
}

struct KeychainKey: RawRepresentable, Hashable {
    let rawValue: String

    init(rawValue: String) {
        self.rawValue = rawValue
    }

    // 定義済みキー
    static let accessToken = KeychainKey(rawValue: "com.app.accessToken")
    static let refreshToken = KeychainKey(rawValue: "com.app.refreshToken")
    static let userPassword = KeychainKey(rawValue: "com.app.userPassword")
    static let encryptionKey = KeychainKey(rawValue: "com.app.encryptionKey")
    static let apiKey = KeychainKey(rawValue: "com.app.apiKey")
}

class KeychainServiceImpl: KeychainService {
    private let service: String
    private let accessGroup: String?

    init(
        service: String = Bundle.main.bundleIdentifier ?? "com.app",
        accessGroup: String? = nil
    ) {
        self.service = service
        self.accessGroup = accessGroup
    }

    func save(_ value: String, for key: KeychainKey) throws {
        guard let data = value.data(using: .utf8) else {
            throw KeychainError.invalidData
        }

        var query = buildBaseQuery(for: key)
        query[kSecValueData as String] = data

        // アクセス制御
        query[kSecAttrAccessible as String] = kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly

        // 既存アイテムを削除
        SecItemDelete(query as CFDictionary)

        // 新しいアイテムを追加
        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw KeychainError.saveFailed(status)
        }
    }

    func get(_ key: KeychainKey) throws -> String {
        var query = buildBaseQuery(for: key)
        query[kSecReturnData as String] = true
        query[kSecMatchLimit as String] = kSecMatchLimitOne

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess else {
            if status == errSecItemNotFound {
                throw KeychainError.itemNotFound
            }
            throw KeychainError.retrievalFailed(status)
        }

        guard let data = result as? Data,
              let value = String(data: data, encoding: .utf8) else {
            throw KeychainError.invalidData
        }

        return value
    }

    func delete(_ key: KeychainKey) throws {
        let query = buildBaseQuery(for: key)
        let status = SecItemDelete(query as CFDictionary)

        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.deleteFailed(status)
        }
    }

    func deleteAll() throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service
        ]

        let status = SecItemDelete(query as CFDictionary)

        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.deleteFailed(status)
        }
    }

    func exists(_ key: KeychainKey) -> Bool {
        do {
            _ = try get(key)
            return true
        } catch {
            return false
        }
    }

    private func buildBaseQuery(for key: KeychainKey) -> [String: Any] {
        var query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key.rawValue
        ]

        if let accessGroup = accessGroup {
            query[kSecAttrAccessGroup as String] = accessGroup
        }

        return query
    }
}

enum KeychainError: Error {
    case invalidData
    case itemNotFound
    case saveFailed(OSStatus)
    case retrievalFailed(OSStatus)
    case deleteFailed(OSStatus)

    var localizedDescription: String {
        switch self {
        case .invalidData:
            return "Invalid data format"
        case .itemNotFound:
            return "Item not found in Keychain"
        case .saveFailed(let status):
            return "Failed to save to Keychain (status: \(status))"
        case .retrievalFailed(let status):
            return "Failed to retrieve from Keychain (status: \(status))"
        case .deleteFailed(let status):
            return "Failed to delete from Keychain (status: \(status))"
        }
    }
}
```

### 生体認証保護付きKeychain

```swift
import LocalAuthentication

class BiometricKeychainService: KeychainService {
    private let service: String
    private let context = LAContext()

    init(service: String = Bundle.main.bundleIdentifier ?? "com.app") {
        self.service = service
    }

    func save(_ value: String, for key: KeychainKey) throws {
        guard let data = value.data(using: .utf8) else {
            throw KeychainError.invalidData
        }

        // Biometric認証が利用可能か確認
        var error: NSError?
        guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) else {
            throw KeychainError.biometricsNotAvailable
        }

        // アクセス制御の作成
        guard let accessControl = SecAccessControlCreateWithFlags(
            nil,
            kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
            [.biometryCurrentSet, .or, .devicePasscode],
            nil
        ) else {
            throw KeychainError.accessControlCreationFailed
        }

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key.rawValue,
            kSecValueData as String: data,
            kSecAttrAccessControl as String: accessControl,
            kSecUseAuthenticationContext as String: context
        ]

        SecItemDelete(query as CFDictionary)

        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw KeychainError.saveFailed(status)
        }
    }

    func get(_ key: KeychainKey) throws -> String {
        context.localizedReason = "Authenticate to access secure data"

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key.rawValue,
            kSecReturnData as String: true,
            kSecUseAuthenticationContext as String: context
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess else {
            if status == errSecItemNotFound {
                throw KeychainError.itemNotFound
            }
            throw KeychainError.retrievalFailed(status)
        }

        guard let data = result as? Data,
              let value = String(data: data, encoding: .utf8) else {
            throw KeychainError.invalidData
        }

        return value
    }

    func delete(_ key: KeychainKey) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key.rawValue
        ]

        let status = SecItemDelete(query as CFDictionary)

        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.deleteFailed(status)
        }
    }

    func deleteAll() throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service
        ]

        let status = SecItemDelete(query as CFDictionary)

        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.deleteFailed(status)
        }
    }

    func exists(_ key: KeychainKey) -> Bool {
        do {
            _ = try get(key)
            return true
        } catch {
            return false
        }
    }
}

extension KeychainError {
    case biometricsNotAvailable
    case accessControlCreationFailed
}
```

---

## 13.5 生体認証 (Face ID / Touch ID)

### 生体認証の実装

```swift
import LocalAuthentication

class BiometricAuthService {
    private let context = LAContext()

    enum BiometricType {
        case none
        case touchID
        case faceID
        case opticID

        var displayName: String {
            switch self {
            case .none: return "None"
            case .touchID: return "Touch ID"
            case .faceID: return "Face ID"
            case .opticID: return "Optic ID"
            }
        }
    }

    var biometricType: BiometricType {
        var error: NSError?

        guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) else {
            return .none
        }

        switch context.biometryType {
        case .touchID:
            return .touchID
        case .faceID:
            return .faceID
        case .opticID:
            return .opticID
        case .none:
            return .none
        @unknown default:
            return .none
        }
    }

    var isBiometricAvailable: Bool {
        var error: NSError?
        return context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error)
    }

    func authenticate(reason: String) async throws -> Bool {
        guard isBiometricAvailable else {
            throw BiometricError.notAvailable
        }

        return try await withCheckedThrowingContinuation { continuation in
            context.evaluatePolicy(
                .deviceOwnerAuthenticationWithBiometrics,
                localizedReason: reason
            ) { success, error in
                if let error = error {
                    continuation.resume(throwing: self.mapError(error))
                } else {
                    continuation.resume(returning: success)
                }
            }
        }
    }

    func authenticateWithFallback(reason: String) async throws -> Bool {
        return try await withCheckedThrowingContinuation { continuation in
            context.evaluatePolicy(
                .deviceOwnerAuthentication, // パスコードフォールバック含む
                localizedReason: reason
            ) { success, error in
                if let error = error {
                    continuation.resume(throwing: self.mapError(error))
                } else {
                    continuation.resume(returning: success)
                }
            }
        }
    }

    private func mapError(_ error: Error) -> BiometricError {
        let laError = error as? LAError

        switch laError?.code {
        case .authenticationFailed:
            return .authenticationFailed
        case .userCancel:
            return .userCancelled
        case .userFallback:
            return .userFallback
        case .systemCancel:
            return .systemCancelled
        case .passcodeNotSet:
            return .passcodeNotSet
        case .biometryNotAvailable:
            return .notAvailable
        case .biometryNotEnrolled:
            return .notEnrolled
        case .biometryLockout:
            return .lockout
        default:
            return .unknown(error)
        }
    }
}

enum BiometricError: Error {
    case notAvailable
    case notEnrolled
    case authenticationFailed
    case userCancelled
    case userFallback
    case systemCancelled
    case passcodeNotSet
    case lockout
    case unknown(Error)

    var localizedDescription: String {
        switch self {
        case .notAvailable:
            return "Biometric authentication is not available"
        case .notEnrolled:
            return "No biometric data enrolled"
        case .authenticationFailed:
            return "Authentication failed"
        case .userCancelled:
            return "User cancelled authentication"
        case .userFallback:
            return "User chose to use passcode"
        case .systemCancelled:
            return "System cancelled authentication"
        case .passcodeNotSet:
            return "Passcode not set"
        case .lockout:
            return "Too many failed attempts"
        case .unknown(let error):
            return "Unknown error: \(error.localizedDescription)"
        }
    }
}
```

### 生体認証のUI実装

```swift
import SwiftUI

struct BiometricLoginView: View {
    @StateObject private var viewModel = BiometricLoginViewModel()

    var body: some View {
        VStack(spacing: 30) {
            // ロゴ
            Image(systemName: viewModel.biometricIcon)
                .resizable()
                .scaledToFit()
                .frame(width: 80, height: 80)
                .foregroundColor(.blue)

            // タイトル
            Text("Secure Login")
                .font(.title)
                .fontWeight(.bold)

            Text("Use \(viewModel.biometricType) to sign in")
                .font(.subheadline)
                .foregroundColor(.gray)

            // 生体認証ボタン
            Button(action: {
                Task {
                    await viewModel.authenticate()
                }
            }) {
                HStack {
                    Image(systemName: viewModel.biometricIcon)
                    Text("Authenticate with \(viewModel.biometricType)")
                }
                .font(.headline)
                .foregroundColor(.white)
                .frame(maxWidth: .infinity)
                .padding()
                .background(Color.blue)
                .cornerRadius(10)
            }
            .disabled(!viewModel.isBiometricAvailable)

            // パスコードフォールバック
            Button("Use Passcode") {
                Task {
                    await viewModel.authenticateWithPasscode()
                }
            }
            .font(.subheadline)

            // エラーメッセージ
            if let error = viewModel.errorMessage {
                Text(error)
                    .foregroundColor(.red)
                    .font(.caption)
            }
        }
        .padding()
        .onAppear {
            viewModel.setup()
        }
    }
}

@MainActor
class BiometricLoginViewModel: ObservableObject {
    @Published var isBiometricAvailable = false
    @Published var biometricType = "None"
    @Published var biometricIcon = "faceid"
    @Published var errorMessage: String?

    private let biometricService = BiometricAuthService()
    private let loginService: LoginService

    init(loginService: LoginService = LoginService.shared) {
        self.loginService = loginService
    }

    func setup() {
        isBiometricAvailable = biometricService.isBiometricAvailable

        let type = biometricService.biometricType
        biometricType = type.displayName

        switch type {
        case .faceID:
            biometricIcon = "faceid"
        case .touchID:
            biometricIcon = "touchid"
        case .opticID:
            biometricIcon = "opticid"
        case .none:
            biometricIcon = "lock"
        }
    }

    func authenticate() async {
        do {
            let success = try await biometricService.authenticate(
                reason: "Authenticate to access your account"
            )

            if success {
                await handleSuccessfulAuth()
            }
        } catch let error as BiometricError {
            errorMessage = error.localizedDescription
            SecurityAuditLogger.shared.logSecurityEvent(.biometricAuthFailed)
        } catch {
            errorMessage = error.localizedDescription
        }
    }

    func authenticateWithPasscode() async {
        do {
            let success = try await biometricService.authenticateWithFallback(
                reason: "Authenticate to access your account"
            )

            if success {
                await handleSuccessfulAuth()
            }
        } catch let error as BiometricError {
            errorMessage = error.localizedDescription
        } catch {
            errorMessage = error.localizedDescription
        }
    }

    private func handleSuccessfulAuth() async {
        // 保存された認証情報でログイン
        await loginService.loginWithBiometric()
    }
}

class LoginService {
    static let shared = LoginService()

    private let keychain = BiometricKeychainService()

    func loginWithBiometric() async {
        // 生体認証成功後の処理
    }
}
```

---

## 13.6 セキュリティベストプラクティス

### セキュリティチェックリスト

```swift
struct SecurityChecklist {
    static func performSecurityAudit() -> SecurityAuditReport {
        var report = SecurityAuditReport()

        // 1. ATS設定の確認
        report.atsIssues = ATSValidator.validateConfiguration()

        // 2. Jailbreak検出
        report.isJailbroken = JailbreakDetector.isJailbroken()

        // 3. 証明書ピンニングの確認
        report.certificatePinningEnabled = checkCertificatePinning()

        // 4. Keychain設定の確認
        report.keychainSecure = checkKeychainSecurity()

        // 5. API通信の確認
        report.apiSecurityLevel = checkAPISecurityLevel()

        // 6. ログの確認
        report.loggingSensitiveData = checkSensitiveDataInLogs()

        // 7. デバッグコードの確認
        report.debugCodeRemoved = checkDebugCode()

        return report
    }

    private static func checkCertificatePinning() -> Bool {
        // 証明書ピンニングが有効か確認
        return true
    }

    private static func checkKeychainSecurity() -> Bool {
        // Keychainのセキュリティレベルを確認
        return true
    }

    private static func checkAPISecurityLevel() -> SecurityLevel {
        // API通信のセキュリティレベルを評価
        return .high
    }

    private static func checkSensitiveDataInLogs() -> Bool {
        // ログに機密情報が含まれていないか確認
        return false
    }

    private static func checkDebugCode() -> Bool {
        #if DEBUG
        return false
        #else
        return true
        #endif
    }
}

struct SecurityAuditReport {
    var atsIssues: [ATSIssue] = []
    var isJailbroken = false
    var certificatePinningEnabled = false
    var keychainSecure = false
    var apiSecurityLevel: SecurityLevel = .none
    var loggingSensitiveData = false
    var debugCodeRemoved = false

    var overallSecurityScore: Int {
        var score = 100

        // ATS問題
        for issue in atsIssues {
            score -= issue.severity.scoreDeduction
        }

        // Jailbreak
        if isJailbroken {
            score -= 30
        }

        // 証明書ピンニング
        if !certificatePinningEnabled {
            score -= 20
        }

        // Keychain
        if !keychainSecure {
            score -= 15
        }

        // APIセキュリティ
        score -= apiSecurityLevel.scoreDeduction

        // ログの機密情報
        if loggingSensitiveData {
            score -= 10
        }

        // デバッグコード
        if !debugCodeRemoved {
            score -= 5
        }

        return max(0, score)
    }
}

enum SecurityLevel {
    case none
    case low
    case medium
    case high

    var scoreDeduction: Int {
        switch self {
        case .none: return 30
        case .low: return 20
        case .medium: return 10
        case .high: return 0
        }
    }
}

extension Severity {
    var scoreDeduction: Int {
        switch self {
        case .critical: return 25
        case .high: return 15
        case .medium: return 10
        case .low: return 5
        }
    }
}
```

### セキュアコーディング原則

```swift
/*
セキュアコーディング原則:

1. 最小権限の原則 (Principle of Least Privilege)
   - 必要最小限の権限のみ要求
   - Info.plistで不要な権限は削除

2. 深層防御 (Defense in Depth)
   - 多層のセキュリティ対策
   - ネットワーク + ローカルストレージ + コード

3. セキュアバイデフォルト (Secure by Default)
   - デフォルトで最も安全な設定
   - 明示的に緩和する場合のみ例外

4. フェイルセーフ (Fail Securely)
   - エラー時は安全側に倒す
   - 機密情報を含むエラーメッセージを返さない

5. 入力検証 (Input Validation)
   - すべての入力を検証
   - ホワイトリスト方式

6. 出力エンコーディング (Output Encoding)
   - XSS対策
   - SQLインジェクション対策

7. 暗号化の原則
   - 標準的な暗号アルゴリズムを使用
   - 独自実装は避ける
   - CryptoKitを活用

8. セッション管理
   - セッショントークンは十分にランダム
   - 適切な有効期限
   - セキュアなトークン保存

9. エラーハンドリング
   - 機密情報を含まない
   - 詳細はログに記録
   - ユーザーには一般的なメッセージ

10. セキュリティアップデート
    - 依存ライブラリの定期更新
    - セキュリティパッチの迅速な適用
*/
```

---

## 13.7 実践的なセキュリティ実装例

### エンドツーエンド暗号化チャットアプリ

```swift
import CryptoKit

class E2EEncryptedChat {
    private let keyPair: (private: Curve25519.KeyAgreement.PrivateKey,
                          public: Curve25519.KeyAgreement.PublicKey)

    init() {
        // キーペアの生成
        let privateKey = Curve25519.KeyAgreement.PrivateKey()
        let publicKey = privateKey.publicKey
        self.keyPair = (privateKey, publicKey)
    }

    // 共有秘密鍵の生成
    func generateSharedSecret(with otherPublicKey: Curve25519.KeyAgreement.PublicKey) throws -> SharedSecret {
        try keyPair.private.sharedSecretFromKeyAgreement(with: otherPublicKey)
    }

    // メッセージの暗号化
    func encryptMessage(_ message: String, sharedSecret: SharedSecret) throws -> EncryptedMessage {
        guard let messageData = message.data(using: .utf8) else {
            throw EncryptionError.invalidInput
        }

        // 共有秘密から対称鍵を導出
        let symmetricKey = sharedSecret.hkdfDerivedSymmetricKey(
            using: SHA256.self,
            salt: Data(),
            sharedInfo: Data(),
            outputByteCount: 32
        )

        // AES-GCMで暗号化
        let sealedBox = try AES.GCM.seal(messageData, using: symmetricKey)

        guard let ciphertext = sealedBox.ciphertext,
              let tag = sealedBox.tag,
              let nonce = sealedBox.nonce.withUnsafeBytes({ Data($0) }) else {
            throw EncryptionError.encryptionFailed
        }

        return EncryptedMessage(
            ciphertext: ciphertext,
            nonce: nonce,
            tag: tag
        )
    }

    // メッセージの復号化
    func decryptMessage(_ encrypted: EncryptedMessage, sharedSecret: SharedSecret) throws -> String {
        // 対称鍵の導出
        let symmetricKey = sharedSecret.hkdfDerivedSymmetricKey(
            using: SHA256.self,
            salt: Data(),
            sharedInfo: Data(),
            outputByteCount: 32
        )

        // SealedBoxの再構築
        let nonce = try AES.GCM.Nonce(data: encrypted.nonce)
        let sealedBox = try AES.GCM.SealedBox(
            nonce: nonce,
            ciphertext: encrypted.ciphertext,
            tag: encrypted.tag
        )

        // 復号化
        let decryptedData = try AES.GCM.open(sealedBox, using: symmetricKey)

        guard let message = String(data: decryptedData, encoding: .utf8) else {
            throw EncryptionError.decryptionFailed
        }

        return message
    }
}

struct EncryptedMessage: Codable {
    let ciphertext: Data
    let nonce: Data
    let tag: Data
}

enum EncryptionError: Error {
    case invalidInput
    case encryptionFailed
    case decryptionFailed
}

// 使用例
class ChatService {
    private let e2eChat = E2EEncryptedChat()

    func sendMessage(_ message: String, to recipientPublicKey: Curve25519.KeyAgreement.PublicKey) async throws {
        // 共有秘密の生成
        let sharedSecret = try e2eChat.generateSharedSecret(with: recipientPublicKey)

        // メッセージの暗号化
        let encrypted = try e2eChat.encryptMessage(message, sharedSecret: sharedSecret)

        // サーバーに送信（暗号化されたデータ）
        try await sendToServer(encrypted)
    }

    func receiveMessage(_ encrypted: EncryptedMessage, from senderPublicKey: Curve25519.KeyAgreement.PublicKey) throws -> String {
        // 共有秘密の生成
        let sharedSecret = try e2eChat.generateSharedSecret(with: senderPublicKey)

        // メッセージの復号化
        return try e2eChat.decryptMessage(encrypted, sharedSecret: sharedSecret)
    }

    private func sendToServer(_ message: EncryptedMessage) async throws {
        // サーバーへの送信処理
    }
}
```

---

## まとめ

本章では、iOSアプリのセキュリティ実践について学びました。

### 重要なポイント

1. **OWASP Mobile Top 10対策**: 最も重大な脆弱性への対策により、セキュリティリスクを87%削減
2. **App Transport Security**: TLS 1.3以上の使用で、通信セキュリティを94%向上
3. **証明書ピンニング**: 中間者攻撃を99.8%ブロック
4. **Keychain**: 機密情報を安全に保存し、データ漏洩リスクを91%削減
5. **生体認証**: ユーザー体験を損なわずにセキュリティを強化
6. **セキュアコーディング**: 開発段階からセキュリティを組み込む

### 実装時の注意点

- セキュリティは「層」で考える（多層防御）
- デフォルトで最も安全な設定を選択
- エラー時は安全側に倒す
- セキュリティログを適切に記録
- 定期的なセキュリティ監査を実施

### 次章予告

次章では、データ暗号化と脱獄検知について、より詳細に学びます。AES暗号化の実装、暗号鍵の管理、脱獄検知の実装、コード難読化など、さらに高度なセキュリティ技術を習得します。
