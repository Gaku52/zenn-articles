---
title: "モノレポ依存関係管理"
---

# モノレポ依存関係管理

## モノレポとは

モノレポ（Monorepo）は、複数のプロジェクトやパッケージを1つのリポジトリで管理する手法です。Google、Facebook、Microsoft、Uberなどの大企業で広く採用されており、大規模開発における効率性と一貫性を実現します。

### モノレポの利点

モノレポは以下のような利点を提供します。

#### コード共有の容易性

複数のプロジェクト間で共通コードを簡単に共有できます。UIコンポーネント、ユーティリティ関数、型定義などを中央集約し、重複を排除できます。

```typescript
// packages/utils/src/format.ts
export function formatCurrency(amount: number): string {
  return new Intl.NumberFormat('ja-JP', {
    style: 'currency',
    currency: 'JPY'
  }).format(amount);
}

// apps/web/src/components/Price.tsx
import { formatCurrency } from '@mycompany/utils';
```

#### アトミックなコミット

複数パッケージに跨る変更を1つのコミットで行えます。APIの変更とそれを使用するクライアントコードを同時に更新できるため、バージョン不整合を防げます。

```bash
# 1つのコミットで複数パッケージを更新
git add packages/api/src/schema.ts
git add packages/client/src/api.ts
git add apps/web/src/features/user.tsx
git commit -m "feat: add user profile endpoint"
```

#### 統一されたツール設定

ESLint、TypeScript、Prettier、テストフレームワークなどの設定を一元管理できます。これにより、プロジェクト間の一貫性が保たれ、開発者の認知負荷が軽減されます。

```json
// ルートのtsconfig.base.json
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "skipLibCheck": true
  }
}
```

#### 依存関係の一元管理

すべての依存関係をルートで管理することで、バージョン重複を防ぎ、セキュリティアップデートを一括適用できます。

### モノレポの課題

モノレポには以下のような課題も存在します。

#### ビルド時間の増加

プロジェクト規模が大きくなると、ビルド時間が増加します。これに対しては、Turborepo、Nxなどのビルドキャッシュツールで対応可能です。

#### 依存関係の複雑化

パッケージ間の依存関係が複雑になり、循環依存が発生する可能性があります。適切なアーキテクチャ設計と依存関係の可視化が重要です。

#### スケーラビリティ

チーム数やコード量が増えると、Gitの操作が遅くなる可能性があります。GitのPartial Clone、Sparse Checkoutなどの機能を活用することで緩和できます。

## npm Workspaces

npm 7.0以降で標準サポートされたモノレポ機能です。

### 基本構成

```json
{
  "name": "my-monorepo",
  "version": "1.0.0",
  "private": true,
  "workspaces": [
    "packages/*",
    "apps/*"
  ],
  "scripts": {
    "build": "npm run build --workspaces",
    "test": "npm run test --workspaces --if-present",
    "lint": "npm run lint --workspaces"
  },
  "devDependencies": {
    "typescript": "^5.3.3",
    "eslint": "^8.56.0",
    "vitest": "^1.2.0"
  }
}
```

ディレクトリ構成例:

```
my-monorepo/
├── package.json
├── package-lock.json
├── tsconfig.base.json
├── .eslintrc.json
├── packages/
│   ├── ui-components/
│   │   ├── package.json
│   │   ├── src/
│   │   │   ├── Button.tsx
│   │   │   └── Input.tsx
│   │   └── tsconfig.json
│   ├── utils/
│   │   ├── package.json
│   │   ├── src/
│   │   │   ├── format.ts
│   │   │   └── validate.ts
│   │   └── tsconfig.json
│   └── api-client/
│       ├── package.json
│       └── src/
│           └── client.ts
└── apps/
    ├── web/
    │   ├── package.json
    │   ├── src/
    │   │   └── main.tsx
    │   └── vite.config.ts
    └── mobile/
        ├── package.json
        └── src/
            └── App.tsx
```

### パッケージ定義例

```json
// packages/ui-components/package.json
{
  "name": "@mycompany/ui-components",
  "version": "1.0.0",
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "types": "./dist/index.d.ts"
    }
  },
  "scripts": {
    "build": "tsc",
    "test": "vitest run",
    "lint": "eslint src"
  },
  "dependencies": {
    "react": "^18.2.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.48",
    "typescript": "^5.3.3"
  }
}
```

```json
// apps/web/package.json
{
  "name": "@mycompany/web",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "@mycompany/ui-components": "*",
    "@mycompany/utils": "*",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.2.1",
    "vite": "^5.0.11"
  }
}
```

### コマンド

```bash
# すべてのワークスペースで実行
npm run build --workspaces
npm test --workspaces

# 特定のワークスペースで実行
npm run build --workspace=@mycompany/ui-components
npm test --workspace=apps/web

# 特定のワークスペースに依存関係を追加
npm install lodash --workspace=packages/utils

# すべての依存関係をインストール
npm install

# 依存関係を更新
npm update --workspaces

# 特定のワークスペースの情報を表示
npm list --workspace=@mycompany/ui-components

# すべてのワークスペースを一覧表示
npm list --workspaces --depth=0
```

## pnpm Workspaces

pnpmは高速で効率的なパッケージマネージャーで、ディスク容量を節約できます。

### pnpm-workspace.yaml設定

```yaml
packages:
  # すべてのpackages配下
  - 'packages/*'
  # すべてのapps配下
  - 'apps/*'
  # ドキュメント
  - 'docs'
  # テストフォルダは除外
  - '!**/test/**'
  - '!**/__tests__/**'
```

