---
title: "データ暗号化"
---

# データ暗号化

機密データを安全に保存・送信するには、適切な暗号化が不可欠です。本章では、CryptoKitを使ったデータ暗号化の実装方法を解説します。

## CryptoKitの概要

CryptoKitは、iOS 13以降で利用可能なApple純正の暗号化フレームワークです。

**主な機能:**
- 対称鍵暗号化（AES-GCM）
- 非対称鍵暗号化（楕円曲線暗号）
- ハッシュ関数（SHA-256、SHA-384、SHA-512）
- メッセージ認証コード（HMAC）
- 鍵導出関数（HKDF、PBKDF2）

## 対称鍵暗号化（AES-GCM）

同じ鍵で暗号化と復号化を行います。

```swift
// Core/Security/EncryptionManager.swift
import CryptoKit
import Foundation

final class EncryptionManager {
    static let shared = EncryptionManager()

    private init() {}

    // MARK: - Symmetric Encryption

    func encrypt(_ data: Data, using key: SymmetricKey) throws -> Data {
        let sealedBox = try AES.GCM.seal(data, using: key)

        guard let combined = sealedBox.combined else {
            throw EncryptionError.encryptionFailed
        }

        return combined
    }

    func decrypt(_ data: Data, using key: SymmetricKey) throws -> Data {
        let sealedBox = try AES.GCM.SealedBox(combined: data)
        return try AES.GCM.open(sealedBox, using: key)
    }

    // MARK: - Key Generation

    func generateKey() -> SymmetricKey {
        SymmetricKey(size: .bits256)
    }

    func deriveKey(from password: String, salt: Data) -> SymmetricKey {
        let passwordData = Data(password.utf8)

        return HKDF<SHA256>.deriveKey(
            inputKeyMaterial: SymmetricKey(data: passwordData),
            salt: salt,
            outputByteCount: 32
        )
    }

    // MARK: - Helper Methods

    func generateSalt() -> Data {
        var bytes = [UInt8](repeating: 0, count: 32)
        _ = SecRandomCopyBytes(kSecRandomDefault, bytes.count, &bytes)
        return Data(bytes)
    }
}

enum EncryptionError: Error {
    case encryptionFailed
    case decryptionFailed
    case invalidKey
}
```

### 使用例

```swift
let encryption = EncryptionManager.shared

// 鍵の生成
let key = encryption.generateKey()

// データの暗号化
let plainText = "機密情報".data(using: .utf8)!
let encrypted = try encryption.encrypt(plainText, using: key)

// データの復号化
let decrypted = try encryption.decrypt(encrypted, using: key)
let decryptedText = String(data: decrypted, encoding: .utf8)!
print("Decrypted: \(decryptedText)")
```

## 鍵管理

暗号化鍵をKeychainで安全に管理します。

```swift
extension EncryptionManager {
    func saveKey(_ key: SymmetricKey, withIdentifier identifier: String) throws {
        let keyData = key.withUnsafeBytes { Data($0) }
        try KeychainManager.shared.save(keyData, forKey: identifier)
    }

    func loadKey(withIdentifier identifier: String) throws -> SymmetricKey {
        let keyData = try KeychainManager.shared.load(forKey: identifier)
        return SymmetricKey(data: keyData)
    }

    func deleteKey(withIdentifier identifier: String) throws {
        try KeychainManager.shared.delete(forKey: identifier)
    }
}
```

## ファイル暗号化

大きなファイルを暗号化して保存します。

```swift
final class SecureFileManager {
    private let encryption = EncryptionManager.shared
    private let fileManager = FileManager.default

    private var documentsDirectory: URL {
        fileManager.urls(for: .documentDirectory, in: .userDomainMask)[0]
    }

    // MARK: - Save Encrypted File

    func saveEncrypted(_ data: Data, filename: String, key: SymmetricKey) throws {
        let encrypted = try encryption.encrypt(data, using: key)
        let url = documentsDirectory.appendingPathComponent(filename)

        try encrypted.write(to: url, options: [.completeFileProtection, .atomic])
    }

    // MARK: - Load Encrypted File

    func loadEncrypted(filename: String, key: SymmetricKey) throws -> Data {
        let url = documentsDirectory.appendingPathComponent(filename)
        let encrypted = try Data(contentsOf: url)

        return try encryption.decrypt(encrypted, using: key)
    }

    // MARK: - Stream Encryption

    func encryptLargeFile(at sourceURL: URL, to destinationURL: URL, key: SymmetricKey) throws {
        let inputStream = InputStream(url: sourceURL)!
        let outputStream = OutputStream(url: destinationURL, append: false)!

        inputStream.open()
        outputStream.open()

        defer {
            inputStream.close()
            outputStream.close()
        }

        let bufferSize = 1024 * 1024 // 1MB
        var buffer = [UInt8](repeating: 0, count: bufferSize)

        while inputStream.hasBytesAvailable {
            let bytesRead = inputStream.read(&buffer, maxLength: bufferSize)

            if bytesRead > 0 {
                let data = Data(buffer[0..<bytesRead])
                let encrypted = try encryption.encrypt(data, using: key)

                encrypted.withUnsafeBytes { ptr in
                    outputStream.write(ptr.bindMemory(to: UInt8.self).baseAddress!, maxLength: encrypted.count)
                }
            }
        }
    }
}
```

## パスワードベースの暗号化

ユーザーのパスワードから鍵を導出します。

