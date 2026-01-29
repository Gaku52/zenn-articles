---
title: "【2026年版】iOS ネットワーク通信・データ永続化 完全ガイドを公開しました"
emoji: "📡"
type: "tech"
topics: ["ios", "swift", "ネットワーク", "coredata", "realm"]
published: false
---

# iOS ネットワーク通信・データ永続化 完全ガイド 2026 を公開しました

## iOS開発でこんな悩みありませんか？

「URLSessionの使い方、これで合ってる？」
「Core DataとRealmどっち使えばいい？」
「API通信のエラーハンドリング、どこまでやれば...」
「オフライン対応ってどうやるの？」

iOS開発において、ネットワーク通信とデータ永続化は避けて通れない重要な要素ですが、多くの開発者が適切な実装方法に悩んでいます。

**よくある課題：**

- API通信の実装パターンが定まらない
- データ永続化の選択基準がわからない
- エラーハンドリングが複雑で保守しづらい
- オフライン対応の実装方法が不明

そこで、**URLSessionを使った型安全なAPI通信からCore Data、Realm、Keychainまで、iOS開発におけるネットワーク通信とデータ永続化を体系的にまとめた完全ガイド**を執筆しました。

https://zenn.dev/gaku/books/ios-networking-data-guide-2026

## なぜネットワーク通信・データ永続化で挫折するのか

### 1. 公式ドキュメントだけでは実務レベルに到達できない

Appleの公式ドキュメントは基礎的な使い方は説明していますが、「実務でどう設計するか」「どのパターンを選ぶべきか」は書かれていません。

**よくある失敗:**
- URLSessionを直接ViewControllerで使ってしまう
- エラーハンドリングが不十分
- データの永続化手法を適切に選べない
- オフライン対応を後回しにして実装が複雑化

### 2. データ永続化の選択肢が多すぎる

iOSには複数のデータ永続化手法があり、それぞれ適切な用途が異なります：

```
UserDefaults、Keychain、Core Data、Realm、SQLite、FileManager...
```

**どれをいつ使えばいいのか？** この判断基準を理解していないと、後から大規模なリファクタリングが必要になることがあります。

### 3. エラーハンドリングが複雑になりがち

ネットワーク通信では様々なエラーが発生する可能性があります：

**考えられるエラー:**
- ネットワーク接続エラー
- タイムアウト
- サーバーエラー（4xx, 5xx）
- データのパースエラー
- 認証エラー

これらを適切にハンドリングしないと、ユーザーに不親切なアプリになってしまいます。

## よくある3つの間違い

本書で扱う内容から、特によくある間違いを3つ紹介します。

### 間違い1: URLSessionを直接ViewControllerで使っている

**❌ 悪い例:**

```swift
class UserListViewController: UIViewController {
    var users: [User] = []

    func loadUsers() {
        let url = URL(string: "https://api.example.com/users")!

        URLSession.shared.dataTask(with: url) { data, response, error in
            guard let data = data else { return }

            do {
                let users = try JSONDecoder().decode([User].self, from: data)
                DispatchQueue.main.async {
                    self.users = users
                    self.tableView.reloadData()
                }
            } catch {
                print("Error: \(error)")
            }
        }.resume()
    }
}
```

**何が問題？**
- ビジネスロジックがViewControllerに集中
- テストが困難
- エラーハンドリングが不十分
- 再利用性が低い
- 複数の画面で同じコードを書くことになる

**✅ 正しい例: Repository パターン**

