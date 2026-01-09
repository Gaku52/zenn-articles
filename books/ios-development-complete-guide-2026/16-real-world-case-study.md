---
title: "Chapter 16: 実戦ケーススタディ - SNSアプリ完全実装"
---

# Chapter 16: 実戦ケーススタディ - SNSアプリ完全実装

## 16.1 プロジェクト概要

### 16.1.1 アプリケーション仕様

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

### 16.1.2 プロジェクト構成

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

## 16.2 認証機能の実装

### 16.2.1 ドメイン層

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

### 16.2.2 データ層

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

### 16.2.3 プレゼンテーション層

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

## 16.3 タイムライン機能の実装

### 16.3.1 ドメイン層

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

### 16.3.2 データ層

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

### 16.3.3 プレゼンテーション層

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

## 16.4 投稿機能の実装

### 16.4.1 ドメイン層

```swift
// Features/Post/Domain/UseCases/CreatePostUseCase.swift

import Foundation
import UIKit

protocol CreatePostUseCaseProtocol {
    func execute(
        content: String,
        images: [UIImage],
        location: Post.Location?
    ) async throws -> Post
}

final class CreatePostUseCase: CreatePostUseCaseProtocol {
    private let postRepository: PostRepositoryProtocol
    private let storageService: StorageServiceProtocol

    init(
        postRepository: PostRepositoryProtocol,
        storageService: StorageServiceProtocol
    ) {
        self.postRepository = postRepository
        self.storageService = storageService
    }

    func execute(
        content: String,
        images: [UIImage],
        location: Post.Location?
    ) async throws -> Post {
        // バリデーション
        guard !content.isEmpty || !images.isEmpty else {
            throw PostError.emptyContent
        }

        // 画像のアップロード
        var imageURLs: [URL] = []

        for (index, image) in images.enumerated() {
            let url = try await storageService.uploadImage(
                image,
                path: "posts/\(UUID().uuidString)_\(index).jpg"
            )
            imageURLs.append(url)
        }

        // ハッシュタグの抽出
        let hashtags = extractHashtags(from: content)

        // 投稿の作成
        let post = try await postRepository.createPost(
            content: content,
            images: imageURLs,
            hashtags: hashtags,
            location: location
        )

        return post
    }

    private func extractHashtags(from text: String) -> [String] {
        let pattern = "#[\\w]+"
        let regex = try? NSRegularExpression(pattern: pattern)
        let nsString = text as NSString
        let results = regex?.matches(in: text, range: NSRange(location: 0, length: nsString.length))

        return results?.compactMap { result in
            let hashtag = nsString.substring(with: result.range)
            return String(hashtag.dropFirst()) // Remove #
        } ?? []
    }
}
```

### 16.4.2 データ層

```swift
// Features/Post/Data/Repositories/PostRepository.swift

import Foundation
import FirebaseFirestore

protocol PostRepositoryProtocol {
    func createPost(
        content: String,
        images: [URL],
        hashtags: [String],
        location: Post.Location?
    ) async throws -> Post
    func deletePost(postId: String) async throws
    func updatePost(postId: String, content: String) async throws
}

final class PostRepository: PostRepositoryProtocol {
    private let firestore: Firestore
    private let currentUser: User

    init(firestore: Firestore = .firestore(), currentUser: User) {
        self.firestore = firestore
        self.currentUser = currentUser
    }

    func createPost(
        content: String,
        images: [URL],
        hashtags: [String],
        location: Post.Location?
    ) async throws -> Post {
        let postRef = firestore.collection("posts").document()

        let post = Post(
            id: postRef.documentID,
            authorId: currentUser.id,
            authorName: currentUser.displayName,
            authorPhotoURL: currentUser.photoURL,
            content: content,
            images: images,
            hashtags: hashtags,
            location: location,
            likesCount: 0,
            commentsCount: 0,
            sharesCount: 0,
            createdAt: Date(),
            isLiked: false
        )

        let data = try Firestore.Encoder().encode(post)
        try await postRef.setData(data)

        // ユーザーの投稿数をインクリメント
        let userRef = firestore.collection("users").document(currentUser.id)
        try await userRef.updateData([
            "posts_count": FieldValue.increment(Int64(1))
        ])

        return post
    }

    func deletePost(postId: String) async throws {
        let batch = firestore.batch()

        // 投稿の削除
        let postRef = firestore.collection("posts").document(postId)
        batch.deleteDocument(postRef)

        // ユーザーの投稿数をデクリメント
        let userRef = firestore.collection("users").document(currentUser.id)
        batch.updateData([
            "posts_count": FieldValue.increment(Int64(-1))
        ], forDocument: userRef)

        try await batch.commit()
    }

    func updatePost(postId: String, content: String) async throws {
        let postRef = firestore.collection("posts").document(postId)
        try await postRef.updateData([
            "content": content,
            "updated_at": FieldValue.serverTimestamp()
        ])
    }
}
```

