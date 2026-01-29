---
title: "キャッシュ戦略: パフォーマンスの最適化"
---

# キャッシュ戦略: パフォーマンスの最適化

効果的なキャッシュ戦略は、アプリのパフォーマンスとユーザー体験を大きく向上させます。この章では、メモリキャッシュとディスクキャッシュの実装方法を学びます。

## キャッシュの基本原則

### キャッシュ戦略の選択

適切なキャッシュ戦略を選択することが重要です:

```swift
enum CacheStrategy {
    case networkOnly           // ネットワークのみ
    case cacheOnly            // キャッシュのみ
    case cacheFirst           // キャッシュ優先
    case networkFirst         // ネットワーク優先
    case cacheAndNetwork      // キャッシュ表示後に更新
    case cacheWithTimeout     // 期限付きキャッシュ

    static func recommended(for contentType: ContentType) -> CacheStrategy {
        switch contentType {
        case .staticAssets:
            return .cacheFirst
        case .userContent:
            return .cacheAndNetwork
        case .realtime:
            return .networkFirst
        case .criticalData:
            return .networkOnly
        }
    }

    enum ContentType {
        case staticAssets    // 静的アセット
        case userContent     // ユーザーコンテンツ
        case realtime        // リアルタイムデータ
        case criticalData    // 重要なデータ
    }
}
```

### キャッシュポリシー

キャッシュの制限を定義します:

```swift
struct CachePolicy {
    let maxAge: TimeInterval          // 最大キャッシュ期間
    let maxSize: Int                   // 最大キャッシュサイズ（バイト）
    let maxCount: Int                  // 最大アイテム数
    let evictionPolicy: EvictionPolicy // 削除ポリシー

    enum EvictionPolicy {
        case lru   // Least Recently Used
        case lfu   // Least Frequently Used
        case fifo  // First In First Out
        case ttl   // Time To Live
    }

    static let `default` = CachePolicy(
        maxAge: 3600,              // 1時間
        maxSize: 50 * 1024 * 1024, // 50MB
        maxCount: 100,
        evictionPolicy: .lru
    )

    static let aggressive = CachePolicy(
        maxAge: 86400,              // 24時間
        maxSize: 200 * 1024 * 1024, // 200MB
        maxCount: 500,
        evictionPolicy: .lru
    )

    static let conservative = CachePolicy(
        maxAge: 300,               // 5分
        maxSize: 10 * 1024 * 1024, // 10MB
        maxCount: 50,
        evictionPolicy: .ttl
    )
}
```

## メモリキャッシュ

NSCache をベースにした柔軟なメモリキャッシュを実装します:

```swift
class MemoryCache<Key: Hashable, Value> {
    private let cache = NSCache<WrappedKey, Entry>()
    private let dateProvider: () -> Date
    private let entryLifetime: TimeInterval

    init(
        maximumEntryCount: Int = 50,
        maximumTotalCost: Int = 50 * 1024 * 1024,
        entryLifetime: TimeInterval = 3600,
        dateProvider: @escaping () -> Date = Date.init
    ) {
        self.dateProvider = dateProvider
        self.entryLifetime = entryLifetime

        cache.countLimit = maximumEntryCount
        cache.totalCostLimit = maximumTotalCost
    }

    func insert(_ value: Value, forKey key: Key) {
        let date = dateProvider().addingTimeInterval(entryLifetime)
        let entry = Entry(value: value, expirationDate: date)
        cache.setObject(entry, forKey: WrappedKey(key))
    }

    func value(forKey key: Key) -> Value? {
        guard let entry = cache.object(forKey: WrappedKey(key)) else {
            return nil
        }

        guard dateProvider() < entry.expirationDate else {
            removeValue(forKey: key)
            return nil
        }

        return entry.value
    }

    func removeValue(forKey key: Key) {
        cache.removeObject(forKey: WrappedKey(key))
    }

    func removeAllValues() {
        cache.removeAllObjects()
    }

    // MARK: - Supporting Types
    private final class WrappedKey: NSObject {
        let key: Key

        init(_ key: Key) {
            self.key = key
        }

        override var hash: Int {
            key.hashValue
        }

        override func isEqual(_ object: Any?) -> Bool {
            guard let other = object as? WrappedKey else {
                return false
            }
            return key == other.key
        }
    }

    private final class Entry {
        let value: Value
        let expirationDate: Date

        init(value: Value, expirationDate: Date) {
            self.value = value
            self.expirationDate = expirationDate
        }
    }
}
```

## ディスクキャッシュ

ファイルシステムを使用したディスクキャッシュを実装します:

```swift
class DiskCache {
    private let fileManager = FileManager.default
    private let cacheDirectory: URL
    private let policy: CachePolicy
    private let queue = DispatchQueue(label: "com.app.diskcache", qos: .utility)

    init(name: String = "DiskCache", policy: CachePolicy = .default) {
        let cacheURL = fileManager.urls(for: .cachesDirectory, in: .userDomainMask)[0]
        self.cacheDirectory = cacheURL.appendingPathComponent(name)
        self.policy = policy

        createCacheDirectoryIfNeeded()
        Task {
            await cleanupExpiredCache()
        }
    }

    // MARK: - Write
    func save(_ data: Data, forKey key: String) async throws {
        try await withCheckedThrowingContinuation { (continuation: CheckedContinuation<Void, Error>) in
            queue.async {
                do {
                    let fileURL = self.fileURL(forKey: key)
                    try data.write(to: fileURL, options: .atomic)

                    // メタデータを保存
                    try self.saveMetadata(forKey: key, size: data.count)

                    continuation.resume()
                } catch {
                    continuation.resume(throwing: error)
                }
            }
        }
    }

    // MARK: - Read
    func data(forKey key: String) async -> Data? {
        await withCheckedContinuation { continuation in
            queue.async {
                do {
                    // メタデータチェック
                    guard let metadata = try? self.loadMetadata(forKey: key) else {
                        continuation.resume(returning: nil)
                        return
                    }

                    // 期限チェック
                    if Date().timeIntervalSince(metadata.timestamp) > self.policy.maxAge {
                        try? self.remove(forKey: key)
                        continuation.resume(returning: nil)
                        return
                    }

                    // データ読み込み
                    let fileURL = self.fileURL(forKey: key)
                    let data = try Data(contentsOf: fileURL)

                    // アクセス時刻を更新
                    try? self.updateAccessTime(forKey: key)

                    continuation.resume(returning: data)
                } catch {
                    continuation.resume(returning: nil)
                }
            }
        }
    }

    // MARK: - Delete
    func remove(forKey key: String) throws {
        let fileURL = fileURL(forKey: key)
        try fileManager.removeItem(at: fileURL)
        try removeMetadata(forKey: key)
    }

    func removeAll() throws {
        try fileManager.removeItem(at: cacheDirectory)
        createCacheDirectoryIfNeeded()
    }

    // MARK: - Metadata
    private struct CacheMetadata: Codable {
        let key: String
        let size: Int
        let timestamp: Date
        var lastAccessTime: Date
    }

    private func saveMetadata(forKey key: String, size: Int) throws {
        let metadata = CacheMetadata(
            key: key,
            size: size,
            timestamp: Date(),
            lastAccessTime: Date()
        )

        let data = try JSONEncoder().encode(metadata)
        let metadataURL = metadataURL(forKey: key)
        try data.write(to: metadataURL)
    }

    private func loadMetadata(forKey key: String) throws -> CacheMetadata {
        let metadataURL = metadataURL(forKey: key)
        let data = try Data(contentsOf: metadataURL)
        return try JSONDecoder().decode(CacheMetadata.self, from: data)
    }

    private func removeMetadata(forKey key: String) throws {
        let metadataURL = metadataURL(forKey: key)
        try fileManager.removeItem(at: metadataURL)
    }

    private func updateAccessTime(forKey key: String) throws {
        var metadata = try loadMetadata(forKey: key)
        metadata.lastAccessTime = Date()
        let data = try JSONEncoder().encode(metadata)
        try data.write(to: metadataURL(forKey: key))
    }

    // MARK: - Cleanup
    private func cleanupExpiredCache() async {
        do {
            let contents = try fileManager.contentsOfDirectory(
                at: cacheDirectory,
                includingPropertiesForKeys: [.contentModificationDateKey],
                options: .skipsHiddenFiles
            )

            for url in contents where url.pathExtension == "meta" {
                let key = url.deletingPathExtension().lastPathComponent

                if let metadata = try? loadMetadata(forKey: key),
                   Date().timeIntervalSince(metadata.timestamp) > policy.maxAge {
                    try? remove(forKey: key)
                }
            }
        } catch {
            print("キャッシュのクリーンアップに失敗しました: \(error)")
        }
    }

    // MARK: - Helper Methods
    private func createCacheDirectoryIfNeeded() {
        if !fileManager.fileExists(atPath: cacheDirectory.path) {
            try? fileManager.createDirectory(at: cacheDirectory, withIntermediateDirectories: true)
        }
    }

    private func fileURL(forKey key: String) -> URL {
        let filename = key.addingPercentEncoding(withAllowedCharacters: .alphanumerics) ?? key
        return cacheDirectory.appendingPathComponent(filename)
    }

    private func metadataURL(forKey key: String) -> URL {
        fileURL(forKey: key).appendingPathExtension("meta")
    }
}
```

