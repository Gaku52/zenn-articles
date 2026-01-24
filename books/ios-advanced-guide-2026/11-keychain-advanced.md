---
title: "Keychain応用"
---

# Keychain応用

本章では、Keychainのより高度な機能と、実務で役立つ応用テクニックを解説します。

## Biometric認証との統合

Face ID / Touch IDを使ってKeychainのデータを保護します。

```swift
// Core/Security/BiometricKeychain.swift
import LocalAuthentication

extension KeychainManager {
    func saveWithBiometric(_ data: Data, forKey key: String) throws {
        let context = LAContext()

        // ACL (Access Control List) の作成
        guard let accessControl = SecAccessControlCreateWithFlags(
            nil,
            kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
            .biometryCurrentSet,
            nil
        ) else {
            throw KeychainError.unableToSave(errSecInternalError)
        }

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessControl as String: accessControl,
            kSecUseAuthenticationContext as String: context
        ]

        SecItemDelete(query as CFDictionary)

        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw KeychainError.unableToSave(status)
        }
    }

    func loadWithBiometric(
        forKey key: String,
        reason: String = "認証が必要です"
    ) async throws -> Data {
        let context = LAContext()
        context.localizedReason = reason

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecUseAuthenticationContext as String: context
        ]

        return try await withCheckedThrowingContinuation { continuation in
            var result: AnyObject?
            let status = SecItemCopyMatching(query as CFDictionary, &result)

            if status == errSecSuccess, let data = result as? Data {
                continuation.resume(returning: data)
            } else {
                continuation.resume(throwing: KeychainError.itemNotFound(status))
            }
        }
    }
}
```

### 使用例

```swift
// 生体認証付きで保存
let sensitiveData = "Very secret information".data(using: .utf8)!
try? KeychainManager.shared.saveWithBiometric(sensitiveData, forKey: "sensitiveData")

// 生体認証して読み込み
Task {
    do {
        let data = try await KeychainManager.shared.loadWithBiometric(
            forKey: "sensitiveData",
            reason: "機密情報にアクセスします"
        )
        print("Data loaded successfully")
    } catch {
        print("Biometric authentication failed: \(error)")
    }
}
```

## App Groups でのKeychain共有

アプリとExtension間でKeychainを共有します。

```swift
extension KeychainManager {
    func save(
        _ data: Data,
        forKey key: String,
        accessGroup: String? = nil
    ) throws {
        var query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly
        ]

        // Access Groupの追加
        if let accessGroup = accessGroup {
            query[kSecAttrAccessGroup as String] = accessGroup
        }

        SecItemDelete(query as CFDictionary)

        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw KeychainError.unableToSave(status)
        }
    }

    func load(forKey key: String, accessGroup: String? = nil) throws -> Data {
        var query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]

        if let accessGroup = accessGroup {
            query[kSecAttrAccessGroup as String] = accessGroup
        }

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess,
              let data = result as? Data else {
            throw KeychainError.itemNotFound(status)
        }

        return data
    }
}
```

### 設定方法

```plaintext
1. Xcode > Target > Signing & Capabilities
2. + Capability > Keychain Sharing
3. Keychain Groups に追加:
   - $(AppIdentifierPrefix)com.company.myapp
```

### 使用例

```swift
let accessGroup = "group.com.company.myapp"

// アプリから保存
try? KeychainManager.shared.save(
    tokenData,
    forKey: "sharedToken",
    accessGroup: accessGroup
)

// Extensionから読み込み
let data = try? KeychainManager.shared.load(
    forKey: "sharedToken",
    accessGroup: accessGroup
)
```

## Keychain Synchronization

iCloud Keychainでデバイス間同期を有効にします。

```swift
extension KeychainManager {
    func saveSynchronizable(_ data: Data, forKey key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrSynchronizable as String: kCFBooleanTrue!,
            kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlock
        ]

        SecItemDelete(query as CFDictionary)

        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw KeychainError.unableToSave(status)
        }
    }

    func loadSynchronizable(forKey key: String) throws -> Data {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecAttrSynchronizable as String: kCFBooleanTrue!,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess,
              let data = result as? Data else {
            throw KeychainError.itemNotFound(status)
        }

        return data
    }
}
```

## 証明書の保存

クライアント証明書をKeychainに保存します。

```swift
extension KeychainManager {
    func saveCertificate(_ certificate: SecCertificate, label: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassCertificate,
            kSecValueRef as String: certificate,
            kSecAttrLabel as String: label
        ]

        SecItemDelete(query as CFDictionary)

        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw KeychainError.unableToSave(status)
        }
    }

    func loadCertificate(label: String) throws -> SecCertificate {
        let query: [String: Any] = [
            kSecClass as String: kSecClassCertificate,
            kSecAttrLabel as String: label,
            kSecReturnRef as String: true
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess,
              let certificate = result as! SecCertificate? else {
            throw KeychainError.itemNotFound(status)
        }

        return certificate
    }
}
```

## 秘密鍵の生成と保存

暗号化用の秘密鍵を生成してKeychainに保存します。

