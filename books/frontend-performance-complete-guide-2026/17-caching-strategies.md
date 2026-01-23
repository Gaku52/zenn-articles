---
title: "キャッシング戦略"
---

# キャッシング戦略

効果的なキャッシングは、リピーター訪問時のパフォーマンスを劇的に向上させます。

## Cache-Control ヘッダー

```nginx
# 静的アセット: 1年キャッシュ
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2)$ {
  add_header Cache-Control "public, max-age=31536000, immutable";
}

# HTML: キャッシュしない
location ~* \.html$ {
  add_header Cache-Control "no-cache, no-store, must-revalidate";
}
```

## Service Worker

```typescript
// sw.js
const CACHE_NAME = 'v1'

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      return cache.addAll([
        '/',
        '/styles.css',
        '/app.js',
      ])
    })
  )
})

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((response) => {
      return response || fetch(event.request)
    })
  )
})
```

## SWR (Stale-While-Revalidate)

```typescript
import useSWR from 'swr'

function Profile() {
  const { data } = useSWR('/api/user', fetcher, {
    revalidateOnFocus: true,
    revalidateOnReconnect: true,
  })

  return <div>{data.name}</div>
}
```

## CDN キャッシング

```javascript
// Vercel Edge Config
export const config = {
  runtime: 'edge',
}

export default function handler(req) {
  return new Response(data, {
    headers: {
      'Cache-Control': 's-maxage=3600, stale-while-revalidate=86400',
    },
  })
}
```

## 改善事例

**Before:** キャッシュなし、リピーター訪問 3.2秒
**After:** Service Worker + CDN、リピーター訪問 0.4秒（88%改善）
