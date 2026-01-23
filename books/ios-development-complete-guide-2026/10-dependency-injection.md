---
title: "Dependency Injection - プロトコル指向設計とテスタビリティの向上"
---

# Dependency Injection - プロトコル指向設計とテスタビリティの向上

## はじめに

Dependency Injection (DI) は、依存関係の管理とテスタビリティの向上に不可欠な設計パターンです。本章では、iOS開発におけるDIの完全な実装方法、DIコンテナの構築、そして実践的なテスト手法を解説します。

想定シナリオでは、DI の適切な実装により、**ユニットテストの実行時間が74%短縮**し、**テストカバレッジが85%に到達**することが期待できます。

## Dependency Injection の基礎

### DIの3つの主要パターン

```swift
// MARK: - 1. Constructor Injection (推奨)
// 依存関係をイニシャライザで注入

class UserViewModel {
    private let userRepository: UserRepository
    private let analytics: AnalyticsService

    // ✅ 依存関係が明示的
    // ✅ 不変性を保証できる
    // ✅ テストが容易
    init(
        userRepository: UserRepository,
        analytics: AnalyticsService
    ) {
        self.userRepository = userRepository
        self.analytics = analytics
    }

    func fetchUsers() async {
        do {
            let users = try await userRepository.fetchUsers()
            analytics.track(.usersFetched(count: users.count))
        } catch {
            analytics.track(.error(error))
        }
    }
}

// MARK: - 2. Property Injection
// プロパティを通じて依存関係を注入

class UserViewController: UIViewController {
    // ⚠️ オプショナルになるため、使用時にチェックが必要
    var viewModel: UserViewModel?

    override func viewDidLoad() {
        super.viewDidLoad()

        // 依存関係が注入されているか確認
        guard let viewModel = viewModel else {
            fatalError("ViewModel must be injected")
        }

        Task {
            await viewModel.fetchUsers()
        }
    }
}

// 使用例
let viewController = UserViewController()
viewController.viewModel = UserViewModel(
    userRepository: userRepository,
    analytics: analytics
)

// MARK: - 3. Method Injection
// メソッドの引数として依存関係を注入

class UserService {
    // 特定の操作でのみ必要な依存関係に使用
    func exportUsers(
        users: [User],
        formatter: DataFormatter,
        exporter: FileExporter
    ) async throws {
        let formattedData = formatter.format(users)
        try await exporter.export(formattedData)
    }
}

// MARK: - ❌ アンチパターン: Service Locator
class UserViewModel {
    private let userRepository: UserRepository
    private let analytics: AnalyticsService

    init() {
        // ❌ 依存関係が隠蔽されている
        // ❌ グローバルな状態に依存
        // ❌ テストが困難
        self.userRepository = ServiceLocator.shared.resolve(UserRepository.self)
        self.analytics = ServiceLocator.shared.resolve(AnalyticsService.self)
    }
}

// MARK: - ❌ アンチパターン: Singleton への直接依存
class UserViewModel {
    func fetchUsers() async {
        // ❌ 具象クラスに直接依存
        // ❌ テスト時にモックできない
        let users = try? await NetworkManager.shared.fetchUsers()
    }
}
```

### プロトコル指向設計

Swiftの強力な型システムとプロトコル指向プログラミングを活用します。

```swift
// MARK: - Protocol First Design

// ❌ 悪い例: 具象クラスに依存
class OrderViewModel {
    private let apiClient = APIClient() // 具象クラス
    private let database = Database() // 具象クラス

    func createOrder(_ order: Order) async {
        // 実装
    }
}

// ✅ 良い例: プロトコルに依存
protocol OrderRepository {
    func createOrder(_ order: Order) async throws -> Order
    func fetchOrders() async throws -> [Order]
    func updateOrder(_ order: Order) async throws -> Order
}

protocol PaymentService {
    func processPayment(amount: Decimal, method: PaymentMethod) async throws -> PaymentResult
}

protocol NotificationService {
    func sendOrderConfirmation(order: Order) async throws
}

class OrderViewModel {
    private let orderRepository: OrderRepository
    private let paymentService: PaymentService
    private let notificationService: NotificationService

    init(
        orderRepository: OrderRepository,
        paymentService: PaymentService,
        notificationService: NotificationService
    ) {
        self.orderRepository = orderRepository
        self.paymentService = paymentService
        self.notificationService = notificationService
    }

    func createOrder(_ order: Order, paymentMethod: PaymentMethod) async throws {
        // 注文を作成
        let createdOrder = try await orderRepository.createOrder(order)

        // 支払いを処理
        let paymentResult = try await paymentService.processPayment(
            amount: order.totalAmount,
            method: paymentMethod
        )

        // 確認通知を送信
        try await notificationService.sendOrderConfirmation(order: createdOrder)
    }
}

// MARK: - Default Implementation Pattern
protocol ImageCache {
    func store(_ image: UIImage, forKey key: String)
    func retrieve(forKey key: String) -> UIImage?
    func clear()
}

// デフォルト実装
extension ImageCache where Self == MemoryImageCache {
    static var `default`: ImageCache {
        MemoryImageCache()
    }
}

class MemoryImageCache: ImageCache {
    private var cache: [String: UIImage] = [:]

    func store(_ image: UIImage, forKey key: String) {
        cache[key] = image
    }

    func retrieve(forKey key: String) -> UIImage? {
        cache[key]
    }

    func clear() {
        cache.removeAll()
    }
}

// 使用例
class ImageLoader {
    private let cache: ImageCache

    init(cache: ImageCache = .default) {
        self.cache = cache
    }
}
```

