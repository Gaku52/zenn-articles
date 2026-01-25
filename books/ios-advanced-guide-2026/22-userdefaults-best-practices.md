---
title: "UserDefaultsベストプラクティス"
---

# UserDefaultsベストプラクティス

UserDefaultsは軽量な設定値の保存に適した標準的なストレージ機構です。本章では、UserDefaultsの効果的な活用方法と、型安全性を確保した実装パターンを詳しく解説します。

## UserDefaultsの基本

UserDefaultsは、キーと値のペアでデータを保存するシンプルなストレージです。内部的にはplistファイルとして保存され、アプリケーション起動時にメモリに読み込まれます。

### 基本的な使い方

```swift
// 値の保存
UserDefaults.standard.set(true, forKey: "notificationsEnabled")
UserDefaults.standard.set("dark", forKey: "theme")
UserDefaults.standard.set(42, forKey: "fontSize")

// 値の取得
let notificationsEnabled = UserDefaults.standard.bool(forKey: "notificationsEnabled")
let theme = UserDefaults.standard.string(forKey: "theme")
let fontSize = UserDefaults.standard.integer(forKey: "fontSize")

// 値の削除
UserDefaults.standard.removeObject(forKey: "theme")

// 同期（通常は不要だが、即座にディスクに書き込みたい場合）
UserDefaults.standard.synchronize()
```

### サポートされるデータ型

UserDefaultsは以下のデータ型をネイティブにサポートしています：

- `Bool`
- `Int`, `Float`, `Double`
- `String`
- `Data`
- `Date`
- `Array`, `Dictionary`（要素が上記の型の場合）
- `URL`

## 型安全なアクセス - Property Wrapper

生の文字列キーを使うと、タイポによるバグや型の不一致が発生しやすくなります。Property Wrapperを使うことで、型安全性を確保できます。

### カスタムProperty Wrapper

```swift
@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T

    var wrappedValue: T {
        get {
            UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
        }
        set {
            UserDefaults.standard.set(newValue, forKey: key)
        }
    }
}

// 使用例
struct AppSettings {
    @UserDefault(key: "notificationsEnabled", defaultValue: true)
    static var notificationsEnabled: Bool

    @UserDefault(key: "theme", defaultValue: "light")
    static var theme: String

    @UserDefault(key: "fontSize", defaultValue: 16)
    static var fontSize: Int
}

// 型安全なアクセス
AppSettings.notificationsEnabled = false
print(AppSettings.theme) // "light"
```

### SwiftUIの@AppStorage

SwiftUIでは、`@AppStorage`を使うことで、UserDefaultsとViewを自動的にバインドできます。

```swift
struct SettingsView: View {
    @AppStorage("notificationsEnabled") private var notificationsEnabled = true
    @AppStorage("theme") private var theme = "light"
    @AppStorage("fontSize") private var fontSize = 16

    var body: some View {
        Form {
            Section("通知設定") {
                Toggle("通知を有効にする", isOn: $notificationsEnabled)
            }

            Section("表示設定") {
                Picker("テーマ", selection: $theme) {
                    Text("ライト").tag("light")
                    Text("ダーク").tag("dark")
                    Text("自動").tag("auto")
                }

                Stepper("フォントサイズ: \(fontSize)", value: $fontSize, in: 12...24)
            }
        }
    }
}
```

## Codableを使った複雑なデータの保存

カスタムオブジェクトを保存する場合は、Codableプロトコルを使って、JSONエンコード/デコードを行います。

### Codable拡張

```swift
extension UserDefaults {
    func setCodable<T: Codable>(_ value: T, forKey key: String) {
        guard let encoded = try? JSONEncoder().encode(value) else {
            print("Failed to encode \(T.self)")
            return
        }
        set(encoded, forKey: key)
    }

    func codable<T: Codable>(forKey key: String) -> T? {
        guard let data = data(forKey: key) else { return nil }
        return try? JSONDecoder().decode(T.self, from: data)
    }
}

// 使用例
struct UserPreferences: Codable {
    var language: String
    var region: String
    var currency: String
    var notifications: NotificationSettings
}

struct NotificationSettings: Codable {
    var email: Bool
    var push: Bool
    var sms: Bool
}

// 保存
let preferences = UserPreferences(
    language: "ja",
    region: "JP",
    currency: "JPY",
    notifications: NotificationSettings(email: true, push: true, sms: false)
)
UserDefaults.standard.setCodable(preferences, forKey: "userPreferences")

// 取得
if let savedPreferences: UserPreferences = UserDefaults.standard.codable(forKey: "userPreferences") {
    print(savedPreferences.language) // "ja"
}
```