## 画像キャッシュ

2層キャッシュシステムで画像を効率的に管理します:

```swift
actor ImageCacheManager {
    static let shared = ImageCacheManager()

    private let memoryCache: MemoryCache<URL, UIImage>
    private let diskCache: DiskCache
    private var inProgressTasks: [URL: Task<UIImage, Error>] = [:]

    init() {
        self.memoryCache = MemoryCache(
            maximumEntryCount: 100,
            maximumTotalCost: 50 * 1024 * 1024,
            entryLifetime: 3600
        )
        self.diskCache = DiskCache(name: "ImageCache")

        setupMemoryWarningObserver()
    }

    // MARK: - Fetch Image
    func image(for url: URL) async throws -> UIImage {
        // メモリキャッシュをチェック
        if let cached = memoryCache.value(forKey: url) {
            return cached
        }

        // ディスクキャッシュをチェック
        if let data = await diskCache.data(forKey: url.absoluteString),
           let image = UIImage(data: data) {
            memoryCache.insert(image, forKey: url)
            return image
        }

        // 進行中のタスクをチェック
        if let task = inProgressTasks[url] {
            return try await task.value
        }

        // ダウンロード
        let task = Task {
            try await downloadImage(from: url)
        }

        inProgressTasks[url] = task

        do {
            let image = try await task.value
            inProgressTasks.removeValue(forKey: url)
            return image
        } catch {
            inProgressTasks.removeValue(forKey: url)
            throw error
        }
    }

    private func downloadImage(from url: URL) async throws -> UIImage {
        let (data, response) = try await URLSession.shared.data(from: url)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw ImageError.downloadFailed
        }

        guard let image = UIImage(data: data) else {
            throw ImageError.invalidData
        }

        // キャッシュに保存
        memoryCache.insert(image, forKey: url)
        try? await diskCache.save(data, forKey: url.absoluteString)

        return image
    }

    // MARK: - Clear
    func clearMemoryCache() {
        memoryCache.removeAllValues()
    }

    func clearDiskCache() {
        try? diskCache.removeAll()
    }

    // MARK: - Memory Warning
    private func setupMemoryWarningObserver() {
        NotificationCenter.default.addObserver(
            forName: UIApplication.didReceiveMemoryWarningNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            Task {
                await self?.clearMemoryCache()
            }
        }
    }
}

enum ImageError: Error {
    case downloadFailed
    case invalidData
}
```

## API レスポンスキャッシュ

API のレスポンスをキャッシュします:

```swift
class APIResponseCache {
    private let diskCache = DiskCache(name: "APICache")

    func cacheResponse<T: Codable>(_ response: T, for endpoint: Endpoint) async throws {
        let key = cacheKey(for: endpoint)
        let data = try JSONEncoder().encode(response)
        try await diskCache.save(data, forKey: key)
    }

    func cachedResponse<T: Codable>(for endpoint: Endpoint) async -> T? {
        let key = cacheKey(for: endpoint)
        guard let data = await diskCache.data(forKey: key) else {
            return nil
        }

        return try? JSONDecoder().decode(T.self, from: data)
    }

    func removeCachedResponse(for endpoint: Endpoint) throws {
        let key = cacheKey(for: endpoint)
        try diskCache.remove(forKey: key)
    }

    private func cacheKey(for endpoint: Endpoint) -> String {
        let request = try? endpoint.makeRequest()
        let url = request?.url?.absoluteString ?? ""
        let method = request?.httpMethod ?? ""
        return "\(method)_\(url)".sha256Hash()
    }
}

extension String {
    func sha256Hash() -> String {
        // 簡易ハッシュ実装
        return self.data(using: .utf8)?.base64EncodedString() ?? self
    }
}
```

## まとめ

この章では、効率的なキャッシュ戦略を学びました:

- キャッシュ戦略の選択
- メモリキャッシュとディスクキャッシュの実装
- 2層キャッシュシステム
- 画像キャッシュの最適化
- API レスポンスのキャッシュ

次の章では、オフラインファースト設計について学びます。

## 参考リソース

- [URLCache - Apple Developer](https://developer.apple.com/documentation/foundation/urlcache)
- [NSCache - Apple Developer](https://developer.apple.com/documentation/foundation/nscache)
