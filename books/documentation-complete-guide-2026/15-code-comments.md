---
title: "コメントのベストプラクティス"
---

# コメントのベストプラクティス

## この章で学ぶこと

コメントは、コードとともに開発者が日々向き合う最も身近なドキュメントです。しかし、適切なコメントを書くことは思いのほか難しく、多くの開発者が悩むポイントでもあります。

この章では、以下のトピックを扱います：

- コメントの本質的な目的と役割
- 良いコメントと悪いコメントの具体的な違い
- Why（なぜ）を説明することの重要性
- What（何を）やHow（どのように）とのバランス
- 自己文書化コードとコメントの適切な使い分け
- 言語別のコメント規約とベストプラクティス
- コメントのメンテナンス戦略

本章を通じて、コードの意図を正確に伝え、将来のメンテナンス性を高めるコメント技術を習得できます。

## なぜこの章が重要か

コメントは、以下の理由から開発において極めて重要です：

**1. コンテキストの保存**
コードそのものは「何をしているか」は示しますが、「なぜそうしているか」は示しません。将来の開発者（または未来の自分）がコードを変更する際、その判断の背景を理解できることは極めて重要です。

**2. チームコミュニケーション**
コメントは、コードレビューや非同期のコミュニケーションツールとして機能します。適切なコメントは、チームメンバーの認知負荷を下げ、開発速度を向上させます。

**3. 技術的負債の可視化**
一時的な回避策やトレードオフを明示的に記録することで、将来のリファクタリングの優先順位付けが可能になります。

**4. オンボーディングの効率化**
新しいチームメンバーが、コードベースの複雑な部分を理解する際、適切なコメントは学習曲線を大幅に改善します。

一方で、不適切なコメントは、コードの理解を妨げ、メンテナンスコストを増大させます。本章では、この微妙なバランスを理解し、実践できるようになることを目指します。

## 前提知識

この章を最大限活用するには、以下の知識があると理解が深まります：

- 基本的なプログラミング経験（変数、関数、制御構造の理解）
- コードレビューの基本的な流れ
- バージョン管理システム（Git）の基礎

ただし、高度な技術知識は必要ありません。具体的な例を通じて、コメントの本質を理解していきます。

---

## コメントの本質的な目的

### コメントが解決すべき問題

コメントは、以下の問題を解決するために存在します：

**1. コンテキストのギャップ**
コードは実装の詳細を示しますが、その決定に至った経緯、ビジネスロジック、制約条件などのコンテキストは示しません。

**2. 複雑性の説明**
アルゴリズムや複雑なビジネスロジックの意図を明確にします。

**3. 警告と注意喚起**
落とし穴や注意すべき点を事前に知らせます。

**4. TODO・FIXME・HACKの追跡**
技術的負債や将来の改善点を明示的に記録します。

### コメントが解決すべきでない問題

逆に、以下の問題はコメントではなく、コードそのもので解決すべきです：

- コードが何をしているかの説明（自己文書化コードで対応）
- 複雑すぎるロジックの補足（コードのリファクタリングで対応）
- 引数や戻り値の型情報（型システムで対応）
- 関数の基本的な動作（関数名・変数名の改善で対応）

## 良いコメント vs 悪いコメント

### 原則：Why not What

コメントの最も重要な原則は、「何をしているか（What）」ではなく「なぜそうしているか（Why）」を説明することです。

#### ❌ 悪い例：Whatを説明するコメント

```typescript
// ユーザーIDを取得
const userId = req.params.id;

// ユーザーを検索
const user = await db.users.findById(userId);

// ユーザーが存在しない場合はエラー
if (!user) {
  throw new Error('User not found');
}

// ユーザーを返す
return user;
```

このコメントは、コードを読めば分かることを繰り返しているだけで、何の価値も追加していません。むしろ、ノイズとなってコードの可読性を下げています。

#### ✅ 良い例：Whyを説明するコメント

```typescript
const userId = req.params.id;
const user = await db.users.findById(userId);

if (!user) {
  // GDPR準拠: 削除されたユーザーへのアクセスは404を返す必要がある
  // 詳細: docs/privacy/gdpr-compliance.md
  throw new NotFoundError('User not found');
}

return user;
```

このコメントは、なぜ特定のエラータイプを使用しているのか、その背景にあるビジネス要件を説明しています。

### パターン1：ビジネスロジックの背景

#### ❌ 悪い例

```typescript
// 90日以上ログインがないユーザーをアーカイブ
if (daysSinceLastLogin > 90) {
  await archiveUser(userId);
}
```

このコメントは、コードが何をしているかを説明しているだけです。

#### ✅ 良い例

```typescript
// GDPR第17条（忘れられる権利）への対応として、90日間ログインがない
// ユーザーをアーカイブする。この期間はプライバシーチームと法務チームで
// 合意された最小限の期間。
// 関連: ADR-023, JIRA-1234
if (daysSinceLastLogin > 90) {
  await archiveUser(userId);
}
```

このコメントは、なぜ90日なのか、誰がこの決定をしたのか、関連ドキュメントはどこにあるのかを明確にしています。

### パターン2：技術的な制約

#### ❌ 悪い例

```typescript
// タイムアウトを3秒に設定
const TIMEOUT = 3000;
```

#### ✅ 良い例

```typescript
// サードパーティAPIのSLA（99.9%）を維持するため、3秒でタイムアウト。
// これより短いと誤検知が増え、長いとユーザー体験が悪化する。
// パフォーマンステスト結果: docs/performance/api-timeout-analysis.md
const TIMEOUT = 3000;
```

### パターン3：非自明なアルゴリズム

#### ❌ 悪い例

```typescript
// ソートする
arr.sort((a, b) => {
  // aとbを比較
  if (a.priority !== b.priority) {
    // priorityで比較
    return b.priority - a.priority;
  }
  // createdAtで比較
  return a.createdAt - b.createdAt;
});
```

#### ✅ 良い例

```typescript
// タスクは優先度（高→低）でソートし、同じ優先度の場合は
// 作成日時（古→新）でソート。これはプロダクトチームの要件：
// 「高優先度の古いタスクが埋もれないようにする」
arr.sort((a, b) => {
  if (a.priority !== b.priority) {
    return b.priority - a.priority; // 降順
  }
  return a.createdAt - b.createdAt; // 昇順
});
```

### パターン4：パフォーマンス最適化

#### ❌ 悪い例

```typescript
// キャッシュをチェック
const cached = cache.get(key);
if (cached) {
  return cached;
}
```

#### ✅ 良い例

