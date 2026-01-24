---
title: "UserDefaultsベストプラクティス"
---

# UserDefaultsベストプラクティス

UserDefaultsは軽量な設定値の保存に適しています。本章では、UserDefaultsの効果的な活用方法を解説します。

## 型安全なアクセス

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

struct AppSettings {
    @UserDefault(key: "notificationsEnabled", defaultValue: true)
    static var notificationsEnabled: Bool

    @UserDefault(key: "theme", defaultValue: "light")
    static var theme: String
}
```

## Codableの保存

```swift
extension UserDefaults {
    func setCodable<T: Codable>(_ value: T, forKey key: String) {
        if let encoded = try? JSONEncoder().encode(value) {
            set(encoded, forKey: key)
        }
    }

    func codable<T: Codable>(forKey key: String) -> T? {
        guard let data = data(forKey: key) else { return nil }
        return try? JSONDecoder().decode(T.self, from: data)
    }
}
```

## まとめ

UserDefaultsは設定値の保存に適していますが、機密情報はKeychainに保存することが推奨されます。Part 3ではネットワーク通信とデータ永続化を学びました。Part 4では実践パターンを解説します。
