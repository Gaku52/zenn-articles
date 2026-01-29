---
title: "UserDefaults と FileManager: 軽量データの永続化"
---

# UserDefaults と FileManager: 軽量データの永続化

この章では、iOS で最もシンプルなデータ永続化手段である UserDefaults と FileManager について学びます。適切な使い分けと実装パターンを理解することで、効率的なデータ管理が可能になります。

## UserDefaults の基本

UserDefaults は、アプリの設定や小さなデータを保存するための Key-Value ストレージです。

### 適切な使用例

- アプリの設定（テーマ、言語設定など）
- ユーザーの選択状態
- 最終ログイン日時
- フラグやカウンター

### 避けるべき使用例

- 大量のデータ（画像、動画など）
- 機密情報（パスワード、トークン）
- 複雑な構造化データ

## 基本的な使用方法

```swift
class AppSettings {
    static let shared = AppSettings()
    private let defaults = UserDefaults.standard

    // Bool値
    var notificationsEnabled: Bool {
        get { defaults.bool(forKey: Keys.notificationsEnabled) }
        set { defaults.set(newValue, forKey: Keys.notificationsEnabled) }
    }

    // String値
    var username: String? {
        get { defaults.string(forKey: Keys.username) }
        set { defaults.set(newValue, forKey: Keys.username) }
    }

    // Int値
    var loginCount: Int {
        get { defaults.integer(forKey: Keys.loginCount) }
        set { defaults.set(newValue, forKey: Keys.loginCount) }
    }

    // Date値
    var lastLoginDate: Date? {
        get { defaults.object(forKey: Keys.lastLoginDate) as? Date }
        set { defaults.set(newValue, forKey: Keys.lastLoginDate) }
    }

    private enum Keys {
        static let notificationsEnabled = "notificationsEnabled"
        static let username = "username"
        static let loginCount = "loginCount"
        static let lastLoginDate = "lastLoginDate"
    }

    private init() {
        registerDefaults()
    }

    private func registerDefaults() {
        defaults.register(defaults: [
            Keys.notificationsEnabled: true,
            Keys.loginCount: 0
        ])
    }
}
```

## Codable での使用

複雑な構造体を UserDefaults に保存します:

```swift
extension UserDefaults {
    func setCodable<T: Codable>(_ value: T, forKey key: String) throws {
        let encoder = JSONEncoder()
        let data = try encoder.encode(value)
        set(data, forKey: key)
    }

    func codable<T: Codable>(forKey key: String) throws -> T? {
        guard let data = data(forKey: key) else { return nil }
        let decoder = JSONDecoder()
        return try decoder.decode(T.self, from: data)
    }

    func removeCodable(forKey key: String) {
        removeObject(forKey: key)
    }
}

// 使用例
struct UserPreferences: Codable {
    var theme: Theme
    var language: String
    var fontSize: FontSize

    enum Theme: String, Codable {
        case light, dark, auto
    }

    enum FontSize: Int, Codable {
        case small = 12
        case medium = 14
        case large = 16
    }
}

class PreferencesManager {
    private let defaults = UserDefaults.standard
    private let preferencesKey = "userPreferences"

    var preferences: UserPreferences {
        get {
            (try? defaults.codable(forKey: preferencesKey)) ?? .default
        }
        set {
            try? defaults.setCodable(newValue, forKey: preferencesKey)
        }
    }
}

extension UserPreferences {
    static let `default` = UserPreferences(
        theme: .auto,
        language: "ja",
        fontSize: .medium
    )
}
```

## Property Wrapper による型安全な実装

Property Wrapper を使用すると、より簡潔で型安全なコードが書けます:

```swift
@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T
    var container: UserDefaults = .standard

    var wrappedValue: T {
        get {
            container.object(forKey: key) as? T ?? defaultValue
        }
        set {
            container.set(newValue, forKey: key)
        }
    }
}

@propertyWrapper
struct CodableUserDefault<T: Codable> {
    let key: String
    let defaultValue: T
    var container: UserDefaults = .standard

    var wrappedValue: T {
        get {
            guard let data = container.data(forKey: key),
                  let value = try? JSONDecoder().decode(T.self, from: data) else {
                return defaultValue
            }
            return value
        }
        set {
            let data = try? JSONEncoder().encode(newValue)
            container.set(data, forKey: key)
        }
    }
}

// 使用例
class Settings {
    @UserDefault(key: "isDarkMode", defaultValue: false)
    var isDarkMode: Bool

    @UserDefault(key: "volume", defaultValue: 0.5)
    var volume: Double

    @CodableUserDefault(key: "preferences", defaultValue: .default)
    var preferences: UserPreferences
}

// 使用
let settings = Settings()
settings.isDarkMode = true
settings.volume = 0.8
```

## FileManager の基本

FileManager は、ファイルシステムの操作を行うための API です。

### ディレクトリの種類

```swift
enum Directory {
    case documents    // ユーザーが作成したドキュメント
    case caches       // 再作成可能なキャッシュデータ
    case temporary    // 一時ファイル
    case applicationSupport  // アプリのサポートファイル

    var url: URL {
        let fileManager = FileManager.default
        switch self {
        case .documents:
            return fileManager.urls(for: .documentDirectory, in: .userDomainMask)[0]
        case .caches:
            return fileManager.urls(for: .cachesDirectory, in: .userDomainMask)[0]
        case .temporary:
            return fileManager.temporaryDirectory
        case .applicationSupport:
            return fileManager.urls(for: .applicationSupportDirectory, in: .userDomainMask)[0]
        }
    }
}
```

### ファイル操作マネージャー

```swift
class FileStorageManager {
    static let shared = FileStorageManager()
    private let fileManager = FileManager.default

    private init() {}

    // MARK: - Save
    func save<T: Codable>(_ object: T, to directory: Directory, filename: String) throws {
        let url = directory.url.appendingPathComponent(filename)
        let encoder = JSONEncoder()
        encoder.outputFormatting = .prettyPrinted
        let data = try encoder.encode(object)
        try data.write(to: url, options: .atomic)
    }

    func saveData(_ data: Data, to directory: Directory, filename: String) throws {
        let url = directory.url.appendingPathComponent(filename)
        try data.write(to: url, options: .atomic)
    }

    // MARK: - Load
    func load<T: Codable>(from directory: Directory, filename: String) throws -> T {
        let url = directory.url.appendingPathComponent(filename)
        let data = try Data(contentsOf: url)
        let decoder = JSONDecoder()
        return try decoder.decode(T.self, from: data)
    }

    func loadData(from directory: Directory, filename: String) throws -> Data {
        let url = directory.url.appendingPathComponent(filename)
        return try Data(contentsOf: url)
    }

    // MARK: - Delete
    func delete(from directory: Directory, filename: String) throws {
        let url = directory.url.appendingPathComponent(filename)
        try fileManager.removeItem(at: url)
    }

    // MARK: - Exists
    func fileExists(in directory: Directory, filename: String) -> Bool {
        let url = directory.url.appendingPathComponent(filename)
        return fileManager.fileExists(atPath: url.path)
    }

    // MARK: - List Files
    func listFiles(in directory: Directory) throws -> [String] {
        let contents = try fileManager.contentsOfDirectory(
            at: directory.url,
            includingPropertiesForKeys: nil
        )
        return contents.map { $0.lastPathComponent }
    }

    // MARK: - File Attributes
    func fileSize(in directory: Directory, filename: String) throws -> Int {
        let url = directory.url.appendingPathComponent(filename)
        let attributes = try fileManager.attributesOfItem(atPath: url.path)
        return attributes[.size] as? Int ?? 0
    }

    func modificationDate(in directory: Directory, filename: String) throws -> Date? {
        let url = directory.url.appendingPathComponent(filename)
        let attributes = try fileManager.attributesOfItem(atPath: url.path)
        return attributes[.modificationDate] as? Date
    }
}
```

