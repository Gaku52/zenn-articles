---
title: "実践レシピ"
---

# 実践レシピ

この章では、Claude Codeを使った日常開発の実践的なワークフローをレシピ形式で解説します。それぞれのレシピは、実際の開発現場で頻繁に発生するタスクに対応しており、すぐに活用できる具体的な手順を示しています。

## レシピ1: バグ修正ワークフロー

バグ修正は開発者が最も頻繁に行う作業の一つです。Claude Codeを使うことで、原因の特定から修正、テスト、コミットまでの一連の流れを効率化できます。

### 基本ワークフロー

```
① エラーログ・再現手順をClaudeに共有
② Claudeがコードベースを調査（Grep, Read）
③ 原因を特定して修正案を提示
④ 修正を実装→テスト実行→確認
⑤ コミット・PR作成
```

### 実践例

バグ報告を受けた場合の具体的な操作例を見てみましょう。

**ステップ1: エラー情報の共有**

```bash
claude
```

```
ユーザーログイン時に以下のエラーが発生します：

```
TypeError: Cannot read property 'token' of undefined
  at src/auth/login.ts:42:15
```

再現手順：
1. ログインフォームでメールアドレスとパスワードを入力
2. "ログイン"ボタンをクリック
3. エラーが発生してログインできない

期待する動作：
正常にログインしてダッシュボードに遷移する
```

**ステップ2-3: Claudeが調査と原因特定**

Claudeは自動的に以下のような調査を行います。

```
1. src/auth/login.ts:42付近のコードを読む
2. 関連するファイル（API呼び出し、型定義など）を調査
3. エラーの原因を特定
```

Claudeの応答例：

```
調査結果を報告します。

src/auth/login.ts:42でエラーが発生しているのは、API呼び出しが失敗した
場合に response.data が undefined になるためです。

src/api/auth.ts:28 を確認したところ、ネットワークエラー時のエラー
ハンドリングが不十分で、response が undefined のまま返されています。

修正案：
1. src/api/auth.ts でエラーハンドリングを追加
2. src/auth/login.ts でnullチェックを追加

修正を実装しますか？
```

**ステップ4: 修正の実装とテスト**

```
はい、修正してください。修正後にテストも実行してください。
```

Claudeが修正を実装し、以下を実行します。

```bash
npm test src/auth/login.test.ts
npm test src/api/auth.test.ts
```

**ステップ5: コミットとPR作成**

```
テストが通ったので、コミットしてPRを作成してください。
```

Claudeが以下を実行します。

```bash
git add src/auth/login.ts src/api/auth.ts
git commit -m "Fix login error handling for undefined response

- Add error handling in auth API for network failures
- Add null check in login component
- Fixes #123

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
git push -u origin fix-login-error
gh pr create --title "Fix login error handling" --body "..."
```

### 効果的なプロンプトのコツ

**具体的に書く**

❌ 悪い例：
```
ログインが動かないので修正して
```

✅ 良い例：
```
src/auth/login.tsの42行目でTypeError: Cannot read property 'token' of
undefinedが発生します。ユーザーがログインボタンをクリックしたときに
発生し、正常にログインできません。
```

**コンテキストを与える**

エラーログだけでなく、以下の情報も含めると効果的です。

- 再現手順
- 期待する動作
- 環境情報（ブラウザ、OSなど）
- 最近の変更内容

## レシピ2: 新機能追加ワークフロー

新機能の追加は、設計から実装、テストまで複数のステップを必要とします。Claude CodeのPlan Modeを活用することで、体系的に機能を追加できます。

### 基本ワークフロー

```
① 要件をClaudeに説明
② EnterPlanMode で設計（Claudeが計画を提示）
③ ユーザーが計画を承認
④ Claudeが実装（複数ファイル横断）
⑤ テスト作成→実行→修正ループ
⑥ PR作成
```

### 実践例

ユーザープロフィール編集機能を追加する場合の例です。

**ステップ1: 要件の説明**

```bash
claude --plan
```

```
ユーザープロフィール編集機能を追加したいです。

要件：
- ユーザーは自分の名前、メールアドレス、アバター画像を編集できる
- 編集画面は /profile/edit に配置
- バリデーション：メールアドレスは必須かつ有効な形式
- 画像アップロードはS3に保存
- 更新成功時はトースト通知を表示

技術スタック：
- React + TypeScript
- React Router v6
- TanStack Query
- Tailwind CSS
```

**ステップ2: Claudeが計画を提示**

