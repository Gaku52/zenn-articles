---
title: "npm基礎 - コマンドとベストプラクティス"
---

# npm基礎 - コマンドとベストプラクティス

## npmとは

npm (Node Package Manager) は、Node.jsのデフォルトパッケージマネージャーです。2010年に登場して以来、JavaScriptエコシステムの中核を担っています。

### npmのインストール確認

npmはNode.jsと一緒にインストールされます。

```bash
# Node.jsとnpmのバージョン確認
node -v  # v18.0.0以上推奨
npm -v   # v9.0.0以上推奨

# 最新版への更新
npm install -g npm@latest
```

## 基本コマンド

### プロジェクトの初期化

```bash
# 対話形式でpackage.jsonを作成
npm init

# デフォルト値で作成
npm init -y

# 特定のイニシャライザーを使用
npm init react-app my-app
npm init next-app my-next-app
```

### パッケージのインストール

```bash
# package.jsonの依存関係をすべてインストール
npm install

# 特定のパッケージをインストール
npm install lodash

# 複数パッケージを同時にインストール
npm install react react-dom

# バージョン指定
npm install lodash@4.17.21     # 特定バージョン
npm install lodash@^4.17.0     # 4.x系の最新
npm install lodash@~4.17.0     # 4.17.x系の最新

# 開発依存としてインストール
npm install --save-dev jest
npm install -D eslint prettier

# グローバルインストール
npm install -g typescript
```

### パッケージの削除

```bash
# パッケージを削除
npm uninstall lodash

# 開発依存を削除
npm uninstall -D jest

# グローバルパッケージを削除
npm uninstall -g typescript
```

### パッケージの更新

```bash
# すべてのパッケージを更新（パッチバージョンのみ）
npm update

# 特定のパッケージを更新
npm update lodash

# 古いパッケージを確認
npm outdated

# 出力例:
# Package   Current  Wanted  Latest
# react     18.2.0   18.2.0  18.3.1
# lodash     4.17.20  4.17.21  4.17.21
```

### npm ci - CI/CD用の決定論的インストール

```bash
# package-lock.jsonに基づいて正確にインストール
npm ci

# 特徴:
# - package-lock.jsonが必須
# - node_modulesを削除してからインストール
# - package.jsonとロックファイルの不一致でエラー
# - npm installより高速（20〜40%）
```

**CI/CDでは必ずnpm ciを使用:**

```yaml
# .github/workflows/test.yml
- name: Install dependencies
  run: npm ci  # ❌ npm installではなく✅ npm ci
```

## package.jsonの構造

```json
{
  "name": "my-awesome-app",
  "version": "1.0.0",
  "description": "素晴らしいアプリケーション",
  "main": "dist/index.js",
  "type": "module",

  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "test": "vitest run",
    "lint": "eslint . --ext .ts,.tsx"
  },

  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },

  "devDependencies": {
    "@types/react": "^18.2.0",
    "typescript": "^5.3.0",
    "vite": "^5.0.0"
  },

  "engines": {
    "node": ">=18.0.0",
    "npm": ">=9.0.0"
  },

  "repository": {
    "type": "git",
    "url": "https://github.com/username/my-awesome-app"
  },

  "keywords": ["react", "typescript", "vite"],
  "author": "Your Name <you@example.com>",
  "license": "MIT"
}
```

### 重要なフィールド

**name**: パッケージ名（スコープ付き: @username/package-name）
**version**: セマンティックバージョニング（詳細は後述）
**dependencies**: 本番環境で必要な依存関係
**devDependencies**: 開発時のみ必要な依存関係
**scripts**: カスタムコマンド
**engines**: Node.jsとnpmの必須バージョン

## .npmrcによる設定

プロジェクトルートに`.npmrc`ファイルを配置することで、npmの動作をカスタマイズできます。

```bash
# .npmrc

# バージョン表記を正確に
save-exact=true
save-prefix=""

# エンジンバージョンを厳密にチェック
engine-strict=true

# セキュリティレベル
audit-level=moderate

# 不要な出力を抑制
fund=false
loglevel=warn

# パフォーマンス最適化
prefer-offline=true
cache-min=86400

# プライベートレジストリ設定（企業内利用）
@mycompany:registry=https://npm.pkg.github.com/
//npm.pkg.github.com/:_authToken=${NPM_TOKEN}
```