### 16.4.3 プレゼンテーション層

```swift
// Features/Post/Presentation/ViewModels/CreatePostViewModel.swift

import Foundation
import UIKit
import Combine

@MainActor
final class CreatePostViewModel: ObservableObject {
    // MARK: - Published Properties

    @Published var content: String = ""
    @Published var selectedImages: [UIImage] = []
    @Published var isLoading: Bool = false
    @Published var errorMessage: String?
    @Published var isPosted: Bool = false

    // MARK: - Private Properties

    private let createPostUseCase: CreatePostUseCaseProtocol
    private let maxImages: Int = 4

    // MARK: - Initialization

    init(createPostUseCase: CreatePostUseCaseProtocol) {
        self.createPostUseCase = createPostUseCase
    }

    // MARK: - Public Methods

    func addImage(_ image: UIImage) {
        guard selectedImages.count < maxImages else {
            errorMessage = "最大\(maxImages)枚まで選択できます"
            return
        }
        selectedImages.append(image)
    }

    func removeImage(at index: Int) {
        selectedImages.remove(at: index)
    }

    func createPost() async {
        isLoading = true
        errorMessage = nil
        defer { isLoading = false }

        do {
            _ = try await createPostUseCase.execute(
                content: content,
                images: selectedImages,
                location: nil
            )
            isPosted = true
        } catch {
            errorMessage = error.localizedDescription
        }
    }

    var canPost: Bool {
        !content.isEmpty || !selectedImages.isEmpty
    }
}
```

```swift
// Features/Post/Presentation/Views/CreatePostView.swift

import SwiftUI
import PhotosUI

struct CreatePostView: View {
    @StateObject private var viewModel: CreatePostViewModel
    @Environment(\.dismiss) private var dismiss
    @State private var selectedItem: PhotosPickerItem?

    init(viewModel: CreatePostViewModel) {
        _viewModel = StateObject(wrappedValue: viewModel)
    }

    var body: some View {
        NavigationView {
            VStack(spacing: 0) {
                // Content Editor
                TextEditor(text: $viewModel.content)
                    .frame(minHeight: 150)
                    .padding()
                    .background(Color(.systemGray6))
                    .cornerRadius(12)
                    .padding()

                // Image Grid
                if !viewModel.selectedImages.isEmpty {
                    ScrollView(.horizontal, showsIndicators: false) {
                        HStack(spacing: 12) {
                            ForEach(Array(viewModel.selectedImages.enumerated()), id: \.offset) { index, image in
                                ZStack(alignment: .topTrailing) {
                                    Image(uiImage: image)
                                        .resizable()
                                        .aspectRatio(contentMode: .fill)
                                        .frame(width: 120, height: 120)
                                        .cornerRadius(12)
                                        .clipped()

                                    Button {
                                        viewModel.removeImage(at: index)
                                    } label: {
                                        Image(systemName: "xmark.circle.fill")
                                            .foregroundColor(.white)
                                            .background(Circle().fill(Color.black.opacity(0.6)))
                                    }
                                    .padding(8)
                                }
                            }
                        }
                        .padding(.horizontal)
                    }
                    .padding(.bottom)
                }

                Spacer()

                // Actions
                HStack(spacing: 20) {
                    // Add Photo
                    PhotosPicker(
                        selection: $selectedItem,
                        matching: .images
                    ) {
                        Label("Photo", systemImage: "photo")
                            .foregroundColor(.blue)
                    }
                    .onChange(of: selectedItem) { newItem in
                        Task {
                            if let data = try? await newItem?.loadTransferable(type: Data.self),
                               let image = UIImage(data: data) {
                                viewModel.addImage(image)
                            }
                        }
                    }

                    Spacer()

                    // Character Count
                    Text("\(viewModel.content.count)/280")
                        .font(.caption)
                        .foregroundColor(.gray)
                }
                .padding()
                .background(Color(.systemGray6))
            }
            .navigationTitle("New Post")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .navigationBarLeading) {
                    Button("Cancel") {
                        dismiss()
                    }
                }

                ToolbarItem(placement: .navigationBarTrailing) {
                    Button {
                        Task {
                            await viewModel.createPost()
                        }
                    } label: {
                        if viewModel.isLoading {
                            ProgressView()
                        } else {
                            Text("Post")
                                .fontWeight(.semibold)
                        }
                    }
                    .disabled(!viewModel.canPost || viewModel.isLoading)
                }
            }
            .onChange(of: viewModel.isPosted) { isPosted in
                if isPosted {
                    dismiss()
                }
            }
            .alert("Error", isPresented: .constant(viewModel.errorMessage != nil)) {
                Button("OK") {
                    viewModel.errorMessage = nil
                }
            } message: {
                if let errorMessage = viewModel.errorMessage {
                    Text(errorMessage)
                }
            }
        }
    }
}
```

