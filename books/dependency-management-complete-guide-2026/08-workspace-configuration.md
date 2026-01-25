---
title: "ワークスペース設定詳細"
---

# ワークスペース設定詳細

## ワークスペース間の依存関係

ワークスペース間の依存関係を適切に設定することで、開発効率とビルドパフォーマンスが大幅に向上します。

### 内部パッケージの参照

ワークスペース内のパッケージを相互参照する際は、workspace protocolを使用することが推奨されます。

#### npm Workspacesの場合

```json
// apps/web/package.json
{
  "name": "@mycompany/web",
  "version": "1.0.0",
  "dependencies": {
    "@mycompany/ui-components": "*",
    "@mycompany/utils": "*",
    "@mycompany/api-client": "1.0.0",
    "react": "^18.2.0"
  }
}
```

`*`を使用すると、常にローカルの最新バージョンが参照されます。

#### pnpm Workspacesの場合

```json
// apps/web/package.json
{
  "name": "@mycompany/web",
  "version": "1.0.0",
  "dependencies": {
    "@mycompany/ui-components": "workspace:*",
    "@mycompany/utils": "workspace:^",
    "@mycompany/api-client": "workspace:~1.0.0",
    "react": "^18.2.0"
  }
}
```

workspace protocolのバリエーション:

- `workspace:*`: ローカルの最新バージョン（開発環境推奨）
- `workspace:^`: セマンティックバージョニング互換（マイナーバージョン更新許可）
- `workspace:~`: パッチバージョンのみ更新許可
- `workspace:1.0.0`: 固定バージョン（本番環境推奨）

### パッケージ公開時の挙動

`pnpm publish`実行時、workspace protocolは実際のバージョン番号に自動変換されます。

```json
// publish前（開発時）
{
  "dependencies": {
    "@mycompany/utils": "workspace:^"
  }
}

// publish後（npm registry上）
{
  "dependencies": {
    "@mycompany/utils": "^1.2.3"
  }
}
```

## ホイスティング（Hoisting）

ホイスティングは、依存関係を上位のnode_modulesに集約する仕組みです。

### npm/yarnのホイスティング

npm/yarnでは、デフォルトで依存関係がルートのnode_modulesにホイストされます。

```
my-monorepo/
├── node_modules/          # ここに集約される
│   ├── react/
│   ├── lodash/
│   └── typescript/
├── packages/
│   └── ui-components/
│       └── package.json   # react: ^18.2.0
└── apps/
    └── web/
        └── package.json   # react: ^18.2.0
```

メリット:
- ディスク容量の節約
- インストール時間の短縮

デメリット:
- 宣言していない依存関係にアクセスできる（Phantom Dependencies）
- バージョン不整合が起きる可能性

### pnpmの厳格なホイスティング

pnpmは、依存関係を厳密に管理し、Phantom Dependenciesを防ぎます。

```
my-monorepo/
├── node_modules/
│   ├── .pnpm/                    # 実体はここ
│   │   ├── react@18.2.0/
│   │   └── lodash@4.17.21/
│   └── @mycompany/
│       ├── ui-components -> .pnpm/@mycompany+ui-components@1.0.0/
│       └── web -> .pnpm/@mycompany+web@1.0.0/
└── packages/
```

### ホイスティング設定

#### .npmrc（pnpm）

```ini
# .npmrc
# ホイスティングを有効化（デフォルト）
hoist=true

# パターンを指定してホイスト
public-hoist-pattern[]=*eslint*
public-hoist-pattern[]=*prettier*

# すべての依存関係をホイスト（非推奨）
shamefully-hoist=false
```

#### package.json（Yarn）

```json
{
  "workspaces": {
    "packages": ["packages/*", "apps/*"],
    "nohoist": [
      "**/react-native",
      "**/react-native/**"
    ]
  }
}
```

## npm scripts の共有と実行

ワークスペース全体でscriptsを効率的に管理する方法を解説します。

### ルートでのscripts定義

