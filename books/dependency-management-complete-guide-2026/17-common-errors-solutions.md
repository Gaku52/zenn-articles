---
title: "よくあるエラーと解決方法"
---

# よくあるエラーと解決方法

## npmエラー

### EACCES: permission denied

```bash
# エラー
Error: EACCES: permission denied, access '/usr/local/lib/node_modules'

# 解決方法1: nvmを使用（推奨）
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install node

# 解決方法2: npmのディレクトリを変更
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
export PATH=~/.npm-global/bin:$PATH
```

### Cannot find module

```bash
# エラー
Error: Cannot find module 'lodash'

# 解決方法
rm -rf node_modules package-lock.json
npm install
```

### Conflicting package.json and package-lock.json

```bash
# エラー
npm ERR! Conflicting package.json and package-lock.json

# 解決方法
npm install  # ロックファイルを更新
```

## pnpmエラー

### ERR_PNPM_NO_MATCHING_VERSION

```bash
# エラー
ERR_PNPM_NO_MATCHING_VERSION No matching version found for package@^2.0.0

# 解決方法
pnpm install package@latest
# または
pnpm update package
```

### Phantom Dependency

```bash
# エラー
Cannot find module 'some-package'

# 解決方法: 明示的に追加
pnpm add some-package
```

## Yarnエラー

### Integrity check failed

```bash
# エラー
error Integrity check failed for "package"

# 解決方法
yarn cache clean
rm -rf node_modules
yarn install
```

## CocoaPodsエラー

### Unable to find a specification for Pod

```bash
# エラー
Unable to find a specification for `Alamofire`

# 解決方法
pod repo update
pod install
```

### [!] The platform of the target is not compatible

```bash
# エラー
The platform of the target `Pods-MyApp` is not compatible with `MyPod`

# 解決方法: Podfileでプラットフォームバージョンを上げる
platform :ios, '15.0'
```

## Swift Package Managerエラー

### ERR_PNPM_PEER_DEP_ISSUES（pnpm）

```bash
# エラー
ERR_PNPM_PEER_DEP_ISSUES Unmet peer dependencies

# 原因: peerDependenciesが満たされていない
# 解決方法1: 必要なpeerDependencyをインストール
pnpm add react@18 react-dom@18

# 解決方法2: auto-install-peersを有効化（.npmrc）
echo "auto-install-peers=true" >> .npmrc
pnpm install
```

### Version conflict in lock file

```bash
# エラー
Version conflict detected in lock file

# 解決方法
rm -rf node_modules pnpm-lock.yaml
pnpm install
```

### Package has unresolved peer dependencies

```bash
# エラー
Package "webpack-dev-server" has unresolved peer dependencies

# 原因を確認
pnpm why webpack-dev-server

# 解決方法: 手動で追加
pnpm add -D webpack@5
```

## ビルド・実行時エラー

### MODULE_NOT_FOUND

```bash
# エラー
Error: Cannot find module 'some-package'

# 原因1: パッケージがインストールされていない
npm install some-package

# 原因2: node_modulesが壊れている
rm -rf node_modules
npm install

# 原因3: Phantom Dependency（pnpm）
# 直接インストールされていないパッケージを参照している
pnpm add some-package
```

### Peer dependency warnings

```bash
# エラー
npm WARN react-router-dom@6.0.0 requires a peer of react@>=16.8.0

# 解決方法1: 推奨バージョンをインストール
npm install react@18

# 解決方法2: 警告を無視（非推奨）
npm install --legacy-peer-deps
```

### Engine compatibility error

```bash
# エラー
error engines: The engine "node" is incompatible with this module

# package.jsonで要求されているNode.jsバージョン
{
  "engines": {
    "node": ">=18.0.0"
  }
}

# 解決方法1: Node.jsをアップグレード
nvm install 18
nvm use 18

# 解決方法2: 強制インストール（非推奨）
npm install --ignore-engines
```

## 依存関係競合の解決

### Conflicting versions

