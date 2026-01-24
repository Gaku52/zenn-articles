---
title: "ライセンス管理"
---

# ライセンス管理

## ライセンス管理の重要性

依存関係のライセンスを適切に管理しないと、法的リスクが生じる可能性があります。

### 主なライセンスカテゴリ

**Permissive（寛容）:**
- MIT
- Apache 2.0
- BSD (2-Clause, 3-Clause)
- ISC

**Copyleft（相互主義）:**
- GPL（コード公開必須）
- LGPL（動的リンクのみOK）
- AGPL（ネットワーク経由も公開必須）

## ライセンスチェック

### license-checker

```bash
# インストール
npm install -g license-checker

# ライセンス一覧を表示
license-checker

# 許可されたライセンスのみ
license-checker --onlyAllow "MIT;Apache-2.0;BSD-3-Clause;ISC"

# 禁止ライセンスで失敗
license-checker --failOn "GPL;AGPL;LGPL"

# CSV出力
license-checker --csv > licenses.csv
```

### package.jsonで制限

```json
{
  "license": "MIT",
  "licenses": [
    {
      "type": "MIT",
      "url": "https://opensource.org/licenses/MIT"
    }
  ]
}
```

## CI/CDでの自動チェック

```yaml
# .github/workflows/license-check.yml
name: License Check

on: [pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: |
          npx license-checker \
            --onlyAllow "MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC;0BSD" \
            --failOn "GPL;AGPL;LGPL;CC-BY-NC"
```

## ライセンス帰属表示の生成

```bash
# LICENSES.md生成スクリプト
npx license-checker --json | \
  jq -r 'to_entries | 
    sort_by(.key) | 
    .[] | 
    "## \(.key)\n\n**License:** \(.value.licenses)\n\n---\n"' \
  > LICENSES.md
```

## まとめ

ライセンス管理は法的リスク回避のために重要です。自動化ツールで定期的にチェックしましょう。

次章では、よくあるエラーと解決方法を解説します。