## DIコンテナの実装

### シンプルなDIコンテナ

```swift
// MARK: - Basic DI Container
class DIContainer {
    private var services: [String: Any] = [:]
    private var factories: [String: () -> Any] = [:]

    // MARK: - Singleton Registration
    func register<T>(_ type: T.Type, instance: T) {
        let key = String(describing: type)
        services[key] = instance
    }

    // MARK: - Factory Registration
    func register<T>(_ type: T.Type, factory: @escaping () -> T) {
        let key = String(describing: type)
        factories[key] = factory
    }

    // MARK: - Resolution
    func resolve<T>(_ type: T.Type) -> T {
        let key = String(describing: type)

        // シングルトンをチェック
        if let service = services[key] as? T {
            return service
        }

        // ファクトリーをチェック
        if let factory = factories[key] {
            if let service = factory() as? T {
                return service
            }
        }

        fatalError("No registration for type \(type)")
    }

    // MARK: - Optional Resolution
    func resolveOptional<T>(_ type: T.Type) -> T? {
        let key = String(describing: type)

        if let service = services[key] as? T {
            return service
        }

        if let factory = factories[key] {
            return factory() as? T
        }

        return nil
    }
}

// 使用例
let container = DIContainer()

// シングルトンの登録
container.register(NetworkService.self, instance: NetworkServiceImpl())

// ファクトリーの登録
container.register(UserRepository.self) {
    UserRepositoryImpl(
        networkService: container.resolve(NetworkService.self)
    )
}

// 解決
let repository = container.resolve(UserRepository.self)
```

### 高度なDIコンテナ

```swift
// MARK: - Advanced DI Container
class AdvancedDIContainer {
    // MARK: - Lifecycle
    enum Lifecycle {
        case singleton  // 1つのインスタンスを共有
        case transient  // 常に新しいインスタンスを生成
        case scoped     // スコープ内で1つのインスタンスを共有
    }

    // MARK: - Registration
    private struct Registration {
        let factory: () -> Any
        let lifecycle: Lifecycle
    }

    private var registrations: [String: Registration] = [:]
    private var singletons: [String: Any] = [:]
    private var scopedInstances: [String: Any] = [:]

    // MARK: - Register Methods
    func register<T>(
        _ type: T.Type,
        lifecycle: Lifecycle = .transient,
        factory: @escaping (AdvancedDIContainer) -> T
    ) {
        let key = String(describing: type)
        registrations[key] = Registration(
            factory: { factory(self) },
            lifecycle: lifecycle
        )
    }

    // MARK: - Resolve
    func resolve<T>(_ type: T.Type) -> T {
        let key = String(describing: type)

        guard let registration = registrations[key] else {
            fatalError("No registration for type \(type)")
        }

        switch registration.lifecycle {
        case .singleton:
            if let instance = singletons[key] as? T {
                return instance
            }

            let instance = registration.factory() as! T
            singletons[key] = instance
            return instance

        case .transient:
            return registration.factory() as! T

        case .scoped:
            if let instance = scopedInstances[key] as? T {
                return instance
            }

            let instance = registration.factory() as! T
            scopedInstances[key] = instance
            return instance
        }
    }

    // MARK: - Scope Management
    func beginScope() -> AdvancedDIContainer {
        let scopedContainer = AdvancedDIContainer()
        scopedContainer.registrations = self.registrations
        scopedContainer.singletons = self.singletons
        return scopedContainer
    }

    func endScope() {
        scopedInstances.removeAll()
    }

    // MARK: - Auto-wiring
    func resolve<T>(_ type: T.Type, with parameters: Any...) -> T {
        // 自動配線の実装
        // リフレクションを使用してコンストラクタの依存関係を解決
        return resolve(type)
    }
}

// MARK: - Property Wrapper for DI
@propertyWrapper
struct Injected<T> {
    private let container: AdvancedDIContainer
    private let key: String

    init(container: AdvancedDIContainer = .shared) {
        self.container = container
        self.key = String(describing: T.self)
    }

    var wrappedValue: T {
        container.resolve(T.self)
    }
}

// shared container
extension AdvancedDIContainer {
    static let shared = AdvancedDIContainer()
}

// 使用例
class UserViewModel {
    @Injected private var userRepository: UserRepository
    @Injected private var analytics: AnalyticsService

    func fetchUsers() async {
        do {
            let users = try await userRepository.fetchUsers()
            analytics.track(.usersFetched(count: users.count))
        } catch {
            analytics.track(.error(error))
        }
    }
}
```

