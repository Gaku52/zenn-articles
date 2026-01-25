---
title: "パフォーマンス最適化"
---

# パフォーマンス最適化

## バンドルサイズ最適化

### 依存関係の軽量化

```javascript
// ❌ 重いライブラリ（moment.js: 530KB）
import moment from 'moment';

// ✅ 軽量な代替（date-fns: 13KB）
import { format } from 'date-fns';

// 🏆 ネイティブAPI（0KB）
new Intl.DateTimeFormat('ja-JP').format(new Date());
```

### Tree Shaking

```javascript
// ❌ 全体をインポート
import _ from 'lodash';

// ✅ 必要な関数のみ
import { debounce } from 'lodash-es';
```

### Code Splitting

```javascript
// React.lazy
const AdminPanel = React.lazy(() => import('./AdminPanel'));

<Suspense fallback={<Loading />}>
  <AdminPanel />
</Suspense>
```

## インストール速度の最適化

### pnpmの使用

```bash
# npmからpnpmへ移行
# インストール速度: 3〜8倍高速化
# ディスク使用量: 60〜70%削減
```

### CI/CDキャッシュ

```yaml
# GitHub Actions
- uses: actions/setup-node@v4
  with:
    node-version: '18'
    cache: 'npm'  # npmキャッシュを有効化
```

## ビルド時間の最適化

### 並列ビルド

```json
{
  "scripts": {
    "build": "npm-run-all --parallel build:*",
    "build:js": "webpack",
    "build:css": "sass",
    "build:assets": "imagemin"
  }
}
```

### Turbopack / Turborepo

```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"],
      "cache": true
    }
  }
}
```

## 依存関係の最適化

### 重複パッケージの削減

```bash
# 重複を検出
npm dedupe

# pnpmは自動的に重複排除
pnpm install
```

### 不要な依存関係の削除

```bash
# 未使用パッケージを検出
npx depcheck

# 削除
npm uninstall unused-package
```

## モニタリング

### Bundle Analyzer

```bash
# webpack-bundle-analyzer
npm install --save-dev webpack-bundle-analyzer

# 分析レポート生成
npm run build
open dist/bundle-report.html
```

### CI/CDでのサイズ監視

```yaml
- name: Check bundle size
  uses: andresz1/size-limit-action@v1
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
```

## インストール速度の詳細最適化

### パッケージマネージャー別の性能比較

