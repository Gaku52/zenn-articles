---
title: "ライセンス管理"
---

# ライセンス管理

## ライセンス管理の重要性

オープンソースソフトウェア（OSS）のライセンスを適切に管理しないと、法的リスクや知的財産権の侵害につながる可能性があります。商用アプリケーションでは特に重要です。

### 法的リスク

#### ライセンス違反

- ライセンス条項に違反すると、訴訟リスクがあります
- ソースコード公開義務のあるライセンスもあります
- 著作権侵害として損害賠償請求される可能性があります

#### コンプライアンス

- 企業はOSSライセンスの遵守が求められます
- 監査で不適切なライセンス使用が発覚すると問題になります
- アプリストアの審査で拒否される可能性があります

## 主なOSSライセンス

### Permissive Licenses（寛容なライセンス）

商用利用が比較的自由なライセンスです。

#### MIT License

最も人気のある寛容なライセンスです。

**特徴**:
- 商用利用可能
- 修正・配布自由
- ライセンス表記が必要
- 無保証

**使用例**: React、Vue.js、Node.js、jQuery

```
MIT License

Copyright (c) <year> <copyright holders>

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
```

#### Apache License 2.0

企業向けに設計された寛容なライセンスです。

**特徴**:
- 商用利用可能
- 特許権の明示的な許諾
- 変更箇所の明示が必要
- ライセンス表記が必要

**使用例**: Android、Kubernetes、Apache HTTP Server

#### BSD License (2-Clause, 3-Clause)

シンプルで寛容なライセンスです。

**特徴**:
- 商用利用可能
- 再配布時にライセンス表記が必要
- 3-ClauseではTrademark条項あり

**使用例**: FreeBSD、NGINX

#### ISC License

MITに似たシンプルなライセンスです。

**特徴**:
- MITとほぼ同等
- より簡潔な文言

**使用例**: npm、Vite

### Copyleft Licenses（相互主義ライセンス）

派生ソフトウェアも同じライセンスで公開する必要があるライセンスです。

#### GPL (GNU General Public License)

最も厳格なCopyleftライセンスです。

**特徴**:
- 派生ソフトウェアもGPLで公開必須
- ソースコード公開義務
- 商用利用は実質困難

**バージョン**:
- **GPLv2**: Linux Kernelで使用
- **GPLv3**: より厳格（特許条項追加）

**使用例**: Linux Kernel (GPLv2)、Git (GPLv2)

#### LGPL (GNU Lesser General Public License)

GPLより緩やかなCopyleftライセンスです。

**特徴**:
- 動的リンクであればソースコード公開不要
- 静的リンクの場合は公開必要
- ライブラリ向け

**使用例**: Qt、GTK+

#### AGPL (GNU Affero General Public License)

ネットワーク経由の利用にも適用されるライセンスです。

**特徴**:
- ネットワーク経由でもソースコード公開義務
- SaaS等でも適用
- 最も厳格

**使用例**: MongoDB (サーバー部分)

## ライセンス互換性

### 互換性マトリクス

| 配布先ライセンス | MIT | Apache 2.0 | BSD | GPL | LGPL | AGPL |
|-----------------|-----|-----------|-----|-----|------|------|
| **MIT** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Apache 2.0** | ❌ | ✅ | ❌ | ⚠️ | ⚠️ | ⚠️ |
| **BSD** | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| **GPL** | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| **LGPL** | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ |
| **AGPL** | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |

凡例:
- ✅: 互換性あり
- ⚠️: 条件付きで互換
- ❌: 互換性なし

### 商用利用での推奨ライセンス

商用アプリケーションでは以下のライセンスが推奨されます。

```
推奨度（高 → 低）:
1. MIT License
2. Apache License 2.0
3. BSD License (2-Clause, 3-Clause)
4. ISC License

避けるべき:
- GPL系（ソースコード公開義務）
- AGPL（ネットワーク経由も公開義務）
```

## ライセンスチェックツール

### license-checker

npm向けのライセンスチェックツールです。

#### インストール

```bash
# グローバルインストール
npm install -g license-checker

# プロジェクトローカル
npm install --save-dev license-checker
```

#### 基本使用

