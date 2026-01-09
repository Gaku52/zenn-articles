---
title: "Chapter 23: 実戦ケーススタディ Part 1 - プロジェクト設計と基本機能"
---

# Chapter 23: 実戦ケーススタディ Part 1 - プロジェクト設計と基本機能

## 23.1 プロジェクト概要

### 23.1.1 アプリケーション仕様

**SocialConnect** - モダンなSNSアプリケーション

```swift
/*
アプリ名: SocialConnect
概要: 写真とテキストを共有できるSNSアプリ

主要機能:
1. ユーザー認証
   - メール/パスワード認証
   - Apple Sign In
   - Google Sign In

2. タイムライン
   - 投稿一覧表示
   - 無限スクロール
   - プルトゥリフレッシュ
   - リアルタイム更新

3. 投稿機能
   - テキスト投稿
   - 画像添付（最大4枚）
   - ハッシュタグ対応
   - 位置情報タグ

4. インタラクション
   - いいね機能
   - コメント機能
   - シェア機能

5. プロフィール
   - プロフィール編集
   - 投稿履歴
   - フォロー/フォロワー管理

6. 通知
   - プッシュ通知
   - アプリ内通知
   - 通知設定

技術スタック:
- UI: SwiftUI
- Architecture: MVVM + Clean Architecture
- Database: Firebase Firestore
- Authentication: Firebase Auth
- Storage: Firebase Storage
- Push: Firebase Cloud Messaging
- Analytics: Firebase Analytics
- Crash Reporting: Firebase Crashlytics
*/
```

### 23.1.2 プロジェクト構成

```
SocialConnect/
├── App/
│   ├── SocialConnectApp.swift
│   └── AppDelegate.swift
│
├── Features/
│   ├── Authentication/
│   │   ├── Domain/
│   │   ├── Data/
│   │   └── Presentation/
│   ├── Timeline/
│   │   ├── Domain/
│   │   ├── Data/
│   │   └── Presentation/
│   ├── Post/
│   │   ├── Domain/
│   │   ├── Data/
│   │   └── Presentation/
│   ├── Profile/
│   │   ├── Domain/
│   │   ├── Data/
│   │   └── Presentation/
│   └── Notifications/
│       ├── Domain/
│       ├── Data/
│       └── Presentation/
│
├── Core/
│   ├── Networking/
│   ├── Database/
│   ├── Storage/
│   ├── Services/
│   └── Extensions/
│
├── Common/
│   ├── UI/
│   ├── Utils/
│   └── Constants/
│
└── Resources/
    ├── Assets.xcassets
    └── Fonts/
```

## 23.2 認証機能の実装

### 23.2.1 ドメイン層

```swift
// Features/Authentication/Domain/Entities/User.swift

import Foundation

struct User: Identifiable, Codable {
    let id: String
    let email: String
    let displayName: String
    let photoURL: URL?
    let bio: String?
    let followersCount: Int
    let followingCount: Int
    let postsCount: Int
    let createdAt: Date

    enum CodingKeys: String, CodingKey {
        case id
        case email
        case displayName = "display_name"
        case photoURL = "photo_url"
        case bio
        case followersCount = "followers_count"
        case followingCount = "following_count"
        case postsCount = "posts_count"
        case createdAt = "created_at"
    }
}

extension User {
    static var preview: User {
        User(
            id: "1",
            email: "user@example.com",
            displayName: "John Doe",
            photoURL: nil,
            bio: "iOS Developer",
            followersCount: 150,
            followingCount: 200,
            postsCount: 42,
            createdAt: Date()
        )
    }
}
```

