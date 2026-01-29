---
title: "オフラインファースト設計"
---

# オフラインファースト設計

オフラインファースト設計は、ネットワーク接続がない状態でもアプリを使用できるようにするアプローチです。この章では、オフライン対応の実装方法を学びます。

## ネットワーク監視

まず、ネットワークの状態を監視する仕組みを実装します:

```swift
import Network

class NetworkMonitor: ObservableObject {
    static let shared = NetworkMonitor()

    @Published var isConnected = true
    @Published var connectionType: ConnectionType = .unknown

    private let monitor = NWPathMonitor()
    private let queue = DispatchQueue(label: "NetworkMonitor")

    enum ConnectionType {
        case wifi
        case cellular
        case ethernet
        case unknown
    }

    private init() {
        monitor.pathUpdateHandler = { [weak self] path in
            DispatchQueue.main.async {
                self?.isConnected = path.status == .satisfied
                self?.updateConnectionType(path)
            }
        }
        monitor.start(queue: queue)
    }

    private func updateConnectionType(_ path: NWPath) {
        if path.usesInterfaceType(.wifi) {
            connectionType = .wifi
        } else if path.usesInterfaceType(.cellular) {
            connectionType = .cellular
        } else if path.usesInterfaceType(.wiredEthernet) {
            connectionType = .ethernet
        } else {
            connectionType = .unknown
        }
    }
}
```

## オフラインファーストリポジトリ

キャッシュを優先し、バックグラウンドで更新する Repository を実装します:

```swift
class OfflineFirstRepository: UserRepository {
    private let remoteAPI: APIService
    private let localCache: DiskCache
    private let networkMonitor = NetworkMonitor.shared

    init(remoteAPI: APIService) {
        self.remoteAPI = remoteAPI
        self.localCache = DiskCache(name: "OfflineCache")
    }

    func fetchUser(id: Int) async throws -> User {
        let cacheKey = "user_\(id)"

        // オフラインの場合は即座にキャッシュを返す
        if !networkMonitor.isConnected {
            if let data = await localCache.data(forKey: cacheKey),
               let user = try? JSONDecoder().decode(User.self, from: data) {
                return user
            }
            throw NetworkError.networkUnavailable
        }

        // キャッシュを先に返し、バックグラウンドで更新
        if let data = await localCache.data(forKey: cacheKey),
           let cachedUser = try? JSONDecoder().decode(User.self, from: data) {

            Task {
                try? await updateCache(id: id, cacheKey: cacheKey)
            }

            return cachedUser
        }

        // キャッシュがない場合はネットワークから取得
        return try await updateCache(id: id, cacheKey: cacheKey)
    }

    private func updateCache(id: Int, cacheKey: String) async throws -> User {
        let user: User = try await remoteAPI.request(UserEndpoint.getUser(id: id))

        // キャッシュに保存
        if let data = try? JSONEncoder().encode(user) {
            try? await localCache.save(data, forKey: cacheKey)
        }

        return user
    }

    func fetchAllUsers() async throws -> [User] {
        let cacheKey = "users_all"

        // キャッシュから取得
        if let data = await localCache.data(forKey: cacheKey),
           let cachedUsers = try? JSONDecoder().decode([User].self, from: data) {

            // バックグラウンドで更新
            if networkMonitor.isConnected {
                Task {
                    try? await updateUsersCache(cacheKey: cacheKey)
                }
            }

            return cachedUsers
        }

        // キャッシュがない場合はネットワークから取得
        return try await updateUsersCache(cacheKey: cacheKey)
    }

    private func updateUsersCache(cacheKey: String) async throws -> [User] {
        let users: [User] = try await remoteAPI.request(UserEndpoint.getAllUsers)

        // キャッシュに保存
        if let data = try? JSONEncoder().encode(users) {
            try? await localCache.save(data, forKey: cacheKey)
        }

        return users
    }
}
```

## 同期マネージャー

オフライン時の変更を同期する仕組みを実装します:

```swift
class SyncManager: ObservableObject {
    @Published var isSyncing = false
    @Published var pendingChanges = 0

    private let repository: UserRepository
    private let localStore: UserDataStore
    private let networkMonitor = NetworkMonitor.shared
    private var cancellables = Set<AnyCancellable>()

    init(repository: UserRepository, localStore: UserDataStore) {
        self.repository = repository
        self.localStore = localStore

        setupNetworkObserver()
    }

    private func setupNetworkObserver() {
        networkMonitor.$isConnected
            .sink { [weak self] isConnected in
                if isConnected {
                    Task { await self?.sync() }
                }
            }
            .store(in: &cancellables)
    }

    func sync() async {
        guard networkMonitor.isConnected else { return }

        isSyncing = true
        defer { isSyncing = false }

        do {
            // サーバーから最新データを取得
            let remoteUsers = try await repository.fetchAllUsers()

            // ローカルデータベースを更新
            for user in remoteUsers {
                try? await localStore.upsertUser(user)
            }

            // ローカルの未送信変更を同期
            let pendingUsers = try await localStore.fetchPendingUsers()
            for user in pendingUsers {
                try? await repository.updateUser(id: user.id, UpdateUserRequest(
                    name: user.name,
                    email: user.email
                ))
                try? await localStore.markAsSynced(user)
            }

            pendingChanges = 0
        } catch {
            print("同期に失敗しました: \(error)")
        }
    }

    func addPendingChange() {
        pendingChanges += 1
    }
}

// UserDataStore の拡張
extension UserDataStore {
    func upsertUser(_ user: User) async throws {
        if let existing = try? await fetch(byId: user.id) {
            try await update(existing, name: user.name, email: user.email)
        } else {
            _ = try await create(name: user.name, email: user.email)
        }
    }

    func fetchPendingUsers() async throws -> [User] {
        // 未同期のユーザーを取得
        // 実装はデータベースに依存
        []
    }

    func markAsSynced(_ user: User) async throws {
        // 同期済みフラグを立てる
        // 実装はデータベースに依存
    }
}
```