```json
// ルートのpackage.json
{
  "name": "my-monorepo",
  "scripts": {
    // すべてのワークスペースで実行
    "build": "npm run build --workspaces",
    "test": "npm run test --workspaces --if-present",
    "lint": "npm run lint --workspaces",

    // 並列実行
    "build:parallel": "npm-run-all --parallel build:*",
    "build:packages": "npm run build --workspace=packages/*",
    "build:apps": "npm run build --workspace=apps/*",

    // 順序指定実行
    "build:all": "npm run build:packages && npm run build:apps",

    // クリーンアップ
    "clean": "npm run clean --workspaces --if-present",
    "clean:deep": "rm -rf node_modules packages/*/node_modules apps/*/node_modules"
  }
}
```

### パッケージ個別のscripts

```json
// packages/ui-components/package.json
{
  "name": "@mycompany/ui-components",
  "scripts": {
    "build": "tsc --build",
    "dev": "tsc --watch",
    "test": "vitest run",
    "test:watch": "vitest watch",
    "lint": "eslint src --ext .ts,.tsx",
    "typecheck": "tsc --noEmit",
    "clean": "rm -rf dist"
  }
}
```

### pnpmでの実行

```bash
# すべてのパッケージで実行（再帰的）
pnpm -r run build
pnpm -r run test

# 並列実行
pnpm -r --parallel run build

# フィルタ指定
pnpm --filter @mycompany/ui-components build
pnpm --filter "./packages/**" test

# 依存関係を含めて実行
pnpm --filter @mycompany/web... build  # webとその依存関係
pnpm --filter ...@mycompany/utils test # utilsに依存するすべて

# 変更されたパッケージのみ
pnpm --filter "...[origin/main]" test
```

## タスクの並列実行と依存関係

### Turborepoによる最適化

Turborepoは、タスクの依存関係を解析し、最適な順序で並列実行します。

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"],
      "cache": true
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"],
      "cache": true
    },
    "lint": {
      "cache": true,
      "outputs": []
    },
    "typecheck": {
      "dependsOn": ["^build"],
      "cache": true,
      "outputs": []
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  },
  "globalDependencies": [
    ".eslintrc.json",
    "tsconfig.base.json",
    "turbo.json"
  ]
}
```

#### dependsOnの理解

- `^build`: 依存パッケージのbuildが完了してから実行
- `build`: 自パッケージ内でbuildが完了してから実行

```typescript
// 実行順序の例
// 1. packages/utils: build
// 2. packages/ui-components: build (utils依存)
// 3. apps/web: build (ui-components依存)
```

### npm-run-allでの並列実行

```json
{
  "scripts": {
    "lint": "npm-run-all --parallel lint:*",
    "lint:js": "eslint .",
    "lint:css": "stylelint '**/*.css'",
    "lint:types": "tsc --noEmit",

    "test": "npm-run-all --sequential clean test:*",
    "test:unit": "vitest run",
    "test:integration": "playwright test",

    "build": "npm-run-all clean --parallel build:*",
    "build:types": "tsc --emitDeclarationOnly",
    "build:js": "esbuild src/index.ts --bundle --outfile=dist/index.js"
  },
  "devDependencies": {
    "npm-run-all": "^4.1.5"
  }
}
```

## プライベートパッケージの管理

### プライベートパッケージの定義

公開したくないパッケージは`private: true`を設定します。

```json
// apps/web/package.json
{
  "name": "@mycompany/web",
  "version": "1.0.0",
  "private": true,
  "description": "Internal web application"
}
```

```json
// packages/shared-config/package.json
{
  "name": "@mycompany/shared-config",
  "version": "1.0.0",
  "private": true,
  "description": "Shared ESLint and TypeScript configs"
}
```

### publishConfigの設定

```json
// packages/ui-components/package.json（公開用）
{
  "name": "@mycompany/ui-components",
  "version": "1.0.0",
  "private": false,
  "publishConfig": {
    "access": "public",
    "registry": "https://registry.npmjs.org/"
  }
}
```

プライベートレジストリを使用する場合:

```json
{
  "publishConfig": {
    "access": "restricted",
    "registry": "https://npm.pkg.github.com/"
  }
}
```

## TypeScript設定の共有

### ベース設定の作成

```json
// tsconfig.base.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "incremental": true
  }
}
```

### パッケージごとの拡張

```json
// packages/ui-components/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "jsx": "react-jsx",
    "composite": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

