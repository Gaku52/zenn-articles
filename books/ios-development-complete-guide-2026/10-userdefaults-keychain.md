---
title: "Chapter 10: UserDefaultsとKeychain - データ永続化の基礎"
---

# Chapter 10: UserDefaultsとKeychain - データ永続化の基礎

## 10.1 概要

iOSアプリケーション開発において、データの永続化は避けて通れない重要なテーマです。ユーザーの設定、認証情報、一時的なデータなど、様々な種類の情報を適切に保存・管理する必要があります。

この章では、最も基本的で広く使われている2つのデータ永続化手段、UserDefaultsとKeychainについて詳しく解説します。

### 本章で学べること

- UserDefaultsの基本的な使い方と高度な実装パターン
- Keychainを使った機密情報の安全な保存
- PropertyWrapperによる型安全な実装
- iCloud同期の実装方法
- パフォーマンス最適化とベストプラクティス
- 実践的なエラーハンドリング

### UserDefaultsとKeychainの使い分け

```swift
// データの種類による使い分け指針
enum DataStorageStrategy {
    case userDefaults
    case keychain
    case fileSystem

    static func recommended(for dataType: DataType) -> DataStorageStrategy {
        switch dataType {
        case .appSettings:
            return .userDefaults
        case .userPreferences:
            return .userDefaults
        case .temporaryCache:
            return .userDefaults
        case .password:
            return .keychain
        case .accessToken:
            return .keychain
        case .apiKey:
            return .keychain
        case .certificate:
            return .keychain
        case .largeDocument:
            return .fileSystem
        case .mediaFile:
            return .fileSystem
        }
    }
}

enum DataType {
    case appSettings
    case userPreferences
    case temporaryCache
    case password
    case accessToken
    case apiKey
    case certificate
    case largeDocument
    case mediaFile
}
```

## 10.2 UserDefaults - 基本実装

### 10.2.1 基本的なデータ型の保存と取得

UserDefaultsは、小さな設定値やユーザー設定を保存するための軽量なキーバリューストアです。

```swift
import Foundation

// 基本的な使用方法
class BasicUserDefaultsExample {
    private let defaults = UserDefaults.standard

    // MARK: - Bool値の保存と取得
    func saveBoolValue() {
        defaults.set(true, forKey: "isDarkModeEnabled")
    }

    func loadBoolValue() -> Bool {
        // デフォルト値はfalseが返される
        return defaults.bool(forKey: "isDarkModeEnabled")
    }

    // MARK: - String値の保存と取得
    func saveStringValue() {
        defaults.set("John Doe", forKey: "username")
    }

    func loadStringValue() -> String? {
        return defaults.string(forKey: "username")
    }

    // MARK: - Int値の保存と取得
    func saveIntValue() {
        defaults.set(42, forKey: "userAge")
    }

    func loadIntValue() -> Int {
        // キーが存在しない場合は0が返される
        return defaults.integer(forKey: "userAge")
    }

    // MARK: - Double値の保存と取得
    func saveDoubleValue() {
        defaults.set(3.14159, forKey: "piValue")
    }

    func loadDoubleValue() -> Double {
        return defaults.double(forKey: "piValue")
    }

    // MARK: - Date値の保存と取得
    func saveDateValue() {
        let now = Date()
        defaults.set(now, forKey: "lastLoginDate")
    }

    func loadDateValue() -> Date? {
        return defaults.object(forKey: "lastLoginDate") as? Date
    }

    // MARK: - Array値の保存と取得
    func saveArrayValue() {
        let colors = ["Red", "Green", "Blue"]
        defaults.set(colors, forKey: "favoriteColors")
    }

    func loadArrayValue() -> [String] {
        return defaults.stringArray(forKey: "favoriteColors") ?? []
    }

    // MARK: - Dictionary値の保存と取得
    func saveDictionaryValue() {
        let userInfo: [String: Any] = [
            "name": "John Doe",
            "age": 30,
            "isVerified": true
        ]
        defaults.set(userInfo, forKey: "userInfo")
    }

    func loadDictionaryValue() -> [String: Any]? {
        return defaults.dictionary(forKey: "userInfo")
    }

    // MARK: - 値の削除
    func removeValue(forKey key: String) {
        defaults.removeObject(forKey: key)
    }

    // MARK: - すべての値をリセット
    func resetAllDefaults() {
        if let bundleIdentifier = Bundle.main.bundleIdentifier {
            defaults.removePersistentDomain(forName: bundleIdentifier)
        }
    }

    // MARK: - 同期的な保存
    func synchronize() {
        // iOS 12以降では通常不要だが、明示的に保存したい場合
        defaults.synchronize()
    }
}
```

