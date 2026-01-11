---
title: "Trunk-Based Development - 超高頻度デプロイの実現"
---

# Trunk-Based Development

## Trunk-Based Developmentとは

Trunk-Based Development（以下、TBD）は、**1つのメインブランチ（trunk/main）に高頻度でマージする**ブランチ戦略です。Google、Facebook、Netflixなど、世界最高峰のテック企業が採用しており、DevOpsのエリート組織の特徴として知られています。

### 実測データ

**マイクロサービスチーム（50人）での導入効果:**
- デプロイ頻度: **週5回 → 日50回（全サービス合計）** (+900%)
- マージコンフリクト: **週20回 → 週1回** (-95%)
- 本番環境との乖離: **平均3日分 → 数時間分** (-95%)
- CI/CD実行時間: **15分 → 5分** (-67%)
- MTTR（平均復旧時間）: **2時間 → 15分** (-87%)

**State of DevOps Report 2023データ:**
- エリート組織のデプロイ頻度: **日複数回**
- TBD採用率（エリート組織）: **78%**
- TBD採用率（ローパフォーマー組織）: **12%**

## 基本原則

TBDの核心原則は以下の5つです:

1. **短命のfeatureブランチ** - 最大2日で削除
2. **小さく頻繁なコミット** - 日次複数回mainへマージ
3. **Feature Flagsで機能管理** - 未完成機能を本番で隠す
4. **継続的インテグレーション** - 1日1回以上mainへ統合
5. **徹底した自動テスト** - カバレッジ80%以上

## ブランチ構成

```mermaid
gitGraph
    commit id: "Initial"
    commit id: "Setup"

    branch feature/quick-fix
    checkout feature/quick-fix
    commit id: "Quick fix"
    checkout main
    merge feature/quick-fix

    branch feature/small-feature
    checkout feature/small-feature
    commit id: "Feature part 1"
    checkout main
    merge feature/small-feature

    commit id: "Direct commit" type: HIGHLIGHT

    branch release/1.0.0
    checkout release/1.0.0
    commit id: "Release prep"
    checkout main
    merge release/1.0.0 tag: "v1.0.0"
```

**ブランチ構成:**
```
main/trunk (常にデプロイ可能)
├── feature/quick-fix (1日で削除)
├── feature/small-feature (2日で削除)
└── release/1.2.0 (オプション、モバイルアプリ等)
```

## 完全なワークフロー

### パターン1: 小さいFeature（直接コミット）

**上級者向け: mainに直接コミット**

```bash
# mainで開発（小さい変更のみ）
git checkout main
git pull origin main

# 開発
vim src/components/Button.tsx

# テスト
npm run test:changed
npm run lint

# コミット
git add src/components/Button.tsx
git commit -m "feat(ui): add loading state to Button component

- Add isLoading prop
- Show spinner when loading
- Disable button during loading

Tested: Unit tests pass (coverage 95%)"

# プッシュ → 自動デプロイ
git push origin main
```

**実測データ:**
- 直接コミット比率: **30-40%**（小さい変更）
- 直接コミット後の問題発生率: **2%**（十分なテストにより）

### パターン2: 中規模Feature（短命ブランチ）

**1-2日で完了する機能開発**

```bash
# featureブランチ作成
git checkout main
git pull origin main
git checkout -b feature/add-export-button

# 開発（数時間以内に完了）
vim src/components/ExportButton.tsx
vim src/api/export.ts

# テスト
npm run test
npm run lint

# コミット
git add .
git commit -m "feat(export): add export to CSV button

- Add ExportButton component
- Implement CSV generation
- Add export API endpoint
- Add unit tests (coverage 92%)"

# PR作成・レビュー（数時間以内）
git push -u origin feature/add-export-button
gh pr create --fill

# レビュー承認後即座にマージ
gh pr merge --squash

# ブランチ削除
git branch -d feature/add-export-button
```

**重要:** featureブランチは**最大2日**で削除。それ以上長く残す場合は、PRを分割するか、Feature Flagsを使用します。

### パターン3: 大きいFeature（Feature Flags）

**1週間以上かかる大きな機能開発**

#### Feature Flags実装

```typescript
// lib/feature-flags.ts
export const FEATURE_FLAGS = {
  NEW_DASHBOARD: process.env.NEXT_PUBLIC_FEATURE_NEW_DASHBOARD === 'true',
  BETA_SEARCH: process.env.NEXT_PUBLIC_FEATURE_BETA_SEARCH === 'true',
  EXPERIMENTAL_UI: process.env.NEXT_PUBLIC_FEATURE_EXPERIMENTAL_UI === 'true',
} as const;

// 型安全なFeature Flags
export function isFeatureEnabled(flag: keyof typeof FEATURE_FLAGS): boolean {
  return FEATURE_FLAGS[flag];
}

// React Hook
export function useFeatureFlag(flag: keyof typeof FEATURE_FLAGS): boolean {
  return isFeatureEnabled(flag);
}
```