```swift
// API Client（通信層）
protocol APIClient {
    func request<T: Decodable>(endpoint: Endpoint) async throws -> T
}

class DefaultAPIClient: APIClient {
    func request<T: Decodable>(endpoint: Endpoint) async throws -> T {
        let request = try endpoint.makeURLRequest()
        let (data, response) = try await URLSession.shared.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }

        guard (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.httpError(statusCode: httpResponse.statusCode)
        }

        do {
            return try JSONDecoder().decode(T.self, from: data)
        } catch {
            throw NetworkError.decodingError(error)
        }
    }
}

// Repository（データアクセス層）
protocol UserRepository {
    func fetchUsers() async throws -> [User]
    func fetchUser(id: String) async throws -> User
}

class DefaultUserRepository: UserRepository {
    private let apiClient: APIClient

    init(apiClient: APIClient) {
        self.apiClient = apiClient
    }

    func fetchUsers() async throws -> [User] {
        return try await apiClient.request(endpoint: .users)
    }

    func fetchUser(id: String) async throws -> User {
        return try await apiClient.request(endpoint: .user(id: id))
    }
}

// ViewModel（プレゼンテーション層）
@MainActor
class UserListViewModel: ObservableObject {
    @Published var users: [User] = []
    @Published var isLoading = false
    @Published var error: Error?

    private let repository: UserRepository

    init(repository: UserRepository) {
        self.repository = repository
    }

    func loadUsers() async {
        isLoading = true
        error = nil

        do {
            users = try await repository.fetchUsers()
        } catch {
            self.error = error
        }

        isLoading = false
    }
}
```

**結果:**
- 責任が明確に分離され、保守性が向上
- Repositoryをモックに差し替えて単体テスト可能
- 複数の画面で同じRepositoryを再利用
- エラーハンドリングが一元化

### 間違い2: 認証トークンをUserDefaultsに保存している

**❌ 悪い例:**

```swift
class AuthManager {
    func saveToken(_ token: String) {
        UserDefaults.standard.set(token, forKey: "authToken")
    }

    func getToken() -> String? {
        return UserDefaults.standard.string(forKey: "authToken")
    }
}
```

**何が問題？**
- UserDefaultsは暗号化されていない
- plistファイルとして平文で保存される
- デバイスのバックアップに含まれる
- Jailbreak端末では簡単に読み取られる可能性がある
- セキュリティリスクが高い

**✅ 正しい例: Keychainを使用**

```swift
import Security

class KeychainManager {
    enum KeychainError: Error {
        case itemNotFound
        case duplicateItem
        case unexpectedStatus(OSStatus)
    }

    func save(key: String, data: Data) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlock
        ]

        // 既存のアイテムを削除
        SecItemDelete(query as CFDictionary)

        // 新しいアイテムを追加
        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw KeychainError.unexpectedStatus(status)
        }
    }

    func load(key: String) throws -> Data {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess else {
            throw KeychainError.itemNotFound
        }

        guard let data = result as? Data else {
            throw KeychainError.unexpectedStatus(status)
        }

        return data
    }

    func delete(key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key
        ]

        let status = SecItemDelete(query as CFDictionary)

        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.unexpectedStatus(status)
        }
    }
}

// 使用例
class SecureAuthManager {
    private let keychain = KeychainManager()
    private let tokenKey = "authToken"

    func saveToken(_ token: String) throws {
        let data = token.data(using: .utf8)!
        try keychain.save(key: tokenKey, data: data)
    }

    func getToken() throws -> String? {
        guard let data = try? keychain.load(key: tokenKey),
              let token = String(data: data, encoding: .utf8) else {
            return nil
        }
        return token
    }

    func deleteToken() throws {
        try keychain.delete(key: tokenKey)
    }
}
```

**結果:**
- トークンが暗号化されてKeychainに保存される
- iOS標準のセキュリティ保護が適用される
- デバイスロック時のアクセス制御が可能
- セキュリティリスクの大幅な軽減

### 間違い3: Core DataとRealmの選択基準がわからない

多くの開発者が「なんとなく」でデータ永続化の手法を選んでいます。

**選択基準の比較:**