```bash
# ライセンス一覧を表示
license-checker

# CSV形式で出力
license-checker --csv > licenses.csv

# JSON形式で出力
license-checker --json > licenses.json

# 特定のフィールドのみ表示
license-checker --customFormat "{name},{license},{repository}"

# サマリーを表示
license-checker --summary
```

#### ライセンス制限

```bash
# 許可されたライセンスのみ
license-checker --onlyAllow "MIT;Apache-2.0;BSD-3-Clause;BSD-2-Clause;ISC"

# 禁止ライセンスで失敗
license-checker --failOn "GPL;AGPL;LGPL"

# 特定のライセンスを除外
license-checker --exclude "MIT;Apache-2.0"

# 不明なライセンスで失敗
license-checker --failOn "UNKNOWN"
```

### package.jsonでのスクリプト化

```json
{
  "scripts": {
    "license:check": "license-checker --onlyAllow 'MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC;0BSD'",
    "license:fail": "license-checker --failOn 'GPL;AGPL;LGPL;CC-BY-NC'",
    "license:summary": "license-checker --summary",
    "license:csv": "license-checker --csv > licenses.csv",
    "license:report": "license-checker --json | jq . > licenses.json"
  }
}
```

### CI/CDでの自動チェック

```yaml
# .github/workflows/license-check.yml
name: License Check

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  license-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Check licenses
        run: |
          npx license-checker \
            --onlyAllow "MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC;0BSD" \
            --failOn "GPL;AGPL;LGPL;CC-BY-NC"

      - name: Generate license report
        if: always()
        run: |
          npx license-checker --json > licenses.json
          npx license-checker --csv > licenses.csv

      - name: Upload license report
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: license-report
          path: |
            licenses.json
            licenses.csv
```

## ライセンス帰属表示の生成

### LICENSES.md生成

```bash
# license-checkerでLICENSES.mdを生成
npx license-checker --json | \
  jq -r 'to_entries |
    sort_by(.key) |
    .[] |
    "## \(.key)\n\n**License:** \(.value.licenses)\n**Repository:** \(.value.repository // "N/A")\n**Publisher:** \(.value.publisher // "N/A")\n\n---\n"' \
  > LICENSES.md
```

### package.json設定

```json
{
  "scripts": {
    "licenses:generate": "license-checker --markdown > LICENSES.md",
    "postinstall": "npm run licenses:generate"
  }
}
```

### THIRD-PARTY-NOTICES生成

商用アプリケーションでは、サードパーティライブラリのライセンス表記が必要です。

```bash
# ライセンステキスト付きで生成
npx license-checker --json | \
  jq -r 'to_entries |
    sort_by(.key) |
    .[] |
    "====================\n\(.key)\n====================\n\nLicense: \(.value.licenses)\n\n\(.value.licenseText // "License text not available")\n\n"' \
  > THIRD-PARTY-NOTICES.txt
```

### アプリ内表示

モバイルアプリでは、設定画面にライセンス情報を表示することが推奨されます。

```javascript
// src/licenses.ts
import licenses from '../licenses.json';

export function getLicenses() {
  return Object.entries(licenses).map(([name, info]) => ({
    name,
    license: info.licenses,
    repository: info.repository,
    publisher: info.publisher
  }));
}
```

```tsx
// React example
import { getLicenses } from './licenses';

function LicensesScreen() {
  const licenses = getLicenses();

  return (
    <div>
      <h1>Third-Party Licenses</h1>
      {licenses.map((lib) => (
        <div key={lib.name}>
          <h2>{lib.name}</h2>
          <p>License: {lib.license}</p>
          <p>Repository: {lib.repository}</p>
        </div>
      ))}
    </div>
  );
}
```

## package.jsonでのライセンス表記

### 自プロジェクトのライセンス設定

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "license": "MIT",
  "author": {
    "name": "Your Name",
    "email": "your.email@example.com",
    "url": "https://yourwebsite.com"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/yourusername/my-app.git"
  }
}
```

### 複数ライセンスの場合

```json
{
  "license": "(MIT OR Apache-2.0)",
  "licenses": [
    {
      "type": "MIT",
      "url": "https://opensource.org/licenses/MIT"
    },
    {
      "type": "Apache-2.0",
      "url": "https://opensource.org/licenses/Apache-2.0"
    }
  ]
}
```

### プライベート・独自ライセンス

```json
{
  "license": "UNLICENSED",
  "private": true
}
```

または

```json
{
  "license": "SEE LICENSE IN LICENSE.txt",
  "files": [
    "LICENSE.txt"
  ]
}
```

## iOS向けライセンス管理

### CocoaPods

```ruby
# Podfile
target 'MyApp' do
  pod 'Alamofire', '~> 5.8'
  pod 'Kingfisher', '~> 7.10'