### 推奨される.npmrc設定

```bash
# 本番環境向け推奨設定
save-exact=true
engine-strict=true
audit-level=moderate
fund=false
prefer-offline=true
```

## npm scripts

npm scriptsは、プロジェクト固有のコマンドを定義する強力な機能です。

### 基本的なscripts

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write \"src/**/*.{ts,tsx}\"",
    "type-check": "tsc --noEmit"
  }
}
```

### ライフサイクルスクリプト

npmには特別なタイミングで自動実行されるスクリプトがあります。

```json
{
  "scripts": {
    "preinstall": "node scripts/check-node-version.js",
    "postinstall": "patch-package",
    "prepare": "husky install",
    "prebuild": "npm run lint",
    "build": "tsc && vite build",
    "postbuild": "npm run test"
  }
}
```

**実行順序:**
```
npm run build
  ↓
prebuild (自動実行)
  ↓
build
  ↓
postbuild (自動実行)
```

### スクリプトの連鎖

```json
{
  "scripts": {
    "clean": "rm -rf dist",
    "compile": "tsc",
    "bundle": "vite build",

    // 順次実行（&&）
    "build": "npm run clean && npm run compile && npm run bundle",

    // 並列実行（npm-run-all使用）
    "lint:all": "npm-run-all --parallel lint:js lint:css lint:html",
    "lint:js": "eslint .",
    "lint:css": "stylelint '**/*.css'",
    "lint:html": "htmlhint '**/*.html'"
  }
}
```

## セキュリティ監査

### npm audit

```bash
# セキュリティ監査を実行
npm audit

# 出力例:
# ┌───────────────┬──────────────────────────────────────────────────────────┐
# │ moderate      │ Regular Expression Denial of Service                     │
# ├───────────────┼──────────────────────────────────────────────────────────┤
# │ Package       │ minimatch                                                │
# ├───────────────┼──────────────────────────────────────────────────────────┤
# │ Patched in    │ >=3.0.5                                                  │
# └───────────────┴──────────────────────────────────────────────────────────┘

# 自動修正を試みる
npm audit fix

# 破壊的変更を含む修正も実行（注意が必要）
npm audit fix --force

# JSON形式で出力
npm audit --json

# 特定の深刻度以上をチェック
npm audit --audit-level=high
```

### 本番環境のみの監査

```bash
# 開発依存を除外して監査
npm audit --production
```

## package-lock.json

package-lock.jsonは、依存関係ツリーの正確なスナップショットです。

### なぜpackage-lock.jsonが必要か

```bash
# package.json
"dependencies": {
  "react": "^18.2.0"  # 18.2.0以上、19.0.0未満
}

# 開発者Aがインストール（2023年1月）
→ react 18.2.0

# 開発者Bがインストール（2023年6月、新バージョンリリース後）
→ react 18.3.5  # 異なるバージョン！

# package-lock.jsonがあれば
→ 全員が18.2.0（ロックファイルに記載されたバージョン）
```

### package-lock.jsonの管理

```bash
# 必ずGitにコミット
git add package-lock.json
git commit -m "chore: update dependencies"

# .gitignoreに追加しない！
# ❌ package-lock.json

# ロックファイルの再生成（通常は不要）
rm package-lock.json
npm install
```

## トラブルシューティング

### キャッシュのクリア

```bash
# npmキャッシュをクリア
npm cache clean --force

# キャッシュの確認
npm cache verify
```

### node_modulesの再インストール

```bash
# 完全なクリーンインストール
rm -rf node_modules package-lock.json
npm install
```

### グローバルパッケージの確認

```bash
# インストール済みグローバルパッケージを表示
npm list -g --depth=0

# 特定パッケージのパス
npm root -g
```

## まとめ

npmの基本を押さえることで、依存関係管理の土台ができます。

**重要ポイント:**

1. **CI/CDではnpm ciを使用** - 再現性を保証
2. **package-lock.jsonを必ずコミット** - チーム全体で同じバージョンを使用
3. **.npmrcで設定を統一** - プロジェクト固有の設定を共有
4. **npm auditで定期的にセキュリティチェック** - 脆弱性を早期発見
5. **npm scriptsで作業を自動化** - 開発効率を向上

次章では、pnpmの利点と実践的な使い方を解説します。
