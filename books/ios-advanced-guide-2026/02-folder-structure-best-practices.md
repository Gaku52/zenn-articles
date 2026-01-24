---
title: "フォルダ構成のベストプラクティス"
---

# フォルダ構成のベストプラクティス

適切なフォルダ構成は、プロジェクトのスケーラビリティと保守性を大きく左右します。本章では、iOS開発で推奨されるフォルダ構成とその実装方法を解説します。

## フォルダ構成の重要性

プロジェクトの成長に伴い、ファイル数は増加します。適切なフォルダ構成により、以下のメリットが期待されます：

- ファイルの検索性向上
- 機能の責任範囲の明確化
- チーム開発での作業分担の容易化
- 新メンバーのオンボーディング効率化

## アーキテクチャによる分類

### MVVM構成

MVVMアーキテクチャを採用する場合の推奨フォルダ構成です。

```plaintext
MyAwesomeApp/
├── App/
│   ├── MyAwesomeAppApp.swift
│   └── AppDelegate.swift
├── Core/
│   ├── Extensions/
│   │   ├── String+Extensions.swift
│   │   ├── Date+Extensions.swift
│   │   └── View+Extensions.swift
│   ├── Utilities/
│   │   ├── Logger.swift
│   │   ├── DateFormatter.swift
│   │   └── Constants.swift
│   └── Networking/
│       ├── NetworkManager.swift
│       ├── Endpoint.swift
│       └── NetworkError.swift
├── Models/
│   ├── User.swift
│   ├── Post.swift
│   └── Comment.swift
├── Views/
│   ├── Home/
│   │   ├── HomeView.swift
│   │   ├── HomeViewModel.swift
│   │   └── Components/
│   │       ├── PostCard.swift
│   │       └── UserAvatar.swift
│   ├── Profile/
│   │   ├── ProfileView.swift
│   │   ├── ProfileViewModel.swift
│   │   └── Components/
│   │       └── ProfileHeader.swift
│   └── Common/
│       ├── LoadingView.swift
│       └── ErrorView.swift
├── Services/
│   ├── UserService.swift
│   ├── PostService.swift
│   └── AuthService.swift
├── Repositories/
│   ├── UserRepository.swift
│   └── PostRepository.swift
└── Resources/
    ├── Assets.xcassets/
    ├── Localizable.strings
    └── Info.plist
```

### Clean Architecture構成

より大規模なプロジェクトでは、Clean Architectureの採用が推奨されます。

```plaintext
MyAwesomeApp/
├── App/
│   └── MyAwesomeAppApp.swift
├── Domain/
│   ├── Entities/
│   │   ├── User.swift
│   │   └── Post.swift
│   ├── UseCases/
│   │   ├── GetUserUseCase.swift
│   │   ├── CreatePostUseCase.swift
│   │   └── UpdateProfileUseCase.swift
│   └── RepositoryInterfaces/
│       ├── UserRepositoryInterface.swift
│       └── PostRepositoryInterface.swift
├── Data/
│   ├── Repositories/
│   │   ├── UserRepositoryImpl.swift
│   │   └── PostRepositoryImpl.swift
│   ├── Network/
│   │   ├── API/
│   │   │   ├── UserAPI.swift
│   │   │   └── PostAPI.swift
│   │   └── DTOs/
│   │       ├── UserDTO.swift
│   │       └── PostDTO.swift
│   └── Local/
│       ├── CoreData/
│       │   └── CoreDataManager.swift
│       └── UserDefaults/
│           └── UserDefaultsManager.swift
├── Presentation/
│   ├── Home/
│   │   ├── HomeView.swift
│   │   ├── HomeViewModel.swift
│   │   └── Components/
│   ├── Profile/
│   │   ├── ProfileView.swift
│   │   └── ProfileViewModel.swift
│   └── Common/
│       └── Components/
└── Resources/
    └── Assets.xcassets/
```

## 機能単位での分類（Feature-based）

機能単位でフォルダを分けることで、各機能の独立性が高まります。

```plaintext
MyAwesomeApp/
├── Features/
│   ├── Authentication/
│   │   ├── Models/
│   │   │   └── AuthUser.swift
│   │   ├── Views/
│   │   │   ├── LoginView.swift
│   │   │   └── SignUpView.swift
│   │   ├── ViewModels/
│   │   │   ├── LoginViewModel.swift
│   │   │   └── SignUpViewModel.swift
│   │   └── Services/
│   │       └── AuthService.swift
│   ├── Home/
│   │   ├── Models/
│   │   ├── Views/
│   │   ├── ViewModels/
│   │   └── Services/
│   └── Profile/
│       ├── Models/
│       ├── Views/
│       ├── ViewModels/
│       └── Services/
├── Core/
│   ├── Networking/
│   ├── Extensions/
│   └── Utilities/
└── Resources/
```

## Xcodeでのフォルダ作成

### 物理フォルダとの同期

Xcodeのグループ構造と実際のファイルシステムを一致させることが推奨されます。

**手順:**

1. Finderでプロジェクトフォルダを開く
2. 必要なフォルダ構造を作成
3. Xcodeで「Add Files to Project」から追加
4. 「Create folder references」を選択

### New Groupの使用

プロジェクト内で右クリックし、「New Group」を選択してグループを作成します。

```plaintext
右クリック > New Group
または
⌘ + Option + N
```

## ファイル命名規則

一貫性のある命名規則により、ファイルの役割が明確になります。

### Viewファイル

```plaintext
機能名 + View.swift
例: HomeView.swift, ProfileView.swift
```

