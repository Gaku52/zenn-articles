---
title: "Chapter 14: UserDefaultsとKeychain Part 2 - Keychainとセキュア管理"
---

# Chapter 14: UserDefaultsとKeychain Part 2 - Keychainとセキュア管理

## 14.1 概要

前章ではUserDefaultsを使った設定管理について学びました。この章では、パスワード、トークン、証明書などの機密情報を安全に保存するためのKeychainについて詳しく解説します。

### 本章で学べること

- Keychainの基本的な使い方
- 認証情報の安全な管理
- 生体認証付きKeychain
- App Groups対応
- テストとデバッグ
- 実践的なベストプラクティス

## 14.2 Keychain - 機密情報の安全な保存

### 14.2.1 Keychainの基本実装

Keychainは、パスワード、トークン、証明書などの機密情報を安全に保存するためのシステムです。

```swift
import Security
import Foundation

class KeychainManager {
    static let shared = KeychainManager()

    private init() {}

    // MARK: - Save
    func save(_ data: Data, service: String, account: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecValueData as String: data
        ]

        // 既存のアイテムを削除
        SecItemDelete(query as CFDictionary)

        // 新しいアイテムを追加
        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw KeychainError.saveFailed(status)
        }
    }

    func save(_ string: String, service: String, account: String) throws {
        guard let data = string.data(using: .utf8) else {
            throw KeychainError.stringConversionFailed
        }
        try save(data, service: service, account: account)
    }

    // MARK: - Load
    func load(service: String, account: String) throws -> Data {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess else {
            if status == errSecItemNotFound {
                throw KeychainError.itemNotFound
            }
            throw KeychainError.loadFailed(status)
        }

        guard let data = result as? Data else {
            throw KeychainError.dataConversionFailed
        }

        return data
    }

    func loadString(service: String, account: String) throws -> String {
        let data = try load(service: service, account: account)
        guard let string = String(data: data, encoding: .utf8) else {
            throw KeychainError.stringConversionFailed
        }
        return string
    }

    // MARK: - Update
    func update(_ data: Data, service: String, account: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account
        ]

        let attributes: [String: Any] = [
            kSecValueData as String: data
        ]

        let status = SecItemUpdate(query as CFDictionary, attributes as CFDictionary)

        guard status == errSecSuccess else {
            if status == errSecItemNotFound {
                // アイテムが存在しない場合は新規作成
                try save(data, service: service, account: account)
                return
            }
            throw KeychainError.updateFailed(status)
        }
    }

    func update(_ string: String, service: String, account: String) throws {
        guard let data = string.data(using: .utf8) else {
            throw KeychainError.stringConversionFailed
        }
        try update(data, service: service, account: account)
    }

    // MARK: - Delete
    func delete(service: String, account: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account
        ]

        let status = SecItemDelete(query as CFDictionary)

        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.deleteFailed(status)
        }
    }

    // MARK: - Exists
    func exists(service: String, account: String) -> Bool {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecReturnData as String: false
        ]

        let status = SecItemCopyMatching(query as CFDictionary, nil)
        return status == errSecSuccess
    }

    // MARK: - List All
    func listAll(service: String) throws -> [String] {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecReturnAttributes as String: true,
            kSecMatchLimit as String: kSecMatchLimitAll
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess else {
            if status == errSecItemNotFound {
                return []
            }
            throw KeychainError.loadFailed(status)
        }

        guard let items = result as? [[String: Any]] else {
            return []
        }

        return items.compactMap { $0[kSecAttrAccount as String] as? String }
    }

    // MARK: - Delete All
    func deleteAll(service: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service
        ]

        let status = SecItemDelete(query as CFDictionary)

        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.deleteFailed(status)
        }
    }
}

// MARK: - Keychain Errors
enum KeychainError: Error, LocalizedError {
    case saveFailed(OSStatus)
    case loadFailed(OSStatus)
    case updateFailed(OSStatus)
    case deleteFailed(OSStatus)
    case itemNotFound
    case stringConversionFailed
    case dataConversionFailed
    case unexpectedData

    var errorDescription: String? {
        switch self {
        case .saveFailed(let status):
            return "Failed to save to keychain: \(status) - \(statusMessage(status))"
        case .loadFailed(let status):
            return "Failed to load from keychain: \(status) - \(statusMessage(status))"
        case .updateFailed(let status):
            return "Failed to update keychain: \(status) - \(statusMessage(status))"
        case .deleteFailed(let status):
            return "Failed to delete from keychain: \(status) - \(statusMessage(status))"
        case .itemNotFound:
            return "Item not found in keychain"
        case .stringConversionFailed:
            return "String conversion failed"
        case .dataConversionFailed:
            return "Data conversion failed"
        case .unexpectedData:
            return "Unexpected data format"
        }
    }

    private func statusMessage(_ status: OSStatus) -> String {
        switch status {
        case errSecSuccess:
            return "Success"
        case errSecItemNotFound:
            return "Item not found"
        case errSecDuplicateItem:
            return "Duplicate item"
        case errSecAuthFailed:
            return "Authentication failed"
        case errSecParam:
            return "Invalid parameter"
        default:
            return "Unknown error"
        }
    }
}
```