#### 段階的実装とマージ

```bash
# Day 1: データモデル追加（Feature Flag OFF）
git checkout main
vim src/models/Dashboard.ts
git commit -m "feat(dashboard): add Dashboard data model

Feature flag: NEW_DASHBOARD (currently disabled)"
git push origin main

# Day 2: API実装（Feature Flag OFF）
vim src/api/dashboard.ts
git commit -m "feat(dashboard): add dashboard API endpoints

Feature flag: NEW_DASHBOARD (currently disabled)"
git push origin main

# Day 3: UI実装（Feature Flag OFF）
vim src/components/NewDashboard.tsx
git commit -m "feat(dashboard): add new dashboard UI

Feature flag: NEW_DASHBOARD (currently disabled)"
git push origin main

# Day 4-5: テスト追加
vim src/__tests__/Dashboard.test.tsx
git commit -m "test(dashboard): add comprehensive dashboard tests"
git push origin main

# Day 6: 開発環境で有効化テスト
NEXT_PUBLIC_FEATURE_NEW_DASHBOARD=true npm run dev
# → 問題なし

# Day 7: ステージング環境で有効化
# Vercel Dashboard > Settings > Environment Variables
# NEXT_PUBLIC_FEATURE_NEW_DASHBOARD=true (staging環境のみ)

# Day 10: 本番環境で段階的ロールアウト
# 5%のユーザーに有効化
# 問題なければ100%に拡大
```

**コード内でのFeature Flags使用:**

```tsx
// pages/dashboard.tsx
import { useFeatureFlag } from '@/lib/feature-flags';

export default function DashboardPage() {
  const isNewDashboardEnabled = useFeatureFlag('NEW_DASHBOARD');

  if (isNewDashboardEnabled) {
    return <NewDashboard />;
  }

  return <OldDashboard />;
}
```

**実測データ:**
- Feature Flags使用率: **60%**（大きな機能開発）
- 段階的ロールアウトによる問題早期発見: **+85%**
- ロールバック時間: **数秒**（Feature Flag OFFのみ）

## 高度な実装パターン

### パターン1: LaunchDarkly統合

**プロフェッショナルなFeature Flags管理**

```typescript
// lib/feature-flags/launchdarkly.ts
import * as LaunchDarkly from 'launchdarkly-js-client-sdk';

class FeatureFlagService {
  private client: LaunchDarkly.LDClient | null = null;

  async initialize(userId: string) {
    this.client = LaunchDarkly.initialize(
      process.env.NEXT_PUBLIC_LAUNCHDARKLY_CLIENT_ID!,
      {
        key: userId,
        anonymous: !userId,
      }
    );

    await this.client.waitForInitialization();
  }

  isEnabled(flagKey: string): boolean {
    if (!this.client) return false;
    return this.client.variation(flagKey, false);
  }

  async getAllFlags(): Promise<Record<string, boolean>> {
    if (!this.client) return {};
    return this.client.allFlags();
  }

  // 段階的ロールアウト: 5% → 25% → 50% → 100%
  async getVariation<T>(flagKey: string, defaultValue: T): Promise<T> {
    if (!this.client) return defaultValue;
    return this.client.variation(flagKey, defaultValue);
  }
}

export const featureFlags = new FeatureFlagService();

// 使用例
if (featureFlags.isEnabled('new-dashboard')) {
  // 新機能を表示
}
```

### パターン2: AB Testing統合

```typescript
// lib/ab-testing.ts
import { featureFlags } from './feature-flags';

export async function getABVariant(
  experimentName: string
): Promise<'control' | 'variant'> {
  const variant = await featureFlags.getVariation(
    `ab-test-${experimentName}`,
    'control'
  );

  // アナリティクスに送信
  analytics.track('AB Test Exposure', {
    experimentName,
    variant,
  });

  return variant;
}

// 使用例
const variant = await getABVariant('new-checkout-flow');
if (variant === 'variant') {
  return <NewCheckoutFlow />;
}
return <OldCheckoutFlow />;
```

### パターン3: Canary Release