| 手法 | 適したケース | 学習コスト | パフォーマンス | 複雑なクエリ |
|------|----------|----------|------------|-----------|
| UserDefaults | 設定値、小さなデータ | 低 | 高速 | ❌ |
| Keychain | 認証情報、秘密鍵 | 中 | 高速 | ❌ |
| Core Data | 複雑なリレーション、大量データ | 高 | 高速（適切な設計時） | ✅ |
| Realm | シンプルなオブジェクト保存 | 低 | 非常に高速 | △ |
| FileManager | 画像、動画、大容量ファイル | 低 | 中速 | ❌ |

**Core Dataを選ぶべきケース:**

```swift
// 複雑なリレーションがある場合
// 例: SNSアプリ

User ─┬─ Post ─── Comment
      │          ├─ Like
      │          └─ Tag
      │
      ├─ Follow
      └─ Message

// Core Dataが適している理由：
// - 正規化されたスキーマ
// - 複雑なリレーション
// - NSPredicateによる強力なクエリ
// - iCloudとの同期（CloudKit統合）
```

**Realmを選ぶべきケース:**

```swift
// シンプルなオブジェクトの保存が中心
// 例: ToDoアプリ、メモアプリ

import RealmSwift

class Todo: Object, ObjectKeyIdentifiable {
    @Persisted(primaryKey: true) var id: UUID
    @Persisted var title: String
    @Persisted var isCompleted: Bool
    @Persisted var createdAt: Date
    @Persisted var category: String
}

// Realmが適している理由：
// - シンプルなAPI
// - 学習コストが低い
// - リアクティブな更新（自動UI反映）
// - 高速な書き込み/読み込み
```

## 本の内容を一部公開：API通信アーキテクチャ

本書では、このような実践的な内容を全12章にわたって解説していますが、ここでは特に重要なAPI通信アーキテクチャの設計を紹介します。

### レイヤー分離されたアーキテクチャ

```swift
// 1. Model（データ層）
struct User: Codable, Identifiable {
    let id: String
    let name: String
    let email: String
    let avatarURL: URL?
}

// 2. Endpoint（APIエンドポイント定義）
enum Endpoint {
    case users
    case user(id: String)
    case createUser(name: String, email: String)
    case updateUser(id: String, name: String)

    var path: String {
        switch self {
        case .users:
            return "/users"
        case .user(let id):
            return "/users/\(id)"
        case .createUser:
            return "/users"
        case .updateUser(let id, _):
            return "/users/\(id)"
        }
    }

    var method: HTTPMethod {
        switch self {
        case .users, .user:
            return .get
        case .createUser:
            return .post
        case .updateUser:
            return .put
        }
    }

    func makeURLRequest() throws -> URLRequest {
        guard let url = URL(string: "https://api.example.com\(path)") else {
            throw NetworkError.invalidURL
        }

        var request = URLRequest(url: url)
        request.httpMethod = method.rawValue
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        // Body
        switch self {
        case .createUser(let name, let email):
            let body = ["name": name, "email": email]
            request.httpBody = try? JSONEncoder().encode(body)
        case .updateUser(_, let name):
            let body = ["name": name]
            request.httpBody = try? JSONEncoder().encode(body)
        default:
            break
        }

        return request
    }
}

// 3. Network Error（エラー定義）
enum NetworkError: LocalizedError {
    case invalidURL
    case invalidResponse
    case httpError(statusCode: Int)
    case decodingError(Error)
    case networkError(Error)

    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "無効なURLです"
        case .invalidResponse:
            return "サーバーからの応答が無効です"
        case .httpError(let statusCode):
            return "HTTPエラー: \(statusCode)"
        case .decodingError:
            return "データの解析に失敗しました"
        case .networkError(let error):
            return "ネットワークエラー: \(error.localizedDescription)"
        }
    }
}

// 4. API Client（通信層）
protocol APIClient {
    func request<T: Decodable>(endpoint: Endpoint) async throws -> T
}

class DefaultAPIClient: APIClient {
    private let session: URLSession

    init(session: URLSession = .shared) {
        self.session = session
    }

    func request<T: Decodable>(endpoint: Endpoint) async throws -> T {
        let request = try endpoint.makeURLRequest()

        do {
            let (data, response) = try await session.data(for: request)

            guard let httpResponse = response as? HTTPURLResponse else {
                throw NetworkError.invalidResponse
            }

            guard (200...299).contains(httpResponse.statusCode) else {
                throw NetworkError.httpError(statusCode: httpResponse.statusCode)
            }

            do {
                return try JSONDecoder().decode(T.self, from: data)
            } catch {
                throw NetworkError.decodingError(error)
            }
        } catch let error as NetworkError {
            throw error
        } catch {
            throw NetworkError.networkError(error)
        }
    }
}

// 5. Repository（データアクセス層）
protocol UserRepository {
    func fetchUsers() async throws -> [User]
    func fetchUser(id: String) async throws -> User
    func createUser(name: String, email: String) async throws -> User
}

class DefaultUserRepository: UserRepository {
    private let apiClient: APIClient

    init(apiClient: APIClient) {
        self.apiClient = apiClient
    }

    func fetchUsers() async throws -> [User] {
        return try await apiClient.request(endpoint: .users)
    }

    func fetchUser(id: String) async throws -> User {
        return try await apiClient.request(endpoint: .user(id: id))
    }

    func createUser(name: String, email: String) async throws -> User {
        return try await apiClient.request(endpoint: .createUser(name: name, email: email))
    }
}
```

