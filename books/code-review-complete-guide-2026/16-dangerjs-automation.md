---
title: "第16章 Danger.jsによる自動化"
---

# 第16章 Danger.jsによる自動化

本章ではDanger.jsを使ったコードレビュー自動化について、具体的なセットアップ方法と実践的なルール実装を詳しく解説します。Danger.jsは、Pull Requestに対して自動的なチェックを実行し、レビューコメントを投稿できる強力なツールです。

## Danger.jsとは

Danger.jsは、Pull Requestの自動レビューを実現するJavaScript/TypeScriptベースのツールです。GitHubやGitLabなどのプラットフォームと統合し、PRのメタデータや変更内容に基づいて自動的なチェックを実行できます。

### Danger.jsの利点

- PRサイズの監視と警告
- コードオーナーへの自動通知
- テストカバレッジの検証
- コーディング規約の自動チェック
- レビューの負担軽減

## Danger.jsのセットアップ

プロジェクトにDanger.jsをセットアップする手順を解説します。

### インストール

```bash
# npm を使用する場合
npm install --save-dev danger

# yarn を使用する場合
yarn add -D danger

# pnpm を使用する場合
pnpm add -D danger
```

### package.json への追加

```json
{
  "name": "your-project",
  "scripts": {
    "danger:pr": "danger pr",
    "danger:local": "danger local"
  },
  "devDependencies": {
    "danger": "^11.3.0"
  }
}
```

### GitHub Token の設定

Danger.jsがGitHub APIにアクセスするために、Personal Access Tokenが必要です。

1. GitHubで Personal Access Token を作成（repo権限が必要）
2. 環境変数に設定: `DANGER_GITHUB_API_TOKEN`

## 基本的な Dangerfile の作成

プロジェクトルートに `dangerfile.ts` または `dangerfile.js` を作成します。

### シンプルな例

```typescript
// dangerfile.ts
import { danger, warn, fail, message, markdown } from 'danger';

// PR の基本情報を取得
const pr = danger.github.pr;
const modifiedFiles = danger.git.modified_files;
const createdFiles = danger.git.created_files;
const deletedFiles = danger.git.deleted_files;

// PR タイトルのチェック
if (!pr.title.match(/^(feat|fix|docs|style|refactor|test|chore):/)) {
  warn('PR タイトルは Conventional Commits の形式にしてください（例: feat: 新機能の追加）');
}

// PR 説明文のチェック
if (pr.body.length < 10) {
  fail('PR の説明文を追加してください（最低10文字以上）');
}

// 大きすぎる PR の警告
const changedFiles = modifiedFiles.length + createdFiles.length + deletedFiles.length;
if (changedFiles > 50) {
  warn(`このPRは ${changedFiles} ファイルを変更しています。PRを分割することを検討してください。`);
}

// 変更行数の確認
const additions = pr.additions;
const deletions = pr.deletions;
if (additions + deletions > 1000) {
  fail(`このPRは ${additions + deletions} 行を変更しています。レビューが困難なため、PRを分割してください。`);
}
```

## PRサイズチェック

大規模なPRはレビューが困難になるため、適切なサイズに保つことが推奨されます。

```typescript
// dangerfile.ts
import { danger, warn, fail, markdown } from 'danger';

/**
 * PR サイズをチェックし、警告またはエラーを出す
 */
function checkPRSize() {
  const pr = danger.github.pr;
  const additions = pr.additions;
  const deletions = pr.deletions;
  const totalChanges = additions + deletions;

  // ファイル数のチェック
  const modifiedFiles = danger.git.modified_files;
  const createdFiles = danger.git.created_files;
  const changedFiles = modifiedFiles.length + createdFiles.length;

  // サイズ判定
  let prSize = 'XS';
  if (totalChanges > 1000) {
    prSize = 'XL';
  } else if (totalChanges > 500) {
    prSize = 'L';
  } else if (totalChanges > 200) {
    prSize = 'M';
  } else if (totalChanges > 50) {
    prSize = 'S';
  }

  // レポート
  markdown(`