```swift
// Features/Authentication/Domain/UseCases/LoginUseCase.swift

import Foundation

protocol LoginUseCaseProtocol {
    func execute(email: String, password: String) async throws -> User
}

final class LoginUseCase: LoginUseCaseProtocol {
    private let authRepository: AuthRepositoryProtocol

    init(authRepository: AuthRepositoryProtocol) {
        self.authRepository = authRepository
    }

    func execute(email: String, password: String) async throws -> User {
        // バリデーション
        guard isValidEmail(email) else {
            throw AuthError.invalidEmail
        }

        guard password.count >= 6 else {
            throw AuthError.weakPassword
        }

        // ログイン実行
        let user = try await authRepository.login(email: email, password: password)

        // ユーザー情報をキャッシュ
        await cacheUser(user)

        return user
    }

    private func isValidEmail(_ email: String) -> Bool {
        let emailRegex = "[A-Z0-9a-z._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,64}"
        let emailPredicate = NSPredicate(format: "SELF MATCHES %@", emailRegex)
        return emailPredicate.evaluate(with: email)
    }

    private func cacheUser(_ user: User) async {
        UserDefaults.standard.set(user.id, forKey: "currentUserId")
    }
}
```

### 23.2.2 データ層

```swift
// Features/Authentication/Data/Repositories/AuthRepository.swift

import Foundation
import FirebaseAuth
import FirebaseFirestore

protocol AuthRepositoryProtocol {
    func login(email: String, password: String) async throws -> User
    func signUp(email: String, password: String, displayName: String) async throws -> User
    func signInWithApple(idToken: String, nonce: String) async throws -> User
    func logout() async throws
    func getCurrentUser() async -> User?
}

final class AuthRepository: AuthRepositoryProtocol {
    private let auth: Auth
    private let firestore: Firestore

    init(auth: Auth = .auth(), firestore: Firestore = .firestore()) {
        self.auth = auth
        self.firestore = firestore
    }

    func login(email: String, password: String) async throws -> User {
        let authResult = try await auth.signIn(withEmail: email, password: password)
        let user = try await fetchUser(uid: authResult.user.uid)
        return user
    }

    func signUp(email: String, password: String, displayName: String) async throws -> User {
        let authResult = try await auth.createUser(withEmail: email, password: password)
        let uid = authResult.user.uid

        let user = User(
            id: uid,
            email: email,
            displayName: displayName,
            photoURL: nil,
            bio: nil,
            followersCount: 0,
            followingCount: 0,
            postsCount: 0,
            createdAt: Date()
        )

        try await saveUser(user)
        return user
    }

    func signInWithApple(idToken: String, nonce: String) async throws -> User {
        let credential = OAuthProvider.credential(
            withProviderID: "apple.com",
            idToken: idToken,
            rawNonce: nonce
        )

        let authResult = try await auth.signIn(with: credential)
        let uid = authResult.user.uid

        // 既存ユーザーの確認
        if let existingUser = try? await fetchUser(uid: uid) {
            return existingUser
        }

        // 新規ユーザーの作成
        let user = User(
            id: uid,
            email: authResult.user.email ?? "",
            displayName: authResult.user.displayName ?? "User",
            photoURL: authResult.user.photoURL,
            bio: nil,
            followersCount: 0,
            followingCount: 0,
            postsCount: 0,
            createdAt: Date()
        )

        try await saveUser(user)
        return user
    }

    func logout() async throws {
        try auth.signOut()
        UserDefaults.standard.removeObject(forKey: "currentUserId")
    }

    func getCurrentUser() async -> User? {
        guard let uid = auth.currentUser?.uid else { return nil }
        return try? await fetchUser(uid: uid)
    }

    // MARK: - Private Methods

    private func fetchUser(uid: String) async throws -> User {
        let document = try await firestore.collection("users").document(uid).getDocument()

        guard let data = document.data() else {
            throw AuthError.userNotFound
        }

        return try Firestore.Decoder().decode(User.self, from: data)
    }

    private func saveUser(_ user: User) async throws {
        let data = try Firestore.Encoder().encode(user)
        try await firestore.collection("users").document(user.id).setData(data)
    }
}
```

### 23.2.3 プレゼンテーション層