```
計画を作成しました：

Phase 1: バックエンドAPI（推定時間: 30分）
- PUT /api/users/:id のエンドポイント追加
- 画像アップロード用のS3プリサインドURL取得API追加
- バリデーションロジックの実装

Phase 2: フロントエンド（推定時間: 45分）
- ProfileEditPage コンポーネント作成
- 画像アップロード用のhooks実装
- フォームバリデーション
- API統合（TanStack Query）

Phase 3: テスト（推定時間: 30分）
- APIのユニットテスト
- フロントエンドのコンポーネントテスト

Phase 4: ルーティング・統合（推定時間: 15分）
- ルート定義の追加
- ナビゲーションメニューへのリンク追加

この計画で進めますか？
```

**ステップ3: 計画の承認と実装**

```
良さそうです。Phase 1から順番に実装してください。
```

Claudeが各フェーズを順番に実装していきます。各フェーズ完了後に進捗を報告し、次のフェーズに進む前に確認を求めます。

**ステップ5: テストの実行**

```
Phase 3まで完了しました。テストを実行します。

$ npm test src/api/users.test.ts
$ npm test src/components/ProfileEditPage.test.tsx

全てのテストが通りました。
```

**ステップ6: PR作成**

```
実装が完了したので、PRを作成してください。
```

```bash
git add .
git commit -m "Add user profile edit feature

- Add PUT /api/users/:id endpoint
- Add S3 image upload support
- Create ProfileEditPage component
- Add form validation and error handling
- Add tests for API and components

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
git push -u origin feature-profile-edit
gh pr create --title "Add user profile edit feature" --body "..."
```

### Plan Modeの活用ポイント

**スコープを明確にする**

計画フェーズで実装範囲を明確にすることで、予想外の作業が発生するのを防げます。

**段階的に進める**

大きな機能はフェーズに分割し、各フェーズごとに確認することで、方向性のズレを早期に発見できます。

**テストを計画に含める**

実装と同時にテストも計画に含めることで、テストカバレッジを確保できます。

## レシピ3: コードレビュー

Pull Requestのレビューは、コードの品質を保つために重要な作業です。Claude Codeを使うことで、レビューの効率と品質を向上できます。

### 基本ワークフロー

```
① `--from-pr 123` でPRベースのセッション開始
② Claudeがdiffを分析
③ 問題点・改善提案を報告
④ 必要なら修正もその場で実行
```

### 実践例

**ステップ1: PRベースのセッション開始**

```bash
claude --from-pr 123
```

Claudeが自動的にPRの内容を読み込みます。

**ステップ2-3: レビューの実行**

```
このPRをレビューして、以下の観点でチェックしてください：
1. コードの品質（可読性、保守性）
2. セキュリティ上の問題
3. パフォーマンスの問題
4. テストカバレッジ
5. 型安全性
```

Claudeの応答例：

```
PR #123 のレビュー結果：

## 問題点

### 🔴 Critical: セキュリティリスク
src/api/auth.ts:45
パスワードがログに出力されています。

```typescript
console.log('Login attempt:', { email, password }); // ❌
```

### 🟡 Medium: パフォーマンス改善
src/components/UserList.tsx:78
useEffectの依存配列が不適切で、不要な再レンダリングが発生します。

### 🟢 Minor: コードの可読性
src/utils/date.ts:12
複雑なロジックにコメントがありません。

## 良い点
- 適切なエラーハンドリング
- 型定義が明確
- テストカバレッジ: 85%

## 提案
1. パスワードのログ出力を削除
2. useEffectの依存配列を修正
3. 複雑なロジックにコメントを追加

これらの修正を実装しますか？
```

**ステップ4: 修正の実行**

```
Critical と Medium の問題を修正してください。
```

Claudeが修正を実装し、コミットします。

### レビューの観点を CLAUDE.md に記述

プロジェクト固有のレビュー基準を CLAUDE.md に記述しておくと、一貫したレビューが可能になります。

```markdown
# Code Review Guidelines

## Security
- パスワードやAPIキーをログに出力しない
- XSS対策: ユーザー入力をサニタイズ
- CSRF対策: トークン検証

## Performance
- 不要な再レンダリングを避ける
- 大きなリストは仮想化を検討
- 画像は最適化してから使用

## Testing
- カバレッジ80%以上を目標
- Edgeケースのテストを含める
```

## レシピ4: リファクタリング

大規模なコードベースのリファクタリングは、手作業では時間がかかり、ミスも発生しやすい作業です。Claude CodeのBatch Modeを活用することで、効率的かつ安全にリファクタリングできます。

