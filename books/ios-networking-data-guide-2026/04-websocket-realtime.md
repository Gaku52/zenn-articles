---
title: "WebSocket によるリアルタイム通信"
---

# WebSocket によるリアルタイム通信

WebSocket は、クライアントとサーバー間で双方向のリアルタイム通信を実現するプロトコルです。この章では、URLSessionWebSocketTask を使った実装方法を学びます。

## WebSocket の基礎

WebSocket は以下の特徴があります:

- **双方向通信**: サーバーからクライアントへのプッシュ通知が可能
- **低レイテンシ**: HTTP ポーリングと比較して遅延が少ない
- **接続維持**: 一度確立した接続を再利用できる

一般的な使用例:

- チャットアプリケーション
- リアルタイム通知
- 株価やスポーツのスコア更新
- 協同編集ツール

## URLSessionWebSocketTask の基本

まず、基本的な WebSocket 接続を実装します:

```swift
actor WebSocketManager {
    enum State {
        case disconnected
        case connecting
        case connected
        case disconnecting
    }

    private(set) var state: State = .disconnected
    private var webSocketTask: URLSessionWebSocketTask?
    private let url: URL

    init(url: URL) {
        self.url = url
    }

    func connect() {
        guard state == .disconnected else { return }

        state = .connecting

        let session = URLSession(configuration: .default)
        webSocketTask = session.webSocketTask(with: url)
        webSocketTask?.resume()

        state = .connected

        Task {
            await startReceiving()
        }

        Task {
            await sendPing()
        }
    }

    func disconnect() {
        state = .disconnecting
        webSocketTask?.cancel(with: .goingAway, reason: nil)
        webSocketTask = nil
        state = .disconnected
    }

    func send(_ message: String) async throws {
        guard state == .connected else {
            throw WebSocketError.notConnected
        }

        let message = URLSessionWebSocketTask.Message.string(message)
        try await webSocketTask?.send(message)
    }

    private func startReceiving() async {
        while state == .connected {
            do {
                guard let message = try await webSocketTask?.receive() else {
                    break
                }

                await handleMessage(message)
            } catch {
                print("WebSocket receive error: \(error)")
                break
            }
        }
    }

    private func handleMessage(_ message: URLSessionWebSocketTask.Message) async {
        switch message {
        case .string(let text):
            await onMessageReceived(text)
        case .data(let data):
            if let text = String(data: data, encoding: .utf8) {
                await onMessageReceived(text)
            }
        @unknown default:
            break
        }
    }

    private func sendPing() async {
        while state == .connected {
            try? await Task.sleep(nanoseconds: 30_000_000_000)  // 30秒ごと

            do {
                try await webSocketTask?.sendPing(pongReceiveHandler: { error in
                    if let error = error {
                        print("Ping failed: \(error)")
                    }
                })
            } catch {
                print("Ping error: \(error)")
            }
        }
    }

    private func onMessageReceived(_ message: String) async {
        // サブクラスでオーバーライド
    }
}

enum WebSocketError: Error {
    case notConnected
    case invalidMessage
}
```

## リアルタイムチャットの実装

WebSocket を使ったチャットアプリケーションを実装します:

```swift
@MainActor
class ChatViewModel: ObservableObject {
    @Published var messages: [ChatMessage] = []
    @Published var isConnected = false
    @Published var connectionError: String?

    private var webSocket: ChatWebSocket?
    private let decoder = JSONDecoder()
    private let encoder = JSONEncoder()

    struct ChatMessage: Codable, Identifiable {
        let id: String
        let userId: String
        let username: String
        let text: String
        let timestamp: Date

        init(userId: String, username: String, text: String) {
            self.id = UUID().uuidString
            self.userId = userId
            self.username = username
            self.text = text
            self.timestamp = Date()
        }
    }

    init() {
        decoder.dateDecodingStrategy = .iso8601
        encoder.dateEncodingStrategy = .iso8601
    }

    func connect(roomId: String) async {
        let url = URL(string: "wss://api.example.com/chat/\(roomId)")!
        webSocket = ChatWebSocket(url: url) { [weak self] message in
            await self?.handleIncomingMessage(message)
        }

        await webSocket?.connect()
        isConnected = true
        connectionError = nil
    }

    func disconnect() async {
        await webSocket?.disconnect()
        webSocket = nil
        isConnected = false
    }

    func sendMessage(_ text: String) async {
        guard let webSocket = webSocket else { return }

        let message = ChatMessage(
            userId: currentUserId,
            username: currentUsername,
            text: text
        )

        guard let data = try? encoder.encode(message),
              let jsonString = String(data: data, encoding: .utf8) else {
            return
        }

        do {
            try await webSocket.send(jsonString)
            messages.append(message)
        } catch {
            connectionError = "メッセージの送信に失敗しました"
        }
    }

    private func handleIncomingMessage(_ jsonString: String) {
        guard let data = jsonString.data(using: .utf8),
              let message = try? decoder.decode(ChatMessage.self, from: data) else {
            return
        }

        if !messages.contains(where: { $0.id == message.id }) {
            messages.append(message)
        }
    }

    private var currentUserId: String {
        // 実際のユーザーID を取得
        "user-123"
    }

    private var currentUsername: String {
        // 実際のユーザー名を取得
        "John Doe"
    }
}

actor ChatWebSocket: WebSocketManager {
    private let onMessageReceived: (String) async -> Void

    init(url: URL, onMessageReceived: @escaping (String) async -> Void) {
        self.onMessageReceived = onMessageReceived
        super.init(url: url)
    }

    override func onMessageReceived(_ message: String) async {
        await onMessageReceived(message)
    }
}
```

