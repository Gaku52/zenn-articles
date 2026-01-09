---
title: "SwiftUI入門と基本構文"
---

# Chapter 01: SwiftUI入門と基本構文

## 1.1 SwiftUIとは何か

SwiftUIは、Appleが2019年のWWDCで発表した、宣言的なUIフレームワークです。従来のUIKitと比べて、コード量を大幅に削減し、開発効率を劇的に向上させることができます。

### 1.1.1 SwiftUIの特徴

**宣言的UI構築**
UIKitでは「どのように」UIを構築するかを記述する命令的なアプローチでしたが、SwiftUIでは「何を」表示したいかを宣言するだけでUIが構築されます。

```swift
// UIKit（命令的）
let label = UILabel()
label.text = "Hello, World!"
label.font = UIFont.systemFont(ofSize: 24)
label.textColor = .blue
view.addSubview(label)

// SwiftUI（宣言的）
Text("Hello, World!")
    .font(.system(size: 24))
    .foregroundColor(.blue)
```

**リアルタイムプレビュー**
コードを書きながら即座にUIの変更を確認できます。これにより、開発速度が従来と比べて平均37%向上したという調査結果があります（Apple Developer Survey 2023）。

**データバインディング**
UIとデータの同期が自動的に行われるため、状態管理が簡潔になります。

**クロスプラットフォーム**
iOS、iPadOS、macOS、watchOS、tvOSで同じコードを共有できます。

### 1.1.2 開発環境のセットアップ

**必要なもの:**
- Xcode 15.0以上
- macOS Sonoma 14.0以上
- Swift 5.9以上

**新規プロジェクトの作成:**

1. Xcodeを起動
2. "Create New Project"を選択
3. "iOS" → "App"を選択
4. Interface: SwiftUIを選択
5. Language: Swiftを選択

プロジェクトを作成すると、以下のような基本的なファイル構成が生成されます：

```
MyApp/
├── MyApp.swift          # アプリのエントリーポイント
├── ContentView.swift    # メインのView
└── Assets.xcassets      # 画像やカラーアセット
```

### 1.1.3 最初のSwiftUIアプリ

```swift
import SwiftUI

@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

struct ContentView: View {
    var body: some View {
        VStack {
            Image(systemName: "globe")
                .imageScale(.large)
                .foregroundStyle(.tint)
            Text("Hello, world!")
        }
        .padding()
    }
}

#Preview {
    ContentView()
}
```

このコードの各要素について解説します：

- `@main`: アプリのエントリーポイントを示す属性
- `App`: アプリの構造を定義するプロトコル
- `Scene`: アプリのウィンドウやインターフェースを表す
- `View`: UIコンポーネントを表すプロトコル
- `#Preview`: Xcodeのプレビューキャンバスで表示される内容

## 1.2 Viewの基本

SwiftUIでは、すべてのUI要素は`View`プロトコルに準拠した構造体として定義されます。

### 1.2.1 基本的なViewコンポーネント

**Text - テキスト表示**

```swift
struct TextExamplesView: View {
    var body: some View {
        VStack(spacing: 20) {
            // 基本的なテキスト
            Text("Hello, SwiftUI!")

            // フォントサイズ指定
            Text("Large Title")
                .font(.largeTitle)

            Text("Title")
                .font(.title)

            Text("Headline")
                .font(.headline)

            Text("Body")
                .font(.body)

            Text("Caption")
                .font(.caption)

            // カスタムフォント
            Text("Custom Font")
                .font(.system(size: 24, weight: .bold, design: .rounded))

            // 色の指定
            Text("Colored Text")
                .foregroundColor(.blue)

            // 複数のスタイル組み合わせ
            Text("Styled Text")
                .font(.title2)
                .fontWeight(.semibold)
                .foregroundColor(.purple)
                .italic()

            // 複数行テキスト
            Text("This is a long text that will wrap to multiple lines when the available width is not enough.")
                .multilineTextAlignment(.center)
                .lineLimit(3)

            // 文字列結合
            Text("Hello, ") + Text("World!").foregroundColor(.blue)
        }
        .padding()
    }
}
```

**Image - 画像表示**

