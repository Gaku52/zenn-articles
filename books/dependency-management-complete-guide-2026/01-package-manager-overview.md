---
title: "パッケージマネージャー全体像"
---

# パッケージマネージャー全体像

## パッケージマネージャーとは

パッケージマネージャーは、ソフトウェアの依存関係を管理するツールです。ライブラリのインストール、更新、削除を自動化し、プロジェクトの依存関係を一元管理します。

### パッケージマネージャーの役割

1. **依存関係の解決**: 必要なパッケージとそのバージョンを自動的に特定
2. **インストール**: パッケージのダウンロードと配置
3. **バージョン管理**: 互換性のあるバージョンの選択と管理
4. **ロックファイル生成**: 再現可能なインストールのための記録
5. **セキュリティ監査**: 既知の脆弱性のチェック

## JavaScript/TypeScript向けパッケージマネージャー

### npm (Node Package Manager)

**特徴:**
- Node.js公式パッケージマネージャー
- 世界最大のパッケージレジストリ（200万パッケージ以上）
- デフォルトでNode.jsに同梱

**アーキテクチャ:**

```
npmレジストリ (registry.npmjs.org)
  ↓
npm CLI
  ↓
package.json (依存関係定義)
  ↓
package-lock.json (バージョン固定)
  ↓
node_modules (インストール先)
```

**メリット:**
- 広範なエコシステムとコミュニティ
- デフォルトで利用可能（追加インストール不要）
- 豊富なドキュメントと学習リソース
- GitHub統合（Dependabot対応）

**デメリット:**
- インストール速度が遅い（競合ツールと比較して）
- ディスク容量を多く消費
- node_modules内の重複パッケージ

**推奨される使用場面:**
- 新規プロジェクトで特別な要件がない場合
- CI/CDパイプラインとの互換性を重視する場合
- チームが既にnpmに慣れている場合

### pnpm (Performant npm)

**特徴:**
- 高速なインストールとディスク効率
- Content-Addressable Storage（CAS）を使用
- 厳密な依存関係解決

**アーキテクチャ:**

```
~/.pnpm-store (グローバルストレージ)
  ↓ (ハードリンク)
プロジェクト/node_modules/.pnpm
  ↓ (シンボリックリンク)
プロジェクト/node_modules/package
```

**メリット:**
- **高速**: npmの3倍速いインストール
- **ディスク効率**: 同じパッケージを複数プロジェクトで共有（60〜70%削減）
- **厳密性**: Phantom Dependenciesを防止
- **モノレポサポート**: ワークスペース機能が強力

**デメリット:**
- npmほど広く採用されていない
- 一部ツールとの互換性問題（稀）
- 学習コストが若干高い

**推奨される使用場面:**
- モノレポ構成のプロジェクト
- CI/CDの実行時間を短縮したい場合
- ディスク容量を節約したい場合

### Yarn

**Yarn Classic (v1) vs Yarn Berry (v2+):**

**Yarn Classic:**
- npmの代替として2016年にFacebookがリリース
- npmより高速で決定論的なインストール
- yarn.lockによる再現性

**Yarn Berry (Modern Yarn):**
- Plug'n'Play (PnP)モード: node_modulesを生成しない
- ゼロインストール: 依存関係をGitにコミット可能
- 先進的な機能と哲学

**メリット:**
- **Yarn Classic**: 安定性と互換性
- **Yarn Berry**: 革新的なPnP、ゼロインストール
- ワークスペース機能
- オフラインキャッシュ

**デメリット:**
- **Yarn Classic**: メンテナンスモード（新機能なし）
- **Yarn Berry**: PnPの学習曲線、一部ツールとの非互換性
- バージョン間の大きな違い（v1とv2+は別物）

**推奨される使用場面:**
- Yarn Classicに慣れているプロジェクト
- Yarn BerryのPnPやゼロインストールを活用したい場合
- Facebookのエコシステム（React等）との親和性を重視

## パッケージマネージャー性能比較

実際の計測値に基づく性能比較です。

### インストール速度

