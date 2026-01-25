---
title: "第14章 Swiftレビューガイド"
---

# 第14章 Swiftレビューガイド

本章ではSwiftコードレビューにおける重要なチェックポイントと、具体的なレビュー観点について詳しく解説します。Swiftは型安全性とパフォーマンスを重視した言語であり、特にOptional型の扱いやメモリ管理が重要なレビューポイントとなります。

## Swiftコードレビューの特性

SwiftはAppleのプラットフォーム向けアプリケーション開発に最適化された言語です。レビューでは以下の観点を重点的に確認することが推奨されます。

### レビューの重点領域

- Optional型の適切な扱い（if let, guard let）
- メモリ管理（ARC、weak, unowned）
- プロトコル指向プログラミング
- エラーハンドリング（Result型、throws）
- Swiftコーディング規約の遵守

## Optional型の適切な扱い

SwiftのOptional型は、値が存在しない可能性を型システムで表現します。適切なアンラップ方法を選択することが、安全なコードの基本です。

### 悪い例: 強制アンラップの多用

```swift
// 危険: 強制アンラップでクラッシュの可能性
func getUserName(userId: String) -> String {
    let user = database.findUser(id: userId)!  // クラッシュの可能性
    let profile = user.profile!  // クラッシュの可能性
    return profile.name!  // クラッシュの可能性
}

// 危険: 暗黙的アンラップの不適切な使用
class UserViewController: UIViewController {
    var userName: String!  // 初期化前にアクセスされる可能性

    func displayName() {
        nameLabel.text = userName  // クラッシュの可能性
    }
}
```

### 良い例: 安全なOptionalハンドリング

```swift
// if letによる安全なアンラップ
func getUserName(userId: String) -> String? {
    if let user = database.findUser(id: userId),
       let profile = user.profile,
       let name = profile.name {
        return name
    }
    return nil
}

// guard letによる早期リターン
func processUser(userId: String) throws {
    guard let user = database.findUser(id: userId) else {
        throw UserError.notFound
    }

    guard let profile = user.profile else {
        throw UserError.profileNotFound
    }

    guard let name = profile.name else {
        throw UserError.nameNotFound
    }

    // 以降は非Optional値として扱える
    updateUI(with: name)
}

// Optional chainingとnil coalescing演算子
func getDisplayName(userId: String) -> String {
    return database.findUser(id: userId)?.profile?.name ?? "Unknown User"
}
```

guard letを使うことで、関数の本質的なロジックをネストせずに記述でき、可読性が向上します。

## メモリ管理: 参照サイクルの防止

SwiftはARC（Automatic Reference Counting）でメモリ管理を行いますが、参照サイクルによるメモリリークに注意が必要です。

### 悪い例: 参照サイクルが発生するコード

```swift
class User {
    var name: String
    var profile: Profile?

    init(name: String) {
        self.name = name
    }
}

class Profile {
    var bio: String
    var user: User  // 強参照によるサイクル

    init(bio: String, user: User) {
        self.bio = bio
        self.user = user  // メモリリークの原因
    }
}

// クロージャでの参照サイクル
class ViewController: UIViewController {
    var completionHandler: (() -> Void)?

    func setupHandler() {
        completionHandler = {
            self.updateUI()  // 強参照サイクル
        }
    }

    func updateUI() {
        // UI更新処理
    }
}
```

### 良い例: weak/unownedを使った参照サイクルの防止

```swift
class User {
    var name: String
    weak var profile: Profile?  // weak参照

    init(name: String) {
        self.name = name
    }
}

class Profile {
    var bio: String
    weak var user: User?  // weak参照でサイクルを防ぐ

    init(bio: String, user: User) {
        self.bio = bio
        self.user = user
    }
}

// クロージャでのキャプチャリスト
class ViewController: UIViewController {
    var completionHandler: (() -> Void)?

    func setupHandler() {
        completionHandler = { [weak self] in
            self?.updateUI()  // weak selfで参照サイクルを防ぐ
        }
    }

    // unownedの使用例（selfが必ず存在する場合）
    func setupTimer() {
        Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [unowned self] _ in
            self.updateTime()  // selfが解放されることがないと確信できる場合
        }
    }

    func updateUI() {
        // UI更新処理
    }

    func updateTime() {
        // 時刻更新処理
    }
}
```

weak参照とunowned参照の使い分け:
- weak: 参照先がnilになる可能性がある場合（Optional型になる）
- unowned: 参照先が必ず存在すると確信できる場合（非Optional型）

## プロトコル指向プログラミング

Swiftはプロトコル指向プログラミングを推奨しています。継承よりもプロトコルを使うことで、柔軟で再利用可能なコードを書けます。

### 悪い例: 継承に依存した設計

