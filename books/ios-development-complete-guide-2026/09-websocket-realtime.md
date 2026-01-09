---
title: "WebSocketとリアルタイム通信"
---

# WebSocketとリアルタイム通信

## この章で学ぶこと

- WebSocketの基礎と仕組み
- URLSessionWebSocketTaskの実装
- 接続管理と再接続戦略
- リアルタイムメッセージングパターン
- チャットアプリの実装例
- エラーハンドリングとデバッグ
- パフォーマンス最適化

## WebSocketの基礎

### WebSocketとは

WebSocketは、クライアントとサーバー間で双方向の永続的な接続を確立するプロトコルです。HTTPとは異なり、一度接続を確立すると、サーバーからクライアントへプッシュ通知を送信できます。

#### WebSocketの特徴

- **双方向通信**: クライアントとサーバーの両方が任意のタイミングでメッセージを送信可能
- **低レイテンシ**: ポーリングと比較して大幅に低レイテンシ
- **効率的**: 接続を維持することでオーバーヘッドを削減
- **リアルタイム**: チャット、通知、ライブアップデートに最適

### WebSocketとHTTPの比較

```
HTTP (ポーリング)
Client -> Server: リクエスト
Server -> Client: レスポンス
[繰り返し...]

WebSocket
Client <-> Server: 双方向通信
[接続が維持される]
```

### 使用例

- リアルタイムチャット
- ライブ通知
- オンラインゲーム
- 株価・為替のリアルタイム更新
- コラボレーションツール
- IoTデバイス制御

## URLSessionWebSocketTaskの実装

### 基本的なWebSocket接続

```swift
class WebSocketClient {
    private var webSocketTask: URLSessionWebSocketTask?
    private let url: URL

    init(url: URL) {
        self.url = url
    }

    func connect() {
        let session = URLSession(configuration: .default)
        webSocketTask = session.webSocketTask(with: url)
        webSocketTask?.resume()

        // メッセージ受信を開始
        receiveMessage()
    }

    func disconnect() {
        webSocketTask?.cancel(with: .goingAway, reason: nil)
        webSocketTask = nil
    }

    func send(_ message: String) {
        let message = URLSessionWebSocketTask.Message.string(message)
        webSocketTask?.send(message) { error in
            if let error = error {
                print("❌ Send error: \(error)")
            }
        }
    }

    private func receiveMessage() {
        webSocketTask?.receive { [weak self] result in
            switch result {
            case .success(let message):
                switch message {
                case .string(let text):
                    print("📩 Received: \(text)")
                case .data(let data):
                    print("📦 Received data: \(data)")
                @unknown default:
                    break
                }

                // 次のメッセージを待機
                self?.receiveMessage()

            case .failure(let error):
                print("❌ Receive error: \(error)")
            }
        }
    }
}
```

### async/awaitを使った実装

より現代的なasync/await実装:

```swift
actor WebSocketManager {
    enum ConnectionState {
        case disconnected
        case connecting
        case connected
        case disconnecting
    }

    enum WebSocketError: Error {
        case invalidURL
        case notConnected
        case connectionFailed
        case sendFailed
        case receiveFailed
    }

    private(set) var state: ConnectionState = .disconnected
    private var webSocketTask: URLSessionWebSocketTask?
    private var receiveTask: Task<Void, Never>?
    private var pingTask: Task<Void, Never>?

    private let url: URL
    private let session: URLSession

    // メッセージ受信のコールバック
    var onMessageReceived: ((String) -> Void)?
    var onDataReceived: ((Data) -> Void)?
    var onDisconnected: ((Error?) -> Void)?

    init(url: URL, session: URLSession = .shared) {
        self.url = url
        self.session = session
    }

    func connect() async throws {
        guard state == .disconnected else {
            return
        }

        state = .connecting

        webSocketTask = session.webSocketTask(with: url)
        webSocketTask?.resume()

        state = .connected

        // メッセージ受信を開始
        receiveTask = Task {
            await startReceiving()
        }

        // Ping送信を開始
        pingTask = Task {
            await startPinging()
        }
    }

    func disconnect() async {
        guard state == .connected else {
            return
        }

        state = .disconnecting

        // タスクをキャンセル
        receiveTask?.cancel()
        pingTask?.cancel()

        // WebSocket接続を閉じる
        webSocketTask?.cancel(with: .goingAway, reason: nil)
        webSocketTask = nil

        state = .disconnected

        onDisconnected?(nil)
    }

    func send(_ message: String) async throws {
        guard state == .connected else {
            throw WebSocketError.notConnected
        }

        let message = URLSessionWebSocketTask.Message.string(message)

        do {
            try await webSocketTask?.send(message)
        } catch {
            throw WebSocketError.sendFailed
        }
    }

    func send(_ data: Data) async throws {
        guard state == .connected else {
            throw WebSocketError.notConnected
        }

        let message = URLSessionWebSocketTask.Message.data(data)

        do {
            try await webSocketTask?.send(message)
        } catch {
            throw WebSocketError.sendFailed
        }
    }

    private func startReceiving() async {
        while state == .connected {
            do {
                guard let message = try await webSocketTask?.receive() else {
                    break
                }

                handleMessage(message)

            } catch {
                print("❌ Receive error: \(error)")

                // 接続エラーの場合は切断
                if case URLError.networkConnectionLost = error {
                    await handleDisconnection(error: error)
                    break
                }
            }
        }
    }

    private func handleMessage(_ message: URLSessionWebSocketTask.Message) {
        switch message {
        case .string(let text):
            onMessageReceived?(text)

        case .data(let data):
            onDataReceived?(data)

        @unknown default:
            break
        }
    }

    private func startPinging() async {
        while state == .connected {
            // 30秒ごとにPingを送信
            try? await Task.sleep(nanoseconds: 30_000_000_000)

            guard state == .connected else {
                break
            }

            do {
                try await webSocketTask?.sendPing { error in
                    if let error = error {
                        print("⚠️ Ping failed: \(error)")
                    }
                }
            } catch {
                print("❌ Ping error: \(error)")
            }
        }
    }

    private func handleDisconnection(error: Error?) async {
        state = .disconnected
        webSocketTask = nil
        onDisconnected?(error)
    }
}
```

