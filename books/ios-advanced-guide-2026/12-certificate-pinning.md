---
title: "証明書ピンニング"
---

# 証明書ピンニング

証明書ピンニングは、中間者攻撃（MITM）を防ぐための重要なセキュリティ手法です。本章では、証明書ピンニングの実装方法を解説します。

## 証明書ピンニングとは

通常のHTTPS通信では、サーバー証明書がCA（認証局）によって署名されているかを確認します。証明書ピンニングは、さらに特定の証明書または公開鍵のみを信頼する仕組みです。

### 攻撃シナリオ

```plaintext
通常のHTTPS:
1. 攻撃者が偽のCA証明書をデバイスにインストール
2. 攻撃者が中間者として通信を傍受
3. デバイスは偽の証明書を信頼してしまう

証明書ピンニング:
1. アプリが特定の証明書のみを信頼
2. 偽の証明書は拒否される
3. 通信は確立されない
```

## ピンニング方式の選択

### Certificate Pinning（証明書ピンニング）

証明書全体のハッシュを検証します。

**メリット:**
- 実装が比較的簡単
- 証明書の完全一致を確認

**デメリット:**
- 証明書更新時にアプリの更新が必要

### Public Key Pinning（公開鍵ピンニング）

公開鍵のハッシュを検証します。

**メリット:**
- 証明書を更新しても公開鍵が同じなら動作
- より柔軟な運用が可能

**デメリット:**
- 実装がやや複雑

## URLSessionDelegate での実装

```swift
// Core/Networking/CertificatePinner.swift
import Foundation
import CryptoKit

final class CertificatePinner: NSObject {
    // ピンニングする証明書のSHA256ハッシュ
    private let pinnedCertificateHashes: Set<String> = [
        "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=",
        "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB="  // バックアップ証明書
    ]

    func validate(_ serverTrust: SecTrust, forDomain domain: String) -> Bool {
        // サーバー証明書チェーンを取得
        guard let certificateChain = SecTrustCopyCertificateChain(serverTrust) as? [SecCertificate] else {
            return false
        }

        // チェーン内の証明書を検証
        for certificate in certificateChain {
            let certificateData = SecCertificateCopyData(certificate) as Data
            let hash = sha256Hash(of: certificateData)

            if pinnedCertificateHashes.contains(hash) {
                return true
            }
        }

        return false
    }

    private func sha256Hash(of data: Data) -> String {
        let hash = SHA256.hash(data: data)
        return "sha256/" + Data(hash).base64EncodedString()
    }
}

// MARK: - Network Manager with Pinning

final class SecureNetworkManager: NSObject {
    static let shared = SecureNetworkManager()

    private lazy var session: URLSession = {
        let configuration = URLSessionConfiguration.default
        configuration.timeoutIntervalForRequest = 30
        return URLSession(configuration: configuration, delegate: self, delegateQueue: nil)
    }()

    private let certificatePinner = CertificatePinner()

    func request(_ url: URL) async throws -> Data {
        let (data, response) = try await session.data(from: url)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.invalidResponse
        }

        return data
    }
}

// MARK: - URLSessionDelegate

extension SecureNetworkManager: URLSessionDelegate {
    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        // Server Trust認証のみ処理
        guard challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
              let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        // 証明書ピンニングの検証
        if certificatePinner.validate(serverTrust, forDomain: challenge.protectionSpace.host) {
            let credential = URLCredential(trust: serverTrust)
            completionHandler(.useCredential, credential)
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }
}

enum NetworkError: Error {
    case invalidResponse
    case pinningFailed
}
```

## 公開鍵ピンニングの実装

```swift
final class PublicKeyPinner {
    private let pinnedPublicKeyHashes: Set<String> = [
        "sha256/CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC=",
        "sha256/DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDD="
    ]

    func validate(_ serverTrust: SecTrust) -> Bool {
        // 証明書チェーンから公開鍵を抽出
        guard let certificateChain = SecTrustCopyCertificateChain(serverTrust) as? [SecCertificate] else {
            return false
        }

        for certificate in certificateChain {
            // 公開鍵を取得
            guard let publicKey = SecCertificateCopyKey(certificate) else {
                continue
            }

            // 公開鍵データを取得
            guard let publicKeyData = SecKeyCopyExternalRepresentation(publicKey, nil) as Data? else {
                continue
            }

            let hash = sha256Hash(of: publicKeyData)

            if pinnedPublicKeyHashes.contains(hash) {
                return true
            }
        }

        return false
    }

    private func sha256Hash(of data: Data) -> String {
        let hash = SHA256.hash(data: data)
        return "sha256/" + Data(hash).base64EncodedString()
    }
}
```

## ローカル証明書ファイルでのピンニング

証明書ファイルをバンドルに含めて検証します。