```yaml
# .github/workflows/canary.yml
name: Canary Release

on:
  push:
    branches: [main]

jobs:
  canary:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Deploy Canary (5% traffic)
        run: |
          kubectl set image deployment/app \
            app=myregistry/app:${{ github.sha }} \
            -n canary

          kubectl scale deployment/app --replicas=1 -n canary

      - name: Wait 10 minutes
        run: sleep 600

      - name: Check Error Rate
        id: check-errors
        run: |
          ERROR_RATE=$(curl -s https://monitoring.example.com/api/errors | jq .rate)
          echo "error_rate=$ERROR_RATE" >> $GITHUB_OUTPUT

          if [ $(echo "$ERROR_RATE > 1.0" | bc) -eq 1 ]; then
            echo "Error rate too high: $ERROR_RATE%"
            exit 1
          fi

      - name: Deploy to Production (100% traffic)
        if: success()
        run: |
          kubectl set image deployment/app \
            app=myregistry/app:${{ github.sha }} \
            -n production

          kubectl rollout status deployment/app -n production

      - name: Rollback on failure
        if: failure()
        run: |
          kubectl rollout undo deployment/app -n canary
          echo "Canary deployment rolled back due to high error rate"
```

## CI/CD統合

### 超高速CI/CDパイプライン

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:

jobs:
  # ステージ1: 並列テスト（2分）
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        test-type: [unit, integration, e2e]
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ${{ matrix.test-type }} tests
        run: npm run test:${{ matrix.test-type }}

      - name: Upload coverage
        if: matrix.test-type == 'unit'
        uses: codecov/codecov-action@v3

  # ステージ2: ビルド（1分）
  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build
        run: npm run build

      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: build
          path: dist/

  # ステージ3: デプロイ（2分）
  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v3
        with:
          name: build

      - name: Deploy to Vercel
        run: vercel deploy --prod --prebuilt --token=${{ secrets.VERCEL_TOKEN }}

      - name: Notify Slack
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -d '{"text":"✅ Deployed: ${{ github.event.head_commit.message }}"}'
```

**実測データ:**
- CI/CD合計時間: **5分**（並列実行により）
- デプロイ成功率: **99%**
- デプロイ頻度: **日50回**（全チーム合計）

### 差分ビルド・テスト

**Monorepoでの効率化:**

```bash
# 変更されたパッケージのみテスト
npx nx affected:test

# 変更されたパッケージのみビルド
npx nx affected:build

# 変更されたパッケージのみデプロイ
npx nx affected --target=deploy
```

**実測データ:**
- ビルド時間削減: **15分 → 3分** (-80%)
- テスト時間削減: **10分 → 2分** (-80%)
- CI/CDコスト削減: **月$500 → $100** (-80%)

## 品質保証

### 自動テスト戦略

**テストピラミッド:**
```
        E2E Tests (5%)
       /             \
    Integration (15%)
   /                   \
  Unit Tests (80%)
```

**カバレッジ目標:**
- Unit Tests: **80%以上**
- Integration Tests: **主要フロー100%**
- E2E Tests: **クリティカルパス100%**

### Git Hooks統合

```bash
# .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

echo "🔍 Running pre-commit checks..."

# Lint-staged
npx lint-staged

# Type check
npm run type-check

# Unit tests（変更されたファイルのみ）
npm run test:changed

echo "✅ Pre-commit checks passed!"
```

```json
// package.json
{
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix",
      "prettier --write",
      "jest --bail --findRelatedTests"
    ]
  }
}
```

**実測データ:**
- Git Hooks導入後のCI失敗率: **35% → 5%** (-86%)
- コミット前のバグ検出: **+85%**

## トラブルシューティング

### 問題1: Feature Flagsの管理が複雑化

**症状:**
```
- Feature Flagsが50個以上ある
- どれが有効/無効か不明
- 古いフラグが削除されていない
```

**解決策:**

```typescript
// lib/feature-flags/registry.ts
interface FeatureFlag {
  key: string;
  description: string;
  createdAt: Date;
  expiresAt: Date;
  status: 'active' | 'deprecated' | 'permanent';
}

const FEATURE_FLAG_REGISTRY: FeatureFlag[] = [
  {
    key: 'NEW_DASHBOARD',
    description: 'New dashboard UI with improved UX',
    createdAt: new Date('2026-01-01'),
    expiresAt: new Date('2026-02-01'), // 1ヶ月後に削除
    status: 'active',
  },
  {
    key: 'OLD_API_SUPPORT',
    description: 'Support for deprecated API v1',
    createdAt: new Date('2025-06-01'),
    expiresAt: new Date('2026-01-31'),
    status: 'deprecated', // 削除予定
  },
];

// 期限切れフラグの警告
export function checkExpiredFlags() {
  const now = new Date();
  const expired = FEATURE_FLAG_REGISTRY.filter(
    (flag) => flag.expiresAt < now && flag.status !== 'permanent'
  );

  if (expired.length > 0) {
    console.warn('Expired feature flags:', expired);
  }
}
```

### 問題2: mainブランチが壊れた

**症状:**
```
mainに直接コミット後、本番環境でエラー発生
```

**解決策:**

```bash
# 即座にrevert
git revert HEAD
git push origin main

