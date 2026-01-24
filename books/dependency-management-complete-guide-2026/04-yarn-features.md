---
title: "Yarnの特徴と使い方"
---

# Yarnの特徴と使い方

## Yarnの歴史

Yarnは2016年にFacebookがリリースしたパッケージマネージャーです。当時のnpmの課題（速度、決定論的インストール）を解決するために開発されました。

### Yarn ClassicとYarn Berry

Yarnには2つの大きなバージョンがあります。

- **Yarn Classic (v1.x)**: 従来のYarn、安定版
- **Yarn Berry (v2+)**: 新アーキテクチャ、Plug'n'Play機能

## Yarn Classicの基本

### インストール

```bash
# npm経由でインストール
npm install -g yarn

# Homebrewでインストール（macOS）
brew install yarn

# バージョン確認
yarn --version
```

### 基本コマンド

```bash
# プロジェクト初期化
yarn init

# 依存関係のインストール
yarn install
yarn install --frozen-lockfile  # CI用

# パッケージの追加
yarn add lodash                 # 依存関係に追加
yarn add --dev jest             # 開発依存に追加
yarn global add typescript      # グローバルに追加

# パッケージの削除
yarn remove lodash

# 更新
yarn upgrade                    # すべて更新
yarn upgrade lodash             # 特定パッケージのみ
yarn outdated                   # 古いパッケージを確認

# スクリプト実行
yarn build                      # "run"は不要
yarn test
```

### yarn.lockファイル

yarn.lockは、インストールされる正確なバージョンを記録します。

```yaml
# yarn.lock
lodash@^4.17.0:
  version "4.17.21"
  resolved "https://registry.yarnpkg.com/lodash/-/lodash-4.17.21.tgz"
  integrity sha512-v2kDEe57lecTulaDIuNTPy3Ry4gLGJ6Z1O3vE1krgXZNrsQ+LFTGHVxVjcXPs17LhbZVGedAJv8XZ1tvj5FvSg==
```

## Yarn Berry (Modern Yarn)

### Plug'n'Play (PnP)

Yarn BerryのPnPモードは、node_modulesを生成せず、`.pnp.cjs`ファイルで依存関係を管理します。

**従来の方式:**
```
node_modules/
  ├── package-a/
  ├── package-b/
  └── ... (数千〜数万ファイル)
```

**Plug'n'Play:**
```
.yarn/
  ├── cache/             # 圧縮されたパッケージ
  └── releases/
.pnp.cjs                # 依存関係マップ
```

### Yarn Berryへの移行

```bash
# Yarn Berryに更新
yarn set version stable

# PnPモードを有効化（デフォルト）
# .yarnrc.yml が生成される

# ゼロインストールを有効化
echo ".yarn/*" >> .gitignore
echo "!.yarn/cache" >> .gitignore
echo "!.yarn/releases" >> .gitignore
echo "!.yarn/plugins" >> .gitignore

# 依存関係をGitにコミット
git add .yarn/cache
git commit -m "feat: enable Zero-Install"
```

### .yarnrc.yml設定

```yaml
# .yarnrc.yml
nodeLinker: pnp                  # PnPモード

compressionLevel: mixed          # 圧縮レベル

enableGlobalCache: true          # グローバルキャッシュ有効化

# パッケージの拡張（サードパーティの修正）
packageExtensions:
  "react-redux@*":
    peerDependencies:
      react: "*"

# ネットワーク設定
httpTimeout: 60000
networkConcurrency: 8
```

## Yarnワークスペース

Yarnは、モノレポに優れたワークスペース機能を提供します。

### package.jsonの設定

```json
{
  "private": true,
  "workspaces": [
    "packages/*",
    "apps/*"
  ]
}
```

### ディレクトリ構造

```
monorepo/
├── package.json
├── yarn.lock
├── packages/
│   ├── ui-components/
│   │   └── package.json
│   └── utils/
│       └── package.json
└── apps/
    ├── web/
    │   └── package.json
    └── mobile/
        └── package.json
```

### ワークスペースコマンド

```bash
# すべてのワークスペースで実行
yarn workspaces run build
yarn workspaces run test

# 特定のワークスペースで実行
yarn workspace ui-components build
yarn workspace web dev

# 依存関係を追加
yarn workspace web add react
yarn workspace ui-components add lodash
```

## Yarn vs npm vs pnpm

| 機能 | npm | Yarn Classic | Yarn Berry | pnpm |
|------|-----|--------------|------------|------|
| 速度 | 中 | 高 | 最高 | 最高 |
| ディスク効率 | 低 | 低 | 最高 | 最高 |
| 決定論的 | ✅ | ✅ | ✅ | ✅ |
| ワークスペース | ✅ | ✅ | ✅ | ✅ |
| PnP | ❌ | ❌ | ✅ | ❌ |
| ゼロインストール | ❌ | ❌ | ✅ | ❌ |

## CI/CDでのYarn使用

### GitHub Actions（Yarn Classic）

```yaml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'yarn'

      - name: Install dependencies
        run: yarn install --frozen-lockfile

      - name: Run tests
        run: yarn test

      - name: Build
        run: yarn build
```

### GitHub Actions（Yarn Berry + Zero-Install）

```yaml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '18'

      # Zero-Installのため、yarn installは不要

      - name: Run tests
        run: yarn test

      - name: Build
        run: yarn build
```

## Yarnの利点と課題

### Yarn Classicの利点

- 安定性と成熟度
- 広範な互換性
- 豊富なドキュメント
- オフラインキャッシュ

### Yarn Berryの利点

- Plug'n'Play（劇的に高速）
- ゼロインストール
- ディスク効率最高
- 高度なワークスペース機能

### 課題

- **Yarn Classic**: メンテナンスモード（新機能なし）
- **Yarn Berry**: 学習曲線、一部ツールとの非互換性
- バージョン間の大きな違い

## npmからYarnへの移行

```bash
# 1. Yarnをインストール
npm install -g yarn

# 2. package-lock.jsonからyarn.lockを生成
yarn import

# 3. node_modulesを再生成
rm -rf node_modules
yarn install

# 4. package-lock.jsonを削除
rm package-lock.json

# 5. CIスクリプトを更新
# npm ci → yarn install --frozen-lockfile
```

## まとめ

Yarnは、特にモノレポやReactエコシステムで有用です。

**Yarn Classicを選ぶ場合:**
- 安定性を重視
- 既存プロジェクトがYarnを使用
- 学習コストを最小化したい

**Yarn Berryを選ぶ場合:**
- 最先端技術を試したい
- ゼロインストールの利点を活用
- ディスク容量と速度を最大化

次章では、package.jsonのベストプラクティスを解説します。
