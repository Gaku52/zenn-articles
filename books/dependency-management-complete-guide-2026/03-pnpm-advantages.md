---
title: "pnpmの利点と実践"
---

# pnpmの利点と実践

## pnpmとは

pnpm (Performant npm) は、npmの代替となる高速で効率的なパッケージマネージャーです。Content-Addressable Storage（内容アドレス可能ストレージ）という革新的なアプローチにより、従来のパッケージマネージャーの課題を解決しています。

## pnpmの仕組み

### Content-Addressable Storage

pnpmは、すべてのパッケージをグローバルストレージに保存し、プロジェクトからハードリンクで参照します。

```
通常のnpm:
project-A/node_modules/lodash/  (100MB)
project-B/node_modules/lodash/  (100MB)
project-C/node_modules/lodash/  (100MB)
→ 合計 300MB

pnpm:
~/.pnpm-store/lodash@4.17.21/   (100MB)
  ↑ (ハードリンク)
  ├─ project-A/node_modules/.pnpm/lodash@4.17.21/
  ├─ project-B/node_modules/.pnpm/lodash@4.17.21/
  └─ project-C/node_modules/.pnpm/lodash@4.17.21/
→ 合計 100MB（実体は1つのみ）
```

### ディレクトリ構造

```
project/
├── node_modules/
│   ├── .pnpm/                    # 実際のパッケージ（ハードリンク）
│   │   ├── lodash@4.17.21/
│   │   └── react@18.2.0/
│   ├── lodash -> .pnpm/lodash@4.17.21/node_modules/lodash  (シンボリックリンク)
│   └── react -> .pnpm/react@18.2.0/node_modules/react
├── package.json
└── pnpm-lock.yaml
```

## pnpmのインストール

### インストール方法

```bash
# npm経由でインストール
npm install -g pnpm

# Homebrewでインストール（macOS）
brew install pnpm

# Standalone scriptでインストール（推奨）
curl -fsSL https://get.pnpm.io/install.sh | sh -

# バージョン確認
pnpm -v
```

### シェル補完の設定

```bash
# Bash
pnpm completion bash >> ~/.bashrc

# Zsh
pnpm completion zsh >> ~/.zshrc

# Fish
pnpm completion fish > ~/.config/fish/completions/pnpm.fish
```

## 基本コマンド

pnpmのコマンドは、npmとほぼ同じです。

```bash
# プロジェクト初期化
pnpm init

# 依存関係のインストール
pnpm install              # すべてインストール
pnpm install --frozen-lockfile  # CI用（ロックファイル固定）

# パッケージの追加
pnpm add lodash           # 依存関係に追加
pnpm add -D jest          # 開発依存に追加
pnpm add -g typescript    # グローバルに追加

# パッケージの削除
pnpm remove lodash
pnpm remove -D jest

# 更新
pnpm update               # すべて更新
pnpm update lodash        # 特定のパッケージのみ更新
pnpm outdated             # 古いパッケージを確認

# スクリプト実行
pnpm run build
pnpm test                 # "run"は省略可能
```

## npmとの違い

### Strict mode（厳密モード）

pnpmは、Phantom Dependencies（幽霊依存）を防止します。

```javascript
// package.jsonには"react"のみ定義
{
  "dependencies": {
    "react": "^18.2.0"
  }
}

// Reactは内部でobject-assignを使用

// ❌ npmの場合（動いてしまう）
import objectAssign from 'object-assign';
// Reactの依存としてインストールされているため、
// package.jsonに明記していなくても動作してしまう

// ✅ pnpmの場合（エラー）
import objectAssign from 'object-assign';
// Error: Cannot find module 'object-assign'
// → package.jsonに明示的に追加する必要がある
```

**対処法:**

```bash
# 必要なパッケージを明示的に追加
pnpm add object-assign
```

### インストール速度の違い

pnpm公式ベンチマーク[^1]に基づく比較（中規模プロジェクト想定）:

```
初回インストール:
  npm:  基準値
  pnpm: 約3倍高速

キャッシュあり:
  npm:  基準値
  pnpm: 約6倍高速

node_modulesあり（再インストール）:
  npm:  基準値
  pnpm: 約8倍高速
```

[^1]: pnpm公式ベンチマーク: https://pnpm.io/benchmarks

## .npmrcの設定

pnpmも`.npmrc`ファイルで設定をカスタマイズできます。

