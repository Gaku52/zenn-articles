---
title: "脱獄検知"
---

# 脱獄検知

脱獄されたデバイスでは、アプリのセキュリティが脅かされる可能性があります。本章では、脱獄検知の実装方法と対処方法を解説します。

## 脱獄のリスク

脱獄されたデバイスでは以下のリスクがあります：

```plaintext
セキュリティリスク:
- Keychainデータへのアクセス
- メモリ内のデータ読み取り
- SSLピンニングの回避
- アプリの改ざん
- デバッグツールの使用
```

## 脱獄検知の実装

```swift
// Core/Security/JailbreakDetector.swift
import UIKit

final class JailbreakDetector {
    static let shared = JailbreakDetector()

    private init() {}

    func isJailbroken() -> Bool {
        #if targetEnvironment(simulator)
        return false
        #else
        return checkSuspiciousFiles() ||
               checkSuspiciousApps() ||
               checkWritableLocations() ||
               checkCydiaURLScheme() ||
               checkSystemCalls() ||
               checkEnvironmentVariables()
        #endif
    }

    // MARK: - Detection Methods

    private func checkSuspiciousFiles() -> Bool {
        let suspiciousFiles = [
            "/Applications/Cydia.app",
            "/Applications/Sileo.app",
            "/Applications/Zebra.app",
            "/Library/MobileSubstrate/MobileSubstrate.dylib",
            "/bin/bash",
            "/usr/sbin/sshd",
            "/etc/apt",
            "/private/var/lib/apt/",
            "/private/var/lib/cydia",
            "/private/var/tmp/cydia.log",
            "/usr/bin/ssh",
            "/usr/libexec/ssh-keysign",
            "/usr/bin/sshd",
            "/usr/libexec/sftp-server",
            "/var/cache/apt",
            "/var/lib/apt",
            "/var/lib/cydia",
            "/etc/ssh/sshd_config"
        ]

        for path in suspiciousFiles {
            if FileManager.default.fileExists(atPath: path) {
                return true
            }
        }

        return false
    }

    private func checkSuspiciousApps() -> Bool {
        let schemes = ["cydia://", "sileo://", "zbra://"]

        for scheme in schemes {
            if let url = URL(string: scheme),
               UIApplication.shared.canOpenURL(url) {
                return true
            }
        }

        return false
    }

    private func checkWritableLocations() -> Bool {
        let testPath = "/private/jailbreak-test.txt"
        let testString = "jailbreak test"

        do {
            try testString.write(toFile: testPath, atomically: true, encoding: .utf8)
            try FileManager.default.removeItem(atPath: testPath)
            return true // 書き込めたら脱獄済み
        } catch {
            return false
        }
    }

    private func checkCydiaURLScheme() -> Bool {
        guard let url = URL(string: "cydia://package/com.example.package") else {
            return false
        }
        return UIApplication.shared.canOpenURL(url)
    }

    private func checkSystemCalls() -> Bool {
        let paths = ["/bin/bash", "/usr/bin/ssh"]

        for path in paths {
            if let file = fopen(path, "r") {
                fclose(file)
                return true
            }
        }

        return false
    }

    private func checkEnvironmentVariables() -> Bool {
        guard let env = getenv("DYLD_INSERT_LIBRARIES") else {
            return false
        }

        return String(cString: env).count > 0
    }

    // MARK: - Advanced Detection

    func detectHooking() -> Bool {
        // メソッドスウィズリング検出
        let originalMethod = class_getInstanceMethod(UIApplication.self, #selector(UIApplication.canOpenURL(_:)))
        let swizzledMethod = class_getInstanceMethod(UIApplication.self, #selector(UIApplication.canOpenURL(_:)))

        return originalMethod != swizzledMethod
    }

    func checkIntegrity() -> Bool {
        // アプリのバイナリ整合性チェック
        guard let executableURL = Bundle.main.executableURL else {
            return false
        }

        do {
            let data = try Data(contentsOf: executableURL)
            let hash = sha256(data)

            // 期待されるハッシュと比較
            let expectedHash = getExpectedHash()
            return hash == expectedHash
        } catch {
            return false
        }
    }

    private func sha256(_ data: Data) -> String {
        var hash = [UInt8](repeating: 0, count: Int(CC_SHA256_DIGEST_LENGTH))
        data.withUnsafeBytes {
            _ = CC_SHA256($0.baseAddress, CC_LONG(data.count), &hash)
        }
        return hash.map { String(format: "%02x", $0) }.joined()
    }

    private func getExpectedHash() -> String {
        // ビルド時に計算されたハッシュを返す
        return UserDefaults.standard.string(forKey: "expectedBinaryHash") ?? ""
    }
}

import CommonCrypto
```

## 検知時の対応

脱獄を検知した場合の処理です。