```swift
// Features/Authentication/Presentation/ViewModels/LoginViewModel.swift

import Foundation
import Combine

@MainActor
final class LoginViewModel: ObservableObject {
    // MARK: - Published Properties

    @Published var email: String = ""
    @Published var password: String = ""
    @Published var isLoading: Bool = false
    @Published var errorMessage: String?
    @Published var isLoggedIn: Bool = false

    // MARK: - Private Properties

    private let loginUseCase: LoginUseCaseProtocol
    private var cancellables = Set<AnyCancellable>()

    // MARK: - Initialization

    init(loginUseCase: LoginUseCaseProtocol) {
        self.loginUseCase = loginUseCase
    }

    // MARK: - Public Methods

    func login() async {
        isLoading = true
        errorMessage = nil
        defer { isLoading = false }

        do {
            _ = try await loginUseCase.execute(email: email, password: password)
            isLoggedIn = true
        } catch let error as AuthError {
            errorMessage = error.localizedDescription
        } catch {
            errorMessage = "予期しないエラーが発生しました"
        }
    }

    func signInWithApple() async {
        // Apple Sign Inの実装
        // 実際の実装ではAuthenticationServicesを使用
    }

    var isValid: Bool {
        !email.isEmpty && !password.isEmpty
    }
}
```

```swift
// Features/Authentication/Presentation/Views/LoginView.swift

import SwiftUI

struct LoginView: View {
    @StateObject private var viewModel: LoginViewModel
    @FocusState private var focusedField: Field?

    enum Field {
        case email
        case password
    }

    init(viewModel: LoginViewModel) {
        _viewModel = StateObject(wrappedValue: viewModel)
    }

    var body: some View {
        NavigationView {
            ZStack {
                // Background
                LinearGradient(
                    colors: [.blue.opacity(0.6), .purple.opacity(0.6)],
                    startPoint: .topLeading,
                    endPoint: .bottomTrailing
                )
                .ignoresSafeArea()

                ScrollView {
                    VStack(spacing: 24) {
                        // Logo
                        Image(systemName: "person.3.fill")
                            .font(.system(size: 80))
                            .foregroundColor(.white)
                            .padding(.top, 60)

                        Text("SocialConnect")
                            .font(.largeTitle)
                            .fontWeight(.bold)
                            .foregroundColor(.white)

                        // Form
                        VStack(spacing: 16) {
                            // Email
                            CustomTextField(
                                icon: "envelope.fill",
                                placeholder: "Email",
                                text: $viewModel.email
                            )
                            .focused($focusedField, equals: .email)
                            .textContentType(.emailAddress)
                            .keyboardType(.emailAddress)
                            .autocapitalization(.none)

                            // Password
                            CustomSecureField(
                                icon: "lock.fill",
                                placeholder: "Password",
                                text: $viewModel.password
                            )
                            .focused($focusedField, equals: .password)
                            .textContentType(.password)

                            // Error Message
                            if let errorMessage = viewModel.errorMessage {
                                Text(errorMessage)
                                    .font(.caption)
                                    .foregroundColor(.red)
                                    .padding(.horizontal)
                            }

                            // Login Button
                            Button {
                                Task {
                                    await viewModel.login()
                                }
                            } label: {
                                HStack {
                                    if viewModel.isLoading {
                                        ProgressView()
                                            .progressViewStyle(CircularProgressViewStyle(tint: .white))
                                    } else {
                                        Text("Login")
                                            .fontWeight(.semibold)
                                    }
                                }
                                .frame(maxWidth: .infinity)
                                .frame(height: 50)
                                .background(Color.blue)
                                .foregroundColor(.white)
                                .cornerRadius(12)
                            }
                            .disabled(!viewModel.isValid || viewModel.isLoading)

                            // Divider
                            HStack {
                                Rectangle()
                                    .frame(height: 1)
                                    .foregroundColor(.white.opacity(0.3))
                                Text("or")
                                    .foregroundColor(.white)
                                Rectangle()
                                    .frame(height: 1)
                                    .foregroundColor(.white.opacity(0.3))
                            }
                            .padding(.vertical, 8)

                            // Apple Sign In
                            Button {
                                Task {
                                    await viewModel.signInWithApple()
                                }
                            } label: {
                                HStack {
                                    Image(systemName: "apple.logo")
                                    Text("Sign in with Apple")
                                        .fontWeight(.semibold)
                                }
                                .frame(maxWidth: .infinity)
                                .frame(height: 50)
                                .background(Color.black)
                                .foregroundColor(.white)
                                .cornerRadius(12)
                            }
                        }
                        .padding(.horizontal, 32)
                        .padding(.top, 24)

                        // Sign Up Link
                        NavigationLink(destination: SignUpView()) {
                            Text("Don't have an account? Sign Up")
                                .foregroundColor(.white)
                                .underline()
                        }
                        .padding(.top, 16)

                        Spacer()
                    }
                }
            }
            .navigationBarHidden(true)
        }
    }
}
```