```typescript
// この関数は秒間1000回以上呼ばれるため、キャッシュは必須。
// キャッシュなしでは平均レスポンスタイムが50ms→500msに悪化。
// ベンチマーク: tests/benchmarks/user-lookup.bench.ts
const cached = cache.get(key);
if (cached) {
  return cached;
}
```

### パターン5：セキュリティ上の考慮

#### ❌ 悪い例

```typescript
// パスワードをハッシュ化
const hashedPassword = await bcrypt.hash(password, 10);
```

#### ✅ 良い例

```typescript
// bcryptのコストファクター10は、現在のハードウェアで約100msの
// ハッシュ時間を提供し、ブルートフォース攻撃への耐性とユーザー体験の
// バランスを取る。OWASP推奨値（2024年時点）。
// 参照: https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
const hashedPassword = await bcrypt.hash(password, 10);
```

### パターン6：回避策とハック

#### ❌ 悪い例

```typescript
// タイムアウトを追加
await Promise.race([
  fetchData(),
  new Promise((_, reject) => setTimeout(() => reject(new Error('Timeout')), 5000))
]);
```

#### ✅ 良い例

```typescript
// HACK: サードパーティAPIにタイムアウト機能がないため、
// Promise.raceで実装。理想的にはライブラリ側で対応すべき。
// Issue: https://github.com/vendor/library/issues/123
// この実装は、ライブラリがv2.0でネイティブタイムアウトを
// サポートするまでの暫定措置。
await Promise.race([
  fetchData(),
  new Promise((_, reject) => setTimeout(() => reject(new Error('Timeout')), 5000))
]);
```

このコメントは、なぜこのような実装が必要なのか、いつまでこの状態が続くのか、どこで改善できるのかを明確にしています。

## What、Why、Howのバランス

### Whatコメント（通常は不要）

Whatコメントは、コードが何をしているかを説明します。多くの場合、これは不要です。

#### ❌ 避けるべきWhatコメント

```typescript
// 変数を宣言
let total = 0;

// ループを回す
for (const item of items) {
  // totalに加算
  total += item.price;
}

// 結果を返す
return total;
```

これらのコメントは、コードを読めば明らかなことを繰り返しているだけです。

#### ✅ Whatコメントが許される例外

複雑な正規表現や、ドメイン固有の計算式など、コードだけでは意味が分かりにくい場合：

```typescript
// RFC 5322準拠のメールアドレス検証
// 簡略版: ローカル部@ドメイン部の基本構造をチェック
const emailRegex = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/;

// BMI = 体重(kg) / 身長(m)^2
// 例: 70kg, 1.75m → BMI = 70 / (1.75 * 1.75) = 22.86
const bmi = weight / (height * height);
```

### Whyコメント（推奨）

Whyコメントは、なぜその実装を選んだのかを説明します。これが最も価値のあるコメントです。

#### ✅ 優れたWhyコメント

```typescript
// SQLインジェクション対策として、プレースホルダーを使用。
// 文字列連結は絶対に使わないこと。
// セキュリティインシデント履歴: docs/security/sql-injection-incident-2023.md
const result = await db.query(
  'SELECT * FROM users WHERE id = $1',
  [userId]
);

// 並行実行数を5に制限。これはAPIプロバイダーのレート制限
// （秒間10リクエスト）を考慮し、他の処理のための余裕を持たせている。
const results = await pMap(tasks, processTask, { concurrency: 5 });

// エラー時は常にログを出力してからリスロー。
// これにより、エラーのスタックトレースが保持され、
// デバッグが容易になる（過去に何度も救われた）。
try {
  await riskyOperation();
} catch (error) {
  logger.error('riskyOperation failed', { error, context });
  throw error;
}
```

### Howコメント（場合による）

Howコメントは、どのように実装しているかを説明します。通常、コードのリファクタリングで対応すべきですが、複雑なアルゴリズムでは有用です。

#### ✅ Howコメントが有用な例

```typescript
/**
 * Dijkstraのアルゴリズムを使用して最短経路を計算
 *
 * アルゴリズムの概要:
 * 1. 開始ノードの距離を0、他を無限大に初期化
 * 2. 未訪問ノードの中で最小距離のノードを選択
 * 3. 隣接ノードの距離を更新
 * 4. すべてのノードを訪問するまで2-3を繰り返す
 *
 * 時間計算量: O((V + E) log V) - Vはノード数、Eはエッジ数
 * 空間計算量: O(V)
 *
 * 参考: https://en.wikipedia.org/wiki/Dijkstra%27s_algorithm
 */
function findShortestPath(graph: Graph, start: Node, end: Node): Path {
  // 実装...
}
```

```typescript
/**
 * Boyer-Moore文字列検索アルゴリズム
 *
 * 特徴: 不一致時に複数文字スキップできるため、
 *       単純な線形探索より高速（平均O(n/m)）
 *
 * 実装手順:
 * 1. 不良文字テーブルを構築（パターン内の各文字の最右位置）
 * 2. テキストをパターンの長さずつスキャン
 * 3. 不一致時は不良文字テーブルを参照してスキップ量を決定
 */
function boyerMooreSearch(text: string, pattern: string): number {
  // 不良文字テーブルの構築
  const badCharTable = new Map<string, number>();
  for (let i = 0; i < pattern.length - 1; i++) {
    badCharTable.set(pattern[i], pattern.length - 1 - i);
  }

  // 検索ロジック...
}
```

## 自己文書化コードとの使い分け

### 原則：コードで表現できることはコードで表現する

コメントは、コードで表現できない情報を補完するものです。まず、コード自体を改善できないか検討しましょう。

### ケース1：変数名の改善

#### ❌ コメントで補完

```typescript
// ユーザーの年齢
const a = 25;

// ユーザーがプレミアム会員かどうか
const b = true;

// 割引率（パーセント）
const c = 15;
```

#### ✅ 自己文書化コード

```typescript
const userAge = 25;
const isPremiumMember = true;
const discountPercentage = 15;
```

### ケース2：関数名の改善

#### ❌ コメントで補完

```typescript
// ユーザーが18歳以上かチェック
function check(user: User): boolean {
  return user.age >= 18;
}
```

#### ✅ 自己文書化コード

```typescript
function isAdult(user: User): boolean {
  return user.age >= 18;
}
```

### ケース3：マジックナンバーの解消

#### ❌ コメントで補完

```typescript
// 30日間の秒数
if (timestamp > Date.now() - 2592000000) {
  // ...
}
```

#### ✅ 自己文書化コード

```typescript
const THIRTY_DAYS_IN_MS = 30 * 24 * 60 * 60 * 1000;

if (timestamp > Date.now() - THIRTY_DAYS_IN_MS) {
  // ...
}
```

もしくは、より読みやすいライブラリを使用：