```bash
# .npmrc

# pnpm固有設定
store-dir=~/.pnpm-store
verify-store-integrity=true
package-import-method=hardlink

# Phantom Dependenciesを厳密にチェック
shamefully-hoist=false

# モノレポ設定
shared-workspace-lockfile=true
link-workspace-packages=true

# パフォーマンス
network-concurrency=16
fetch-retries=3

# 共通設定（npmと同じ）
save-exact=true
engine-strict=true
```

### 推奨設定

```bash
# プロジェクトルートの.npmrc
shamefully-hoist=false           # 厳密モード（推奨）
shared-workspace-lockfile=true   # モノレポ用
link-workspace-packages=true     # ワークスペース間リンク
save-exact=true                  # 正確なバージョンで保存
```

## pnpmの利点

### 1. ディスク容量の節約

```bash
# 実例：3つのプロジェクトで同じパッケージを使用

npm使用時:
  project-1/node_modules: 500MB
  project-2/node_modules: 500MB
  project-3/node_modules: 500MB
  合計: 1,500MB

pnpm使用時:
  ~/.pnpm-store: 550MB
  project-1/node_modules: リンクのみ
  project-2/node_modules: リンクのみ
  project-3/node_modules: リンクのみ
  合計: 550MB（約63%削減）
```

### 2. インストール速度

pnpmが高速な理由:

1. **並列ダウンロード**: 複数パッケージを同時ダウンロード
2. **ハードリンク**: ファイルコピー不要
3. **Content-Addressable Storage**: 重複チェックが高速

### 3. セキュリティ

厳密な依存関係解決により、意図しないパッケージへのアクセスを防止:

```javascript
// package.jsonに記載のないパッケージは使用不可
// → 依存関係が明確化される
// → セキュリティリスクが低減
```

## モノレポサポート

pnpmは、モノレポ（複数パッケージを1つのリポジトリで管理）に優れたサポートを提供します。

### pnpm-workspace.yaml

```yaml
# pnpm-workspace.yaml
packages:
  - 'packages/*'
  - 'apps/*'
  - '!**/test/**'  # テストディレクトリは除外
```

### ディレクトリ構造

```
my-monorepo/
├── pnpm-workspace.yaml
├── package.json
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
# すべてのワークスペースに対して実行
pnpm -r install          # すべてをインストール
pnpm -r build            # すべてをビルド
pnpm -r test             # すべてをテスト

# 特定のワークスペースに対して実行
pnpm --filter ui-components build
pnpm --filter web dev

# 依存関係のあるワークスペースのみ実行
pnpm --filter "...[origin/main]" test  # 変更されたパッケージのみ

# ワークスペース間の依存関係を追加
cd apps/web
pnpm add ui-components --workspace
# または
pnpm add @mycompany/ui-components --workspace
```

## CI/CDでのpnpm使用

### GitHub Actions

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      # pnpmのインストール
      - uses: pnpm/action-setup@v2
        with:
          version: 8

      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'pnpm'

      # 依存関係のインストール
      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Run tests
        run: pnpm test

      - name: Build
        run: pnpm build
```

## トラブルシューティング

### ストアの破損

```bash
# ストアの整合性を検証
pnpm store status

# ストアを修復
pnpm store prune

# ストアを完全にクリア（注意: 全プロジェクトに影響）
rm -rf ~/.pnpm-store
```

### Phantom Dependencyエラー

```
ERROR: Cannot find module 'some-package'
```

**解決方法:**

```bash
# 1. package.jsonに明示的に追加
pnpm add some-package

# 2. または、一時的にshamefully-hoistを有効化（非推奨）
# .npmrc
shamefully-hoist=true
```

### npmからpnpmへの移行

```bash
# 1. pnpmをインストール
npm install -g pnpm

# 2. 既存のnode_modulesを削除
rm -rf node_modules

# 3. package-lock.jsonを削除（pnpmは独自のロックファイルを使用）
rm package-lock.json

# 4. pnpmでインストール
pnpm install

# 5. .gitignoreを更新
echo "pnpm-lock.yaml" >> .gitignore

# 6. CIスクリプトを更新
# npm ci → pnpm install --frozen-lockfile
```

## まとめ

pnpmは、以下の点で優れた選択肢です。

**メリット:**
- ディスク容量を60〜70%削減
- インストール速度が3〜8倍高速
- Phantom Dependenciesを防止（厳密性）
- モノレポサポートが強力

**推奨される使用場面:**
- 新規プロジェクト
- モノレポ構成
- CI/CD実行時間を短縮したい場合
- 複数プロジェクトを同時に開発する環境

**注意点:**
- npmより採用率は低い（チーム教育が必要）
- 一部ツールとの互換性問題（稀）
- Phantom Dependencyエラーへの対応が必要

次章では、Yarnの特徴と使い方を解説します。
