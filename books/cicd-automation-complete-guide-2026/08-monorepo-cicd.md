---
title: "モノレポCI/CD戦略 - 効率的な大規模開発"
---

# Chapter 08 - モノレポCI/CD戦略

## モノレポとは

モノレポ(Monorepo)は、複数のプロジェクトやパッケージを単一のリポジトリで管理する開発手法です。適切なCI/CD戦略により、大規模開発の効率を飛躍的に向上させます。

### 想定される効果: モノレポCI/CD導入効果

あるSaaS企業(従業員80名、15個のサービス)での想定シナリオ:

**最適化前(マルチレポ):**
- CI実行時間: 各リポジトリ8-15分
- 共通ライブラリ更新: 15リポジトリへ手動PR作成
- ビルド重複: 各リポジトリで同じビルド実行
- バージョン不整合: 月3-5件の問題発生

**最適化後(モノレポ):**
- ✅ CI実行時間: 15分 → 5分(変更検出により)(-67%)
- ✅ 共通ライブラリ更新: 1PR で全体更新
- ✅ ビルド重複: キャッシュ共有で排除
- ✅ バージョン不整合: 0件
- ✅ コード共有: 重複コード40%削減

## モノレポ構成

### Turborepo による構成

```json
// package.json
{
  "name": "my-monorepo",
  "private": true,
  "workspaces": [
    "apps/*",
    "packages/*"
  ],
  "scripts": {
    "build": "turbo run build",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "dev": "turbo run dev"
  },
  "devDependencies": {
    "turbo": "^1.11.0"
  }
}
```

```javascript
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "dist/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"]
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

### プロジェクト構造

```
my-monorepo/
├── apps/
│   ├── web/                 # Next.jsアプリ
│   │   ├── package.json
│   │   └── src/
│   ├── mobile/              # React Nativeアプリ
│   │   ├── package.json
│   │   └── src/
│   └── admin/               # 管理画面
│       ├── package.json
│       └── src/
├── packages/
│   ├── ui/                  # 共通UIコンポーネント
│   │   ├── package.json
│   │   └── src/
│   ├── config/              # 共通設定
│   │   ├── package.json
│   │   ├── eslint-config/
│   │   └── tsconfig/
│   └── utils/               # ユーティリティ
│       ├── package.json
│       └── src/
├── package.json
└── turbo.json
```

## 変更検出とビルド最適化

### 変更されたパッケージの検出

```yaml
# .github/workflows/monorepo-ci.yml
name: Monorepo CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  # 変更検出
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      packages: ${{ steps.filter.outputs.changes }}
      has-changes: ${{ steps.filter.outputs.changes != '[]' }}
    steps:
      - uses: actions/checkout@v4

      - uses: dorny/paths-filter@v2
        id: filter
        with:
          filters: |
            web:
              - 'apps/web/**'
              - 'packages/**'
            mobile:
              - 'apps/mobile/**'
              - 'packages/**'
            admin:
              - 'apps/admin/**'
              - 'packages/**'
            ui:
              - 'packages/ui/**'
            utils:
              - 'packages/utils/**'

  # 変更されたパッケージのみビルド
  build-changed:
    needs: detect-changes
    if: needs.detect-changes.outputs.has-changes == 'true'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        package: ${{ fromJson(needs.detect-changes.outputs.packages) }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: 依存関係インストール
        run: npm ci

      - name: ビルド(${{ matrix.package }})
        run: npm run build --workspace=${{ matrix.package }}

      - name: テスト(${{ matrix.package }})
        run: npm run test --workspace=${{ matrix.package }}
```

**想定効果:**
- 全パッケージビルド: 30分
- 変更パッケージのみ(平均3個): 8分 (-73%)

### Turborepo によるキャッシュ活用

```yaml
# .github/workflows/turborepo-ci.yml
name: Turborepo CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: 依存関係インストール
        run: npm ci

      - name: Turbo キャッシュ
        uses: actions/cache@v4
        with:
          path: .turbo
          key: ${{ runner.os }}-turbo-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-turbo-

      - name: ビルド(変更検出 + キャッシュ)
        run: npm run build
        env:
          TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
          TURBO_TEAM: ${{ secrets.TURBO_TEAM }}

      - name: テスト(変更検出 + キャッシュ)
        run: npm run test
```

**想定される効果(15パッケージ規模):**

| 状況 | ビルド時間 | 削減率 |
|------|-----------|--------|
| 全パッケージ(キャッシュなし) | 30分 | - |
| 全パッケージ(キャッシュあり) | 8分 | -73% |
| 変更3パッケージ(キャッシュあり) | 3分 | -90% |
| 変更なし(キャッシュヒット) | 30秒 | -98% |

## 依存関係グラフの活用

### 依存関係に基づく実行順序

```yaml
# .github/workflows/dependency-graph.yml
name: Dependency-Aware Build

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      # Turboが依存関係を自動解決
      # utils → ui → web の順でビルド
      - name: 依存関係を考慮したビルド
        run: npx turbo run build --filter=web
        # web に必要な utils, ui も自動ビルド

      - name: 影響を受けるパッケージのみテスト
        run: npx turbo run test --filter=...utils
        # utils に依存する全パッケージをテスト
```

### Nx による高度な依存関係管理

```json
// nx.json
{
  "tasksRunnerOptions": {
    "default": {
      "runner": "nx/tasks-runners/default",
      "options": {
        "cacheableOperations": ["build", "test", "lint"],
        "parallel": 3
      }
    }
  },
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["{projectRoot}/dist"]
    },
    "test": {
      "dependsOn": ["build"]
    }
  }
}
```

```yaml
# .github/workflows/nx-ci.yml
name: Nx CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 変更検出に必要

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Nx キャッシュ設定
        uses: nrwl/nx-set-shas@v3

      - name: 影響を受けるプロジェクトのみビルド
        run: npx nx affected --target=build --parallel=3

      - name: 影響を受けるプロジェクトのみテスト
        run: npx nx affected --target=test --parallel=3
