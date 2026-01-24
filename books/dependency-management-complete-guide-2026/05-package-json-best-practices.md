---
title: "package.json ベストプラクティス"
---

# package.json ベストプラクティス

## package.jsonの基本構造

package.jsonは、プロジェクトのメタデータと依存関係を定義する重要なファイルです。

```json
{
  "name": "@mycompany/my-app",
  "version": "1.0.0",
  "description": "素晴らしいアプリケーション",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "type": "module",
  
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "test": "vitest run"
  },
  
  "dependencies": {
    "react": "^18.2.0"
  },
  
  "devDependencies": {
    "typescript": "^5.3.0"
  },
  
  "engines": {
    "node": ">=18.0.0",
    "npm": ">=9.0.0"
  },
  
  "keywords": ["react", "typescript"],
  "author": "Your Name <you@example.com>",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "https://github.com/username/repo"
  }
}
```

## バージョン指定のベストプラクティス

### Semantic Versioning (SemVer)

```
MAJOR.MINOR.PATCH
  2  .  1  .  3

MAJOR: 破壊的変更
MINOR: 後方互換性のある新機能
PATCH: 後方互換性のあるバグ修正
```

### バージョン範囲指定

```json
{
  "dependencies": {
    // 正確なバージョン（推奨：本番依存）
    "react": "18.2.0",
    
    // Caret（^）: マイナーバージョンまで許可
    "lodash": "^4.17.21",    // >=4.17.21 <5.0.0
    
    // Tilde（~）: パッチバージョンのみ許可
    "axios": "~1.6.0",       // >=1.6.0 <1.7.0
    
    // 範囲指定
    "typescript": ">=5.0.0 <5.4.0",
    
    // Latest（非推奨）
    "some-package": "latest"  // ❌ 避けるべき
  }
}
```

### 推奨されるバージョン戦略

```json
{
  "dependencies": {
    // クリティカルなパッケージ：正確なバージョン
    "react": "18.2.0",
    "next": "14.0.4",
    
    // 重要なパッケージ：パッチのみ許可
    "lodash": "~4.17.21",
    "date-fns": "~2.30.0",
    
    // 一般的なパッケージ：マイナー更新許可
    "axios": "^1.6.0"
  },
  
  "devDependencies": {
    // 開発ツール：柔軟に更新
    "typescript": "^5.3.0",
    "eslint": "^8.56.0"
  }
}
```

## scriptsセクションのベストプラクティス

### 標準的なscripts

```json
{
  "scripts": {
    // 開発
    "dev": "vite",
    "start": "node dist/index.js",
    
    // ビルド
    "build": "tsc && vite build",
    "build:prod": "NODE_ENV=production npm run build",
    
    // テスト
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test",
    
    // リント・フォーマット
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write \"src/**/*.{ts,tsx}\"",
    "type-check": "tsc --noEmit",
    
    // 依存関係管理
    "deps:update": "ncu -u && npm install",
    "deps:audit": "npm audit",
    
    // クリーン
    "clean": "rm -rf dist node_modules",
    "clean:install": "npm run clean && npm install"
  }
}
```

### ライフサイクルフック

```json
{
  "scripts": {
    "preinstall": "node scripts/check-node-version.js",
    "postinstall": "patch-package",
    "prepare": "husky install",
    "prebuild": "npm run lint && npm run type-check",
    "build": "tsc && vite build",
    "postbuild": "npm run test"
  }
}
```

## engines フィールド

プロジェクトで必要なNode.jsとnpmのバージョンを指定します。

```json
{
  "engines": {
    "node": ">=18.0.0",
    "npm": ">=9.0.0"
  }
}
```

`.npmrc`で厳密にチェック：

```bash
# .npmrc
engine-strict=true
```

## exports フィールド（パッケージ公開時）

```json
{
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    },
    "./utils": {
      "import": "./dist/utils.mjs",
      "types": "./dist/utils.d.ts"
    }
  }
}
```

## peerDependencies

ライブラリを公開する場合、ホストアプリが提供すべき依存関係を指定します。

```json
{
  "peerDependencies": {
    "react": ">=16.8.0",
    "react-dom": ">=16.8.0"
  },
  "peerDependenciesMeta": {
    "react-dom": {
      "optional": true
    }
  }
}
```

## overrides / resolutions

依存関係のバージョンを強制的に上書きします。

### npm (v8.3+)

```json
{
  "overrides": {
    "lodash": "4.17.21",
    "minimist": ">=1.2.6"
  }
}
```

### Yarn

```json
{
  "resolutions": {
    "lodash": "4.17.21",
    "**/minimist": ">=1.2.6"
  }
}
```

### pnpm

```json
{
  "pnpm": {
    "overrides": {
      "lodash": "4.17.21"
    }
  }
}
```

## まとめ

package.jsonのベストプラクティス：

1. **バージョン指定**: クリティカルなパッケージは正確なバージョンを使用
2. **scripts**: 標準的な命名規則を採用
3. **engines**: 必要なNode.jsバージョンを明記
4. **exports**: パッケージ公開時は適切なエントリーポイントを定義

次章では、ロックファイルの管理について解説します。