### 14.2.2 認証情報マネージャー

```swift
class AuthenticationManager {
    static let shared = AuthenticationManager()

    private let keychain = KeychainManager.shared
    private let service = Bundle.main.bundleIdentifier ?? "com.example.app"

    private enum Account {
        static let accessToken = "accessToken"
        static let refreshToken = "refreshToken"
        static let username = "username"
        static let password = "password"
        static let apiKey = "apiKey"
        static let userID = "userID"
    }

    private init() {}

    // MARK: - Access Token
    func saveAccessToken(_ token: String) throws {
        try keychain.save(token, service: service, account: Account.accessToken)
    }

    func loadAccessToken() throws -> String {
        try keychain.loadString(service: service, account: Account.accessToken)
    }

    func deleteAccessToken() throws {
        try keychain.delete(service: service, account: Account.accessToken)
    }

    var hasAccessToken: Bool {
        keychain.exists(service: service, account: Account.accessToken)
    }

    // MARK: - Refresh Token
    func saveRefreshToken(_ token: String) throws {
        try keychain.save(token, service: service, account: Account.refreshToken)
    }

    func loadRefreshToken() throws -> String {
        try keychain.loadString(service: service, account: Account.refreshToken)
    }

    func deleteRefreshToken() throws {
        try keychain.delete(service: service, account: Account.refreshToken)
    }

    // MARK: - Credentials
    func saveCredentials(username: String, password: String) throws {
        try keychain.save(username, service: service, account: Account.username)
        try keychain.save(password, service: service, account: Account.password)
    }

    func loadCredentials() throws -> (username: String, password: String) {
        let username = try keychain.loadString(service: service, account: Account.username)
        let password = try keychain.loadString(service: service, account: Account.password)
        return (username, password)
    }

    func deleteCredentials() throws {
        try keychain.delete(service: service, account: Account.username)
        try keychain.delete(service: service, account: Account.password)
    }

    // MARK: - API Key
    func saveAPIKey(_ key: String) throws {
        try keychain.save(key, service: service, account: Account.apiKey)
    }

    func loadAPIKey() throws -> String {
        try keychain.loadString(service: service, account: Account.apiKey)
    }

    func deleteAPIKey() throws {
        try keychain.delete(service: service, account: Account.apiKey)
    }

    // MARK: - User ID
    func saveUserID(_ id: String) throws {
        try keychain.save(id, service: service, account: Account.userID)
    }

    func loadUserID() throws -> String {
        try keychain.loadString(service: service, account: Account.userID)
    }

    func deleteUserID() throws {
        try keychain.delete(service: service, account: Account.userID)
    }

    // MARK: - Session Management
    func saveSession(accessToken: String, refreshToken: String, userID: String) throws {
        try saveAccessToken(accessToken)
        try saveRefreshToken(refreshToken)
        try saveUserID(userID)
    }

    func clearSession() throws {
        try? deleteAccessToken()
        try? deleteRefreshToken()
        try? deleteUserID()
    }

    func isLoggedIn() -> Bool {
        hasAccessToken
    }

    // MARK: - Complete Logout
    func logout() throws {
        try clearSession()
        try? deleteCredentials()
        try? deleteAPIKey()
    }
}

// 使用例
let authManager = AuthenticationManager.shared

// ログイン時
do {
    try authManager.saveSession(
        accessToken: "eyJhbGciOiJIUzI1NiIs...",
        refreshToken: "eyJhbGciOiJIUzI1NiIs...",
        userID: "user123"
    )
    print("Session saved successfully")
} catch {
    print("Failed to save session: \(error)")
}

// トークン取得
do {
    let accessToken = try authManager.loadAccessToken()
    print("Access token: \(accessToken)")
} catch {
    print("Failed to load access token: \(error)")
}

// ログアウト
do {
    try authManager.logout()
    print("Logged out successfully")
} catch {
    print("Failed to logout: \(error)")
}
```