## SwiftUI での Chat View

チャット画面を SwiftUI で実装します:

```swift
struct ChatView: View {
    @StateObject private var viewModel = ChatViewModel()
    @State private var messageText = ""
    let roomId: String

    var body: some View {
        VStack {
            // メッセージリスト
            ScrollViewReader { proxy in
                ScrollView {
                    LazyVStack(spacing: 12) {
                        ForEach(viewModel.messages) { message in
                            MessageRow(message: message)
                                .id(message.id)
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

            // 入力欄
            HStack {
                TextField("メッセージを入力", text: $messageText)
                    .textFieldStyle(.roundedBorder)

                Button("送信") {
                    Task {
                        await viewModel.sendMessage(messageText)
                        messageText = ""
                    }
                }
                .disabled(messageText.isEmpty || !viewModel.isConnected)
            }
            .padding()
        }
        .navigationTitle("チャット")
        .task {
            await viewModel.connect(roomId: roomId)
        }
        .onDisappear {
            Task {
                await viewModel.disconnect()
            }
        }
        .overlay {
            if !viewModel.isConnected {
                VStack {
                    ProgressView("接続中...")
                        .padding()
                        .background(Color(uiColor: .systemBackground))
                        .cornerRadius(10)
                        .shadow(radius: 5)
                }
            }
        }
        .alert("エラー", isPresented: .constant(viewModel.connectionError != nil)) {
            Button("OK") {
                viewModel.connectionError = nil
            }
        } message: {
            Text(viewModel.connectionError ?? "")
        }
    }
}

struct MessageRow: View {
    let message: ChatViewModel.ChatMessage

    var body: some View {
        VStack(alignment: .leading, spacing: 4) {
            HStack {
                Text(message.username)
                    .font(.caption)
                    .fontWeight(.bold)

                Spacer()

                Text(message.timestamp, style: .time)
                    .font(.caption2)
                    .foregroundColor(.secondary)
            }

            Text(message.text)
                .padding(10)
                .background(Color.blue.opacity(0.1))
                .cornerRadius(8)
        }
    }
}
```

## 自動再接続機能

接続が切れた場合に自動的に再接続する機能を追加します:

```swift
actor ReconnectableWebSocket {
    private var webSocket: WebSocketManager?
    private let url: URL
    private let maxReconnectAttempts: Int
    private var reconnectAttempt = 0
    private var shouldReconnect = true

    var onMessageReceived: ((String) async -> Void)?
    var onConnectionStateChanged: ((Bool) async -> Void)?

    init(url: URL, maxReconnectAttempts: Int = 5) {
        self.url = url
        self.maxReconnectAttempts = maxReconnectAttempts
    }

    func connect() async {
        shouldReconnect = true
        await performConnect()
    }

    func disconnect() async {
        shouldReconnect = false
        await webSocket?.disconnect()
        webSocket = nil
        await onConnectionStateChanged?(false)
    }

    func send(_ message: String) async throws {
        try await webSocket?.send(message)
    }

    private func performConnect() async {
        let socket = WebSocketManager(url: url)
        webSocket = socket

        await socket.connect()
        await onConnectionStateChanged?(true)
        reconnectAttempt = 0

        // メッセージ受信の監視
        await monitorConnection()
    }

    private func monitorConnection() async {
        // 接続が切れた場合の再接続ロジック
        guard shouldReconnect, reconnectAttempt < maxReconnectAttempts else {
            return
        }

        // 指数バックオフで待機
        let delay = pow(2.0, Double(reconnectAttempt))
        try? await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))

        reconnectAttempt += 1
        await performConnect()
    }
}
```

## プレゼンス機能

ユーザーのオンライン状態を管理します:

```swift
class PresenceManager {
    private var webSocket: WebSocketManager?
    private var presenceTimer: Timer?

    func startPresence() {
        // 30秒ごとにプレゼンスを送信
        presenceTimer = Timer.scheduledTimer(withTimeInterval: 30, repeats: true) { [weak self] _ in
            Task {
                await self?.sendPresence()
            }
        }
    }

    func stopPresence() {
        presenceTimer?.invalidate()
        presenceTimer = nil

        Task {
            await sendOfflineStatus()
        }
    }

    private func sendPresence() async {
        let presence = [
            "type": "presence",
            "status": "online",
            "userId": currentUserId
        ]

        if let data = try? JSONSerialization.data(withJSONObject: presence),
           let jsonString = String(data: data, encoding: .utf8) {
            try? await webSocket?.send(jsonString)
        }
    }

    private func sendOfflineStatus() async {
        let presence = [
            "type": "presence",
            "status": "offline",
            "userId": currentUserId
        ]

        if let data = try? JSONSerialization.data(withJSONObject: presence),
           let jsonString = String(data: data, encoding: .utf8) {
            try? await webSocket?.send(jsonString)
        }
    }

    private var currentUserId: String {
        "user-123"
    }
}
```

## まとめ

この章では、WebSocket を使ったリアルタイム通信を学びました:

- URLSessionWebSocketTask の基本的な使い方
- リアルタイムチャットアプリケーションの実装
- 自動再接続機能
- プレゼンス管理

次の章では、UserDefaults と FileManager によるデータ永続化について学びます。

## 参考リソース

- [URLSessionWebSocketTask - Apple Developer](https://developer.apple.com/documentation/foundation/urlsessionwebsockettask)
- [WebSocket - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
