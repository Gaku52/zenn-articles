---
title: "実戦ケーススタディ Part 1 - SNSアプリUI"
---

# 実戦ケーススタディ Part 1 - SNSアプリUI

本章では、これまで学んだすべての技術を統合し、**実際のSNSアプリケーション**を構築します。完全なアーキテクチャ設計から実装まで、プロダクションレディなコードを提供します。

## アプリケーション要件

### 機能要件

**コア機能:**
- タイムライン表示(無限スクロール)
- 投稿作成・編集・削除
- いいね・コメント機能
- ユーザープロフィール
- リアルタイム更新

**非機能要件:**
- 60FPS維持
- 起動時間 < 1秒
- メモリ使用量 < 100MB
- オフライン対応
- アクセシビリティ準拠

## アーキテクチャ設計

### レイヤー構造

```
┌─────────────────────────────────┐
│      Presentation Layer         │
│  (SwiftUI Views + ViewModels)   │
├─────────────────────────────────┤
│       Domain Layer              │
│     (Use Cases + Entities)      │
├─────────────────────────────────┤
│        Data Layer               │
│  (Repositories + Data Sources)  │
└─────────────────────────────────┘
```

### ドメインモデル

```swift
// MARK: - Domain Entities

struct Post: Identifiable, Codable, Equatable {
    let id: String
    let authorId: String
    var content: String
    var imageURLs: [URL]
    var likeCount: Int
    var commentCount: Int
    var createdAt: Date
    var updatedAt: Date
    var isLikedByCurrentUser: Bool

    var author: User?
}

struct User: Identifiable, Codable, Equatable {
    let id: String
    var username: String
    var displayName: String
    var avatarURL: URL?
    var bio: String
    var followerCount: Int
    var followingCount: Int
    var isFollowedByCurrentUser: Bool
}

struct Comment: Identifiable, Codable, Equatable {
    let id: String
    let postId: String
    let authorId: String
    var content: String
    var createdAt: Date

    var author: User?
}

// MARK: - Use Cases

protocol PostUseCaseProtocol {
    func fetchTimeline(page: Int) async throws -> [Post]
    func createPost(content: String, images: [Data]) async throws -> Post
    func likePost(id: String) async throws
    func unlikePost(id: String) async throws
    func deletePost(id: String) async throws
}

class PostUseCase: PostUseCaseProtocol {
    private let repository: PostRepositoryProtocol

    init(repository: PostRepositoryProtocol) {
        self.repository = repository
    }

    func fetchTimeline(page: Int) async throws -> [Post] {
        try await repository.fetchTimeline(page: page)
    }

    func createPost(content: String, images: [Data]) async throws -> Post {
        // バリデーション
        guard !content.isEmpty || !images.isEmpty else {
            throw PostError.emptyContent
        }

        guard content.count <= 500 else {
            throw PostError.contentTooLong
        }

        // 画像アップロード
        var imageURLs: [URL] = []
        for imageData in images {
            let url = try await repository.uploadImage(imageData)
            imageURLs.append(url)
        }

        // 投稿作成
        return try await repository.createPost(content: content, imageURLs: imageURLs)
    }

    func likePost(id: String) async throws {
        try await repository.likePost(id: id)
    }

    func unlikePost(id: String) async throws {
        try await repository.unlikePost(id: id)
    }

    func deletePost(id: String) async throws {
        try await repository.deletePost(id: id)
    }
}

enum PostError: LocalizedError {
    case emptyContent
    case contentTooLong
    case uploadFailed

    var errorDescription: String? {
        switch self {
        case .emptyContent:
            return "投稿内容が空です"
        case .contentTooLong:
            return "投稿内容が長すぎます(最大500文字)"
        case .uploadFailed:
            return "画像のアップロードに失敗しました"
        }
    }
}

// MARK: - Repository Protocol

protocol PostRepositoryProtocol {
    func fetchTimeline(page: Int) async throws -> [Post]
    func createPost(content: String, imageURLs: [URL]) async throws -> Post
    func uploadImage(_ data: Data) async throws -> URL
    func likePost(id: String) async throws
    func unlikePost(id: String) async throws
    func deletePost(id: String) async throws
}
```