### Property WrapperとCodableの組み合わせ

```swift
@propertyWrapper
struct CodableUserDefault<T: Codable> {
    let key: String
    let defaultValue: T

    var wrappedValue: T {
        get {
            guard let data = UserDefaults.standard.data(forKey: key),
                  let value = try? JSONDecoder().decode(T.self, from: data) else {
                return defaultValue
            }
            return value
        }
        set {
            if let encoded = try? JSONEncoder().encode(newValue) {
                UserDefaults.standard.set(encoded, forKey: key)
            }
        }
    }
}

struct AppSettings {
    @CodableUserDefault(
        key: "userPreferences",
        defaultValue: UserPreferences(
            language: "en",
            region: "US",
            currency: "USD",
            notifications: NotificationSettings(email: true, push: true, sms: false)
        )
    )
    static var preferences: UserPreferences
}
```

## App Groupsでのデータ共有

App Extensions（Widget、Share Extension、Today Extensionなど）とメインアプリでデータを共有する場合は、App Groupsを使用します。

### App Groupsの設定

1. Xcode > Targets > Signing & Capabilities
2. "+ Capability" から "App Groups" を追加
3. グループIDを作成（例: `group.com.example.myapp`）
4. Extensionターゲットにも同じApp Groupsを追加

### コード実装

```swift
extension UserDefaults {
    static let shared = UserDefaults(suiteName: "group.com.example.myapp")!
}

// メインアプリで保存
UserDefaults.shared.set(100, forKey: "widgetCounter")

// Widgetで取得
let counter = UserDefaults.shared.integer(forKey: "widgetCounter")
```

### App Groups対応のProperty Wrapper

```swift
@propertyWrapper
struct AppGroupUserDefault<T> {
    let key: String
    let defaultValue: T
    let suiteName: String

    private var userDefaults: UserDefaults {
        UserDefaults(suiteName: suiteName)!
    }

    var wrappedValue: T {
        get {
            userDefaults.object(forKey: key) as? T ?? defaultValue
        }
        set {
            userDefaults.set(newValue, forKey: key)
        }
    }
}

struct SharedSettings {
    @AppGroupUserDefault(
        key: "widgetCounter",
        defaultValue: 0,
        suiteName: "group.com.example.myapp"
    )
    static var widgetCounter: Int
}
```

## 機密情報を保存してはいけない理由

UserDefaultsは**暗号化されていない**ため、以下の情報を保存してはいけません：

- パスワード
- 認証トークン（OAuth Token、API Key）
- クレジットカード情報
- 個人を特定できる情報（PII）
- その他の機密データ

### 理由

1. **平文で保存される**: plistファイルとして平文で保存される
2. **iOSバックアップに含まれる**: iTunesやiCloudバックアップで抽出可能
3. **脱獄デバイスで容易にアクセス可能**: ファイルシステムから直接読み取れる
4. **アプリ間で共有される可能性**: App Groupsを使うと他のアプリからアクセス可能

### 正しい保存先

機密情報は**Keychain**に保存することが推奨されます（第16章参照）。

```swift
// ❌ 絶対にやってはいけない
UserDefaults.standard.set("my_secret_token", forKey: "authToken")

// ✅ 正しい方法
KeychainManager.shared.saveToken("my_secret_token")
```

## UserDefaultsのパフォーマンス最適化

UserDefaultsは起動時に全データをメモリに読み込むため、大量のデータを保存するとアプリの起動時間に影響します。

### パフォーマンス最適化のベストプラクティス

#### 1. データサイズを小さく保つ

```swift
// ❌ 悪い例: 大量の画像データを保存
let imageData = UIImage(named: "large")!.pngData()!
UserDefaults.standard.set(imageData, forKey: "profileImage") // 数MB

// ✅ 良い例: ファイルパスのみ保存
let imagePath = FileManager.default.saveImage(image, filename: "profile.png")
UserDefaults.standard.set(imagePath, forKey: "profileImagePath") // 数十バイト
```

#### 2. 不要なデータを定期的に削除

```swift
class UserDefaultsCleaner {
    static func removeObsoleteData() {
        let obsoleteKeys = [
            "oldFeatureFlag",
            "deprecatedSetting",
            "temporaryCache"
        ]

        obsoleteKeys.forEach { key in
            UserDefaults.standard.removeObject(forKey: key)
        }
    }
}
```

#### 3. 遅延読み込み

起動時に必要ない設定は、遅延読み込みにします。