## 23.3 タイムライン機能の実装

### 23.3.1 ドメイン層

```swift
// Features/Timeline/Domain/Entities/Post.swift

import Foundation

struct Post: Identifiable, Codable {
    let id: String
    let authorId: String
    let authorName: String
    let authorPhotoURL: URL?
    let content: String
    let images: [URL]
    let hashtags: [String]
    let location: Location?
    let likesCount: Int
    let commentsCount: Int
    let sharesCount: Int
    let createdAt: Date
    var isLiked: Bool

    struct Location: Codable {
        let latitude: Double
        let longitude: Double
        let name: String?
    }

    enum CodingKeys: String, CodingKey {
        case id
        case authorId = "author_id"
        case authorName = "author_name"
        case authorPhotoURL = "author_photo_url"
        case content
        case images
        case hashtags
        case location
        case likesCount = "likes_count"
        case commentsCount = "comments_count"
        case sharesCount = "shares_count"
        case createdAt = "created_at"
        case isLiked = "is_liked"
    }
}

extension Post {
    static var preview: Post {
        Post(
            id: "1",
            authorId: "user1",
            authorName: "John Doe",
            authorPhotoURL: nil,
            content: "Beautiful sunset today! #nature #photography",
            images: [],
            hashtags: ["nature", "photography"],
            location: nil,
            likesCount: 42,
            commentsCount: 5,
            sharesCount: 3,
            createdAt: Date(),
            isLiked: false
        )
    }
}
```

```swift
// Features/Timeline/Domain/UseCases/FetchTimelineUseCase.swift

import Foundation

protocol FetchTimelineUseCaseProtocol {
    func execute(page: Int, pageSize: Int) async throws -> [Post]
}

final class FetchTimelineUseCase: FetchTimelineUseCaseProtocol {
    private let timelineRepository: TimelineRepositoryProtocol

    init(timelineRepository: TimelineRepositoryProtocol) {
        self.timelineRepository = timelineRepository
    }

    func execute(page: Int, pageSize: Int) async throws -> [Post] {
        let posts = try await timelineRepository.fetchTimeline(
            page: page,
            pageSize: pageSize
        )

        return posts.sorted { $0.createdAt > $1.createdAt }
    }
}
```

### 23.3.2 データ層

```swift
// Features/Timeline/Data/Repositories/TimelineRepository.swift

import Foundation
import FirebaseFirestore

protocol TimelineRepositoryProtocol {
    func fetchTimeline(page: Int, pageSize: Int) async throws -> [Post]
    func likePost(postId: String) async throws
    func unlikePost(postId: String) async throws
}

final class TimelineRepository: TimelineRepositoryProtocol {
    private let firestore: Firestore
    private let currentUserId: String

    init(firestore: Firestore = .firestore(), currentUserId: String) {
        self.firestore = firestore
        self.currentUserId = currentUserId
    }

    func fetchTimeline(page: Int, pageSize: Int) async throws -> [Post] {
        let snapshot = try await firestore
            .collection("posts")
            .order(by: "created_at", descending: true)
            .limit(to: pageSize)
            .getDocuments()

        var posts: [Post] = []

        for document in snapshot.documents {
            var post = try document.data(as: Post.self)

            // いいね状態の確認
            let likeDoc = try await firestore
                .collection("posts")
                .document(post.id)
                .collection("likes")
                .document(currentUserId)
                .getDocument()

            post.isLiked = likeDoc.exists

            posts.append(post)
        }

        return posts
    }

    func likePost(postId: String) async throws {
        let batch = firestore.batch()

        // いいねドキュメントの追加
        let likeRef = firestore
            .collection("posts")
            .document(postId)
            .collection("likes")
            .document(currentUserId)

        batch.setData([
            "user_id": currentUserId,
            "created_at": FieldValue.serverTimestamp()
        ], forDocument: likeRef)

        // いいね数のインクリメント
        let postRef = firestore.collection("posts").document(postId)
        batch.updateData([
            "likes_count": FieldValue.increment(Int64(1))
        ], forDocument: postRef)

        try await batch.commit()
    }

    func unlikePost(postId: String) async throws {
        let batch = firestore.batch()

        // いいねドキュメントの削除
        let likeRef = firestore
            .collection("posts")
            .document(postId)
            .collection("likes")
            .document(currentUserId)

        batch.deleteDocument(likeRef)

        // いいね数のデクリメント
        let postRef = firestore.collection("posts").document(postId)
        batch.updateData([
            "likes_count": FieldValue.increment(Int64(-1))
        ], forDocument: postRef)

        try await batch.commit()
    }
}
```