## タイムライン実装

### TimelineViewModel

```swift
import Observation
import Combine

@Observable
class TimelineViewModel {
    var posts: [Post] = []
    var isLoading = false
    var isLoadingMore = false
    var error: Error?
    var hasMorePages = true

    private let postUseCase: PostUseCaseProtocol
    private var currentPage = 0
    private let pageSize = 20

    init(postUseCase: PostUseCaseProtocol) {
        self.postUseCase = postUseCase
    }

    @MainActor
    func loadInitialPosts() async {
        guard !isLoading else { return }

        isLoading = true
        error = nil
        currentPage = 0

        do {
            posts = try await postUseCase.fetchTimeline(page: currentPage)
            hasMorePages = posts.count >= pageSize
            currentPage += 1
        } catch {
            self.error = error
        }

        isLoading = false
    }

    @MainActor
    func loadMorePosts() async {
        guard !isLoadingMore && hasMorePages else { return }

        isLoadingMore = true

        do {
            let newPosts = try await postUseCase.fetchTimeline(page: currentPage)

            if newPosts.isEmpty {
                hasMorePages = false
            } else {
                posts.append(contentsOf: newPosts)
                currentPage += 1
            }
        } catch {
            self.error = error
        }

        isLoadingMore = false
    }

    @MainActor
    func toggleLike(for post: Post) async {
        // 楽観的更新
        if let index = posts.firstIndex(where: { $0.id == post.id }) {
            var updatedPost = posts[index]
            updatedPost.isLikedByCurrentUser.toggle()
            updatedPost.likeCount += updatedPost.isLikedByCurrentUser ? 1 : -1
            posts[index] = updatedPost
        }

        do {
            if post.isLikedByCurrentUser {
                try await postUseCase.unlikePost(id: post.id)
            } else {
                try await postUseCase.likePost(id: post.id)
            }
        } catch {
            // エラー時はロールバック
            if let index = posts.firstIndex(where: { $0.id == post.id }) {
                var updatedPost = posts[index]
                updatedPost.isLikedByCurrentUser.toggle()
                updatedPost.likeCount += updatedPost.isLikedByCurrentUser ? 1 : -1
                posts[index] = updatedPost
            }
            self.error = error
        }
    }

    @MainActor
    func deletePost(_ post: Post) async {
        do {
            try await postUseCase.deletePost(id: post.id)
            posts.removeAll { $0.id == post.id }
        } catch {
            self.error = error
        }
    }

    @MainActor
    func refresh() async {
        await loadInitialPosts()
    }
}
```

### TimelineView

```swift
struct TimelineView: View {
    @State private var viewModel: TimelineViewModel

    init(postUseCase: PostUseCaseProtocol = PostUseCase(repository: PostRepository())) {
        _viewModel = State(initialValue: TimelineViewModel(postUseCase: postUseCase))
    }

    var body: some View {
        NavigationStack {
            ScrollView {
                LazyVStack(spacing: 0) {
                    ForEach(viewModel.posts) { post in
                        PostCard(
                            post: post,
                            onLike: {
                                Task {
                                    await viewModel.toggleLike(for: post)
                                }
                            },
                            onDelete: {
                                Task {
                                    await viewModel.deletePost(post)
                                }
                            }
                        )
                        .onAppear {
                            if post.id == viewModel.posts.last?.id {
                                Task {
                                    await viewModel.loadMorePosts()
                                }
                            }
                        }

                        Divider()
                    }

                    if viewModel.isLoadingMore {
                        ProgressView()
                            .frame(height: 60)
                    }
                }
            }
            .refreshable {
                await viewModel.refresh()
            }
            .overlay {
                if viewModel.isLoading && viewModel.posts.isEmpty {
                    ProgressView()
                }
            }
            .navigationTitle("タイムライン")
            .toolbar {
                ToolbarItem(placement: .primaryAction) {
                    NavigationLink {
                        CreatePostView()
                    } label: {
                        Image(systemName: "square.and.pencil")
                    }
                    .accessibilityLabel("新規投稿")
                }
            }
            .alert(error: $viewModel.error)
        }
        .task {
            await viewModel.loadInitialPosts()
        }
    }
}

// エラーアラート用のView拡張
extension View {
    func alert(error: Binding<Error?>) -> some View {
        alert(
            "エラー",
            isPresented: Binding(
                get: { error.wrappedValue != nil },
                set: { if !$0 { error.wrappedValue = nil } }
            )
        ) {
            Button("OK", role: .cancel) {
                error.wrappedValue = nil
            }
        } message: {
            if let error = error.wrappedValue {
                Text(error.localizedDescription)
            }
        }
    }
}
```