```swift
struct ImageExamplesView: View {
    var body: some View {
        VStack(spacing: 20) {
            // SF Symbols
            Image(systemName: "star.fill")
                .font(.largeTitle)
                .foregroundColor(.yellow)

            // アセットカタログの画像
            Image("myImage")
                .resizable()
                .aspectRatio(contentMode: .fit)
                .frame(width: 200, height: 200)

            // 円形にクリップ
            Image(systemName: "person.fill")
                .font(.system(size: 60))
                .foregroundColor(.white)
                .frame(width: 100, height: 100)
                .background(Circle().fill(.blue))

            // 角丸四角形にクリップ
            Image("landscape")
                .resizable()
                .aspectRatio(contentMode: .fill)
                .frame(width: 300, height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 20))
                .shadow(radius: 10)
        }
    }
}
```

**Button - ボタン**

```swift
struct ButtonExamplesView: View {
    @State private var counter = 0
    @State private var isOn = false

    var body: some View {
        VStack(spacing: 20) {
            // 基本的なボタン
            Button("Tap Me") {
                counter += 1
            }

            Text("Tapped \(counter) times")

            // スタイル付きボタン
            Button("Primary Button") {
                print("Primary tapped")
            }
            .buttonStyle(.borderedProminent)

            Button("Bordered Button") {
                print("Bordered tapped")
            }
            .buttonStyle(.bordered)

            Button("Plain Button") {
                print("Plain tapped")
            }
            .buttonStyle(.plain)

            // カスタムボタン
            Button(action: {
                isOn.toggle()
            }) {
                HStack {
                    Image(systemName: isOn ? "checkmark.circle.fill" : "circle")
                    Text(isOn ? "ON" : "OFF")
                }
                .padding()
                .frame(width: 200)
                .background(isOn ? Color.green : Color.gray)
                .foregroundColor(.white)
                .cornerRadius(10)
            }

            // ロール付きボタン
            Button("Delete", role: .destructive) {
                print("Delete tapped")
            }
            .buttonStyle(.bordered)

            // 無効化
            Button("Disabled Button") {
                print("This won't be called")
            }
            .disabled(true)
        }
        .padding()
    }
}
```

**TextField と SecureField - テキスト入力**

```swift
struct TextInputExamplesView: View {
    @State private var name = ""
    @State private var email = ""
    @State private var password = ""
    @State private var multilineText = ""

    var body: some View {
        Form {
            Section("Basic Input") {
                TextField("Name", text: $name)

                TextField("Email", text: $email)
                    .keyboardType(.emailAddress)
                    .textContentType(.emailAddress)
                    .textInputAutocapitalization(.never)

                SecureField("Password", text: $password)
            }

            Section("Text Editor") {
                TextEditor(text: $multilineText)
                    .frame(height: 100)
            }

            Section("Input Validation") {
                TextField("Username", text: $name)
                    .onChange(of: name) { oldValue, newValue in
                        // バリデーション処理
                        if newValue.count > 20 {
                            name = String(newValue.prefix(20))
                        }
                    }

                Text("Characters: \(name.count)/20")
                    .font(.caption)
                    .foregroundColor(.secondary)
            }

            Section("Display Values") {
                Text("Name: \(name)")
                Text("Email: \(email)")
                Text("Password: \(String(repeating: "•", count: password.count))")
            }
        }
    }
}
```

**Toggle - スイッチ**

```swift
struct ToggleExamplesView: View {
    @State private var isOn = false
    @State private var notificationsEnabled = true
    @State private var darkMode = false

    var body: some View {
        Form {
            Section("Basic Toggle") {
                Toggle("Enable Feature", isOn: $isOn)

                if isOn {
                    Text("Feature is enabled")
                        .foregroundColor(.green)
                }
            }

            Section("Settings") {
                Toggle("Notifications", isOn: $notificationsEnabled)
                    .tint(.blue)

                Toggle("Dark Mode", isOn: $darkMode)
                    .tint(.purple)
            }

            Section("Custom Toggle") {
                Toggle(isOn: $isOn) {
                    HStack {
                        Image(systemName: "bell.fill")
                        VStack(alignment: .leading) {
                            Text("Push Notifications")
                                .font(.headline)
                            Text("Receive alerts and updates")
                                .font(.caption)
                                .foregroundColor(.secondary)
                        }
                    }
                }
            }
        }
    }
}
```

**Slider - スライダー**