### 14.2.3 生体認証付きKeychain

Face IDやTouch IDを使った生体認証を必要とするKeychainアイテムの実装です。

```swift
import LocalAuthentication

class BiometricKeychainManager {
    static let shared = BiometricKeychainManager()

    private init() {}

    // MARK: - Biometric Authentication
    func canUseBiometrics() -> Bool {
        let context = LAContext()
        var error: NSError?
        return context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error)
    }

    func biometricType() -> LABiometryType {
        let context = LAContext()
        _ = context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: nil)
        return context.biometryType
    }

    // MARK: - Save with Biometric Protection
    func savewithBiometric(
        _ data: Data,
        service: String,
        account: String,
        reason: String = "Authenticate to save data"
    ) async throws {
        // 生体認証を実行
        try await authenticate(reason: reason)

        // アクセスコントロールを作成
        guard let access = SecAccessControlCreateWithFlags(
            kCFAllocatorDefault,
            kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
            .biometryAny,
            nil
        ) else {
            throw BiometricKeychainError.accessControlCreationFailed
        }

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecValueData as String: data,
            kSecAttrAccessControl as String: access
        ]

        // 既存のアイテムを削除
        SecItemDelete(query as CFDictionary)

        // 新しいアイテムを追加
        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw BiometricKeychainError.saveFailed(status)
        }
    }

    func saveWithBiometric(
        _ string: String,
        service: String,
        account: String,
        reason: String = "Authenticate to save data"
    ) async throws {
        guard let data = string.data(using: .utf8) else {
            throw BiometricKeychainError.stringConversionFailed
        }
        try await savewithBiometric(data, service: service, account: account, reason: reason)
    }

    // MARK: - Load with Biometric Protection
    func loadWithBiometric(
        service: String,
        account: String,
        reason: String = "Authenticate to access data"
    ) async throws -> Data {
        let context = LAContext()
        context.localizedReason = reason

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne,
            kSecUseAuthenticationContext as String: context
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess else {
            if status == errSecItemNotFound {
                throw BiometricKeychainError.itemNotFound
            }
            throw BiometricKeychainError.loadFailed(status)
        }

        guard let data = result as? Data else {
            throw BiometricKeychainError.dataConversionFailed
        }

        return data
    }

    func loadStringWithBiometric(
        service: String,
        account: String,
        reason: String = "Authenticate to access data"
    ) async throws -> String {
        let data = try await loadWithBiometric(service: service, account: account, reason: reason)
        guard let string = String(data: data, encoding: .utf8) else {
            throw BiometricKeychainError.stringConversionFailed
        }
        return string
    }

    // MARK: - Authentication Helper
    private func authenticate(reason: String) async throws {
        let context = LAContext()

        guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: nil) else {
            throw BiometricKeychainError.biometricsNotAvailable
        }

        do {
            let success = try await context.evaluatePolicy(
                .deviceOwnerAuthenticationWithBiometrics,
                localizedReason: reason
            )

            guard success else {
                throw BiometricKeychainError.authenticationFailed
            }
        } catch {
            throw BiometricKeychainError.authenticationFailed
        }
    }
}

// MARK: - Biometric Keychain Errors
enum BiometricKeychainError: Error, LocalizedError {
    case biometricsNotAvailable
    case authenticationFailed
    case accessControlCreationFailed
    case saveFailed(OSStatus)
    case loadFailed(OSStatus)
    case itemNotFound
    case stringConversionFailed
    case dataConversionFailed

    var errorDescription: String? {
        switch self {
        case .biometricsNotAvailable:
            return "Biometric authentication is not available"
        case .authenticationFailed:
            return "Biometric authentication failed"
        case .accessControlCreationFailed:
            return "Failed to create access control"
        case .saveFailed(let status):
            return "Failed to save to keychain: \(status)"
        case .loadFailed(let status):
            return "Failed to load from keychain: \(status)"
        case .itemNotFound:
            return "Item not found in keychain"
        case .stringConversionFailed:
            return "String conversion failed"
        case .dataConversionFailed:
            return "Data conversion failed"
        }
    }
}

// 使用例
Task {
    let manager = BiometricKeychainManager.shared

    // 生体認証の可用性確認
    if manager.canUseBiometrics() {
        let biometryType = manager.biometricType()
        print("Biometry type: \(biometryType)")

        // 機密データを保存
        do {
            try await manager.saveWithBiometric(
                "SecretPassword123",
                service: "com.example.app",
                account: "masterPassword",
                reason: "Save your master password"
            )
            print("Password saved with biometric protection")

            // データを読み込み（生体認証が必要）
            let password = try await manager.loadStringWithBiometric(
                service: "com.example.app",
                account: "masterPassword",
                reason: "Access your master password"
            )
            print("Password loaded: \(password)")
        } catch {
            print("Error: \(error)")
        }
    }
}
```

