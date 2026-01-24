---
title: "ロックファイル管理"
---

# ロックファイル管理

## ロックファイルとは

ロックファイルは、インストールされた依存関係の正確なバージョンを記録するファイルです。チーム全体で同じバージョンの依存関係を使用することを保証します。

### なぜロックファイルが必要か

```bash
# package.json
{
  "dependencies": {
    "react": "^18.2.0"
  }
}

# 開発者Aがインストール（2023年1月）
→ react 18.2.0がインストールされる

# 開発者Bがインストール（2023年6月、新バージョンリリース後）
→ react 18.3.5がインストールされる

# 結果：異なるバージョン！
→ "開発環境では動くが本番で動かない"問題の原因
```

**ロックファイルがあれば:**

```bash
# 全員が同じバージョン
→ react 18.2.0（ロックファイルに記載されたバージョン）
```

## 主要なロックファイル

| パッケージマネージャー | ロックファイル | 形式 |
|----------------------|---------------|------|
| npm | package-lock.json | JSON |
| Yarn Classic | yarn.lock | YAML風 |
| Yarn Berry | yarn.lock | YAML風 |
| pnpm | pnpm-lock.yaml | YAML |

## package-lock.json（npm）

### 構造

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "lockfileVersion": 3,
  "requires": true,
  "packages": {
    "": {
      "name": "my-app",
      "version": "1.0.0",
      "dependencies": {
        "react": "^18.2.0"
      }
    },
    "node_modules/react": {
      "version": "18.2.0",
      "resolved": "https://registry.npmjs.org/react/-/react-18.2.0.tgz",
      "integrity": "sha512-...",
      "engines": {
        "node": ">=0.10.0"
      }
    }
  }
}
```

### npm ciの重要性

```bash
# ❌ CI/CDで npm install を使用（非推奨）
npm install
# - package-lock.jsonを更新する可能性がある
# - package.jsonとロックファイルの不一致を許容

# ✅ CI/CDでは npm ci を使用（推奨）
npm ci
# - package-lock.jsonを厳密に遵守
# - ロックファイルがなければエラー
# - node_modulesを削除してからインストール
# - 20〜40%高速
```

## yarn.lock（Yarn）

### 構造

```yaml
# yarn.lock

react@^18.2.0:
  version "18.2.0"
  resolved "https://registry.yarnpkg.com/react/-/react-18.2.0.tgz#..."
  integrity sha512-...
  dependencies:
    loose-envify "^1.1.0"

loose-envify@^1.1.0:
  version "1.4.0"
  resolved "https://registry.yarnpkg.com/loose-envify/-/loose-envify-1.4.0.tgz#..."
  integrity sha512-...
```

### Yarn決定論的インストール

```bash
# 決定論的インストール
yarn install --frozen-lockfile

# CI/CDで使用
yarn install --frozen-lockfile --production
```

## pnpm-lock.yaml（pnpm）

### 構造

```yaml
# pnpm-lock.yaml

lockfileVersion: '6.0'

dependencies:
  react:
    specifier: ^18.2.0
    version: 18.2.0

packages:

  /react@18.2.0:
    resolution: {integrity: sha512-...}
    engines: {node: '>=0.10.0'}
    dependencies:
      loose-envify: 1.4.0
    dev: false
```

### pnpm決定論的インストール

```bash
# 決定論的インストール
pnpm install --frozen-lockfile

# ストアの整合性も検証
pnpm install --frozen-lockfile --verify-store-integrity
```

## ロックファイルのベストプラクティス

### 1. 必ずGitにコミット

```bash
# ✅ ロックファイルをコミット
git add package-lock.json
git commit -m "chore: update dependencies"

# ❌ .gitignoreに追加しない
# package-lock.json  # これは絶対にダメ
```

### 2. CI/CDで決定論的インストールを使用

```yaml
# GitHub Actions
- name: Install dependencies
  run: npm ci  # ✅ npm ciを使用

# ❌ 以下は避ける
# run: npm install
```

### 3. ロックファイルのコンフリクト解決

```bash
# 方法1: 最新版を採用して再インストール
git checkout --theirs package-lock.json
npm install

# 方法2: マージツールを使用
npx npm-merge-driver install

# 方法3: 完全に再生成（最終手段）
rm package-lock.json
npm install
git add package-lock.json
```

### 4. 定期的な更新

```bash
# 週次または月次で更新
npm update
git add package-lock.json
git commit -m "chore: update lockfile"
```

## ロックファイル比較表

| 機能 | npm | Yarn | pnpm |
|------|-----|------|------|
| 決定論的 | ✅ | ✅ | ✅ |
| 整合性チェック | SHA-512 | SHA-1 | SHA-512 |
| サイズ | 大 | 中 | 中 |
| 人間可読性 | 低 | 高 | 高 |
| マージ容易性 | 低 | 中 | 中 |

## トラブルシューティング

### ロックファイルと package.jsonの不一致

```bash
# エラー例
npm ERR! Conflicting package.json and package-lock.json

# 解決方法
npm install  # package-lock.jsonを更新
```

### ロックファイルの破損

```bash
# ロックファイルを再生成
rm package-lock.json
npm install

# または
npm install --package-lock-only
```

### CI/CDでのロックファイルエラー

```bash
# npm ci が失敗する場合
# 1. ローカルでロックファイルを更新
npm install
git add package-lock.json
git commit -m "fix: update lockfile"
git push

# 2. CI/CDが成功することを確認
```

## まとめ

ロックファイル管理の重要ポイント：

1. **必ずGitにコミット**: チーム全体で同じバージョンを保証
2. **CI/CDでは決定論的インストール**: npm ci / yarn install --frozen-lockfile / pnpm install --frozen-lockfile
3. **定期的に更新**: セキュリティパッチを取得
4. **コンフリクトは慎重に解決**: 不一致は予期しない動作の原因

次章では、モノレポ管理について解説します。