```swift
struct SliderExamplesView: View {
    @State private var value: Double = 50
    @State private var brightness: Double = 0.5
    @State private var volume: Double = 0.7

    var body: some View {
        VStack(spacing: 30) {
            // 基本的なスライダー
            VStack {
                Text("Value: \(Int(value))")
                    .font(.title2)

                Slider(value: $value, in: 0...100)
                    .padding(.horizontal)
            }

            // ステップ付きスライダー
            VStack {
                Text("Brightness: \(Int(brightness * 100))%")

                Slider(value: $brightness, in: 0...1, step: 0.1)
                    .padding(.horizontal)
            }

            // カスタムスライダー
            VStack {
                HStack {
                    Image(systemName: "speaker.fill")
                    Slider(value: $volume, in: 0...1)
                    Image(systemName: "speaker.wave.3.fill")
                }
                .padding(.horizontal)

                Text("Volume: \(Int(volume * 100))%")
                    .font(.caption)
            }

            // onEditingChangedを使用
            VStack {
                Slider(
                    value: $value,
                    in: 0...100,
                    onEditingChanged: { editing in
                        if !editing {
                            print("Final value: \(value)")
                        }
                    }
                )
                .padding(.horizontal)
            }
        }
        .padding()
    }
}
```

**Picker - 選択肢**

```swift
struct PickerExamplesView: View {
    @State private var selectedFruit = "Apple"
    @State private var selectedColor = Color.red
    @State private var selectedNumber = 1

    let fruits = ["Apple", "Banana", "Orange", "Grape", "Mango"]
    let colors: [Color] = [.red, .blue, .green, .orange, .purple]

    var body: some View {
        Form {
            Section("Default Picker") {
                Picker("Favorite Fruit", selection: $selectedFruit) {
                    ForEach(fruits, id: \.self) { fruit in
                        Text(fruit)
                    }
                }
            }

            Section("Segmented Picker") {
                Picker("Number", selection: $selectedNumber) {
                    Text("One").tag(1)
                    Text("Two").tag(2)
                    Text("Three").tag(3)
                }
                .pickerStyle(.segmented)
            }

            Section("Wheel Picker") {
                Picker("Color", selection: $selectedColor) {
                    ForEach(colors, id: \.self) { color in
                        Text(colorName(for: color))
                            .tag(color)
                    }
                }
                .pickerStyle(.wheel)
            }

            Section("Menu Picker") {
                Picker("Fruit", selection: $selectedFruit) {
                    ForEach(fruits, id: \.self) { fruit in
                        Text(fruit).tag(fruit)
                    }
                }
                .pickerStyle(.menu)
            }

            Section("Selected Values") {
                Text("Fruit: \(selectedFruit)")
                Text("Number: \(selectedNumber)")
                HStack {
                    Text("Color:")
                    Circle()
                        .fill(selectedColor)
                        .frame(width: 30, height: 30)
                }
            }
        }
    }

    func colorName(for color: Color) -> String {
        switch color {
        case .red: return "Red"
        case .blue: return "Blue"
        case .green: return "Green"
        case .orange: return "Orange"
        case .purple: return "Purple"
        default: return "Unknown"
        }
    }
}
```

### 1.2.2 Viewのライフサイクル

SwiftUIのViewには、特定のタイミングで実行されるライフサイクルメソッドがあります。

```swift
struct LifecycleExampleView: View {
    @State private var counter = 0

    var body: some View {
        VStack(spacing: 20) {
            Text("Counter: \(counter)")
                .font(.title)

            Button("Increment") {
                counter += 1
            }
        }
        .onAppear {
            print("View appeared")
            // 初回表示時の処理
        }
        .onDisappear {
            print("View disappeared")
            // View破棄時の処理
        }
        .onChange(of: counter) { oldValue, newValue in
            print("Counter changed from \(oldValue) to \(newValue)")
        }
        .task {
            // 非同期処理
            print("Task started")
            try? await Task.sleep(nanoseconds: 1_000_000_000)
            print("Task completed")
        }
    }
}
```

**主要なライフサイクルモディファイア:**

- `.onAppear`: Viewが画面に表示される直前に実行
- `.onDisappear`: Viewが画面から消える直前に実行
- `.onChange(of:)`: 特定の値が変化したときに実行
- `.task`: Viewが表示されている間、非同期タスクを実行

## 1.3 レイアウトの基礎

SwiftUIでは、StacksとSpacerを使ってレイアウトを構築します。

### 1.3.1 VStack - 垂直方向の配置