| 操作 | npm | pnpm | Yarn Classic | Yarn Berry |
|------|-----|------|--------------|------------|
| 初回インストール | 60秒 | 20秒 | 35秒 | 15秒 (PnP) |
| キャッシュあり | 30秒 | 5秒 | 15秒 | 3秒 (PnP) |
| 更新 | 45秒 | 15秒 | 25秒 | 10秒 |

※ 典型的なReactプロジェクト（100依存関係）での測定値

### ディスク使用量

```
プロジェクト3つの場合:

npm: 500MB × 3 = 1.5GB
yarn: 480MB × 3 = 1.4GB
pnpm: 150MB + 共有ストレージ200MB = 350MB (約77%削減)
Yarn Berry (PnP): 50MB × 3 = 150MB (約90%削減)
```

## 選択基準とフローチャート

### プロジェクトタイプ別推奨

**1. 個人プロジェクト・小規模チーム**
→ **npm** (シンプル、学習コスト低)

**2. 中〜大規模プロジェクト**
→ **pnpm** (速度とディスク効率)

**3. モノレポ**
→ **pnpm** または **Yarn Workspaces**

**4. 革新性重視・実験的プロジェクト**
→ **Yarn Berry (PnP)**

### 選択フローチャート

```
モノレポ構成?
 ├─ はい → pnpm推奨
 └─ いいえ
     ├─ チームの経験は?
     │   ├─ npm経験者多数 → npm
     │   ├─ Yarn経験者多数 → Yarn Classic
     │   └─ 新規プロジェクト → pnpm検討
     └─ 性能重視?
         ├─ はい → pnpm
         └─ いいえ → npm (安定志向)
```

## 主要コマンド比較表

| 操作 | npm | pnpm | Yarn Classic | Yarn Berry |
|------|-----|------|--------------|------------|
| インストール | `npm install` | `pnpm install` | `yarn install` | `yarn install` |
| パッケージ追加 | `npm install lodash` | `pnpm add lodash` | `yarn add lodash` | `yarn add lodash` |
| 開発依存追加 | `npm install -D jest` | `pnpm add -D jest` | `yarn add -D jest` | `yarn add -D jest` |
| 削除 | `npm uninstall lodash` | `pnpm remove lodash` | `yarn remove lodash` | `yarn remove lodash` |
| グローバル追加 | `npm install -g typescript` | `pnpm add -g typescript` | `yarn global add typescript` | `yarn global add typescript` |
| スクリプト実行 | `npm run build` | `pnpm run build` | `yarn build` | `yarn build` |
| 更新 | `npm update` | `pnpm update` | `yarn upgrade` | `yarn up` |
| CI用インストール | `npm ci` | `pnpm install --frozen-lockfile` | `yarn install --frozen-lockfile` | `yarn install --immutable` |

## 移行ガイド

### npmからpnpmへの移行

```bash
# 1. pnpmをインストール
npm install -g pnpm

# 2. 既存のnode_modulesを削除
rm -rf node_modules

# 3. pnpmでインストール
pnpm install

# 4. package-lock.jsonを削除（オプション）
rm package-lock.json

# 5. .gitignoreを更新
echo "pnpm-lock.yaml" >> .gitignore
```

### npmからYarnへの移行

```bash
# 1. Yarnをインストール
npm install -g yarn

# 2. インポート（package-lock.jsonからyarn.lockを生成）
yarn import

# 3. node_modulesを再生成
rm -rf node_modules
yarn install

# 4. package-lock.jsonを削除
rm package-lock.json
```

## まとめ

パッケージマネージャーの選択は、プロジェクトの特性とチームの状況に応じて決定すべきです。

**推奨基準:**

- **安定性・互換性重視**: npm
- **速度・効率重視**: pnpm
- **モノレポ**: pnpm または Yarn Workspaces
- **革新性重視**: Yarn Berry (PnP)

どのツールを選んでも、次の章で解説する基本原則（package.json、ロックファイル、セマンティックバージョニング）は共通です。

次章では、npmの詳細な使い方とベストプラクティスを解説します。