### PostCard コンポーネント

```swift
struct PostCard: View {
    let post: Post
    let onLike: () -> Void
    let onDelete: () -> Void

    @State private var showDeleteConfirmation = false
    @Environment(\.dynamicTypeSize) var dynamicTypeSize

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            // ヘッダー
            PostHeader(user: post.author ?? User.placeholder)

            // 本文
            Text(post.content)
                .font(.body)
                .lineLimit(nil)
                .padding(.horizontal)

            // 画像
            if !post.imageURLs.isEmpty {
                PostImageGallery(imageURLs: post.imageURLs)
            }

            // アクションボタン
            PostActions(
                post: post,
                onLike: onLike,
                onDelete: {
                    showDeleteConfirmation = true
                }
            )
            .padding(.horizontal)
        }
        .padding(.vertical, 12)
        .confirmationDialog(
            "この投稿を削除しますか?",
            isPresented: $showDeleteConfirmation,
            titleVisibility: .visible
        ) {
            Button("削除", role: .destructive, action: onDelete)
            Button("キャンセル", role: .cancel) { }
        }
        // アクセシビリティ
        .accessibilityElement(children: .contain)
        .accessibilityLabel("投稿")
    }
}

struct PostHeader: View {
    let user: User

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: user.avatarURL) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                Circle()
                    .fill(Color(.systemGray5))
            }
            .frame(width: 40, height: 40)
            .clipShape(Circle())
            .accessibilityHidden(true)

            VStack(alignment: .leading, spacing: 2) {
                Text(user.displayName)
                    .font(.headline)

                Text("@\(user.username)")
                    .font(.subheadline)
                    .foregroundColor(.secondary)
            }

            Spacer()
        }
        .padding(.horizontal)
    }
}

struct PostImageGallery: View {
    let imageURLs: [URL]

    var body: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            LazyHStack(spacing: 8) {
                ForEach(imageURLs, id: \.self) { url in
                    AsyncImage(url: url) { phase in
                        switch phase {
                        case .empty:
                            Rectangle()
                                .fill(Color(.systemGray6))
                                .overlay(ProgressView())
                        case .success(let image):
                            image
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                        case .failure:
                            Rectangle()
                                .fill(Color(.systemGray6))
                                .overlay(
                                    Image(systemName: "photo")
                                        .foregroundColor(.secondary)
                                )
                        @unknown default:
                            EmptyView()
                        }
                    }
                    .frame(width: 300, height: 200)
                    .clipShape(RoundedRectangle(cornerRadius: 12))
                }
            }
            .padding(.horizontal)
        }
    }
}

struct PostActions: View {
    let post: Post
    let onLike: () -> Void
    let onDelete: () -> Void

    var body: some View {
        HStack(spacing: 24) {
            Button(action: onLike) {
                Label("\(post.likeCount)", systemImage: post.isLikedByCurrentUser ? "heart.fill" : "heart")
                    .foregroundColor(post.isLikedByCurrentUser ? .red : .primary)
            }
            .accessibilityLabel(post.isLikedByCurrentUser ? "いいね済み" : "いいね")
            .accessibilityValue("\(post.likeCount)件")

            Button {
                // コメント画面へ遷移
            } label: {
                Label("\(post.commentCount)", systemImage: "bubble.right")
            }
            .accessibilityLabel("コメント")
            .accessibilityValue("\(post.commentCount)件")

            Button {
                // 共有
            } label: {
                Label("共有", systemImage: "square.and.arrow.up")
            }
            .accessibilityLabel("共有")

            Spacer()

            Menu {
                Button(role: .destructive, action: onDelete) {
                    Label("削除", systemImage: "trash")
                }
            } label: {
                Image(systemName: "ellipsis")
                    .foregroundColor(.secondary)
            }
            .accessibilityLabel("その他のオプション")
        }
        .buttonStyle(.plain)
        .labelStyle(.titleAndIcon)
    }
}

// プレースホルダー
extension User {
    static let placeholder = User(
        id: "0",
        username: "unknown",
        displayName: "Unknown User",
        avatarURL: nil,
        bio: "",
        followerCount: 0,
        followingCount: 0,
        isFollowedByCurrentUser: false
    )
}
```