```swift
class AppSettings {
    // 起動時に即座に必要
    @UserDefault(key: "language", defaultValue: "en")
    static var language: String

    // 遅延読み込み
    private static var _advancedSettings: AdvancedSettings?
    static var advancedSettings: AdvancedSettings {
        if let cached = _advancedSettings {
            return cached
        }
        let settings: AdvancedSettings = UserDefaults.standard.codable(forKey: "advancedSettings") ?? .default
        _advancedSettings = settings
        return settings
    }
}
```

#### 4. バッチ更新

複数の値を更新する場合は、まとめて更新します。

```swift
// ❌ 悪い例: 個別に更新（何度もディスクI/Oが発生する可能性）
UserDefaults.standard.set(true, forKey: "notifications")
UserDefaults.standard.set("dark", forKey: "theme")
UserDefaults.standard.set(16, forKey: "fontSize")

// ✅ 良い例: オブジェクトとして一括保存
struct AppSettings: Codable {
    var notifications: Bool
    var theme: String
    var fontSize: Int
}
let settings = AppSettings(notifications: true, theme: "dark", fontSize: 16)
UserDefaults.standard.setCodable(settings, forKey: "appSettings")
```

## UserDefaultsのテスト

テスト時には、テスト専用のUserDefaultsインスタンスを使うことで、本番データへの影響を避けます。

```swift
protocol UserDefaultsProtocol {
    func set(_ value: Any?, forKey key: String)
    func object(forKey key: String) -> Any?
    func bool(forKey key: String) -> Bool
    func integer(forKey key: String) -> Int
    func string(forKey key: String) -> String?
}

extension UserDefaults: UserDefaultsProtocol {}

class SettingsManager {
    private let userDefaults: UserDefaultsProtocol

    init(userDefaults: UserDefaultsProtocol = UserDefaults.standard) {
        self.userDefaults = userDefaults
    }

    var notificationsEnabled: Bool {
        get { userDefaults.bool(forKey: "notificationsEnabled") }
        set { userDefaults.set(newValue, forKey: "notificationsEnabled") }
    }
}

// テスト
class MockUserDefaults: UserDefaultsProtocol {
    private var storage: [String: Any] = [:]

    func set(_ value: Any?, forKey key: String) {
        storage[key] = value
    }

    func object(forKey key: String) -> Any? {
        storage[key]
    }

    func bool(forKey key: String) -> Bool {
        storage[key] as? Bool ?? false
    }

    func integer(forKey key: String) -> Int {
        storage[key] as? Int ?? 0
    }

    func string(forKey key: String) -> String? {
        storage[key] as? String
    }
}

class SettingsManagerTests: XCTestCase {
    func testNotificationsEnabled() {
        let mockDefaults = MockUserDefaults()
        let manager = SettingsManager(userDefaults: mockDefaults)

        manager.notificationsEnabled = true
        XCTAssertTrue(manager.notificationsEnabled)
    }
}
```

## チェックリスト

UserDefaults使用時のベストプラクティスチェックリスト：

- [ ] **型安全性**: Property WrapperまたはSwiftUIの`@AppStorage`を使用している
- [ ] **機密情報の保護**: パスワード・トークン等をUserDefaultsに保存していない
- [ ] **データサイズ**: 大量のデータ（画像・動画など）を保存していない
- [ ] **Codable活用**: 複雑なオブジェクトはCodableで保存している
- [ ] **App Groups**: Extensionとのデータ共有にApp Groupsを使用している
- [ ] **パフォーマンス**: 不要なデータを定期的に削除している
- [ ] **テスタビリティ**: テスト時にモックUserDefaultsを使えるように設計している
- [ ] **デフォルト値**: すべてのキーに適切なデフォルト値を設定している

## まとめ

UserDefaultsは設定値の保存に適していますが、機密情報はKeychainに保存することが推奨されます。Property Wrapperを活用することで、型安全で保守性の高いコードを実現できます。また、パフォーマンスを考慮し、適切なデータサイズと更新頻度を保つことが重要です。

次章では、ネットワーク層アーキテクチャについて詳しく解説します。

## 参考文献

- [Apple Developer Documentation - UserDefaults](https://developer.apple.com/documentation/foundation/userdefaults)
- [Apple Developer Documentation - App Groups](https://developer.apple.com/documentation/bundleresources/entitlements/com_apple_security_application-groups)
- [Swift by Sundell - Property Wrappers in Swift](https://www.swiftbysundell.com/articles/property-wrappers-in-swift/)
- [Hacking with Swift - How to use @AppStorage in SwiftUI](https://www.hackingwithswift.com/quick-start/swiftui/how-to-use-appstorage-to-read-and-write-to-userdefaults)