### ルートpackage.json

```json
{
  "name": "my-monorepo",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "build": "pnpm -r run build",
    "test": "pnpm -r run test",
    "lint": "pnpm -r run lint",
    "clean": "pnpm -r run clean",
    "typecheck": "pnpm -r run typecheck"
  },
  "devDependencies": {
    "typescript": "^5.3.3",
    "eslint": "^8.56.0",
    "prettier": "^3.1.1"
  }
}
```

### workspace protocolの使用

```json
// apps/web/package.json
{
  "name": "@mycompany/web",
  "dependencies": {
    "@mycompany/ui-components": "workspace:*",
    "@mycompany/utils": "workspace:^",
    "@mycompany/api-client": "workspace:~1.0.0"
  }
}
```

workspace protocolのバリエーション:
- `workspace:*`: 常に最新のローカルバージョン
- `workspace:^`: セマンティックバージョニングに基づく互換バージョン
- `workspace:~`: パッチバージョンのみ更新

### pnpmコマンド

```bash
# すべてのパッケージで実行
pnpm -r run build
pnpm -r run test

# フィルタを使用
pnpm --filter @mycompany/ui-components build
pnpm --filter "@mycompany/*" test
pnpm --filter "./packages/**" lint

# 依存グラフに基づいて実行（並列実行）
pnpm -r --parallel run build

# 変更されたパッケージのみテスト
pnpm --filter "...[origin/main]" test

# 特定のパッケージとその依存関係をビルド
pnpm --filter @mycompany/web... build

# 特定のパッケージに依存するすべてをテスト
pnpm --filter ...@mycompany/utils test

# 依存関係を追加
pnpm add lodash --filter @mycompany/utils

# 開発環境で実行
pnpm --filter @mycompany/web dev
```

## Lerna

Lernaはモノレポ管理ツールの先駆けです。現在はNxと統合されています。

### Lerna設定

```json
// lerna.json
{
  "$schema": "node_modules/lerna/schemas/lerna-schema.json",
  "version": "independent",
  "npmClient": "pnpm",
  "useWorkspaces": true,
  "packages": [
    "packages/*",
    "apps/*"
  ],
  "command": {
    "publish": {
      "conventionalCommits": true,
      "message": "chore(release): publish %s"
    },
    "version": {
      "allowBranch": "main",
      "message": "chore(release): version %s"
    }
  }
}
```

### Lernaコマンド

```bash
# すべてのパッケージで実行
lerna run build
lerna run test

# 変更されたパッケージのみ
lerna run build --since origin/main

# 並列実行
lerna run build --parallel

# バージョン管理
lerna version --conventional-commits

# 公開
lerna publish from-git
```

## Turborepo

Turborepoは高速なビルドシステムで、キャッシュと並列実行を最適化します。

### Turborepo設定

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "build/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"],
      "cache": false
    },
    "lint": {
      "outputs": []
    },
    "typecheck": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  },
  "globalDependencies": [
    ".eslintrc.json",
    "tsconfig.base.json"
  ]
}
```

### Turboコマンド

```bash
# ビルド（依存関係順に実行）
turbo run build

# テスト（並列実行）
turbo run test

# 複数タスクを実行
turbo run build test lint

# キャッシュをクリア
turbo run build --force

# リモートキャッシュを使用
turbo run build --token=<your-token>
```

## モノレポ管理のベストプラクティス

### チェックリスト

- [ ] パッケージ間の依存関係を明確に定義する
- [ ] ビルド順序を最適化する（Turborepo、Nx等を活用）
- [ ] 共通の設定ファイル（ESLint、TypeScript等）をルートで管理する
- [ ] ワークスペースプロトコルを使用して内部依存を明示する
- [ ] CIでキャッシュを活用してビルド時間を短縮する
- [ ] パッケージのバージョニング戦略を決定する（固定 or independent）
- [ ] 循環依存を防ぐためのリンティングルールを設定する
- [ ] 変更されたパッケージのみをテスト・ビルドする仕組みを導入する

### 推奨事項

1. **小さく始める**: 最初から完璧なモノレポを目指さず、段階的に移行する
2. **ビルドキャッシュを活用**: Turborepo、Nxなどでビルド時間を最適化
3. **依存関係を可視化**: `pnpm why`、`npm explain`などで依存関係を理解する
4. **共通コードを抽出**: 重複コードを見つけて共通パッケージに移動
5. **CI/CDを最適化**: 変更されたパッケージのみをビルド・テストする

## まとめ

モノレポは、複数のプロジェクトを効率的に管理するための強力な手法です。npm Workspaces、pnpm、Lerna、Turborepoなど、目的に応じた適切なツールを選択することで、開発効率を大幅に向上させることができます。

ツール選択の推奨基準:
- **小規模プロジェクト**: npm Workspaces（追加ツール不要）
- **中規模プロジェクト**: pnpm Workspaces（高速・効率的）
- **大規模プロジェクト**: Turborepo + pnpm（ビルドキャッシュ・並列実行）

次章では、ワークスペース設定の詳細を解説します。

## 参考文献

- [npm Workspaces Documentation](https://docs.npmjs.com/cli/v10/using-npm/workspaces)
- [pnpm Workspace Documentation](https://pnpm.io/workspaces)
- [Lerna Documentation](https://lerna.js.org/)
- [Turborepo Documentation](https://turbo.build/repo/docs)
- [Monorepo.tools - Monorepo Explained](https://monorepo.tools/)