```swift
// 基底クラスに依存
class BaseNetworkService {
    func request(url: URL) -> Data? {
        // 共通処理
        return nil
    }
}

class UserService: BaseNetworkService {
    func getUser(id: String) -> User? {
        guard let data = request(url: URL(string: "/users/\(id)")!) else {
            return nil
        }
        return parseUser(data)
    }

    private func parseUser(_ data: Data) -> User? {
        // パース処理
        return nil
    }
}
```

### 良い例: プロトコルを活用した設計

```swift
// プロトコル定義
protocol NetworkServiceProtocol {
    func request(url: URL) async throws -> Data
}

protocol UserServiceProtocol {
    func getUser(id: String) async throws -> User
}

// デフォルト実装（Protocol Extension）
extension NetworkServiceProtocol {
    func request(url: URL) async throws -> Data {
        let (data, response) = try await URLSession.shared.data(from: url)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.invalidResponse
        }

        return data
    }
}

// 実装
struct UserService: UserServiceProtocol, NetworkServiceProtocol {
    func getUser(id: String) async throws -> User {
        let url = URL(string: "https://api.example.com/users/\(id)")!
        let data = try await request(url: url)
        return try JSONDecoder().decode(User.self, from: data)
    }
}

// モックの作成が容易
struct MockUserService: UserServiceProtocol {
    func getUser(id: String) async throws -> User {
        return User(id: id, name: "Mock User")
    }
}
```

プロトコルを使うことで、依存性の注入やテストが容易になります。

## Result型によるエラーハンドリング

Swift 5で導入されたResult型を使うことで、成功と失敗を明示的に扱えます。

### 悪い例: エラー情報の欠落

```swift
// エラー情報が失われる
func fetchUserData(userId: String, completion: @escaping (User?) -> Void) {
    URLSession.shared.dataTask(with: url) { data, response, error in
        guard let data = data else {
            completion(nil)  // なぜ失敗したかわからない
            return
        }

        let user = try? JSONDecoder().decode(User.self, from: data)
        completion(user)  // パースエラーも無視される
    }.resume()
}
```

### 良い例: Result型で明示的なエラーハンドリング

```swift
enum NetworkError: Error {
    case invalidURL
    case noData
    case decodingFailed(Error)
    case httpError(statusCode: Int)
}

// Result型を使った明示的なエラーハンドリング
func fetchUserData(userId: String) async -> Result<User, NetworkError> {
    guard let url = URL(string: "https://api.example.com/users/\(userId)") else {
        return .failure(.invalidURL)
    }

    do {
        let (data, response) = try await URLSession.shared.data(from: url)

        guard let httpResponse = response as? HTTPURLResponse else {
            return .failure(.noData)
        }

        guard (200...299).contains(httpResponse.statusCode) else {
            return .failure(.httpError(statusCode: httpResponse.statusCode))
        }

        do {
            let user = try JSONDecoder().decode(User.self, from: data)
            return .success(user)
        } catch {
            return .failure(.decodingFailed(error))
        }
    } catch {
        return .failure(.noData)
    }
}

// 使用例
Task {
    let result = await fetchUserData(userId: "123")

    switch result {
    case .success(let user):
        print("User: \(user.name)")
    case .failure(let error):
        switch error {
        case .invalidURL:
            print("Invalid URL")
        case .noData:
            print("No data received")
        case .decodingFailed(let decodingError):
            print("Decoding failed: \(decodingError)")
        case .httpError(let statusCode):
            print("HTTP error: \(statusCode)")
        }
    }
}
```

## 構造体とクラスの使い分け

Swiftでは構造体とクラスの適切な使い分けが重要です。基本的には構造体を優先し、参照型が必要な場合にのみクラスを使用します。

### 推奨される使い分け

```swift
// 値型: 構造体を使用（推奨）
struct User: Codable {
    let id: String
    let name: String
    let email: String

    // 不変性を保つ
    func withUpdatedName(_ newName: String) -> User {
        return User(id: id, name: newName, email: email)
    }
}

// 参照型: クラスを使用（必要な場合のみ）
class UserSession {
    var currentUser: User?
    var authToken: String?

    // シングルトン
    static let shared = UserSession()

    private init() {}

    func login(user: User, token: String) {
        self.currentUser = user
        self.authToken = token
    }

    func logout() {
        self.currentUser = nil
        self.authToken = nil
    }
}
```

構造体を使うべきケース:
- データモデル
- 値の等価性が重要な場合
- コピーされても問題ない場合

クラスを使うべきケース:
- 参照の同一性が重要な場合
- 継承が必要な場合
- Objective-Cとの互換性が必要な場合

## Swiftレビューチェックリスト

レビュー時に確認すべき項目をチェックリストにまとめました。

### Optional型

- [ ] 強制アンラップ（!）を使用していないか
- [ ] guard letによる早期リターンが適切に使われているか
- [ ] Optional chainingとnil coalescing演算子が活用されているか
- [ ] 暗黙的アンラップOptional（!）の使用が最小限か

