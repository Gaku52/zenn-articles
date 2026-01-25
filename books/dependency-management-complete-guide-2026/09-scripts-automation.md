---
title: "スクリプト自動化"
---

# スクリプト自動化

## npm scriptsの基本

npm scriptsは、package.jsonに定義されたタスクを実行するための仕組みです。ビルド、テスト、デプロイなどの開発フローを自動化できます。

### 基本的なscripts定義

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "scripts": {
    "start": "node dist/index.js",
    "dev": "ts-node-dev --respawn src/index.ts",
    "build": "tsc",
    "test": "vitest run",
    "lint": "eslint src --ext .ts,.tsx",
    "format": "prettier --write \"src/**/*.{ts,tsx,json}\""
  }
}
```

### 実行方法

```bash
# npm run <script-name>
npm run build
npm run test
npm run lint

# 特殊なscripts（runを省略可能）
npm start    # npm run start と同じ
npm test     # npm run test と同じ
```

### 引数の渡し方

```bash
# -- を使用して引数を渡す
npm run lint -- --fix
npm run test -- --coverage
npm run build -- --mode production

# package.jsonで引数を含める
{
  "scripts": {
    "build:prod": "tsc --mode production",
    "test:coverage": "vitest run --coverage"
  }
}
```

## pre/post Hooks

npm scriptsは、自動的にpre/postフックを実行します。

### フックの仕組み

```json
{
  "scripts": {
    "prebuild": "npm run clean",
    "build": "tsc",
    "postbuild": "npm run copy-assets",

    "pretest": "npm run lint",
    "test": "vitest run",
    "posttest": "npm run coverage-report",

    "clean": "rm -rf dist",
    "copy-assets": "cp -r public dist/public",
    "coverage-report": "echo 'Test coverage report generated'"
  }
}
```

実行順序:
```bash
npm run build