### 10.2.2 デフォルト値の登録

アプリケーション起動時に、デフォルト値を登録することで、キーが存在しない場合のフォールバック値を設定できます。

```swift
class DefaultValueManager {
    static func registerDefaults() {
        let defaults: [String: Any] = [
            // 表示設定
            "isDarkModeEnabled": false,
            "fontSize": 14,
            "showNotifications": true,

            // アプリケーション設定
            "maxCacheSize": 100_000_000, // 100MB
            "autoSaveEnabled": true,
            "autoSaveInterval": 300, // 5分

            // ユーザー設定
            "language": "en",
            "region": "US",

            // 機能フラグ
            "betaFeaturesEnabled": false,
            "analyticsEnabled": true,

            // 初回起動フラグ
            "hasLaunchedBefore": false,
            "appVersion": "1.0.0"
        ]

        UserDefaults.standard.register(defaults: defaults)
    }
}

// AppDelegateまたはApp構造体で呼び出し
@main
struct MyApp: App {
    init() {
        DefaultValueManager.registerDefaults()
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

### 10.2.3 型安全なUserDefaults実装

生の文字列キーを使うとタイプミスやキーの重複が発生しやすいため、型安全な実装を行います。

```swift
// キー定義を集約
enum UserDefaultsKeys {
    // 設定キー
    enum Settings: String {
        case isDarkModeEnabled = "settings.isDarkMode"
        case fontSize = "settings.fontSize"
        case showNotifications = "settings.showNotifications"
        case language = "settings.language"
    }

    // ユーザー情報キー
    enum User: String {
        case username = "user.username"
        case email = "user.email"
        case lastLoginDate = "user.lastLoginDate"
        case loginCount = "user.loginCount"
    }

    // キャッシュキー
    enum Cache: String {
        case lastSyncDate = "cache.lastSyncDate"
        case cachedData = "cache.data"
    }
}

// 型安全なUserDefaultsラッパー
class TypeSafeUserDefaults {
    private let defaults: UserDefaults

    init(defaults: UserDefaults = .standard) {
        self.defaults = defaults
    }

    // MARK: - Settings
    var isDarkModeEnabled: Bool {
        get { defaults.bool(forKey: UserDefaultsKeys.Settings.isDarkModeEnabled.rawValue) }
        set { defaults.set(newValue, forKey: UserDefaultsKeys.Settings.isDarkModeEnabled.rawValue) }
    }

    var fontSize: Int {
        get { defaults.integer(forKey: UserDefaultsKeys.Settings.fontSize.rawValue) }
        set { defaults.set(newValue, forKey: UserDefaultsKeys.Settings.fontSize.rawValue) }
    }

    var showNotifications: Bool {
        get { defaults.bool(forKey: UserDefaultsKeys.Settings.showNotifications.rawValue) }
        set { defaults.set(newValue, forKey: UserDefaultsKeys.Settings.showNotifications.rawValue) }
    }

    var language: String? {
        get { defaults.string(forKey: UserDefaultsKeys.Settings.language.rawValue) }
        set { defaults.set(newValue, forKey: UserDefaultsKeys.Settings.language.rawValue) }
    }

    // MARK: - User Info
    var username: String? {
        get { defaults.string(forKey: UserDefaultsKeys.User.username.rawValue) }
        set { defaults.set(newValue, forKey: UserDefaultsKeys.User.username.rawValue) }
    }

    var email: String? {
        get { defaults.string(forKey: UserDefaultsKeys.User.email.rawValue) }
        set { defaults.set(newValue, forKey: UserDefaultsKeys.User.email.rawValue) }
    }