### メモリ管理

- [ ] クロージャでキャプチャリスト [weak self] または [unowned self] が適切に使われているか
- [ ] Delegate パターンでプロトコルがclass-bound（AnyObject）で、weak参照になっているか
- [ ] 参照サイクルが発生していないか

### プロトコルとアーキテクチャ

- [ ] 継承よりもプロトコルが優先されているか
- [ ] Protocol Extensionでデフォルト実装が提供されているか
- [ ] 依存性注入が可能な設計になっているか

### エラーハンドリング

- [ ] Result型またはthrowsを使った明示的なエラーハンドリングがされているか
- [ ] try?による無言のエラー無視が適切に使われているか
- [ ] カスタムエラー型が適切に定義されているか

### コーディング規約

- [ ] SwiftLintのルールに準拠しているか
- [ ] 命名規則が統一されているか（lowerCamelCase, UpperCamelCase）
- [ ] アクセス制御（private, fileprivate, internal, public）が適切か

### パフォーマンス

- [ ] 構造体とクラスが適切に使い分けられているか
- [ ] 不要なコピーが発生していないか
- [ ] 高コストな操作がメインスレッドで実行されていないか

## 実践例: ViewModelのレビュー

実際のコードレビューの例を見てみましょう。

```swift
import Foundation
import Combine

// MVVM パターンでのViewModel実装例
protocol UserListViewModelProtocol {
    var users: Published<[User]>.Publisher { get }
    var isLoading: Published<Bool>.Publisher { get }
    var errorMessage: Published<String?>.Publisher { get }

    func fetchUsers() async
    func refreshUsers() async
}

final class UserListViewModel: UserListViewModelProtocol, ObservableObject {
    // Published プロパティ
    @Published private(set) var usersValue: [User] = []
    @Published private(set) var isLoadingValue: Bool = false
    @Published private(set) var errorMessageValue: String?

    var users: Published<[User]>.Publisher { $usersValue }
    var isLoading: Published<Bool>.Publisher { $isLoadingValue }
    var errorMessage: Published<String?>.Publisher { $errorMessageValue }

    // 依存性注入
    private let userService: UserServiceProtocol
    private var cancellables = Set<AnyCancellable>()

    init(userService: UserServiceProtocol) {
        self.userService = userService
    }

    func fetchUsers() async {
        await loadUsers()
    }

    func refreshUsers() async {
        usersValue = []
        await loadUsers()
    }

    private func loadUsers() async {
        isLoadingValue = true
        errorMessageValue = nil

        do {
            let fetchedUsers = try await userService.fetchUsers()
            usersValue = fetchedUsers
        } catch {
            errorMessageValue = handleError(error)
        }

        isLoadingValue = false
    }

    private func handleError(_ error: Error) -> String {
        switch error {
        case NetworkError.noConnection:
            return "インターネット接続を確認してください"
        case NetworkError.timeout:
            return "タイムアウトしました。再試行してください"
        default:
            return "エラーが発生しました: \(error.localizedDescription)"
        }
    }
}

// SwiftUIでの使用例
struct UserListView: View {
    @StateObject private var viewModel: UserListViewModel

    init(viewModel: UserListViewModel) {
        _viewModel = StateObject(wrappedValue: viewModel)
    }

    var body: some View {
        NavigationView {
            Group {
                if viewModel.isLoadingValue {
                    ProgressView("読み込み中...")
                } else if let errorMessage = viewModel.errorMessageValue {
                    VStack {
                        Text(errorMessage)
                            .foregroundColor(.red)
                        Button("再試行") {
                            Task {
                                await viewModel.fetchUsers()
                            }
                        }
                    }
                } else {
                    List(viewModel.usersValue) { user in
                        UserRow(user: user)
                    }
                }
            }
            .navigationTitle("ユーザー一覧")
            .task {
                await viewModel.fetchUsers()
            }
        }
    }
}
```

このコードは、プロトコル、依存性注入、async/await、Combineのベストプラクティスを実践しています。

## まとめ

本章ではSwiftコードレビューの重要なポイントを解説しました。

主なポイント:
- Optional型はguard letやif letで安全にアンラップする
- weak/unownedを使って参照サイクルを防ぐ
- プロトコル指向プログラミングで柔軟な設計を実現する
- Result型やthrowsで明示的にエラーハンドリングする
- 構造体を優先し、必要な場合のみクラスを使う

次章では、Goコードレビューのガイドラインについて学びます。

## 参考文献

- [The Swift Programming Language](https://docs.swift.org/swift-book/)
- [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/)
- [SwiftLint Rules](https://realm.github.io/SwiftLint/rule-directory.html)
- [Apple Developer Documentation - Memory Safety](https://developer.apple.com/documentation/swift/memory-safety)
- [raywenderlich.com - Swift Style Guide](https://github.com/raywenderlich/swift-style-guide)