```swift
import CryptoKit

extension KeychainManager {
    func generateAndSavePrivateKey(tag: String) throws -> SecKey {
        // 既存の鍵を削除
        try? deleteKey(tag: tag)

        // 鍵生成パラメータ
        let attributes: [String: Any] = [
            kSecAttrKeyType as String: kSecAttrKeyTypeECSECPrimeRandom,
            kSecAttrKeySizeInBits as String: 256,
            kSecPrivateKeyAttrs as String: [
                kSecAttrIsPermanent as String: true,
                kSecAttrApplicationTag as String: tag.data(using: .utf8)!
            ]
        ]

        var error: Unmanaged<CFError>?
        guard let privateKey = SecKeyCreateRandomKey(attributes as CFDictionary, &error) else {
            throw error!.takeRetainedValue() as Error
        }

        return privateKey
    }

    func loadPrivateKey(tag: String) throws -> SecKey {
        let query: [String: Any] = [
            kSecClass as String: kSecClassKey,
            kSecAttrApplicationTag as String: tag.data(using: .utf8)!,
            kSecAttrKeyType as String: kSecAttrKeyTypeECSECPrimeRandom,
            kSecReturnRef as String: true
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess,
              let key = result as! SecKey? else {
            throw KeychainError.itemNotFound(status)
        }

        return key
    }

    func deleteKey(tag: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassKey,
            kSecAttrApplicationTag as String: tag.data(using: .utf8)!
        ]

        let status = SecItemDelete(query as CFDictionary)

        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.unableToDelete(status)
        }
    }
}
```

### 使用例：データの暗号化と復号化

```swift
// 秘密鍵の生成
let privateKey = try KeychainManager.shared.generateAndSavePrivateKey(tag: "com.app.privateKey")
let publicKey = SecKeyCopyPublicKey(privateKey)!

// データの暗号化（公開鍵で）
let plainText = "Secret message".data(using: .utf8)!
var error: Unmanaged<CFError>?

let encrypted = SecKeyCreateEncryptedData(
    publicKey,
    .eciesEncryptionStandardX963SHA256AESGCM,
    plainText as CFData,
    &error
)! as Data

// データの復号化（秘密鍵で）
let decrypted = SecKeyCreateDecryptedData(
    privateKey,
    .eciesEncryptionStandardX963SHA256AESGCM,
    encrypted as CFData,
    &error
)! as Data

let decryptedString = String(data: decrypted, encoding: .utf8)!
print("Decrypted: \(decryptedString)")
```

## マイグレーション

アプリの更新時にKeychainデータを移行します。

```swift
final class KeychainMigration {
    static func migrate() {
        let keychain = KeychainManager.shared
        let currentVersion = UserDefaults.standard.integer(forKey: "keychainVersion")

        switch currentVersion {
        case 0:
            migrateV0ToV1()
            UserDefaults.standard.set(1, forKey: "keychainVersion")
            fallthrough

        case 1:
            migrateV1ToV2()
            UserDefaults.standard.set(2, forKey: "keychainVersion")

        default:
            break
        }
    }

    private static func migrateV0ToV1() {
        // 旧形式から新形式へ移行
        if let oldToken = try? KeychainManager.shared.loadString(forKey: "token") {
            try? KeychainManager.shared.save(oldToken, forKey: "authToken")
            try? KeychainManager.shared.delete(forKey: "token")
        }
    }

    private static func migrateV1ToV2() {
        // セキュリティ強化のため、生体認証必須に変更
        if let data = try? KeychainManager.shared.load(forKey: "authToken") {
            try? KeychainManager.shared.saveWithBiometric(data, forKey: "authToken")
        }
    }
}

// AppDelegateまたはApp.swiftで実行
KeychainMigration.migrate()
```

## デバッグとトラブルシューティング

### Keychain内容の確認

```swift
func printAllKeychainItems() {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecReturnAttributes as String: true,
        kSecMatchLimit as String: kSecMatchLimitAll
    ]

    var result: AnyObject?
    let status = SecItemCopyMatching(query as CFDictionary, &result)

    if status == errSecSuccess, let items = result as? [[String: Any]] {
        for item in items {
            if let account = item[kSecAttrAccount as String] as? String {
                print("Keychain item: \(account)")
            }
        }
    }
}
```

### エラーコードのデバッグ

```swift
extension OSStatus {
    var errorMessage: String {
        switch self {
        case errSecSuccess:
            return "Success"
        case errSecItemNotFound:
            return "Item not found"
        case errSecDuplicateItem:
            return "Duplicate item"
        case errSecAuthFailed:
            return "Authentication failed"
        case errSecUserCanceled:
            return "User canceled"
        default:
            return "Error code: \(self)"
        }
    }
}
```

## まとめ

本章では、Keychainの高度な機能を解説しました。適切なKeychainの活用により、以下の効果が期待されます：

- 生体認証によるセキュリティ強化
- アプリ間でのデータ共有
- デバイス間でのデータ同期
- 暗号化キーの安全な管理

次章では、証明書ピンニングについて解説します。