### 基本ワークフロー

```
① リファクタリングの方針をCLAUDE.mdに記述
② `/batch migrate src/ from class components to hooks`
③ 並列ワーカーが各ファイルを変換
④ テスト実行で破壊的変更を検出
```

### 実践例: Reactクラスコンポーネントから関数コンポーネントへの移行

**ステップ1: CLAUDE.md に方針を記述**

```markdown
# React Component Migration

クラスコンポーネントを関数コンポーネント + Hooks に移行する。

## 変換ルール
- `this.state` → `useState`
- `componentDidMount` → `useEffect(..., [])`
- `componentDidUpdate` → `useEffect`
- `this.props` → 関数の引数

## 注意点
- ライフサイクルメソッドの依存関係を正確に変換
- `this` の参照をすべて削除
- propsの型定義を保持
```

**ステップ2: Batch Modeでの実行**

```bash
claude
```

```
/batch migrate src/components/ from class components to hooks
```

Claudeが並列ワーカーを起動し、各ファイルを変換します。

```
Batch operation started: 45 files to process

[Worker 1] ✓ src/components/UserList.tsx
[Worker 2] ✓ src/components/Header.tsx
[Worker 3] ✓ src/components/Sidebar.tsx
...

Progress: 45/45 files processed
Success: 43 files
Skipped: 2 files (already using hooks)
```

**ステップ4: テストの実行**

```
全ファイルの変換が完了しました。テストを実行します。

$ npm test

Test Suites: 45 passed, 45 total
Tests:       234 passed, 234 total

全てのテストが通りました。
```

### 段階的なリファクタリング

大規模なリファクタリングは、段階的に進めることが重要です。

**Phase 1: 影響範囲の小さいコンポーネント**

```
/batch migrate src/components/atoms/ from class to hooks
```

**Phase 2: 中規模のコンポーネント**

```
/batch migrate src/components/molecules/ from class to hooks
```

**Phase 3: 大規模で複雑なコンポーネント**

複雑なコンポーネントは、Batch Modeではなく個別に対応することも検討します。

```
src/components/pages/Dashboard.tsx を関数コンポーネントに移行してください。
複雑なライフサイクルロジックがあるので、慎重に変換してください。
```

## レシピ5: テスト生成

テストの作成は重要ですが、時間のかかる作業です。Claude Codeを使うことで、既存コードからテストを自動生成できます。

### 基本ワークフロー

```
① `claude -p "Generate tests for src/auth/"`
② Claudeが既存コードを分析してテストを生成
③ テスト実行→修正ループ
```

### 実践例

**ステップ1: テスト生成の実行**

```bash
claude -p "Generate comprehensive tests for src/auth/login.ts"
```

Claudeが以下のようなテストを生成します。

```typescript
// src/auth/login.test.ts
import { describe, it, expect, vi } from 'vitest';
import { login } from './login';
import { authAPI } from '../api/auth';

vi.mock('../api/auth');

describe('login', () => {
  it('正常にログインできる', async () => {
    const mockResponse = { token: 'test-token', user: { id: 1 } };
    vi.mocked(authAPI.login).mockResolvedValue(mockResponse);

    const result = await login('test@example.com', 'password');

    expect(result).toEqual(mockResponse);
    expect(authAPI.login).toHaveBeenCalledWith({
      email: 'test@example.com',
      password: 'password',
    });
  });

  it('不正なメールアドレスでエラー', async () => {
    await expect(login('invalid', 'password')).rejects.toThrow(
      'Invalid email address'
    );
  });

  it('パスワードが短すぎる場合エラー', async () => {
    await expect(login('test@example.com', '123')).rejects.toThrow(
      'Password must be at least 8 characters'
    );
  });

  it('API呼び出しが失敗した場合のエラーハンドリング', async () => {
    vi.mocked(authAPI.login).mockRejectedValue(
      new Error('Network error')
    );

    await expect(
      login('test@example.com', 'password')
    ).rejects.toThrow('Network error');
  });
});
```

**ステップ3: テスト実行と修正**

```bash
npm test src/auth/login.test.ts
```

テストが失敗した場合、Claudeに修正を依頼します。

```
2つのテストが失敗しました。エラーメッセージを見て修正してください。

FAIL src/auth/login.test.ts
  ● login › パスワードが短すぎる場合エラー
    Expected substring: "Password must be at least 8 characters"
    Received message: "Password too short"
```

Claudeが実際のエラーメッセージに合わせてテストを修正します。

### テストカバレッジの向上

**Edge ケースのテスト追加**