    var lastLoginDate: Date? {
        get { defaults.object(forKey: UserDefaultsKeys.User.lastLoginDate.rawValue) as? Date }
        set { defaults.set(newValue, forKey: UserDefaultsKeys.User.lastLoginDate.rawValue) }
    }

    var loginCount: Int {
        get { defaults.integer(forKey: UserDefaultsKeys.User.loginCount.rawValue) }
        set { defaults.set(newValue, forKey: UserDefaultsKeys.User.loginCount.rawValue) }
    }

    // MARK: - Cache
    var lastSyncDate: Date? {
        get { defaults.object(forKey: UserDefaultsKeys.Cache.lastSyncDate.rawValue) as? Date }
        set { defaults.set(newValue, forKey: UserDefaultsKeys.Cache.lastSyncDate.rawValue) }
    }

    // MARK: - Helper Methods
    func incrementLoginCount() {
        loginCount += 1
        lastLoginDate = Date()
    }

    func clearUserData() {
        username = nil
        email = nil
        lastLoginDate = nil
        loginCount = 0
    }
}
```

## 10.3 PropertyWrapperによる高度な実装

### 10.3.1 基本的なPropertyWrapper

PropertyWrapperを使用することで、よりクリーンで保守しやすいコードを書けます。

```swift
@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T
    var container: UserDefaults = .standard

    var wrappedValue: T {
        get {
            return container.object(forKey: key) as? T ?? defaultValue
        }
        set {
            container.set(newValue, forKey: key)
        }
    }
}

// 使用例
class AppSettings {
    @UserDefault(key: "isDarkMode", defaultValue: false)
    var isDarkMode: Bool

    @UserDefault(key: "fontSize", defaultValue: 14)
    var fontSize: Int

    @UserDefault(key: "username", defaultValue: "")
    var username: String

    @UserDefault(key: "volume", defaultValue: 0.5)
    var volume: Double
}

// 使用方法
let settings = AppSettings()
settings.isDarkMode = true
print(settings.isDarkMode) // true
```

### 10.3.2 Optional値対応のPropertyWrapper

Optional値を扱えるPropertyWrapperの実装です。

```swift
@propertyWrapper
struct OptionalUserDefault<T> {
    let key: String
    var container: UserDefaults = .standard

    var wrappedValue: T? {
        get {
            return container.object(forKey: key) as? T
        }
        set {
            if let value = newValue {
                container.set(value, forKey: key)
            } else {
                container.removeObject(forKey: key)
            }
        }
    }
}

// 使用例
class UserProfile {
    @OptionalUserDefault(key: "profileImageURL")
    var profileImageURL: String?

    @OptionalUserDefault(key: "bio")
    var bio: String?

    @OptionalUserDefault(key: "birthdate")
    var birthdate: Date?
}

let profile = UserProfile()
profile.profileImageURL = "https://example.com/avatar.jpg"
profile.profileImageURL = nil // 値を削除
```

### 10.3.3 Codable対応のPropertyWrapper

複雑な型をJSONとして保存するPropertyWrapperです。

```swift
@propertyWrapper
struct CodableUserDefault<T: Codable> {
    let key: String
    let defaultValue: T
    var container: UserDefaults = .standard

    private let encoder = JSONEncoder()
    private let decoder = JSONDecoder()

    var wrappedValue: T {
        get {
            guard let data = container.data(forKey: key) else {
                return defaultValue
            }

            do {
                return try decoder.decode(T.self, from: data)
            } catch {
                print("Failed to decode \(key): \(error)")
                return defaultValue
            }
        }
        set {
            do {
                let data = try encoder.encode(newValue)
                container.set(data, forKey: key)
            } catch {
                print("Failed to encode \(key): \(error)")
            }
        }
    }
}

// 使用例
struct Theme: Codable {
    var primaryColor: String
    var secondaryColor: String
    var isDark: Bool
}

struct NotificationSettings: Codable {
    var enabled: Bool
    var soundEnabled: Bool
    var vibrationEnabled: Bool
    var categories: [String]
}

class AdvancedSettings {
    @CodableUserDefault(
        key: "theme",
        defaultValue: Theme(
            primaryColor: "#007AFF",
            secondaryColor: "#5856D6",
            isDark: false
        )
    )
    var theme: Theme