## 投稿作成画面

### CreatePostViewModel

```swift
@Observable
class CreatePostViewModel {
    var content = ""
    var selectedImages: [UIImage] = []
    var isPosting = false
    var error: Error?

    private let postUseCase: PostUseCaseProtocol
    private let maxImages = 4
    private let maxContentLength = 500

    init(postUseCase: PostUseCaseProtocol) {
        self.postUseCase = postUseCase
    }

    var canPost: Bool {
        (!content.isEmpty || !selectedImages.isEmpty) && !isPosting
    }

    var remainingCharacters: Int {
        maxContentLength - content.count
    }

    var isContentTooLong: Bool {
        content.count > maxContentLength
    }

    var canAddMoreImages: Bool {
        selectedImages.count < maxImages
    }

    @MainActor
    func addImage(_ image: UIImage) {
        guard canAddMoreImages else { return }
        selectedImages.append(image)
    }

    @MainActor
    func removeImage(at index: Int) {
        selectedImages.remove(at: index)
    }

    @MainActor
    func post() async throws {
        guard canPost else { return }

        isPosting = true
        error = nil

        defer { isPosting = false }

        // 画像をData形式に変換
        let imageDataArray = selectedImages.compactMap { image in
            image.jpegData(compressionQuality: 0.8)
        }

        do {
            _ = try await postUseCase.createPost(content: content, images: imageDataArray)
        } catch {
            self.error = error
            throw error
        }
    }

    @MainActor
    func reset() {
        content = ""
        selectedImages = []
        error = nil
    }
}
```

### CreatePostView

