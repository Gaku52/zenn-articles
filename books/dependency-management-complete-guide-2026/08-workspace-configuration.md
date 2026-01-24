---
title: "ワークスペース設定詳細"
---

# ワークスペース設定詳細

## ワークスペース間の依存関係

### 内部パッケージの参照

```json
{
  "name": "@mycompany/web",
  "dependencies": {
    "@mycompany/ui-components": "*",
    "@mycompany/utils": "workspace:*"
  }
}
```

### バージョン管理戦略

- **固定バージョン**: 安定性重視
- **workspace:***: ローカル開発の柔軟性

## タスクの並列実行

### Turboレポ

```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": []
    }
  }
}
```

## まとめ

ワークスペースの適切な設定により、モノレポの生産性が大幅に向上します。

次章では、スクリプト自動化について解説します。
