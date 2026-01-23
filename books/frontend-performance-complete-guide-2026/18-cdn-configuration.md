---
title: "CDN設定"
---

# CDN設定

CDN（Content Delivery Network）は、地理的に分散されたサーバーからコンテンツを配信し、レイテンシを削減します。

## CDNの選択

| CDN | 特徴 | 用途 |
|-----|------|------|
| Cloudflare | 無料プランあり、DDoS対策 | 汎用 |
| Vercel | Next.js最適化 | Next.jsアプリ |
| Netlify | Jamstack最適化 | 静的サイト |
| AWS CloudFront | 高機能、従量課金 | エンタープライズ |

## Cloudflare 設定

```javascript
// Cloudflare Workers
export default {
  async fetch(request) {
    const response = await fetch(request)
    const newResponse = new Response(response.body, response)

    newResponse.headers.set(
      'Cache-Control',
      'public, max-age=3600, s-maxage=86400'
    )

    return newResponse
  },
}
```

## 画像CDN

```typescript
// Cloudinary
const imageUrl = cloudinary.url('sample.jpg', {
  fetch_format: 'auto',
  quality: 'auto',
  width: 800,
})
```

## CDN Purge/Invalidation

```bash
# Cloudflare CLI
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  --data '{"purge_everything":true}'
```

## 改善事例

**Before:** 単一リージョン、東京→NY 280ms
**After:** CDN導入、世界中どこでも 50ms以下（82%改善）
