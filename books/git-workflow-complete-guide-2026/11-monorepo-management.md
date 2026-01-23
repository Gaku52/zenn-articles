---
title: "モノレポ管理戦略 - 複数プロジェクトの統合運用"
---

# モノレポ管理戦略

## モノレポとは

Monorepo（モノレポ）は、複数のプロジェクト・パッケージを1つのGitリポジトリで管理する手法です。Google、Facebook、Microsoftなどの大企業で採用されています。

**想定される効果（想定プロジェクト: 15パッケージ程度）:**
- コード共有効率: **+350%**（重複コード削減）
- デプロイ時間: **45分 → 12分** (-73%)
- 依存関係の不整合: **月8件 → 0件** (-100%)
- リファクタリング効率: **+280%**（一括変更可能）

**マルチレポとの比較:**
```
マルチレポ（15個のリポジトリ）:
  依存関係更新: 15箇所 × 30分 = 7.5時間
  横断的変更: 15箇所 × 1時間 = 15時間
  バージョン管理: 複雑

モノレポ（1個のリポジトリ）:
  依存関係更新: 1箇所 × 30分 = 30分 (-93%)
  横断的変更: 1箇所 × 2時間 = 2時間 (-87%)
  バージョン管理: 統一的
```

## モノレポのメリット・デメリット

### メリット

**1. コード共有の容易さ**
```
packages/
├── ui/          # UIコンポーネント
├── api/         # APIクライアント
├── utils/       # ユーティリティ
├── web/         # Webアプリ（ui, api, utils使用）
└── mobile/      # モバイルアプリ（ui, api, utils使用）

→ web, mobile が ui, api, utils を即座にimport可能
```

**2. 原子的なコミット（Atomic Commits）**
```bash
# 1つのコミットで複数パッケージを変更
git commit -m "feat(api): change user endpoint format

- Update @myapp/api response format
- Update @myapp/web to use new format
- Update @myapp/mobile to use new format
- Update @myapp/ui types

BREAKING CHANGE: API response structure changed"
```

**3. 依存関係の統一管理**
```json
{
  "workspaces": ["packages/*"],
  "devDependencies": {
    "typescript": "5.3.0",
    "eslint": "8.56.0",
    "prettier": "3.1.0"
  }
}

→ 全パッケージで同じバージョン使用
```

**4. 効率的なリファクタリング**
```bash
# API変更時、全ての使用箇所を一度に修正
# packages/api/src/user.ts
export interface User {
  // name: string;  削除
  firstName: string;  // 追加
  lastName: string;   // 追加
}

# packages/web/src/UserProfile.tsx
# packages/mobile/src/UserProfile.tsx
# → 同じPRで修正
```

### デメリット

**1. リポジトリサイズの増大**
```
想定例（15パッケージ、2年運用規模）:
  リポジトリサイズ: 2.3GB
  .git ディレクトリ: 1.1GB
  clone時間: 8分

対策: Git LFS、shallow clone
```

**2. CI/CD時間の増加**
```
全パッケージビルド: 45分

対策: 変更検出による選択的ビルド
```

**3. 権限管理の複雑さ**
```
問題: チームAはpackages/web、チームBはpackages/mobile担当
     でも両方とも全コードにアクセス可能

対策: CODEOWNERS、ブランチ保護、PRテンプレート
```

## モノレポツール比較

### 主要ツール

| ツール | タスク実行 | キャッシュ | リモートキャッシュ | 学習コスト | 人気度 |
|--------|----------|----------|----------------|----------|--------|
| **Turborepo** | ⭐⭐⭐ | ✅ | ✅ | 低 | 高 |
| **Nx** | ⭐⭐⭐ | ✅ | ✅ | 中 | 高 |
| **Lerna** | ⭐⭐ | ❌ | ❌ | 低 | 中（レガシー） |
| **pnpm workspace** | ⭐⭐ | ❌ | ❌ | 低 | 中 |
| **Yarn workspace** | ⭐⭐ | ❌ | ❌ | 低 | 中 |

### Turborepo（推奨）

**特徴:**
- 高速なタスク実行
- 優れたキャッシュ機構
- シンプルな設定

**セットアップ:**
```bash
# インストール
npx create-turbo@latest

# 構成
my-monorepo/
├── apps/
│   ├── web/          # Next.js
│   └── mobile/       # React Native
├── packages/
│   ├── ui/           # React components
│   ├── utils/        # Utilities
│   └── tsconfig/     # Shared TS config
├── turbo.json
└── package.json
```

