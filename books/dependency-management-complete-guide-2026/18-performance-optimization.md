---
title: "パフォーマンス最適化"
---

# パフォーマンス最適化

## バンドルサイズ最適化

### 依存関係の軽量化

```javascript
// ❌ 重いライブラリ（moment.js: 530KB）
import moment from 'moment';

// ✅ 軽量な代替（date-fns: 13KB）
import { format } from 'date-fns';

// 🏆 ネイティブAPI（0KB）
new Intl.DateTimeFormat('ja-JP').format(new Date());
```

### Tree Shaking

```javascript
// ❌ 全体をインポート
import _ from 'lodash';

// ✅ 必要な関数のみ
import { debounce } from 'lodash-es';
```

### Code Splitting

```javascript
// React.lazy
const AdminPanel = React.lazy(() => import('./AdminPanel'));

<Suspense fallback={<Loading />}>
  <AdminPanel />
</Suspense>
```

## インストール速度の最適化

### pnpmの使用

```bash
# npmからpnpmへ移行
# インストール速度: 3〜8倍高速化
# ディスク使用量: 60〜70%削減
```

### CI/CDキャッシュ

```yaml
# GitHub Actions
- uses: actions/setup-node@v4
  with:
    node-version: '18'
    cache: 'npm'  # npmキャッシュを有効化
```

## ビルド時間の最適化

### 並列ビルド

```json
{
  "scripts": {
    "build": "npm-run-all --parallel build:*",
    "build:js": "webpack",
    "build:css": "sass",
    "build:assets": "imagemin"
  }
}
```

### Turbopack / Turborepo

```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"],
      "cache": true
    }
  }
}
```

## 依存関係の最適化

### 重複パッケージの削減

```bash
# 重複を検出
npm dedupe

# pnpmは自動的に重複排除
pnpm install
```

### 不要な依存関係の削除

```bash
# 未使用パッケージを検出
npx depcheck

# 削除
npm uninstall unused-package
```

## モニタリング

### Bundle Analyzer

```bash
# webpack-bundle-analyzer
npm install --save-dev webpack-bundle-analyzer

# 分析レポート生成
npm run build
open dist/bundle-report.html
```

### CI/CDでのサイズ監視

```yaml
- name: Check bundle size
  uses: andresz1/size-limit-action@v1
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
```

## まとめ

パフォーマンス最適化は継続的な取り組みが重要です。

### 最適化チェックリスト

- [ ] 軽量な代替ライブラリを検討
- [ ] Tree Shakingを有効化
- [ ] Code Splittingを活用
- [ ] pnpmで高速化
- [ ] CI/CDキャッシュを設定
- [ ] Bundle Analyzerで定期監視

これで、依存関係管理完全ガイドは完了です。本書で学んだ知識を活用して、セキュアで効率的なプロジェクトを構築してください。
