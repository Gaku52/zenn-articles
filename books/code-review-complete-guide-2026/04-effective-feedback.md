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

## まとめ

効果的なフィードバックの要点:

- 具体的で実行可能
- 質問形式で丁寧に
- コンテキストを提供
- 優先度を明示
- ポジティブな表現

次章では、セルフレビューのテクニックを学びます。