```bash
# エラー
Found: react@17.0.2
Could not resolve dependency:
peer react@"^18.0.0" from react-router-dom@6.0.0

# 原因を詳しく調査
npm ls react

# 解決方法1: メインバージョンを更新
npm install react@18 react-dom@18

# 解決方法2: npm overridesを使用（package.json）
{
  "overrides": {
    "react": "^18.0.0"
  }
}

# 解決方法3: pnpm.overrides（pnpm）
{
  "pnpm": {
    "overrides": {
      "react": "^18.0.0"
    }
  }
}

# 解決方法4: yarn resolutions（Yarn）
{
  "resolutions": {
    "react": "^18.0.0"
  }
}
```

### Duplicate packages

```bash
# 重複パッケージを検出
npm ls lodash

# 出力例:
# ├─┬ package-a@1.0.0
# │ └── lodash@4.17.20
# └─┬ package-b@2.0.0
#   └── lodash@4.17.21

# 解決方法1: npm dedupe
npm dedupe

# 解決方法2: pnpmは自動的に重複排除
pnpm install

# 解決方法3: overridesで統一
{
  "overrides": {
    "lodash": "4.17.21"
  }
}
```

## プラットフォーム固有のエラー

### Windows: EPERM operation not permitted

```bash
# エラー
Error: EPERM: operation not permitted, unlink

# 原因: ファイルロック、アンチウイルスソフトウェア

# 解決方法1: 管理者権限で実行
# PowerShellを管理者として実行
npm install

# 解決方法2: アンチウイルスの除外設定
# node_modulesフォルダを除外リストに追加

# 解決方法3: キャッシュをクリア
npm cache clean --force
rm -rf node_modules
npm install
```

### macOS: Code signing error

```bash
# エラー
codesign: operation not permitted

# 解決方法: Xcodeコマンドラインツールを再インストール
sudo rm -rf /Library/Developer/CommandLineTools
xcode-select --install
```

### Linux: Permission denied

```bash
# エラー
npm ERR! Error: EACCES: permission denied

# 解決方法1: nvmを使用（推奨）
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install node

# 解決方法2: npmディレクトリの権限を変更
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

## 一般的なトラブルシューティング

### キャッシュクリア

```bash
# npm
npm cache clean --force
npm cache verify  # キャッシュの整合性を確認

# Yarn
yarn cache clean
yarn cache list   # キャッシュ一覧を表示

# pnpm
pnpm store prune
pnpm store status # ストアの状態を確認
```

### 完全クリーンインストール

```bash
# npm
rm -rf node_modules package-lock.json
npm install

# Yarn
rm -rf node_modules yarn.lock
yarn install

# pnpm
rm -rf node_modules pnpm-lock.yaml
pnpm install

# すべてのパッケージマネージャーのキャッシュをクリア
rm -rf ~/.npm
rm -rf ~/.yarn
rm -rf ~/.pnpm-store
```

### ロックファイルの再生成

```bash
# npm
rm package-lock.json
npm install

# Yarn
rm yarn.lock
yarn install

# pnpm
rm pnpm-lock.yaml
pnpm install
```

## CI/CD環境でのエラー

### GitHub Actions: npm ci fails

```bash
# エラー
npm ERR! The `npm ci` command can only install with an existing package-lock.json

# 原因: package-lock.jsonがGitにコミットされていない

# 解決方法
git add package-lock.json
git commit -m "chore: add package-lock.json"
git push
```

### Out of memory error

```bash
# エラー
FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory

# 解決方法1: Node.jsのメモリ上限を増やす
export NODE_OPTIONS="--max-old-space-size=4096"
npm install

# 解決方法2: package.jsonにscriptを追加
{
  "scripts": {
    "install": "node --max-old-space-size=4096 $(which npm) install"
  }
}

# GitHub Actionsの場合
- name: Install dependencies
  run: npm install
  env:
    NODE_OPTIONS: --max-old-space-size=4096
```

### Network timeout

```bash
# エラー
npm ERR! network request to https://registry.npmjs.org/ failed, reason: connect ETIMEDOUT

# 解決方法1: タイムアウトを延長
npm config set fetch-timeout 60000
npm install

# 解決方法2: リトライ回数を増やす
npm config set fetch-retries 5
npm install

# 解決方法3: ミラーを使用
npm config set registry https://registry.npmmirror.com
npm install
```

## デバッグテクニック

### ログレベルの調整

```bash
# npm - 詳細ログ
npm install --loglevel verbose