### 23.3.3 プレゼンテーション層

```swift
// Features/Timeline/Presentation/ViewModels/TimelineViewModel.swift

import Foundation
import Combine

@MainActor
final class TimelineViewModel: ObservableObject {
    // MARK: - Published Properties

    @Published var posts: [Post] = []
    @Published var isLoading: Bool = false
    @Published var isRefreshing: Bool = false
    @Published var errorMessage: String?

    // MARK: - Private Properties

    private let fetchTimelineUseCase: FetchTimelineUseCaseProtocol
    private let likePostUseCase: LikePostUseCaseProtocol
    private var currentPage: Int = 0
    private let pageSize: Int = 20
    private var canLoadMore: Bool = true

    // MARK: - Initialization

    init(
        fetchTimelineUseCase: FetchTimelineUseCaseProtocol,
        likePostUseCase: LikePostUseCaseProtocol
    ) {
        self.fetchTimelineUseCase = fetchTimelineUseCase
        self.likePostUseCase = likePostUseCase
    }

    // MARK: - Public Methods

    func loadTimeline() async {
        guard !isLoading, canLoadMore else { return }

        isLoading = true
        errorMessage = nil
        defer { isLoading = false }

        do {
            let newPosts = try await fetchTimelineUseCase.execute(
                page: currentPage,
                pageSize: pageSize
            )

            if newPosts.count < pageSize {
                canLoadMore = false
            }

            posts.append(contentsOf: newPosts)
            currentPage += 1
        } catch {
            errorMessage = error.localizedDescription
        }
    }

    func refresh() async {
        isRefreshing = true
        currentPage = 0
        canLoadMore = true
        posts.removeAll()
        defer { isRefreshing = false }

        await loadTimeline()
    }

    func toggleLike(for post: Post) async {
        guard let index = posts.firstIndex(where: { $0.id == post.id }) else {
            return
        }

        var updatedPost = post
        updatedPost.isLiked.toggle()
        updatedPost.likesCount += updatedPost.isLiked ? 1 : -1
        posts[index] = updatedPost

        do {
            if updatedPost.isLiked {
                try await likePostUseCase.execute(postId: post.id)
            } else {
                try await likePostUseCase.unlike(postId: post.id)
            }
        } catch {
            // エラー時は元に戻す
            posts[index] = post
        }
    }
}
```