## 16.5 通知機能の実装

### 16.5.1 Push Notification設定

```swift
// Core/Services/PushNotificationService.swift

import Foundation
import UserNotifications
import FirebaseMessaging

final class PushNotificationService: NSObject {
    static let shared = PushNotificationService()

    private override init() {
        super.init()
    }

    func configure() {
        UNUserNotificationCenter.current().delegate = self
        Messaging.messaging().delegate = self

        requestAuthorization()
    }

    private func requestAuthorization() {
        UNUserNotificationCenter.current().requestAuthorization(
            options: [.alert, .sound, .badge]
        ) { granted, error in
            if granted {
                DispatchQueue.main.async {
                    UIApplication.shared.registerForRemoteNotifications()
                }
            }
        }
    }

    func updateToken(_ token: String) async {
        guard let userId = Auth.auth().currentUser?.uid else { return }

        let firestore = Firestore.firestore()
        try? await firestore
            .collection("users")
            .document(userId)
            .updateData([
                "fcm_token": token,
                "token_updated_at": FieldValue.serverTimestamp()
            ])
    }
}

// MARK: - UNUserNotificationCenterDelegate

extension PushNotificationService: UNUserNotificationCenterDelegate {
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        willPresent notification: UNNotification,
        withCompletionHandler completionHandler: @escaping (UNNotificationPresentationOptions) -> Void
    ) {
        completionHandler([.banner, .sound, .badge])
    }

    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        didReceive response: UNNotificationResponse,
        withCompletionHandler completionHandler: @escaping () -> Void
    ) {
        let userInfo = response.notification.request.content.userInfo

        // Handle notification tap
        handleNotificationTap(userInfo: userInfo)

        completionHandler()
    }

    private func handleNotificationTap(userInfo: [AnyHashable: Any]) {
        guard let type = userInfo["type"] as? String else { return }

        switch type {
        case "like":
            if let postId = userInfo["post_id"] as? String {
                // Navigate to post
                NotificationCenter.default.post(
                    name: .navigateToPost,
                    object: nil,
                    userInfo: ["postId": postId]
                )
            }
        case "comment":
            if let postId = userInfo["post_id"] as? String {
                // Navigate to post comments
                NotificationCenter.default.post(
                    name: .navigateToComments,
                    object: nil,
                    userInfo: ["postId": postId]
                )
            }
        case "follow":
            if let userId = userInfo["user_id"] as? String {
                // Navigate to user profile
                NotificationCenter.default.post(
                    name: .navigateToProfile,
                    object: nil,
                    userInfo: ["userId": userId]
                )
            }
        default:
            break
        }
    }
}

// MARK: - MessagingDelegate

extension PushNotificationService: MessagingDelegate {
    func messaging(_ messaging: Messaging, didReceiveRegistrationToken fcmToken: String?) {
        guard let token = fcmToken else { return }

        Task {
            await updateToken(token)
        }
    }
}

// MARK: - Notification Names

extension Notification.Name {
    static let navigateToPost = Notification.Name("navigateToPost")
    static let navigateToComments = Notification.Name("navigateToComments")
    static let navigateToProfile = Notification.Name("navigateToProfile")
}
```

## 16.6 テスト実装

### 16.6.1 Unit Tests

