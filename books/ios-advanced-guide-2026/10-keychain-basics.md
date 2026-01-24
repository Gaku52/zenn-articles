---
title: "Keychain基礎"
---

# Keychain基礎

Keychainは、iOSでパスワードやトークンなどの機密情報を安全に保存するための仕組みです。本章では、Keychainの基本的な使い方と実装方法を解説します。

## Keychainとは

Keychainは、暗号化されたデータベースで、以下の特徴があります：

- ハードウェアレベルで暗号化
- アプリ間でのデータ共有が可能（App Groups使用時）
- デバイスロック時のアクセス制御
- iCloud経由でのバックアップ・同期（オプション）

### UserDefaultsとの違い

```plaintext
UserDefaults:
- 平文で保存
- 設定値など機密でない情報向け
- 読み書きが高速

Keychain:
- 暗号化して保存
- パスワード、トークンなど機密情報向け
- 読み書きがやや遅い
```

## Keychain の基本操作

### KeychainManager の実装

```swift
// Core/Security/KeychainManager.swift
import Security
import Foundation

final class KeychainManager {
    static let shared = KeychainManager()

    private init() {}

    // MARK: - Save

    func save(_ data: Data, forKey key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data
        ]

        // 既存のアイテムを削除
        SecItemDelete(query as CFDictionary)

        // 新しいアイテムを追加
        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw KeychainError.unableToSave(status)
        }
    }

    // MARK: - Load

    func load(forKey key: String) throws -> Data {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess else {
            throw KeychainError.itemNotFound(status)
        }

        guard let data = result as? Data else {
            throw KeychainError.unexpectedData
        }

        return data
    }

    // MARK: - Update

    func update(_ data: Data, forKey key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key
        ]

        let attributes: [String: Any] = [
            kSecValueData as String: data
        ]

        let status = SecItemUpdate(query as CFDictionary, attributes as CFDictionary)

        guard status == errSecSuccess else {
            throw KeychainError.unableToUpdate(status)
        }
    }

    // MARK: - Delete

    func delete(forKey key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key
        ]

        let status = SecItemDelete(query as CFDictionary)

        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.unableToDelete(status)
        }
    }

    // MARK: - Check Existence

    func exists(forKey key: String) -> Bool {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: false
        ]

        let status = SecItemCopyMatching(query as CFDictionary, nil)
        return status == errSecSuccess
    }

    // MARK: - Delete All

    func deleteAll() throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword
        ]

        let status = SecItemDelete(query as CFDictionary)

        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.unableToDelete(status)
        }
    }
}

// MARK: - Errors

enum KeychainError: Error {
    case unableToSave(OSStatus)
    case itemNotFound(OSStatus)
    case unableToUpdate(OSStatus)
    case unableToDelete(OSStatus)
    case unexpectedData

    var localizedDescription: String {
        switch self {
        case .unableToSave(let status):
            return "Keychainへの保存に失敗しました (Status: \(status))"
        case .itemNotFound(let status):
            return "アイテムが見つかりません (Status: \(status))"
        case .unableToUpdate(let status):
            return "更新に失敗しました (Status: \(status))"
        case .unableToDelete(let status):
            return "削除に失敗しました (Status: \(status))"
        case .unexpectedData:
            return "予期しないデータ形式です"
        }
    }
}
```

## 便利なExtension

String、Codableオブジェクトの保存を簡単にします。

```swift
// Core/Security/KeychainManager+Extensions.swift
extension KeychainManager {
    // MARK: - String

    func save(_ string: String, forKey key: String) throws {
        guard let data = string.data(using: .utf8) else {
            throw KeychainError.unexpectedData
        }
        try save(data, forKey: key)
    }

    func loadString(forKey key: String) throws -> String {
        let data = try load(forKey: key)
        guard let string = String(data: data, encoding: .utf8) else {
            throw KeychainError.unexpectedData
        }
        return string
    }

    // MARK: - Codable

    func save<T: Codable>(_ object: T, forKey key: String) throws {
        let data = try JSONEncoder().encode(object)
        try save(data, forKey: key)
    }

    func load<T: Codable>(forKey key: String, as type: T.Type) throws -> T {
        let data = try load(forKey: key)
        return try JSONDecoder().decode(type, from: data)
    }

    // MARK: - Token Management

    func saveToken(_ token: String) throws {
        try save(token, forKey: "authToken")
    }

    func loadToken() throws -> String {
        try loadString(forKey: "authToken")
    }

    func deleteToken() throws {
        try delete(forKey: "authToken")
    }
}
```

## 使用例

### トークンの保存と読み込み

