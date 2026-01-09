---
title: "Chapter 24: 実戦ケーススタディ Part 2 - 高度な機能とデプロイ"
---

# Chapter 24: 実戦ケーススタディ Part 2 - 高度な機能とデプロイ

## 24.1 投稿機能の実装

### 24.1.1 ドメイン層

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

### 24.1.2 データ層

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

### 24.1.3 プレゼンテーション層

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

## 24.2 通知機能の実装

### 24.2.1 Push Notification設定

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

## 24.3 テスト実装

### 24.3.1 Unit Tests

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

### 24.3.2 UI Tests

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

## 24.4 CI/CD設定

### 24.4.1 GitHub Actions

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

### 24.4.2 Fastlane設定

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

## 24.5 パフォーマンス最適化

### 24.5.1 画像の最適化

```swift
// Core/Services/ImageOptimizationService.swift

import UIKit

final class ImageOptimizationService {
    static let shared = ImageOptimizationService()

    private init() {}

    func optimizeImage(_ image: UIImage, maxSizeInMB: Int = 2) -> Data? {
        let maxSize = maxSizeInMB * 1024 * 1024
        var compression: CGFloat = 1.0
        var imageData = image.jpegData(compressionQuality: compression)

        // サイズが上限を超える場合、圧縮率を調整
        while let data = imageData, data.count > maxSize, compression > 0.1 {
            compression -= 0.1
            imageData = image.jpegData(compressionQuality: compression)
        }

        return imageData
    }

    func resizeImage(_ image: UIImage, maxDimension: CGFloat = 1920) -> UIImage {
        let size = image.size
        let aspectRatio = size.width / size.height

        var newSize: CGSize
        if size.width > size.height {
            newSize = CGSize(width: maxDimension, height: maxDimension / aspectRatio)
        } else {
            newSize = CGSize(width: maxDimension * aspectRatio, height: maxDimension)
        }

        UIGraphicsBeginImageContextWithOptions(newSize, false, 1.0)
        image.draw(in: CGRect(origin: .zero, size: newSize))
        let resizedImage = UIGraphicsGetImageFromCurrentImageContext()
        UIGraphicsEndImageContext()

        return resizedImage ?? image
    }
}
```

### 24.5.2 キャッシング戦略

```swift
// Core/Services/CacheService.swift

import Foundation

final class CacheService {
    static let shared = CacheService()

    private let cache = NSCache<NSString, AnyObject>()
    private let fileManager = FileManager.default
    private lazy var cacheDirectory: URL = {
        fileManager.urls(for: .cachesDirectory, in: .userDomainMask)[0]
    }()

    private init() {
        cache.countLimit = 100
        cache.totalCostLimit = 50 * 1024 * 1024 // 50 MB
    }

    // MARK: - Memory Cache

    func setObject<T: AnyObject>(_ object: T, forKey key: String) {
        cache.setObject(object, forKey: key as NSString)
    }

    func object<T: AnyObject>(forKey key: String) -> T? {
        return cache.object(forKey: key as NSString) as? T
    }

    func removeObject(forKey key: String) {
        cache.removeObject(forKey: key as NSString)
    }

    func removeAllObjects() {
        cache.removeAllObjects()
    }

    // MARK: - Disk Cache

    func saveToDisk<T: Codable>(_ object: T, forKey key: String) throws {
        let fileURL = cacheDirectory.appendingPathComponent(key)
        let data = try JSONEncoder().encode(object)
        try data.write(to: fileURL)
    }

    func loadFromDisk<T: Codable>(forKey key: String) throws -> T? {
        let fileURL = cacheDirectory.appendingPathComponent(key)
        guard fileManager.fileExists(atPath: fileURL.path) else {
            return nil
        }
        let data = try Data(contentsOf: fileURL)
        return try JSONDecoder().decode(T.self, from: data)
    }

    func removeFromDisk(forKey key: String) throws {
        let fileURL = cacheDirectory.appendingPathComponent(key)
        try fileManager.removeItem(at: fileURL)
    }

    func clearDiskCache() throws {
        let contents = try fileManager.contentsOfDirectory(
            at: cacheDirectory,
            includingPropertiesForKeys: nil
        )
        for file in contents {
            try fileManager.removeItem(at: file)
        }
    }
}
```

## まとめ

Part 2では、SNSアプリ「SocialConnect」の高度な機能と運用基盤を完成させました。

**実装した主要機能:**

1. **投稿機能**
   - テキスト投稿
   - 画像アップロード（最大4枚）
   - ハッシュタグ抽出
   - Firebase Storageの活用

2. **通知機能**
   - Push Notification
   - Firebase Cloud Messaging
   - 通知ハンドリング
   - ディープリンク対応

3. **テスト**
   - Unit Tests
   - UI Tests
   - Mock実装
   - テストカバレッジ

4. **CI/CD**
   - GitHub Actions
   - Fastlane
   - 自動テスト実行
   - TestFlightデプロイ

5. **最適化**
   - 画像の最適化
   - キャッシング戦略
   - パフォーマンス改善

**アーキテクチャの総括:**

- **Clean Architecture**: Domain/Data/Presentationの明確な分離
- **MVVM**: ViewとViewModelの責務分離により、テストしやすい設計
- **依存性注入**: Protocolベースの抽象化によるテスタビリティ
- **リポジトリパターン**: データソースの抽象化とビジネスロジックの分離
- **ユースケース**: 再利用可能なビジネスロジックのカプセル化

**学んだベストプラクティス:**

1. Firebase統合の実践的なパターン
2. SwiftUIとMVVMの効果的な組み合わせ
3. 非同期処理とエラーハンドリング
4. テスト駆動開発とモック実装
5. CI/CDパイプラインの構築
6. パフォーマンス最適化の実践

このケーススタディを通じて、本書で学んだすべてのベストプラクティスを実践的に適用し、プロダクション品質のiOSアプリケーションを構築する方法を習得できました。

**次のステップ:**

- ユーザーフィードバックの収集と分析
- A/Bテストの実装
- 機能拡張（ストーリー機能、ライブ配信など）
- 国際化対応
- アクセシビリティの向上
- セキュリティ監査と改善