### Type-Safe DI Container

```swift
// MARK: - Type-Safe DI Container using Phantom Types
struct DependencyKey<T> {
    let identifier: String

    init(_ identifier: String = String(describing: T.self)) {
        self.identifier = identifier
    }
}

class TypeSafeDIContainer {
    private var services: [String: Any] = [:]
    private var factories: [String: (TypeSafeDIContainer) -> Any] = [:]

    // MARK: - Registration
    func register<T>(
        _ key: DependencyKey<T>,
        lifecycle: Lifecycle = .singleton,
        factory: @escaping (TypeSafeDIContainer) -> T
    ) {
        let identifier = key.identifier

        switch lifecycle {
        case .singleton:
            factories[identifier] = { container in
                if let cached = container.services[identifier] as? T {
                    return cached
                }
                let instance = factory(container)
                container.services[identifier] = instance
                return instance
            }

        case .transient:
            factories[identifier] = { container in
                factory(container)
            }

        case .scoped:
            // Scoped implementation
            factories[identifier] = factory
        }
    }

    // MARK: - Resolution
    func resolve<T>(_ key: DependencyKey<T>) -> T {
        let identifier = key.identifier

        guard let factory = factories[identifier] else {
            fatalError("No registration for \(identifier)")
        }

        return factory(self) as! T
    }

    enum Lifecycle {
        case singleton
        case transient
        case scoped
    }
}

// MARK: - Dependency Keys
extension DependencyKey {
    static var userRepository: DependencyKey<UserRepository> {
        DependencyKey<UserRepository>()
    }

    static var networkService: DependencyKey<NetworkService> {
        DependencyKey<NetworkService>()
    }

    static var analytics: DependencyKey<AnalyticsService> {
        DependencyKey<AnalyticsService>()
    }
}

// 使用例
let container = TypeSafeDIContainer()

// 登録
container.register(.networkService) { _ in
    NetworkServiceImpl()
}

container.register(.userRepository) { container in
    UserRepositoryImpl(
        networkService: container.resolve(.networkService)
    )
}

container.register(.analytics) { _ in
    AnalyticsServiceImpl()
}

// 解決（型安全）
let repository = container.resolve(.userRepository)
let analytics = container.resolve(.analytics)
```

## 実践的なDI実装例

### アプリ全体のDI構成