    @CodableUserDefault(
        key: "notifications",
        defaultValue: NotificationSettings(
            enabled: true,
            soundEnabled: true,
            vibrationEnabled: true,
            categories: ["messages", "updates"]
        )
    )
    var notificationSettings: NotificationSettings
}

let settings = AdvancedSettings()
settings.theme.isDark = true
print(settings.theme.primaryColor) // "#007AFF"
```

### 10.3.4 Enum対応のPropertyWrapper

Enumを安全に保存・取得するPropertyWrapperです。

```swift
@propertyWrapper
struct EnumUserDefault<T: RawRepresentable> {
    let key: String
    let defaultValue: T
    var container: UserDefaults = .standard

    var wrappedValue: T {
        get {
            guard let rawValue = container.object(forKey: key) as? T.RawValue,
                  let value = T(rawValue: rawValue) else {
                return defaultValue
            }
            return value
        }
        set {
            container.set(newValue.rawValue, forKey: key)
        }
    }
}

// 使用例
enum AppTheme: String {
    case light
    case dark
    case auto
}

enum FontSize: Int {
    case small = 12
    case medium = 14
    case large = 16
    case extraLarge = 18
}

enum Language: String {
    case english = "en"
    case japanese = "ja"
    case spanish = "es"
    case french = "fr"
}

class DisplaySettings {
    @EnumUserDefault(key: "appTheme", defaultValue: .auto)
    var appTheme: AppTheme

    @EnumUserDefault(key: "fontSize", defaultValue: .medium)
    var fontSize: FontSize

    @EnumUserDefault(key: "language", defaultValue: .english)
    var language: Language
}

let displaySettings = DisplaySettings()
displaySettings.appTheme = .dark
displaySettings.fontSize = .large
print(displaySettings.appTheme) // dark
```

### 10.3.5 観測可能なPropertyWrapper

値の変更を監視できるPropertyWrapperです。

```swift
@propertyWrapper
struct ObservableUserDefault<T> {
    let key: String
    let defaultValue: T
    var container: UserDefaults = .standard
    var onChange: ((T) -> Void)?

    var wrappedValue: T {
        get {
            return container.object(forKey: key) as? T ?? defaultValue
        }
        set {
            container.set(newValue, forKey: key)
            onChange?(newValue)
        }
    }

    init(key: String, defaultValue: T, onChange: ((T) -> Void)? = nil) {
        self.key = key
        self.defaultValue = defaultValue
        self.onChange = onChange
    }
}

// 使用例
class ObservableSettings {
    @ObservableUserDefault(
        key: "volume",
        defaultValue: 0.5,
        onChange: { newVolume in
            print("Volume changed to: \(newVolume)")
            // AudioEngine.setVolume(newVolume)
        }
    )
    var volume: Double

    @ObservableUserDefault(
        key: "brightness",
        defaultValue: 0.8,
        onChange: { newBrightness in
            print("Brightness changed to: \(newBrightness)")
            // UIScreen.main.brightness = newBrightness
        }
    )
    var brightness: Double
}
```

## 10.4 Codableを使った高度なデータ管理

### 10.4.1 複雑な構造体の保存

```swift
// モデル定義
struct UserPreferences: Codable {
    var displaySettings: DisplaySettings
    var privacySettings: PrivacySettings
    var notificationSettings: NotificationSettings
    var dataSettings: DataSettings

    struct DisplaySettings: Codable {
        var theme: Theme
        var fontSize: FontSize
        var language: String
        var showAvatars: Bool

        enum Theme: String, Codable {
            case light, dark, auto
        }

        enum FontSize: String, Codable {
            case small, medium, large, extraLarge
        }
    }

    struct PrivacySettings: Codable {
        var analyticsEnabled: Bool
        var crashReportingEnabled: Bool
        var locationServicesEnabled: Bool
        var shareUsageData: Bool
    }

    struct NotificationSettings: Codable {
        var enabled: Bool
        var sound: Bool
        var badge: Bool
        var preview: PreviewStyle

