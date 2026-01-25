---
title: "Node.js CLIセットアップ"
---

# Node.js CLIセットアップ

本章では、Node.js/TypeScriptでCLIツールを開発するためのプロジェクトセットアップを学びます。適切な初期設定により、開発効率と品質が大きく向上します。

## プロジェクト初期化

### 基本セットアップ

```bash
# プロジェクト作成
mkdir my-cli-tool
cd my-cli-tool

# package.json作成
npm init -y

# TypeScript環境セットアップ
npm install -D typescript @types/node ts-node

# TypeScript設定ファイル作成
npx tsc --init
```

### package.json設定

```json
{
  "name": "my-cli-tool",
  "version": "1.0.0",
  "description": "A powerful CLI tool",
  "main": "dist/index.js",
  "bin": {
    "mycli": "./dist/index.js"
  },
  "scripts": {
    "dev": "ts-node src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "watch": "tsc --watch",
    "test": "jest",
    "lint": "eslint src/**/*.ts",
    "format": "prettier --write \"src/**/*.ts\""
  },
  "keywords": ["cli", "tool"],
  "author": "Your Name <your.email@example.com>",
  "license": "MIT",
  "files": [
    "dist",
    "README.md",
    "LICENSE"
  ],
  "engines": {
    "node": ">=18.0.0"
  }
}
```

## ディレクトリ構造

推奨されるプロジェクト構造:

```
my-cli-tool/
├── src/
│   ├── commands/         # コマンド実装
│   │   ├── create.ts
│   │   ├── list.ts
│   │   └── delete.ts
│   ├── core/            # ビジネスロジック
│   │   ├── project.ts
│   │   └── template.ts
│   ├── utils/           # ユーティリティ
│   │   ├── logger.ts
│   │   ├── config.ts
│   │   └── error.ts
│   ├── types/           # 型定義
│   │   └── index.ts
│   └── index.ts         # エントリーポイント
├── tests/               # テスト
│   ├── commands/
│   └── core/
├── templates/           # テンプレートファイル
├── dist/                # ビルド出力
├── .gitignore
├── .eslintrc.json
├── .prettierrc
├── tsconfig.json
├── jest.config.js
├── package.json
└── README.md
```

## TypeScript設定

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "moduleResolution": "node",
    "types": ["node"]
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

### パス解決

```json
{
  "compilerOptions": {
    "baseUrl": "./",
    "paths": {
      "@/*": ["src/*"],
      "@commands/*": ["src/commands/*"],
      "@core/*": ["src/core/*"],
      "@utils/*": ["src/utils/*"]
    }
  }
}
```

使用例:

```typescript
// パスエイリアスを使用
import { Logger } from '@utils/logger'
import { CreateCommand } from '@commands/create'
```

## エントリーポイント

### Shebang

CLIとして実行可能にするため、shebangを追加します。

```typescript
#!/usr/bin/env node

// src/index.ts
import { program } from 'commander'
import { createCommand } from './commands/create'
import { listCommand } from './commands/list'
import { deleteCommand } from './commands/delete'

program
  .name('mycli')
  .description('A powerful CLI tool')
  .version('1.0.0')

program.addCommand(createCommand)
program.addCommand(listCommand)
program.addCommand(deleteCommand)

program.parse()
```

### ビルド後のパーミッション

```json
{
  "scripts": {
    "build": "tsc && chmod +x dist/index.js"
  }
}
```

## 依存関係

### CLI開発に必須のパッケージ

```bash
# コマンドラインパーサー
npm install commander

# インタラクティブプロンプト
npm install inquirer
npm install -D @types/inquirer

# カラー出力
npm install chalk

# スピナー・プログレスバー
npm install ora

# ファイル操作
npm install fs-extra
npm install -D @types/fs-extra
```

### 推奨パッケージ

```bash
# 設定ファイル管理
npm install cosmiconfig

# バリデーション
npm install zod

# 日付操作
npm install date-fns

# HTTP リクエスト
npm install axios
```

## リンター・フォーマッター

### ESLint設定

