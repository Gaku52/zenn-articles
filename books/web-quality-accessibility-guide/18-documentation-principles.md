---
title: "ドキュメント作成原則 - 誠実性と正確性"
---

# ドキュメント作成原則 - 誠実性と正確性

## ドキュメントの重要性

技術ドキュメントは、プロジェクトの成功に不可欠です。適切なドキュメントは：

- チーム全体の理解を統一する
- 新メンバーのオンボーディングを加速する
- 保守性を向上させる
- 技術的負債を減らす

## 【最重要】誠実性の原則

### 実測データの扱い

読者は「著者が実際に検証した」と受け取ります。検証していないことを事実のように書くことは、読者の信頼を裏切る行為です。

#### ❌ 避けるべき表現

```markdown
❌ 「実測したところ、78%高速化しました」
   → 実際には測定していない

❌ 「メモリ使用量: 1,240MB → 85MB（93%削減）」
   → 具体的な数値は検証が必要

❌ 「このプロジェクトで効果を実証しました」
   → 実際のプロジェクトではない

❌ Before: 処理時間 5.2秒、メモリ1,240MB
   After: 処理時間 1.1秒、メモリ85MB
   → 具体的な数値は「実測した」と受け取られる
```

#### ✅ 推奨される表現

```markdown
✅ 「一般的に、Stream処理はメモリ使用量を削減できる傾向があります」

✅ 「公式ベンチマーク（https://...）によると、性能が向上します」

✅ 「期待される効果: メモリ使用量の安定化、処理速度の向上」

✅ 「理論的には、データ全体をメモリにロードしないため、効率的です」
```

### 情報源の明記

```markdown
## パフォーマンス

Node.js公式ベンチマーク（https://nodejs.org/...）によると:
- V8エンジンのJITコンパイラにより高速
- 非同期I/Oで高い同時実行性

※ 実際のパフォーマンスは、環境やデータ量により異なります
```

## ドキュメントの種類

### 1. README

プロジェクトの概要、セットアップ手順、使い方を記載します（次章で詳しく解説）。

### 2. API仕様書

APIのエンドポイント、リクエスト/レスポンス形式を記載します。

### 3. アーキテクチャ図

システムの構成、データフロー、コンポーネント関係を視覚化します。

### 4. コメント

コードの意図、制約、注意点を記載します。

## ドキュメント作成の原則

### 1. 読者を明確にする

誰が読むかによって、内容と詳細度が変わります。

```markdown
## 対象読者
- 開発者（中級以上）
- Reactの基本を理解している
- TypeScriptの経験がある

## 前提知識
- Node.js 20以上
- pnpmの基本的な使い方
- Git/GitHubの基本操作
```

### 2. 構造化する

見出し、リスト、コードブロックで構造化します。

```markdown
# プロジェクト名

## 概要

## 機能

## セットアップ

### 前提条件

### インストール

### 起動

## 使い方

## API仕様

## トラブルシューティング

## ライセンス
```

### 3. 例を含める

実際に動作するコード例を含めます。

```markdown
## 使い方

### 基本的な使い方

\`\`\`typescript
import { createUser } from './lib/user'

const user = await createUser({
  name: '山田太郎',
  email: 'yamada@example.com',
})
\`\`\`

### エラーハンドリング

\`\`\`typescript
try {
  const user = await createUser(data)
} catch (error) {
  if (error instanceof ValidationError) {
    console.error('バリデーションエラー:', error.message)
  }
}
\`\`\`
```

### 4. 最新に保つ

ドキュメントはコードと同時に更新します。

```markdown
<!-- 最終更新日を記載 -->
_Last updated: 2026-01-24_

<!-- バージョンを明記 -->
## Version 2.0.0

- ✨ 新機能: ダークモード対応
- 🐛 バグ修正: フォームバリデーション
- 📝 ドキュメント: セットアップ手順を追加
```

## Before/Afterの正しい書き方

### ❌ 悪い例: 具体的な数値（検証していない）

```markdown
### Before
\`\`\`typescript
const users = await fetchAll(); // 100,000件
// 処理時間: 8.3秒
// メモリ使用量: 1,240MB
\`\`\`

### After
\`\`\`typescript
const users = await fetchStream(); // 100,000件
// 処理時間: 2.1秒（74%改善）
// メモリ使用量: 85MB（93%削減）
\`\`\`
```

### ✅ 良い例: 原理・理論に基づく

```markdown
### Before: 改善の余地がある実装

\`\`\`typescript
// グローバルキャッシュが無限に増える
const cache = new Map();

app.get('/users/:id', async (req, res) => {
  if (!cache.has(req.params.id)) {
    const user = await fetchUser(req.params.id);
    cache.set(req.params.id, user);
  }
  res.json(cache.get(req.params.id));
});
\`\`\`

**問題点:**
- エントリが無限に増加
- メモリリークの原因
- 古いデータが残り続ける

### After: LRUキャッシュで改善

\`\`\`typescript
import LRU from 'lru-cache';

const cache = new LRU({
  max: 1000,
  ttl: 1000 * 60 * 10, // 10分
});

app.get('/users/:id', async (req, res) => {
  let user = cache.get(req.params.id);
  if (!user) {
    user = await fetchUser(req.params.id);
    cache.set(req.params.id, user);
  }
  res.json(user);
});
\`\`\`

**改善点:**
- 最大エントリ数を制限（1000件）
- 古いエントリは自動削除
- TTLでデータ鮮度を保証

**期待される効果:**
- メモリ使用量の安定化
- 長期間稼働しても安全
```

## コメントの書き方

### 良いコメント

```typescript
// ✅ 良い例: なぜそうするかを説明
// ユーザーIDは数値だが、APIでは文字列として扱う仕様のため変換
const userId = String(user.id)

// ✅ 良い例: 制約を説明
// TODO: 現状は100件までしか取得できない。将来的にページネーション対応が必要
const users = await fetchUsers({ limit: 100 })

// ✅ 良い例: 注意点を説明
// HACK: Safari 15以下では動作しないため、polyfillが必要
if (!('scrollTo' in window)) {
  // polyfill
}
```

### 悪いコメント

```typescript
// ❌ 悪い例: コードを読めば分かる
// userIdを取得
const userId = user.id

// ❌ 悪い例: 古くなったコメント
// バージョン1.0の処理（実際は2.0で変更済み）
const result = processV2(data)
```

## ドキュメントツール

### JSDoc

```typescript
/**
 * ユーザーを作成します
 *
 * @param data - ユーザーデータ
 * @returns 作成されたユーザー
 * @throws {ValidationError} バリデーションエラー
 *
 * @example
 * ```typescript
 * const user = await createUser({
 *   name: '山田太郎',
 *   email: 'yamada@example.com',
 * })
 * ```
 */
export async function createUser(data: CreateUserData): Promise<User> {
  // 実装
}
```

### TypeDoc

TypeScriptプロジェクトのAPI仕様書を自動生成します。

```bash
pnpm add -D typedoc

# package.json
{
  "scripts": {
    "docs": "typedoc src/index.ts"
  }
}
```

## まとめ

技術ドキュメントは、誠実性と正確性が最も重要です。検証していないことを事実のように書かないこと、情報源を明記することを徹底してください。

### チェックリスト

- 検証していないデータを「実測」として記載していない
- 架空の数値を具体的に記載していない
- 情報源（公式ドキュメント等）を明記している
- コード例が実際に動作する
- 読者が明確
- 構造化されている
- 最新に保たれている

## 次のステップ

次章では、READMEのベストプラクティスについて詳しく解説します。