### 使用例

```swift
struct Note: Codable {
    let id: UUID
    let title: String
    let content: String
    let createdAt: Date
}

class NotesManager {
    private let storage = FileStorageManager.shared

    func saveNote(_ note: Note) throws {
        let filename = "\(note.id.uuidString).json"
        try storage.save(note, to: .documents, filename: filename)
    }

    func loadNote(id: UUID) throws -> Note {
        let filename = "\(id.uuidString).json"
        return try storage.load(from: .documents, filename: filename)
    }

    func loadAllNotes() throws -> [Note] {
        let files = try storage.listFiles(in: .documents)
        return try files.compactMap { filename in
            try? storage.load(from: .documents, filename: filename) as Note
        }
    }

    func deleteNote(id: UUID) throws {
        let filename = "\(id.uuidString).json"
        try storage.delete(from: .documents, filename: filename)
    }
}
```

## 画像の保存と読み込み

画像ファイルを扱う専用のマネージャーを実装します:

```swift
class ImageStorageManager {
    static let shared = ImageStorageManager()
    private let storage = FileStorageManager.shared

    func saveImage(_ image: UIImage, filename: String, compressionQuality: CGFloat = 0.8) throws {
        guard let data = image.jpegData(compressionQuality: compressionQuality) else {
            throw StorageError.imageConversionFailed
        }

        try storage.saveData(data, to: .documents, filename: filename)
    }

    func loadImage(filename: String) throws -> UIImage {
        let data = try storage.loadData(from: .documents, filename: filename)
        guard let image = UIImage(data: data) else {
            throw StorageError.imageLoadFailed
        }
        return image
    }

    func deleteImage(filename: String) throws {
        try storage.delete(from: .documents, filename: filename)
    }

    func imageExists(filename: String) -> Bool {
        storage.fileExists(in: .documents, filename: filename)
    }

    func listImages() throws -> [String] {
        let allFiles = try storage.listFiles(in: .documents)
        return allFiles.filter { file in
            let ext = (file as NSString).pathExtension.lowercased()
            return ["jpg", "jpeg", "png"].contains(ext)
        }
    }
}

enum StorageError: Error {
    case imageConversionFailed
    case imageLoadFailed
}
```

## ディレクトリの管理

カスタムディレクトリを作成・管理します:

```swift
class DirectoryManager {
    static let shared = DirectoryManager()
    private let fileManager = FileManager.default

    func createDirectory(name: String, in directory: Directory) throws -> URL {
        let dirURL = directory.url.appendingPathComponent(name)

        if !fileManager.fileExists(atPath: dirURL.path) {
            try fileManager.createDirectory(
                at: dirURL,
                withIntermediateDirectories: true,
                attributes: nil
            )
        }

        return dirURL
    }

    func removeDirectory(name: String, in directory: Directory) throws {
        let dirURL = directory.url.appendingPathComponent(name)
        try fileManager.removeItem(at: dirURL)
    }

    func directorySize(url: URL) throws -> Int {
        var size = 0
        let contents = try fileManager.contentsOfDirectory(
            at: url,
            includingPropertiesForKeys: [.fileSizeKey]
        )

        for fileURL in contents {
            let attributes = try fileManager.attributesOfItem(atPath: fileURL.path)
            size += attributes[.size] as? Int ?? 0
        }

        return size
    }
}
```

## まとめ

この章では、UserDefaults と FileManager を使った軽量データの永続化を学びました:

- UserDefaults による設定の保存
- Codable を使った構造化データの保存
- Property Wrapper による型安全な実装
- FileManager によるファイル操作
- 画像の保存と読み込み

次の章では、Keychain による機密情報の安全な保存方法を学びます。

## 参考リソース

- [UserDefaults - Apple Developer](https://developer.apple.com/documentation/foundation/userdefaults)
- [FileManager - Apple Developer](https://developer.apple.com/documentation/foundation/filemanager)