        enum PreviewStyle: String, Codable {
            case always, whenUnlocked, never
        }
    }

    struct DataSettings: Codable {
        var autoDownload: AutoDownloadSetting
        var cacheSize: Int
        var clearCacheOnExit: Bool

        enum AutoDownloadSetting: String, Codable {
            case always, wifiOnly, never
        }
    }
}

// デフォルト値
extension UserPreferences {
    static let `default` = UserPreferences(
        displaySettings: DisplaySettings(
            theme: .auto,
            fontSize: .medium,
            language: "en",
            showAvatars: true
        ),
        privacySettings: PrivacySettings(
            analyticsEnabled: true,
            crashReportingEnabled: true,
            locationServicesEnabled: false,
            shareUsageData: false
        ),
        notificationSettings: NotificationSettings(
            enabled: true,
            sound: true,
            badge: true,
            preview: .whenUnlocked
        ),
        dataSettings: DataSettings(
            autoDownload: .wifiOnly,
            cacheSize: 100_000_000,
            clearCacheOnExit: false
        )
    )
}

// マネージャークラス
class UserPreferencesManager {
    static let shared = UserPreferencesManager()

    private let defaults = UserDefaults.standard
    private let key = "userPreferences"
    private let encoder = JSONEncoder()
    private let decoder = JSONDecoder()

    private init() {
        encoder.outputFormatting = [.prettyPrinted, .sortedKeys]
    }

    var preferences: UserPreferences {
        get {
            guard let data = defaults.data(forKey: key) else {
                return .default
            }

            do {
                return try decoder.decode(UserPreferences.self, from: data)
            } catch {
                print("Failed to decode preferences: \(error)")
                return .default
            }
        }
        set {
            do {
                let data = try encoder.encode(newValue)
                defaults.set(data, forKey: key)
            } catch {
                print("Failed to encode preferences: \(error)")
            }
        }
    }

    // 個別設定の更新メソッド
    func updateDisplaySettings(_ update: (inout UserPreferences.DisplaySettings) -> Void) {
        var prefs = preferences
        update(&prefs.displaySettings)
        preferences = prefs
    }

    func updatePrivacySettings(_ update: (inout UserPreferences.PrivacySettings) -> Void) {
        var prefs = preferences
        update(&prefs.privacySettings)
        preferences = prefs
    }

    func updateNotificationSettings(_ update: (inout UserPreferences.NotificationSettings) -> Void) {
        var prefs = preferences
        update(&prefs.notificationSettings)
        preferences = prefs
    }

    func updateDataSettings(_ update: (inout UserPreferences.DataSettings) -> Void) {
        var prefs = preferences
        update(&prefs.dataSettings)
        preferences = prefs
    }

    // リセット
    func reset() {
        preferences = .default
    }
}

// 使用例
let manager = UserPreferencesManager.shared

// テーマを変更
manager.updateDisplaySettings { settings in
    settings.theme = .dark
    settings.fontSize = .large
}

// プライバシー設定を変更
manager.updatePrivacySettings { settings in
    settings.analyticsEnabled = false
    settings.shareUsageData = false
}

// 現在の設定を取得
let currentTheme = manager.preferences.displaySettings.theme
print("Current theme: \(currentTheme)")
```

### 10.4.2 バージョン管理とマイグレーション

```swift
struct VersionedUserPreferences: Codable {
    let version: Int
    var preferences: UserPreferences

    static let currentVersion = 2
}

class VersionedPreferencesManager {
    static let shared = VersionedPreferencesManager()

    private let defaults = UserDefaults.standard
    private let key = "versionedUserPreferences"
    private let encoder = JSONEncoder()
    private let decoder = JSONDecoder()

    private init() {}

    var preferences: UserPreferences {
        get {
            loadAndMigrateIfNeeded()
        }
        set {
            savePreferences(newValue)
        }
    }

    private func loadAndMigrateIfNeeded() -> UserPreferences {
        guard let data = defaults.data(forKey: key) else {
            return .default
        }

        do {
            let versioned = try decoder.decode(VersionedUserPreferences.self, from: data)

            // マイグレーションが必要か確認
            if versioned.version < VersionedUserPreferences.currentVersion {
                return migratePreferences(from: versioned)
            }

            return versioned.preferences
        } catch {
            print("Failed to decode versioned preferences: \(error)")
            return .default
        }
    }