```swift
// SocialConnectTests/Features/Authentication/LoginViewModelTests.swift

import XCTest
@testable import SocialConnect

@MainActor
final class LoginViewModelTests: XCTestCase {
    var viewModel: LoginViewModel!
    var mockLoginUseCase: MockLoginUseCase!

    override func setUp() {
        super.setUp()
        mockLoginUseCase = MockLoginUseCase()
        viewModel = LoginViewModel(loginUseCase: mockLoginUseCase)
    }

    override func tearDown() {
        viewModel = nil
        mockLoginUseCase = nil
        super.tearDown()
    }

    func testLoginSuccess() async {
        // Given
        viewModel.email = "test@example.com"
        viewModel.password = "password123"
        mockLoginUseCase.shouldSucceed = true

        // When
        await viewModel.login()

        // Then
        XCTAssertTrue(viewModel.isLoggedIn)
        XCTAssertNil(viewModel.errorMessage)
        XCTAssertFalse(viewModel.isLoading)
    }

    func testLoginFailure() async {
        // Given
        viewModel.email = "test@example.com"
        viewModel.password = "wrong"
        mockLoginUseCase.shouldSucceed = false

        // When
        await viewModel.login()

        // Then
        XCTAssertFalse(viewModel.isLoggedIn)
        XCTAssertNotNil(viewModel.errorMessage)
        XCTAssertFalse(viewModel.isLoading)
    }

    func testIsValidReturnsFalseWhenEmailEmpty() {
        // Given
        viewModel.email = ""
        viewModel.password = "password"

        // Then
        XCTAssertFalse(viewModel.isValid)
    }

    func testIsValidReturnsFalseWhenPasswordEmpty() {
        // Given
        viewModel.email = "test@example.com"
        viewModel.password = ""

        // Then
        XCTAssertFalse(viewModel.isValid)
    }

    func testIsValidReturnsTrueWhenBothFieldsFilled() {
        // Given
        viewModel.email = "test@example.com"
        viewModel.password = "password"

        // Then
        XCTAssertTrue(viewModel.isValid)
    }
}

// MARK: - Mock

final class MockLoginUseCase: LoginUseCaseProtocol {
    var shouldSucceed = true

    func execute(email: String, password: String) async throws -> User {
        if shouldSucceed {
            return User.preview
        } else {
            throw AuthError.invalidCredentials
        }
    }
}
```

### 16.6.2 UI Tests

```swift
// SocialConnectUITests/LoginFlowUITests.swift

import XCTest

final class LoginFlowUITests: XCTestCase {
    var app: XCUIApplication!

    override func setUp() {
        super.setUp()
        continueAfterFailure = false

        app = XCUIApplication()
        app.launchArguments = ["-uitesting"]
        app.launch()
    }

    func testLoginWithValidCredentials() {
        // Given
        let emailTextField = app.textFields["Email"]
        let passwordSecureField = app.secureTextFields["Password"]
        let loginButton = app.buttons["Login"]

        // When
        emailTextField.tap()
        emailTextField.typeText("test@example.com")

        passwordSecureField.tap()
        passwordSecureField.typeText("password123")

        loginButton.tap()

        // Then
        let timelineTitle = app.navigationBars["Timeline"]
        XCTAssertTrue(timelineTitle.waitForExistence(timeout: 5))
    }

    func testLoginWithInvalidCredentials() {
        // Given
        let emailTextField = app.textFields["Email"]
        let passwordSecureField = app.secureTextFields["Password"]
        let loginButton = app.buttons["Login"]

        // When
        emailTextField.tap()
        emailTextField.typeText("invalid@example.com")

        passwordSecureField.tap()
        passwordSecureField.typeText("wrong")

        loginButton.tap()

        // Then
        let errorMessage = app.staticTexts["Invalid credentials"]
        XCTAssertTrue(errorMessage.waitForExistence(timeout: 3))
    }

    func testNavigateToSignUp() {
        // Given
        let signUpLink = app.buttons["Don't have an account? Sign Up"]

        // When
        signUpLink.tap()

        // Then
        let signUpTitle = app.navigationBars["Sign Up"]
        XCTAssertTrue(signUpTitle.exists)
    }
}
```

## 16.7 CI/CD設定

### 16.7.1 GitHub Actions