```

## パッケージごとのデプロイ戦略

### 個別デプロイワークフロー

```yaml
# .github/workflows/deploy-apps.yml
name: Deploy Apps

on:
  push:
    branches: [main]

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      web: ${{ steps.filter.outputs.web }}
      mobile: ${{ steps.filter.outputs.mobile }}
      admin: ${{ steps.filter.outputs.admin }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v2
        id: filter
        with:
          filters: |
            web:
              - 'apps/web/**'
              - 'packages/**'
            mobile:
              - 'apps/mobile/**'
              - 'packages/**'
            admin:
              - 'apps/admin/**'
              - 'packages/**'

  deploy-web:
    needs: detect-changes
    if: needs.detect-changes.outputs.web == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci

      - name: Webアプリビルド
        run: npm run build --workspace=@my-app/web

      - name: Vercelにデプロイ
        env:
          VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}
        run: |
          cd apps/web
          npx vercel --prod --token=$VERCEL_TOKEN

  deploy-admin:
    needs: detect-changes
    if: needs.detect-changes.outputs.admin == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci

      - name: 管理画面ビルド
        run: npm run build --workspace=@my-app/admin

      - name: Netlifyにデプロイ
        env:
          NETLIFY_AUTH_TOKEN: ${{ secrets.NETLIFY_AUTH_TOKEN }}
          NETLIFY_SITE_ID: ${{ secrets.NETLIFY_ADMIN_SITE_ID }}
        run: |
          cd apps/admin
          npx netlify deploy --prod
```

**想定効果:**
- 全アプリデプロイ: 15分
- 変更アプリのみ(平均1個): 5分 (-67%)

## バージョン管理

### Changesets によるバージョン管理

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Changesets リリース
        uses: changesets/action@v1
        with:
          publish: npm run release
          commit: 'chore: release packages'
          title: 'chore: release packages'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

**Changeset ファイル例:**

```markdown
---
"@my-app/ui": minor
"@my-app/web": patch
---

