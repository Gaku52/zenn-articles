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

## セマンティックバージョニング

```
MAJOR.MINOR.PATCH
  1  .  2  .  3

MAJOR: 破壊的変更
MINOR: 後方互換性のある新機能
PATCH: 後方互換性のあるバグ修正
```

### バージョン更新例

```bash
# Patch更新（安全）
lodash: 4.17.20 → 4.17.21

# Minor更新（通常安全）
react: 18.2.0 → 18.3.0

# Major更新（慎重に）
next: 13.5.6 → 14.0.0
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

## 破壊的変更への対応

### 移行ガイドの確認

```bash
# React 17 → 18の例
1. 公式移行ガイドを読む
2. TypeScriptエラーを修正
3. テストを更新
4. 段階的にデプロイ
```

### Feature Flagの活用

```javascript
const USE_NEW_VERSION = process.env.USE_V2 === 'true';

if (USE_NEW_VERSION) {
  // 新バージョンのコード
} else {
  // 旧バージョンのコード
}
```

## まとめ

計画的な更新戦略により、セキュリティと安定性を両立できます。

次章では、ライセンス管理について解説します。