```
src/utils/date.ts のテストカバレッジを100%にしてください。
現在カバーされていないEdgeケースを特定して、テストを追加してください。
```

Claudeが未カバーの分岐を特定し、テストを追加します。

**統合テストの生成**

```
src/features/auth/ の統合テストを生成してください。
ログインからダッシュボード表示までの一連のフローをテストします。
```

## レシピ6: ドキュメント生成

ドキュメントは重要ですが、メンテナンスが追いつかないことが多い領域です。Claude Codeを使うことで、コードから自動的にドキュメントを生成できます。

### 基本ワークフロー

```
① `claude -p "Generate API docs for src/api/"`
② Claudeがコードからドキュメントを生成
```

### 実践例: API ドキュメントの生成

```bash
claude -p "Generate API documentation for src/api/ in Markdown format"
```

Claudeが以下のようなドキュメントを生成します。

```markdown
# API Documentation

## Authentication API

### POST /api/auth/login

ユーザーのログイン認証を行います。

**Request:**

```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

**Response:**

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": 1,
    "email": "user@example.com",
    "name": "John Doe"
  }
}
```

**Errors:**

- `400 Bad Request`: Invalid email or password format
- `401 Unauthorized`: Invalid credentials
- `500 Internal Server Error`: Server error

### POST /api/auth/logout

...
```

### README の自動生成

```bash
claude -p "Generate a comprehensive README.md for this project"
```

Claudeがプロジェクトの構造を分析し、以下を含むREADMEを生成します。

- プロジェクトの概要
- インストール手順
- 使い方
- ディレクトリ構造
- 開発環境のセットアップ
- テストの実行方法
- デプロイ手順

### JSDoc コメントの追加

```
src/utils/ 以下の全ての関数にJSDocコメントを追加してください。
パラメータ、戻り値、例外を含めてください。
```

Claudeが各関数に適切なJSDocを追加します。

```typescript
/**
 * 日付を指定したフォーマットで文字列に変換します
 *
 * @param date - 変換する日付オブジェクト
 * @param format - フォーマット文字列（例: "YYYY-MM-DD"）
 * @returns フォーマットされた日付文字列
 * @throws {Error} 無効な日付オブジェクトが渡された場合
 *
 * @example
 * ```typescript
 * formatDate(new Date('2024-01-15'), 'YYYY-MM-DD')
 * // => "2024-01-15"
 * ```
 */
export function formatDate(date: Date, format: string): string {
  // ...
}
```

## レシピ7: 依存関係の更新

依存関係の更新は、破壊的変更への対応が必要で、時間のかかる作業です。Claude Codeを使うことで、更新から修正までを自動化できます。

### 基本ワークフロー

```
① `claude "Update all dependencies and fix breaking changes"`
② Claudeがpackage.jsonを更新→ビルド→テスト→修正
```

### 実践例: メジャーバージョンアップ

**React 17 → 18 への移行**

```bash
claude
```

```
Reactを17から18にアップデートして、破壊的変更に対応してください。

手順：
1. package.json の更新
2. 新しいルートAPI（createRoot）への移行
3. 自動バッチング対応の確認
4. 破壊的変更への対応
5. テストの実行と修正
```

Claudeが以下を実行します。

**ステップ1: package.json の更新**

```bash
npm install react@18 react-dom@18
npm install -D @types/react@18 @types/react-dom@18
```

**ステップ2: エントリーポイントの修正**

```typescript
// Before (React 17)
import ReactDOM from 'react-dom';
ReactDOM.render(<App />, document.getElementById('root'));

// After (React 18)
import { createRoot } from 'react-dom/client';
const root = createRoot(document.getElementById('root')!);
root.render(<App />);
```

**ステップ3-4: 破壊的変更への対応**

Claudeがビルドエラーを確認し、必要な修正を実行します。

```
以下の破壊的変更に対応しました：

1. ReactDOM.render → createRoot API
2. unmountComponentAtNode → root.unmount
3. children の型定義（ReactNode に統一）
4. useEffect のクリーンアップタイミング変更に対応
```

**ステップ5: テスト実行**

```bash
npm test
```

テストが失敗した場合、Claudeが原因を分析して修正します。

### パッチバージョンの一括更新

```
全ての依存関係を最新のパッチバージョンに更新してください。
破壊的変更がないことを確認してください。
```

```bash
npm update
npm audit fix
npm test
```

### セキュリティ脆弱性への対応

```
npm auditで検出された脆弱性を修正してください。
```

Claudeがセキュリティレポートを分析し、適切な対応を実行します。

