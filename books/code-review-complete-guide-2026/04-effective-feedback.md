---
title: "第4章 効果的なフィードバック技術"
---

# 第4章 効果的なフィードバック技術

本章では、建設的で効果的なフィードバックを提供するための具体的なテクニックを学びます。

## フィードバックの原則

### 1. 具体的で実行可能なフィードバック

```typescript
// 悪い例
const badFeedback = '

このコード良くないです。';

// 良い例
const goodFeedback = `
この関数は50行あり、3つの異なる責務を持っています。
単一責任原則に違反しているため、テストが困難です。

提案: 以下のように分割してください

\`\`\`typescript
// バリデーション部分を分離
function validateUser(user: User): ValidationResult {
  // バリデーションロジック
}

// 保存処理
async function saveUser(user: User): Promise<void> {
  const result = validateUser(user);
  if (!result.isValid) {
    throw new ValidationError(result.errors);
  }
  await db.users.save(user);
}
\`\`\`
`;
```

参考: [Google Engineering Practices - Writing Good Code Review Comments](https://google.github.io/eng-practices/review/reviewer/comments.html)

### 2. 質問形式で指摘する

```typescript
// 命令形（避けるべき）
const commandForm = 'このnullチェックを追加してください。';

// 質問形（推奨）
const questionForm = `
この場合、userがnullになる可能性はありませんか？
もしそうなら、呼び出し側でnullチェックが必要そうです。

\`\`\`typescript
if (!user) {
  throw new Error('User not found');
}
\`\`\`
`;
```

### 3. コンテキストを提供する

```typescript
const feedbackWithContext = `
このプロジェクトでは、エラーハンドリングにResult型パターンを使用しています。

参考実装: src/utils/result.ts

\`\`\`typescript
function saveUser(user: User): Result<void, Error> {
  try {
    db.users.save(user);
    return ok(undefined);
  } catch (error) {
    return err(error);
  }
}
\`\`\`
`;
```

## フィードバックテンプレート

### バグ指摘テンプレート

```markdown
**問題**: [具体的な問題の説明]

**影響**: [どのような問題が発生するか]

**再現手順**:
1. [ステップ1]
2. [ステップ2]
3. [結果]

**修正案**:
\`\`\`[language]
[修正後のコード]
\`\`\`

**参考**: [関連ドキュメント・Issue]
```

### 改善提案テンプレート

```markdown
**現状**: [現在の実装]

**提案**: [改善案]

**理由**: [なぜ改善すべきか]

**メリット**:
- [メリット1]
- [メリット2]

**実装例**:
\`\`\`[language]
[コード例]
\`\`\`

**参考**: [ベストプラクティス・ドキュメント]
```