## 再接続戦略

### 自動再接続の実装

接続が切れた時に自動的に再接続する機能:

```swift
actor ReconnectingWebSocketManager {
    enum ReconnectionStrategy {
        case immediate
        case linear(delay: TimeInterval)
        case exponential(baseDelay: TimeInterval, maxDelay: TimeInterval)

        func delay(for attempt: Int) -> TimeInterval {
            switch self {
            case .immediate:
                return 0

            case .linear(let delay):
                return delay

            case .exponential(let baseDelay, let maxDelay):
                let exponentialDelay = baseDelay * pow(2.0, Double(attempt))
                return min(exponentialDelay, maxDelay)
            }
        }
    }

    private let baseManager: WebSocketManager
    private let strategy: ReconnectionStrategy
    private let maxReconnectAttempts: Int

    private var reconnectAttempt = 0
    private var shouldReconnect = true
    private var reconnectTask: Task<Void, Never>?

    var onConnectionStateChanged: ((WebSocketManager.ConnectionState) -> Void)?

    init(
        url: URL,
        strategy: ReconnectionStrategy = .exponential(baseDelay: 1.0, maxDelay: 60.0),
        maxReconnectAttempts: Int = 5
    ) {
        self.baseManager = WebSocketManager(url: url)
        self.strategy = strategy
        self.maxReconnectAttempts = maxReconnectAttempts

        // 切断時の処理を設定
        Task {
            await baseManager.setOnDisconnected { [weak self] error in
                Task {
                    await self?.handleDisconnection(error: error)
                }
            }
        }
    }

    func connect() async throws {
        shouldReconnect = true
        reconnectAttempt = 0

        try await baseManager.connect()
        onConnectionStateChanged?(.connected)
    }

    func disconnect() async {
        shouldReconnect = false
        reconnectTask?.cancel()

        await baseManager.disconnect()
        onConnectionStateChanged?(.disconnected)
    }

    func send(_ message: String) async throws {
        try await baseManager.send(message)
    }

    private func handleDisconnection(error: Error?) async {
        guard shouldReconnect else {
            return
        }

        guard reconnectAttempt < maxReconnectAttempts else {
            print("❌ Max reconnection attempts reached")
            onConnectionStateChanged?(.disconnected)
            return
        }

        let delay = strategy.delay(for: reconnectAttempt)
        reconnectAttempt += 1

        print("🔄 Reconnecting in \(delay)s (attempt \(reconnectAttempt)/\(maxReconnectAttempts))...")

        reconnectTask = Task {
            try? await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))

            guard !Task.isCancelled, shouldReconnect else {
                return
            }

            do {
                try await baseManager.connect()
                reconnectAttempt = 0 // 成功したらリセット
                onConnectionStateChanged?(.connected)
                print("✅ Reconnected successfully")

            } catch {
                print("❌ Reconnection failed: \(error)")
                await handleDisconnection(error: error)
            }
        }
    }
}

extension WebSocketManager {
    func setOnDisconnected(_ handler: @escaping (Error?) -> Void) {
        self.onDisconnected = handler
    }
}
```

### ネットワーク監視との統合

ネットワーク状態を監視して再接続を制御:

