---
title: "セキュリティチェックリスト"
---

# セキュリティチェックリスト

本書で学んだセキュリティ対策を総括し、実践的なチェックリストとコード例として整理します。iOS開発におけるセキュリティは、開発フェーズから運用フェーズまで継続的に意識する必要があります。

## 開発フェーズ

### コーディング

**チェックリスト**:

- [ ] 機密情報（APIキー、トークン、パスワード）をコードに直接記述していない
- [ ] ログ出力に機密情報を含めていない
- [ ] UserDefaultsに機密情報を保存していない
- [ ] デバッグコードを本番ビルドから除外している
- [ ] Force unwrap（`!`）を最小限にし、Optional Bindingを使用している

**実装例**:

```swift
// ❌ 悪い例: APIキーをハードコーディング
let apiKey = "sk_live_12345abcdef"

// ✅ 良い例: 環境変数から読み込む
guard let apiKey = ProcessInfo.processInfo.environment["API_KEY"] else {
    fatalError("API_KEY not found in environment")
}

// ✅ 良い例: xcconfig ファイルで管理
// Config.xcconfig:
// API_KEY = sk_live_12345abcdef
// Info.plist:
// <key>API_KEY</key>
// <string>$(API_KEY)</string>

let apiKey = Bundle.main.object(forInfoDictionaryKey: "API_KEY") as? String

// ❌ 悪い例: ログに機密情報を出力
print("Auth token: \(authToken)")

// ✅ 良い例: 本番ビルドではログを出力しない
#if DEBUG
print("Debug info: \(debugInfo)")
#endif

// または専用のログ関数を使用
func secureLog(_ message: String) {
    #if DEBUG
    print(message)
    #endif
}
```

### 認証・認可

**チェックリスト**:

- [ ] OAuth 2.0 / OpenID Connectなどの標準的な認証プロトコルを使用している
- [ ] アクセストークンをKeychainに安全に保存している
- [ ] トークンリフレッシュ機構を実装している
- [ ] 生体認証（Face ID/Touch ID）を実装している
- [ ] セッションタイムアウトを適切に設定している

**実装例**:

```swift
// Keychainへのトークン保存
class KeychainManager {
    static let shared = KeychainManager()

    func saveToken(_ token: String, forKey key: String) throws {
        let data = token.data(using: .utf8)!

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlock
        ]

        // 既存のアイテムを削除
        SecItemDelete(query as CFDictionary)

        // 新しいアイテムを追加
        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw KeychainError.saveFailed(status)
        }
    }

    func loadToken(forKey key: String) throws -> String {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess,
              let data = result as? Data,
              let token = String(data: data, encoding: .utf8) else {
            throw KeychainError.loadFailed(status)
        }

        return token
    }

    func deleteToken(forKey key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key
        ]

        let status = SecItemDelete(query as CFDictionary)

        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.deleteFailed(status)
        }
    }
}

enum KeychainError: Error {
    case saveFailed(OSStatus)
    case loadFailed(OSStatus)
    case deleteFailed(OSStatus)
}

// 生体認証の実装
import LocalAuthentication

class BiometricAuthManager {
    static let shared = BiometricAuthManager()

    func authenticate(reason: String) async throws -> Bool {
        let context = LAContext()
        var error: NSError?

        guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) else {
            throw BiometricError.notAvailable
        }

        do {
            let success = try await context.evaluatePolicy(
                .deviceOwnerAuthenticationWithBiometrics,
                localizedReason: reason
            )
            return success
        } catch {
            throw BiometricError.authenticationFailed(error)
        }
    }

    func isBiometricAvailable() -> Bool {
        let context = LAContext()
        return context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: nil)
    }

    func biometricType() -> BiometricType {
        let context = LAContext()
        guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: nil) else {
            return .none
        }

        switch context.biometryType {
        case .faceID:
            return .faceID
        case .touchID:
            return .touchID
        case .none:
            return .none
        @unknown default:
            return .none
        }
    }
}

enum BiometricType {
    case faceID
    case touchID
    case none
}

enum BiometricError: Error {
    case notAvailable
    case authenticationFailed(Error)
}
```

### データ保護

**チェックリスト**:

- [ ] 機密データをAES暗号化している
- [ ] ファイル保護属性（File Protection）を設定している
- [ ] Keychainのアクセシビリティ属性を適切に設定している
- [ ] パスワードをハッシュ化して保存している
- [ ] データベースを暗号化している（必要に応じて）

**実装例**:

```swift
import CryptoKit

// AES-256暗号化
class EncryptionManager {
    static let shared = EncryptionManager()

    // 暗号化キーの生成と保存
    func generateKey() throws -> SymmetricKey {
        let key = SymmetricKey(size: .bits256)
        let keyData = key.withUnsafeBytes { Data($0) }

        // Keychainに保存
        let query: [String: Any] = [
            kSecClass as String: kSecClassKey,
            kSecAttrApplicationTag as String: "encryptionKey",
            kSecValueData as String: keyData,
            kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlock
        ]

        SecItemDelete(query as CFDictionary)
        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw EncryptionError.keyGenerationFailed
        }

        return key
    }

    func loadKey() throws -> SymmetricKey {
        let query: [String: Any] = [
            kSecClass as String: kSecClassKey,
            kSecAttrApplicationTag as String: "encryptionKey",
            kSecReturnData as String: true
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess,
              let keyData = result as? Data else {
            throw EncryptionError.keyLoadFailed
        }

        return SymmetricKey(data: keyData)
    }

    // データの暗号化
    func encrypt(_ data: Data) throws -> Data {
        let key = try loadKey()
        let sealedBox = try AES.GCM.seal(data, using: key)

        guard let combined = sealedBox.combined else {
            throw EncryptionError.encryptionFailed
        }

        return combined
    }

    // データの復号化
    func decrypt(_ encryptedData: Data) throws -> Data {
        let key = try loadKey()
        let sealedBox = try AES.GCM.SealedBox(combined: encryptedData)
        let decryptedData = try AES.GCM.open(sealedBox, using: key)

        return decryptedData
    }
}

enum EncryptionError: Error {
    case keyGenerationFailed
    case keyLoadFailed
    case encryptionFailed
    case decryptionFailed
}

// ファイル保護属性の設定
func saveSecureFile(data: Data, filename: String) throws {
    let fileManager = FileManager.default
    let documentsURL = fileManager.urls(for: .documentDirectory, in: .userDomainMask)[0]
    let fileURL = documentsURL.appendingPathComponent(filename)

    // ファイルを保存
    try data.write(to: fileURL)

    // ファイル保護属性を設定
    try fileManager.setAttributes(
        [FileAttributeKey.protectionKey: FileProtectionType.complete],
        ofItemAtPath: fileURL.path
    )
}

// パスワードのハッシュ化
import CryptoKit

func hashPassword(_ password: String, salt: String) -> String {
    let combined = password + salt
    let data = Data(combined.utf8)
    let hashed = SHA256.hash(data: data)
    return hashed.compactMap { String(format: "%02x", $0) }.joined()
}

// 使用例
let salt = UUID().uuidString
let hashedPassword = hashPassword("userPassword123", salt: salt)

// saltとhashedPasswordを保存（平文のパスワードは破棄）
```

### ネットワークセキュリティ

**チェックリスト**:

- [ ] すべての通信でHTTPSを使用している
- [ ] 証明書ピンニングを実装している
- [ ] App Transport Security（ATS）を有効化している
- [ ] API通信を認証トークンで保護している
- [ ] レスポンスデータを検証している

**実装例**:

```swift
// App Transport Security（Info.plist）
/*
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <false/>
    <key>NSExceptionDomains</key>
    <dict>
        <key>api.example.com</key>
        <dict>
            <key>NSIncludesSubdomains</key>
            <true/>
            <key>NSRequiresCertificateTransparency</key>
            <true/>
            <key>NSExceptionMinimumTLSVersion</key>
            <string>TLSv1.3</string>
        </dict>
    </dict>
</dict>
*/

// 証明書ピンニング
class CertificatePinningDelegate: NSObject, URLSessionDelegate {
    private let expectedCertificateHash = "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA="

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

        // サーバー証明書を取得
        guard let certificate = SecTrustCopyCertificateChain(serverTrust) as? [SecCertificate],
              let serverCertificate = certificate.first else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        // 証明書のハッシュを計算
        let serverCertificateData = SecCertificateCopyData(serverCertificate) as Data
        let serverCertificateHash = sha256(data: serverCertificateData)

        // ピンニングされた証明書のハッシュと比較
        if serverCertificateHash == expectedCertificateHash {
            let credential = URLCredential(trust: serverTrust)
            completionHandler(.useCredential, credential)
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }

    private func sha256(data: Data) -> String {
        let hash = SHA256.hash(data: data)
        let hashData = Data(hash)
        return "sha256/" + hashData.base64EncodedString()
    }
}

// 使用例
let delegate = CertificatePinningDelegate()
let session = URLSession(configuration: .default, delegate: delegate, delegateQueue: nil)

// APIリクエストに認証トークンを追加
class AuthenticatedAPIClient {
    private let session: URLSession

    init(session: URLSession = .shared) {
        self.session = session
    }

    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        var request = try endpoint.asURLRequest()

        // 認証トークンをヘッダーに追加
        if let token = try? KeychainManager.shared.loadToken(forKey: "authToken") {
            request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }

        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.invalidResponse
        }

        // レスポンスデータを検証
        let decoder = JSONDecoder()
        do {
            let decoded = try decoder.decode(T.self, from: data)
            return decoded
        } catch {
            throw NetworkError.decodingFailed(error)
        }
    }
}
```