**turbo.json:**
```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": [],
      "cache": true
    },
    "lint": {
      "outputs": []
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

**実行:**
```bash
# 全パッケージビルド（並列、キャッシュ活用）
turbo build

# 変更されたパッケージのみテスト
turbo test --filter=...[origin/main]

# 特定パッケージのみ実行
turbo build --filter=@myapp/web

# 開発モード（全パッケージwatch）
turbo dev
```

**想定効果:**
- ビルド時間: **45分 → 12分** (-73%)（キャッシュ活用）
- 2回目以降: **12分 → 30秒** (-96%)（フルキャッシュ）

### Nx

**特徴:**
- より高機能（コード生成、マイグレーション等）
- 大規模プロジェクト向け
- Angular発祥だがReact等も対応

**セットアップ:**
```bash
npx create-nx-workspace@latest

# プロジェクト追加
nx generate @nrwl/react:app web
nx generate @nrwl/react:lib ui
```

**実行:**
```bash
# ビルド
nx build web

# 影響を受けるプロジェクトのみテスト
nx affected:test --base=main

# 依存関係グラフ表示
nx graph
```

## モノレポのディレクトリ構成

### パターン1: アプリケーション+ライブラリ

```
monorepo/
├── apps/                    # アプリケーション
│   ├── web/                 # Webアプリ
│   │   ├── src/
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── mobile/              # モバイルアプリ
│   │   ├── ios/
│   │   ├── android/
│   │   ├── src/
│   │   └── package.json
│   └── admin/               # 管理画面
│       ├── src/
│       └── package.json
├── packages/                # 共有ライブラリ
│   ├── ui/                  # UIコンポーネント
│   │   ├── src/
│   │   │   ├── Button/
│   │   │   ├── Input/
│   │   │   └── index.ts
│   │   └── package.json
│   ├── api/                 # APIクライアント
│   │   ├── src/
│   │   │   ├── user.ts
│   │   │   ├── auth.ts
│   │   │   └── index.ts
│   │   └── package.json
│   ├── utils/               # ユーティリティ
│   │   ├── src/
│   │   └── package.json
│   └── config/              # 共有設定
│       ├── eslint/
│       ├── typescript/
│       └── prettier/
├── turbo.json
├── package.json
└── pnpm-workspace.yaml
```

### パターン2: マイクロサービス

```
monorepo/
├── services/                # バックエンドサービス
│   ├── user-service/
│   │   ├── src/
│   │   ├── Dockerfile
│   │   └── package.json
│   ├── payment-service/
│   │   ├── src/
│   │   ├── Dockerfile
│   │   └── package.json
│   └── notification-service/
│       ├── src/
│       └── package.json
├── packages/                # 共有パッケージ
│   ├── shared-types/        # 共通型定義
│   ├── database/            # DB接続・モデル
│   └── logger/              # ロガー
└── infrastructure/          # インフラコード
    ├── terraform/
    └── k8s/
```

### パターン3: フルスタック

```
monorepo/
├── apps/
│   ├── frontend/            # Next.js
│   └── backend/             # NestJS
├── packages/
│   ├── shared-types/        # フロント・バック共通型
│   ├── ui/                  # UIコンポーネント
│   └── api-client/          # API型定義
└── database/
    ├── migrations/
    └── seeds/
```

## モノレポでのブランチ戦略

### パターン1: 機能別ブランチ

```bash
# 特定パッケージのみ変更
git checkout -b feature/ui-button-component
# → packages/ui のみ変更

# 複数パッケージに跨る変更
git checkout -b feature/user-api-update
# → packages/api, apps/web, apps/mobile 変更
```

### パターン2: パッケージプレフィックス

```bash
# ブランチ名にパッケージ名を含める
git checkout -b packages/ui/add-button
git checkout -b apps/web/integrate-new-api
git checkout -b services/user/add-endpoint
```

### パターン3: スコープ付きコミット

```bash
# 単一パッケージ
git commit -m "feat(ui): add Button component"

# 複数パッケージ
git commit -m "feat(api,web,mobile): update user endpoint

- packages/api: Change response format
- apps/web: Update UserProfile component
- apps/mobile: Update UserScreen

BREAKING CHANGE: User API response structure changed"