**このアーキテクチャの利点:**

1. **責任の分離**: 各レイヤーが明確な責任を持つ
2. **テストが容易**: 各レイヤーを独立してテスト可能
3. **再利用性**: Repositoryは複数のViewModelで使い回せる
4. **保守性**: 変更の影響範囲が限定される
5. **型安全**: Swiftの型システムを最大限活用

## データ永続化の選択フローチャート

本書の第5章では、データ永続化手法の選択基準を詳しく解説していますが、ここでは簡単な判断フローを紹介します。

```
データ永続化の選択

Q1: 保存するデータは機密情報？
YES → Keychain を使用
NO  → Q2へ

Q2: 保存するデータは設定値や小さなデータ（<1MB）？
YES → UserDefaults を使用
NO  → Q3へ

Q3: 保存するデータは画像や動画などのメディアファイル？
YES → FileManager を使用
NO  → Q4へ

Q4: 複雑なリレーションやクエリが必要？
YES → Core Data を使用
NO  → Q5へ

Q5: シンプルなオブジェクト保存でリアクティブな更新が欲しい？
YES → Realm を使用
NO  → SQLite.swift を検討
```

### 実装例：キャッシュ戦略

```swift
// キャッシュ付きRepository
class CachedUserRepository: UserRepository {
    private let apiClient: APIClient
    private let cache: UserCache

    init(apiClient: APIClient, cache: UserCache) {
        self.apiClient = apiClient
        self.cache = cache
    }

    func fetchUsers() async throws -> [User] {
        // キャッシュをチェック
        if let cachedUsers = try? cache.load(),
           !cache.isExpired() {
            return cachedUsers
        }

        // APIから取得
        let users = try await apiClient.request(endpoint: .users)

        // キャッシュに保存
        try? cache.save(users)

        return users
    }
}

// キャッシュ管理
class UserCache {
    private let cacheKey = "users_cache"
    private let expirationKey = "users_cache_expiration"
    private let cacheLifetime: TimeInterval = 300 // 5分

    func save(_ users: [User]) throws {
        let data = try JSONEncoder().encode(users)
        UserDefaults.standard.set(data, forKey: cacheKey)
        UserDefaults.standard.set(Date().timeIntervalSince1970, forKey: expirationKey)
    }

    func load() throws -> [User]? {
        guard let data = UserDefaults.standard.data(forKey: cacheKey) else {
            return nil
        }
        return try JSONDecoder().decode([User].self, from: data)
    }

    func isExpired() -> Bool {
        let savedTime = UserDefaults.standard.double(forKey: expirationKey)
        let elapsed = Date().timeIntervalSince1970 - savedTime
        return elapsed > cacheLifetime
    }

    func clear() {
        UserDefaults.standard.removeObject(forKey: cacheKey)
        UserDefaults.standard.removeObject(forKey: expirationKey)
    }
}
```