```swift
// Core/Security/JailbreakHandler.swift
final class JailbreakHandler {
    enum Action {
        case alert
        case disableFeatures
        case exit
        case reportToServer
    }

    static func handle(jailbroken: Bool) {
        guard jailbroken else { return }

        #if !DEBUG
        performActions([.alert, .disableFeatures, .reportToServer])
        #endif
    }

    private static func performActions(_ actions: [Action]) {
        for action in actions {
            switch action {
            case .alert:
                showJailbreakAlert()
            case .disableFeatures:
                disableSensitiveFeatures()
            case .exit:
                exit(0)
            case .reportToServer:
                reportToServer()
            }
        }
    }

    private static func showJailbreakAlert() {
        DispatchQueue.main.async {
            let alert = UIAlertController(
                title: "セキュリティ警告",
                message: "このアプリは改変されたデバイスでは動作しません。",
                preferredStyle: .alert
            )

            alert.addAction(UIAlertAction(title: "OK", style: .default) { _ in
                exit(0)
            })

            if let rootVC = UIApplication.shared.windows.first?.rootViewController {
                rootVC.present(alert, animated: true)
            }
        }
    }

    private static func disableSensitiveFeatures() {
        // 機密機能を無効化
        UserDefaults.standard.set(true, forKey: "jailbroken")
        NotificationCenter.default.post(name: .jailbreakDetected, object: nil)
    }

    private static func reportToServer() {
        // サーバーに通知
        Task {
            do {
                let url = URL(string: "https://api.example.com/security/jailbreak")!
                var request = URLRequest(url: url)
                request.httpMethod = "POST"

                let body = [
                    "device_id": UIDevice.current.identifierForVendor?.uuidString ?? "",
                    "timestamp": ISO8601DateFormatter().string(from: Date())
                ]

                request.httpBody = try JSONSerialization.data(withJSONObject: body)

                let (_, _) = try await URLSession.shared.data(for: request)
            } catch {
                print("Failed to report jailbreak: \(error)")
            }
        }
    }
}

extension Notification.Name {
    static let jailbreakDetected = Notification.Name("jailbreakDetected")
}
```

## SwiftUIでの統合

```swift
// App/MyAwesomeAppApp.swift
import SwiftUI

@main
struct MyAwesomeAppApp: App {
    @State private var isJailbroken = false

    var body: some Scene {
        WindowGroup {
            if isJailbroken {
                JailbreakWarningView()
            } else {
                ContentView()
            }
        }
        .onAppear {
            checkJailbreak()
        }
    }

    private func checkJailbreak() {
        let jailbroken = JailbreakDetector.shared.isJailbroken()
        isJailbroken = jailbroken

        if jailbroken {
            JailbreakHandler.handle(jailbroken: true)
        }
    }
}

struct JailbreakWarningView: View {
    var body: some View {
        VStack(spacing: 20) {
            Image(systemName: "exclamationmark.shield.fill")
                .font(.system(size: 60))
                .foregroundColor(.red)

            Text("セキュリティ警告")
                .font(.title)
                .fontWeight(.bold)

            Text("このアプリは改変されたデバイスでは動作しません。")
                .multilineTextAlignment(.center)
                .padding()

            Button("終了") {
                exit(0)
            }
            .buttonStyle(.borderedProminent)
            .tint(.red)
        }
        .padding()
    }
}
```

## 機能制限の実装

脱獄デバイスで特定機能を制限します。

```swift
final class FeatureGate {
    static func canUseFeature(_ feature: String) -> Bool {
        let isJailbroken = UserDefaults.standard.bool(forKey: "jailbroken")

        guard !isJailbroken else {
            return false
        }

        // その他の条件チェック
        return true
    }
}

// 使用例
if FeatureGate.canUseFeature("payment") {
    // 決済処理
} else {
    showRestrictedFeatureAlert()
}
```

## 偽装対策

検知ロジックの偽装を防ぎます。

```swift
extension JailbreakDetector {
    func performObfuscatedCheck() -> Bool {
        // 検知ロジックを難読化
        let checks: [() -> Bool] = [
            { self.checkSuspiciousFiles() },
            { self.checkSuspiciousApps() },
            { self.checkWritableLocations() },
            { self.checkCydiaURLScheme() }
        ]

        // ランダムな順序で実行
        let shuffled = checks.shuffled()

        for check in shuffled {
            if check() {
                return true
            }
        }

        return false
    }

    func delayedCheck(completion: @escaping (Bool) -> Void) {
        // 遅延実行で検知を難読化
        DispatchQueue.global().asyncAfter(deadline: .now() + Double.random(in: 1...3)) {
            let result = self.isJailbroken()
            DispatchQueue.main.async {
                completion(result)
            }
        }
    }
}
```

## Part 2 セキュリティ実装のまとめ

Part 2では、iOS開発における包括的なセキュリティ実装を学びました：

- **第9章**: OAuth 2.0 / OpenID Connectによる認証
- **第10章**: Keychainの基本操作
- **第11章**: Keychainの高度な活用
- **第12章**: 証明書ピンニングによる通信保護
- **第13章**: App Transport Securityの設定
- **第14章**: CryptoKitによるデータ暗号化
- **第15章**: 脱獄検知と対処

これらの技術を組み合わせることで、セキュアなiOSアプリケーションの構築が期待されます。Part 3では、ネットワーク通信とデータ永続化について解説します。