```bash
npm install -D eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

```json
// .eslintrc.json
{
  "parser": "@typescript-eslint/parser",
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended"
  ],
  "parserOptions": {
    "ecmaVersion": 2020,
    "sourceType": "module"
  },
  "rules": {
    "@typescript-eslint/explicit-function-return-type": "warn",
    "@typescript-eslint/no-unused-vars": "error",
    "@typescript-eslint/no-explicit-any": "warn"
  }
}
```

### Prettier設定

```bash
npm install -D prettier eslint-config-prettier
```

```json
// .prettierrc
{
  "semi": false,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100
}
```

## テスト環境

### Jest設定

```bash
npm install -D jest ts-jest @types/jest
```

```javascript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/tests'],
  testMatch: ['**/__tests__/**/*.ts', '**/?(*.)+(spec|test).ts'],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/index.ts'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  }
}
```

## 開発ツール

### Nodemon（ホットリロード）

```bash
npm install -D nodemon
```

```json
{
  "scripts": {
    "dev": "nodemon --exec ts-node src/index.ts",
    "dev:watch": "nodemon --watch src --ext ts --exec ts-node src/index.ts"
  }
}
```

```json
// nodemon.json
{
  "watch": ["src"],
  "ext": "ts",
  "ignore": ["src/**/*.spec.ts"],
  "exec": "ts-node src/index.ts"
}
```

### ts-node-dev

より高速な開発体験:

```bash
npm install -D ts-node-dev
```

```json
{
  "scripts": {
    "dev": "ts-node-dev --respawn --transpile-only src/index.ts"
  }
}
```

## ローカルテスト

### npm link

グローバルにリンクしてテストします。

```bash
# プロジェクトディレクトリで
npm link

# これでグローバルから実行可能に
mycli --help

# リンク解除
npm unlink -g mycli
```

### ローカルインストール

```bash
# プロジェクトディレクトリで
npm pack
# my-cli-tool-1.0.0.tgz が生成される

# 別のディレクトリでインストール
npm install /path/to/my-cli-tool-1.0.0.tgz

# 実行
npx mycli --help
```

## デバッグ

### VS Code設定

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug CLI",
      "runtimeArgs": ["-r", "ts-node/register"],
      "args": ["${workspaceFolder}/src/index.ts", "create", "myapp"],
      "cwd": "${workspaceFolder}",
      "protocol": "inspector",
      "console": "integratedTerminal"
    }
  ]
}
```

### console.log デバッグ

```typescript
// src/utils/logger.ts
export class Logger {
  private debug: boolean

  constructor(debug: boolean = false) {
    this.debug = debug
  }

  log(message: string): void {
    console.log(message)
  }

  error(message: string): void {
    console.error(message)
  }

  debugLog(message: string): void {
    if (this.debug) {
      console.log(`[DEBUG] ${message}`)
    }
  }
}

// 使用例
const logger = new Logger(process.env.DEBUG === 'true')
logger.debugLog('Processing file: example.txt')
```

## Git設定

### .gitignore

```
# Dependencies
node_modules/

# Build
dist/
*.tgz

# Testing
coverage/

# Environment
.env
.env.local

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db

# Logs
*.log
```

### Husky + lint-staged

コミット前の自動チェック:

```bash
npm install -D husky lint-staged
npx husky install
```

```json
// package.json
{
  "scripts": {
    "prepare": "husky install"
  },
  "lint-staged": {
    "*.ts": [
      "eslint --fix",
      "prettier --write"
    ]
  }
}
```

```bash
# pre-commit フック作成
npx husky add .husky/pre-commit "npx lint-staged"
```

## まとめ

本章では、Node.js CLIプロジェクトのセットアップを学びました。

**重要ポイント**:
- 適切なディレクトリ構造でプロジェクトを整理
- TypeScriptで型安全性を確保
- ESLint/Prettierでコード品質を維持
- Jestでテストを自動化
- npm linkでローカルテストを実施
- Huskyでコミット前チェックを自動化

次章では、Commander.jsを使った実践的な引数パースを学びます。