```swift
// MARK: - App DI Container
class AppDIContainer {
    static let shared = AppDIContainer()

    private let container = AdvancedDIContainer()

    private init() {
        registerInfrastructure()
        registerDataLayer()
        registerDomainLayer()
        registerPresentationLayer()
    }

    // MARK: - Infrastructure Layer
    private func registerInfrastructure() {
        // HTTP Client
        container.register(HTTPClient.self, lifecycle: .singleton) { _ in
            URLSessionHTTPClient(
                session: URLSession.shared,
                baseURL: URL(string: "https://api.example.com")!
            )
        }

        // Logger
        container.register(Logger.self, lifecycle: .singleton) { _ in
            #if DEBUG
            return ConsoleLogger(level: .debug)
            #else
            return ConsoleLogger(level: .warning)
            #endif
        }

        // Analytics
        container.register(AnalyticsService.self, lifecycle: .singleton) { _ in
            AnalyticsServiceImpl(
                providers: [
                    FirebaseAnalyticsProvider(),
                    AmplitudeAnalyticsProvider()
                ]
            )
        }

        // Keychain
        container.register(KeychainService.self, lifecycle: .singleton) { _ in
            KeychainServiceImpl()
        }

        // Network Monitor
        container.register(NetworkMonitor.self, lifecycle: .singleton) { _ in
            NetworkMonitorImpl()
        }

        // Core Data Stack
        container.register(CoreDataStack.self, lifecycle: .singleton) { _ in
            CoreDataStack(modelName: "AppModel")
        }

        // Event Bus
        container.register(EventBus.self, lifecycle: .singleton) { _ in
            EventBusImpl()
        }
    }

    // MARK: - Data Layer
    private func registerDataLayer() {
        // Remote Data Sources
        container.register(UserRemoteDataSource.self, lifecycle: .singleton) { container in
            UserAPIDataSource(
                httpClient: container.resolve(HTTPClient.self),
                logger: container.resolve(Logger.self)
            )
        }

        container.register(AuthRemoteDataSource.self, lifecycle: .singleton) { container in
            AuthAPIDataSource(
                httpClient: container.resolve(HTTPClient.self)
            )
        }

        // Local Data Sources
        container.register(UserLocalDataSource.self, lifecycle: .singleton) { container in
            UserCoreDataSource(
                context: container.resolve(CoreDataStack.self).mainContext
            )
        }

        container.register(AuthLocalDataSource.self, lifecycle: .singleton) { container in
            AuthKeychainDataSource(
                keychain: container.resolve(KeychainService.self)
            )
        }

        // Repositories
        container.register(UserRepository.self, lifecycle: .singleton) { container in
            UserRepositoryImpl(
                remoteDataSource: container.resolve(UserRemoteDataSource.self),
                localDataSource: container.resolve(UserLocalDataSource.self),
                networkMonitor: container.resolve(NetworkMonitor.self),
                logger: container.resolve(Logger.self)
            )
        }

        container.register(AuthRepository.self, lifecycle: .singleton) { container in
            AuthRepositoryImpl(
                remoteDataSource: container.resolve(AuthRemoteDataSource.self),
                localDataSource: container.resolve(AuthLocalDataSource.self),
                logger: container.resolve(Logger.self)
            )
        }
    }

    // MARK: - Domain Layer
    private func registerDomainLayer() {
        // Validators
        container.register(UserValidator.self, lifecycle: .singleton) { _ in
            UserValidatorImpl()
        }

        container.register(PasswordHasher.self, lifecycle: .singleton) { _ in
            BCryptPasswordHasher()
        }

        // Use Cases - Users
        container.register(FetchUsersUseCase.self, lifecycle: .transient) { container in
            FetchUsersUseCaseImpl(
                userRepository: container.resolve(UserRepository.self),
                logger: container.resolve(Logger.self),
                eventBus: container.resolve(EventBus.self)
            )
        }

        container.register(CreateUserUseCase.self, lifecycle: .transient) { container in
            CreateUserUseCaseImpl(
                userRepository: container.resolve(UserRepository.self),
                authRepository: container.resolve(AuthRepository.self),
                passwordHasher: container.resolve(PasswordHasher.self),
                validator: container.resolve(UserValidator.self),
                eventBus: container.resolve(EventBus.self),
                logger: container.resolve(Logger.self)
            )
        }

        container.register(UpdateUserUseCase.self, lifecycle: .transient) { container in
            UpdateUserUseCaseImpl(
                userRepository: container.resolve(UserRepository.self),
                authRepository: container.resolve(AuthRepository.self),
                validator: container.resolve(UserValidator.self),
                eventBus: container.resolve(EventBus.self),
                logger: container.resolve(Logger.self)
            )
        }

        container.register(DeleteUserUseCase.self, lifecycle: .transient) { container in
            DeleteUserUseCaseImpl(
                userRepository: container.resolve(UserRepository.self),
                authRepository: container.resolve(AuthRepository.self),
                eventBus: container.resolve(EventBus.self),
                logger: container.resolve(Logger.self)
            )
        }

        // Use Cases - Authentication
        container.register(LoginUseCase.self, lifecycle: .transient) { container in
            LoginUseCaseImpl(
                authRepository: container.resolve(AuthRepository.self),
                analytics: container.resolve(AnalyticsService.self),
                logger: container.resolve(Logger.self)
            )
        }

        container.register(LogoutUseCase.self, lifecycle: .transient) { container in
            LogoutUseCaseImpl(
                authRepository: container.resolve(AuthRepository.self),
                logger: container.resolve(Logger.self)
            )
        }
    }

    // MARK: - Presentation Layer
    private func registerPresentationLayer() {
        // Coordinators
        container.register(AppCoordinator.self, lifecycle: .singleton) { container in
            AppCoordinator(
                window: UIWindow(),
                container: container
            )
        }

        // ViewModels are created on-demand with factory methods
    }

    // MARK: - ViewModel Factories
    func makeUserListViewModel() -> UserListViewModel {
        UserListViewModel(
            fetchUsersUseCase: container.resolve(FetchUsersUseCase.self),
            createUserUseCase: container.resolve(CreateUserUseCase.self),
            updateUserUseCase: container.resolve(UpdateUserUseCase.self),
            deleteUserUseCase: container.resolve(DeleteUserUseCase.self)
        )
    }

    func makeLoginViewModel() -> LoginViewModel {
        LoginViewModel(
            loginUseCase: container.resolve(LoginUseCase.self),
            analytics: container.resolve(AnalyticsService.self)
        )
    }

    func makeUserDetailViewModel(user: User) -> UserDetailViewModel {
        UserDetailViewModel(
            user: user,
            updateUserUseCase: container.resolve(UpdateUserUseCase.self),
            deleteUserUseCase: container.resolve(DeleteUserUseCase.self)
        )
    }

    // MARK: - Direct Resolution
    func resolve<T>(_ type: T.Type) -> T {
        container.resolve(type)
    }
}

// MARK: - App Entry Point
@main
struct MyApp: App {
    private let container = AppDIContainer.shared

    init() {
        setupDependencies()
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(container.resolve(AppCoordinator.self))
        }
    }

    private func setupDependencies() {
        // 初期化処理
        let logger = container.resolve(Logger.self)
        logger.info("App started")

        // Event Bus の購読設定
        let eventBus = container.resolve(EventBus.self)
        setupEventHandlers(eventBus: eventBus)
    }

    private func setupEventHandlers(eventBus: EventBus) {
        let analytics = container.resolve(AnalyticsService.self)

        // ユーザー作成イベント
        eventBus.subscribe(to: UserCreatedEvent.self) { event in
            await analytics.track(.userCreated(userId: event.userId))
        }

        // ユーザー更新イベント
        eventBus.subscribe(to: UserUpdatedEvent.self) { event in
            await analytics.track(.userUpdated(userId: event.userId, changes: event.changes))
        }

        // ユーザー削除イベント
        eventBus.subscribe(to: UserDeletedEvent.self) { event in
            await analytics.track(.userDeleted(userId: event.userId))
        }
    }
}
```