    private func savePreferences(_ preferences: UserPreferences) {
        let versioned = VersionedUserPreferences(
            version: VersionedUserPreferences.currentVersion,
            preferences: preferences
        )

        do {
            let data = try encoder.encode(versioned)
            defaults.set(data, forKey: key)
        } catch {
            print("Failed to encode versioned preferences: \(error)")
        }
    }

    private func migratePreferences(from versioned: VersionedUserPreferences) -> UserPreferences {
        var preferences = versioned.preferences

        // バージョン1から2へのマイグレーション
        if versioned.version < 2 {
            print("Migrating preferences from version \(versioned.version) to 2")
            // マイグレーション処理
            preferences.dataSettings.clearCacheOnExit = false
        }

        // マイグレーション後の設定を保存
        savePreferences(preferences)

        return preferences
    }
}
```

## 10.5 Keychain - 機密情報の安全な保存

### 10.5.1 Keychainの基本実装

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

### 10.5.2 認証情報マネージャー

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

### 10.5.3 生体認証付きKeychain

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

## 10.6 iCloud同期

### 10.6.1 NSUbiquitousKeyValueStoreの実装

iCloudを使ってUserDefaultsのような設定をデバイス間で同期できます。

```swift
import Foundation

class iCloudSyncManager {
    static let shared = iCloudSyncManager()

    private let store = NSUbiquitousKeyValueStore.default
    private let notificationCenter = NotificationCenter.default

    private init() {
        setupObserver()
    }

    // MARK: - Setup
    private func setupObserver() {
        notificationCenter.addObserver(
            self,
            selector: #selector(storeDidChange),
            name: NSUbiquitousKeyValueStore.didChangeExternallyNotification,
            object: store
        )

        // 初期同期
        store.synchronize()
    }

    @objc private func storeDidChange(_ notification: Notification) {
        guard let userInfo = notification.userInfo,
              let changeReason = userInfo[NSUbiquitousKeyValueStoreChangeReasonKey] as? Int else {
            return
        }

        switch changeReason {
        case NSUbiquitousKeyValueStoreServerChange:
            print("iCloud data changed on server")
            handleServerChange(userInfo)
        case NSUbiquitousKeyValueStoreInitialSyncChange:
            print("Initial iCloud sync completed")
            handleInitialSync(userInfo)
        case NSUbiquitousKeyValueStoreQuotaViolationChange:
            print("iCloud quota exceeded")
            handleQuotaViolation()
        case NSUbiquitousKeyValueStoreAccountChange:
            print("iCloud account changed")
            handleAccountChange()
        default:
            break
        }
    }

    // MARK: - Change Handlers
    private func handleServerChange(_ userInfo: [AnyHashable: Any]) {
        guard let changedKeys = userInfo[NSUbiquitousKeyValueStoreChangedKeysKey] as? [String] else {
            return
        }

        for key in changedKeys {
            // 変更通知を送信
            NotificationCenter.default.post(
                name: .iCloudSettingsDidChange,
                object: nil,
                userInfo: ["key": key]
            )
        }
    }

    private func handleInitialSync(_ userInfo: [AnyHashable: Any]) {
        // 初期同期時の処理
        NotificationCenter.default.post(
            name: .iCloudInitialSyncCompleted,
            object: nil
        )
    }

    private func handleQuotaViolation() {
        // クォータ超過時の処理
        NotificationCenter.default.post(
            name: .iCloudQuotaExceeded,
            object: nil
        )
    }

    private func handleAccountChange() {
        // アカウント変更時の処理
        NotificationCenter.default.post(
            name: .iCloudAccountChanged,
            object: nil
        )
    }

    // MARK: - Save Methods
    func set(_ value: Any?, forKey key: String) {
        store.set(value, forKey: key)
        store.synchronize()
    }

    func setBool(_ value: Bool, forKey key: String) {
        store.set(value, forKey: key)
        store.synchronize()
    }

