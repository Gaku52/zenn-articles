# Node.jsスクリプト自動化

本章では、Node.js/TypeScriptを使った自動化スクリプトの開発を学びます。

## 基本スクリプト

```typescript
#!/usr/bin/env ts-node

import { execSync } from 'child_process'

const env = process.argv[2] || 'development'

console.log(`Deploying to ${env}...`)

// ビルド
execSync('npm run build', { stdio: 'inherit' })

// デプロイ
if (env === 'production') {
    execSync('ssh user@server "cd /var/www && git pull"', { stdio: 'inherit' })
}
```

## ファイル操作

```typescript
import fs from 'fs/promises'
import path from 'path'

async function processFiles(dirPath: string) {
    const files = await fs.readdir(dirPath)

    for (const file of files) {
        const filePath = path.join(dirPath, file)
        const content = await fs.readFile(filePath, 'utf-8')
        const processed = content.toUpperCase()
        await fs.writeFile(filePath, processed)
    }
}
```

## まとめ

Node.jsスクリプトにより、TypeScriptの型安全性を活かした自動化が可能です。