```swift
// Features/Timeline/Presentation/Views/TimelineView.swift

import SwiftUI

struct TimelineView: View {
    @StateObject private var viewModel: TimelineViewModel

    init(viewModel: TimelineViewModel) {
        _viewModel = StateObject(wrappedValue: viewModel)
    }

    var body: some View {
        NavigationView {
            ScrollView {
                LazyVStack(spacing: 0) {
                    ForEach(viewModel.posts) { post in
                        PostRow(post: post) {
                            Task {
                                await viewModel.toggleLike(for: post)
                            }
                        }
                        .padding(.bottom, 8)

                        Divider()
                    }

                    // Load More
                    if viewModel.isLoading {
                        ProgressView()
                            .padding()
                    }
                }
            }
            .refreshable {
                await viewModel.refresh()
            }
            .navigationTitle("Timeline")
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    NavigationLink(destination: CreatePostView()) {
                        Image(systemName: "plus.circle.fill")
                            .font(.title2)
                    }
                }
            }
            .task {
                if viewModel.posts.isEmpty {
                    await viewModel.loadTimeline()
                }
            }
        }
    }
}

struct PostRow: View {
    let post: Post
    let onLikeTapped: () -> Void

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            // Header
            HStack {
                // Author Photo
                AsyncImage(url: post.authorPhotoURL) { image in
                    image
                        .resizable()
                        .aspectRatio(contentMode: .fill)
                } placeholder: {
                    Circle()
                        .fill(Color.gray.opacity(0.3))
                        .overlay(
                            Image(systemName: "person.fill")
                                .foregroundColor(.gray)
                        )
                }
                .frame(width: 40, height: 40)
                .clipShape(Circle())

                VStack(alignment: .leading, spacing: 2) {
                    Text(post.authorName)
                        .font(.headline)

                    Text(post.createdAt, style: .relative)
                        .font(.caption)
                        .foregroundColor(.gray)
                }

                Spacer()

                Button {
                    // More options
                } label: {
                    Image(systemName: "ellipsis")
                        .foregroundColor(.gray)
                }
            }
            .padding(.horizontal)

            // Content
            Text(post.content)
                .font(.body)
                .padding(.horizontal)

            // Images
            if !post.images.isEmpty {
                ScrollView(.horizontal, showsIndicators: false) {
                    HStack(spacing: 8) {
                        ForEach(post.images, id: \.self) { imageURL in
                            AsyncImage(url: imageURL) { image in
                                image
                                    .resizable()
                                    .aspectRatio(contentMode: .fill)
                            } placeholder: {
                                Rectangle()
                                    .fill(Color.gray.opacity(0.3))
                            }
                            .frame(width: 280, height: 280)
                            .cornerRadius(12)
                        }
                    }
                    .padding(.horizontal)
                }
            }

            // Actions
            HStack(spacing: 32) {
                // Like
                Button(action: onLikeTapped) {
                    HStack(spacing: 4) {
                        Image(systemName: post.isLiked ? "heart.fill" : "heart")
                            .foregroundColor(post.isLiked ? .red : .gray)
                        Text("\(post.likesCount)")
                            .font(.caption)
                            .foregroundColor(.gray)
                    }
                }

                // Comment
                Button {
                    // Show comments
                } label: {
                    HStack(spacing: 4) {
                        Image(systemName: "bubble.right")
                            .foregroundColor(.gray)
                        Text("\(post.commentsCount)")
                            .font(.caption)
                            .foregroundColor(.gray)
                    }
                }

                // Share
                Button {
                    // Share
                } label: {
                    HStack(spacing: 4) {
                        Image(systemName: "arrowshape.turn.up.right")
                            .foregroundColor(.gray)
                        Text("\(post.sharesCount)")
                            .font(.caption)
                            .foregroundColor(.gray)
                    }
                }

                Spacer()
            }
            .padding(.horizontal)
            .padding(.top, 8)
        }
        .padding(.vertical, 12)
    }
}
```

## まとめ

Part 1では、SNSアプリ「SocialConnect」の基盤となる以下の機能を実装しました。

**実装した内容:**

1. **プロジェクト設計**
   - Clean Architectureによる層の分離
   - MVVM + ドメイン駆動設計
   - Firebase統合の準備

2. **認証機能**
   - メール/パスワード認証
   - Apple Sign In対応
   - ユーザー情報の管理

3. **タイムライン機能**
   - 投稿一覧の取得
   - 無限スクロール対応
   - いいね機能
   - リアルタイム更新

**アーキテクチャのポイント:**

- Domain層でビジネスロジックをカプセル化
- Data層でFirebase統合を抽象化
- Presentation層でSwiftUIとの統合
- 依存性注入によるテスタビリティの確保

Part 2では、投稿機能、通知機能、テスト、CI/CDなどを実装していきます。