```swift
final class BundleCertificatePinner {
    private let certificateNames: [String]

    init(certificateNames: [String]) {
        self.certificateNames = certificateNames
    }

    func validate(_ serverTrust: SecTrust) -> Bool {
        // バンドルから証明書を読み込み
        let bundleCertificates = loadCertificatesFromBundle()

        // サーバー証明書と比較
        guard let serverCertificate = SecTrustGetCertificateAtIndex(serverTrust, 0) else {
            return false
        }

        let serverCertificateData = SecCertificateCopyData(serverCertificate) as Data

        for bundleCertificate in bundleCertificates {
            let bundleCertificateData = SecCertificateCopyData(bundleCertificate) as Data

            if serverCertificateData == bundleCertificateData {
                return true
            }
        }

        return false
    }

    private func loadCertificatesFromBundle() -> [SecCertificate] {
        var certificates: [SecCertificate] = []

        for name in certificateNames {
            guard let path = Bundle.main.path(forResource: name, ofType: "cer"),
                  let certificateData = try? Data(contentsOf: URL(fileURLWithPath: path)),
                  let certificate = SecCertificateCreateWithData(nil, certificateData as CFData) else {
                continue
            }

            certificates.append(certificate)
        }

        return certificates
    }
}
```

## 証明書ハッシュの取得方法

### コマンドラインから取得

```bash
# 証明書のダウンロード
openssl s_client -connect api.example.com:443 -showcerts < /dev/null | openssl x509 -outform DER > certificate.der

# SHA256ハッシュの計算
openssl x509 -inform DER -in certificate.der -pubkey -noout | \
  openssl pkey -pubin -outform DER | \
  openssl dgst -sha256 -binary | \
  base64
```

### Swift での取得

```swift
func printCertificateHash(from url: URL) async {
    let configuration = URLSessionConfiguration.ephemeral
    let session = URLSession(configuration: configuration, delegate: HashExtractor(), delegateQueue: nil)

    let task = session.dataTask(with: url)
    task.resume()
}

class HashExtractor: NSObject, URLSessionDelegate {
    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        guard let serverTrust = challenge.protectionSpace.serverTrust,
              let certificate = SecTrustGetCertificateAtIndex(serverTrust, 0) else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        let certificateData = SecCertificateCopyData(certificate) as Data
        let hash = SHA256.hash(data: certificateData)
        let hashString = "sha256/" + Data(hash).base64EncodedString()

        print("Certificate Hash: \(hashString)")

        completionHandler(.cancelAuthenticationChallenge, nil)
    }
}
```

## Alamofire での実装

Alamofireを使用する場合のピンニング実装です。

```swift
import Alamofire

class AlamofirePinningManager {
    static let shared = AlamofirePinningManager()

    lazy var session: Session = {
        let evaluators = [
            "api.example.com": PublicKeysTrustEvaluator(
                keys: pinnedKeys,
                performDefaultValidation: true,
                validateHost: true
            )
        ]

        let manager = ServerTrustManager(evaluators: evaluators)

        let configuration = URLSessionConfiguration.af.default
        return Session(configuration: configuration, serverTrustManager: manager)
    }()

    private var pinnedKeys: [SecKey] {
        let certificateNames = ["api_certificate"]

        return certificateNames.compactMap { name in
            guard let path = Bundle.main.path(forResource: name, ofType: "cer"),
                  let certificateData = try? Data(contentsOf: URL(fileURLWithPath: path)),
                  let certificate = SecCertificateCreateWithData(nil, certificateData as CFData),
                  let publicKey = SecCertificateCopyKey(certificate) else {
                return nil
            }

            return publicKey
        }
    }

    func request(_ url: String) async throws -> Data {
        try await withCheckedThrowingContinuation { continuation in
            session.request(url).responseData { response in
                switch response.result {
                case .success(let data):
                    continuation.resume(returning: data)
                case .failure(let error):
                    continuation.resume(throwing: error)
                }
            }
        }
    }
}
```

## 証明書ローテーション戦略

証明書更新時の対応戦略です。

### 複数証明書のピンニング

```swift
private let pinnedCertificateHashes: Set<String> = [
    "sha256/CurrentCertificateHash=",     // 現在の証明書
    "sha256/NextCertificateHash=",        // 次の証明書（事前登録）
    "sha256/BackupCertificateHash="       // バックアップ証明書
]
```

### 段階的な移行

```plaintext
1. 新証明書をサーバーに配置（旧証明書も維持）
2. アプリに新証明書のハッシュを追加（旧ハッシュも維持）
3. アプリをリリース
4. 十分な移行期間（例：3ヶ月）経過後
5. 旧証明書とハッシュを削除
```

## まとめ

本章では、証明書ピンニングの実装方法を解説しました。適切なピンニング実装により、以下の効果が期待されます：

- 中間者攻撃の防止
- 通信の安全性向上
- 企業・組織レベルのセキュリティ要件への対応

次章では、App Transport Securityについて解説します。