# モノレポ全体
git commit -m "chore(monorepo): update dependencies"
```

## CI/CDでの選択的実行

### 変更検出によるビルド最適化

**問題:**
```
packages/ui を変更
→ 全パッケージ（15個）ビルド: 45分
→ 非効率
```

**解決策:**
```yaml
# .github/workflows/ci.yml
name: CI

on: [pull_request]

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      ui: ${{ steps.filter.outputs.ui }}
      api: ${{ steps.filter.outputs.api }}
      web: ${{ steps.filter.outputs.web }}
    steps:
      - uses: actions/checkout@v3

      - uses: dorny/paths-filter@v2
        id: filter
        with:
          filters: |
            ui:
              - 'packages/ui/**'
            api:
              - 'packages/api/**'
            web:
              - 'apps/web/**'
              - 'packages/ui/**'
              - 'packages/api/**'

  test-ui:
    needs: detect-changes
    if: needs.detect-changes.outputs.ui == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: turbo test --filter=@myapp/ui

  test-web:
    needs: detect-changes
    if: needs.detect-changes.outputs.web == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: turbo test --filter=@myapp/web
```

**想定効果:**
- UI変更時のCI時間: **45分 → 8分** (-82%)
- 平均CI時間: **45分 → 15分** (-67%)

### Turborepoによる最適化

```yaml
# .github/workflows/ci.yml
name: CI

on: [pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 20

      - name: Install dependencies
        run: npm install

      # 変更されたパッケージのみビルド
      - name: Build
        run: turbo build --filter=...[origin/main]

      # 変更されたパッケージのみテスト
      - name: Test
        run: turbo test --filter=...[origin/main]

      # キャッシュ保存
      - name: Cache Turbo
        uses: actions/cache@v3
        with:
          path: .turbo
          key: ${{ runner.os }}-turbo-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-turbo-
```

## 依存関係管理

### パッケージ間依存

**package.json（apps/web）:**
```json
{
  "name": "@myapp/web",
  "dependencies": {
    "@myapp/ui": "workspace:*",
    "@myapp/api": "workspace:*",
    "@myapp/utils": "workspace:*",
    "react": "^18.2.0",
    "next": "^14.0.0"
  }
}
```

**package.json（packages/ui）:**
```json
{
  "name": "@myapp/ui",
  "peerDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  },
  "devDependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  }
}
```

### バージョン統一

**ルートpackage.json:**
```json
{
  "name": "monorepo",
  "private": true,
  "workspaces": ["apps/*", "packages/*"],
  "devDependencies": {
    "typescript": "5.3.0",
    "eslint": "8.56.0",
    "prettier": "3.1.0",
    "@types/react": "18.2.0",
    "@types/node": "20.10.0"
  },
  "scripts": {
    "build": "turbo build",
    "test": "turbo test",
    "lint": "turbo lint",
    "dev": "turbo dev"
  }
}
```

### 依存関係更新戦略

```bash
# 全パッケージの依存を一括更新
npm install typescript@latest -D -w

# 特定パッケージのみ
npm install react@latest -w @myapp/web

# 依存関係チェック
npm ls

# 重複パッケージ検出
npx find-duplicate-dependencies
```

## モノレポでのリリース戦略

### パターン1: 独立バージョニング

各パッケージが独自のバージョンを持つ。

```json
// packages/ui/package.json
{
  "name": "@myapp/ui",
  "version": "1.2.0"
}

// packages/api/package.json
{
  "name": "@myapp/api",
  "version": "2.0.1"
}
```

**リリース:**
```bash
# 変更されたパッケージのみバージョンアップ
npx changeset version

# 変更されたパッケージのみ公開
npx changeset publish
```

**想定される効果:**
- リリース頻度: **週3回**（パッケージ独立）
- リリース時間: **5分/パッケージ**

### パターン2: 固定バージョニング

全パッケージが同じバージョン。

```json
// lerna.json
{
  "version": "1.2.0",
  "packages": ["apps/*", "packages/*"]
}
```

**リリース:**
```bash
# 全パッケージを同じバージョンでリリース
lerna version --conventional-commits
lerna publish
```

**想定される効果:**
- リリース頻度: **月1回**（全体同期）
- リリース時間: **15分**

### パターン3: アプリケーション単位

アプリケーションのみバージョン管理。

```bash
# apps/web のみデプロイ
turbo build --filter=@myapp/web
vercel deploy

# apps/mobile のみリリース
turbo build --filter=@myapp/mobile
fastlane ios release
```

## トラブルシューティング

### 問題1: node_modulesの肥大化

**症状:**
```
node_modules サイズ: 2.5GB
npm install 時間: 8分
```

**対策:**
```bash
# pnpm使用（シンボリックリンクで共有）
pnpm install

# 結果:
# node_modules サイズ: 800MB (-68%)
# pnpm install 時間: 2分 (-75%)
```

### 問題2: ビルドが遅い

**対策1: Turborepoキャッシュ**
```bash
# ローカルキャッシュ
turbo build
# 2回目: 30秒（ほぼ瞬時）

# リモートキャッシュ（Vercel）
turbo build --token=$TURBO_TOKEN
# チーム全体でキャッシュ共有
```

**対策2: 並列実行最適化**
```json
{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    }
  }
}
```

### 問題3: コンフリクトが複雑

**対策:**
```bash
# パッケージごとにブランチ分離
git checkout -b packages/ui/add-button
# → packages/ui のみ変更