以下は、[pnpm公式ベンチマーク](https://pnpm.io/benchmarks)を参考にした、一般的な性能特性です（環境により変動します）。

| 指標 | npm | Yarn Classic | Yarn PnP | pnpm |
|-----|-----|-------------|----------|------|
| 初回インストール | 基準 | 若干高速 | 高速 | 最速 |
| キャッシュヒット時 | 基準 | 2〜3倍高速 | 5〜10倍高速 | 3〜5倍高速 |
| ディスク使用量 | 基準 | 同等 | 大幅削減 | 60〜70%削減 |
| node_modules生成 | あり | あり | なし | シンボリックリンク |

### pnpm移行による高速化

```bash
# npmからpnpmへの移行手順
# 1. pnpmをインストール
npm install -g pnpm

# 2. 既存のnode_modulesを削除
rm -rf node_modules package-lock.json

# 3. pnpmでインストール
pnpm install

# 4. パフォーマンスを計測
time pnpm install  # 2回目以降は大幅に高速化
```

期待される効果:
- 初回インストール: npmと比較して期待される速度向上は1.5〜2倍
- 2回目以降: 3〜5倍の高速化
- ディスク使用量: 60〜70%削減

### CI/CD環境での最適化

```yaml
# GitHub Actions - 最適化版
name: Optimized CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      # pnpmのセットアップ
      - uses: pnpm/action-setup@v2
        with:
          version: 8

      # Node.jsのセットアップ（キャッシュ有効化）
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'

      # 依存関係のキャッシュ（追加）
      - name: Cache pnpm store
        uses: actions/cache@v3
        with:
          path: ~/.pnpm-store
          key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            ${{ runner.os }}-pnpm-

      # frozen-lockfileで高速化
      - run: pnpm install --frozen-lockfile --prefer-offline

      - run: pnpm build
      - run: pnpm test
```

期待される効果:
- キャッシュヒット時: 期待されるインストール時間の短縮は70〜90%
- 並列ビルド: 複数ジョブの同時実行で期待される全体時間の短縮は40〜60%

### ローカル開発環境の最適化

```bash
# .npmrc設定（pnpm）
# プロジェクトルートに配置
store-dir=~/.pnpm-store
package-import-method=hardlink
symlink=true
prefer-offline=true
auto-install-peers=false
strict-peer-dependencies=true

# 期待される効果:
# - ハードリンクによるディスク使用量削減
# - オフラインモードでの高速化
# - 厳密な依存関係管理
```

## ビルド時間の詳細最適化

### Webpack最適化設定

```javascript
// webpack.config.js
const webpack = require('webpack');

module.exports = {
  mode: 'production',

  // ソースマップの最適化（ビルド時間短縮）
  devtool: 'source-map',  // 本番: source-map、開発: eval-source-map

  // キャッシュ有効化（Webpack 5以降）
  cache: {
    type: 'filesystem',
    buildDependencies: {
      config: [__filename],
    },
  },

  // 並列処理
  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        parallel: true,  // CPUコア数に応じた並列処理
        terserOptions: {
          compress: {
            drop_console: true,  // console.logを削除
          },
        },
      }),
    ],

    // コード分割
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 10,
        },
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true,
        },
      },
    },
  },

  // 解決速度の最適化
  resolve: {
    extensions: ['.js', '.jsx', '.ts', '.tsx'],
    modules: ['node_modules'],
    // シンボリックリンクの解決を無効化（高速化）
    symlinks: false,
  },
};
```

期待される効果:
- キャッシュ有効化: 期待される2回目以降のビルド時間短縮は60〜80%
- 並列処理: 期待されるビルド時間短縮は30〜40%

### Vite高速化設定

```javascript
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],

  // 依存関係の事前バンドル
  optimizeDeps: {
    include: ['react', 'react-dom'],  // 頻繁に使用する依存関係
    exclude: ['@my-local-package'],   // ローカルパッケージは除外
  },

  build: {
    // ターゲットブラウザの指定（ポリフィル削減）
    target: 'es2020',

    // チャンクサイズの最適化
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          ui: ['@mui/material'],
        },
      },
    },

    // 圧縮設定
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true,
      },
    },
  },
});
```

期待される効果:
- 開発サーバー起動: 期待される起動時間はWebpackと比較して5〜10倍高速
- HMR（Hot Module Replacement）: 期待される更新速度は即座（<100ms）

### TypeScriptコンパイル最適化

```json
// tsconfig.json
{
  "compilerOptions": {
    // 型チェックの高速化
    "skipLibCheck": true,  // 型定義ファイルのチェックをスキップ

    // 増分ビルド
    "incremental": true,
    "tsBuildInfoFile": "./.tsbuildinfo",

    // 並列処理（TypeScript 4.0以降）
    "assumeChangesOnlyAffectDirectDependencies": true,

    // モジュール解決の最適化
    "moduleResolution": "bundler",  // TypeScript 5.0以降
    "resolveJsonModule": true,

    // 出力の最適化
    "removeComments": true,
    "declaration": false,  # 型定義ファイル不要な場合
  }
}
```

期待される効果:
- 増分ビルド: 期待される2回目以降のビルド時間短縮は70〜90%
- skipLibCheck: 期待される型チェック時間短縮は30〜50%

## バンドルサイズの詳細最適化

### 依存関係のサイズ分析

```bash
# バンドルサイズの分析
npx webpack-bundle-analyzer dist/stats.json

# パッケージごとのサイズ確認
npx cost-of-modules

# 特定パッケージのインストールサイズ確認
npx package-size lodash
```

### 軽量代替ライブラリへの置き換え

| 従来のライブラリ | サイズ | 軽量代替 | サイズ | 削減率 |
|---------------|------|---------|------|-------|
| moment.js | 530KB | date-fns | 13KB | 97% |
| lodash | 531KB | lodash-es（Tree Shaking） | 24KB | 95% |
| axios | 13KB | fetch（ネイティブ） | 0KB | 100% |
| jQuery | 87KB | ネイティブDOM API | 0KB | 100% |

```javascript
// ❌ moment.js（重い）
import moment from 'moment';
const formatted = moment().format('YYYY-MM-DD');

// ✅ date-fns（軽量）
import { format } from 'date-fns';
const formatted = format(new Date(), 'yyyy-MM-dd');

// 🏆 ネイティブAPI（最軽量）
const formatted = new Intl.DateTimeFormat('ja-JP').format(new Date());
```

### Tree Shakingの最大活用

```javascript
// ❌ 全体をインポート（Tree Shaking効かない）
import _ from 'lodash';
_.debounce(() => {}, 300);

// ⚠️ 個別インポート（若干改善）
import debounce from 'lodash/debounce';

// ✅ ES Modules版を使用（Tree Shaking有効）
import { debounce } from 'lodash-es';

// package.json設定
{
  "sideEffects": false  // Tree Shaking完全有効化
}
```

### 動的インポートによるCode Splitting

```javascript
// ❌ 静的インポート（初期バンドルに含まれる）
import AdminPanel from './AdminPanel';

// ✅ 動的インポート（必要時のみロード）
const AdminPanel = lazy(() => import('./AdminPanel'));

// Next.js App Routerの場合
import dynamic from 'next/dynamic';
const AdminPanel = dynamic(() => import('./AdminPanel'), {
  loading: () => <p>Loading...</p>,
  ssr: false,  // SSR無効化でバンドルサイズ削減
});
```

期待される効果:
- 初期バンドルサイズ: 期待される削減率は30〜50%
- 初回ロード時間: 期待される短縮率は20〜40%

## 依存関係の最小化戦略

### 不要な依存関係の削除

```bash
# 未使用依存関係の検出
npx depcheck

# 出力例:
# Unused dependencies
# * unused-package
# * another-unused-package

# 削除
npm uninstall unused-package another-unused-package
```

### peerDependenciesの適切な管理

```json
// library/package.json
{
  "peerDependencies": {
    "react": "^18.0.0"  // ホストアプリが提供
  },
  "devDependencies": {
    "react": "^18.0.0"  // 開発時のみ使用
  }
  // dependencies には含めない → バンドルサイズ削減
}
```

### モノレポでの依存関係共有

```yaml
# pnpm-workspace.yaml
packages:
  - 'packages/*'
  - 'apps/*'

# .npmrc
shared-workspace-lockfile=true  # 依存関係を共有
```

期待される効果:
- ディスク使用量: 期待される削減率は50〜70%
- インストール時間: 期待される短縮率は40〜60%

## キャッシュ戦略の最適化

### ローカルキャッシュの活用

```bash
# npm キャッシュの場所確認
npm config get cache
# デフォルト: ~/.npm

# pnpm ストアの場所確認
pnpm store path
# デフォルト: ~/.pnpm-store

# キャッシュサイズの確認
du -sh ~/.npm
du -sh ~/.pnpm-store
```

### CI/CD環境でのキャッシュ最適化

```yaml
# GitHub Actions - 多層キャッシュ戦略
- name: Cache dependencies
  uses: actions/cache@v3
  with:
    path: |
      ~/.pnpm-store
      node_modules
      .next/cache  # Next.js build cache
    key: ${{ runner.os }}-deps-${{ hashFiles('**/pnpm-lock.yaml') }}
    restore-keys: |
      ${{ runner.os }}-deps-
```

期待される効果:
- キャッシュヒット時: 期待されるCI時間短縮は70〜90%
- 月間CI時間: 期待される削減率は50〜70%

### Dockerビルドキャッシュ

```dockerfile
# Dockerfile - マルチステージビルドで最適化
FROM node:20-alpine AS deps

WORKDIR /app

# 依存関係のみを先にインストール（キャッシュ活用）
COPY package.json pnpm-lock.yaml ./
RUN corepack enable pnpm && pnpm install --frozen-lockfile

# アプリケーションコードをコピー
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm build

# 本番イメージ
FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/node_modules ./node_modules
CMD ["pnpm", "start"]
```

期待される効果:
- 依存関係変更なし: 期待されるビルド時間短縮は80〜95%
- イメージサイズ: 期待される削減率は60〜80%

## パフォーマンス監視とベンチマーク

### Lighthouse CI

```yaml
# .github/workflows/lighthouse-ci.yml
name: Lighthouse CI

on: [push]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm ci
      - run: npm run build

      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v10
        with:
          urls: |
            http://localhost:3000
          budgetPath: ./budget.json
          uploadArtifacts: true
```

```json
// budget.json - パフォーマンス予算
[
  {
    "path": "/*",
    "timings": [
      {
        "metric": "interactive",
        "budget": 3000  // 3秒以内
      },
      {
        "metric": "first-contentful-paint",
        "budget": 1000  // 1秒以内
      }
    ],
    "resourceSizes": [
      {
        "resourceType": "script",
        "budget": 300  // 300KB以内
      },
      {
        "resourceType": "total",
        "budget": 500  // 500KB以内
      }
    ]
  }
]
```

### size-limitによる自動監視

```json
// package.json
{
  "scripts": {
    "size": "size-limit",
    "size:why": "size-limit --why"
  },
  "size-limit": [
    {
      "path": "dist/bundle.js",
      "limit": "300 KB",
      "webpack": false
    },
    {
      "path": "dist/vendors.js",
      "limit": "100 KB"
    }
  ]
}
```

```yaml
# GitHub Actions統合
- name: Check bundle size
  run: npm run size
```

期待される効果:
- バンドルサイズ超過の自動検出
- PRでの自動コメント
- パフォーマンス劣化の防止

## まとめ

パフォーマンス最適化は継続的な取り組みが重要です。

### 最適化の優先順位

1. **高優先度**（即座に実施）
   - [ ] pnpmへの移行（大幅な高速化）
   - [ ] CI/CDキャッシュの設定
   - [ ] 不要な依存関係の削除

2. **中優先度**（1〜2週間以内）
   - [ ] Tree Shakingの有効化
   - [ ] Code Splittingの実装
   - [ ] バンドルサイズ監視の導入

3. **低優先度**（時間があるとき）
   - [ ] 軽量代替ライブラリへの置き換え
   - [ ] Dockerイメージの最適化
   - [ ] パフォーマンス予算の設定

### 期待される効果まとめ

| 最適化項目 | 期待される効果 |
|----------|--------------|
| pnpmへの移行 | インストール時間: 2〜3倍高速、ディスク: 60〜70%削減 |
| CI/CDキャッシュ | ビルド時間: 70〜90%短縮 |
| Tree Shaking | バンドルサイズ: 30〜50%削減 |
| Code Splitting | 初回ロード: 20〜40%高速化 |
| 軽量代替ライブラリ | バンドルサイズ: パッケージにより95%以上削減 |

### 継続的な改善サイクル

```
計測 → 分析 → 最適化 → 検証 → 計測...

1. 計測: Lighthouse、Bundle Analyzer
2. 分析: ボトルネックの特定
3. 最適化: 具体的な改善実施
4. 検証: パフォーマンステスト
5. 計測: 効果測定、次の改善へ
```

**参考文献**:
- [pnpm Benchmarks](https://pnpm.io/benchmarks)
- [Webpack Performance Guide](https://webpack.js.org/guides/build-performance/)
- [Vite Performance](https://vitejs.dev/guide/performance.html)
- [Web.dev - Fast load times](https://web.dev/fast/)
- [Bundle Size Optimization](https://bundlesize.io/)

これで、依存関係管理完全ガイドは完了です。本書で学んだ知識を活用して、セキュアで効率的なプロジェクトを構築してください。