```swift
extension EncryptionManager {
    struct EncryptedData: Codable {
        let salt: Data
        let data: Data
    }

    func encryptWithPassword(_ data: Data, password: String) throws -> EncryptedData {
        let salt = generateSalt()
        let key = deriveKey(from: password, salt: salt)
        let encrypted = try encrypt(data, using: key)

        return EncryptedData(salt: salt, data: encrypted)
    }

    func decryptWithPassword(_ encryptedData: EncryptedData, password: String) throws -> Data {
        let key = deriveKey(from: password, salt: encryptedData.salt)
        return try decrypt(encryptedData.data, using: key)
    }
}

// 使用例
let manager = EncryptionManager.shared

// パスワードで暗号化
let plainData = "Secret message".data(using: .utf8)!
let encrypted = try manager.encryptWithPassword(plainData, password: "userPassword123")

// パスワードで復号化
let decrypted = try manager.decryptWithPassword(encrypted, password: "userPassword123")
```

## 非対称鍵暗号化

公開鍵と秘密鍵のペアを使った暗号化です。

```swift
import CryptoKit

final class AsymmetricEncryption {
    // MARK: - Key Pair Generation

    func generateKeyPair() -> (privateKey: Curve25519.KeyAgreement.PrivateKey, publicKey: Curve25519.KeyAgreement.PublicKey) {
        let privateKey = Curve25519.KeyAgreement.PrivateKey()
        let publicKey = privateKey.publicKey
        return (privateKey, publicKey)
    }

    // MARK: - Encryption with Public Key

    func encrypt(_ data: Data, publicKey: Curve25519.KeyAgreement.PublicKey) throws -> Data {
        // 一時的な鍵ペアを生成
        let ephemeralKey = Curve25519.KeyAgreement.PrivateKey()

        // 共有秘密の生成
        let sharedSecret = try ephemeralKey.sharedSecretFromKeyAgreement(with: publicKey)

        // 共有秘密から対称鍵を導出
        let symmetricKey = sharedSecret.hkdfDerivedSymmetricKey(
            using: SHA256.self,
            salt: Data(),
            sharedInfo: Data(),
            outputByteCount: 32
        )

        // データを暗号化
        let sealedBox = try AES.GCM.seal(data, using: symmetricKey)

        guard let combined = sealedBox.combined else {
            throw EncryptionError.encryptionFailed
        }

        // 一時公開鍵と暗号化データを結合
        return ephemeralKey.publicKey.rawRepresentation + combined
    }

    // MARK: - Decryption with Private Key

    func decrypt(_ data: Data, privateKey: Curve25519.KeyAgreement.PrivateKey) throws -> Data {
        // 一時公開鍵を抽出
        let ephemeralKeyData = data.prefix(32)
        let encryptedData = data.dropFirst(32)

        let ephemeralPublicKey = try Curve25519.KeyAgreement.PublicKey(rawRepresentation: ephemeralKeyData)

        // 共有秘密の生成
        let sharedSecret = try privateKey.sharedSecretFromKeyAgreement(with: ephemeralPublicKey)

        // 対称鍵の導出
        let symmetricKey = sharedSecret.hkdfDerivedSymmetricKey(
            using: SHA256.self,
            salt: Data(),
            sharedInfo: Data(),
            outputByteCount: 32
        )

        // データの復号化
        let sealedBox = try AES.GCM.SealedBox(combined: encryptedData)
        return try AES.GCM.open(sealedBox, using: symmetricKey)
    }
}
```

## ハッシュ関数

データの整合性確認やパスワードの保存に使用します。

```swift
extension EncryptionManager {
    func hash(_ data: Data) -> String {
        let digest = SHA256.hash(data: data)
        return digest.compactMap { String(format: "%02x", $0) }.joined()
    }

    func verifyHash(_ data: Data, expectedHash: String) -> Bool {
        hash(data) == expectedHash
    }

    // MARK: - Password Hashing

    func hashPassword(_ password: String, salt: Data) -> Data {
        let passwordData = Data(password.utf8)

        let hash = HKDF<SHA256>.deriveKey(
            inputKeyMaterial: SymmetricKey(data: passwordData),
            salt: salt,
            outputByteCount: 64
        )

        return hash.withUnsafeBytes { Data($0) }
    }

    func verifyPassword(_ password: String, hash: Data, salt: Data) -> Bool {
        let computedHash = hashPassword(password, salt: salt)
        return computedHash == hash
    }
}
```

## データベース暗号化

Core Dataと暗号化を組み合わせます。

```swift
extension NSManagedObject {
    func setEncryptedValue(_ value: String, forKey key: String, using encryption: EncryptionManager) throws {
        guard let keyData = try? encryption.loadKey(withIdentifier: "dbEncryptionKey") else {
            let newKey = encryption.generateKey()
            try encryption.saveKey(newKey, withIdentifier: "dbEncryptionKey")
            return try setEncryptedValue(value, forKey: key, using: encryption)
        }

        let plainData = value.data(using: .utf8)!
        let encrypted = try encryption.encrypt(plainData, using: keyData)

        self.setValue(encrypted, forKey: key)
    }

    func getEncryptedValue(forKey key: String, using encryption: EncryptionManager) throws -> String {
        guard let encryptedData = self.value(forKey: key) as? Data else {
            throw EncryptionError.decryptionFailed
        }

        let keyData = try encryption.loadKey(withIdentifier: "dbEncryptionKey")
        let decrypted = try encryption.decrypt(encryptedData, using: keyData)

        guard let value = String(data: decrypted, encoding: .utf8) else {
            throw EncryptionError.decryptionFailed
        }

        return value
    }
}
```

## まとめ

本章では、CryptoKitを使ったデータ暗号化の実装方法を解説しました。適切な暗号化により、以下の効果が期待されます：

- データの機密性保護
- 不正アクセスの防止
- コンプライアンス要件への対応

次章では、脱獄検知の実装について解説します。