git checkout -b packages/api/add-endpoint
# → packages/api のみ変更

# 後でマージ
git checkout main
git merge packages/ui/add-button
git merge packages/api/add-endpoint
```

### 問題4: CIが全パッケージテスト

**対策:**
```bash
# 変更検出（Turborepo）
turbo test --filter=...[origin/main]

# 変更検出（Nx）
nx affected:test --base=main
```

## ベストプラクティス

### 1. CODEOWNERS設定

```
# .github/CODEOWNERS

# UI team
/packages/ui/                @ui-team
/apps/web/src/components/    @ui-team

# Backend team
/packages/api/               @backend-team
/services/                   @backend-team

# Mobile team
/apps/mobile/                @mobile-team

# DevOps team
/.github/workflows/          @devops-team
/infrastructure/             @devops-team
```

### 2. パッケージREADME

各パッケージにREADMEを作成。

```markdown
# @myapp/ui

UIコンポーネントライブラリ

## インストール

この パッケージはmonorepo内部用です。

## 使用方法

```typescript
import { Button } from '@myapp/ui';

<Button variant="primary">Click me</Button>
```

## 開発

```bash
# 開発サーバー
turbo dev --filter=@myapp/ui

# テスト
turbo test --filter=@myapp/ui
```
```

### 3. 共通設定の集約

```
packages/
├── config-eslint/
│   └── index.js         # 共通ESLint設定
├── config-typescript/
│   └── tsconfig.json    # 共通TypeScript設定
└── config-prettier/
    └── index.js         # 共通Prettier設定
```

**使用:**
```json
// apps/web/tsconfig.json
{
  "extends": "@myapp/config-typescript/tsconfig.json",
  "compilerOptions": {
    "outDir": "dist"
  }
}
```

## まとめ

### モノレポ vs マルチレポ選択基準

**モノレポが適している:**
- ✅ コードを頻繁に共有
- ✅ 同じチームが複数プロジェクト管理
- ✅ 横断的な変更が多い
- ✅ 依存関係が密接
- ✅ 統一的なツールチェーン

**マルチレポが適している:**
- ✅ プロジェクトが完全独立
- ✅ チームが地理的/組織的に分離
- ✅ 異なるリリースサイクル
- ✅ 異なる技術スタック
- ✅ 権限分離が重要

### 想定効果（まとめ）

| 項目 | 改善率 | 具体的な数値 |
|------|--------|------------|
| コード共有効率向上 | +350% | 重複コード削減 |
| ビルド時間短縮 | -73% | 45分 → 12分 |
| 依存関係不整合削減 | -100% | 月8件 → 0件 |
| リファクタリング効率向上 | +280% | 一括変更可能 |
| CI時間短縮 | -67% | 45分 → 15分 |
| node_modules削減 | -68% | 2.5GB → 800MB |

### 推奨ツールチェーン

```
パッケージマネージャー: pnpm
タスクランナー: Turborepo
バージョニング: Changesets
CI/CD: GitHub Actions + Turborepo Cache
```

モノレポは、複数プロジェクトの管理を効率化する強力な手法です。適切なツールとワークフローを組み合わせることで、開発生産性を大幅に向上できます。

次の章では、**実戦ケーススタディ集**として、想定シナリオでのGit運用事例を詳しく学びます。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