## 環境別のDI設定

```swift
// MARK: - Environment Configuration
enum Environment {
    case development
    case staging
    case production

    static var current: Environment {
        #if DEBUG
        return .development
        #elseif STAGING
        return .staging
        #else
        return .production
        #endif
    }

    var baseURL: URL {
        switch self {
        case .development:
            return URL(string: "https://dev-api.example.com")!
        case .staging:
            return URL(string: "https://staging-api.example.com")!
        case .production:
            return URL(string: "https://api.example.com")!
        }
    }

    var logLevel: Logger.Level {
        switch self {
        case .development:
            return .debug
        case .staging:
            return .info
        case .production:
            return .warning
        }
    }

    var analyticsEnabled: Bool {
        switch self {
        case .development:
            return false
        case .staging:
            return true
        case .production:
            return true
        }
    }
}

// MARK: - Environment-specific DI
extension AppDIContainer {
    static func create(for environment: Environment) -> AppDIContainer {
        let container = AppDIContainer()
        container.configureForEnvironment(environment)
        return container
    }

    private func configureForEnvironment(_ environment: Environment) {
        // HTTPClient の再設定
        container.register(HTTPClient.self, lifecycle: .singleton) { _ in
            URLSessionHTTPClient(
                session: URLSession.shared,
                baseURL: environment.baseURL
            )
        }

        // Logger の再設定
        container.register(Logger.self, lifecycle: .singleton) { _ in
            ConsoleLogger(level: environment.logLevel)
        }

        // Analytics の再設定
        if environment.analyticsEnabled {
            container.register(AnalyticsService.self, lifecycle: .singleton) { _ in
                AnalyticsServiceImpl(
                    providers: [
                        FirebaseAnalyticsProvider(),
                        AmplitudeAnalyticsProvider()
                    ]
                )
            }
        } else {
            container.register(AnalyticsService.self, lifecycle: .singleton) { _ in
                NoOpAnalyticsService()
            }
        }
    }
}
```

## モックとテスト

DIの最大の利点は、テスト時に依存関係を簡単にモックできることです。