```typescript
import { subDays } from 'date-fns';

if (timestamp > subDays(new Date(), 30).getTime()) {
  // ...
}
```

### ケース4：条件式の抽出

#### ❌ コメントで補完

```typescript
// ユーザーが有効で、メール認証済みで、サスペンドされていない場合
if (user.isActive && user.emailVerified && !user.isSuspended) {
  // ...
}
```

#### ✅ 自己文書化コード

```typescript
function canAccessDashboard(user: User): boolean {
  return user.isActive && user.emailVerified && !user.isSuspended;
}

if (canAccessDashboard(user)) {
  // ...
}
```

### ケース5：複雑なロジックの分割

#### ❌ コメントで補完

```typescript
// 割引計算: プレミアム会員は20%、通常会員は購入金額に応じて5-15%
function calculateDiscount(user: User, amount: number): number {
  if (user.isPremium) {
    return amount * 0.2;
  } else {
    if (amount > 10000) {
      return amount * 0.15;
    } else if (amount > 5000) {
      return amount * 0.10;
    } else {
      return amount * 0.05;
    }
  }
}
```

#### ✅ 自己文書化コード

```typescript
function calculateDiscount(user: User, amount: number): number {
  if (user.isPremium) {
    return calculatePremiumDiscount(amount);
  }
  return calculateStandardDiscount(amount);
}

function calculatePremiumDiscount(amount: number): number {
  const PREMIUM_DISCOUNT_RATE = 0.2;
  return amount * PREMIUM_DISCOUNT_RATE;
}

function calculateStandardDiscount(amount: number): number {
  if (amount > 10000) return amount * 0.15;
  if (amount > 5000) return amount * 0.10;
  return amount * 0.05;
}
```

この場合、各関数名が明確な意図を示すため、コメントは不要です。

### コメントが必要な場合

自己文書化コードでは表現できない情報：

1. **ビジネスルールの背景**
2. **技術的な制約**
3. **パフォーマンスの考慮事項**
4. **セキュリティ上の理由**
5. **外部依存関係の制限**
6. **暫定的な回避策**

```typescript
function calculateStandardDiscount(amount: number): number {
  // 割引率は財務部門と合意した固定値。変更する場合は
  // 財務部門の承認が必要。承認プロセス: docs/finance/discount-policy.md
  if (amount > 10000) return amount * 0.15;
  if (amount > 5000) return amount * 0.10;
  return amount * 0.05;
}
```

## TODOコメントのベストプラクティス

### TODO、FIXME、HACKの使い分け

プロジェクトで一般的に使用される特殊なコメントタグとその使い分け：

#### TODO：将来の改善や機能追加

```typescript
// TODO: ページネーション対応が必要。現在は最大100件まで。
// 優先度: 中 / 担当: 未定 / 期限: 2026-Q2
async function getUsers(): Promise<User[]> {
  return db.users.findMany({ limit: 100 });
}

// TODO(@alice): ユニットテストを追加
// 関連: JIRA-456
function complexCalculation(input: number): number {
  // ...
}
```

#### FIXME：既知のバグや問題

```typescript
// FIXME: エッジケースで整数オーバーフローの可能性
// 再現手順: tests/edge-cases/overflow.test.ts
// 報告: JIRA-789
function calculateTotal(items: number[]): number {
  return items.reduce((sum, item) => sum + item, 0);
}

// FIXME: メモリリークの可能性あり。長時間実行でヒープが増加。
// プロファイリング結果: docs/performance/memory-leak-analysis.md
const cache = new Map();
```

#### HACK：暫定的な回避策

```typescript
// HACK: ライブラリのバグ回避のための暫定措置
// Issue: https://github.com/library/repo/issues/123
// 修正予定バージョン: v3.0.0
// このコードはライブラリ更新後に削除すること
const workaround = JSON.parse(JSON.stringify(data));

// HACK: レガシーAPIとの互換性のため、データ構造を変換
// 新APIへの移行が完了したら（2026-Q3予定）、この変換は不要
function transformLegacyData(data: LegacyFormat): ModernFormat {
  // ...
}
```

#### NOTE / INFO：重要な情報

```typescript
// NOTE: この関数は副作用がある（グローバルステートを変更）
// 並行実行時は注意。必要ならロックを使用すること。
function updateGlobalState(newState: State): void {
  // ...
}

// INFO: パフォーマンス上の理由でキャッシュを使用
// キャッシュの有効期限は5分。詳細: docs/caching-strategy.md
const cachedResult = await getCachedOrFetch(key);
```

### 良いTODOコメントの条件

優れたTODOコメントは、以下の情報を含みます：

1. **具体的な作業内容**
2. **優先度や期限（可能であれば）**
3. **担当者（可能であれば）**
4. **関連するチケットやドキュメント**
5. **なぜ今やらないのか**

#### ❌ 悪いTODO

```typescript
// TODO: これを直す
// TODO: あとで対応
// TODO: リファクタリング
```

これらのTODOは曖昧で、いつ、誰が、何をすべきか不明確です。

#### ✅ 良いTODO

```typescript
// TODO(@bob): バリデーションエラーのレスポンスを国際化対応
// 優先度: 低（英語版リリース後）
// 期限: 2026-Q3
// 関連: JIRA-1001, docs/i18n/validation-messages.md
// 今やらない理由: まず英語版を安定させる必要がある
function validateInput(input: string): ValidationResult {
  // ...
}
```

### TODOの管理戦略

TODOコメントを効果的に管理するための戦略：

#### 1. 定期的なレビュー

```typescript
// スクリプト例: 古いTODOを検出
// scripts/check-old-todos.sh

#!/bin/bash
# 6ヶ月以上前のTODOコメントを検出
git log --all --format=%H --since="6 months ago" | while read commit; do
  git grep -n "TODO" $commit -- "*.ts" "*.js"
done
```

#### 2. TODOをイシュートラッキングシステムと連携

```typescript
// TODO: キャッシュの有効期限を設定可能にする
// GitHub Issue: #456
// この実装により、環境ごとに異なるキャッシュ戦略を採用できる
const DEFAULT_CACHE_TTL = 300;
```

#### 3. CI/CDでのTODOチェック

```yaml
# .github/workflows/todo-check.yml
name: TODO Check

on: [pull_request]

jobs:
  check-todos:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Check for TODOs without tickets
        run: |
          # チケット番号がないTODOを検出
          if grep -r "TODO" --include="*.ts" --include="*.js" | grep -v "JIRA-\|#[0-9]"; then
            echo "Error: TODOs must reference a ticket"
            exit 1
          fi
```

## 言語別のコメント規約

### JavaScript / TypeScript

#### 単一行コメント