# npm - デバッグログ
npm install --loglevel silly

# Yarn - 詳細ログ
yarn install --verbose

# pnpm - デバッグログ
pnpm install --debug
```

### 依存関係ツリーの可視化

```bash
# npm - 依存関係ツリーを表示
npm ls
npm ls lodash  # 特定パッケージのみ
npm ls --depth=1  # 深さを制限

# Yarn
yarn list
yarn why lodash  # パッケージが必要な理由を表示

# pnpm
pnpm list
pnpm why lodash
```

### パッケージの詳細情報

```bash
# パッケージのメタデータを表示
npm info react
npm info react versions  # 全バージョン一覧
npm info react version   # 最新バージョン

# パッケージの依存関係を表示
npm info react dependencies
npm info react peerDependencies
```

## よくある質問（FAQ）

### Q1: node_modulesを.gitignoreに追加すべきですか？

**A**: はい、必ず追加してください。node_modulesは非常に大きく、Gitで管理する必要はありません。代わりにロックファイル（package-lock.json、yarn.lock、pnpm-lock.yaml）をコミットします。

```bash
# .gitignore
node_modules/
```

### Q2: ロックファイルをコミットすべきですか？

**A**: はい、必ずコミットしてください。ロックファイルにより、全員が同じバージョンの依存関係を使用できます。

```bash
# Gitにコミット
git add package-lock.json
git commit -m "chore: update dependencies"
```

### Q3: npm install と npm ci の違いは？

**A**: `npm ci`は、package-lock.jsonを厳密に守り、CI/CD環境に最適化されています。

| 特徴 | npm install | npm ci |
|-----|------------|---------|
| package-lock.json | 更新される | 更新されない |
| node_modules | 既存を保持 | 完全削除して再作成 |
| 速度 | 通常 | より高速 |
| 用途 | 開発環境 | CI/CD環境 |

```bash
# 開発環境
npm install

# CI/CD環境
npm ci
```

### Q4: "gyp ERR!"エラーの対処法は？

**A**: ネイティブモジュールのビルドエラーです。ビルドツールをインストールしてください。

```bash
# macOS
xcode-select --install

# Windows
npm install --global windows-build-tools

# Linux (Ubuntu)
sudo apt-get install build-essential

# または、プリビルドバイナリを使用
npm rebuild
```

### Q5: エラーログはどこにありますか？

**A**: エラーログは以下の場所に保存されています。

```bash
# npmエラーログ
~/.npm/_logs/