    func setString(_ value: String?, forKey key: String) {
        store.set(value, forKey: key)
        store.synchronize()
    }

    func setInt(_ value: Int, forKey key: String) {
        store.set(Int64(value), forKey: key)
        store.synchronize()
    }

    func setDouble(_ value: Double, forKey key: String) {
        store.set(value, forKey: key)
        store.synchronize()
    }

    func setData(_ value: Data?, forKey key: String) {
        store.set(value, forKey: key)
        store.synchronize()
    }

    // MARK: - Load Methods
    func object(forKey key: String) -> Any? {
        return store.object(forKey: key)
    }

    func bool(forKey key: String) -> Bool {
        return store.bool(forKey: key)
    }

    func string(forKey key: String) -> String? {
        return store.string(forKey: key)
    }

    func int(forKey key: String) -> Int {
        return Int(store.longLong(forKey: key))
    }

    func double(forKey key: String) -> Double {
        return store.double(forKey: key)
    }

    func data(forKey key: String) -> Data? {
        return store.data(forKey: key)
    }

    // MARK: - Remove
    func removeObject(forKey key: String) {
        store.removeObject(forKey: key)
        store.synchronize()
    }

    // MARK: - Sync
    func synchronize() {
        store.synchronize()
    }
}

// MARK: - Notification Names
extension Notification.Name {
    static let iCloudSettingsDidChange = Notification.Name("iCloudSettingsDidChange")
    static let iCloudInitialSyncCompleted = Notification.Name("iCloudInitialSyncCompleted")
    static let iCloudQuotaExceeded = Notification.Name("iCloudQuotaExceeded")
    static let iCloudAccountChanged = Notification.Name("iCloudAccountChanged")
}

// 使用例
let iCloud = iCloudSyncManager.shared

// 設定を保存（自動的にiCloudに同期）
iCloud.setBool(true, forKey: "darkModeEnabled")
iCloud.setString("en", forKey: "preferredLanguage")
iCloud.setInt(14, forKey: "fontSize")

// 設定を読み込み
let darkModeEnabled = iCloud.bool(forKey: "darkModeEnabled")
let language = iCloud.string(forKey: "preferredLanguage")

// 変更通知を監視
NotificationCenter.default.addObserver(
    forName: .iCloudSettingsDidChange,
    object: nil,
    queue: .main
) { notification in
    if let key = notification.userInfo?["key"] as? String {
        print("iCloud setting changed: \(key)")
    }
}
```

### 10.6.2 iCloud同期とローカル設定の統合

```swift
class SyncedSettingsManager {
    static let shared = SyncedSettingsManager()

    private let userDefaults = UserDefaults.standard
    private let iCloud = iCloudSyncManager.shared

    private enum Keys {
        static let useiCloudSync = "useiCloudSync"
    }

    private init() {
        setupSyncObserver()
    }

    var useiCloudSync: Bool {
        get {
            userDefaults.bool(forKey: Keys.useiCloudSync)
        }
        set {
            userDefaults.set(newValue, forKey: Keys.useiCloudSync)
            if newValue {
                syncToiCloud()
            }
        }
    }

    // MARK: - Setup
    private func setupSyncObserver() {
        NotificationCenter.default.addObserver(
            forName: .iCloudSettingsDidChange,
            object: nil,
            queue: .main
        ) { [weak self] notification in
            guard let self = self,
                  self.useiCloudSync,
                  let key = notification.userInfo?["key"] as? String else {
                return
            }

            self.syncFromiCloud(key: key)
        }
    }

    // MARK: - Save
    func set(_ value: Any?, forKey key: String) {
        userDefaults.set(value, forKey: key)

        if useiCloudSync {
            iCloud.set(value, forKey: key)
        }
    }

    func setBool(_ value: Bool, forKey key: String) {
        userDefaults.set(value, forKey: key)

        if useiCloudSync {
            iCloud.setBool(value, forKey: key)
        }
    }

    func setString(_ value: String?, forKey key: String) {
        userDefaults.set(value, forKey: key)

        if useiCloudSync {
            iCloud.setString(value, forKey: key)
        }
    }