```json
// apps/web/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "jsx": "react-jsx",
    "types": ["vite/client"]
  },
  "references": [
    { "path": "../../packages/ui-components" },
    { "path": "../../packages/utils" }
  ],
  "include": ["src/**/*"]
}
```

## ESLint/Prettier設定の共有

### 共有ESLint設定パッケージ

```json
// packages/eslint-config/package.json
{
  "name": "@mycompany/eslint-config",
  "version": "1.0.0",
  "private": true,
  "main": "index.js",
  "dependencies": {
    "eslint": "^8.56.0",
    "@typescript-eslint/eslint-plugin": "^6.19.0",
    "@typescript-eslint/parser": "^6.19.0",
    "eslint-config-prettier": "^9.1.0"
  }
}
```

```javascript
// packages/eslint-config/index.js
module.exports = {
  parser: '@typescript-eslint/parser',
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
    'prettier'
  ],
  rules: {
    '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
    '@typescript-eslint/no-explicit-any': 'warn'
  }
};
```

### パッケージでの使用

```json
// apps/web/.eslintrc.json
{
  "extends": "@mycompany/eslint-config",
  "env": {
    "browser": true,
    "es2022": true
  }
}
```

## ワークスペース設定のベストプラクティス

### チェックリスト

- [ ] workspace protocolを使用して内部依存を明示する
- [ ] ホイスティング設定を理解し、Phantom Dependenciesを防ぐ
- [ ] TypeScript Project Referencesを活用してビルド時間を短縮する
- [ ] 共通設定（ESLint、TypeScript等）をパッケージ化する
- [ ] プライベートパッケージには`private: true`を設定する
- [ ] Turborepo/Nxでタスク実行を最適化する
- [ ] scriptsの命名規則を統一する（例: build、test、lint、clean）
- [ ] package.jsonのexportsフィールドを適切に設定する

### 推奨設定パターン

#### 小規模プロジェクト（2-5パッケージ）

```json
// package.json
{
  "workspaces": ["packages/*"],
  "scripts": {
    "build": "npm run build --workspaces",
    "test": "npm run test --workspaces"
  }
}
```

#### 中規模プロジェクト（5-20パッケージ）

pnpm + npm-run-all

```json
{
  "scripts": {
    "build": "pnpm -r run build",
    "test": "pnpm -r run test",
    "lint": "npm-run-all --parallel lint:*"
  }
}
```

#### 大規模プロジェクト（20+パッケージ）

Turborepo + pnpm

```json
{
  "scripts": {
    "build": "turbo run build",
    "test": "turbo run test",
    "dev": "turbo run dev --parallel"
  }
}
```

## まとめ

ワークスペースの適切な設定により、モノレポの生産性が大幅に向上します。workspace protocol、ホイスティング、タスク並列実行などの概念を理解し、プロジェクト規模に応じた最適なツールを選択することが重要です。

主要なポイント:
- **依存関係管理**: workspace protocolで明示的に管理
- **ホイスティング**: pnpmで厳格に制御し、Phantom Dependenciesを防ぐ
- **タスク実行**: Turborepoで依存関係を解析し、並列実行を最適化
- **設定共有**: TypeScript、ESLintなどの設定を一元管理

次章では、スクリプト自動化について解説します。

## 参考文献

- [npm Workspaces Documentation](https://docs.npmjs.com/cli/v10/using-npm/workspaces)
- [pnpm Workspaces Documentation](https://pnpm.io/workspaces)
- [pnpm workspace protocol](https://pnpm.io/workspaces#workspace-protocol-workspace)
- [Turborepo Documentation](https://turbo.build/repo/docs)
- [TypeScript Project References](https://www.typescriptlang.org/docs/handbook/project-references.html)
- [npm-run-all Documentation](https://github.com/mysticatea/npm-run-all)