```swift
import Network

actor NetworkMonitor {
    enum NetworkStatus {
        case satisfied
        case unsatisfied
        case requiresConnection
    }

    private let monitor = NWPathMonitor()
    private let queue = DispatchQueue(label: "NetworkMonitor")

    private(set) var status: NetworkStatus = .unsatisfied

    var onStatusChanged: ((NetworkStatus) -> Void)?

    func startMonitoring() {
        monitor.pathUpdateHandler = { [weak self] path in
            let newStatus: NetworkStatus

            if path.status == .satisfied {
                newStatus = .satisfied
            } else if path.status == .requiresConnection {
                newStatus = .requiresConnection
            } else {
                newStatus = .unsatisfied
            }

            Task {
                await self?.updateStatus(newStatus)
            }
        }

        monitor.start(queue: queue)
    }

    func stopMonitoring() {
        monitor.cancel()
    }

    private func updateStatus(_ newStatus: NetworkStatus) {
        guard status != newStatus else {
            return
        }

        status = newStatus
        onStatusChanged?(newStatus)
    }
}

class SmartReconnectingWebSocketManager {
    private let webSocketManager: ReconnectingWebSocketManager
    private let networkMonitor = NetworkMonitor()

    private var shouldBeConnected = false

    init(url: URL) {
        self.webSocketManager = ReconnectingWebSocketManager(url: url)

        // ネットワーク状態の変化を監視
        Task {
            await networkMonitor.setOnStatusChanged { [weak self] status in
                Task {
                    await self?.handleNetworkStatusChange(status)
                }
            }

            await networkMonitor.startMonitoring()
        }
    }

    func connect() async throws {
        shouldBeConnected = true
        try await webSocketManager.connect()
    }

    func disconnect() async {
        shouldBeConnected = false
        await webSocketManager.disconnect()
    }

    func send(_ message: String) async throws {
        try await webSocketManager.send(message)
    }

    private func handleNetworkStatusChange(_ status: NetworkMonitor.NetworkStatus) async {
        switch status {
        case .satisfied:
            // ネットワークが回復したら再接続
            if shouldBeConnected {
                print("🌐 Network available, reconnecting...")
                try? await webSocketManager.connect()
            }

        case .unsatisfied:
            // ネットワークが利用不可になったら切断
            print("📵 Network unavailable")
            await webSocketManager.disconnect()

        case .requiresConnection:
            print("⚠️ Network requires connection")
        }
    }
}

extension NetworkMonitor {
    func setOnStatusChanged(_ handler: @escaping (NetworkStatus) -> Void) {
        self.onStatusChanged = handler
    }
}
```

## メッセージングパターン

### メッセージプロトコルの定義

型安全なメッセージング:

```swift
protocol WebSocketMessage: Codable {
    var type: String { get }
}

struct MessageEnvelope: Codable {
    let type: String
    let payload: Data

    init<T: WebSocketMessage>(_ message: T) throws {
        self.type = T.messageType
        self.payload = try JSONEncoder().encode(message)
    }

    func decode<T: WebSocketMessage>(as type: T.Type) throws -> T {
        try JSONDecoder().decode(T.self, from: payload)
    }
}

extension WebSocketMessage {
    static var messageType: String {
        String(describing: Self.self)
    }

    var type: String {
        Self.messageType
    }
}

// メッセージ定義例

struct ChatMessage: WebSocketMessage {
    let id: String
    let userId: String
    let username: String
    let text: String
    let timestamp: Date
}

struct TypingIndicator: WebSocketMessage {
    let userId: String
    let isTyping: Bool
}

struct UserPresence: WebSocketMessage {
    let userId: String
    let status: String // "online", "offline", "away"
}

struct ReadReceipt: WebSocketMessage {
    let userId: String
    let messageId: String
}
```

### メッセージルーター

メッセージタイプに応じて処理を振り分け:

```swift
actor MessageRouter {
    private var handlers: [String: (Data) -> Void] = [:]

    func register<T: WebSocketMessage>(
        _ type: T.Type,
        handler: @escaping (T) -> Void
    ) {
        let messageType = T.messageType

        handlers[messageType] = { data in
            do {
                let decoder = JSONDecoder()
                decoder.dateDecodingStrategy = .iso8601
                let message = try decoder.decode(T.self, from: data)
                handler(message)
            } catch {
                print("❌ Failed to decode message: \(error)")
            }
        }
    }

    func route(messageData: Data) {
        // メッセージタイプを取得
        guard let envelope = try? JSONDecoder().decode(
            MessageEnvelope.self,
            from: messageData
        ) else {
            print("❌ Failed to decode message envelope")
            return
        }

        // ハンドラーを実行
        if let handler = handlers[envelope.type] {
            handler(envelope.payload)
        } else {
            print("⚠️ No handler registered for message type: \(envelope.type)")
        }
    }

    func route(messageString: String) {
        guard let data = messageString.data(using: .utf8) else {
            return
        }
        route(messageData: data)
    }
}

// 使用例
@MainActor
class ChatViewModel: ObservableObject {
    @Published var messages: [ChatMessage] = []
    @Published var typingUsers: Set<String> = []

    private let webSocketManager: WebSocketManager
    private let messageRouter = MessageRouter()

    init(webSocketManager: WebSocketManager) {
        self.webSocketManager = webSocketManager

        Task {
            await setupMessageHandlers()
            await setupWebSocketCallbacks()
        }
    }

    private func setupMessageHandlers() async {
        // チャットメッセージハンドラー
        await messageRouter.register(ChatMessage.self) { [weak self] message in
            Task { @MainActor in
                self?.messages.append(message)
            }
        }

        // タイピングインジケーターハンドラー
        await messageRouter.register(TypingIndicator.self) { [weak self] indicator in
            Task { @MainActor in
                if indicator.isTyping {
                    self?.typingUsers.insert(indicator.userId)
                } else {
                    self?.typingUsers.remove(indicator.userId)
                }
            }
        }

        // ユーザープレゼンスハンドラー
        await messageRouter.register(UserPresence.self) { presence in
            print("👤 User \(presence.userId) is \(presence.status)")
        }
    }

    private func setupWebSocketCallbacks() async {
        await webSocketManager.setOnMessageReceived { [weak self] messageString in
            Task {
                await self?.messageRouter.route(messageString: messageString)
            }
        }
    }

    func sendMessage(_ text: String) async throws {
        let message = ChatMessage(
            id: UUID().uuidString,
            userId: currentUserId,
            username: currentUsername,
            text: text,
            timestamp: Date()
        )

        let envelope = try MessageEnvelope(message)
        let data = try JSONEncoder().encode(envelope)
        let jsonString = String(data: data, encoding: .utf8)!

        try await webSocketManager.send(jsonString)
    }

    func sendTypingIndicator(isTyping: Bool) async throws {
        let indicator = TypingIndicator(
            userId: currentUserId,
            isTyping: isTyping
        )

        let envelope = try MessageEnvelope(indicator)
        let data = try JSONEncoder().encode(envelope)
        let jsonString = String(data: data, encoding: .utf8)!

        try await webSocketManager.send(jsonString)
    }
}

extension WebSocketManager {
    func setOnMessageReceived(_ handler: @escaping (String) -> Void) {
        self.onMessageReceived = handler
    }
}
```

## チャットアプリの実装

### チャットサービス

完全なチャット機能を提供するサービス:

```swift
actor ChatService {
    private let webSocketManager: SmartReconnectingWebSocketManager
    private let messageRouter = MessageRouter()

    // イベントハンドラー
    var onMessageReceived: ((ChatMessage) -> Void)?
    var onTypingIndicatorReceived: ((TypingIndicator) -> Void)?
    var onUserPresenceChanged: ((UserPresence) -> Void)?
    var onConnectionStateChanged: ((Bool) -> Void)?

    init(serverURL: URL) {
        self.webSocketManager = SmartReconnectingWebSocketManager(url: serverURL)
        setupMessageHandlers()
        setupConnectionHandlers()
    }

    func connect() async throws {
        try await webSocketManager.connect()
    }

    func disconnect() async {
        await webSocketManager.disconnect()
    }

    func sendMessage(_ text: String, channelId: String) async throws {
        let message = ChatMessage(
            id: UUID().uuidString,
            userId: currentUserId,
            username: currentUsername,
            text: text,
            timestamp: Date()
        )

        try await sendMessage(message)
    }

    func sendTypingIndicator(isTyping: Bool, channelId: String) async throws {
        let indicator = TypingIndicator(
            userId: currentUserId,
            isTyping: isTyping
        )

        try await sendMessage(indicator)
    }

    func markAsRead(messageId: String) async throws {
        let receipt = ReadReceipt(
            userId: currentUserId,
            messageId: messageId
        )

        try await sendMessage(receipt)
    }

    private func sendMessage<T: WebSocketMessage>(_ message: T) async throws {
        let encoder = JSONEncoder()
        encoder.dateEncodingStrategy = .iso8601

        let envelope = try MessageEnvelope(message)
        let data = try encoder.encode(envelope)
        let jsonString = String(data: data, encoding: .utf8)!

        try await webSocketManager.send(jsonString)
    }

    private func setupMessageHandlers() {
        Task {
            await messageRouter.register(ChatMessage.self) { [weak self] message in
                self?.onMessageReceived?(message)
            }

            await messageRouter.register(TypingIndicator.self) { [weak self] indicator in
                self?.onTypingIndicatorReceived?(indicator)
            }

            await messageRouter.register(UserPresence.self) { [weak self] presence in
                self?.onUserPresenceChanged?(presence)
            }

            await webSocketManager.setOnMessageReceived { [weak self] messageString in
                Task {
                    await self?.messageRouter.route(messageString: messageString)
                }
            }
        }
    }

    private func setupConnectionHandlers() {
        Task {
            await webSocketManager.setOnConnectionStateChanged { [weak self] state in
                let isConnected = state == .connected
                self?.onConnectionStateChanged?(isConnected)
            }
        }
    }
}

extension SmartReconnectingWebSocketManager {
    func setOnMessageReceived(_ handler: @escaping (String) -> Void) async {
        // Implementation
    }

    func setOnConnectionStateChanged(_ handler: @escaping (WebSocketManager.ConnectionState) -> Void) async {
        // Implementation
    }
}
```

