---
title: "Keychain: セキュアなデータ保存"
---

# Keychain: セキュアなデータ保存

Keychain は、パスワード、トークン、証明書などの機密情報を安全に保存するための iOS 標準の仕組みです。この章では、Keychain の適切な使用方法を学びます。

## Keychain の特徴

Keychain には以下の特徴があります:

- **暗号化**: データは自動的に暗号化される
- **永続性**: アプリの再インストール後もデータは残る（オプション）
- **アクセス制御**: Touch ID / Face ID による保護が可能
- **共有**: Keychain Sharing でアプリ間でデータを共有できる

## Keychain の適切な使用例

- **認証トークン**: アクセストークン、リフレッシュトークン
- **パスワード**: ユーザーのパスワード
- **秘密鍵**: 暗号化に使用する鍵
- **証明書**: SSL 証明書

## Keychain Manager の実装

基本的な Keychain 操作をラップするマネージャーを実装します:

```swift
import Security

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
            // アイテムが存在しない場合は新規作成
            if status == errSecItemNotFound {
                try save(data, service: service, account: account)
            } else {
                throw KeychainError.updateFailed(status)
            }
        }
    }
}

enum KeychainError: Error, LocalizedError {
    case saveFailed(OSStatus)
    case loadFailed(OSStatus)
    case deleteFailed(OSStatus)
    case updateFailed(OSStatus)
    case stringConversionFailed
    case dataConversionFailed

    var errorDescription: String? {
        switch self {
        case .saveFailed(let status):
            return "Keychain への保存に失敗しました: \(status)"
        case .loadFailed(let status):
            return "Keychain からの読み込みに失敗しました: \(status)"
        case .deleteFailed(let status):
            return "Keychain からの削除に失敗しました: \(status)"
        case .updateFailed(let status):
            return "Keychain の更新に失敗しました: \(status)"
        case .stringConversionFailed:
            return "文字列変換に失敗しました"
        case .dataConversionFailed:
            return "データ変換に失敗しました"
        }
    }
}
```

## トークンマネージャー

認証トークンを管理する専用のマネージャーを実装します:

```swift
class TokenManager {
    private let keychain = KeychainManager.shared
    private let service = "com.example.app"

    // アクセストークン
    func saveAccessToken(_ token: String) throws {
        try keychain.save(token, service: service, account: "accessToken")
    }

    func loadAccessToken() throws -> String {
        try keychain.loadString(service: service, account: "accessToken")
    }

    func deleteAccessToken() throws {
        try keychain.delete(service: service, account: "accessToken")
    }

    // リフレッシュトークン
    func saveRefreshToken(_ token: String) throws {
        try keychain.save(token, service: service, account: "refreshToken")
    }

    func loadRefreshToken() throws -> String {
        try keychain.loadString(service: service, account: "refreshToken")
    }

    // 全トークンの削除
    func deleteAllTokens() throws {
        try? deleteAccessToken()
        try? keychain.delete(service: service, account: "refreshToken")
    }

    // トークンの存在確認
    func hasAccessToken() -> Bool {
        (try? loadAccessToken()) != nil
    }
}
```

## アクセス制御の追加

Touch ID / Face ID による保護を追加します:

```swift
extension KeychainManager {
    func saveWithBiometrics(
        _ data: Data,
        service: String,
        account: String
    ) throws {
        let access = SecAccessControlCreateWithFlags(
            nil,
            kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
            .biometryCurrentSet,
            nil
        )

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecValueData as String: data,
            kSecAttrAccessControl as String: access as Any
        ]

        SecItemDelete(query as CFDictionary)

        let status = SecItemAdd(query as CFDictionary, nil)
        guard status == errSecSuccess else {
            throw KeychainError.saveFailed(status)
        }
    }

    func loadWithBiometrics(
        service: String,
        account: String,
        reason: String = "データにアクセスするために認証が必要です"
    ) async throws -> Data {
        let context = LAContext()
        var error: NSError?

        guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) else {
            throw KeychainError.biometricsNotAvailable
        }

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecReturnData as String: true,
            kSecUseOperationPrompt as String: reason
        ]

        return try await withCheckedThrowingContinuation { continuation in
            var result: AnyObject?
            let status = SecItemCopyMatching(query as CFDictionary, &result)

            if status == errSecSuccess, let data = result as? Data {
                continuation.resume(returning: data)
            } else {
                continuation.resume(throwing: KeychainError.loadFailed(status))
            }
        }
    }
}

extension KeychainError {
    static let biometricsNotAvailable = KeychainError.loadFailed(-1)
}
```

## Codable オブジェクトの保存

複雑な構造体を Keychain に保存します:

```swift
extension KeychainManager {
    func saveCodable<T: Codable>(
        _ object: T,
        service: String,
        account: String
    ) throws {
        let encoder = JSONEncoder()
        let data = try encoder.encode(object)
        try save(data, service: service, account: account)
    }

    func loadCodable<T: Codable>(
        service: String,
        account: String
    ) throws -> T {
        let data = try load(service: service, account: account)
        let decoder = JSONDecoder()
        return try decoder.decode(T.self, from: data)
    }
}

// 使用例
struct Credentials: Codable {
    let username: String
    let password: String
    let apiKey: String
}

class CredentialsManager {
    private let keychain = KeychainManager.shared
    private let service = "com.example.app"

    func saveCredentials(_ credentials: Credentials) throws {
        try keychain.saveCodable(
            credentials,
            service: service,
            account: "userCredentials"
        )
    }

    func loadCredentials() throws -> Credentials {
        try keychain.loadCodable(
            service: service,
            account: "userCredentials"
        )
    }

    func deleteCredentials() throws {
        try keychain.delete(service: service, account: "userCredentials")
    }
}
```

## Property Wrapper による簡潔な実装

Property Wrapper を使用して、Keychain へのアクセスを簡潔にします:

```swift
@propertyWrapper
struct KeychainStorage {
    let key: String
    let service: String

    var wrappedValue: String? {
        get {
            try? KeychainManager.shared.loadString(service: service, account: key)
        }
        set {
            if let value = newValue {
                try? KeychainManager.shared.save(value, service: service, account: key)
            } else {
                try? KeychainManager.shared.delete(service: service, account: key)
            }
        }
    }
}

// 使用例
class AuthManager {
    @KeychainStorage(key: "accessToken", service: "com.example.app")
    var accessToken: String?

    @KeychainStorage(key: "refreshToken", service: "com.example.app")
    var refreshToken: String?

    func login(username: String, password: String) async throws {
        // ログイン処理
        let token = "obtained-access-token"
        accessToken = token  // 自動的に Keychain に保存
    }

    func logout() {
        accessToken = nil  // 自動的に Keychain から削除
        refreshToken = nil
    }
}
```

## Keychain Sharing

複数のアプリ間で Keychain を共有します:

```swift
class SharedKeychainManager {
    private let keychain = KeychainManager.shared
    private let accessGroup = "group.com.example.shared"

    func saveToSharedKeychain(_ data: Data, account: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccessGroup as String: accessGroup,
            kSecAttrAccount as String: account,
            kSecValueData as String: data
        ]

        SecItemDelete(query as CFDictionary)

        let status = SecItemAdd(query as CFDictionary, nil)
        guard status == errSecSuccess else {
            throw KeychainError.saveFailed(status)
        }
    }

    func loadFromSharedKeychain(account: String) throws -> Data {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccessGroup as String: accessGroup,
            kSecAttrAccount as String: account,
            kSecReturnData as String: true
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess else {
            throw KeychainError.loadFailed(status)
        }

        guard let data = result as? Data else {
            throw KeychainError.dataConversionFailed
        }

        return data
    }
}
```

## セキュリティのベストプラクティス

### 1. アクセシビリティの設定

```swift
extension KeychainManager {
    func saveWithAccessibility(
        _ data: Data,
        service: String,
        account: String,
        accessibility: CFString = kSecAttrAccessibleWhenUnlocked
    ) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecValueData as String: data,
            kSecAttrAccessible as String: accessibility
        ]

        SecItemDelete(query as CFDictionary)

        let status = SecItemAdd(query as CFDictionary, nil)
        guard status == errSecSuccess else {
            throw KeychainError.saveFailed(status)
        }
    }
}

// アクセシビリティの種類:
// - kSecAttrAccessibleWhenUnlocked: デバイスのロック解除時のみアクセス可能（推奨）
// - kSecAttrAccessibleAfterFirstUnlock: 初回ロック解除後はアクセス可能
// - kSecAttrAccessibleAlways: 常にアクセス可能（非推奨）
// - kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly: パスコード設定時のみ、デバイス固有
```

### 2. データの自動削除

```swift
class SecureDataManager {
    private let keychain = KeychainManager.shared

    func saveTemporaryToken(_ token: String) throws {
        // 24時間後に自動削除されるトークンを保存
        try keychain.save(token, service: "com.example.app", account: "temporaryToken")

        // 24時間後に削除
        DispatchQueue.main.asyncAfter(deadline: .now() + 86400) { [weak self] in
            try? self?.keychain.delete(service: "com.example.app", account: "temporaryToken")
        }
    }
}
```

## まとめ

この章では、Keychain を使った安全なデータ保存方法を学びました:

- Keychain Manager の実装
- トークン管理
- 生体認証の統合
- Codable オブジェクトの保存
- Keychain Sharing
- セキュリティのベストプラクティス

次の章では、Core Data を使った構造化データの管理について学びます。

## 参考リソース

- [Keychain Services - Apple Developer](https://developer.apple.com/documentation/security/keychain_services)
- [Local Authentication - Apple Developer](https://developer.apple.com/documentation/localauthentication)