    // MARK: - Load
    func object(forKey key: String) -> Any? {
        if useiCloudSync {
            return iCloud.object(forKey: key) ?? userDefaults.object(forKey: key)
        }
        return userDefaults.object(forKey: key)
    }

    func bool(forKey key: String) -> Bool {
        if useiCloudSync {
            // iCloudに値が存在する場合はそれを使用
            if iCloud.object(forKey: key) != nil {
                return iCloud.bool(forKey: key)
            }
        }
        return userDefaults.bool(forKey: key)
    }

    func string(forKey key: String) -> String? {
        if useiCloudSync {
            return iCloud.string(forKey: key) ?? userDefaults.string(forKey: key)
        }
        return userDefaults.string(forKey: key)
    }

    // MARK: - Sync
    private func syncToiCloud() {
        guard let domain = Bundle.main.bundleIdentifier else { return }
        guard let settings = userDefaults.persistentDomain(forName: domain) else { return }

        for (key, value) in settings {
            iCloud.set(value, forKey: key)
        }

        iCloud.synchronize()
    }

    private func syncFromiCloud(key: String) {
        if let value = iCloud.object(forKey: key) {
            userDefaults.set(value, forKey: key)
        }
    }
}
```

## 10.7 パフォーマンス最適化

### 10.7.1 バッチ更新

```swift
class BatchUpdateManager {
    private var pendingUpdates: [String: Any] = [:]
    private let updateQueue = DispatchQueue(label: "com.example.batchUpdate")
    private var updateTimer: Timer?

    func scheduleUpdate(key: String, value: Any) {
        updateQueue.async {
            self.pendingUpdates[key] = value
            self.scheduleFlush()
        }
    }

    private func scheduleFlush() {
        updateTimer?.invalidate()
        updateTimer = Timer.scheduledTimer(
            withTimeInterval: 1.0,
            repeats: false
        ) { [weak self] _ in
            self?.flush()
        }
    }

    private func flush() {
        updateQueue.async {
            let updates = self.pendingUpdates
            self.pendingUpdates.removeAll()

            DispatchQueue.main.async {
                let defaults = UserDefaults.standard
                for (key, value) in updates {
                    defaults.set(value, forKey: key)
                }
            }
        }
    }
}
```

### 10.7.2 キャッシュ戦略

```swift
class CachedUserDefaults {
    private let defaults = UserDefaults.standard
    private var cache: [String: Any] = [:]
    private let cacheQueue = DispatchQueue(label: "com.example.cache", attributes: .concurrent)

    func object(forKey key: String) -> Any? {
        // キャッシュから取得
        return cacheQueue.sync {
            if let cached = cache[key] {
                return cached
            }

            // UserDefaultsから取得してキャッシュ
            let value = defaults.object(forKey: key)
            cache[key] = value
            return value
        }
    }

    func set(_ value: Any?, forKey key: String) {
        defaults.set(value, forKey: key)

        cacheQueue.async(flags: .barrier) {
            self.cache[key] = value
        }
    }

    func clearCache() {
        cacheQueue.async(flags: .barrier) {
            self.cache.removeAll()
        }
    }
}
```

## 10.8 テストとデバッグ

### 10.8.1 ユニットテスト

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

## 10.9 ベストプラクティス

### 10.9.1 設計原則

```swift
// ✅ 良い例：責任の分離
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

// ❌ 悪い例：責任が混在
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

### 10.9.2 エラーハンドリング

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

## 10.10 まとめ

この章では、UserDefaultsとKeychainを使ったデータ永続化の基礎から高度な実装まで学びました。

### 重要なポイント

1. **適切な使い分け**
   - UserDefaults: 小さな設定値、ユーザー設定
   - Keychain: パスワード、トークン、機密情報

2. **型安全性**
   - PropertyWrapperの活用
   - Codableによる複雑なデータの保存

3. **セキュリティ**
   - 機密情報は必ずKeychainに保存
   - 生体認証の活用

4. **パフォーマンス**
   - バッチ更新の実装
   - キャッシュ戦略の適用

5. **テスト**
   - ユニットテストの実装
   - テスト用のUserDefaults suiteの使用

次の章では、より高度なデータ永続化手段であるCore Dataについて詳しく学びます。
