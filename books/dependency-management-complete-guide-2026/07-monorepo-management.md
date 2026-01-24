---
title: "モノレポ依存関係管理"
---

# モノレポ依存関係管理

## モノレポとは

モノレポ（Monorepo）は、複数のプロジェクトやパッケージを1つのリポジトリで管理する手法です。

### モノレポの利点

- コード共有が容易
- アトミックなコミット（複数パッケージを同時変更）
- 統一されたツール設定
- 依存関係の一元管理

### モノレポの課題

- ビルド時間の増加
- 依存関係の複雑化
- スケーラビリティ

## npm Workspaces

### 基本構成

```json
{
  "name": "my-monorepo",
  "private": true,
  "workspaces": [
    "packages/*",
    "apps/*"
  ]
}
```

```
my-monorepo/
├── package.json
├── package-lock.json
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

### コマンド

```bash
# すべてのワークスペースで実行
npm run build --workspaces
npm test --workspaces

# 特定のワークスペースに依存関係を追加
npm install lodash --workspace=packages/utils

# すべてをインストール
npm install
```

## pnpm Workspaces

pnpm-workspace.yaml:

```yaml
packages:
  - 'packages/*'
  - 'apps/*'
  - '!**/test/**'
```

### コマンド

```bash
# フィルタを使用
pnpm --filter ui-components build
pnpm --filter "@mycompany/*" test

# 変更されたパッケージのみ
pnpm --filter "...[origin/main]" test
```

## まとめ

モノレポは大規模プロジェクトで有効ですが、適切なツール選択が重要です。

次章では、ワークスペース設定の詳細を解説します。