### チャットViewModel

SwiftUIで使用するViewModel:

```swift
@MainActor
class ChatViewModel: ObservableObject {
    @Published var messages: [ChatMessage] = []
    @Published var typingUsers: Set<String> = []
    @Published var isConnected = false
    @Published var connectionError: Error?
    @Published var inputText = ""

    private let chatService: ChatService
    private let channelId: String

    private var typingDebounceTask: Task<Void, Never>?

    init(chatService: ChatService, channelId: String) {
        self.chatService = chatService
        self.channelId = channelId

        setupEventHandlers()
    }

    func connect() async {
        do {
            try await chatService.connect()
            connectionError = nil
        } catch {
            connectionError = error
            print("❌ Connection error: \(error)")
        }
    }

    func disconnect() async {
        await chatService.disconnect()
    }

    func sendMessage() async {
        guard !inputText.isEmpty else { return }

        let text = inputText
        inputText = ""

        do {
            try await chatService.sendMessage(text, channelId: channelId)
            stopTyping()
        } catch {
            print("❌ Failed to send message: \(error)")
        }
    }

    func handleInputChange() {
        // タイピングインジケーターの送信をデバウンス
        typingDebounceTask?.cancel()

        if !inputText.isEmpty {
            startTyping()

            typingDebounceTask = Task {
                try? await Task.sleep(nanoseconds: 3_000_000_000) // 3秒

                if !Task.isCancelled {
                    stopTyping()
                }
            }
        } else {
            stopTyping()
        }
    }

    func markAllAsRead() async {
        guard let lastMessage = messages.last else { return }

        do {
            try await chatService.markAsRead(messageId: lastMessage.id)
        } catch {
            print("❌ Failed to mark as read: \(error)")
        }
    }

    private func startTyping() {
        Task {
            try? await chatService.sendTypingIndicator(
                isTyping: true,
                channelId: channelId
            )
        }
    }

    private func stopTyping() {
        Task {
            try? await chatService.sendTypingIndicator(
                isTyping: false,
                channelId: channelId
            )
        }
    }

    private func setupEventHandlers() {
        Task {
            await chatService.setOnMessageReceived { [weak self] message in
                Task { @MainActor in
                    self?.messages.append(message)
                }
            }

            await chatService.setOnTypingIndicatorReceived { [weak self] indicator in
                Task { @MainActor in
                    if indicator.isTyping {
                        self?.typingUsers.insert(indicator.userId)
                    } else {
                        self?.typingUsers.remove(indicator.userId)
                    }
                }
            }

            await chatService.setOnConnectionStateChanged { [weak self] isConnected in
                Task { @MainActor in
                    self?.isConnected = isConnected
                }
            }
        }
    }
}

extension ChatService {
    func setOnMessageReceived(_ handler: @escaping (ChatMessage) -> Void) {
        self.onMessageReceived = handler
    }

    func setOnTypingIndicatorReceived(_ handler: @escaping (TypingIndicator) -> Void) {
        self.onTypingIndicatorReceived = handler
    }

    func setOnConnectionStateChanged(_ handler: @escaping (Bool) -> Void) {
        self.onConnectionStateChanged = handler
    }
}
```

### チャットView

SwiftUIビュー:

```swift
struct ChatView: View {
    @StateObject private var viewModel: ChatViewModel
    @FocusState private var isInputFocused: Bool

    init(chatService: ChatService, channelId: String) {
        _viewModel = StateObject(wrappedValue: ChatViewModel(
            chatService: chatService,
            channelId: channelId
        ))
    }

    var body: some View {
        VStack(spacing: 0) {
            // 接続状態インジケーター
            if !viewModel.isConnected {
                ConnectionStatusBanner(isConnected: viewModel.isConnected)
            }

            // メッセージリスト
            ScrollViewReader { proxy in
                ScrollView {
                    LazyVStack(spacing: 12) {
                        ForEach(viewModel.messages) { message in
                            MessageRow(message: message)
                                .id(message.id)
                        }

                        // タイピングインジケーター
                        if !viewModel.typingUsers.isEmpty {
                            TypingIndicatorView(typingUsers: viewModel.typingUsers)
                        }
                    }
                    .padding()
                }
                .onChange(of: viewModel.messages.count) { _ in
                    if let lastMessage = viewModel.messages.last {
                        withAnimation {
                            proxy.scrollTo(lastMessage.id, anchor: .bottom)
                        }
                    }
                }
            }

            Divider()

            // 入力エリア
            MessageInputView(
                text: $viewModel.inputText,
                isInputFocused: _isInputFocused,
                onSend: {
                    Task {
                        await viewModel.sendMessage()
                    }
                }
            )
            .onChange(of: viewModel.inputText) { _ in
                viewModel.handleInputChange()
            }
        }
        .navigationTitle("Chat")
        .task {
            await viewModel.connect()
        }
        .onDisappear {
            Task {
                await viewModel.disconnect()
            }
        }
    }
}

struct MessageRow: View {
    let message: ChatMessage

    var body: some View {
        HStack(alignment: .top, spacing: 12) {
            // アバター
            Circle()
                .fill(Color.blue)
                .frame(width: 40, height: 40)
                .overlay(
                    Text(message.username.prefix(1))
                        .foregroundColor(.white)
                        .font(.headline)
                )

            VStack(alignment: .leading, spacing: 4) {
                // ユーザー名と時刻
                HStack {
                    Text(message.username)
                        .font(.headline)

                    Text(message.timestamp, style: .time)
                        .font(.caption)
                        .foregroundColor(.secondary)
                }

                // メッセージテキスト
                Text(message.text)
                    .font(.body)
            }

            Spacer()
        }
    }
}

struct MessageInputView: View {
    @Binding var text: String
    @FocusState var isInputFocused: Bool
    let onSend: () -> Void

    var body: some View {
        HStack(spacing: 12) {
            TextField("Message", text: $text)
                .textFieldStyle(.roundedBorder)
                .focused($isInputFocused)
                .onSubmit {
                    onSend()
                }

            Button(action: onSend) {
                Image(systemName: "arrow.up.circle.fill")
                    .font(.title2)
                    .foregroundColor(text.isEmpty ? .gray : .blue)
            }
            .disabled(text.isEmpty)
        }
        .padding()
    }
}

struct TypingIndicatorView: View {
    let typingUsers: Set<String>

    var body: some View {
        HStack(spacing: 4) {
            ForEach(0..<3) { index in
                Circle()
                    .fill(Color.gray)
                    .frame(width: 8, height: 8)
                    .opacity(0.5)
                    .animation(
                        Animation.easeInOut(duration: 0.6)
                            .repeatForever()
                            .delay(Double(index) * 0.2),
                        value: typingUsers
                    )
            }

            Text("\(typingUsers.count) \(typingUsers.count == 1 ? "person" : "people") typing...")
                .font(.caption)
                .foregroundColor(.secondary)
        }
        .padding(.horizontal)
    }
}

struct ConnectionStatusBanner: View {
    let isConnected: Bool

    var body: some View {
        HStack {
            Image(systemName: isConnected ? "wifi" : "wifi.slash")
                .foregroundColor(.white)

            Text(isConnected ? "Connected" : "Reconnecting...")
                .font(.subheadline)
                .foregroundColor(.white)
        }
        .frame(maxWidth: .infinity)
        .padding(.vertical, 8)
        .background(isConnected ? Color.green : Color.orange)
    }
}
```

## エラーハンドリング

### WebSocketエラーの処理

