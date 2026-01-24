---
title: "App Transport Security"
---

# App Transport Security

App Transport Security（ATS）は、iOSアプリの通信を保護するためのセキュリティ機能です。本章では、ATSの設定と適切な運用方法を解説します。

## ATSの概要

ATSは、アプリとサーバー間の通信に対して以下の要件を強制します：

- HTTPS通信のみ許可
- TLS 1.2以降の使用
- 強力な暗号化スイートの使用
- 有効な証明書の使用

### デフォルトの動作

```plaintext
許可される通信:
✅ https://api.example.com
✅ TLS 1.2以降

拒否される通信:
❌ http://api.example.com（HTTP）
❌ TLS 1.0/1.1
❌ 弱い暗号化
```

## Info.plistの設定

ATSの設定はInfo.plistで行います。

### 全体的にATSを有効化（推奨）

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <!-- ATSを有効（デフォルト） -->
    <key>NSAllowsArbitraryLoads</key>
    <false/>
</dict>
```

### 特定ドメインのみHTTPを許可

開発環境など特定の状況下でのみHTTPを許可します。

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSExceptionDomains</key>
    <dict>
        <!-- 開発環境のみHTTPを許可 -->
        <key>localhost</key>
        <dict>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/>
            <key>NSIncludesSubdomains</key>
            <true/>
        </dict>

        <!-- ステージング環境 -->
        <key>staging-api.example.com</key>
        <dict>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/>
            <key>NSIncludesSubdomains</key>
            <false/>
        </dict>
    </dict>
</dict>
```

### より厳格な設定

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSExceptionDomains</key>
    <dict>
        <key>api.example.com</key>
        <dict>
            <!-- TLS 1.2を最低要件に -->
            <key>NSExceptionMinimumTLSVersion</key>
            <string>TLSv1.2</string>

            <!-- 証明書の透明性を要求 -->
            <key>NSRequiresCertificateTransparency</key>
            <true/>

            <!-- Forward Secrecyを要求 -->
            <key>NSExceptionRequiresForwardSecrecy</key>
            <true/>
        </dict>
    </dict>
</dict>
```

## プログラマティックチェック

コード内でURLのセキュリティを検証します。

```swift
// Core/Security/NetworkSecurityValidator.swift
import Foundation

enum NetworkSecurityValidator {
    static func validate(_ url: URL, allowHTTP: Bool = false) throws {
        // スキームの検証
        guard let scheme = url.scheme?.lowercased() else {
            throw SecurityError.invalidURL
        }

        // 本番環境ではHTTPSのみ
        #if !DEBUG
        guard scheme == "https" else {
            throw SecurityError.httpsRequired
        }
        #else
        // 開発環境ではlocalhostのみHTTP許可
        if scheme == "http" {
            guard allowHTTP || url.host == "localhost" || url.host == "127.0.0.1" else {
                throw SecurityError.httpsRequired
            }
        }
        #endif

        // プライベートIPへのアクセスチェック
        if let host = url.host {
            guard !isPrivateIP(host) || allowHTTP else {
                throw SecurityError.privateIPNotAllowed
            }
        }
    }

    private static func isPrivateIP(_ host: String) -> Bool {
        // プライベートIPアドレスの範囲
        let privateRanges = [
            "10.",
            "172.16.", "172.17.", "172.18.", "172.19.",
            "172.20.", "172.21.", "172.22.", "172.23.",
            "172.24.", "172.25.", "172.26.", "172.27.",
            "172.28.", "172.29.", "172.30.", "172.31.",
            "192.168."
        ]

        return privateRanges.contains { host.hasPrefix($0) }
    }
}

enum SecurityError: Error, LocalizedError {
    case invalidURL
    case httpsRequired
    case privateIPNotAllowed

    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "無効なURLです"
        case .httpsRequired:
            return "HTTPSが必要です"
        case .privateIPNotAllowed:
            return "プライベートIPへのアクセスは許可されていません"
        }
    }
}
```

### NetworkManagerでの統合

```swift
class SecureNetworkManager {
    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        let request = try endpoint.makeRequest()

        // URLのセキュリティ検証
        try NetworkSecurityValidator.validate(request.url!)

        let (data, response) = try await URLSession.shared.data(for: request)

        // レスポンスの検証
        try validateResponse(response)