## 14.3 App Groups対応

複数のアプリや拡張機能でKeychainを共有する方法です。

```swift
class SharedKeychainManager {
    static let shared = SharedKeychainManager()

    private let keychain = KeychainManager.shared
    // App Groupsのアクセスグループを指定
    private let accessGroup = "group.com.example.myapp"

    private init() {}

    // MARK: - Save with Access Group
    func save(_ data: Data, service: String, account: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecValueData as String: data,
            kSecAttrAccessGroup as String: accessGroup
        ]

        SecItemDelete(query as CFDictionary)

        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw KeychainError.saveFailed(status)
        }
    }

    // MARK: - Load with Access Group
    func load(service: String, account: String) throws -> Data {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne,
            kSecAttrAccessGroup as String: accessGroup
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess else {
            if status == errSecItemNotFound {
                throw KeychainError.itemNotFound
            }
            throw KeychainError.loadFailed(status)
        }

        guard let data = result as? Data else {
            throw KeychainError.dataConversionFailed
        }

        return data
    }

    // MARK: - Delete with Access Group
    func delete(service: String, account: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecAttrAccessGroup as String: accessGroup
        ]

        let status = SecItemDelete(query as CFDictionary)

        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.deleteFailed(status)
        }
    }
}
```

## 14.4 テストとデバッグ

### 14.4.1 ユニットテスト