参考: [Conventional Comments](https://conventionalcomments.org/)

### 質問テンプレート

```markdown
**質問**: [理解したい点]

**背景**: [なぜ質問するか]

**提案** (もしあれば):
[代替案]
```

### 称賛テンプレート

```markdown
**Good**: [良かった点]

**理由**: [なぜ良いか]

例: このエラーハンドリングは完璧です！
エッジケースまで考慮されており、ユーザーにも分かりやすいメッセージが表示されますね。
```

## 優先度の明示

### ラベルの使用

```typescript
enum FeedbackPriority {
  CRITICAL = '[Critical]',  // 必ず修正が必要
  HIGH = '[High]',          // 修正を強く推奨
  MEDIUM = '[Medium]',      // 修正を推奨
  LOW = '[Low]',            // 修正は任意
  NIT = '[Nit]',            // 細かい指摘
  QUESTION = '[Question]',  // 質問
  PRAISE = '[Praise]',      // 称賛
}

// 使用例
const examples = [
  '[Critical] セキュリティ: SQLインジェクションの脆弱性があります',
  '[High] パフォーマンス: N+1クエリが発生しています',
  '[Medium] 可読性: この変数名は意図が不明確です',
  '[Nit] タイポ: "recieve" -> "receive"',
  '[Question] この処理の意図を教えてください',
  '[Praise] このテストケースは素晴らしいです！',
];
```

## コミュニケーションのベストプラクティス

### 1. ポジティブな表現を使う

```typescript
// ネガティブ（避けるべき）
const negative = 'このコードは間違っています。';

// ポジティブ（推奨）
const positive = '別のアプローチを検討してみてはいかがでしょうか。';
```

### 2. "We"を使う

```typescript
// 個人攻撃に聞こえる
const you = 'あなたのコードにバグがあります。';

// チームとして
const we = 'このコードを一緒に改善しましょう。';
```

### 3. 学習機会として捉える

```typescript
const learningOpportunity = `
この実装は初めて見ました。興味深いアプローチですね。

質問: なぜこの方法を選択されたのですか？
他のアプローチと比較して、どのようなメリットがありますか？
`;
```

## 良い例・悪い例の対比

効果的なフィードバックを学ぶために、実際の例を見ていきましょう。

### 例1: 変数名の指摘

```typescript
// 悪い例
const badComment = 'この変数名は良くないです。';

// 良い例
const goodComment = `
[Medium] 可読性: 変数名\`d\`は意図が不明確です。

この変数は「ユーザー登録日からの経過日数」を表しているようですが、
\`d\`という名前からは意図が読み取れません。

提案: 以下のように意図を明確にしてはいかがでしょうか。

\`\`\`typescript
const daysSinceRegistration = calculateDays(user.registeredAt, new Date());
\`\`\`

参考: プロジェクトの命名規約（docs/naming-conventions.md）
`;
```

### 例2: パフォーマンスの指摘

```typescript
// 悪い例
const badComment = 'このコードは遅いです。';

// 良い例
const goodComment = `
[High] パフォーマンス: このループ内でデータベースクエリが実行されています（N+1問題）。

現在の実装:
\`\`\`typescript
for (const user of users) {
  const profile = await db.profiles.findOne({ userId: user.id });
  // ...
}
\`\`\`

ユーザーが100人いる場合、101回のクエリが発生します。

提案: 一括取得に変更してはいかがでしょうか。

\`\`\`typescript
const userIds = users.map(u => u.id);
const profiles = await db.profiles.find({ userId: { $in: userIds } });
const profileMap = new Map(profiles.map(p => [p.userId, p]));

for (const user of users) {
  const profile = profileMap.get(user.id);
  // ...
}
\`\`\`

この変更により、クエリ数が2回に削減されることが期待されます。
`;
```

### 例3: セキュリティの指摘

```typescript
// 悪い例
const badComment = 'SQLインジェクションの脆弱性があります。修正してください。';

// 良い例
const goodComment = `
[Critical] セキュリティ: SQLインジェクションの脆弱性があります。

現在の実装:
\`\`\`typescript
const query = \`SELECT * FROM users WHERE name = '\${userName}'\`;
\`\`\`

ユーザーが \`'; DROP TABLE users; --\` のような入力をすると、
データベースが破壊される可能性があります。

修正（必須）: プレースホルダーを使用してください。

\`\`\`typescript
const query = 'SELECT * FROM users WHERE name = ?';
const result = await db.execute(query, [userName]);
\`\`\`

参考:
- [OWASP SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- プロジェクトのセキュリティガイドライン（docs/security.md）
`;
```

### 例4: エラーハンドリングの指摘

```typescript
// 悪い例
const badComment = 'エラーハンドリングがないです。';

// 良い例
const goodComment = `
[High] エラーハンドリング: API呼び出しの失敗時の処理が考慮されていません。

質問: ネットワークエラーやタイムアウトが発生した場合、どのような挙動を期待していますか？

提案: ユーザーに適切なエラーメッセージを表示し、リトライ可能にすることを検討してください。

\`\`\`typescript
try {
  const response = await fetch('/api/users');
  if (!response.ok) {
    throw new Error(\`HTTP error! status: \${response.status}\`);
  }
  const data = await response.json();
  return data;
} catch (error) {
  console.error('Failed to fetch users:', error);
  // ユーザーに通知
  showErrorNotification('ユーザー情報の取得に失敗しました。もう一度お試しください。');
  // エラーログを送信
  logError(error);
  throw error; // または適切なフォールバック処理
}
\`\`\`

参考: 第6章「機能性レビュー」のエラーハンドリングセクション
`;
```

### 例5: テストの指摘

```typescript
// 悪い例
const badComment = 'テストを追加してください。';

// 良い例
const goodComment = `
[High] テスト: この新機能に対するテストケースが見当たりません。

特に以下のケースのテストが必要と思われます:

1. 正常系: 有効な入力で期待通りの結果が返ること
2. 異常系: 不正な入力でエラーが適切にハンドリングされること
3. 境界値: 空文字列、null、undefined、最大長などの境界ケース

テスト例:
\`\`\`typescript
describe('validateEmail', () => {
  it('有効なメールアドレスを受け入れる', () => {
    expect(validateEmail('user@example.com')).toBe(true);
  });

  it('無効なメールアドレスを拒否する', () => {
    expect(validateEmail('invalid-email')).toBe(false);
  });

  it('nullを適切に処理する', () => {
    expect(validateEmail(null)).toBe(false);
  });
});
\`\`\`

参考: 第11章「テストレビュー」
`;
```

### 例6: コードの複雑度

```typescript
// 悪い例
const badComment = 'このコードは複雑すぎます。';

// 良い例
const goodComment = `
[Medium] 可読性: この関数は循環的複雑度が高く、理解が困難です。

現在の関数は以下を1つで行っています:
1. 入力のバリデーション
2. データの変換
3. 外部API呼び出し
4. 結果の整形
5. エラーハンドリング

提案: 単一責任原則に従って、責務ごとに分割してはいかがでしょうか。

\`\`\`typescript
// バリデーション
function validateInput(input: UserInput): ValidationResult {
  // ...
}

// データ変換
function transformToApiFormat(input: UserInput): ApiRequest {
  // ...
}

// API呼び出し
async function callUserApi(request: ApiRequest): Promise<ApiResponse> {
  // ...
}

// メイン処理
async function processUser(input: UserInput): Promise<User> {
  const validation = validateInput(input);
  if (!validation.isValid) {
    throw new ValidationError(validation.errors);
  }

  const apiRequest = transformToApiFormat(input);
  const apiResponse = await callUserApi(apiRequest);

  return apiResponse.toUser();
}
\`\`\`

この分割により、各関数が単一の責務を持ち、テストも容易になります。

参考: 第7章「設計・アーキテクチャレビュー」
`;
```

### 例7: 型の安全性（TypeScript）

```typescript
// 悪い例
const badComment = 'any型を使わないでください。';

// 良い例
const goodComment = `
[Medium] 型安全性: \`any\`型の使用により型チェックが無効化されています。

現在の実装:
\`\`\`typescript
function processData(data: any) {
  return data.map((item: any) => item.value);
}
\`\`\`

この実装では、\`data\`が配列でない場合や、\`item.value\`が存在しない場合に
実行時エラーが発生します。

提案: 適切な型定義を追加してください。

\`\`\`typescript
interface DataItem {
  value: string;
  // 他のプロパティ...
}

function processData(data: DataItem[]): string[] {
  return data.map(item => item.value);
}
\`\`\`

型定義により、コンパイル時にエラーを検出でき、IDEの補完も有効になります。

参考: 第12章「TypeScript/JavaScriptレビューガイド」
`;
```

### 例8: 非同期処理の誤り

```typescript
// 悪い例
const badComment = 'awaitが抜けています。';

// 良い例
const goodComment = `
[Critical] バグ: 非同期処理の待機漏れにより、意図しない動作になります。

現在の実装:
\`\`\`typescript
async function saveUser(user: User) {
  db.users.save(user); // awaitなし
  console.log('User saved'); // この時点ではまだ保存されていない
}
\`\`\`

\`db.users.save()\`はPromiseを返しますが、awaitしていないため、
保存が完了する前に次の処理が実行されます。

修正（必須）:
\`\`\`typescript
async function saveUser(user: User) {
  await db.users.save(user);
  console.log('User saved'); // 保存完了後に実行される
}
\`\`\`

また、エラーハンドリングも追加することを推奨します:

\`\`\`typescript
async function saveUser(user: User) {
  try {
    await db.users.save(user);
    console.log('User saved');
  } catch (error) {
    console.error('Failed to save user:', error);
    throw error;
  }
}
\`\`\`

参考: 第12章「TypeScript/JavaScriptレビューガイド」の非同期処理セクション
`;
```

### 例9: メモリリーク（Swift）

```swift
// 悪い例
let badComment = "メモリリークがあります。"

// 良い例
let goodComment = """
[High] メモリ管理: 強参照循環によりメモリリークが発生します。

現在の実装:
```swift
class UserViewController: UIViewController {
    var viewModel: UserViewModel?

    override func viewDidLoad() {
        super.viewDidLoad()
        viewModel?.onUpdate = {
            self.updateUI() // selfを強参照
        }
    }
}
```

クロージャ内で`self`を強参照しているため、ViewControllerとViewModelの間に
強参照循環が発生し、どちらも解放されません。

修正（必須）: キャプチャリストで`weak`または`unowned`を使用してください。

```swift
viewModel?.onUpdate = { [weak self] in
    self?.updateUI()
}
```

または、selfが必ず存在する場合:
```swift
viewModel?.onUpdate = { [unowned self] in
    self.updateUI()
}
```

参考:
- [Swift Documentation - Automatic Reference Counting](https://docs.swift.org/swift-book/LanguageGuide/AutomaticReferenceCounting.html)
- 第14章「Swiftレビューガイド」
"""
```

### 例10: 称賛の例

```typescript
// 悪い例
const badComment = 'いいですね。';

// 良い例
const goodComment = `
[Praise] このエラーハンドリングは素晴らしいです！

特に良い点:
1. ユーザーフレンドリーなエラーメッセージ
2. ログへの詳細な情報記録
3. リトライロジックの実装
4. エラー種別ごとの適切な処理

\`\`\`typescript
catch (error) {
  if (error instanceof NetworkError) {
    // ネットワークエラーはリトライ
    return retryWithBackoff(() => fetchData());
  } else if (error instanceof ValidationError) {
    // バリデーションエラーはユーザーに通知
    showError('入力内容を確認してください');
  } else {
    // 予期しないエラーは記録して管理者に通知
    logError(error);
    notifyAdmin(error);
    showError('エラーが発生しました。しばらく経ってから再度お試しください');
  }
}
\`\`\`

このパターンは他の箇所でも参考にできそうです。
チームの知見として共有してもよいでしょうか？
`;
```

### 例11: 質問とディスカッション

```typescript
// 悪い例
const badComment = 'この実装で大丈夫ですか？';

// 良い例
const goodComment = `
[Question] この実装方法について教えてください。

質問: ここでReduxではなくReact Contextを選択された理由を教えていただけますか？

背景: プロジェクトの他の箇所ではReduxを使用しており、
一貫性の観点から気になりました。

もちろん、React Contextの方が適切な場合もありますので、
設計意図を理解したいと思っています。

例えば、以下のような理由が考えられますが:
- この状態はコンポーネントツリーの一部でのみ使用される
- Reduxのボイラープレートを避けたい
- パフォーマンス要件が異なる

どのような判断基準で選択されたか共有いただけると、
チーム全体の学びになりそうです。
`;
```

### 例12: ドキュメント追加の提案

```typescript
// 悪い例
const badComment = 'コメントを書いてください。';

// 良い例
const goodComment = `
[Medium] ドキュメント: この複雑なアルゴリズムにドキュメントを追加してはいかがでしょうか。

この実装は非常に効率的ですが、初見では理解が難しいと感じました。
将来のメンテナンスのため、以下の情報を追加することを提案します:

\`\`\`typescript
/**
 * ユーザーの類似度スコアを計算します。
 *
 * アルゴリズム: コサイン類似度を使用
 * 計算量: O(n) ここでnはユーザーの興味リストの長さ
 *
 * @param user1 - 比較対象のユーザー1
 * @param user2 - 比較対象のユーザー2
 * @returns 類似度スコア（0-1の範囲、1が最も類似）
 *
 * @example
 * const similarity = calculateSimilarity(userA, userB);
 * if (similarity > 0.8) {
 *   recommendUsers(userA, userB);
 * }
 */
function calculateSimilarity(user1: User, user2: User): number {
  // 実装...
}
\`\`\`

特に、アルゴリズムの選択理由や計算量についての情報があると、
将来的な最適化の際に役立ちそうです。

参考: 第8章「可読性・保守性レビュー」のドキュメントセクション
`;
```

## 言語別フィードバック例

### TypeScript

```typescript
// リテラル型の活用
const feedback = `
[Medium] 型安全性: 文字列リテラル型を活用すると、より安全になります。

現在:
\`\`\`typescript
type Status = string; // 任意の文字列を許容
\`\`\`

提案:
\`\`\`typescript
type Status = 'pending' | 'approved' | 'rejected'; // 特定の値のみ許容
\`\`\`

この変更により、タイポによるバグを防ぐことが期待されます。

参考: [TypeScript Handbook - Literal Types](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#literal-types)
`;
```

### Python

```python
# 型ヒントの活用
feedback = """
[Medium] 型安全性: 型ヒントを追加すると、IDEの支援が受けられます。

現在:
```python
def process_users(users):
    return [user.name for user in users]
```

提案:
```python
from typing import List

def process_users(users: List[User]) -> List[str]:
    return [user.name for user in users]
```

型ヒントにより、以下のメリットが期待されます:
- IDEの自動補完
- mypyによる静的型チェック
- ドキュメントとしての役割

参考:
- [PEP 484 - Type Hints](https://www.python.org/dev/peps/pep-0484/)
- 第13章「Pythonレビューガイド」
"""
```

### Swift

```swift
// Optional型の適切な処理
let feedback = """
[Medium] Optional処理: force unwrapは避け、安全なunwrapを使用してください。

現在:
```swift
let userName = user.name! // クラッシュの可能性
```

提案:
```swift
guard let userName = user.name else {
    print("User name is missing")
    return
}
// userNameを安全に使用
```

または、nil coalescing演算子を使用:
```swift
let userName = user.name ?? "Unknown"
```

参考:
- [Swift Documentation - Optional Chaining](https://docs.swift.org/swift-book/LanguageGuide/OptionalChaining.html)
- 第14章「Swiftレビューガイド」
"""
```

## フィードバック作成チェックリスト

レビューコメントを投稿する前に、以下を確認してください。

### 内容の確認

- [ ] 問題点が具体的に説明されているか
- [ ] なぜ問題なのか、理由が明記されているか
- [ ] 修正案または代替案が提示されているか
- [ ] 参考資料・ドキュメントへのリンクがあるか
- [ ] コード例が含まれているか（該当する場合）

### 表現の確認

- [ ] 優先度ラベルが明記されているか（[Critical], [High], [Medium], [Nit]など）
- [ ] 命令形ではなく、質問形・提案形になっているか
- [ ] 「あなた」ではなく「このコード」と表現しているか
- [ ] ポジティブな表現を使っているか
- [ ] 簡潔で分かりやすい文章か

### コンテキストの確認

- [ ] どのファイル・行番号の話か明確か
- [ ] プロジェクトの規約・ガイドラインに言及しているか
- [ ] 必要に応じて関連する章・セクションを参照しているか
- [ ] 影響範囲が説明されているか（該当する場合）

### 建設的フィードバックの確認

- [ ] 学習機会として捉えられる内容か
- [ ] 良い点も指摘しているか（該当する場合）
- [ ] チーム全体の知見として共有できる内容か
- [ ] 相手が次のアクションを取れる内容か

## まとめ

効果的なフィードバックの要点:

- 具体的で実行可能
- 質問形式で丁寧に
- コンテキストを提供
- 優先度を明示
- ポジティブな表現
- コード例と参考資料を含める
- 理由を明確に説明する
- 建設的な改善提案を行う

### 実践のヒント

1. **テンプレートを活用する**: 本章のテンプレートをコピーして、自分のチーム用にカスタマイズしましょう。

2. **過去のレビューを振り返る**: 自分の過去のレビューコメントを読み返し、改善点を見つけましょう。

3. **フィードバックのフィードバックを得る**: チームメンバーに、自分のレビューコメントについてフィードバックをもらいましょう。

4. **継続的に改善する**: レビューのたびに、少しずつフィードバックの質を向上させることを意識しましょう。

次章では、セルフレビューのテクニックを学びます。レビューを依頼する前に自分で確認すべきポイントを詳しく解説します。