```swift
struct CreatePostView: View {
    @State private var viewModel: CreatePostViewModel
    @Environment(\.dismiss) var dismiss
    @State private var showImagePicker = false

    init(postUseCase: PostUseCaseProtocol = PostUseCase(repository: PostRepository())) {
        _viewModel = State(initialValue: CreatePostViewModel(postUseCase: postUseCase))
    }

    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(alignment: .leading, spacing: 16) {
                    // テキスト入力
                    TextEditor(text: $viewModel.content)
                        .frame(minHeight: 150)
                        .overlay(alignment: .topLeading) {
                            if viewModel.content.isEmpty {
                                Text("いまどうしてる?")
                                    .foregroundColor(.secondary)
                                    .padding(.top, 8)
                                    .padding(.leading, 4)
                                    .allowsHitTesting(false)
                            }
                        }
                        .accessibilityLabel("投稿内容")
                        .accessibilityValue(viewModel.content.isEmpty ? "未入力" : viewModel.content)

                    // 文字数カウンター
                    HStack {
                        Spacer()
                        Text("\(viewModel.remainingCharacters)")
                            .font(.caption)
                            .foregroundColor(viewModel.isContentTooLong ? .red : .secondary)
                    }

                    // 画像プレビュー
                    if !viewModel.selectedImages.isEmpty {
                        ImagePreviewGrid(
                            images: viewModel.selectedImages,
                            onRemove: { index in
                                viewModel.removeImage(at: index)
                            }
                        )
                    }

                    // 画像追加ボタン
                    Button(action: { showImagePicker = true }) {
                        Label("画像を追加", systemImage: "photo")
                    }
                    .disabled(!viewModel.canAddMoreImages)
                    .accessibilityLabel("画像を追加")
                    .accessibilityHint(
                        viewModel.canAddMoreImages
                            ? "タップして画像を選択"
                            : "最大\(4)枚まで"
                    )
                }
                .padding()
            }
            .navigationTitle("新規投稿")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") {
                        dismiss()
                    }
                }

                ToolbarItem(placement: .confirmationAction) {
                    Button("投稿") {
                        Task {
                            do {
                                try await viewModel.post()
                                dismiss()
                            } catch {
                                // エラーはアラートで表示
                            }
                        }
                    }
                    .disabled(!viewModel.canPost || viewModel.isContentTooLong)
                    .accessibilityLabel("投稿")
                }
            }
            .sheet(isPresented: $showImagePicker) {
                ImagePicker(onImageSelected: { image in
                    viewModel.addImage(image)
                })
            }
            .overlay {
                if viewModel.isPosting {
                    Color.black.opacity(0.3)
                        .ignoresSafeArea()
                        .overlay {
                            ProgressView("投稿中...")
                                .padding()
                                .background(.regularMaterial)
                                .cornerRadius(12)
                        }
                }
            }
            .alert(error: $viewModel.error)
        }
    }
}

struct ImagePreviewGrid: View {
    let images: [UIImage]
    let onRemove: (Int) -> Void

    var body: some View {
        LazyVGrid(columns: [GridItem(.adaptive(minimum: 150))], spacing: 8) {
            ForEach(Array(images.enumerated()), id: \.offset) { index, image in
                ZStack(alignment: .topTrailing) {
                    Image(uiImage: image)
                        .resizable()
                        .aspectRatio(contentMode: .fill)
                        .frame(height: 150)
                        .clipShape(RoundedRectangle(cornerRadius: 8))

                    Button(action: { onRemove(index) }) {
                        Image(systemName: "xmark.circle.fill")
                            .font(.title2)
                            .foregroundColor(.white)
                            .background(Circle().fill(Color.black.opacity(0.5)))
                    }
                    .padding(4)
                    .accessibilityLabel("画像を削除")
                }
            }
        }
    }
}

// 簡易的なImagePicker(実際はPHPickerViewControllerを使用)
struct ImagePicker: View {
    let onImageSelected: (UIImage) -> Void
    @Environment(\.dismiss) var dismiss

    var body: some View {
        Text("Image Picker")
            .onTapGesture {
                // ダミー画像
                if let image = UIImage(systemName: "photo") {
                    onImageSelected(image)
                }
                dismiss()
            }
    }
}
```

## まとめ

### 実装した機能

1. **タイムライン**:
   - 無限スクロール
   - 楽観的更新
   - Pull-to-refresh

2. **投稿作成**:
   - テキスト + 画像(最大4枚)
   - リアルタイムバリデーション
   - プログレス表示

3. **アーキテクチャ**:
   - Clean Architecture
   - MVVM + UseCases
   - Repository Pattern

4. **パフォーマンス**:
   - LazyVStack使用
   - AsyncImage最適化
   - @Observable活用

### 次のステップ

次章「実戦ケーススタディ Part 2」では、このアプリのパフォーマンス最適化、キャッシング戦略、オフライン対応を実装します。

---

Generated with [Claude Code](https://claude.com/claude-code)