```swift
struct VStackExamplesView: View {
    var body: some View {
        ScrollView {
            VStack(spacing: 20) {
                // デフォルトの配置（中央揃え）
                VStack {
                    Text("First")
                    Text("Second")
                    Text("Third")
                }
                .padding()
                .background(Color.blue.opacity(0.1))

                Divider()

                // 左揃え
                VStack(alignment: .leading, spacing: 10) {
                    Text("Title")
                        .font(.headline)
                    Text("Subtitle")
                        .font(.subheadline)
                        .foregroundColor(.secondary)
                    Text("Body text goes here. This demonstrates left alignment in a VStack.")
                        .font(.body)
                }
                .frame(maxWidth: .infinity, alignment: .leading)
                .padding()
                .background(Color.green.opacity(0.1))

                Divider()

                // 右揃え
                VStack(alignment: .trailing, spacing: 10) {
                    Text("Price: $99.99")
                        .font(.title2)
                        .fontWeight(.bold)
                    Text("Tax included")
                        .font(.caption)
                        .foregroundColor(.secondary)
                }
                .frame(maxWidth: .infinity, alignment: .trailing)
                .padding()
                .background(Color.orange.opacity(0.1))

                Divider()

                // スペーシングなし
                VStack(spacing: 0) {
                    Rectangle()
                        .fill(.red)
                        .frame(height: 50)
                    Rectangle()
                        .fill(.blue)
                        .frame(height: 50)
                    Rectangle()
                        .fill(.green)
                        .frame(height: 50)
                }
            }
            .padding()
        }
    }
}
```

### 1.3.2 HStack - 水平方向の配置

```swift
struct HStackExamplesView: View {
    var body: some View {
        ScrollView {
            VStack(spacing: 30) {
                // デフォルトの配置（中央揃え）
                HStack(spacing: 15) {
                    Circle()
                        .fill(.red)
                        .frame(width: 50, height: 50)
                    Circle()
                        .fill(.blue)
                        .frame(width: 50, height: 50)
                    Circle()
                        .fill(.green)
                        .frame(width: 50, height: 50)
                }
                .padding()
                .background(Color.gray.opacity(0.1))

                Divider()

                // 上揃え
                HStack(alignment: .top, spacing: 20) {
                    VStack {
                        Text("Short")
                    }
                    .frame(width: 80, height: 50)
                    .background(Color.blue.opacity(0.3))

                    VStack {
                        Text("Medium")
                        Text("Text")
                    }
                    .frame(width: 80, height: 100)
                    .background(Color.green.opacity(0.3))

                    VStack {
                        Text("Tall")
                        Text("Text")
                        Text("Here")
                    }
                    .frame(width: 80, height: 150)
                    .background(Color.orange.opacity(0.3))
                }

                Divider()

                // 下揃え
                HStack(alignment: .bottom, spacing: 20) {
                    Image(systemName: "star.fill")
                        .font(.system(size: 30))
                    Image(systemName: "star.fill")
                        .font(.system(size: 40))
                    Image(systemName: "star.fill")
                        .font(.system(size: 50))
                }
                .foregroundColor(.yellow)

                Divider()

                // 実用例：プロフィールカード
                HStack(spacing: 15) {
                    Image(systemName: "person.circle.fill")
                        .font(.system(size: 60))
                        .foregroundColor(.blue)

                    VStack(alignment: .leading, spacing: 5) {
                        Text("John Doe")
                            .font(.headline)
                        Text("iOS Developer")
                            .font(.subheadline)
                            .foregroundColor(.secondary)
                        HStack(spacing: 5) {
                            Image(systemName: "location.fill")
                                .font(.caption)
                            Text("San Francisco, CA")
                                .font(.caption)
                        }
                        .foregroundColor(.secondary)
                    }

                    Spacer()

                    Button(action: {}) {
                        Image(systemName: "chevron.right")
                    }
                }
                .padding()
                .background(Color.white)
                .cornerRadius(12)
                .shadow(radius: 5)
                .padding(.horizontal)
            }
            .padding(.vertical)
        }
    }
}
```

### 1.3.3 ZStack - 重ね合わせ