## 楽観的更新

ユーザー操作を即座に反映し、バックグラウンドで同期します:

```swift
@MainActor
class OptimisticUpdateViewModel: ObservableObject {
    @Published var users: [User] = []
    @Published var error: String?

    private let repository: UserRepository
    private let syncManager: SyncManager

    init(repository: UserRepository, syncManager: SyncManager) {
        self.repository = repository
        self.syncManager = syncManager
    }

    func createUser(name: String, email: String) async {
        // 仮のユーザーを即座に追加
        let temporaryUser = User(
            id: -1, // 仮のID
            name: name,
            email: email,
            createdAt: Date()
        )
        users.insert(temporaryUser, at: 0)

        do {
            // バックグラウンドで実際に作成
            let newUser = try await repository.createUser(CreateUserRequest(
                name: name,
                email: email,
                password: ""
            ))

            // 仮のユーザーを実際のユーザーで置き換え
            if let index = users.firstIndex(where: { $0.id == -1 }) {
                users[index] = newUser
            }
        } catch {
            // エラーの場合は仮のユーザーを削除
            users.removeAll { $0.id == -1 }
            self.error = "ユーザーの作成に失敗しました"
            syncManager.addPendingChange()
        }
    }

    func deleteUser(_ user: User) async {
        // UIから即座に削除
        users.removeAll { $0.id == user.id }

        do {
            // バックグラウンドで削除
            try await repository.deleteUser(id: user.id)
        } catch {
            // エラーの場合は復元
            users.append(user)
            self.error = "ユーザーの削除に失敗しました"
            syncManager.addPendingChange()
        }
    }
}
```

## オフライン状態の UI 表示

ユーザーにオフライン状態を知らせます:

```swift
struct OfflineIndicatorView: View {
    @ObservedObject var networkMonitor = NetworkMonitor.shared

    var body: some View {
        if !networkMonitor.isConnected {
            HStack {
                Image(systemName: "wifi.slash")
                Text("オフライン")
            }
            .font(.caption)
            .foregroundColor(.white)
            .padding(.horizontal, 12)
            .padding(.vertical, 6)
            .background(Color.orange)
            .cornerRadius(20)
            .padding(.top, 8)
        }
    }
}

struct ContentView: View {
    var body: some View {
        VStack(spacing: 0) {
            OfflineIndicatorView()

            List {
                // コンテンツ
            }
        }
    }
}
```

## 同期状態の表示

同期の進行状況を表示します:

```swift
struct SyncStatusView: View {
    @ObservedObject var syncManager: SyncManager

    var body: some View {
        Group {
            if syncManager.isSyncing {
                HStack {
                    ProgressView()
                        .scaleEffect(0.8)
                    Text("同期中...")
                        .font(.caption)
                }
                .padding(.horizontal, 12)
                .padding(.vertical, 6)
                .background(Color.blue.opacity(0.1))
                .cornerRadius(20)
            } else if syncManager.pendingChanges > 0 {
                HStack {
                    Image(systemName: "arrow.clockwise")
                    Text("\(syncManager.pendingChanges)件の変更が未同期")
                        .font(.caption)
                }
                .padding(.horizontal, 12)
                .padding(.vertical, 6)
                .background(Color.yellow.opacity(0.2))
                .cornerRadius(20)
            }
        }
    }
}
```

## データの競合解決

同じデータが異なる場所で変更された場合の処理:

```swift
enum ConflictResolutionStrategy {
    case serverWins      // サーバーの変更を優先
    case clientWins      // クライアントの変更を優先
    case newerWins       // より新しい変更を優先
    case manual          // 手動で解決
}

class ConflictResolver {
    func resolve(
        local: User,
        remote: User,
        strategy: ConflictResolutionStrategy
    ) -> User {
        switch strategy {
        case .serverWins:
            return remote

        case .clientWins:
            return local

        case .newerWins:
            return local.createdAt > remote.createdAt ? local : remote

        case .manual:
            // 手動解決のための UI を表示
            return local
        }
    }
}
```

## まとめ

この章では、オフラインファースト設計の実装方法を学びました:

- ネットワーク監視
- オフラインファーストリポジトリ
- 同期マネージャー
- 楽観的更新
- オフライン状態の UI 表示
- データの競合解決

次の章では、ベストプラクティスとよくある問題の解決策について学びます。

## 参考リソース

- [Network Framework - Apple Developer](https://developer.apple.com/documentation/network)
- [Offline First Design](https://offlinefirst.org/)