```
セキュリティ脆弱性の修正：

1. lodash: 4.17.19 → 4.17.21 (Prototype Pollution)
2. axios: 0.21.1 → 1.6.0 (SSRF)
3. qs: 6.5.2 → 6.11.0 (Prototype Poisoning)

全ての脆弱性を修正しました。テストも通りました。
```

## 効果的なプロンプトの書き方

Claude Codeを最大限に活用するためには、効果的なプロンプトを書くことが重要です。

### 1. 具体的に書く

**❌ 曖昧な指示**

```
ログイン機能を修正して
```

**✅ 具体的な指示**

```
src/auth/login.ts の42行目で発生している TypeError: Cannot read
property 'token' of undefined を修正してください。

エラーは API レスポンスが undefined の場合に発生します。
ネットワークエラー時の適切なエラーハンドリングを追加してください。
```

### 2. コンテキストを与える

**❌ コンテキスト不足**

```
パフォーマンスを改善して
```

**✅ コンテキスト付き**

```
UserList コンポーネントのパフォーマンスを改善してください。

現状：
- 1000件のユーザーリストを表示
- スクロールが重い（60fps → 20fps）
- 全ユーザーを一度にレンダリング

改善案：
- 仮想スクロールの導入（react-virtual）
- メモ化の最適化
```

### 3. スコープを指定する

**❌ スコープが広すぎる**

```
全部リファクタリングして
```

**✅ スコープを限定**

```
src/components/atoms/ 以下のコンポーネントのみをリファクタリングして
ください。

対象：
- 10個のatom コンポーネント
- styled-components から Tailwind CSS へ移行
- props の型定義を厳密化
```

### 4. 制約条件を明示する

**❌ 制約なし**

```
新しいフォームコンポーネントを作って
```

**✅ 制約付き**

```
ユーザー登録フォームコンポーネントを作成してください。

制約条件：
- React Hook Form を使用
- Zod でバリデーション
- アクセシビリティ（ARIA属性）を考慮
- 既存のデザインシステム（src/design-system/）を使用
- レスポンシブ対応（Mobile First）
```

### 5. 期待する成果物を明確にする

**❌ 成果物が不明確**

```
API を実装して
```

**✅ 成果物が明確**

```
ユーザー管理 API を実装してください。

成果物：
1. src/api/users.ts にエンドポイント実装
   - GET /api/users (一覧取得)
   - GET /api/users/:id (詳細取得)
   - POST /api/users (作成)
   - PUT /api/users/:id (更新)
   - DELETE /api/users/:id (削除)

2. src/api/users.test.ts にユニットテスト
   - カバレッジ 80% 以上

3. docs/api/users.md に API ドキュメント
   - リクエスト/レスポンス例
   - エラーケース
```

## まとめ

この章で紹介した実践レシピをまとめます。

| レシピ名 | ユースケース | 主な機能 | 推定時間削減 |
|---------|------------|---------|------------|
| バグ修正ワークフロー | エラーの原因特定と修正 | Grep, Read, Edit | 50-70% |
| 新機能追加ワークフロー | 体系的な機能開発 | Plan Mode | 40-60% |
| コードレビュー | PR の品質チェック | --from-pr | 60-80% |
| リファクタリング | 大規模なコード変更 | Batch Mode | 70-90% |
| テスト生成 | テストコードの自動生成 | 既存コード分析 | 60-80% |
| ドキュメント生成 | API ドキュメント作成 | コードから自動生成 | 80-90% |
| 依存関係の更新 | ライブラリの更新と修正 | 破壊的変更への対応 | 50-70% |

### レシピ活用のベストプラクティス

1. **CLAUDE.md を活用する**: プロジェクト固有のルールやガイドラインを記述しておくことで、一貫した品質を保てます。

2. **段階的に進める**: 大きなタスクは小さなステップに分割し、各ステップごとに確認することで、方向性のズレを防げます。

3. **具体的に指示する**: 曖昧な指示ではなく、具体的な期待値と制約条件を明示することで、望ましい結果が得られます。

4. **テストを忘れない**: 実装と同時にテストも作成・実行することで、品質を保証できます。

5. **フィードバックループを回す**: Claude の提案を確認し、必要に応じて修正を依頼することで、最適な結果が得られます。

これらのレシピは、あなたの開発ワークフローに合わせてカスタマイズできます。実際のプロジェクトで試しながら、自分なりの効率的なワークフローを確立していきましょう。

次の章では、Claude Codeを使った高度なテクニックとカスタマイズ方法について解説します。