```swift
// MARK: - Test DI Container
class TestDIContainer {
    private let container = AdvancedDIContainer()

    // MARK: - Mock Registration
    func registerMocks() {
        // Mock Network Service
        container.register(NetworkService.self, lifecycle: .singleton) { _ in
            MockNetworkService()
        }

        // Mock User Repository
        container.register(UserRepository.self, lifecycle: .singleton) { _ in
            MockUserRepository()
        }

        // Mock Auth Repository
        container.register(AuthRepository.self, lifecycle: .singleton) { _ in
            MockAuthRepository()
        }

        // Mock Analytics
        container.register(AnalyticsService.self, lifecycle: .singleton) { _ in
            MockAnalyticsService()
        }

        // Mock Logger
        container.register(Logger.self, lifecycle: .singleton) { _ in
            MockLogger()
        }

        // Mock Event Bus
        container.register(EventBus.self, lifecycle: .singleton) { _ in
            MockEventBus()
        }

        // Real Use Cases (using mocks)
        container.register(FetchUsersUseCase.self, lifecycle: .transient) { container in
            FetchUsersUseCaseImpl(
                userRepository: container.resolve(UserRepository.self),
                logger: container.resolve(Logger.self),
                eventBus: container.resolve(EventBus.self)
            )
        }
    }

    func resolve<T>(_ type: T.Type) -> T {
        container.resolve(type)
    }

    // MARK: - Mock Access
    var mockUserRepository: MockUserRepository {
        resolve(UserRepository.self) as! MockUserRepository
    }

    var mockAnalytics: MockAnalyticsService {
        resolve(AnalyticsService.self) as! MockAnalyticsService
    }

    var mockLogger: MockLogger {
        resolve(Logger.self) as! MockLogger
    }
}

// MARK: - Test Example
@MainActor
class UserListViewModelTests: XCTestCase {
    var viewModel: UserListViewModel!
    var container: TestDIContainer!

    override func setUp() {
        super.setUp()

        // テスト用DIコンテナの設定
        container = TestDIContainer()
        container.registerMocks()

        // ViewModelの作成
        viewModel = UserListViewModel(
            fetchUsersUseCase: container.resolve(FetchUsersUseCase.self),
            createUserUseCase: container.resolve(CreateUserUseCase.self),
            updateUserUseCase: container.resolve(UpdateUserUseCase.self),
            deleteUserUseCase: container.resolve(DeleteUserUseCase.self)
        )
    }

    func testFetchUsers_Success() async {
        // Given
        let expectedUsers = [
            User(id: UUID(), name: "Alice", email: "alice@example.com", age: 25, isActive: true, role: .user, createdAt: Date(), updatedAt: Date()),
            User(id: UUID(), name: "Bob", email: "bob@example.com", age: 30, isActive: true, role: .admin, createdAt: Date(), updatedAt: Date())
        ]
        container.mockUserRepository.fetchUsersResult = .success(expectedUsers)

        // When
        await viewModel.fetchUsers()

        // Then
        XCTAssertEqual(viewModel.users.count, 2)
        XCTAssertEqual(viewModel.users[0].name, "Alice")
        XCTAssertFalse(viewModel.isLoading)
        XCTAssertNil(viewModel.error)

        // Verify analytics
        XCTAssertEqual(container.mockAnalytics.trackedEvents.count, 1)
    }

    func testFetchUsers_Failure() async {
        // Given
        let expectedError = NSError(domain: "Test", code: 500, userInfo: nil)
        container.mockUserRepository.fetchUsersResult = .failure(expectedError)

        // When
        await viewModel.fetchUsers()

        // Then
        XCTAssertTrue(viewModel.users.isEmpty)
        XCTAssertNotNil(viewModel.error)
        XCTAssertFalse(viewModel.isLoading)

        // Verify error logging
        XCTAssertFalse(container.mockLogger.errorMessages.isEmpty)
    }
}

// MARK: - Mock Implementations
class MockUserRepository: UserRepository {
    var fetchUsersResult: Result<[User], Error> = .success([])
    var fetchUsersCallCount = 0

    func fetchUsers(filter: UserFilter?) async throws -> [User] {
        fetchUsersCallCount += 1
        return try fetchUsersResult.get()
    }

    func fetchUser(id: UUID) async throws -> User {
        throw NSError(domain: "Mock", code: 0, userInfo: nil)
    }

    func createUser(_ user: User) async throws -> User {
        user
    }

    func updateUser(_ user: User) async throws -> User {
        user
    }

    func deleteUser(id: UUID) async throws {
        // Mock
    }

    func searchUsers(query: String, limit: Int) async throws -> [User] {
        []
    }
}

class MockAnalyticsService: AnalyticsService {
    var trackedEvents: [AnalyticsEvent] = []

    func track(_ event: AnalyticsEvent) {
        trackedEvents.append(event)
    }

    func reset() {
        trackedEvents.removeAll()
    }
}

class MockLogger: Logger {
    var infoMessages: [String] = []
    var warningMessages: [String] = []
    var errorMessages: [String] = []
    var debugMessages: [String] = []

    func info(_ message: String) {
        infoMessages.append(message)
    }

    func warning(_ message: String) {
        warningMessages.append(message)
    }

    func error(_ message: String) {
        errorMessages.append(message)
    }

    func debug(_ message: String) {
        debugMessages.append(message)
    }
}

class MockEventBus: EventBus {
    var publishedEvents: [DomainEvent] = []
    private var handlers: [String: [(DomainEvent) async -> Void]] = [:]

    func publish(_ event: DomainEvent) async {
        publishedEvents.append(event)

        let key = String(describing: type(of: event))
        if let eventHandlers = handlers[key] {
            for handler in eventHandlers {
                await handler(event)
            }
        }
    }

    func subscribe<T: DomainEvent>(to eventType: T.Type, handler: @escaping (T) async -> Void) {
        let key = String(describing: eventType)

        let wrappedHandler: (DomainEvent) async -> Void = { event in
            if let typedEvent = event as? T {
                await handler(typedEvent)
            }
        }

        if handlers[key] != nil {
            handlers[key]?.append(wrappedHandler)
        } else {
            handlers[key] = [wrappedHandler]
        }
    }
}
```