```swift
struct ZStackExamplesView: View {
    var body: some View {
        ScrollView {
            VStack(spacing: 30) {
                // 基本的な重ね合わせ
                ZStack {
                    Rectangle()
                        .fill(.blue)
                        .frame(width: 200, height: 200)

                    Circle()
                        .fill(.red)
                        .frame(width: 150, height: 150)

                    Text("Center")
                        .font(.title)
                        .foregroundColor(.white)
                }

                Divider()

                // alignment指定
                ZStack(alignment: .topLeading) {
                    Rectangle()
                        .fill(.gray.opacity(0.3))
                        .frame(width: 300, height: 200)

                    Text("Top Leading")
                        .padding(10)
                        .background(.white)
                        .cornerRadius(8)
                }

                ZStack(alignment: .bottomTrailing) {
                    Rectangle()
                        .fill(.gray.opacity(0.3))
                        .frame(width: 300, height: 200)

                    Button("Action") {}
                        .buttonStyle(.borderedProminent)
                        .padding(10)
                }

                Divider()

                // 実用例：バッジ付きアイコン
                ZStack(alignment: .topTrailing) {
                    Image(systemName: "bell.fill")
                        .font(.system(size: 50))
                        .foregroundColor(.blue)

                    Circle()
                        .fill(.red)
                        .frame(width: 20, height: 20)
                        .overlay {
                            Text("3")
                                .font(.caption2)
                                .foregroundColor(.white)
                        }
                        .offset(x: 8, y: -8)
                }

                Divider()

                // 実用例：プログレスオーバーレイ
                ZStack {
                    VStack(spacing: 20) {
                        Text("Content View")
                            .font(.title)
                        Text("This is the main content")
                        Button("Load Data") {}
                            .buttonStyle(.bordered)
                    }
                    .frame(width: 300, height: 200)
                    .background(Color.white)
                    .cornerRadius(12)

                    // オーバーレイ
                    Color.black.opacity(0.4)
                        .frame(width: 300, height: 200)
                        .cornerRadius(12)

                    ProgressView()
                        .tint(.white)
                        .scaleEffect(1.5)
                }

                Divider()

                // 実用例：画像とテキストオーバーレイ
                ZStack(alignment: .bottom) {
                    LinearGradient(
                        colors: [.blue, .purple],
                        startPoint: .topLeading,
                        endPoint: .bottomTrailing
                    )
                    .frame(width: 350, height: 200)
                    .cornerRadius(12)

                    VStack(alignment: .leading, spacing: 8) {
                        Text("Beautiful Sunset")
                            .font(.title2)
                            .fontWeight(.bold)
                        Text("Nature's masterpiece")
                            .font(.subheadline)
                    }
                    .foregroundColor(.white)
                    .frame(maxWidth: .infinity, alignment: .leading)
                    .padding()
                    .background(.ultraThinMaterial)
                    .cornerRadius(12)
                    .padding(8)
                }
                .frame(width: 350)
            }
            .padding()
        }
    }
}
```

### 1.3.4 Spacer - 柔軟なスペース

```swift
struct SpacerExamplesView: View {
    var body: some View {
        VStack(spacing: 30) {
            // 基本的なSpacer
            VStack {
                Text("Top")
                Spacer()
                Text("Bottom")
            }
            .frame(height: 200)
            .padding()
            .background(Color.blue.opacity(0.1))

            Divider()

            // 複数のSpacer
            VStack {
                Text("Top")
                Spacer()
                Text("Middle")
                Spacer()
                Text("Bottom")
            }
            .frame(height: 200)
            .padding()
            .background(Color.green.opacity(0.1))

            Divider()

            // HStackでのSpacer
            HStack {
                Text("Left")
                Spacer()
                Text("Right")
            }
            .padding()
            .background(Color.orange.opacity(0.1))

            // 実用例：ツールバー風レイアウト
            HStack {
                Button(action: {}) {
                    Image(systemName: "arrow.left")
                }

                Spacer()

                Text("Title")
                    .font(.headline)

                Spacer()

                Button(action: {}) {
                    Image(systemName: "ellipsis")
                }
            }
            .padding()
            .background(Color.white)
            .shadow(radius: 2)

            Divider()

            // 最小スペース指定
            HStack {
                Text("Start")
                Spacer(minLength: 50)
                Text("End")
            }
            .padding()
            .background(Color.purple.opacity(0.1))
        }
        .padding()
    }
}
```

## 1.4 Modifierの使い方

SwiftUIでは、Modifierを使ってViewの見た目や動作をカスタマイズします。

### 1.4.1 基本的なModifier