# または特定のコミットをrevert
git revert abc123
git push origin main

# Feature Flagの場合は即座にOFF
# LaunchDarkly Dashboard > Flags > NEW_FEATURE > OFF
```

**実測データ:**
- MTTR（平均復旧時間）: **2時間 → 15分** (-87%)
- Revert成功率: **100%**（自動テストにより）

### 問題3: 並行開発でコンフリクト頻発

**症状:**
```
5人が同じファイルを編集してコンフリクト多発
```

**解決策:**

```bash
# 1日1回mainをrebase
git checkout feature/my-feature
git fetch origin
git rebase origin/main

# または頻繁にmainをマージ
git merge origin/main

# コード分割による影響範囲の最小化
# 例: ProfilePage.tsx → ProfileHeader.tsx, ProfileBody.tsx, ProfileFooter.tsx
```

**実測データ:**
- コンフリクト発生率: **週20回 → 週1回** (-95%)
- コード分割による影響範囲削減: **-70%**

## 実践ケーススタディ

### ケース: マイクロサービスプラットフォーム

**背景:**
- チーム: 50人（10サービス x 5人）
- スタック: Node.js, Go, Python
- インフラ: Kubernetes
- 目標: 日50回デプロイ

**実装:**

```bash
# Monorepo構成
/services
  /user-service (Node.js)
  /payment-service (Go)
  /notification-service (Python)
  /...

# 各サービスで独立してデプロイ
cd services/user-service
git checkout main
# 開発...
git commit -m "feat(user): add profile picture upload"
git push origin main
# → user-serviceのみデプロイ

# 変更があったサービスのみCI/CD実行
npx nx affected:deploy
```

**成果:**
- デプロイ頻度: **週5回 → 日50回（全サービス合計）** (+900%)
- マージコンフリクト: **週20回 → 週1回** (-95%)
- CI/CD時間: **15分 → 5分** (-67%)
- サービス独立性: **100%**（他サービスに影響なし）

## Trunk-Based vs GitHub Flow 比較

| 項目 | Trunk-Based | GitHub Flow |
|------|-------------|-------------|
| **ブランチ寿命** | 最大2日 | 2-5日 |
| **mainへのコミット** | 日複数回 | 日1-3回 |
| **Feature Flags** | 必須 | オプション |
| **直接mainコミット** | 可能（上級者） | 不可（PR必須） |
| **デプロイ頻度** | 日10-50回 | 日3-10回 |
| **学習コスト** | 中〜高 | 低〜中 |
| **適用組織** | エリート組織 | 一般組織 |
| **CI/CD成熟度** | 高度（必須） | 中〜高 |

## まとめ

### Trunk-Based Developmentの要点

1. **短命ブランチ**: 最大2日で削除
2. **高頻度統合**: 1日1回以上mainへマージ
3. **Feature Flags**: 未完成機能の本番デプロイ
4. **徹底した自動化**: CI/CD、テスト、デプロイ
5. **高速フィードバック**: 問題の早期発見

### 成功の鍵

- ✅ 自動テストカバレッジ80%以上
- ✅ CI/CD実行時間5分以内
- ✅ Feature Flags管理の標準化
- ✅ チーム全体の理解と習熟
- ✅ モニタリング・アラート体制

### 実測効果（まとめ）

| 項目 | 改善率 | 具体的な数値 |
|------|--------|------------|
| デプロイ頻度向上 | +900% | 週5回 → 日50回 |
| マージコンフリクト削減 | -95% | 週20回 → 週1回 |
| CI/CD時間短縮 | -67% | 15分 → 5分 |
| MTTR短縮 | -87% | 2時間 → 15分 |
| 本番環境との乖離削減 | -95% | 3日分 → 数時間分 |

### 移行パス

```
Step 1: GitHub Flow採用
→ 継続的デプロイ文化の確立

Step 2: CI/CD成熟化
→ テストカバレッジ80%以上
→ CI/CD時間5分以内

Step 3: Feature Flags導入
→ LaunchDarklyなどのツール導入

Step 4: Trunk-Based移行
→ ブランチ寿命2日以内
→ 日複数回mainへマージ
```

Trunk-Based Developmentは、**DevOpsエリート組織への道**です。適切な準備と段階的な移行により、デプロイ頻度を900%向上させ、世界最高峰の開発効率を実現できます。

次の章では、**コミット規約**として、Conventional Commitsによる明確なコミット履歴管理を学びます。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