# 実際の実行順序:
# 1. npm run prebuild  (clean)
# 2. npm run build     (tsc)
# 3. npm run postbuild (copy-assets)
```

### よく使われるフックパターン

```json
{
  "scripts": {
    "preinstall": "echo 'Installing dependencies...'",
    "postinstall": "husky install",

    "prepublishOnly": "npm run build && npm run test",
    "preversion": "npm run lint",
    "version": "npm run format && git add -A",
    "postversion": "git push && git push --tags",

    "prepare": "husky install"
  }
}
```

## 並列・直列実行

### npm-run-all

npm-run-allは、複数のscriptsを並列・直列で実行できるツールです。

```bash
npm install --save-dev npm-run-all
```

```json
{
  "scripts": {
    "clean": "rm -rf dist coverage",
    "build:tsc": "tsc",
    "build:esbuild": "esbuild src/index.ts --bundle --outfile=dist/bundle.js",
    "build:copy": "cp -r public dist/public",

    "lint:js": "eslint src --ext .ts,.tsx",
    "lint:css": "stylelint 'src/**/*.css'",
    "lint:types": "tsc --noEmit",

    "test:unit": "vitest run --coverage",
    "test:e2e": "playwright test",

    "build": "npm-run-all clean --parallel build:*",
    "lint": "npm-run-all --parallel lint:*",
    "test": "npm-run-all --sequential test:unit test:e2e",

    "ci": "npm-run-all lint test build"
  }
}
```

#### npm-run-allのオプション

```json
{
  "scripts": {
    "start:all": "npm-run-all --parallel start:api start:web",
    "build:all": "npm-run-all --sequential clean --parallel build:*",
    "test:all": "npm-run-all --continue-on-error test:*",
    "watch": "npm-run-all --parallel watch:*",
    "watch:tsc": "tsc --watch",
    "watch:css": "postcss src/styles.css -o dist/styles.css --watch"
  }
}
```

オプション:
- `--parallel`: 並列実行
- `--sequential`: 直列実行（デフォルト）
- `--continue-on-error`: エラーが発生しても継続
- `--race`: 最初に終了したタスクで全体を終了

### &&と||の使用

```json
{
  "scripts": {
    "build": "npm run clean && npm run compile && npm run bundle",
    "test": "npm run lint && npm run test:unit && npm run test:e2e",

    "deploy": "npm run build && npm run test || echo 'Build or test failed'",

    "start": "npm run build && node dist/index.js",
    "dev": "concurrently \"npm run watch:tsc\" \"npm run watch:server\""
  }
}
```

### concurrentlyの使用

```bash
npm install --save-dev concurrently
```

```json
{
  "scripts": {
    "dev": "concurrently \"npm:dev:*\"",
    "dev:api": "nodemon src/api/index.ts",
    "dev:web": "vite",
    "dev:worker": "ts-node src/worker/index.ts",

    "watch": "concurrently -k -p \"[{name}]\" -n \"TSC,CSS,SERVER\" -c \"blue,magenta,green\" \"npm:watch:tsc\" \"npm:watch:css\" \"npm:watch:server\"",
    "watch:tsc": "tsc --watch",
    "watch:css": "postcss src/styles.css -o dist/styles.css --watch",
    "watch:server": "nodemon dist/index.js"
  }
}
```

オプション:
- `-k, --kill-others`: 1つが終了したら全体を停止
- `-p, --prefix`: 出力にプレフィックスを追加
- `-n, --names`: タスク名を指定
- `-c, --prefix-colors`: 色を指定

## クロスプラットフォーム対応

### cross-env

環境変数をクロスプラットフォームで設定します。

```bash
npm install --save-dev cross-env
```

```json
{
  "scripts": {
    "dev": "cross-env NODE_ENV=development ts-node-dev src/index.ts",
    "build": "cross-env NODE_ENV=production tsc",
    "test": "cross-env NODE_ENV=test vitest run",

    "start:dev": "cross-env PORT=3000 NODE_ENV=development node dist/index.js",
    "start:prod": "cross-env PORT=8080 NODE_ENV=production node dist/index.js"
  }
}
```

### rimraf

クロスプラットフォームでファイル削除を行います。

```bash
npm install --save-dev rimraf
```

```json
{
  "scripts": {
    "clean": "rimraf dist coverage .turbo",
    "clean:all": "rimraf dist coverage .turbo node_modules",
    "prebuild": "rimraf dist"
  }
}
```

### mkdirp

クロスプラットフォームでディレクトリ作成を行います。

```bash
npm install --save-dev mkdirp
```

```json
{
  "scripts": {
    "prepare:dist": "mkdirp dist/assets dist/public",
    "build": "npm run prepare:dist && tsc"
  }
}
```

### copyfiles

クロスプラットフォームでファイルコピーを行います。

```bash
npm install --save-dev copyfiles
```

```json
{
  "scripts": {
    "copy:assets": "copyfiles -u 1 \"src/assets/**/*\" dist",
    "copy:public": "copyfiles -u 1 \"public/**/*\" dist/public",
    "postbuild": "npm-run-all --parallel copy:*"
  }
}
```

## Git Hooks連携

### husky

huskyは、Git hooksを簡単に管理できるツールです。

```bash
npm install --save-dev husky
npx husky init
```

```json
{
  "scripts": {
    "prepare": "husky install"
  }
}
```

#### pre-commit hook

```bash
# .husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npm run lint
npm run typecheck
```

#### commit-msg hook

```bash
# .husky/commit-msg
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npx --no -- commitlint --edit $1
```

#### pre-push hook

```bash
# .husky/pre-push
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npm run test
npm run build
```

### lint-staged

lint-stagedは、ステージングされたファイルのみをlintします。

```bash
npm install --save-dev lint-staged
```

```json
{
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md}": [
      "prettier --write"
    ],
    "*.css": [
      "stylelint --fix",
      "prettier --write"
    ]
  }
}
```

```bash
# .husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npx lint-staged
```

### commitlint

commitlintは、コミットメッセージの形式をチェックします。

```bash
npm install --save-dev @commitlint/cli @commitlint/config-conventional
```

```javascript
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      [
        'feat',     // 新機能
        'fix',      // バグ修正
        'docs',     // ドキュメント
        'style',    // フォーマット
        'refactor', // リファクタリング
        'perf',     // パフォーマンス改善
        'test',     // テスト
        'chore',    // ビルド・補助ツール
        'ci',       // CI設定
        'revert'    // revert
      ]
    ]
  }
};
```

## CI/CD自動化

### GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck

  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 21]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - run: npm ci
      - run: npm run test
      - run: npm run test:e2e

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        if: matrix.node-version == '20'

  build:
    runs-on: ubuntu-latest
    needs: [lint, test]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci
      - run: npm run build

      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: dist
          path: dist/

  deploy:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v3
        with:
          name: dist
          path: dist/

      - name: Deploy to production
        run: npm run deploy
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

### pnpmを使用する場合

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v2
        with:
          version: 8

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'

      - run: pnpm install --frozen-lockfile
      - run: pnpm run lint
      - run: pnpm run test
      - run: pnpm run build
```