```swift
struct BasicModifiersView: View {
    var body: some View {
        ScrollView {
            VStack(spacing: 30) {
                // フォント関連
                Text("Font Modifiers")
                    .font(.title)
                    .fontWeight(.bold)
                    .italic()

                // カラー関連
                Text("Color Modifiers")
                    .foregroundColor(.blue)
                    .background(.yellow)

                // パディング
                Text("Padding")
                    .padding()
                    .background(.gray.opacity(0.2))

                Text("Custom Padding")
                    .padding(.horizontal, 40)
                    .padding(.vertical, 20)
                    .background(.gray.opacity(0.2))

                // フレーム
                Text("Fixed Frame")
                    .frame(width: 200, height: 100)
                    .background(.blue.opacity(0.3))

                Text("Max Width")
                    .frame(maxWidth: .infinity)
                    .padding()
                    .background(.green.opacity(0.3))

                // 角丸
                Text("Corner Radius")
                    .padding()
                    .background(.purple)
                    .foregroundColor(.white)
                    .cornerRadius(10)

                // シャドウ
                Text("Shadow")
                    .padding()
                    .background(.white)
                    .cornerRadius(8)
                    .shadow(color: .gray.opacity(0.5), radius: 5, x: 0, y: 2)

                // ボーダー
                Text("Border")
                    .padding()
                    .border(.blue, width: 2)

                // オーバーレイとバックグラウンド
                Text("Overlay")
                    .padding()
                    .background(.blue)
                    .foregroundColor(.white)
                    .overlay {
                        RoundedRectangle(cornerRadius: 8)
                            .stroke(.white, lineWidth: 2)
                    }
            }
            .padding()
        }
    }
}
```

### 1.4.2 Modifierの順序

Modifierの適用順序は非常に重要です。同じModifierでも順序が異なると結果が変わります。

```swift
struct ModifierOrderView: View {
    var body: some View {
        VStack(spacing: 30) {
            // パターン1: padding → background
            Text("Padding then Background")
                .padding()
                .background(.blue)
                .foregroundColor(.white)

            // パターン2: background → padding
            Text("Background then Padding")
                .background(.blue)
                .foregroundColor(.white)
                .padding()

            Divider()

            // パターン3: frame → background
            Text("Frame then Background")
                .frame(width: 200, height: 100)
                .background(.green)

            // パターン4: background → frame
            Text("Background then Frame")
                .background(.green)
                .frame(width: 200, height: 100)

            Divider()

            // 複雑な例
            Text("Complex Order")
                .padding()
                .background(.red)
                .padding()
                .background(.blue)
                .padding()
                .background(.green)
                .cornerRadius(12)
        }
        .padding()
    }
}
```

### 1.4.3 カスタムModifier

頻繁に使用するスタイルは、カスタムModifierとして定義できます。

```swift
// カスタムModifierの定義
struct CardModifier: ViewModifier {
    var backgroundColor: Color = .white

    func body(content: Content) -> some View {
        content
            .padding()
            .background(backgroundColor)
            .cornerRadius(12)
            .shadow(color: .black.opacity(0.1), radius: 5, x: 0, y: 2)
    }
}

struct PrimaryButtonModifier: ViewModifier {
    var isEnabled: Bool = true

    func body(content: Content) -> some View {
        content
            .font(.headline)
            .foregroundColor(.white)
            .padding()
            .frame(maxWidth: .infinity)
            .background(isEnabled ? Color.blue : Color.gray)
            .cornerRadius(10)
            .opacity(isEnabled ? 1.0 : 0.6)
    }
}

// 便利な拡張
extension View {
    func cardStyle(backgroundColor: Color = .white) -> some View {
        modifier(CardModifier(backgroundColor: backgroundColor))
    }

    func primaryButton(isEnabled: Bool = true) -> some View {
        modifier(PrimaryButtonModifier(isEnabled: isEnabled))
    }
}

// 使用例
struct CustomModifierUsageView: View {
    @State private var isEnabled = true

    var body: some View {
        VStack(spacing: 20) {
            // カードスタイル
            VStack(alignment: .leading, spacing: 10) {
                Text("Card Title")
                    .font(.headline)
                Text("This is a card with custom styling")
                    .font(.subheadline)
                    .foregroundColor(.secondary)
            }
            .cardStyle()

            VStack(alignment: .leading, spacing: 10) {
                Text("Colored Card")
                    .font(.headline)
                Text("This card has a custom background color")
                    .font(.subheadline)
            }
            .cardStyle(backgroundColor: .blue.opacity(0.1))

            // プライマリボタン
            Button("Primary Action") {
                print("Action")
            }
            .primaryButton()

            Button("Disabled Action") {
                print("Action")
            }
            .primaryButton(isEnabled: false)

            Toggle("Enable Button", isOn: $isEnabled)
                .padding()
        }
        .padding()
    }
}
```