```swift
// トークンを保存
do {
    try KeychainManager.shared.saveToken("your_access_token")
    print("Token saved successfully")
} catch {
    print("Failed to save token: \(error)")
}

// トークンを読み込み
do {
    let token = try KeychainManager.shared.loadToken()
    print("Token: \(token)")
} catch {
    print("Failed to load token: \(error)")
}

// トークンを削除
do {
    try KeychainManager.shared.deleteToken()
    print("Token deleted successfully")
} catch {
    print("Failed to delete token: \(error)")
}
```

### Codableオブジェクトの保存

```swift
struct Credentials: Codable {
    let username: String
    let password: String
}

// 保存
let credentials = Credentials(username: "user@example.com", password: "secret123")
try? KeychainManager.shared.save(credentials, forKey: "userCredentials")

// 読み込み
if let saved = try? KeychainManager.shared.load(forKey: "userCredentials", as: Credentials.self) {
    print("Username: \(saved.username)")
}
```

## Keychain Accessibility

データへのアクセス制御を設定します。

```swift
extension KeychainManager {
    enum Accessibility {
        case whenUnlocked
        case afterFirstUnlock
        case always
        case whenPasscodeSetThisDeviceOnly
        case whenUnlockedThisDeviceOnly
        case afterFirstUnlockThisDeviceOnly
        case alwaysThisDeviceOnly

        var rawValue: CFString {
            switch self {
            case .whenUnlocked:
                return kSecAttrAccessibleWhenUnlocked
            case .afterFirstUnlock:
                return kSecAttrAccessibleAfterFirstUnlock
            case .always:
                return kSecAttrAccessibleAlways
            case .whenPasscodeSetThisDeviceOnly:
                return kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly
            case .whenUnlockedThisDeviceOnly:
                return kSecAttrAccessibleWhenUnlockedThisDeviceOnly
            case .afterFirstUnlockThisDeviceOnly:
                return kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly
            case .alwaysThisDeviceOnly:
                return kSecAttrAccessibleAlwaysThisDeviceOnly
            }
        }
    }

    func save(
        _ data: Data,
        forKey key: String,
        accessibility: Accessibility = .whenUnlockedThisDeviceOnly
    ) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessible as String: accessibility.rawValue
        ]

        SecItemDelete(query as CFDictionary)

        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw KeychainError.unableToSave(status)
        }
    }
}
```

### Accessibilityの選択指針

```plaintext
whenUnlockedThisDeviceOnly（推奨）:
- デバイスロック解除時のみアクセス可能
- iCloudバックアップに含まれない
- 最も安全

afterFirstUnlockThisDeviceOnly:
- 起動後最初のロック解除後アクセス可能
- バックグラウンド処理で使用

whenPasscodeSetThisDeviceOnly:
- パスコード設定時のみ保存可能
- より高いセキュリティが必要な場合
```

## テストの実装

Keychainの動作を確認するテストです。

```swift
// Tests/KeychainManagerTests.swift
import XCTest
@testable import MyAwesomeApp

final class KeychainManagerTests: XCTestCase {
    let keychain = KeychainManager.shared
    let testKey = "test_key"

    override func tearDown() {
        try? keychain.delete(forKey: testKey)
        super.tearDown()
    }

    func testSaveAndLoad() throws {
        let testData = "test_value"

        try keychain.save(testData, forKey: testKey)

        let loaded = try keychain.loadString(forKey: testKey)
        XCTAssertEqual(loaded, testData)
    }

    func testUpdate() throws {
        try keychain.save("original", forKey: testKey)

        let updatedData = "updated".data(using: .utf8)!
        try keychain.update(updatedData, forKey: testKey)

        let loaded = try keychain.loadString(forKey: testKey)
        XCTAssertEqual(loaded, "updated")
    }

    func testDelete() throws {
        try keychain.save("test", forKey: testKey)

        XCTAssertTrue(keychain.exists(forKey: testKey))

        try keychain.delete(forKey: testKey)

        XCTAssertFalse(keychain.exists(forKey: testKey))
    }

    func testCodable() throws {
        struct TestModel: Codable, Equatable {
            let name: String
            let age: Int
        }

        let original = TestModel(name: "Test", age: 30)

        try keychain.save(original, forKey: testKey)

        let loaded = try keychain.load(forKey: testKey, as: TestModel.self)
        XCTAssertEqual(loaded, original)
    }
}
```

## まとめ

本章では、Keychainの基本的な使い方と実装方法を解説しました。Keychainを適切に使用することで、以下の効果が期待されます：

- 機密情報の安全な保存
- ハードウェアレベルの暗号化
- デバイスロックとの連携
- アプリ削除後もデータが残らない

次章では、Keychainのより高度な活用方法について解説します。