**期待される効果:**
- ネットワーク通信の削減
- アプリの応答速度向上
- オフライン時の部分的な動作
- サーバー負荷の軽減

## 本書で学べる全内容

この記事で紹介した内容は、本書の一部に過ぎません。全12章で、以下の内容を網羅しています。

### Part 1: ネットワーク通信（6章）

- **Chapter 1: URLSession基礎**
  - URLSessionの基本構成
  - async/awaitによる非同期処理
  - データタスク、ダウンロードタスク、アップロードタスク
  - セッション設定とカスタマイズ

- **Chapter 2: API通信アーキテクチャ**
  - Repositoryパターン
  - Endpointの設計
  - 型安全なAPI Client
  - レイヤー分離

- **Chapter 3: エラーハンドリング**
  - ネットワークエラーの種類
  - Resultパターン
  - リトライロジック
  - ユーザーへのエラー表示

- **Chapter 4: WebSocketとリアルタイム通信**
  - WebSocketの基礎
  - チャット機能の実装
  - 接続管理と再接続
  - Server-Sent Events (SSE)

- **Chapter 5: 認証とセキュリティ**
  - OAuth 2.0
  - JWT
  - リフレッシュトークン
  - Certificate Pinning

- **Chapter 6: ネットワーク監視と最適化**
  - Network.framework
  - 接続状態の監視
  - 通信の最適化
  - バックグラウンド通信

### Part 2: データ永続化（6章）

- **Chapter 7: UserDefaultsとFileManager**
  - UserDefaultsの使い方と制約
  - FileManagerによるファイル操作
  - ディレクトリ構造
  - ファイルの保存と読み込み

- **Chapter 8: Keychainセキュア保存**
  - Keychainの基礎
  - 認証情報の保存
  - バイオメトリクス認証
  - アクセス制御

- **Chapter 9: Core Data基礎**
  - Core Dataの概念
  - スタックのセットアップ
  - エンティティとリレーション
  - CRUD操作

- **Chapter 10: Core Data応用**
  - NSPredicate
  - NSSortDescriptor
  - Batch操作
  - パフォーマンス最適化

- **Chapter 11: Realm**
  - Realmの基礎
  - オブジェクトの定義
  - クエリとフィルタ
  - リアクティブな更新

- **Chapter 12: キャッシュ戦略とオフライン対応**
  - キャッシュパターン
  - Offline-First設計
  - 同期戦略
  - コンフリクト解決

## こんな方におすすめ

- **iOS開発初心者**（API通信の基礎から学べます）
- **ネットワーク通信の設計を学びたい方**
- **データ永続化の選択基準を知りたい方**
- **Core DataやRealmの実践的な使い方を学びたい方**
- **オフライン対応アプリを作りたい方**
- **アーキテクチャパターンを習得したい方**

## 価格

**1,000円**

一般的な技術書（3,000円〜5,000円）の1/3〜1/5の価格で、体系的な完全ガイドが手に入ります。

## サンプル

導入部分とURLSession基礎の章は無料で読めます。ぜひご覧ください！

https://zenn.dev/gaku/books/ios-networking-data-guide-2026

## 本書が必要な5つのサイン

以下のどれか1つでも当てはまるなら、本書があなたの課題を解決します。

### サイン1: URLSessionを直接ViewControllerで使っている

**症状:**
- 通信処理がViewControllerに書かれている
- 同じようなコードを複数の場所で書いている
- テストが書けない（書きづらい）

**本書の解決策:**
- Repositoryパターンの完全実装
- レイヤー分離の設計指針
- テスタブルなコードの書き方

**期待される成果:** コードの再利用性向上、テスト可能なアーキテクチャ

### サイン2: エラーハンドリングが不十分

