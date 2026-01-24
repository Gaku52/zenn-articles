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

## 一般的なトラブルシューティング

### キャッシュクリア

```bash
# npm
npm cache clean --force

# Yarn
yarn cache clean

# pnpm
pnpm store prune
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
```

## まとめ

多くのエラーは、キャッシュクリアや完全な再インストールで解決します。

次章では、パフォーマンス最適化について解説します。