```typescript
// 単一行コメントにはダブルスラッシュを使用

// 複数行のコメントを書く場合は、
// 各行の先頭にダブルスラッシュを付ける。
// これにより、各行が独立してコメントであることが明確になる。
```

#### 複数行コメント

```typescript
/*
 * 複数行コメントはアスタリスクで囲む。
 * 各行の先頭にアスタリスクを揃えると見やすい。
 * ただし、ドキュメントコメント（後述）と区別するため、
 * 通常のコメントではこの形式は避けることが多い。
 */

/**
 * ドキュメントコメントは二重アスタリスクで開始。
 * JSDocやTSDocとして解析され、IDE補完に使用される。
 * 関数、クラス、メソッドの説明に使用。
 */
```

#### JSDoc / TSDocの例

```typescript
/**
 * ユーザーを作成します
 *
 * @param name - ユーザー名（1-50文字）
 * @param email - メールアドレス（RFC 5322準拠）
 * @returns 作成されたユーザーオブジェクト
 * @throws {ValidationError} バリデーションエラー時
 * @throws {DuplicateError} メールアドレスが既に存在する場合
 *
 * @example
 * ```typescript
 * const user = await createUser('Alice', 'alice@example.com');
 * console.log(user.id); // 'user_abc123'
 * ```
 *
 * @see {@link https://docs.example.com/users | User API Documentation}
 */
async function createUser(
  name: string,
  email: string
): Promise<User> {
  // 実装...
}
```

#### TypeScriptの型アノテーションとコメントの使い分け

```typescript
// ❌ 型情報をコメントで説明（不要）
/**
 * @param id - ユーザーID（文字列）
 * @param age - 年齢（数値）
 * @returns ユーザーオブジェクトまたはnull
 */
function getUser(id: string, age: number): User | null {
  // ...
}

// ✅ 型システムに任せ、ビジネスロジックを説明
/**
 * ユーザーを取得します。年齢パラメータは、
 * 年齢確認が必要なコンテンツへのアクセス時のみ使用されます。
 *
 * @param id - ユーザーID
 * @param age - 年齢確認用（18歳以上のコンテンツアクセス時に必要）
 * @returns ユーザー、または存在しない場合はnull
 */
function getUser(id: string, age: number): User | null {
  // ...
}
```

### Python

#### docstring（推奨）

```python
def create_user(name: str, email: str) -> User:
    """
    ユーザーを作成します。

    Args:
        name: ユーザー名（1-50文字）
        email: メールアドレス（RFC 5322準拠）

    Returns:
        User: 作成されたユーザーオブジェクト

    Raises:
        ValidationError: バリデーションエラー時
        DuplicateError: メールアドレスが既に存在する場合

    Examples:
        >>> user = create_user('Alice', 'alice@example.com')
        >>> print(user.id)
        'user_abc123'

    Note:
        この関数はデータベーストランザクション内で呼び出す必要があります。
        トランザクション外で呼び出すと、データの整合性が保証されません。
    """
    # 実装...
    pass
```

#### 単一行コメント

```python
# Pythonでは#を使用してコメントを書く

# 複数行のコメントも
# 各行に#を付けて書く
# これが推奨スタイル

# HACK: APIのバグ回避のための暫定措置
# Issue: https://github.com/library/repo/issues/789
workaround = json.loads(json.dumps(data))
```

#### Type Hintsとコメントの使い分け

```python
from typing import Optional, List

# ❌ 型情報をコメントで説明（不要）
def get_users(limit):  # type: int
    """
    ユーザーのリストを返す

    Args:
        limit: 取得する最大件数（整数）

    Returns:
        ユーザーのリスト
    """
    pass

# ✅ Type Hintsを使用し、ビジネスロジックを説明
def get_users(limit: int) -> List[User]:
    """
    ユーザーのリストを返します。

    Args:
        limit: 取得する最大件数（1-1000）。
               パフォーマンス上の理由で1000件が上限。

    Returns:
        作成日時の降順でソートされたユーザーのリスト
    """
    pass
```

### Swift

#### マークアップコメント

```swift
/**
 ユーザーを作成します。

 - Parameters:
   - name: ユーザー名（1-50文字）
   - email: メールアドレス（RFC 5322準拠）

 - Returns: 作成されたUserオブジェクト

 - Throws:
   - `ValidationError`: バリデーションエラー時
   - `DuplicateError`: メールアドレスが既に存在する場合

 - Note: この関数は非同期で実行されます。
         メインスレッドでの呼び出しは避けてください。

 - Important: データベース接続が確立されていることを確認してください。

 - Warning: メールアドレスの一意性チェックは大文字小文字を区別します。

 # Example
 ```swift
 let user = try await createUser(name: "Alice", email: "alice@example.com")
 print(user.id) // "user_abc123"
 ```

 - SeeAlso: `User`, `updateUser`, `deleteUser`
 */
func createUser(name: String, email: String) async throws -> User {
    // 実装...
}
```

#### MARK、TODO、FIXME

```swift
// MARK: - User Management
// コードセクションの区切りとして使用

/// ユーザー作成
func createUser() { }

/// ユーザー更新
func updateUser() { }

// MARK: - Private Methods

private func validateEmail() { }

// TODO: ページネーション対応
// 関連: JIRA-123

// FIXME: メモリリークの可能性
// 再現: tests/MemoryLeakTests.swift

// WARNING: この関数は非推奨です（iOS 18で削除予定）
```

### Java

#### Javadoc

```java
/**
 * ユーザーを作成します。
 *
 * <p>この関数は新しいユーザーをデータベースに作成します。
 * メールアドレスの一意性は自動的にチェックされます。</p>
 *
 * @param name ユーザー名（1-50文字）
 * @param email メールアドレス（RFC 5322準拠）
 * @return 作成されたUserオブジェクト
 * @throws ValidationException バリデーションエラー時
 * @throws DuplicateEmailException メールアドレスが既に存在する場合
 * @see User
 * @see #updateUser(String, String)
 * @since 1.0
 * @version 1.2
 *
 * @example
 * <pre>{@code
 * User user = createUser("Alice", "alice@example.com");
 * System.out.println(user.getId()); // "user_abc123"
 * }</pre>
 */
public User createUser(String name, String email)
    throws ValidationException, DuplicateEmailException {
    // 実装...
}
```

### Go

#### Go標準のコメント