## PR サイズレポート

- サイズ: **${prSize}**
- 追加行: ${additions} 行
- 削除行: ${deletions} 行
- 変更ファイル数: ${changedFiles} ファイル

### 推奨事項
${getPRSizeRecommendation(prSize, totalChanges, changedFiles)}
  `);

  // 警告とエラー
  if (prSize === 'XL') {
    fail('PRが大きすぎます。複数のPRに分割することを強く推奨します。');
  } else if (prSize === 'L') {
    warn('PRが大きめです。可能であれば分割を検討してください。');
  }
}

function getPRSizeRecommendation(size: string, changes: number, files: number): string {
  if (size === 'XL') {
    return `
- このPRは非常に大きく、レビューが困難です
- 機能ごとに分割することを検討してください
- 最大でも500行程度に収めることが推奨されます
    `;
  } else if (size === 'L') {
    return `
- このPRはやや大きめです
- レビュー時間が長くなる可能性があります
- 可能であれば機能単位で分割してください
    `;
  } else if (size === 'M') {
    return '適度なサイズです。レビューしやすいPRです。';
  } else {
    return '小さめのPRです。迅速なレビューが期待できます。';
  }
}

checkPRSize();
```

## コードオーナー通知

特定のファイルやディレクトリが変更された際に、関連するチームメンバーに通知します。

```typescript
// dangerfile.ts
import { danger, message, markdown } from 'danger';

/**
 * コードオーナーへの通知
 */
function notifyCodeOwners() {
  const modifiedFiles = danger.git.modified_files;
  const owners = new Set<string>();

  // ファイルパターンとオーナーのマッピング
  const codeOwners: Record<string, string[]> = {
    'src/api/': ['@backend-team', '@api-team'],
    'src/components/': ['@frontend-team'],
    'src/utils/': ['@platform-team'],
    'docs/': ['@documentation-team'],
    '.github/': ['@devops-team'],
    'package.json': ['@platform-team'],
    'tsconfig.json': ['@platform-team'],
  };

  // 変更されたファイルをチェック
  modifiedFiles.forEach(file => {
    Object.entries(codeOwners).forEach(([pattern, ownerList]) => {
      if (file.startsWith(pattern) || file.includes(pattern)) {
        ownerList.forEach(owner => owners.add(owner));
      }
    });
  });

  // 通知メッセージ
  if (owners.size > 0) {
    const ownerList = Array.from(owners).join(', ');
    markdown(`
## レビュー推奨メンバー

このPRは以下のチームに関連するファイルを変更しています:

${ownerList}

レビューをお願いします。
    `);
  }
}

notifyCodeOwners();
```

## テストカバレッジ検証

テストファイルの追加を促し、カバレッジレポートを表示します。

```typescript
// dangerfile.ts
import { danger, warn, fail, markdown } from 'danger';
import * as fs from 'fs';

/**
 * テストカバレッジをチェック
 */
function checkTestCoverage() {
  const modifiedFiles = danger.git.modified_files;
  const createdFiles = danger.git.created_files;

  // テスト対象ファイルの判定
  const productionFiles = [...modifiedFiles, ...createdFiles].filter(file =>
    (file.endsWith('.ts') || file.endsWith('.tsx') || file.endsWith('.js') || file.endsWith('.jsx')) &&
    !file.includes('.test.') &&
    !file.includes('.spec.') &&
    !file.includes('__tests__')
  );

  // テストファイルの判定
  const testFiles = [...modifiedFiles, ...createdFiles].filter(file =>
    file.includes('.test.') ||
    file.includes('.spec.') ||
    file.includes('__tests__')
  );

  // 警告: テストが追加されていない
  if (productionFiles.length > 0 && testFiles.length === 0) {
    warn(`