```swift
enum WebSocketError: Error, LocalizedError {
    case connectionFailed(Error)
    case connectionLost
    case invalidMessage
    case sendFailed(Error)
    case receiveFailed(Error)
    case timeout
    case unauthorized
    case serverError(String)

    var errorDescription: String? {
        switch self {
        case .connectionFailed(let error):
            return "Failed to connect: \(error.localizedDescription)"
        case .connectionLost:
            return "Connection was lost"
        case .invalidMessage:
            return "Received invalid message"
        case .sendFailed(let error):
            return "Failed to send message: \(error.localizedDescription)"
        case .receiveFailed(let error):
            return "Failed to receive message: \(error.localizedDescription)"
        case .timeout:
            return "Connection timed out"
        case .unauthorized:
            return "Unauthorized access"
        case .serverError(let message):
            return "Server error: \(message)"
        }
    }

    var recoverySuggestion: String? {
        switch self {
        case .connectionFailed, .connectionLost, .timeout:
            return "Please check your internet connection and try again."
        case .unauthorized:
            return "Please login again."
        case .serverError:
            return "The server is experiencing issues. Please try again later."
        default:
            return nil
        }
    }

    var isRecoverable: Bool {
        switch self {
        case .connectionFailed, .connectionLost, .timeout:
            return true
        case .unauthorized:
            return false
        case .serverError:
            return true
        default:
            return false
        }
    }
}

class ErrorHandlingWebSocketManager {
    private let baseManager: WebSocketManager
    private var errorRecoveryTask: Task<Void, Never>?

    var onError: ((WebSocketError) -> Void)?

    init(url: URL) {
        self.baseManager = WebSocketManager(url: url)
        setupErrorHandlers()
    }

    private func setupErrorHandlers() {
        Task {
            await baseManager.setOnDisconnected { [weak self] error in
                if let error = error {
                    self?.handleError(.connectionLost)
                }
            }
        }
    }

    private func handleError(_ error: WebSocketError) {
        onError?(error)

        // リカバリー可能なエラーの場合は再接続を試みる
        if error.isRecoverable {
            errorRecoveryTask?.cancel()
            errorRecoveryTask = Task {
                await attemptRecovery(from: error)
            }
        }
    }

    private func attemptRecovery(from error: WebSocketError) async {
        print("🔧 Attempting to recover from error: \(error)")

        // エラーの種類に応じた復旧戦略
        switch error {
        case .connectionLost, .timeout:
            // 再接続を試みる
            try? await Task.sleep(nanoseconds: 2_000_000_000) // 2秒待機
            try? await baseManager.connect()

        case .serverError:
            // サーバーエラーの場合は少し長めに待機
            try? await Task.sleep(nanoseconds: 10_000_000_000) // 10秒待機
            try? await baseManager.connect()

        default:
            break
        }
    }
}
```

## パフォーマンス最適化

### メッセージバッファリング

大量のメッセージを効率的に処理:

```swift
actor MessageBuffer {
    private var buffer: [ChatMessage] = []
    private let maxBufferSize: Int
    private let flushInterval: TimeInterval

    private var flushTask: Task<Void, Never>?

    var onFlush: (([ChatMessage]) -> Void)?

    init(maxBufferSize: Int = 50, flushInterval: TimeInterval = 1.0) {
        self.maxBufferSize = maxBufferSize
        self.flushInterval = flushInterval

        startFlushTimer()
    }

    func add(_ message: ChatMessage) {
        buffer.append(message)

        if buffer.count >= maxBufferSize {
            flush()
        }
    }

    func flush() {
        guard !buffer.isEmpty else { return }

        let messagesToFlush = buffer
        buffer.removeAll()

        onFlush?(messagesToFlush)
    }

    private func startFlushTimer() {
        flushTask = Task {
            while !Task.isCancelled {
                try? await Task.sleep(nanoseconds: UInt64(flushInterval * 1_000_000_000))

                if !Task.isCancelled {
                    flush()
                }
            }
        }
    }

    func cancel() {
        flushTask?.cancel()
    }
}

class BufferedChatService {
    private let chatService: ChatService
    private let messageBuffer = MessageBuffer()

    var onMessagesReceived: (([ChatMessage]) -> Void)?

    init(chatService: ChatService) {
        self.chatService = chatService

        Task {
            await messageBuffer.setOnFlush { [weak self] messages in
                self?.onMessagesReceived?(messages)
            }

            await chatService.setOnMessageReceived { [weak self] message in
                Task {
                    await self?.messageBuffer.add(message)
                }
            }
        }
    }

    func flush() async {
        await messageBuffer.flush()
    }
}

extension MessageBuffer {
    func setOnFlush(_ handler: @escaping ([ChatMessage]) -> Void) {
        self.onFlush = handler
    }
}
```

### メモリ管理

古いメッセージを削除してメモリを節約:

```swift
@MainActor
class OptimizedChatViewModel: ObservableObject {
    @Published var messages: [ChatMessage] = []

    private let maxMessagesInMemory = 200
    private let chatService: ChatService

    init(chatService: ChatService) {
        self.chatService = chatService
        setupMessageHandler()
    }

    private func setupMessageHandler() {
        Task {
            await chatService.setOnMessageReceived { [weak self] message in
                Task { @MainActor in
                    self?.addMessage(message)
                }
            }
        }
    }

    private func addMessage(_ message: ChatMessage) {
        messages.append(message)

        // メモリ制限を超えたら古いメッセージを削除
        if messages.count > maxMessagesInMemory {
            let removeCount = messages.count - maxMessagesInMemory
            messages.removeFirst(removeCount)
        }
    }

    func loadOlderMessages() async {
        // サーバーから古いメッセージを読み込む
        // 実装はサーバーAPIに依存
    }
}
```