## SwiftUI Preview との統合

```swift
// MARK: - Preview DI Container
class PreviewDIContainer {
    static let shared = PreviewDIContainer()

    private let container = AdvancedDIContainer()

    private init() {
        registerPreviewDependencies()
    }

    private func registerPreviewDependencies() {
        // Mock データを返すサービス
        container.register(UserRepository.self, lifecycle: .singleton) { _ in
            PreviewUserRepository()
        }

        container.register(AnalyticsService.self, lifecycle: .singleton) { _ in
            NoOpAnalyticsService()
        }

        container.register(Logger.self, lifecycle: .singleton) { _ in
            ConsoleLogger(level: .debug)
        }

        container.register(EventBus.self, lifecycle: .singleton) { _ in
            EventBusImpl()
        }

        // Use Cases
        container.register(FetchUsersUseCase.self, lifecycle: .transient) { container in
            FetchUsersUseCaseImpl(
                userRepository: container.resolve(UserRepository.self),
                logger: container.resolve(Logger.self),
                eventBus: container.resolve(EventBus.self)
            )
        }
    }

    func makeUserListViewModel() -> UserListViewModel {
        UserListViewModel(
            fetchUsersUseCase: container.resolve(FetchUsersUseCase.self),
            createUserUseCase: container.resolve(CreateUserUseCase.self),
            updateUserUseCase: container.resolve(UpdateUserUseCase.self),
            deleteUserUseCase: container.resolve(DeleteUserUseCase.self)
        )
    }
}

// MARK: - Preview Repository
class PreviewUserRepository: UserRepository {
    func fetchUsers(filter: UserFilter?) async throws -> [User] {
        // サンプルデータを返す
        [
            User(id: UUID(), name: "Alice Smith", email: "alice@example.com", age: 25, isActive: true, role: .user, createdAt: Date(), updatedAt: Date()),
            User(id: UUID(), name: "Bob Johnson", email: "bob@example.com", age: 30, isActive: true, role: .admin, createdAt: Date(), updatedAt: Date()),
            User(id: UUID(), name: "Charlie Brown", email: "charlie@example.com", age: 35, isActive: false, role: .moderator, createdAt: Date(), updatedAt: Date())
        ]
    }

    func fetchUser(id: UUID) async throws -> User {
        User(id: id, name: "Sample User", email: "sample@example.com", age: 28, isActive: true, role: .user, createdAt: Date(), updatedAt: Date())
    }

    func createUser(_ user: User) async throws -> User {
        user
    }

    func updateUser(_ user: User) async throws -> User {
        user
    }

    func deleteUser(id: UUID) async throws {
        // No-op
    }

    func searchUsers(query: String, limit: Int) async throws -> [User] {
        []
    }
}

// MARK: - SwiftUI Preview
struct UserListView_Previews: PreviewProvider {
    static var previews: some View {
        Group {
            // ライトモード
            UserListView(
                viewModel: PreviewDIContainer.shared.makeUserListViewModel()
            )
            .preferredColorScheme(.light)

            // ダークモード
            UserListView(
                viewModel: PreviewDIContainer.shared.makeUserListViewModel()
            )
            .preferredColorScheme(.dark)

            // iPad
            UserListView(
                viewModel: PreviewDIContainer.shared.makeUserListViewModel()
            )
            .previewDevice("iPad Pro (12.9-inch)")
        }
    }
}
```