```swift
import XCTest

class UserDefaultsTests: XCTestCase {
    var suiteName: String!
    var defaults: UserDefaults!

    override func setUp() {
        super.setUp()
        suiteName = "TestDefaults"
        defaults = UserDefaults(suiteName: suiteName)
    }

    override func tearDown() {
        defaults.removePersistentDomain(forName: suiteName)
        defaults = nil
        suiteName = nil
        super.tearDown()
    }

    func testSaveAndLoadString() {
        // Given
        let key = "testString"
        let value = "Hello, World!"

        // When
        defaults.set(value, forKey: key)
        let loaded = defaults.string(forKey: key)

        // Then
        XCTAssertEqual(loaded, value)
    }

    func testSaveAndLoadBool() {
        // Given
        let key = "testBool"
        let value = true

        // When
        defaults.set(value, forKey: key)
        let loaded = defaults.bool(forKey: key)

        // Then
        XCTAssertEqual(loaded, value)
    }

    func testRemoveValue() {
        // Given
        let key = "testRemove"
        defaults.set("value", forKey: key)

        // When
        defaults.removeObject(forKey: key)
        let loaded = defaults.string(forKey: key)

        // Then
        XCTAssertNil(loaded)
    }
}

class KeychainTests: XCTestCase {
    var keychain: KeychainManager!
    var service: String!

    override func setUp() {
        super.setUp()
        keychain = KeychainManager.shared
        service = "TestService"
    }

    override func tearDown() {
        try? keychain.deleteAll(service: service)
        keychain = nil
        service = nil
        super.tearDown()
    }

    func testSaveAndLoadString() throws {
        // Given
        let account = "testAccount"
        let value = "SecretPassword"

        // When
        try keychain.save(value, service: service, account: account)
        let loaded = try keychain.loadString(service: service, account: account)

        // Then
        XCTAssertEqual(loaded, value)
    }

    func testUpdateString() throws {
        // Given
        let account = "testAccount"
        let initialValue = "InitialPassword"
        let updatedValue = "UpdatedPassword"

        // When
        try keychain.save(initialValue, service: service, account: account)
        try keychain.update(updatedValue, service: service, account: account)
        let loaded = try keychain.loadString(service: service, account: account)

        // Then
        XCTAssertEqual(loaded, updatedValue)
    }

    func testDeleteString() throws {
        // Given
        let account = "testAccount"
        try keychain.save("value", service: service, account: account)

        // When
        try keychain.delete(service: service, account: account)

        // Then
        XCTAssertThrowsError(try keychain.loadString(service: service, account: account)) { error in
            XCTAssertTrue(error is KeychainError)
        }
    }
}
```

## 14.5 ベストプラクティス

### 14.5.1 設計原則

```swift
// 良い例：責任の分離
class SettingsManager {
    private let userDefaults: UserDefaults
    private let keychain: KeychainManager

    init(userDefaults: UserDefaults = .standard, keychain: KeychainManager = .shared) {
        self.userDefaults = userDefaults
        self.keychain = keychain
    }

    // 設定値はUserDefaults
    func saveSetting(_ value: String, forKey key: String) {
        userDefaults.set(value, forKey: key)
    }

    // 機密情報はKeychain
    func saveCredential(_ value: String, forKey key: String) throws {
        try keychain.save(value, service: "com.example.app", account: key)
    }
}

// 悪い例：責任が混在
class BadSettingsManager {
    func save(_ value: String, forKey key: String, isSecret: Bool) {
        if isSecret {
            // Keychainに保存
        } else {
            // UserDefaultsに保存
        }
    }
}
```

### 14.5.2 エラーハンドリング

```swift
class RobustSettingsManager {
    func loadAccessToken() -> Result<String, SettingsError> {
        do {
            let token = try AuthenticationManager.shared.loadAccessToken()
            return .success(token)
        } catch KeychainError.itemNotFound {
            return .failure(.tokenNotFound)
        } catch {
            return .failure(.keychainError(error))
        }
    }

    enum SettingsError: Error, LocalizedError {
        case tokenNotFound
        case keychainError(Error)

        var errorDescription: String? {
            switch self {
            case .tokenNotFound:
                return "Access token not found"
            case .keychainError(let error):
                return "Keychain error: \(error.localizedDescription)"
            }
        }
    }
}
```

### 14.5.3 セキュリティのベストプラクティス