### 脱獄検知

**チェックリスト**:

- [ ] 脱獄デバイスを検知する機能を実装している
- [ ] 脱獄検知時の適切な対応を実装している
- [ ] デバッグビルドでは脱獄検知をスキップしている

**実装例**:

```swift
class JailbreakDetector {
    static let shared = JailbreakDetector()

    func isJailbroken() -> Bool {
        #if DEBUG
        // デバッグビルドでは常にfalseを返す
        return false
        #else
        return checkSuspiciousFiles() ||
               checkSuspiciousApps() ||
               checkCanWriteToSystemDirectory() ||
               checkCydiaURLScheme() ||
               checkFork()
        #endif
    }

    // 疑わしいファイルの存在チェック
    private func checkSuspiciousFiles() -> Bool {
        let suspiciousFiles = [
            "/Applications/Cydia.app",
            "/Library/MobileSubstrate/MobileSubstrate.dylib",
            "/bin/bash",
            "/usr/sbin/sshd",
            "/etc/apt",
            "/private/var/lib/apt/",
            "/private/var/lib/cydia",
            "/private/var/mobile/Library/SBSettings/Themes",
            "/private/var/tmp/cydia.log",
            "/private/var/stash"
        ]

        for path in suspiciousFiles {
            if FileManager.default.fileExists(atPath: path) {
                return true
            }
        }

        return false
    }

    // 疑わしいアプリの存在チェック
    private func checkSuspiciousApps() -> Bool {
        guard let cydiaURL = URL(string: "cydia://package/com.example.package") else {
            return false
        }
        return UIApplication.shared.canOpenURL(cydiaURL)
    }

    // システムディレクトリへの書き込みチェック
    private func checkCanWriteToSystemDirectory() -> Bool {
        let testPath = "/private/jailbreak_test.txt"
        let testString = "Jailbreak test"

        do {
            try testString.write(toFile: testPath, atomically: true, encoding: .utf8)
            try FileManager.default.removeItem(atPath: testPath)
            return true
        } catch {
            return false
        }
    }

    // CydiaのURLスキームチェック
    private func checkCydiaURLScheme() -> Bool {
        if let url = URL(string: "cydia://") {
            return UIApplication.shared.canOpenURL(url)
        }
        return false
    }

    // fork()システムコールのチェック（サンドボックス外で動作可能か）
    private func checkFork() -> Bool {
        let result = fork()
        if result >= 0 {
            return true
        }
        return false
    }

    // 脱獄検知時の対応
    func handleJailbreakDetected() {
        // 警告を表示
        let alert = UIAlertController(
            title: "セキュリティ警告",
            message: "このデバイスは改造されている可能性があります。セキュリティ上の理由により、一部の機能が制限されます。",
            preferredStyle: .alert
        )
        alert.addAction(UIAlertAction(title: "OK", style: .default))

        // アプリを終了するか、機能を制限する
        // exit(0) // アプリを終了する場合（App Storeの審査で却下される可能性あり）

        // または、機能を制限する
        UserDefaults.standard.set(true, forKey: "isJailbroken")
    }
}
```

## テストフェーズ

### セキュリティテスト

**チェックリスト**:

- [ ] 脱獄デバイスでの動作を確認している
- [ ] 中間者攻撃（MITM）への対策を確認している
- [ ] SQLインジェクション対策を確認している
- [ ] クロスサイトスクリプティング（XSS）対策を確認している
- [ ] セッション管理の安全性を確認している

### ペネトレーションテスト

**チェックリスト**:

- [ ] 脆弱性スキャンを実施している
- [ ] APIエンドポイントのセキュリティを確認している
- [ ] 認証・認可のバイパスを試みている
- [ ] データ漏洩の可能性を検証している
- [ ] サードパーティライブラリの脆弱性を確認している

## リリース前

### コード監査

**チェックリスト**:

- [ ] SwiftLintでコード品質を確認している
- [ ] 静的解析ツールで脆弱性をスキャンしている
- [ ] 依存ライブラリの脆弱性を確認している（`pod outdated`、`swift package outdated`）
- [ ] APIキーやシークレットが含まれていないことを確認している
- [ ] デバッグコードやログ出力を削除している

### App Store申請

**チェックリスト**:

- [ ] Privacy Policy（プライバシーポリシー）を用意している
- [ ] 必要な権限の説明（NSCameraUsageDescription等）を記載している
- [ ] データ収集について明示している
- [ ] 暗号化の使用を申告している（Export Compliance）
- [ ] サードパーティSDKの利用を申告している

## 運用フェーズ

### 監視

**チェックリスト**:

- [ ] クラッシュレポート（Crashlytics、Sentry等）を監視している
- [ ] 異常なAPI呼び出しを検知している
- [ ] セキュリティインシデントの報告体制がある
- [ ] 定期的なセキュリティアップデートを実施している

### 対応

**チェックリスト**:

- [ ] セキュリティパッチの適用計画がある
- [ ] インシデント対応手順が文書化されている
- [ ] 緊急リリースの体制が整っている
- [ ] ユーザーへの通知手段が確立されている

## セキュリティ総合チェックリスト

開発から運用まで、全フェーズで確認すべき項目：

### データ保護
- [ ] 機密情報はKeychainに保存
- [ ] UserDefaultsに機密情報を保存していない
- [ ] ファイルは適切な保護属性で保存
- [ ] データベースは暗号化（必要に応じて）

### ネットワーク
- [ ] すべての通信でHTTPSを使用
- [ ] 証明書ピンニングを実装
- [ ] ATSを有効化
- [ ] APIリクエストに認証トークンを追加

### 認証・認可
- [ ] OAuth 2.0などの標準プロトコルを使用
- [ ] トークンをKeychainに保存
- [ ] 生体認証を実装
- [ ] セッションタイムアウトを設定

### コード品質
- [ ] Force unwrapを最小限に
- [ ] デバッグコードを本番ビルドから除外
- [ ] ログに機密情報を出力していない
- [ ] 静的解析ツールを活用

### ライブラリ管理
- [ ] 依存ライブラリを定期的に更新
- [ ] 脆弱性のあるライブラリを使用していない
- [ ] ライセンスを確認

## まとめ

本書では、iOS開発における包括的な技術とベストプラクティスを学びました：

- **Part 1: プロジェクトセットアップ**
  - Xcodeプロジェクトの初期設定からCI/CD構築まで

- **Part 2: セキュリティ実装**
  - 認証・暗号化・証明書ピンニング・脱獄検知

- **Part 3: ネットワーク・データ永続化**
  - API通信・Core Data・Realm・キャッシュ戦略

- **Part 4: 実践パターン**
  - アーキテクチャ設計・データ永続化戦略・セキュリティチェックリスト

これらの知識を活用し、セキュアで保守性の高いiOSアプリケーションを開発しましょう。セキュリティは一度実装したら終わりではなく、継続的に見直し、改善していく必要があります。

## 参考文献

- [Apple Developer Documentation - Security](https://developer.apple.com/documentation/security)
- [OWASP Mobile Security Project](https://owasp.org/www-project-mobile-security/)
- [Apple Developer Documentation - App Transport Security](https://developer.apple.com/documentation/security/preventing_insecure_network_connections)
- [Apple Developer Documentation - Keychain Services](https://developer.apple.com/documentation/security/keychain_services)
- [Apple Developer Documentation - Local Authentication](https://developer.apple.com/documentation/localauthentication)
- [CryptoKit - Apple Developer](https://developer.apple.com/documentation/cryptokit)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)

---

**付録**

- 公式ドキュメント
  - [Apple Developer Documentation](https://developer.apple.com/documentation/)
  - [Swift.org](https://swift.org/)
  - [Fastlane Documentation](https://docs.fastlane.tools/)

- 推奨ライブラリ
  - [Alamofire](https://github.com/Alamofire/Alamofire)
  - [SwiftLint](https://github.com/realm/SwiftLint)
  - [Realm Swift](https://github.com/realm/realm-swift)

- セキュリティツール
  - [OWASP ZAP](https://www.zaproxy.org/)
  - [MobSF](https://github.com/MobSF/Mobile-Security-Framework-MobSF)
  - [Snyk](https://snyk.io/)

継続的な学習と実践により、iOS開発スキルをさらに向上させていきましょう。