```yaml
# .github/workflows/ios-ci.yml

name: iOS CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

env:
  XCODE_VERSION: '15.2'

jobs:
  lint:
    name: SwiftLint
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4

      - name: Install SwiftLint
        run: brew install swiftlint

      - name: Run SwiftLint
        run: swiftlint lint --strict

  test:
    name: Unit & UI Tests
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_${{ env.XCODE_VERSION }}.app

      - name: Cache SPM
        uses: actions/cache@v3
        with:
          path: .build
          key: ${{ runner.os }}-spm-${{ hashFiles('**/Package.resolved') }}
          restore-keys: |
            ${{ runner.os }}-spm-

      - name: Build and Test
        run: |
          xcodebuild clean test \
            -scheme SocialConnect \
            -destination 'platform=iOS Simulator,name=iPhone 15 Pro,OS=17.2' \
            -enableCodeCoverage YES \
            -resultBundlePath TestResults.xcresult

      - name: Generate Coverage Report
        run: |
          xcrun xccov view --report --json TestResults.xcresult > coverage.json

      - name: Upload Coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.json
          fail_ci_if_error: true

  build:
    name: Build
    runs-on: macos-14
    needs: [lint, test]
    strategy:
      matrix:
        configuration: [Debug, Release]
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_${{ env.XCODE_VERSION }}.app

      - name: Build
        run: |
          xcodebuild build \
            -scheme SocialConnect \
            -configuration ${{ matrix.configuration }} \
            -destination 'platform=iOS Simulator,name=iPhone 15 Pro' \
            CODE_SIGNING_ALLOWED=NO
```

### 16.7.2 Fastlane設定

```ruby
# fastlane/Fastfile

default_platform(:ios)

platform :ios do
  desc "Run tests"
  lane :test do
    run_tests(
      scheme: "SocialConnect",
      devices: ["iPhone 15 Pro"],
      code_coverage: true
    )
  end

  desc "Build for TestFlight"
  lane :beta do
    # Match certificates
    match(
      type: "appstore",
      readonly: is_ci
    )

    # Increment build number
    increment_build_number(
      build_number: latest_testflight_build_number + 1
    )

    # Build app
    build_app(
      scheme: "SocialConnect",
      export_method: "app-store",
      xcargs: "-allowProvisioningUpdates"
    )

    # Upload to TestFlight
    upload_to_testflight(
      skip_waiting_for_build_processing: true,
      changelog: changelog_from_git_commits
    )

    # Send Slack notification
    slack(
      message: "New build uploaded to TestFlight!",
      success: true,
      slack_url: ENV["SLACK_WEBHOOK_URL"]
    )
  end

  desc "Release to App Store"
  lane :release do
    # Ensure we're on main branch
    ensure_git_branch(branch: "main")
    ensure_git_status_clean

    # Match certificates
    match(type: "appstore")

    # Capture screenshots
    capture_screenshots

    # Build and upload
    beta

    # Submit for review
    deliver(
      submit_for_review: true,
      automatic_release: false,
      force: true,
      skip_metadata: false,
      skip_screenshots: false
    )
  end
end
```

## まとめ

この章では、実戦的なSNSアプリケーション「SocialConnect」の完全実装を通じて、以下の内容を学びました。

**実装した主要機能:**

1. **認証機能**
   - メール/パスワード認証
   - Apple Sign In対応
   - Firebase Authenticationの統合

2. **タイムライン機能**
   - 投稿一覧表示
   - 無限スクロール
   - リアルタイム更新
   - いいね機能

3. **投稿機能**
   - テキスト投稿
   - 画像アップロード（最大4枚）
   - ハッシュタグ抽出
   - Firebase Storageの活用

4. **通知機能**
   - Push Notification
   - Firebase Cloud Messaging
   - 通知ハンドリング

5. **テスト**
   - Unit Tests
   - UI Tests
   - テストカバレッジ

6. **CI/CD**
   - GitHub Actions
   - Fastlane
   - 自動デプロイ

**アーキテクチャのポイント:**

- **Clean Architecture**: Domain/Data/Presentationの分離
- **MVVM**: ViewとViewModelの責務分離
- **依存性注入**: テストしやすい設計
- **リポジトリパターン**: データソースの抽象化
- **ユースケース**: ビジネスロジックのカプセル化

このケーススタディを通じて、本書で学んだすべてのベストプラクティスを実践的に適用し、プロダクション品質のiOSアプリケーションを構築する方法を習得できました。