```swift
class SecureStorageManager {
    private let keychain = KeychainManager.shared
    private let service = Bundle.main.bundleIdentifier ?? "com.example.app"

    // 良い例：機密情報を適切に保護
    func savePassword(_ password: String) throws {
        // パスワードはKeychainに保存
        try keychain.save(password, service: service, account: "userPassword")
    }

    // 良い例：トークンをKeychainに保存
    func saveAccessToken(_ token: String) throws {
        try keychain.save(token, service: service, account: "accessToken")
    }

    // 悪い例：機密情報をUserDefaultsに保存（絶対にやってはいけない）
    func badSavePassword(_ password: String) {
        UserDefaults.standard.set(password, forKey: "password") // NG!
    }

    // 良い例：設定値はUserDefaultsに保存
    func saveThemePreference(_ theme: String) {
        UserDefaults.standard.set(theme, forKey: "selectedTheme")
    }
}
```

### 14.5.4 データのライフサイクル管理

```swift
class DataLifecycleManager {
    private let userDefaults = UserDefaults.standard
    private let keychain = KeychainManager.shared
    private let service = Bundle.main.bundleIdentifier ?? "com.example.app"

    // アプリ削除時にもデータを保持したい場合
    func savePersistedCredential(_ credential: String) throws {
        try keychain.save(
            credential,
            service: service,
            account: "persistedCredential"
        )
    }

    // アプリ削除時にデータを削除したい場合
    func saveTemporaryToken(_ token: String) throws {
        // kSecAttrAccessibleWhenUnlockedThisDeviceOnlyを使用
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: "temporaryToken",
            kSecValueData as String: token.data(using: .utf8)!,
            kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlockedThisDeviceOnly
        ]

        SecItemDelete(query as CFDictionary)
        SecItemAdd(query as CFDictionary, nil)
    }

    // ログアウト時の適切なクリーンアップ
    func logout() throws {
        // セッション情報を削除
        try? keychain.delete(service: service, account: "accessToken")
        try? keychain.delete(service: service, account: "refreshToken")

        // ユーザー設定は保持
        // userDefaults.removeObject(forKey: "theme") // 削除しない
    }

    // アカウント削除時の完全なクリーンアップ
    func deleteAccount() throws {
        // すべてのKeychainデータを削除
        try keychain.deleteAll(service: service)

        // すべてのUserDefaultsデータを削除
        if let bundleIdentifier = Bundle.main.bundleIdentifier {
            userDefaults.removePersistentDomain(forName: bundleIdentifier)
        }
    }
}
```

## 14.6 まとめ

この章では、Keychainを使った機密情報の安全な保存方法について学びました。

### 重要なポイント

1. **Keychainの基本**
   - 機密情報の安全な保存
   - 基本的なCRUD操作
   - エラーハンドリング

2. **認証情報管理**
   - トークン管理
   - セッション管理
   - ログアウト処理

3. **生体認証**
   - Face ID/Touch IDの統合
   - セキュアなデータアクセス

4. **App Groups対応**
   - 複数アプリ間でのデータ共有
   - アクセスグループの設定

5. **ベストプラクティス**
   - 適切な使い分け（UserDefaults vs Keychain）
   - セキュリティの確保
   - データライフサイクル管理
   - 包括的なエラーハンドリング

### データストレージの選択指針

| データの種類 | 推奨ストレージ | 理由 |
|------------|--------------|------|
| パスワード | Keychain | 暗号化が必要 |
| アクセストークン | Keychain | 暗号化が必要 |
| APIキー | Keychain | 暗号化が必要 |
| ユーザー設定 | UserDefaults | 非機密情報 |
| テーマ設定 | UserDefaults | 非機密情報 |
| キャッシュ | UserDefaults/ファイル | データサイズによる |
| 大きなファイル | ファイルシステム | サイズが大きい |

次の章では、より高度なデータ永続化手段であるCore Dataについて詳しく学びます。