**症状:**
- エラー時に「Error」とだけ表示される
- ネットワークエラーと認証エラーの区別ができない
- リトライロジックがない

**本書の解決策:**
- エラーの分類と適切なハンドリング
- ユーザーフレンドリーなエラー表示
- 自動リトライの実装

**期待される成果:** ユーザー体験の向上、問題の早期発見

### サイン3: データ永続化の選択に迷う

**症状:**
- Core DataとRealmのどちらを選ぶべきかわからない
- UserDefaultsに大量のデータを保存している
- 認証トークンをUserDefaultsに保存している

**本書の解決策:**
- データ永続化手法の比較表
- 用途別の選択基準
- それぞれの実装パターン

**期待される成果:** 適切な技術選択、セキュリティの向上

### サイン4: オフライン対応ができていない

**症状:**
- ネットワークがないと何もできない
- 一度取得したデータを再度取得している
- キャッシュの仕組みがない

**本書の解決策:**
- キャッシュ戦略の設計
- Offline-Firstアーキテクチャ
- 同期戦略の実装

**期待される成果:** オフライン対応、パフォーマンス向上

### サイン5: Core Dataが難しくて挫折した

**症状:**
- Core Dataのセットアップで詰まった
- リレーションの設定がわからない
- パフォーマンスが出ない

**本書の解決策:**
- ステップバイステップのセットアップ
- リレーションの実践的な使い方
- パフォーマンス最適化のベストプラクティス

**期待される成果:** Core Dataの実践的な習得

## 本書で得られる3つの具体的なスキル

### スキル1: 保守性の高いネットワーク層を設計できる

**習得できること:**
- Repositoryパターンでの設計
- 型安全なAPI Client
- エラーハンドリングの実装
- テスタブルなコード

**実務での価値:**
- 新規機能の追加が容易
- バグの早期発見
- チーム開発での保守性向上

### スキル2: 適切なデータ永続化手法を選択・実装できる

**習得できること:**
- UserDefaults、Keychain、Core Data、Realmの使い分け
- セキュアなデータ保存
- リレーションの設計
- クエリの最適化

**実務での価値:**
- 要件に応じた最適な技術選択
- セキュリティリスクの軽減
- パフォーマンスの向上

### スキル3: オフライン対応アプリを実装できる

**習得できること:**
- キャッシュ戦略の設計
- ネットワーク状態の監視
- 同期処理の実装
- コンフリクト解決

**実務での価値:**
- ユーザー体験の向上
- サーバー負荷の軽減
- 通信コストの削減

## 想定される学習効果

本書の内容を実践することで、以下のような効果が期待できます：

### アーキテクチャ面
- レイヤー分離された保守性の高い設計
- テストカバレッジの向上
- 変更の影響範囲の限定

### パフォーマンス面
- 不要な通信の削減
- キャッシュによる応答速度の向上
- バッテリー消費の最適化

### セキュリティ面
- 機密情報の適切な保存
- 通信の暗号化
- 認証・認可の実装

### ユーザー体験面
- オフライン対応
- 適切なエラー表示
- スムーズな動作

## さいごに

ネットワーク通信とデータ永続化は、iOSアプリ開発の基礎であり、同時に奥が深い領域です。

この本は、私自身がiOS開発で経験した試行錯誤、学んだベストプラクティス、実務で培ったノウハウを全て詰め込みました。

- **体系的な知識**: URLSessionの基礎から高度なアーキテクチャまで
- **実践的なコード**: すぐに使える実装例が豊富
- **設計指針**: データ永続化の選択基準と使い分け
- **初心者から中級者まで**: レベルに応じて学べる

この本が、皆さんのiOS開発をより堅牢で、より保守性の高いものにする一助となれば幸いです。

ご質問やフィードバックがあれば、ぜひコメント欄でお聞かせください！

---

**関連リンク**

- [本書の詳細・購入はこちら](https://zenn.dev/gaku/books/ios-networking-data-guide-2026)