### ViewModelファイル

```plaintext
機能名 + ViewModel.swift
例: HomeViewModel.swift, ProfileViewModel.swift
```

### Modelファイル

```plaintext
エンティティ名.swift
例: User.swift, Post.swift
```

### Serviceファイル

```plaintext
ドメイン名 + Service.swift
例: UserService.swift, AuthService.swift
```

### Repositoryファイル

```plaintext
エンティティ名 + Repository.swift
例: UserRepository.swift, PostRepository.swift
```

## 共通コンポーネントの管理

再利用可能なコンポーネントは、Common/Sharedフォルダで管理します。

```plaintext
Common/
├── Components/
│   ├── Buttons/
│   │   ├── PrimaryButton.swift
│   │   └── SecondaryButton.swift
│   ├── TextFields/
│   │   ├── EmailTextField.swift
│   │   └── PasswordTextField.swift
│   └── Cards/
│       └── InfoCard.swift
├── Modifiers/
│   ├── CardModifier.swift
│   └── ShadowModifier.swift
└── Styles/
    ├── ButtonStyles.swift
    └── TextFieldStyles.swift
```

### 共通ボタンコンポーネント例

```swift
// Common/Components/Buttons/PrimaryButton.swift
import SwiftUI

struct PrimaryButton: View {
    let title: String
    let action: () -> Void

    var body: some View {
        Button(action: action) {
            Text(title)
                .font(.headline)
                .foregroundColor(.white)
                .frame(maxWidth: .infinity)
                .padding()
                .background(Color.blue)
                .cornerRadius(10)
        }
    }
}

#Preview {
    PrimaryButton(title: "ログイン") {
        print("Button tapped")
    }
    .padding()
}
```

## Extensions の管理

Extensionsは型ごとにファイルを分けて管理します。

```plaintext
Core/Extensions/
├── Foundation/
│   ├── String+Extensions.swift
│   ├── Date+Extensions.swift
│   └── Array+Extensions.swift
├── SwiftUI/
│   ├── View+Extensions.swift
│   ├── Color+Extensions.swift
│   └── Font+Extensions.swift
└── UIKit/
    ├── UIColor+Extensions.swift
    └── UIImage+Extensions.swift
```

### Extension実装例

```swift
// Core/Extensions/Foundation/String+Extensions.swift
import Foundation

extension String {
    var isValidEmail: Bool {
        let emailRegex = "[A-Z0-9a-z._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,64}"
        let emailPredicate = NSPredicate(format: "SELF MATCHES %@", emailRegex)
        return emailPredicate.evaluate(with: self)
    }

    func trimmed() -> String {
        self.trimmingCharacters(in: .whitespacesAndNewlines)
    }
}
```

## Resources の管理

アプリのリソースファイルを適切に分類します。

```plaintext
Resources/
├── Assets.xcassets/
│   ├── AppIcon.appiconset/
│   ├── Colors/
│   │   ├── Primary.colorset/
│   │   └── Secondary.colorset/
│   ├── Images/
│   │   ├── logo.imageset/
│   │   └── placeholder.imageset/
│   └── Icons/
│       ├── home.imageset/
│       └── profile.imageset/
├── Fonts/
│   ├── CustomFont-Regular.ttf
│   └── CustomFont-Bold.ttf
├── Localizable/
│   ├── en.lproj/
│   │   └── Localizable.strings
│   └── ja.lproj/
│       └── Localizable.strings
└── Info.plist
```

## Xcodeグループの整理

プロジェクトナビゲーターでの見やすさを向上させるための Tips です。

### アルファベット順の維持

ファイルとグループは常にアルファベット順に並べることで、検索性が向上します。

### 階層の深さを3階層まで

あまりに深い階層は避け、3階層程度に抑えることが推奨されます。

```plaintext
良い例:
Features/Home/Views/HomeView.swift

避けるべき例:
Features/Home/Presentation/UI/Views/Screens/HomeView.swift
```

## フォルダ構成のメンテナンス

プロジェクトの成長に伴い、定期的にフォルダ構成を見直します。

### リファクタリングのタイミング

- 特定のフォルダにファイルが集中した時
- 新機能追加時に適切な配置場所がない時
- チームメンバーからファイルが見つけにくいという意見が出た時

### 段階的な移行

既存プロジェクトのフォルダ構成を変更する際は、一度に全て変更せず、段階的に移行することが推奨されます。

```plaintext
1. 新しいフォルダ構成を定義
2. 新規ファイルは新構成に従う
3. 既存ファイルは機能変更時に移動
4. 最終的に全ファイルを新構成に統一
```

## テストファイルの配置

テストファイルは本体と同じ構造を維持します。

```plaintext
MyAwesomeAppTests/
├── Features/
│   ├── Authentication/
│   │   ├── ViewModels/
│   │   │   └── LoginViewModelTests.swift
│   │   └── Services/
│   │       └── AuthServiceTests.swift
│   └── Home/
│       └── ViewModels/
│           └── HomeViewModelTests.swift
└── Core/
    ├── Networking/
    │   └── NetworkManagerTests.swift
    └── Extensions/
        └── StringExtensionsTests.swift
```

## まとめ

本章では、iOS開発におけるフォルダ構成のベストプラクティスを解説しました。適切なフォルダ構成により、以下の効果が期待されます：

- コードの可読性向上
- ファイル検索の効率化
- チーム開発の円滑化
- プロジェクトの拡張性確保

プロジェクトの規模や特性に応じて、最適なフォルダ構成を選択しましょう。次章では、ビルド設定の詳細について解説します。