### バックグラウンド対応

アプリがバックグラウンドに移行した時の処理:

```swift
class BackgroundAwareWebSocketManager: ObservableObject {
    private let webSocketManager: WebSocketManager
    private var backgroundTask: UIBackgroundTaskIdentifier = .invalid

    init(url: URL) {
        self.webSocketManager = WebSocketManager(url: url)
        setupBackgroundHandling()
    }

    private func setupBackgroundHandling() {
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(willEnterForeground),
            name: UIApplication.willEnterForegroundNotification,
            object: nil
        )

        NotificationCenter.default.addObserver(
            self,
            selector: #selector(didEnterBackground),
            name: UIApplication.didEnterBackgroundNotification,
            object: nil
        )
    }

    @objc private func willEnterForeground() {
        Task {
            // フォアグラウンドに戻った時に再接続
            try? await webSocketManager.connect()
            endBackgroundTask()
        }
    }

    @objc private func didEnterBackground() {
        // バックグラウンドタスクを開始
        backgroundTask = UIApplication.shared.beginBackgroundTask { [weak self] in
            self?.endBackgroundTask()
        }

        Task {
            // バックグラウンドに移行したら切断
            await webSocketManager.disconnect()
        }
    }

    private func endBackgroundTask() {
        if backgroundTask != .invalid {
            UIApplication.shared.endBackgroundTask(backgroundTask)
            backgroundTask = .invalid
        }
    }

    deinit {
        NotificationCenter.default.removeObserver(self)
        endBackgroundTask()
    }
}
```

## デバッグとロギング

### WebSocketロガー

詳細なログを出力:

```swift
class WebSocketLogger {
    enum LogLevel {
        case debug
        case info
        case warning
        case error
    }

    private let logLevel: LogLevel

    init(logLevel: LogLevel = .info) {
        self.logLevel = logLevel
    }

    func log(_ level: LogLevel, _ message: String) {
        guard shouldLog(level) else { return }

        let emoji = emoji(for: level)
        let timestamp = DateFormatter.localizedString(
            from: Date(),
            dateStyle: .none,
            timeStyle: .medium
        )

        print("\(emoji) [\(timestamp)] \(message)")
    }

    private func shouldLog(_ level: LogLevel) -> Bool {
        level.priority >= logLevel.priority
    }

    private func emoji(for level: LogLevel) -> String {
        switch level {
        case .debug: return "🔍"
        case .info: return "ℹ️"
        case .warning: return "⚠️"
        case .error: return "❌"
        }
    }
}

extension WebSocketLogger.LogLevel {
    var priority: Int {
        switch self {
        case .debug: return 0
        case .info: return 1
        case .warning: return 2
        case .error: return 3
        }
    }
}

class LoggingWebSocketManager: WebSocketManager {
    private let logger = WebSocketLogger()

    override func connect() async throws {
        logger.log(.info, "Connecting to WebSocket...")
        try await super.connect()
        logger.log(.info, "Connected successfully")
    }

    override func disconnect() async {
        logger.log(.info, "Disconnecting from WebSocket...")
        await super.disconnect()
        logger.log(.info, "Disconnected")
    }

    override func send(_ message: String) async throws {
        logger.log(.debug, "Sending message: \(message)")
        try await super.send(message)
        logger.log(.debug, "Message sent successfully")
    }
}
```

## まとめ

この章では、WebSocketを使ったリアルタイム通信の実装方法を学びました。

### 重要なポイント

1. **WebSocket基礎**: URLSessionWebSocketTaskの使い方
2. **再接続戦略**: 自動再接続とネットワーク監視
3. **メッセージング**: 型安全なメッセージルーティング
4. **チャット実装**: 実用的なチャットアプリの構築
5. **パフォーマンス**: バッファリングとメモリ管理
6. **エラーハンドリング**: 堅牢なエラー処理

### ベストプラクティス

- 接続状態を常に監視する
- 適切な再接続戦略を実装する
- メッセージを型安全に扱う
- バックグラウンド対応を実装する
- 詳細なログを残す
- メモリ使用量を管理する

### 次のステップ

これでPart 3（ネットワーキング）が完了しました。次のPartでは、データ永続化とキャッシュ戦略について学びます。

### 参考リソース

- [URLSessionWebSocketTask - Apple Developer](https://developer.apple.com/documentation/foundation/urlsessionwebsockettask)
- [WebSocket Protocol - RFC 6455](https://tools.ietf.org/html/rfc6455)
- [Network Framework - Apple Developer](https://developer.apple.com/documentation/network)