## パフォーマンス最適化

### ベンチマーク指標

**想定シナリオ: エンタープライズアプリ（15万行規模）**

| 指標 | Before (Service Locator) | After (DI Container) | 改善率 |
|------|--------------------------|----------------------|--------|
| アプリ起動時間 | 3.2秒 | 2.1秒 | -34% |
| ViewModel生成時間 | 45ms | 12ms | -73% |
| 単体テスト実行時間 | 12.4秒 | 3.2秒 | -74% |
| テストカバレッジ | 52% | 85% | +63% |
| ビルド時間（Incremental） | 18秒 | 9秒 | -50% |

### 遅延初期化の実装

```swift
// MARK: - Lazy Injection
class OptimizedDIContainer {
    private var factories: [String: () -> Any] = [:]
    private var instances: [String: Any] = [:]

    func register<T>(_ type: T.Type, factory: @escaping () -> T) {
        let key = String(describing: type)
        factories[key] = factory
    }

    func resolve<T>(_ type: T.Type) -> T {
        let key = String(describing: type)

        // キャッシュをチェック
        if let instance = instances[key] as? T {
            return instance
        }

        // インスタンスを生成
        guard let factory = factories[key] else {
            fatalError("No registration for \(type)")
        }

        let instance = factory() as! T
        instances[key] = instance
        return instance
    }

    // パフォーマンス測定
    func measureResolution<T>(_ type: T.Type) -> (instance: T, duration: TimeInterval) {
        let start = Date()
        let instance = resolve(type)
        let duration = Date().timeIntervalSince(start)
        return (instance, duration)
    }
}

// 使用例
let container = OptimizedDIContainer()

// 登録
container.register(UserRepository.self) {
    UserRepositoryImpl(/* ... */)
}

// 測定
let (repository, duration) = container.measureResolution(UserRepository.self)
print("Resolution took: \(duration * 1000)ms")  // 例: 2.3ms
```

## ベストプラクティス

### 1. プロトコルファースト

```swift
// ✅ プロトコルを先に定義
protocol DataService {
    func fetch() async throws -> Data
}

class APIDataService: DataService {
    func fetch() async throws -> Data {
        // 実装
    }
}

class MockDataService: DataService {
    func fetch() async throws -> Data {
        // モック実装
    }
}
```

### 2. Lifecycle の適切な選択

```swift
// Singleton: アプリ全体で1つのインスタンス
container.register(NetworkService.self, lifecycle: .singleton)

// Transient: 毎回新しいインスタンス
container.register(UserViewModel.self, lifecycle: .transient)

// Scoped: スコープ内で1つのインスタンス
container.register(DatabaseContext.self, lifecycle: .scoped)
```

### 3. テストの容易さ

```swift
// ✅ テストしやすい構造
class ViewModel {
    init(repository: Repository) {
        // 依存関係を注入
    }
}

// ❌ テストしにくい構造
class ViewModel {
    let repository = Repository() // ハードコーディング
}
```

## まとめ

本章では、Dependency Injectionの完全な実装方法を解説しました。

### 重要ポイント

1. **Constructor Injection**: 依存関係を明示的に注入
2. **プロトコル指向**: 抽象への依存でテスタビリティを向上
3. **DIコンテナ**: 依存関係の一元管理
4. **Lifecycle管理**: Singleton、Transient、Scopedの適切な使い分け
5. **テスト**: モックによる簡単なテスト

### 想定される効果のまとめ

- **テスト実行時間**: 12.4秒 → 3.2秒 (-74%)
- **テストカバレッジ**: 52% → 85% (+63%)
- **アプリ起動時間**: 3.2秒 → 2.1秒 (-34%)
- **ViewModel生成**: 45ms → 12ms (-73%)

DIの適切な実装により、コードの品質、保守性、テスタビリティが劇的に向上します。Part 2で解説したMVVM、Clean Architecture、DIを組み合わせることで、エンタープライズグレードのiOSアプリケーションを構築できます。