Add new Button component to UI package
```

### Lerna によるバージョン管理

```json
// lerna.json
{
  "version": "independent",
  "npmClient": "npm",
  "command": {
    "publish": {
      "conventionalCommits": true,
      "message": "chore(release): publish"
    }
  },
  "ignoreChanges": [
    "**/__tests__/**",
    "**/*.md"
  ]
}
```

```yaml
# .github/workflows/lerna-release.yml
name: Lerna Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Git 設定
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

      - name: バージョンアップとリリース
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: npx lerna publish --yes
```

## モノレポ最適化

### pnpm Workspace による高速化

```yaml
# pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

```yaml
# .github/workflows/pnpm-monorepo.yml
name: pnpm Monorepo

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v2
        with:
          version: 8

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'

      - name: 依存関係インストール
        run: pnpm install --frozen-lockfile

      - name: 全パッケージビルド
        run: pnpm -r build

      - name: 全パッケージテスト
        run: pnpm -r test
```

**想定比較(15パッケージ):**

| ツール | インストール時間 | ビルド時間 | ディスク使用量 |
|--------|---------------|-----------|-------------|
| npm | 280秒 | 30分 | 2.1GB |
| yarn | 180秒 | 28分 | 1.8GB |
| pnpm | 90秒 | 25分 | 850MB |

### ビルドキャッシュの共有

```yaml
# .github/workflows/shared-cache.yml
name: Shared Cache

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'

      - name: グローバルキャッシュ
        uses: actions/cache@v4
        with:
          path: |
            ~/.pnpm-store
            .turbo
            */*/node_modules
          key: ${{ runner.os }}-monorepo-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            ${{ runner.os }}-monorepo-

      - run: pnpm install
      - run: pnpm run build
```

## トラブルシューティング

### 問題1: 循環依存の検出

**症状:**
```
Error: Circular dependency detected
```

**対処法:**

```bash
# 循環依存の検出
npx madge --circular --extensions ts,tsx apps/ packages/

# 依存関係グラフの可視化
npx madge --image graph.png apps/ packages/
```

### 問題2: キャッシュが効きすぎる

**症状:**
```
変更したのにビルドがスキップされる
```

**対処法:**

```yaml
# Turboキャッシュをバイパス
- run: npx turbo run build --force

# または環境変数で無効化
- run: npm run build
  env:
    TURBO_FORCE: true
```

### 問題3: 依存関係の解決が遅い

**対処法:**

```json
// package.json
{
  "resolutions": {
    // バージョン固定で解決を高速化
    "react": "18.2.0",
    "react-dom": "18.2.0"
  }
}
```

## まとめ

この章では、モノレポ環境での効率的なCI/CD戦略を学びました:

✅ **変更検出**: 影響を受けるパッケージのみビルド・テスト
✅ **キャッシュ活用**: Turborepo/Nxによる高速化
✅ **依存関係管理**: グラフベースのビルド順序最適化
✅ **個別デプロイ**: パッケージごとの独立したデプロイ
✅ **バージョン管理**: Changesets/Lernaによる自動化

### 重要な想定される効果まとめ

| 施策 | 効果 |
|------|------|
| 変更検出 | -73% (30分→8分) |
| Turboキャッシュ | -90% (30分→3分) |
| pnpm 採用 | -68% (280秒→90秒) |
| 個別デプロイ | -67% (15分→5分) |

### モノレポ vs マルチレポ

| 項目 | モノレポ | マルチレポ |
|------|---------|-----------|
| コード共有 | ✅ 容易 | ❌ 困難 |
| バージョン管理 | ✅ 統一 | ❌ 分散 |
| CI/CD時間 | ⚠️ 最適化必要 | ✅ 独立 |
| スケーラビリティ | ⚠️ 工夫必要 | ✅ 高い |

### 次のステップ

**Chapter 09 - Docker統合とコンテナビルド**では、Dockerイメージの最適化、マルチステージビルド、コンテナレジストリ連携を学びます。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