end

# ライセンス情報を取得
# pod licenses
```

### Swift Package Manager

Package.swiftにはライセンス情報は含まれません。個別にREADMEやLICENSEファイルを確認する必要があります。

### LicensePlist

iOSアプリ向けのライセンス表示ツールです。

```bash
# インストール
brew install licenseplist

# ライセンス一覧を生成
license-plist --output-path MyApp/Settings.bundle

# または、plistを直接生成
license-plist --output-path MyApp/Licenses.plist
```

## ライセンス管理のベストプラクティス

### チェックリスト

- [ ] 使用する依存関係のライセンスを確認する
- [ ] 商用利用不可のライセンス（GPL等）を避ける
- [ ] CI/CDでライセンスチェックを自動化する
- [ ] LICENSES.mdまたはTHIRD-PARTY-NOTICESを生成する
- [ ] アプリ内にライセンス表示画面を用意する（モバイルアプリ）
- [ ] package.jsonに適切なライセンスを設定する
- [ ] ライセンス違反がないか定期的に確認する
- [ ] 新しい依存関係を追加する際はライセンスを確認する

### 推奨設定

```json
// package.json
{
  "scripts": {
    "license:check": "license-checker --onlyAllow 'MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC;0BSD'",
    "license:generate": "license-checker --json > licenses.json && license-checker --csv > licenses.csv",
    "prelicense:check": "npm ci",
    "postinstall": "npm run license:generate"
  }
}
```

```yaml
# .github/workflows/license.yml
name: License Check

on:
  push:
  pull_request:

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run license:check
```

## コマーシャル利用時の注意点

### 避けるべきライセンス

商用アプリケーションでは以下のライセンスを避けるべきです。

```bash
# 禁止ライセンス
npx license-checker --failOn "GPL;GPLv2;GPLv3;AGPL;AGPLv3;LGPL;LGPLv2;LGPLv3;CC-BY-NC;CC-BY-NC-SA"
```

### デュアルライセンス

一部のライブラリは商用利用のために有料ライセンスを提供しています。

例:
- Qt: LGPLまたは商用ライセンス
- MySQL: GPLまたは商用ライセンス
- MongoDB: SSPL（Server Side Public License）または商用ライセンス

### ライセンス監査

企業では定期的なライセンス監査が推奨されます。

```bash
# 四半期ごとのライセンス監査スクリプト
#!/bin/bash

echo "=== License Audit Report ==="
echo "Date: $(date)"
echo ""

# ライセンスサマリー
echo "=== License Summary ==="
npx license-checker --summary

echo ""
echo "=== Problematic Licenses ==="
npx license-checker --onlyunknown

echo ""
echo "=== GPL/AGPL Libraries ==="
npx license-checker --json | jq -r 'to_entries[] | select(.value.licenses | contains("GPL")) | .key'

# レポートを保存
npx license-checker --json > "license-audit-$(date +%Y%m%d).json"
```

## まとめ

ライセンス管理は、法的リスク回避とコンプライアンス遵守のために極めて重要です。適切なツールで自動化し、定期的にチェックすることで、安全にオープンソースソフトウェアを活用できます。

重要なポイント:
- **ライセンス確認**: 依存関係のライセンスを必ず確認
- **自動化**: CI/CDでライセンスチェックを自動化
- **透明性**: THIRD-PARTY-NOTICESで利用ライブラリを明示
- **コンプライアンス**: 商用利用では特に注意

次章では、よくあるエラーと解決方法を解説します。

## 参考文献

- [Open Source Initiative - Licenses](https://opensource.org/licenses)
- [Choose an Open Source License](https://choosealicense.com/)
- [SPDX License List](https://spdx.org/licenses/)
- [npm package.json - license](https://docs.npmjs.com/cli/v10/configuring-npm/package-json#license)
- [GitHub - license-checker](https://github.com/davglass/license-checker)
- [TLDRLegal - Software Licenses Explained](https://tldrlegal.com/)