${productionFiles.length} 個のコードファイルが変更されていますが、テストファイルの追加がありません。
テストの追加を検討してください。
    `);
  }

  // カバレッジレポートの読み込み（Jest の例）
  if (fs.existsSync('coverage/coverage-summary.json')) {
    const coverageData = JSON.parse(
      fs.readFileSync('coverage/coverage-summary.json', 'utf-8')
    );

    const total = coverageData.total;
    const linesCoverage = total.lines.pct;
    const branchesCoverage = total.branches.pct;
    const functionsCoverage = total.functions.pct;
    const statementsCoverage = total.statements.pct;

    markdown(`
## テストカバレッジ

| 項目 | カバレッジ |
|------|-----------|
| Lines | ${linesCoverage.toFixed(2)}% |
| Branches | ${branchesCoverage.toFixed(2)}% |
| Functions | ${functionsCoverage.toFixed(2)}% |
| Statements | ${statementsCoverage.toFixed(2)}% |
    `);

    // カバレッジが低い場合は警告
    if (linesCoverage < 80) {
      warn(`テストカバレッジが ${linesCoverage.toFixed(2)}% です。80%以上を目指してください。`);
    }
  }
}

checkTestCoverage();
```

## 完全な Dangerfile 実装例

実践的なDangerfileの完全な実装例です。

```typescript
// dangerfile.ts
import { danger, warn, fail, message, markdown } from 'danger';
import * as fs from 'fs';

// ===== PR 基本情報チェック =====

function checkPRBasics() {
  const pr = danger.github.pr;

  // タイトルチェック
  const titlePattern = /^(feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert)(\(.+\))?:/;
  if (!pr.title.match(titlePattern)) {
    warn(`
PR タイトルは Conventional Commits 形式にしてください。

例:
- feat: ユーザー登録機能を追加
- fix: ログイン時のバグを修正
- docs: README を更新
    `);
  }

  // 説明文チェック
  if (!pr.body || pr.body.length < 50) {
    fail('PR の説明文を充実させてください（最低50文字以上推奨）');
  }

  // ラベルチェック
  if (pr.labels.length === 0) {
    message('ラベルを追加することを検討してください（例: bug, feature, documentation）');
  }
}

// ===== PR サイズチェック =====

function checkPRSize() {
  const pr = danger.github.pr;
  const totalChanges = pr.additions + pr.deletions;
  const changedFiles = danger.git.modified_files.length + danger.git.created_files.length;

  let prSize = 'XS';
  if (totalChanges > 1000) prSize = 'XL';
  else if (totalChanges > 500) prSize = 'L';
  else if (totalChanges > 200) prSize = 'M';
  else if (totalChanges > 50) prSize = 'S';

  markdown(`
## PR サイズ: ${prSize}

- 追加: ${pr.additions} 行
- 削除: ${pr.deletions} 行
- ファイル数: ${changedFiles}
  `);

  if (prSize === 'XL') {
    fail('PRが大きすぎます（1000行以上）。分割を検討してください。');
  } else if (prSize === 'L') {
    warn('PRがやや大きめです（500行以上）。レビュー時間が長くなる可能性があります。');
  }
}

// ===== ファイル変更チェック =====

function checkFileChanges() {
  const modifiedFiles = danger.git.modified_files;
  const createdFiles = danger.git.created_files;

  // package.json の変更チェック
  if (modifiedFiles.includes('package.json')) {
    const lockFileChanged =
      modifiedFiles.includes('package-lock.json') ||
      modifiedFiles.includes('yarn.lock') ||
      modifiedFiles.includes('pnpm-lock.yaml');

    if (!lockFileChanged) {
      warn('package.json を変更した場合は、ロックファイルも更新してください。');
    }
  }

  // 設定ファイルの変更
  const configFiles = [
    'tsconfig.json',
    '.eslintrc.js',
    '.prettierrc',
    'jest.config.js',
  ];

  const changedConfigFiles = modifiedFiles.filter(file =>
    configFiles.some(config => file.includes(config))
  );

  if (changedConfigFiles.length > 0) {
    message(`