```go
// Package user は、ユーザー管理機能を提供します。
//
// このパッケージは、ユーザーの作成、更新、削除、検索などの
// 基本的なCRUD操作をサポートします。
package user

// User は、システムのユーザーを表します。
//
// すべてのフィールドは不変です。ユーザー情報を変更するには、
// UpdateUser関数を使用してください。
type User struct {
    // ID は、ユーザーの一意な識別子です。
    // 形式: "user_" + 20文字のランダム文字列
    ID string

    // Name は、ユーザーの表示名です（1-50文字）。
    Name string

    // Email は、ユーザーのメールアドレスです（RFC 5322準拠）。
    Email string
}

// CreateUser は、新しいユーザーを作成します。
//
// メールアドレスの一意性は自動的にチェックされます。
// 重複する場合は ErrDuplicateEmail を返します。
//
// エラー処理の例:
//
//     user, err := CreateUser("Alice", "alice@example.com")
//     if err != nil {
//         if errors.Is(err, ErrDuplicateEmail) {
//             // 重複エラーの処理
//         }
//         return err
//     }
//
// NOTE: この関数はトランザクション内で呼び出す必要があります。
func CreateUser(name, email string) (*User, error) {
    // 実装...
}
```

### Rust

#### Rustdoc

```rust
/// ユーザーを作成します。
///
/// この関数は新しいユーザーをデータベースに作成します。
/// メールアドレスの一意性は自動的にチェックされます。
///
/// # Arguments
///
/// * `name` - ユーザー名（1-50文字）
/// * `email` - メールアドレス（RFC 5322準拠）
///
/// # Returns
///
/// 作成されたUserオブジェクト
///
/// # Errors
///
/// この関数は以下の場合にエラーを返します:
///
/// * `ValidationError` - バリデーションエラー時
/// * `DuplicateError` - メールアドレスが既に存在する場合
///
/// # Examples
///
/// ```
/// use myapp::user::create_user;
///
/// let user = create_user("Alice", "alice@example.com")?;
/// println!("Created user: {}", user.id);
/// ```
///
/// # Panics
///
/// データベース接続が確立されていない場合にパニックします。
///
/// # Safety
///
/// この関数はスレッドセーフです。複数のスレッドから同時に
/// 呼び出すことができます。
pub fn create_user(name: &str, email: &str) -> Result<User, Error> {
    // 実装...
}
```

## 特殊なケースのコメント

### 1. レガシーコードのコメント

レガシーコードを保守する際、コメントは特に重要です。

#### なぜそのコードを変更しないのか

```typescript
// WARNING: このコードは非効率に見えるが、変更してはいけない。
// 理由: レガシーデータベースのバグ回避のための実装。
// 詳細: docs/legacy/database-quirks.md
// データベース移行が完了したら（2026-Q4予定）、この実装は不要。
function queryLegacyDatabase(sql: string): any[] {
  // 3回リトライする（1回目は必ず失敗する既知のバグ）
  for (let i = 0; i < 3; i++) {
    try {
      return db.query(sql);
    } catch (error) {
      if (i === 2) throw error;
      // 100ms待機（データベースの内部状態が安定するのを待つ）
      await sleep(100);
    }
  }
}
```

#### 歴史的な経緯

```typescript
// 歴史的経緯: このフィールドは元々文字列型だったが、
// 2024年のデータベース移行でnumber型に変更された。
// しかし、APIレスポンスの互換性のため、文字列として返す必要がある。
// 移行計画: docs/migrations/user-id-migration.md
function formatUserId(id: number): string {
  return String(id);
}
```

### 2. パフォーマンス最適化のコメント

#### ベンチマーク結果の記録

```typescript
// パフォーマンス最適化: Map使用でO(n)→O(1)に改善
// ベンチマーク結果（n=10000の場合）:
//   - 配列検索: 約50ms
//   - Map検索: 約0.5ms
// ベンチマークコード: benchmarks/user-lookup.bench.ts
const userMap = new Map(users.map(u => [u.id, u]));

function findUser(id: string): User | undefined {
  return userMap.get(id);
}
```

#### トレードオフの説明

```typescript
// この実装はメモリ使用量が多い（約10MB）が、
// レスポンスタイムの要件（<100ms）を満たすために必要。
// トレードオフ分析: docs/performance/caching-tradeoffs.md
const cache = new LRUCache<string, User>({
  max: 10000,
  maxAge: 1000 * 60 * 5 // 5分
});
```

### 3. セキュリティ関連のコメント

#### セキュリティ上の理由

```typescript
// セキュリティ: XSS攻撃防止のため、必ずエスケープ処理を行う
// OWASP推奨: https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html
function sanitizeHtml(html: string): string {
  return DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['p', 'b', 'i', 'em', 'strong'],
    ALLOWED_ATTR: []
  });
}

// セキュリティ: タイミング攻撃防止のため、定時間比較を使用
// 通常の===演算子は、不一致の位置で早期リターンするため、
// パスワード推測の手がかりになる可能性がある。
function comparePasswords(a: string, b: string): boolean {
  return crypto.timingSafeEqual(
    Buffer.from(a),
    Buffer.from(b)
  );
}
```

#### セキュリティ監査のコメント

```typescript
// SECURITY AUDIT: 2025-12-15 by @security-team
// この実装は以下の脅威モデルに対応:
// - SQLインジェクション: プレースホルダー使用で対応
// - XSS: DOMPurifyでサニタイズ
// - CSRF: トークン検証で対応
// 次回監査予定: 2026-06-15
```

### 4. マルチスレッド・並行処理のコメント

```typescript
// スレッドセーフティ: このMapは複数のワーカースレッドから
// 同時にアクセスされる可能性がある。
// 書き込みはメインスレッドのみ、読み取りはすべてのスレッドで可能。
// ロック不要（書き込みが単一スレッドのため）。
const sharedCache = new Map<string, CachedData>();

// RACE CONDITION WARNING: この変数は複数の非同期処理から
// 同時に更新される可能性がある。必ずロックを取得してから更新すること。
let globalCounter = 0;

async function incrementCounter() {
  await lock.acquire();
  try {
    globalCounter++;
  } finally {
    lock.release();
  }
}
```

### 5. テスト関連のコメント

```typescript
// テスト用のフック: プロダクションコードでは使用しない
// テストでの使用例: tests/user.test.ts:45
export function __testOnly__resetUserCache() {
  if (process.env.NODE_ENV !== 'test') {
    throw new Error('This function is for testing only');
  }
  userCache.clear();
}

// モック可能性: この依存関係は、テストで簡単にモックできるように
// 依存性注入パターンを使用している。
function processPayment(
  gateway: PaymentGateway = defaultGateway
): Promise<PaymentResult> {
  // ...
}
```

## コメントのメンテナンス戦略

### コメントの腐敗問題

コメントの最大の問題は、コードと同期しなくなることです。

#### ❌ 古くなったコメント（危険）

```typescript
// ユーザーIDは数値型
const userId: string = req.params.id;  // 型が変更されている！

