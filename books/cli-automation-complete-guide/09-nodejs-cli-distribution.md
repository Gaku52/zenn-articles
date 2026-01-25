---
title: "Node.js CLI配布"
---

# Node.js CLI配布

CLIツールを多くのユーザーに届けるには、適切な配布方法が重要です。本章では、npm公開、バイナリ配布、GitHub Releasesなど、様々な配布手法を学びます。

## npm公開

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
  "files": [
    "dist",
    "README.md",
    "LICENSE"
  ],
  "scripts": {
    "build": "tsc",
    "prepublishOnly": "npm run build && npm test"
  },
  "keywords": ["cli", "tool", "generator"],
  "author": "Your Name <your.email@example.com>",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "https://github.com/yourusername/my-cli-tool.git"
  },
  "bugs": {
    "url": "https://github.com/yourusername/my-cli-tool/issues"
  },
  "homepage": "https://github.com/yourusername/my-cli-tool#readme",
  "engines": {
    "node": ">=18.0.0"
  }
}
```

### .npmignore

```
# ソースファイル
src/
*.ts
tsconfig.json

# テスト
tests/
__tests__/
*.test.js
*.spec.js
jest.config.js

# 設定ファイル
.eslintrc
.prettierrc
.editorconfig

# CI/CD
.github/
.gitlab-ci.yml

# その他
node_modules/
.DS_Store
*.log
coverage/
.vscode/
```

### 公開手順

```bash
# 1. npmアカウント作成
# https://www.npmjs.com/signup

# 2. ログイン
npm login

# 3. パッケージ名の確認
npm search my-cli-tool

# 4. テストとビルド
npm test
npm run build

# 5. 公開前確認
npm pack
tar -xzf my-cli-tool-1.0.0.tgz
cd package
npm install -g

# 6. 公開
npm publish

# スコープ付きパッケージの場合
npm publish --access public
```

### バージョン管理

```bash
# パッチバージョン (1.0.0 -> 1.0.1)
npm version patch

# マイナーバージョン (1.0.0 -> 1.1.0)
npm version minor

# メジャーバージョン (1.0.0 -> 2.0.0)
npm version major

# プレリリース (1.0.0 -> 1.0.1-alpha.0)
npm version prerelease --preid=alpha

# タグをプッシュ
git push --follow-tags

# 公開
npm publish
```

## バイナリ配布

### pkgによるバイナリ化

```bash
npm install -g pkg
```

```json
// package.json
{
  "name": "my-cli-tool",
  "bin": "dist/index.js",
  "pkg": {
    "scripts": "dist/**/*.js",
    "assets": [
      "templates/**/*",
      "node_modules/some-native-module/**/*"
    ],
    "targets": [
      "node18-linux-x64",
      "node18-macos-x64",
      "node18-macos-arm64",
      "node18-win-x64"
    ],
    "outputPath": "binaries"
  },
  "scripts": {
    "build:binary": "pkg . --out-path binaries"
  }
}
```

```bash
# ビルド
npm run build:binary

# 出力
# binaries/my-cli-macos-arm64
# binaries/my-cli-linux-x64
# binaries/my-cli-win-x64.exe
```

### Bunによるバイナリ化

```bash
# インストール
curl -fsSL https://bun.sh/install | bash

# ビルド
bun build ./src/index.ts --compile --outfile my-cli

# 実行
./my-cli --help
```

## GitHub Releases

### リリース自動化

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  npm-publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          registry-url: 'https://registry.npmjs.org'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Test
        run: npm test

      - name: Publish to npm
        run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}

  build-binaries:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        include:
          - os: ubuntu-latest
            target: linux-x64
          - os: macos-latest
            target: macos-arm64
          - os: windows-latest
            target: win-x64

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Build binary
        run: npx pkg . -t node18-${{ matrix.target }} -o my-cli-${{ matrix.target }}

      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: my-cli-${{ matrix.target }}
          path: my-cli-${{ matrix.target }}*

  create-release:
    needs: [npm-publish, build-binaries]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Download all artifacts
        uses: actions/download-artifact@v3

      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          files: |
            my-cli-*/*
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## 自動更新

### update-notifier

```bash
npm install update-notifier
```

```typescript
// src/index.ts
import updateNotifier from 'update-notifier'
import { readFileSync } from 'fs'
import { join } from 'path'

const pkg = JSON.parse(
  readFileSync(join(__dirname, '../package.json'), 'utf-8')
)

const notifier = updateNotifier({
  pkg,
  updateCheckInterval: 1000 * 60 * 60 * 24 // 24時間
})

if (notifier.update) {
  notifier.notify({
    message: `Update available ${notifier.update.current} → ${notifier.update.latest}\nRun {updateCommand} to update`
  })
}

// CLIコマンド実行
program.parse()
```

### セルフアップデート機能

```typescript
// src/commands/update.ts
import { Command } from 'commander'
import { execa } from 'execa'
import ora from 'ora'
import chalk from 'chalk'

export const updateCommand = new Command('update')
  .description('Update CLI to the latest version')
  .action(async () => {
    const spinner = ora('Checking for updates...').start()

    try {
      const { stdout } = await execa('npm', ['view', 'my-cli-tool', 'version'])
      const latestVersion = stdout.trim()

      const pkg = JSON.parse(
        readFileSync(join(__dirname, '../../package.json'), 'utf-8')
      )
      const currentVersion = pkg.version

      if (latestVersion === currentVersion) {
        spinner.succeed(chalk.green('Already up to date!'))
        return
      }

      spinner.text = `Updating ${currentVersion} → ${latestVersion}...`

      await execa('npm', ['install', '-g', 'my-cli-tool@latest'], {
        stdio: 'inherit'
      })

      spinner.succeed(chalk.green('Updated successfully!'))
    } catch (error) {
      spinner.fail(chalk.red('Update failed'))
      console.error(error)
      process.exit(1)
    }
  })
```

## Homebrew配布

### Formula作成

```ruby
# Formula/my-cli.rb
class MyCli < Formula
  desc "A powerful CLI tool"
  homepage "https://github.com/yourusername/my-cli-tool"
  url "https://github.com/yourusername/my-cli-tool/archive/v1.0.0.tar.gz"
  sha256 "abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890"
  license "MIT"

  depends_on "node"

  def install
    system "npm", "install", *Language::Node.std_npm_install_args(libexec)
    bin.install_symlink Dir["#{libexec}/bin/*"]
  end

  test do
    system "#{bin}/mycli", "--version"
  end
end
```

### インストール

```bash
# tap追加
brew tap yourusername/my-cli

# インストール
brew install my-cli

# 更新
brew upgrade my-cli
```

## まとめ

本章では、Node.js CLIの配布方法を学びました。

**重要ポイント**:
- npmで簡単にインストール可能にする
- pkgでバイナリ化し、Node.js不要で実行可能にする
- GitHub Actionsで自動リリース
- update-notifierで更新通知
- Homebrewで macOS/Linux ユーザーに配布

適切な配布方法により、より多くのユーザーにCLIツールを届けられます。次章では、Node.js CLI開発のベストプラクティスを学びます。