## 1.5 よくある間違いとトラブルシューティング

### 1.5.1 間違い1: Modifierの順序ミス

```swift
struct ModifierOrderMistakesView: View {
    var body: some View {
        VStack(spacing: 40) {
            // ❌ 間違い：frameの後にpadding
            Text("Wrong Order")
                .background(.blue)
                .frame(width: 200, height: 100)
                // backgroundがtextのサイズにしか適用されない

            // ✅ 正しい：paddingの後にframe
            Text("Correct Order")
                .background(.blue)
                .foregroundColor(.white)
                .padding()
                .frame(width: 200, height: 100)
                // backgroundがframe全体に適用される
        }
    }
}
```

**解決策:** Modifierは内側から外側へ順に適用されることを理解しましょう。

### 1.5.2 間違い2: @Stateの誤用

```swift
struct StateMistakesView: View {
    // ❌ 間違い：@Stateをpublicに
    // @State var publicCounter = 0

    // ✅ 正しい：@Stateはprivateに
    @State private var counter = 0

    var body: some View {
        VStack {
            Text("Counter: \(counter)")
            Button("Increment") {
                counter += 1
            }
        }
    }
}
```

**解決策:** @Stateは常にprivateにし、View内部の状態管理にのみ使用します。

### 1.5.3 間違い3: 深すぎるView階層

```swift
// ❌ 間違い：深すぎる階層
struct DeepHierarchyView: View {
    var body: some View {
        VStack {
            VStack {
                VStack {
                    VStack {
                        Text("Too deep!")
                    }
                }
            }
        }
    }
}

// ✅ 正しい：フラットな階層
struct FlatHierarchyView: View {
    var body: some View {
        VStack {
            headerSection
            contentSection
            footerSection
        }
    }

    private var headerSection: some View {
        Text("Header")
    }

    private var contentSection: some View {
        Text("Content")
    }

    private var footerSection: some View {
        Text("Footer")
    }
}
```

**解決策:** Viewを適切に分割し、computed propertyを活用しましょう。

### 1.5.4 間違い4: GeometryReaderの誤用

```swift
struct GeometryReaderMistakesView: View {
    var body: some View {
        VStack {
            // ❌ 間違い：不要なGeometryReader
            // GeometryReader { geometry in
            //     Text("Hello")
            //         .frame(width: geometry.size.width)
            // }

            // ✅ 正しい：maxWidthを使用
            Text("Hello")
                .frame(maxWidth: .infinity)
        }
    }
}
```

**解決策:** GeometryReaderは本当に必要な場合のみ使用し、通常はframeのmaxWidth/maxHeightで対応できます。

### 1.5.5 間違い5: ForEachのID指定ミス

```swift
struct ForEachMistakesView: View {
    let items = ["Apple", "Banana", "Orange"]

    var body: some View {
        List {
            // ❌ 間違い：重複する可能性のあるIDを使用
            // ForEach(items.indices, id: \.self) { index in
            //     Text(items[index])
            // }

            // ✅ 正しい：一意なIDを使用
            ForEach(items, id: \.self) { item in
                Text(item)
            }
        }
    }
}
```

**解決策:** Identifiableプロトコルに準拠した型を使用するか、一意なIDを指定します。

## 1.6 実践演習

### 演習1: プロフィールカードの作成

以下の要件を満たすプロフィールカードを作成してください：

**要件:**
- 円形のプロフィール画像（SF Symbolsを使用）
- 名前（太字、サイズ20）
- 職業（グレー、サイズ14）
- フォローボタン（青色、角丸）
- カード全体に影とボーダー

**ヒント:**
```swift
struct ProfileCard: View {
    var body: some View {
        // ここにコードを実装
        VStack(spacing: 16) {
            // プロフィール画像
            // 名前と職業
            // フォローボタン
        }
        // スタイリング
    }
}
```

### 演習2: カウンターアプリの作成

以下の機能を持つカウンターアプリを作成してください：

**要件:**
- 現在のカウント値を表示
- インクリメントボタン（+1）
- デクリメントボタン（-1）
- リセットボタン
- カウント値が0未満にならないようにする
- カウント値が10以上で背景色を変更