        return try JSONDecoder().decode(T.self, from: data)
    }

    private func validateResponse(_ response: URLResponse) throws {
        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }

        // HTTPSかどうか確認
        if let url = httpResponse.url {
            try NetworkSecurityValidator.validate(url)
        }

        guard (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.httpError(httpResponse.statusCode)
        }
    }
}
```

## 環境別の設定管理

開発、ステージング、本番で異なるATS設定を適用します。

### xcconfig での管理

```plaintext
// Debug.xcconfig
ATS_LOCALHOST_EXCEPTION = YES
ATS_STAGING_EXCEPTION = YES

// Staging.xcconfig
ATS_LOCALHOST_EXCEPTION = NO
ATS_STAGING_EXCEPTION = YES

// Release.xcconfig
ATS_LOCALHOST_EXCEPTION = NO
ATS_STAGING_EXCEPTION = NO
```

### Info.plist での条件分岐

Build Settingsで定義した変数をInfo.plistで参照します。

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSExceptionDomains</key>
    <dict>
        <!-- ローカル環境 -->
        #if ATS_LOCALHOST_EXCEPTION
        <key>localhost</key>
        <dict>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/>
        </dict>
        #endif
    </dict>
</dict>
```

## セキュリティヘッダーの検証

サーバーから返されるセキュリティヘッダーを確認します。

```swift
extension SecureNetworkManager {
    func validateSecurityHeaders(_ response: HTTPURLResponse) -> [String: String] {
        var issues: [String: String] = [:]

        // Strict-Transport-Security (HSTS)
        if response.value(forHTTPHeaderField: "Strict-Transport-Security") == nil {
            issues["HSTS"] = "Strict-Transport-Securityヘッダーが設定されていません"
        }

        // Content-Security-Policy
        if response.value(forHTTPHeaderField: "Content-Security-Policy") == nil {
            issues["CSP"] = "Content-Security-Policyヘッダーが設定されていません"
        }

        // X-Content-Type-Options
        if response.value(forHTTPHeaderField: "X-Content-Type-Options") == nil {
            issues["X-Content-Type-Options"] = "X-Content-Type-Optionsヘッダーが設定されていません"
        }

        // X-Frame-Options
        if response.value(forHTTPHeaderField: "X-Frame-Options") == nil {
            issues["X-Frame-Options"] = "X-Frame-Optionsヘッダーが設定されていません"
        }

        #if DEBUG
        if !issues.isEmpty {
            print("⚠️ Security Headers Issues:")
            issues.forEach { key, value in
                print("  - \(key): \(value)")
            }
        }
        #endif

        return issues
    }
}
```

## WebView でのATS

WKWebViewでもATSが適用されます。

```swift
import WebKit

class SecureWebViewController: UIViewController {
    private let webView = WKWebView()

    override func viewDidLoad() {
        super.viewDidLoad()

        view.addSubview(webView)
        webView.frame = view.bounds
        webView.navigationDelegate = self

        // HTTPSのURLのみロード
        if let url = URL(string: "https://example.com") {
            loadURL(url)
        }
    }

    private func loadURL(_ url: URL) {
        do {
            try NetworkSecurityValidator.validate(url)
            let request = URLRequest(url: url)
            webView.load(request)
        } catch {
            showError("セキュアな接続を確立できません: \(error.localizedDescription)")
        }
    }

    private func showError(_ message: String) {
        let alert = UIAlertController(
            title: "エラー",
            message: message,
            preferredStyle: .alert
        )
        alert.addAction(UIAlertAction(title: "OK", style: .default))
        present(alert, animated: true)
    }
}

extension SecureWebViewController: WKNavigationDelegate {
    func webView(
        _ webView: WKWebView,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        // 証明書ピンニングの実装
        if let serverTrust = challenge.protectionSpace.serverTrust {
            // 検証ロジック
            completionHandler(.useCredential, URLCredential(trust: serverTrust))
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }
}
```

## デバッグとトラブルシューティング

### ATS関連エラーの確認

```plaintext
エラーメッセージ例:
App Transport Security has blocked a cleartext HTTP (http://) resource load since it is insecure.

原因:
- HTTPでの通信を試みた
- ATSの例外設定が正しくない
```

### ログの有効化

```swift
#if DEBUG
func logATSConfiguration() {
    guard let atsSettings = Bundle.main.object(forInfoDictionaryKey: "NSAppTransportSecurity") as? [String: Any] else {
        print("ATS設定が見つかりません")
        return
    }

    print("=== ATS Configuration ===")
    print(atsSettings)
    print("========================")
}
#endif
```

## まとめ

本章では、App Transport Securityの設定と運用方法を解説しました。適切なATS設定により、以下の効果が期待されます：

- 通信の暗号化強制
- 中間者攻撃のリスク低減
- セキュアな通信の保証

次章では、データ暗号化について解説します。