設定ファイルが変更されています: ${changedConfigFiles.join(', ')}
チーム全体に影響する可能性があるため、慎重にレビューしてください。
    `);
  }

  // マイグレーションファイルのチェック
  const migrationFiles = createdFiles.filter(file =>
    file.includes('migrations/') || file.includes('migration')
  );

  if (migrationFiles.length > 0) {
    warn(`
データベースマイグレーションファイルが追加されています: ${migrationFiles.join(', ')}

確認事項:
- [ ] ロールバック処理が実装されているか
- [ ] 本番環境での実行計画が立てられているか
- [ ] データ損失のリスクがないか
    `);
  }
}

// ===== テストチェック =====

function checkTests() {
  const modifiedFiles = danger.git.modified_files;
  const createdFiles = danger.git.created_files;

  const codeFiles = [...modifiedFiles, ...createdFiles].filter(file =>
    (file.endsWith('.ts') || file.endsWith('.tsx') || file.endsWith('.js') || file.endsWith('.jsx')) &&
    !file.includes('.test.') &&
    !file.includes('.spec.') &&
    !file.includes('__tests__')
  );

  const testFiles = [...modifiedFiles, ...createdFiles].filter(file =>
    file.includes('.test.') ||
    file.includes('.spec.') ||
    file.includes('__tests__')
  );

  if (codeFiles.length > 0 && testFiles.length === 0) {
    warn('コードの変更がありますが、テストファイルの追加・更新がありません。');
  }
}

// ===== コードオーナー通知 =====

function notifyCodeOwners() {
  const modifiedFiles = danger.git.modified_files;
  const owners = new Set<string>();

  const codeOwners: Record<string, string[]> = {
    'src/api/': ['@backend-team'],
    'src/components/': ['@frontend-team'],
    'docs/': ['@doc-team'],
  };

  modifiedFiles.forEach(file => {
    Object.entries(codeOwners).forEach(([pattern, ownerList]) => {
      if (file.startsWith(pattern)) {
        ownerList.forEach(owner => owners.add(owner));
      }
    });
  });

  if (owners.size > 0) {
    message(`レビュー推奨: ${Array.from(owners).join(', ')}`);
  }
}

// ===== メイン実行 =====

checkPRBasics();
checkPRSize();
checkFileChanges();
checkTests();
notifyCodeOwners();
```

## GitHub Actions との連携

Danger.jsをGitHub Actionsで実行する設定例です。

```yaml
# .github/workflows/danger.yml
name: Danger CI

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  danger:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run Danger
        run: npm run danger:pr
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Dangerレビューチェックリスト

Danger.jsを導入する際の確認事項です。

### セットアップ

- [ ] Danger.jsがインストールされているか
- [ ] GitHub Tokenが設定されているか
- [ ] dangerfile.ts が作成されているか
- [ ] GitHub Actionsワークフローが設定されているか

### ルール設計

- [ ] PRサイズチェックが実装されているか
- [ ] タイトル・説明文のチェックがあるか
- [ ] テストファイルの追加を促すルールがあるか
- [ ] コードオーナーへの通知が設定されているか

### 運用

- [ ] 警告メッセージが明確で実行可能か
- [ ] false positive（誤検知）が少ないか
- [ ] チーム全体でルールが合意されているか

## まとめ

本章ではDanger.jsによるコードレビュー自動化について解説しました。

主なポイント:
- Danger.jsでPRの自動チェックを実現できる
- PRサイズ、タイトル、テストの有無などを自動確認
- コードオーナーへの自動通知でレビュー効率を向上
- GitHub Actionsと統合して完全自動化

次章では、ReviewDogとの統合について学びます。

## 参考文献

- [Danger.js Official Documentation](https://danger.systems/js/)
- [Danger.js GitHub Repository](https://github.com/danger/danger-js)
- [Getting Started with Danger](https://danger.systems/js/guides/getting_started.html)
- [Danger.js Plugins](https://danger.systems/js/usage/extending-danger.html)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