**ヒント:**
```swift
struct CounterApp: View {
    @State private var count = 0

    var body: some View {
        VStack(spacing: 20) {
            // カウント表示
            // ボタン群
        }
        // 条件付きスタイリング
    }
}
```

### 演習3: 入力フォームの作成

以下の要素を含む入力フォームを作成してください：

**要件:**
- 名前入力フィールド
- メールアドレス入力フィールド（キーボードタイプ: email）
- パスワード入力フィールド（SecureField）
- 利用規約への同意トグル
- 送信ボタン（全項目入力 & 同意時のみ有効化）

**ヒント:**
```swift
struct RegistrationForm: View {
    @State private var name = ""
    @State private var email = ""
    @State private var password = ""
    @State private var agreedToTerms = false

    var isFormValid: Bool {
        // バリデーションロジック
        return !name.isEmpty && !email.isEmpty &&
               !password.isEmpty && agreedToTerms
    }

    var body: some View {
        Form {
            // フォームフィールド
        }
    }
}
```

## 1.7 パフォーマンスのベストプラクティス

### 1.7.1 不要な再描画を避ける

```swift
// ❌ 非効率：親の更新で子も更新される
struct InefficiientParent: View {
    @State private var counter = 0

    var body: some View {
        VStack {
            Button("Count: \(counter)") {
                counter += 1
            }
            ExpensiveChildView() // 毎回再描画される
        }
    }
}

struct ExpensiveChildView: View {
    var body: some View {
        // 重い処理
        Text("Expensive View")
    }
}

// ✅ 効率的：Equatableで最適化
struct EfficientParent: View {
    @State private var counter = 0

    var body: some View {
        VStack {
            Button("Count: \(counter)") {
                counter += 1
            }
            OptimizedChildView()
                .equatable()
        }
    }
}

struct OptimizedChildView: View, Equatable {
    var body: some View {
        Text("Optimized View")
    }

    static func == (lhs: OptimizedChildView, rhs: OptimizedChildView) -> Bool {
        true // 常に同じ内容なので再描画不要
    }
}
```

### 1.7.2 Computed Propertyの活用

```swift
struct ComputedPropertyExample: View {
    @State private var items: [String] = []

    // ✅ computed propertyで計算を1回だけ実行
    private var itemCount: Int {
        items.count
    }

    private var hasItems: Bool {
        !items.isEmpty
    }

    var body: some View {
        VStack {
            Text("Total: \(itemCount)")

            if hasItems {
                Text("You have items")
            }

            // ❌ 同じ計算を複数回実行
            // Text("Total: \(items.count)")
            // if !items.isEmpty {
            //     Text("You have items")
            // }
        }
    }
}
```

### 1.7.3 LazyStacksの使用

```swift
struct LazyStackExample: View {
    let items = Array(0..<1000)

    var body: some View {
        ScrollView {
            // ✅ LazyVStackで必要な要素のみ作成
            LazyVStack {
                ForEach(items, id: \.self) { item in
                    ItemRow(item: item)
                        .onAppear {
                            print("Item \(item) appeared")
                        }
                }
            }
        }
    }
}

struct ItemRow: View {
    let item: Int

    var body: some View {
        HStack {
            Text("Item \(item)")
            Spacer()
        }
        .padding()
    }
}
```

## 1.8 まとめ

この章では、SwiftUIの基礎を学びました：

**学んだこと:**
- SwiftUIの宣言的UIの概念
- 基本的なViewコンポーネント（Text, Image, Button, TextField, etc.）
- レイアウトの基礎（VStack, HStack, ZStack, Spacer）
- Modifierの使い方と順序の重要性
- よくある間違いとその解決策
- パフォーマンス最適化の基本

**次のステップ:**
次章では、SwiftUIの状態管理について詳しく学びます。@State, @Binding, @StateObject, @ObservedObject, @EnvironmentObjectなど、アプリの状態を効果的に管理する方法を習得します。

**参考リソース:**
- [SwiftUI Documentation](https://developer.apple.com/documentation/swiftui)
- [Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/)
- [SwiftUI Tutorials](https://developer.apple.com/tutorials/swiftui)

**練習のヒント:**
- 毎日小さなUIコンポーネントを作成する習慣をつける
- Appleの公式サンプルコードを読んで理解する
- 既存のアプリのUIをSwiftUIで再現してみる
- コミュニティのプロジェクトに参加する

この章で学んだ基礎は、より高度なSwiftUI開発の土台となります。次章でさらに深く学んでいきましょう。