### キャッシュの活用

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'  # または 'pnpm', 'yarn'

# カスタムキャッシュ
- uses: actions/cache@v3
  with:
    path: |
      ~/.npm
      node_modules
      .turbo
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

## タスクランナー比較

### npm scripts vs gulp vs webpack

| 機能 | npm scripts | gulp | webpack |
|------|-------------|------|---------|
| 学習曲線 | 低 | 中 | 高 |
| 設定の複雑さ | 低 | 中 | 高 |
| パフォーマンス | 中 | 高 | 高 |
| プラグインエコシステム | 豊富 | 豊富 | 非常に豊富 |
| ビルド最適化 | 限定的 | 可能 | 強力 |
| 推奨用途 | 小〜中規模 | 中〜大規模 | 大規模・複雑なビルド |

### 推奨される選択基準

#### npm scriptsが適している場合
- シンプルなビルドプロセス
- 既存のCLIツールを組み合わせる
- 学習コストを抑えたい

#### gulpが適している場合
- 複雑なファイル操作が必要
- ストリームベースの処理が効率的
- レガシープロジェクトの移行

#### webpackが適している場合
- モジュールバンドリングが必要
- 複雑な最適化が必要
- Single Page Application (SPA)

## scriptsのベストプラクティス

### チェックリスト

- [ ] scriptsの命名規則を統一する（例: build、test、lint、clean）
- [ ] pre/postフックを活用して処理を自動化する
- [ ] クロスプラットフォーム対応のツールを使用する
- [ ] 並列実行可能なタスクは並列化する
- [ ] Git hooksでコード品質を保つ
- [ ] CI/CDでscriptsを再利用する
- [ ] 長いコマンドはscriptsに切り出す
- [ ] 環境変数はcross-envで管理する

### 推奨パターン

```json
{
  "scripts": {
    "dev": "concurrently \"npm:dev:*\"",
    "dev:api": "nodemon src/api",
    "dev:web": "vite",

    "build": "npm-run-all clean --parallel build:*",
    "build:tsc": "tsc",
    "build:copy": "copyfiles -u 1 \"public/**/*\" dist",

    "test": "npm-run-all lint typecheck test:unit",
    "test:unit": "vitest run",
    "test:e2e": "playwright test",
    "test:coverage": "vitest run --coverage",

    "lint": "npm-run-all --parallel lint:*",
    "lint:js": "eslint src --ext .ts,.tsx",
    "lint:css": "stylelint 'src/**/*.css'",
    "lint:fix": "npm-run-all \"lint:* -- --fix\"",

    "typecheck": "tsc --noEmit",
    "format": "prettier --write \"src/**/*.{ts,tsx,json,css}\"",
    "clean": "rimraf dist coverage .turbo",

    "prepare": "husky install",
    "ci": "npm-run-all lint typecheck test build"
  }
}
```

## まとめ

npm scriptsは、開発フローを自動化するための強力なツールです。pre/postフック、並列実行、クロスプラットフォーム対応、Git hooks連携を活用することで、効率的な開発環境を構築できます。

重要なポイント:
- **シンプルさを保つ**: 必要最小限のツールを使用
- **並列化**: 並列実行可能なタスクは並列化してパフォーマンス向上
- **クロスプラットフォーム**: cross-env、rimrafなどで環境依存を排除
- **自動化**: Git hooksとCI/CDで品質を保つ

次章では、iOS向けのSwift Package Managerについて解説します。

## 参考文献

- [npm scripts documentation](https://docs.npmjs.com/cli/v10/using-npm/scripts)
- [npm-run-all documentation](https://github.com/mysticatea/npm-run-all)
- [concurrently documentation](https://github.com/open-cli-tools/concurrently)
- [husky documentation](https://typicode.github.io/husky/)
- [lint-staged documentation](https://github.com/okonet/lint-staged)
- [commitlint documentation](https://commitlint.js.org/)
- [GitHub Actions documentation](https://docs.github.com/en/actions)