// 最大100件まで取得
const users = await db.users.findMany({ limit: 1000 });  // 値が変更されている！

// この関数は同期的に実行される
async function fetchData(): Promise<Data> {  // 非同期に変更されている！
  // ...
}
```

これらのコメントは、コードの変更に追従していないため、誤解を招きます。

### メンテナンス可能なコメントの書き方

#### 1. コードに密結合しない

```typescript
// ❌ コードに密結合（変更に弱い）
// idを取得して、toUpperCase()を呼び出して、trim()する
const userId = req.params.id.toUpperCase().trim();

// ✅ 意図を説明（変更に強い）
// レガシーシステムとの互換性のため、ユーザーIDは大文字で正規化
const userId = req.params.id.toUpperCase().trim();
```

実装が変わっても（例: `.normalize()`を追加）、コメントは依然として正確です。

#### 2. 外部ドキュメントへの参照

```typescript
// ❌ 長い説明をコメントに書く（メンテナンス困難）
// このアルゴリズムは、以下の手順で動作します:
// 1. データを正規化
// 2. 重複を削除
// 3. ソート
// 4. グループ化
// 5. 集計
// 詳細は... (100行のコメント)

// ✅ 外部ドキュメントへの参照（メンテナンス容易）
// データ処理アルゴリズムの詳細: docs/algorithms/data-processing.md
// 概要: 正規化→重複削除→ソート→グループ化→集計
function processData(data: RawData[]): ProcessedData[] {
  // ...
}
```

#### 3. コードの構造でドキュメント化

```typescript
// ❌ コメントで手順を説明
// 1. ユーザーを検証
// 2. 権限をチェック
// 3. データを取得
// 4. レスポンスを整形
async function handleRequest(req: Request): Promise<Response> {
  const user = await validateUser(req);
  if (!hasPermission(user)) throw new Error();
  const data = await fetchData(user);
  return formatResponse(data);
}

// ✅ 関数分割で手順を明確化
async function handleRequest(req: Request): Promise<Response> {
  const user = await validateUser(req);
  await checkPermission(user);
  const data = await fetchData(user);
  return formatResponse(data);
}
```

### コメントレビューのチェックリスト

コードレビュー時に、コメントもレビューしましょう：

```markdown
## コメントレビューチェックリスト

### 必要性
- [ ] このコメントは必要か？コードだけで理解できないか？
- [ ] Whatではなく、Whyを説明しているか？

### 正確性
- [ ] コメントとコードが一致しているか？
- [ ] 古い情報が残っていないか？
- [ ] 実測していないデータを「実測」として書いていないか？

### 明確性
- [ ] 曖昧な表現を避けているか？
- [ ] 具体的な情報（チケット番号、ドキュメントへのリンク）を含むか？
- [ ] 将来の開発者が理解できるか？

### 一貫性
- [ ] プロジェクトのコメント規約に従っているか？
- [ ] 用語が統一されているか？

### 特殊ケース
- [ ] TODOには期限や担当者が明記されているか？
- [ ] HACKには回避策の理由と解決予定が書かれているか？
- [ ] セキュリティ関連のコメントは適切か？
```

### 自動化によるコメント品質の維持

#### ESLintルールの例

```javascript
// .eslintrc.js
module.exports = {
  rules: {
    // TODOコメントにチケット番号を強制
    'no-warning-comments': [
      'warn',
      {
        terms: ['todo', 'fixme', 'hack'],
        location: 'anywhere'
      }
    ],

    // 無意味なコメントを警告
    'spaced-comment': ['error', 'always'],
    'capitalized-comments': ['error', 'always'],
  }
};
```

#### カスタムリンタールール

```typescript
// custom-rules/require-ticket-in-todo.ts
// TODOコメントにJIRAチケット番号があることをチェック

export default {
  meta: {
    type: 'suggestion',
    docs: {
      description: 'TODO comments must include a JIRA ticket number'
    }
  },
  create(context) {
    return {
      Program() {
        const comments = context.getSourceCode().getAllComments();

        for (const comment of comments) {
          if (/TODO/.test(comment.value)) {
            if (!/JIRA-\d+|#\d+/.test(comment.value)) {
              context.report({
                loc: comment.loc,
                message: 'TODO comment must reference a ticket (e.g., JIRA-123 or #123)'
              });
            }
          }
        }
      }
    };
  }
};
```

#### コメント統計スクリプト

```typescript
// scripts/analyze-comments.ts
// コードベース内のコメントを分析

import { Project } from 'ts-morph';

const project = new Project({ tsConfigFilePath: 'tsconfig.json' });

let totalComments = 0;
let todoCount = 0;
let fixmeCount = 0;
let hackCount = 0;

for (const sourceFile of project.getSourceFiles()) {
  const comments = sourceFile
    .getDescendantAtPos(0)
    ?.getLeadingCommentRanges() ?? [];

  for (const comment of comments) {
    const text = comment.getText();
    totalComments++;

    if (/TODO/i.test(text)) todoCount++;
    if (/FIXME/i.test(text)) fixmeCount++;
    if (/HACK/i.test(text)) hackCount++;
  }
}

console.log(`Total comments: ${totalComments}`);
console.log(`TODO: ${todoCount}`);
console.log(`FIXME: ${fixmeCount}`);
console.log(`HACK: ${hackCount}`);

