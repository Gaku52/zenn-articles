---
title: "HTTP/2とHTTP/3"
---

# HTTP/2とHTTP/3

最新のHTTPプロトコルを活用してパフォーマンスを向上させる方法を学びます。

## HTTP/2の特徴

### Multiplexing

1つのTCP接続で複数のリクエストを並列処理。

### Server Push

```nginx
# nginx設定
location / {
  http2_push /styles.css;
  http2_push /app.js;
}
```

### Header Compression (HPACK)

HTTPヘッダーを圧縮して転送量を削減。

## HTTP/3の特徴

### QUIC Protocol

- UDP based
- 0-RTT接続確立
- パケットロス時の高速リカバリ

### 有効化

```nginx
# nginx 1.25+
listen 443 quic reuseport;
listen 443 ssl http2;

add_header Alt-Svc 'h3=":443"; ma=86400';
```

## パフォーマンス比較

| プロトコル | 接続確立 | 並列リクエスト | パケットロス時 |
|-----------|---------|---------------|---------------|
| HTTP/1.1 | 遅い | 6接続 | 影響大 |
| HTTP/2 | 中程度 | 無制限 | 影響中 |
| HTTP/3 | 高速 (0-RTT) | 無制限 | 影響小 |

## 実装

```javascript
// Next.js (自動対応)
module.exports = {
  // Vercelは自動的にHTTP/3対応
}

// Vercel設定
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "Alt-Svc",
          "value": "h3=\":443\"; ma=86400"
        }
      ]
    }
  ]
}
```

## 改善事例

**Before:** HTTP/1.1、初回接続 320ms
**After:** HTTP/3、初回接続 85ms（73%改善）