# 最新のログを表示
npm config get cache
ls -la ~/.npm/_logs/
cat ~/.npm/_logs/*-debug.log
```

### Q6: "ERESOLVE unable to resolve dependency tree"の対処法は？

**A**: npm 7以降で厳格化されたpeer dependencies解決の問題です。

```bash
# 解決方法1: --legacy-peer-depsフラグを使用（npm 6の動作に戻す）
npm install --legacy-peer-deps

# 解決方法2: --force（非推奨、依存関係の競合を無視）
npm install --force

# 解決方法3: package.jsonでoverridesを使用（npm 8.3以降）
{
  "overrides": {
    "problematic-package": "^2.0.0"
  }
}

# 解決方法4: pnpmに移行（厳密な依存関係管理）
pnpm install
```

### Q7: "MODULE_NOT_FOUND"でも、パッケージはインストールされている？

**A**: TypeScript/JavaScriptのモジュール解決の問題です。

```bash
# 原因1: tsconfig.jsonのpathsが正しくない
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]  # 正しいパス設定を確認
    }
  }
}

# 原因2: package.jsonのexportsフィールドの問題
# パッケージの新しいバージョンでexportsフィールドが変更されている可能性

# 原因3: node_modulesの.binディレクトリの問題
rm -rf node_modules/.bin
npm install

# デバッグ方法
node --trace-warnings index.js
```

### Q8: Yarnでの"Resolution field" is set, but is not supported"警告

**A**: npm CLIではresolutionsフィールドをサポートしていません。

```bash
# Yarn専用のresolutionsフィールド
{
  "resolutions": {
    "lodash": "^4.17.21"
  }
}

# npmで同等の機能を使う場合
{
  "overrides": {  # npm 8.3以降
    "lodash": "^4.17.21"
  }
}

# pnpmで同等の機能を使う場合
{
  "pnpm": {
    "overrides": {
      "lodash": "^4.17.21"
    }
  }
}
```

### Q9: "Cannot read property 'version' of undefined"

**A**: 破損したpackage.jsonまたはnode_modulesの問題です。

```bash
# 1. package.jsonの構文を確認
npm pkg get name  # エラーが出る場合、JSONが破損

# 2. JSONフォーマットを検証
npx jsonlint package.json

# 3. node_modulesを再構築
rm -rf node_modules package-lock.json
npm install

# 4. npmキャッシュの検証
npm cache verify
```

### Q10: "Error: ENOSPC: System limit for number of file watchers reached"（Linux）

**A**: inotifyの監視可能ファイル数の上限に達しています。

```bash
# 現在の上限を確認
cat /proc/sys/fs/inotify/max_user_watches

# 一時的に上限を増やす
sudo sysctl fs.inotify.max_user_watches=524288

# 永続的に設定
echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## 高度なトラブルシューティング

### ネットワーク関連のエラー

```bash
# プロキシ設定の確認
npm config get proxy
npm config get https-proxy

# プロキシ設定
npm config set proxy http://proxy.example.com:8080
npm config set https-proxy http://proxy.example.com:8080

# SSL証明書の問題（企業ネットワーク）
npm config set strict-ssl false  # セキュリティリスクあり、一時的な対処のみ

# レジストリの変更
npm config set registry https://registry.npmmirror.com  # 中国ミラー
npm config set registry https://registry.npmjs.org     # 公式に戻す
```

### メモリ関連のエラー

```bash
# Node.jsのメモリ上限を増やす
export NODE_OPTIONS="--max-old-space-size=4096"

# package.jsonで設定
{
  "scripts": {
    "build": "node --max-old-space-size=4096 node_modules/.bin/webpack"
  }
}

# pnpmの場合
pnpm --max-old-space-size=4096 install
```

### CI/CD環境特有のエラー

```bash
# GitHub Actions: npm auditの自動修正が失敗
# エラー: npm ERR! code EAUDITFIX
# 解決: 手動でpackage.jsonを修正

# GitLab CI: キャッシュが効かない
# 解決: cache.pathsを正しく設定
cache:
  paths:
    - node_modules/
    - .npm/

# CircleCI: 並列実行での競合
# 解決: npm ciを使用（並列安全）
- run: npm ci --prefer-offline
```

### TypeScript型定義のエラー

```bash
# エラー: "Cannot find type definition file for 'node'"
npm install --save-dev @types/node

# エラー: "Duplicate identifier"（型定義の競合）
# tsconfig.jsonで明示的に指定
{
  "compilerOptions": {
    "types": ["node", "jest"]  # 必要な型のみ指定
  }
}

# パッケージに型定義が含まれているか確認
npm info package-name

# DefinitelyTypedから型定義をインストール
npm install --save-dev @types/package-name
```

### モノレポ特有のエラー

```bash
# Lerna: "Package not found"
# 原因: workspace内のパッケージが見つからない
lerna bootstrap  # 依存関係を再リンク

# Yarn Workspaces: "Workspaces can only be enabled in private projects"
# package.jsonに追加
{
  "private": true,
  "workspaces": [
    "packages/*"
  ]
}

# pnpm Workspaces: "No projects matched the filters"
# pnpm-workspace.yamlを確認
packages:
  - 'packages/*'
  - '!packages/legacy'  # 除外パッケージ
```

## エラー診断フローチャート

```
エラー発生
    |
    ├─ インストールエラー？
    |   ├─ Yes → キャッシュクリア → node_modules削除 → 再インストール
    |   └─ No → 次へ
    |
    ├─ ビルドエラー？
    |   ├─ Yes → 型定義確認 → tsconfig確認 → 依存関係確認
    |   └─ No → 次へ
    |
    ├─ 実行時エラー？
    |   ├─ Yes → モジュール解決確認 → バージョン確認 → ログ確認
    |   └─ No → 次へ
    |
    └─ その他
        └─ ログファイル確認 → 公式ドキュメント → GitHub Issues検索
```

## プラットフォーム別エラー集

### macOS固有

```bash
# Error: "xcrun: error: invalid active developer path"
# Xcodeコマンドラインツールが見つからない
xcode-select --install

# Error: "dyld: Library not loaded"
# 共有ライブラリのパス問題
brew reinstall node
```

### Windows固有

```bash
# Error: "Cannot find module '../build/Release/xxx.node'"
# ネイティブモジュールのビルド失敗
npm install --global windows-build-tools
npm rebuild

# Error: "Path too long"（パス長制限）
# Windows 10以降で長いパスを有効化
# レジストリエディタで以下を設定
# HKLM\SYSTEM\CurrentControlSet\Control\FileSystem\LongPathsEnabled = 1

# または、プロジェクトパスを短くする
# C:\p\myproject など
```

### Linux固有

```bash
# Error: "EACCES: permission denied, mkdir"
# グローバルインストールの権限問題
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc

# Error: "sh: 1: node: not found"
# Node.jsがPATHにない
which node  # パスを確認
ln -s "$(which node)" /usr/bin/node  # シンボリックリンクを作成
```

## エラー予防のベストプラクティス

### 1. 環境の統一

```json
// package.jsonでNode.jsバージョンを固定
{
  "engines": {
    "node": ">=18.0.0 <21.0.0",
    "npm": ">=9.0.0"
  }
}

// .nvmrcファイルでバージョン指定
18.19.0
```

### 2. ロックファイルの厳守

```bash
# 開発環境: npm install（ロックファイル更新可能）
npm install

# CI/CD環境: npm ci（ロックファイル厳守）
npm ci

# package-lock.jsonを必ずコミット
git add package-lock.json
```

### 3. 依存関係の定期監査

```bash
# 週次実行推奨
npm audit

# 自動修正（Patch更新のみ）
npm audit fix

# 破壊的変更も含む修正（慎重に）
npm audit fix --force

# 監査レポートをJSON形式で出力
npm audit --json > audit-report.json
```

### 4. 型定義の整合性確保

```bash
# TypeScriptプロジェクトで型チェック
npx tsc --noEmit

# 型定義の依存関係を明示
{
  "devDependencies": {
    "@types/node": "^20.10.0",
    "@types/react": "^18.2.0"
  }
}
```

## トラブルシューティングツール

### npm-check-updates

```bash
# インストール
npm install -g npm-check-updates

# 更新可能なパッケージを確認
ncu

# package.jsonを更新（実行前に確認）
ncu -u

# 特定パッケージのみ確認
ncu express
```

### depcheck（未使用依存関係の検出）

```bash
# インストール
npm install -g depcheck

# 未使用依存関係を検出
depcheck

# JSON形式で出力
depcheck --json
```

### npm-why（依存関係の理由を調査）

```bash
# 特定パッケージがなぜインストールされているか確認
npm ls package-name

# pnpmの場合
pnpm why package-name

# Yarnの場合
yarn why package-name
```

## まとめ

多くのエラーは、以下の基本的な手順で解決します。

**トラブルシューティングの基本フロー**:
1. **エラーメッセージを読む**: 何が問題かを正確に把握
2. **キャッシュをクリア**: `npm cache clean --force`
3. **依存関係を再インストール**: `rm -rf node_modules && npm install`
4. **ロックファイルを再生成**: `rm package-lock.json && npm install`
5. **公式ドキュメントを確認**: エラーメッセージで検索
6. **GitHub Issuesを検索**: 同じ問題を抱えている人がいないか確認

**参考リソース**:
- [npm公式ドキュメント - Troubleshooting](https://docs.npmjs.com/cli/v10/using-npm/troubleshooting)
- [pnpm公式ドキュメント - FAQ](https://pnpm.io/faq)
- [Yarn公式ドキュメント - Troubleshooting](https://yarnpkg.com/getting-started/troubleshooting)
- [Stack Overflow - npm tag](https://stackoverflow.com/questions/tagged/npm)

次章では、パフォーマンス最適化について解説します。
