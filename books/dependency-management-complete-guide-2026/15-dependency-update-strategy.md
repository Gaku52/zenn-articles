---
title: "依存関係更新戦略"
---

# 依存関係更新戦略

## 更新の重要性

依存関係を定期的に更新することで、以下のメリットがあります。

- セキュリティパッチの適用
- バグ修正の取得
- 新機能の利用
- 技術的負債の削減

## 更新頻度マトリクス

| 更新タイプ | 頻度 | 自動化 | レビュー |
|-----------|------|--------|---------|
| セキュリティパッチ | 即座 | ✅ 自動マージ | 最小限 |
| Patch (x.x.PATCH) | 週次 | ✅ 自動マージ | CIのみ |
| Minor (x.MINOR.x) | 月次 | ⚠️ 手動マージ | コードレビュー |
| Major (MAJOR.x.x) | 四半期 | ❌ 手動のみ | 完全レビュー |

## セマンティックバージョニング（SemVer）

セマンティックバージョニング（[Semantic Versioning](https://semver.org/)）は、バージョン番号とその変更内容を体系的に管理する仕様です。

### バージョン番号の構造

```
MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD]
  1  .  2  .  3  -  alpha.1  + 20260125

MAJOR (メジャーバージョン): 破壊的変更
MINOR (マイナーバージョン): 後方互換性のある新機能
PATCH (パッチバージョン): 後方互換性のあるバグ修正
PRERELEASE (プレリリース): alpha、beta、rc など
BUILD (ビルドメタデータ): ビルド番号、コミットハッシュなど
```

### セマンティックバージョニングのルール

[Semantic Versioning 2.0.0](https://semver.org/)の仕様によると、以下のルールが定義されています。

**MAJOR バージョン（破壊的変更）**:
- 既存APIの削除
- 既存APIの引数変更
- 既存の動作の変更

```typescript
// v1.x.x
function fetchUser(id: number): Promise<User>

// v2.0.0 (MAJOR: 引数の型が変更)
function fetchUser(id: string): Promise<User>
```

**MINOR バージョン（後方互換性のある新機能）**:
- 新しいAPIの追加
- 既存APIへのオプショナルパラメータ追加
- 非推奨マーク（deprecation）

```typescript
// v1.2.x
function fetchUser(id: number, options?: FetchOptions): Promise<User>
//                              ↑ オプショナルパラメータの追加はMINOR
```

**PATCH バージョン（後方互換性のあるバグ修正）**:
- バグ修正
- パフォーマンス改善
- 内部実装の変更

```typescript
// v1.2.3 → v1.2.4
// 内部実装のバグ修正（APIは変更なし）
```

### バージョン更新例とリスク評価

```bash
# Patch更新（低リスク）
lodash: 4.17.20 → 4.17.21
リスク: ⭐☆☆☆☆ (最低)
影響: バグ修正のみ、破壊的変更なし
推奨対応: 自動マージ

# Minor更新（中リスク）
react: 18.2.0 → 18.3.0
リスク: ⭐⭐☆☆☆ (低〜中)
影響: 新機能追加、既存APIは維持
推奨対応: テスト後マージ

# Major更新（高リスク）
next: 13.5.6 → 14.0.0
リスク: ⭐⭐⭐⭐☆ (高)
影響: 破壊的変更、コード修正が必要
推奨対応: 移行ガイド確認、段階的移行
```

### プレリリースバージョン

```bash
# プレリリースの順序（安定度が上がる順）
1.0.0-alpha.1    # 初期開発版
1.0.0-beta.1     # 機能凍結、バグ修正のみ
1.0.0-rc.1       # リリース候補（Release Candidate）
1.0.0            # 正式リリース

# 本番環境では使用しない
npm install package@next     # ❌ 最新プレリリース
npm install package@latest   # ✅ 最新安定版
```

## 安全な更新プロセス

### Step 1: 古いパッケージの確認

```bash
npm outdated

# 出力例:
# Package      Current  Wanted  Latest  Location
# react        18.2.0   18.2.0  18.3.1  project
# lodash       4.17.20  4.17.21  4.17.21  project
# typescript   5.0.4    5.0.4   5.3.3   project
```

### Step 2: 変更内容の確認

```bash
# Changelogを確認
npm repo react

# または
npm info react versions
npm view react@18.3.1
```

### Step 3: テスト環境で更新

```bash
# ブランチ作成
git checkout -b chore/update-dependencies

# 更新
npm update lodash

# または特定バージョンに更新
npm install react@18.3.1

# テスト
npm test
npm run build
```

### Step 4: ステージング環境で検証

```bash
# ステージングにデプロイ
./deploy.sh staging

# スモークテスト
./smoke-test.sh

# 24時間監視
# - エラーレート
# - パフォーマンス
# - ユーザーフィードバック
```

### Step 5: 本番環境にデプロイ

```bash
# 本番デプロイ
./deploy.sh production

# 監視（48時間）
# - エラー追跡
# - パフォーマンス監視

# ロールバック計画
git revert <commit>
./deploy.sh production
```

## 自動更新ツール

### Renovate設定

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  
  "packageRules": [
    {
      "matchUpdateTypes": ["patch"],
      "automerge": true,
      "minimumReleaseAge": "3 days"
    },
    {
      "matchUpdateTypes": ["minor"],
      "matchDepTypes": ["devDependencies"],
      "automerge": true,
      "minimumReleaseAge": "7 days"
    },
    {
      "matchUpdateTypes": ["major"],
      "enabled": false,
      "schedule": ["on the 1st day of the month"]
    }
  ],
  
  "vulnerabilityAlerts": {
    "enabled": true,
    "automerge": true,
    "minimumReleaseAge": "0"
  }
}
```

## テスト戦略

依存関係更新時には、包括的なテストが不可欠です。

### テストレベルと実施基準

| テストレベル | Patch更新 | Minor更新 | Major更新 |
|------------|----------|----------|----------|
| Unit Test | ✅ 必須 | ✅ 必須 | ✅ 必須 |
| Integration Test | ⚠️ 推奨 | ✅ 必須 | ✅ 必須 |
| E2E Test | △ 選択的 | ✅ 必須 | ✅ 必須 |
| Manual Test | ❌ 不要 | ⚠️ 推奨 | ✅ 必須 |
| Performance Test | ❌ 不要 | △ 選択的 | ✅ 必須 |
| Security Scan | ✅ 必須 | ✅ 必須 | ✅ 必須 |

### 自動テストの設定例

```yaml
# GitHub Actions - 依存関係更新時のテスト
name: Dependency Update Test
on:
  pull_request:
    paths:
      - 'package.json'
      - 'package-lock.json'
      - 'pnpm-lock.yaml'
      - 'yarn.lock'

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      # ユニットテスト
      - name: Run unit tests
        run: npm test

      # 統合テスト
      - name: Run integration tests
        run: npm run test:integration

      # E2Eテスト
      - name: Run E2E tests
        run: npm run test:e2e

      # バンドルサイズチェック
      - name: Check bundle size
        run: npm run build && npx size-limit

      # セキュリティ監査
      - name: Run security audit
        run: npm audit --audit-level=moderate
```

### パフォーマンステスト

依存関係更新後、パフォーマンスが劣化していないか確認します。

```javascript
// performance.test.js
import { performance } from 'perf_hooks';

describe('Performance Tests', () => {
  it('should complete within acceptable time', () => {
    const start = performance.now();

    // テスト対象の処理
    executeFunction();

    const end = performance.now();
    const duration = end - start;

    // 期待される実行時間: 100ms以内
    expect(duration).toBeLessThan(100);
  });
});
```

### Lighthouse CI での自動チェック

```yaml
# .github/workflows/lighthouse-ci.yml
- name: Run Lighthouse CI
  uses: treosh/lighthouse-ci-action@v10
  with:
    urls: |
      https://staging.example.com
    budgetPath: ./budget.json
    uploadArtifacts: true
```

budget.json（パフォーマンス予算）:

```json
[
  {
    "path": "/*",
    "timings": [
      {
        "metric": "interactive",
        "budget": 3000
      },
      {
        "metric": "first-contentful-paint",
        "budget": 1000
      }
    ],
    "resourceSizes": [
      {
        "resourceType": "script",
        "budget": 300
      },
      {
        "resourceType": "total",
        "budget": 500
      }
    ]
  }
]
```

## ロールバック手順

依存関係更新後に問題が発生した場合の、迅速なロールバック手順を準備しておきます。

### Git による即座のロールバック

```bash
# 1. 直前のコミットをrevert（推奨）
git revert HEAD
git push origin main

# 2. 特定のコミットをrevert
git revert <commit-hash>
git push origin main

# 3. 緊急時のforce push（本番環境では避ける）
git reset --hard HEAD~1
git push --force origin main  # ⚠️ 危険: 他の開発者に影響
```

### package.jsonとロックファイルの手動ロールバック

```bash
# 1. 特定のバージョンに戻す
npm install react@18.2.0 --save-exact

# 2. ロックファイルを更新
npm install

# 3. 動作確認
npm test

# 4. コミット
git add package.json package-lock.json
git commit -m "revert: rollback react to 18.2.0"
git push origin main
```

### ブルーグリーンデプロイメントによるリスク低減

本番環境へのリスクを最小化するため、ブルーグリーンデプロイメントを活用します。

```yaml
# デプロイ戦略の例
deploy:
  strategy:
    # ステップ1: 新バージョンを一部のサーバーにデプロイ（Green）
    - name: Deploy to 10% of servers
      percentage: 10
      duration: 1h
      rollback_on_error: true

    # ステップ2: 問題なければ50%に拡大
    - name: Deploy to 50% of servers
      percentage: 50
      duration: 2h
      rollback_on_error: true

    # ステップ3: 全サーバーにデプロイ
    - name: Deploy to all servers
      percentage: 100
```

### Feature Flag による段階的ロールアウト

```typescript
// feature-flags.ts
export const featureFlags = {
  useNewDependencyVersion: process.env.USE_NEW_VERSION === 'true',
};

// app.ts
import { featureFlags } from './feature-flags';

if (featureFlags.useNewDependencyVersion) {
  // 新バージョンの依存関係を使用
  import('new-library').then(lib => lib.doSomething());
} else {
  // 旧バージョンの依存関係を使用
  import('old-library').then(lib => lib.doSomething());
}
```

環境変数での制御:

```bash
# 開発環境: 新バージョンを使用
USE_NEW_VERSION=true npm start

# 本番環境: 旧バージョンを使用（安全確認後に切り替え）
USE_NEW_VERSION=false npm start
```

## 破壊的変更への対応

### 移行ガイドの確認手順

Major バージョン更新時は、必ず公式の移行ガイドを確認します。

```bash
# 1. CHANGELOGを確認
npm repo react  # ブラウザでリポジトリを開く
# CHANGELOG.md や UPGRADING.md を読む

# 2. Breaking Changes を検索
# GitHubのReleasesページで "breaking" を検索

# 3. 移行ガイドを読む
# 例: React 17 → 18
# https://react.dev/blog/2022/03/08/react-18-upgrade-guide
```

主要ライブラリの移行ガイド:
- React: [Upgrade Guide](https://react.dev/learn/upgrade-guide)
- Next.js: [Upgrade Guide](https://nextjs.org/docs/upgrading)
- Vue: [Migration Guide](https://v3-migration.vuejs.org/)
- Angular: [Update Guide](https://update.angular.io/)

### 段階的移行の実施

```bash
# React 17 → 18の移行例

# Step 1: TypeScript定義を更新
npm install --save-dev @types/react@18 @types/react-dom@18

# Step 2: TypeScriptエラーを確認
npm run type-check

# Step 3: Reactを更新
npm install react@18 react-dom@18

# Step 4: テストを実行
npm test

# Step 5: 手動テスト
npm run dev

# Step 6: コミット
git add .
git commit -m "feat: upgrade React to v18"
```

### Codemod の活用

多くの主要ライブラリは、自動マイグレーションツール（Codemod）を提供しています。

```bash
# React Codemod
npx react-codemod update-react-imports

# Next.js Codemod
npx @next/codemod@latest new-link ./pages

# Jest Codemod
npx jest-codemods
```

## 更新スケジュールの策定

組織全体で一貫した更新スケジュールを策定することで、計画的な依存関係管理が可能になります。

### 推奨更新スケジュール

| 更新タイプ | 頻度 | 実施タイミング | 担当 |
|-----------|------|--------------|------|
| セキュリティパッチ | 即座 | 脆弱性発見後24時間以内 | セキュリティチーム |
| Patch 更新 | 週次 | 毎週月曜日 | 開発チーム |
| Minor 更新 | 月次 | 毎月第1週 | テックリード |
| Major 更新 | 四半期 | 四半期初月 | アーキテクト |
| 全体レビュー | 年次 | 年度初め | 全チーム |

### 更新カレンダーの例

```
2026年 依存関係更新スケジュール

Q1 (1-3月)
- 1月: React 19メジャー更新（破壊的変更あり）
- 2月: 定期マイナー更新
- 3月: セキュリティパッチのみ

Q2 (4-6月)
- 4月: Next.js 15メジャー更新
- 5月: 定期マイナー更新
- 6月: Node.js LTS更新

Q3 (7-9月)
- 7月: TypeScript 6.0メジャー更新
- 8月: 定期マイナー更新
- 9月: セキュリティパッチのみ

Q4 (10-12月)
- 10月: 全依存関係レビュー
- 11月: 定期マイナー更新
- 12月: 年末コードフリーズ（セキュリティパッチのみ）
```

## 依存関係更新のリスク評価フレームワーク

更新前にリスクを評価することで、適切な対応策を講じることができます。

### リスク評価マトリクス

| 要素 | 低リスク | 中リスク | 高リスク |
|-----|---------|---------|---------|
| バージョン変更 | Patch (x.x.1→x.x.2) | Minor (x.1.0→x.2.0) | Major (1.x.x→2.0.0) |
| パッケージ影響範囲 | devDependencies | dependencies（ライブラリ） | dependencies（フレームワーク） |
| 使用箇所 | 1〜5箇所 | 6〜20箇所 | 21箇所以上 |
| テストカバレッジ | 80%以上 | 50〜80% | 50%未満 |
| 破壊的変更 | なし | 非推奨警告あり | Breaking Changes明記 |

### リスクレベル別の対応方針

```bash
# 低リスク（合計スコア3以下）
# - 自動マージ可能
# - CI/CDテスト通過後、即座にマージ

# 中リスク（合計スコア4〜7）
# - 手動レビュー必要
# - ステージング環境での検証
# - 24時間監視後、本番デプロイ

# 高リスク（合計スコア8以上）
# - 詳細な移行計画策定
# - 段階的ロールアウト（Canaryデプロイ）
# - 48〜72時間の監視期間
```

### 更新前チェックリスト

依存関係更新前に、以下の項目を確認します。

```markdown
## 更新前チェックリスト

### 1. 調査フェーズ
- [ ] CHANGELOGで変更内容を確認
- [ ] Breaking Changesの有無を確認
- [ ] 既知の問題（GitHub Issues）を確認
- [ ] コミュニティの反応を確認（Twitter、Reddit等）
- [ ] セキュリティアドバイザリを確認

### 2. 準備フェーズ
- [ ] ブランチを作成（例: chore/update-react-18.3.0）
- [ ] ロールバック手順を準備
- [ ] 監視ダッシュボードを準備
- [ ] 関係者に更新予定を通知

### 3. 実行フェーズ
- [ ] ローカル環境で更新
- [ ] ユニットテスト実行
- [ ] 統合テスト実行
- [ ] E2Eテスト実行
- [ ] ステージング環境にデプロイ
- [ ] スモークテスト実行

### 4. 検証フェーズ
- [ ] パフォーマンス計測（Lighthouse、WebPageTest等）
- [ ] エラー監視（Sentry、Bugsnag等）
- [ ] ログ監視（Datadog、Splunk等）
- [ ] ユーザーフィードバック確認

### 5. デプロイフェーズ
- [ ] 本番環境にデプロイ
- [ ] 即座にエラーレートを確認
- [ ] 24時間監視継続
- [ ] 問題なければクローズ
```

## 依存関係更新の自動化レベル

プロジェクトの成熟度に応じて、自動化レベルを段階的に上げていきます。

### レベル1: 手動更新（初期段階）

```bash
# 全て手動で実施
npm outdated
npm update package-name
npm test
git commit -m "chore: update package-name"
```

適用シーン:
- プロジェクト開始直後
- テストカバレッジが低い（50%未満）
- チーム規模が小さい（1〜3人）

### レベル2: 半自動更新（成長段階）

```json
// Renovate設定例
{
  "extends": ["config:base"],
  "packageRules": [
    {
      "matchUpdateTypes": ["patch"],
      "automerge": true,
      "minimumReleaseAge": "3 days"
    }
  ]
}
```

適用シーン:
- テストカバレッジが中程度（50〜80%）
- CI/CDパイプラインが整備済み
- チーム規模が中程度（4〜10人）

### レベル3: 完全自動更新（成熟段階）

```yaml
# GitHub Actions例
name: Auto Merge Dependabot

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  auto-merge:
    runs-on: ubuntu-latest
    if: github.actor == 'dependabot[bot]'

    steps:
      - uses: actions/checkout@v4

      - name: Check if patch update
        id: check
        run: |
          if [[ "${{ github.event.pull_request.title }}" =~ "patch" ]]; then
            echo "is_patch=true" >> $GITHUB_OUTPUT
          fi

      - name: Auto approve
        if: steps.check.outputs.is_patch == 'true'
        run: gh pr review --approve "${{ github.event.pull_request.number }}"
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Auto merge
        if: steps.check.outputs.is_patch == 'true'
        run: gh pr merge --auto --squash "${{ github.event.pull_request.number }}"
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

適用シーン:
- テストカバレッジが高い（80%以上）
- モニタリング体制が完備
- チーム規模が大きい（11人以上）

## Major バージョン更新の詳細プロセス

Major バージョン更新は、最も慎重に行う必要があります。

### Step 1: 影響範囲の調査（1〜2日）

```bash
# 1. 依存関係ツリーを確認
npm ls package-name

# 2. コード内の使用箇所を検索
git grep "import.*from.*'package-name'"
git grep "require.*'package-name'"

# 3. 型定義の変更を確認
# TypeScriptの場合
npx tsc --noEmit  # 型エラーをチェック
```

### Step 2: 移行計画の策定（1日）

```markdown
## React 17 → 18 移行計画（例）

### 概要
- 目的: React 18の新機能（Concurrent Rendering）を活用
- 期間: 2026年2月1日〜2月15日（2週間）
- 担当: フロントエンドチーム

### フェーズ1: 準備（2日）
- [ ] React 18移行ガイドを読む
- [ ] 破壊的変更をリストアップ
- [ ] テストケースを追加（カバレッジ80%以上に）

### フェーズ2: 実装（5日）
- [ ] React 18をインストール
- [ ] 型定義を更新
- [ ] 非推奨APIを置き換え
  - ReactDOM.render → createRoot
  - unmountComponentAtNode → root.unmount
- [ ] 新機能を段階的に導入
  - Automatic Batching（自動バッチング）
  - Transitions API（優先度付きレンダリング）

### フェーズ3: テスト（5日）
- [ ] ユニットテスト
- [ ] E2Eテスト
- [ ] パフォーマンステスト
- [ ] ブラウザ互換性テスト

### フェーズ4: デプロイ（3日）
- [ ] ステージング環境（1日）
- [ ] Canaryデプロイ 10%（1日）
- [ ] 全環境（1日）
```

### Step 3: 段階的移行の実施（1〜2週間）

```typescript
// 新旧API共存パターン
import React from 'react';

// React 17互換モード
if (process.env.REACT_VERSION === '17') {
  ReactDOM.render(<App />, document.getElementById('root'));
}

// React 18新API
else {
  const root = ReactDOM.createRoot(document.getElementById('root')!);
  root.render(<App />);
}
```

### Step 4: 監視とロールバック準備（デプロイ後48時間）

```yaml
# 監視項目
monitoring:
  # パフォーマンス
  - metric: "page_load_time"
    threshold: "+10%"  # 10%以上悪化したらアラート

  # エラーレート
  - metric: "error_rate"
    threshold: "+5%"   # 5%以上増加したらアラート

  # ユーザー行動
  - metric: "bounce_rate"
    threshold: "+15%"  # 15%以上増加したらアラート

# ロールバックトリガー
rollback_triggers:
  - "error_rate > baseline * 1.5"  # エラー率が1.5倍
  - "critical_bug_reported"         # 致命的バグ報告
  - "manual_trigger"                # 手動トリガー
```

## iOS依存関係の更新戦略

Swift Package Manager、CocoaPodsでの更新戦略は、npm系とは異なる考慮が必要です。

### Swift Package Managerの更新

```bash
# 1. 更新可能なパッケージを確認
swift package show-dependencies

# 2. 特定パッケージのみ更新
swift package update Alamofire

# 3. 全パッケージ更新
swift package update

# 4. Package.resolvedの差分確認
git diff Package.resolved

# 5. ビルドとテスト
swift build
swift test

# 6. Xcodeプロジェクトで確認
xcodebuild -scheme MyApp -destination 'platform=iOS Simulator,name=iPhone 15' test
```

### CocoaPodsの更新

```bash
# 1. 更新可能なPodを確認
pod outdated

# 2. Podfileで許可された範囲で更新
pod update

# 3. 特定Podのみ更新
pod update Alamofire

# 4. Podfile.lockの差分確認
git diff Podfile.lock

# 5. ビルドとテスト
xcodebuild -workspace MyApp.xcworkspace -scheme MyApp test
```

### プラットフォーム固有の注意点

```ruby
# Podfile - iOS最小バージョンとの互換性
platform :ios, '15.0'

target 'MyApp' do
  use_frameworks!

  # ✅ バージョン範囲を明示
  pod 'Alamofire', '~> 5.8'

  # ⚠️ 最新バージョンが対象iOSバージョンをサポートしているか確認
  pod 'Kingfisher', '~> 7.10'
end

# 更新後、必ず以下を確認
# - サポート対象iOSバージョンとの互換性
# - Bitcodeサポート（必要な場合）
# - XCFramework対応状況
```

## 更新失敗時のトラブルシューティング

### 一般的な失敗パターンと対処法

```bash
# パターン1: テスト失敗
# 原因: APIの変更
# 対処: テストコードを更新、または更新を延期

# パターン2: ビルドエラー
# 原因: 破壊的変更
# 対処: コードを修正、または互換レイヤーを追加

# パターン3: 実行時エラー
# 原因: ランタイム動作の変更
# 対処: ロールバック、またはポリフィル追加

# パターン4: パフォーマンス劣化
# 原因: 新バージョンの最適化不足
# 対処: Issue報告、または旧バージョンに戻す
```

### ロールバックの判断基準

即座にロールバックすべき状況:

```yaml
critical_issues:
  - "本番環境でのクラッシュ率 > 1%"
  - "主要機能が動作しない"
  - "セキュリティ脆弱性が発見された（新バージョンで）"
  - "データ損失の可能性"

investigate_issues:
  - "エラーレート増加 < 10%"
  - "パフォーマンス劣化 < 20%"
  - "一部ユーザーのみ影響"

acceptable_issues:
  - "マイナーなUI崩れ"
  - "ログに警告が出る（動作は正常）"
  - "非推奨警告"
```

## まとめ

計画的な更新戦略により、セキュリティと安定性を両立できます。

**重要なポイント**:
1. **セマンティックバージョニングの理解**: MAJOR、MINOR、PATCHの違いを把握
2. **リスク評価**: 更新前に影響範囲とリスクレベルを評価
3. **包括的なテスト**: 自動テスト + 手動テストの組み合わせ
4. **段階的デプロイ**: ブルーグリーンデプロイメント、Canaryデプロイの活用
5. **監視とロールバック**: 問題発生時の迅速な対応体制
6. **定期的な更新**: スケジュールを策定し、技術的負債を防止
7. **自動化の段階的導入**: プロジェクト成熟度に応じた自動化レベル

**更新頻度の推奨スケジュール**:
- セキュリティパッチ: 即座（24時間以内）
- Patch更新: 週次
- Minor更新: 月次
- Major更新: 四半期ごと、または年次

**参考文献**:
- [Semantic Versioning 2.0.0](https://semver.org/)
- [npm - Semantic versioning and npm](https://docs.npmjs.com/about-semantic-versioning)
- [GitHub - Dependabot](https://docs.github.com/en/code-security/dependabot)
- [Renovate Bot Documentation](https://docs.renovatebot.com/)
- [Martin Fowler - BlueGreenDeployment](https://martinfowler.com/bliki/BlueGreenDeployment.html)
- [Google SRE Book - Release Engineering](https://sre.google/sre-book/release-engineering/)

次章では、ライセンス管理について解説します。