// 警告: HACKが多すぎる場合
if (hackCount > 50) {
  console.warn('Warning: Too many HACK comments. Consider refactoring.');
  process.exit(1);
}
```

## 実践例：リファクタリング前後の比較

### 例1：APIエンドポイントの実装

#### ❌ Before: コメントに頼った実装

```typescript
// ユーザー一覧取得API
app.get('/api/users', async (req, res) => {
  try {
    // クエリパラメータを取得
    const page = parseInt(req.query.page as string) || 1;
    const limit = parseInt(req.query.limit as string) || 10;

    // offsetを計算
    const offset = (page - 1) * limit;

    // データベースから取得
    const users = await db.query(
      'SELECT * FROM users LIMIT $1 OFFSET $2',
      [limit, offset]
    );

    // 総件数を取得
    const totalResult = await db.query('SELECT COUNT(*) FROM users');
    const total = parseInt(totalResult.rows[0].count);

    // レスポンスを返す
    res.json({
      data: users.rows,
      pagination: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit)
      }
    });
  } catch (error) {
    // エラー処理
    console.error(error);
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

#### ✅ After: 自己文書化コード + 最小限のコメント

```typescript
interface PaginationParams {
  page: number;
  limit: number;
}

interface PaginatedResponse<T> {
  data: T[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  };
}

// ページネーションのデフォルト値は、UXチームとの合意事項。
// 変更する場合は、UXチームの承認が必要。
// 詳細: docs/api/pagination-guidelines.md
const DEFAULT_PAGE = 1;
const DEFAULT_LIMIT = 10;
const MAX_LIMIT = 100;

function parsePaginationParams(query: any): PaginationParams {
  const page = Math.max(1, parseInt(query.page) || DEFAULT_PAGE);
  const limit = Math.min(
    MAX_LIMIT,
    Math.max(1, parseInt(query.limit) || DEFAULT_LIMIT)
  );
  return { page, limit };
}

function calculateOffset(page: number, limit: number): number {
  return (page - 1) * limit;
}

async function fetchPaginatedUsers(
  limit: number,
  offset: number
): Promise<User[]> {
  const result = await db.query(
    'SELECT * FROM users LIMIT $1 OFFSET $2',
    [limit, offset]
  );
  return result.rows;
}

async function countTotalUsers(): Promise<number> {
  const result = await db.query('SELECT COUNT(*) FROM users');
  return parseInt(result.rows[0].count);
}

function createPaginatedResponse<T>(
  data: T[],
  params: PaginationParams,
  total: number
): PaginatedResponse<T> {
  return {
    data,
    pagination: {
      ...params,
      total,
      totalPages: Math.ceil(total / params.limit)
    }
  };
}

app.get('/api/users', async (req, res) => {
  try {
    const params = parsePaginationParams(req.query);
    const offset = calculateOffset(params.page, params.limit);

    const [users, total] = await Promise.all([
      fetchPaginatedUsers(params.limit, offset),
      countTotalUsers()
    ]);

    const response = createPaginatedResponse(users, params, total);
    res.json(response);
  } catch (error) {
    logger.error('Failed to fetch users', { error, query: req.query });
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

この実装では、関数名と変数名が意図を明確に示すため、ほとんどのコメントが不要になりました。唯一のコメントは、デフォルト値の背景を説明するビジネスロジックのコメントです。

### 例2：複雑な計算ロジック

#### ❌ Before: 不十分なコメント

```typescript
function calc(items: any[]): number {
  let total = 0;
  for (const item of items) {
    // 割引を計算
    let discount = 0;
    if (item.qty > 10) {
      discount = 0.1;
    } else if (item.qty > 5) {
      discount = 0.05;
    }

    // 小計を計算
    const subtotal = item.price * item.qty;

    // 割引を適用
    const discounted = subtotal * (1 - discount);

    // 税金を追加
    const withTax = discounted * 1.1;

    total += withTax;
  }

  // 配送料を追加
  if (total < 5000) {
    total += 500;
  }

  return total;
}
```

#### ✅ After: 明確な構造 + 必要なコメント

```typescript
interface OrderItem {
  price: number;
  quantity: number;
}

interface PricingConfig {
  taxRate: number;
  freeShippingThreshold: number;
  shippingFee: number;
  volumeDiscounts: { minQuantity: number; rate: number }[];
}

// 価格設定は営業部門と合意した固定値。
// 変更には営業部門の承認が必要。
// 価格ポリシー: docs/business/pricing-policy.md
const PRICING_CONFIG: PricingConfig = {
  taxRate: 0.10,  // 消費税10%（日本）
  freeShippingThreshold: 5000,  // 5000円以上で送料無料
  shippingFee: 500,  // 一律500円
  volumeDiscounts: [
    { minQuantity: 11, rate: 0.10 },  // 11個以上: 10%割引
    { minQuantity: 6, rate: 0.05 }     // 6-10個: 5%割引
  ]
};

function calculateVolumeDiscount(quantity: number): number {
  for (const { minQuantity, rate } of PRICING_CONFIG.volumeDiscounts) {
    if (quantity >= minQuantity) {
      return rate;
    }
  }
  return 0;
}

function calculateItemSubtotal(item: OrderItem): number {
  return item.price * item.quantity;
}

function applyDiscount(subtotal: number, discountRate: number): number {
  return subtotal * (1 - discountRate);
}

function addTax(amount: number): number {
  return amount * (1 + PRICING_CONFIG.taxRate);
}

function calculateItemTotal(item: OrderItem): number {
  const subtotal = calculateItemSubtotal(item);
  const discountRate = calculateVolumeDiscount(item.quantity);
  const discounted = applyDiscount(subtotal, discountRate);
  return addTax(discounted);
}

function calculateItemsTotal(items: OrderItem[]): number {
  return items.reduce((sum, item) => sum + calculateItemTotal(item), 0);
}

function calculateShippingFee(itemsTotal: number): number {
  if (itemsTotal >= PRICING_CONFIG.freeShippingThreshold) {
    return 0;
  }
  return PRICING_CONFIG.shippingFee;
}

function calculateOrderTotal(items: OrderItem[]): number {
  const itemsTotal = calculateItemsTotal(items);
  const shippingFee = calculateShippingFee(itemsTotal);
  return itemsTotal + shippingFee;
}
```

このリファクタリングにより、コメントは価格設定のビジネスロジックの背景のみに集中し、実装の詳細は関数名が説明しています。

### 例3：エラーハンドリング

#### ❌ Before: 不明瞭なエラー処理

```typescript
async function updateUser(id: string, data: any) {
  try {
    // ユーザーを取得
    const user = await db.users.findById(id);

    // ユーザーが存在しない場合
    if (!user) {
      throw new Error('User not found');
    }

    // メールアドレスの重複チェック
    if (data.email) {
      const existing = await db.users.findByEmail(data.email);
      if (existing && existing.id !== id) {
        throw new Error('Email already exists');
      }
    }

    // ユーザーを更新
    const updated = await db.users.update(id, data);

    return updated;
  } catch (error) {
    // エラーをログ出力
    console.error(error);
    throw error;
  }
}
```

#### ✅ After: 明確なエラー型 + 適切なコメント

```typescript
// カスタムエラー型の定義
class NotFoundError extends Error {
  constructor(resource: string, id: string) {
    super(`${resource} not found: ${id}`);
    this.name = 'NotFoundError';
  }
}

class DuplicateError extends Error {
  constructor(resource: string, field: string, value: string) {
    super(`${resource} with ${field}="${value}" already exists`);
    this.name = 'DuplicateError';
  }
}

interface UpdateUserData {
  email?: string;
  name?: string;
}

async function findUserOrThrow(id: string): Promise<User> {
  const user = await db.users.findById(id);
  if (!user) {
    throw new NotFoundError('User', id);
  }
  return user;
}

async function validateEmailUniqueness(
  email: string,
  excludeUserId: string
): Promise<void> {
  const existingUser = await db.users.findByEmail(email);

  if (existingUser && existingUser.id !== excludeUserId) {
    // GDPR対応: エラーメッセージに既存ユーザーの情報を含めない
    // 詳細: docs/privacy/error-messages.md
    throw new DuplicateError('User', 'email', email);
  }
}

async function updateUser(
  id: string,
  data: UpdateUserData
): Promise<User> {
  // 存在確認
  await findUserOrThrow(id);

  // メールアドレスの重複チェック
  if (data.email) {
    await validateEmailUniqueness(data.email, id);
  }

  try {
    return await db.users.update(id, data);
  } catch (error) {
    // データベースエラーは詳細をログに記録するが、
    // クライアントには一般的なエラーメッセージのみを返す
    logger.error('Failed to update user', {
      userId: id,
      updateData: data,
      error
    });
    throw error;
  }
}
```

## コメント文化の醸成

### チームでのコメント規約

チーム全体で一貫したコメントを書くために、明確な規約を設けましょう。

#### コメント規約の例

```markdown
# コメント規約

## 基本原則

1. **Why not What**: コードが何をしているかではなく、なぜそうしているかを説明する
2. **自己文書化を優先**: コメントの前に、コードの改善で表現できないか検討する
3. **正確性を保つ**: コードを変更したら、関連するコメントも更新する

## 必須コメント

以下の場合は、コメントを必ず書くこと:

- ビジネスロジックの背景や制約
- 非自明なアルゴリズムやパフォーマンス最適化
- セキュリティ上の考慮事項
- 暫定的な回避策（HACK）
- 外部APIの制限や制約
- レガシーコードとの互換性

## 避けるべきコメント

以下のコメントは避けること:

- コードを読めば分かること
- 変数や関数の型情報（型システムで表現）
- 履歴情報（Gitで管理）
- 無意味なコメント（"// コード", "// ここから"など）

## TODOコメント

TODOコメントには以下を含めること:

- 具体的な作業内容
- 優先度や期限（可能であれば）
- 担当者（可能であれば）
- チケット番号（JIRA-XXXまたは#XXX）

例:
```typescript
// TODO(@alice): ページネーション対応が必要
// 優先度: 中 / 期限: 2026-Q2 / チケット: JIRA-123
```

## レビュー時のチェック

コードレビューでは、以下をチェックすること:

- [ ] コメントは必要か？
- [ ] Whyを説明しているか？
- [ ] 正確か？
- [ ] 外部ドキュメントへのリンクは有効か？
- [ ] TODOにチケット番号があるか？
```

### コメントレビューの実践

コードレビュー時に、コメントも積極的にレビューしましょう。

#### レビューコメントの例

```typescript
// レビュー対象のコード
// ユーザーIDを取得
const userId = req.params.id;
```

**レビューコメント:**

> このコメントは、コードを読めば分かることを繰り返しているため、不要です。もしコメントを残すなら、なぜこのパラメータを使用するのか、またはこのIDに関する制約（例: UUIDv4形式である必要がある）などを説明してください。

```typescript
// レビュー対象のコード
// タイムアウトを3秒に設定
const TIMEOUT = 3000;
```

**レビューコメント:**

> なぜ3秒なのかを説明してください。例えば、APIプロバイダーのSLA、ユーザー体験の要件、パフォーマンステストの結果などを含めると、将来この値を変更する際の判断材料になります。

### ペアプログラミングとコメント

ペアプログラミング中に、以下の質問を投げかけることで、良いコメントのタイミングを見つけられます：

- 「なぜこの実装を選んだの？」
- 「この値はどこから来たの？」
- 「これは将来変更される可能性がある？」
- 「他の開発者が混乱しそうなポイントは？」

これらの質問への答えが、コメントに書くべき内容です。

## チェックリスト

この章で学んだ内容を確認しましょう：

### コメントの基本原則

- [ ] コメントの目的は「Why（なぜ）」を説明することだと理解している
- [ ] 「What（何を）」を説明するコメントは避けるべきと理解している
- [ ] 自己文書化コードを優先し、コメントは補完的に使うことを理解している

### 良いコメントの特徴

- [ ] ビジネスロジックの背景や制約を説明できる
- [ ] 技術的な制約やトレードオフを記録できる
- [ ] パフォーマンス最適化の理由を説明できる
- [ ] セキュリティ上の考慮事項を明記できる

### コードとコメントのバランス

- [ ] 変数名や関数名でコメントを不要にできることを理解している
- [ ] マジックナンバーを定数に抽出できる
- [ ] 複雑な条件式を関数に抽出できる
- [ ] コメントが必要な場合と不要な場合を判断できる

### TODOコメント

- [ ] TODO、FIXME、HACKの使い分けを理解している
- [ ] TODOにチケット番号や期限を含められる
- [ ] 暫定的な回避策を適切にドキュメント化できる

### 言語別の規約

- [ ] 使用している言語のコメント規約を理解している
- [ ] ドキュメントコメント（JSDoc、docstring等）の基本を理解している
- [ ] 型システムとコメントの使い分けを理解している

### メンテナンス

- [ ] コメントがコードと同期していることを確認できる
- [ ] 古いコメントを検出し、更新できる
- [ ] コードレビューでコメントもチェックできる
- [ ] 自動化ツールでコメント品質を維持できる

## 次のステップ

この章では、コードコメントのベストプラクティスを学びました。次の章では、コメントをさらに発展させた**ドキュメントコメント（JSDoc、TSDoc、SwiftDoc）**について学びます。

ドキュメントコメントは、コメントの一種ですが、特定のフォーマットに従うことで、以下のメリットがあります：

- IDE補完の強化
- APIリファレンスの自動生成
- 型情報の補完
- サンプルコードの提供

次章では、これらのドキュメントコメントの実践的な使い方を学びます。

### 関連リソース

コメントについてさらに学ぶためのリソース：

**公式ドキュメント:**
- [Google Style Guides](https://google.github.io/styleguide/)
- [TypeScript Coding Guidelines](https://github.com/Microsoft/TypeScript/wiki/Coding-guidelines)
- [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/)
- [PEP 257 - Docstring Conventions](https://www.python.org/dev/peps/pep-0257/)

**書籍:**
- "Clean Code" by Robert C. Martin - Chapter 4: Comments
- "Code Complete" by Steve McConnell - Chapter 32: Self-Documenting Code

**記事:**
- [Comments in Code - Best Practices](https://stackoverflow.blog/2021/12/23/best-practices-for-writing-code-comments/)
- [Why Comments Are Important](https://dave.cheney.net/2019/01/29/you-shouldnt-name-a-variable-the-same-as-a-type)

本章で学んだコメントの原則を実践し、将来のメンテナンス性を高めるドキュメント作成を心がけましょう。
