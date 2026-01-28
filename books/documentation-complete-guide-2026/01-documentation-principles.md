---
title: "ドキュメント作成の原則"
---

# ドキュメント作成の原則

## この章で学ぶこと

技術プロジェクトにおいて、ドキュメントは「書かれたコード」と同じくらい重要です。しかし、ドキュメントの種類、目的、読者は多岐にわたり、それぞれに適した書き方が存在します。

この章では、ドキュメント作成の基礎となる原則を学びます：

- ドキュメントの種類（README、API仕様書、ADR、コメント等）とそれぞれの目的
- 読者を意識した文章構成の考え方
- 良いドキュメントが持つべき特徴
- ドキュメントのライフサイクルと保守性

技術ドキュメントは、チームメンバー、未来の自分、オープンソースコミュニティ、そして製品を使うすべての人々にとって、プロジェクトを理解するための入り口です。適切なドキュメント戦略を持つことで、コードの保守性、プロジェクトの持続可能性、チームの生産性が大きく向上します。

## なぜドキュメントが重要か

### コードだけでは伝わらない情報

コードは「何を行うか（What）」と「どのように行うか（How）」を表現できますが、以下の情報は伝えられません：

- **なぜこの設計を選んだのか（Why）**
  - 他の選択肢を検討したか
  - どのようなトレードオフがあったか
  - どのような制約条件があったか

- **プロジェクトの全体像**
  - システムのアーキテクチャ
  - コンポーネント間の関係
  - データの流れ

- **使い方・環境構築**
  - どのように環境をセットアップするか
  - どのような前提条件が必要か
  - どのようにトラブルシューティングするか

### ドキュメントがもたらす価値

適切なドキュメントは、以下のような価値を提供します：

1. **オンボーディング時間の短縮**
   - 新しいメンバーがプロジェクトを理解する時間が大幅に短縮される
   - 質問対応の時間が減り、既存メンバーの負担が軽減される

2. **保守性の向上**
   - 過去の設計判断の理由が記録され、不要な変更を防げる
   - コードの意図が明確になり、リファクタリングが安全に行える

3. **コミュニケーションの効率化**
   - 非同期コミュニケーションが可能になる
   - 同じ質問への回答を繰り返さなくて済む

4. **知識の民主化**
   - 特定の人だけが知っている状態（属人化）を解消
   - チーム全体の技術レベルが向上

5. **意思決定の透明性**
   - なぜその選択をしたのかが記録される
   - 将来の変更時に判断材料となる

## ドキュメントの種類と目的

技術プロジェクトでは、様々な種類のドキュメントが必要です。それぞれの目的と読者を理解することが、効果的なドキュメント作成の第一歩です。

### 1. README

**目的**: プロジェクトの入り口となる最初のドキュメント

**主な読者**:
- プロジェクトを初めて見る人
- 環境構築を行うエンジニア
- プロジェクトの概要を知りたい関係者

**含まれるべき情報**:
- プロジェクトの概要（何を作っているか）
- インストール・セットアップ手順
- 基本的な使い方
- ライセンス情報
- 貢献方法（オープンソースの場合）

**例：プロジェクトの種類別**

```markdown
# OSSライブラリのREADME

## What is this?
日付操作を簡単にするJavaScriptライブラリ

## Installation
npm install date-helper

## Quick Start
import { formatDate } from 'date-helper';
console.log(formatDate(new Date(), 'YYYY-MM-DD'));

## Documentation
詳細なAPIドキュメントは docs/api.md を参照
```

```markdown
# WebアプリケーションプロジェクトのREADME

## Overview
ECサイトのフロントエンド・バックエンドシステム

## Prerequisites
- Node.js 18+
- PostgreSQL 15+
- Docker

## Setup
1. 依存関係のインストール: npm install
2. データベースのセットアップ: docker-compose up -d
3. 環境変数の設定: cp .env.example .env
4. マイグレーション実行: npm run migrate

## Development
npm run dev
```

```markdown
# CLIツールのREADME

## What
画像を一括でリサイズするCLIツール

## Installation
npm install -g image-resizer

## Usage
image-resizer --input ./images --output ./resized --width 800

## Options
--width, -w: 幅（デフォルト: 1024）
--quality, -q: 品質（1-100、デフォルト: 80）
```

### 2. API仕様書

**目的**: APIの使い方を明確に伝える

**主な読者**:
- APIを利用する開発者
- フロントエンド・バックエンド間の連携を行うエンジニア
- 外部パートナー企業のエンジニア

**含まれるべき情報**:
- エンドポイント一覧
- リクエスト・レスポンスの形式
- 認証方法
- エラーコードとその意味
- レート制限などの制約事項

**例：エンドポイントの記述**

```markdown
## GET /api/users/:id

ユーザー情報を取得します。

### Parameters
- `id` (required): ユーザーID（UUID形式）

### Response (200 OK)
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "John Doe",
  "email": "john@example.com",
  "createdAt": "2026-01-15T10:30:00Z"
}
```

### Error Response (404 Not Found)
```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User not found"
  }
}
```

### Authentication
Bearer token required in Authorization header
```

**良いAPI仕様書の特徴**:
- すべてのエンドポイントが網羅されている
- リクエスト・レスポンスの例が豊富
- エラーケースが明記されている
- 認証方法が明確
- データ型が明示されている

### 3. アーキテクチャドキュメント

**目的**: システムの構造と設計を理解する

**主な読者**:
- 新規参入エンジニア
- システム全体を把握したいエンジニア
- 技術選定を行うアーキテクト

**含まれるべき情報**:
- システムの全体像（コンポーネント図）
- 各コンポーネントの役割
- データフロー
- 技術スタック
- インフラ構成

**例：アーキテクチャの記述**

```markdown
## System Architecture

### Overview
3層アーキテクチャを採用：
- Presentation Layer: Next.js（SSR）
- Application Layer: NestJS（REST API）
- Data Layer: PostgreSQL

### Components

#### Frontend (Next.js)
- サーバーサイドレンダリング
- ユーザー認証（JWT）
- 状態管理（Zustand）

#### Backend (NestJS)
- RESTful API
- TypeORM（データベース接続）
- JWT認証・認可

#### Database (PostgreSQL)
- ユーザーデータ
- トランザクションデータ
- セッション情報

### Data Flow
1. ユーザーがフロントエンドにアクセス
2. フロントエンドがバックエンドAPIをコール
3. バックエンドがデータベースにクエリ
4. データベースが結果を返す
5. バックエンドがレスポンスを整形
6. フロントエンドが結果を表示
```

**図の活用**:

テキストだけでなく、図を使うことで理解が深まります：

```markdown
## Architecture Diagram

```mermaid
graph TD
    A[User Browser] --> B[Next.js Frontend]
    B --> C[NestJS Backend]
    C --> D[PostgreSQL]
    C --> E[Redis Cache]
    C --> F[S3 Storage]
```
```

### 4. ADR（Architecture Decision Records）

**目的**: なぜその技術選定・設計判断をしたのかを記録

**主な読者**:
- 未来の自分・チームメンバー
- 設計判断の理由を知りたいエンジニア
- システムをリファクタリングするエンジニア

**含まれるべき情報**:
- 意思決定の背景（Context）
- 選択した内容（Decision）
- その結果（Consequences）

**例：ADRの記述**

```markdown
# ADR-001: データベースにPostgreSQLを採用

## Status
Accepted

## Context
新規プロジェクトのデータベースを選定する必要がある。

### 要件
- ACID特性が必要
- JSONデータを扱う
- 将来的なスケーラビリティ
- チームの習熟度

### 検討した選択肢
1. PostgreSQL
2. MySQL
3. MongoDB

## Decision
PostgreSQLを採用する。

### 理由
- JSON型のネイティブサポート
- 豊富な拡張機能（PostGIS、pg_trgm等）
- チームメンバーの経験
- ACID特性の完全サポート
- レプリケーション機能

## Consequences

### Positive
- リレーショナル・ドキュメント両方の利点
- 複雑なクエリが可能
- データ整合性が保証される

### Negative
- 初期学習コスト（MySQL経験者向け）
- シャーディングはアプリケーション側で実装が必要

### Risks
- 大規模データでのパフォーマンス
  → 適切なインデックス設計で対応
```

**ADRの価値**:
- なぜその選択をしたのかが明確
- 将来、別の選択肢を検討する際の判断材料
- 同じ議論の繰り返しを防ぐ

### 5. コードコメント

**目的**: コードの意図・理由を説明

**主な読者**:
- コードを読むエンジニア
- コードをレビューするエンジニア
- 将来のメンテナンス担当者

**含まれるべき情報**:
- なぜこのように実装したのか（Why）
- 複雑なロジックの説明
- 注意すべきポイント
- 外部リソースへの参照

**例：良いコメント vs 悪いコメント**

```typescript
// ❌ 悪い例：コードを読めばわかること（What）
// ユーザーIDを取得
const userId = req.params.id;

// ユーザーを検索
const user = await prisma.user.findUnique({
  where: { id: userId }
});

// JSONを返す
res.json(user);
```

```typescript
// ✅ 良い例：なぜこうしているか（Why）

// GDPR対応のため、削除リクエストから90日以内のユーザーは
// 「削除予定」状態として扱い、完全削除は行わない
// 詳細: docs/gdpr-compliance.md
const user = await prisma.user.findUnique({
  where: { id: userId },
  // 削除予定ユーザーも含める
  include: { deletionRequest: true }
});

if (user.deletionRequest) {
  const daysSinceRequest =
    (Date.now() - user.deletionRequest.createdAt) / (1000 * 60 * 60 * 24);

  // 90日経過後に完全削除を実行
  if (daysSinceRequest > 90) {
    await permanentlyDeleteUser(userId);
    return res.status(404).json({ error: 'User not found' });
  }
}
```

**コメントの原則**:
- **Whatは書かない**: コードを見ればわかることは書かない
- **Whyを書く**: なぜこの実装を選んだのかを説明
- **外部リンクを活用**: 詳細なドキュメントへのリンクを記載

### 6. ドキュメントコメント（JSDoc/TSDoc/SwiftDoc）

**目的**: API・関数の使い方を自動生成可能な形式で記述

**主な読者**:
- ライブラリを使うエンジニア
- IDEの補完機能を使うエンジニア
- APIリファレンスの読者

**含まれるべき情報**:
- 関数の説明
- パラメータの型・説明
- 戻り値の型・説明
- 例外・エラー
- 使用例

**例：JSDoc/TSDocの記述**

```typescript
/**
 * ユーザーを作成します
 *
 * メールアドレスの重複チェックを行い、パスワードをハッシュ化してから
 * データベースに保存します。
 *
 * @param name - ユーザー名（1-50文字）
 * @param email - メールアドレス（RFC 5322準拠）
 * @param password - パスワード（8文字以上、英数字記号を含む）
 * @returns 作成されたユーザー（パスワードフィールドは含まれません）
 * @throws {ValidationError} バリデーションエラー
 * @throws {DuplicateEmailError} メールアドレスが既に使用されている
 *
 * @example
 * ```typescript
 * const user = await createUser(
 *   'John Doe',
 *   'john@example.com',
 *   'SecurePass123!'
 * );
 * console.log(user.id); // "550e8400-e29b-41d4-a716-446655440000"
 * ```
 */
export async function createUser(
  name: string,
  email: string,
  password: string
): Promise<User> {
  // バリデーション
  if (name.length < 1 || name.length > 50) {
    throw new ValidationError('Name must be 1-50 characters');
  }

  // メールアドレス重複チェック
  const existingUser = await prisma.user.findUnique({
    where: { email }
  });
  if (existingUser) {
    throw new DuplicateEmailError('Email already in use');
  }

  // パスワードハッシュ化
  const hashedPassword = await bcrypt.hash(password, 10);

  // ユーザー作成
  const user = await prisma.user.create({
    data: {
      name,
      email,
      password: hashedPassword,
    },
    // パスワードは返さない
    select: {
      id: true,
      name: true,
      email: true,
      createdAt: true,
    }
  });

  return user;
}
```

**ドキュメントコメントの利点**:
- IDEで補完時に表示される
- 型情報が明確になる
- 自動的にAPIリファレンスを生成できる
- 使用例が提示される

### 7. トラブルシューティングガイド

**目的**: よくある問題と解決方法を記録

**主な読者**:
- 問題に直面したエンジニア
- サポート担当者
- ユーザー

**含まれるべき情報**:
- よくある問題のパターン
- 原因の特定方法
- 解決手順
- 参考リンク

**例：トラブルシューティングの記述**

```markdown
## Troubleshooting

### データベース接続エラー

#### 症状
```
Error: Unable to connect to database
ECONNREFUSED 127.0.0.1:5432
```

#### 原因
PostgreSQLが起動していない、または接続情報が間違っている

#### 解決方法

1. PostgreSQLが起動しているか確認
```bash
docker ps | grep postgres
```

2. 起動していない場合は起動
```bash
docker-compose up -d postgres
```

3. 接続情報を確認
```bash
cat .env | grep DATABASE_URL
```

正しい形式: `postgresql://user:password@localhost:5432/dbname`

4. それでも解決しない場合
- ファイアウォール設定を確認
- PostgreSQLのログを確認: `docker logs postgres`

### マイグレーションエラー

#### 症状
```
Error: Migration failed: duplicate key value violates unique constraint
```

#### 原因
データベースに既存のデータがあり、マイグレーションと競合している

#### 解決方法

1. 開発環境の場合（データを失っても良い）
```bash
npm run db:reset  # データベースをリセット
npm run migrate   # マイグレーション実行
npm run seed      # サンプルデータ投入
```

2. 本番環境の場合（データを保持）
- マイグレーションファイルを修正
- 既存データと競合しないように調整
- 詳細: docs/migration-guide.md
```

### 8. チェンジログ（CHANGELOG）

**目的**: バージョンごとの変更内容を記録

**主な読者**:
- プロジェクトを利用するエンジニア
- アップグレードを検討するエンジニア
- リリースマネージャー

**含まれるべき情報**:
- 追加機能
- 修正したバグ
- 破壊的変更（Breaking Changes）
- 非推奨機能（Deprecated）

**例：CHANGELOGの記述**

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [2.0.0] - 2026-01-28

### Added
- ユーザー認証機能（JWT）
- パスワードリセット機能
- メール通知システム

### Changed
- API v1からv2へ移行
- データベーススキーマの最適化

### Deprecated
- `/api/v1/auth` エンドポイント（2026-06-30に削除予定）

### Removed
- レガシー認証システム

### Fixed
- メモリリークの修正（Issue #123）
- タイムゾーン処理のバグ修正

### Security
- パスワードハッシュアルゴリズムをbcrypt 10ラウンドに変更

### Breaking Changes
- `/api/auth` のレスポンス形式が変更
  - Before: `{ token: "..." }`
  - After: `{ accessToken: "...", refreshToken: "..." }`
```

## 読者を意識したドキュメント

すべてのドキュメントには「読者」がいます。読者の背景、目的、知識レベルを意識することで、効果的なドキュメントを作成できます。

### 読者の分類

**1. 技術レベルによる分類**

```markdown
# 初心者向け
対象: プログラミング経験が浅い、特定の技術スタックに不慣れ
必要な情報:
- 前提知識の説明
- 用語の定義
- ステップバイステップの手順
- スクリーンショット・図解

例:
## セットアップ

### 前提条件
Node.js: JavaScriptを実行するための環境
npm: Node.jsのパッケージマネージャー

### インストール手順
1. ターミナルを開く（Mac: Spotlight検索で「ターミナル」）
2. 以下のコマンドを入力してEnterキーを押す
```

```markdown
# 中級者向け
対象: 基本的な開発経験がある、フレームワークに慣れている
必要な情報:
- 概要と主要な手順
- 設定ファイルの例
- よくあるパターン

例:
## セットアップ

```bash
npm install
cp .env.example .env
npm run dev
```

環境変数の設定: `.env`ファイルを編集
```

```markdown
# 上級者向け
対象: 経験豊富、内部実装を理解したい
必要な情報:
- アーキテクチャの詳細
- パフォーマンス特性
- カスタマイズポイント

例:
## アーキテクチャ

依存性注入（DI）コンテナにNestJSを使用。
サービスはシングルトンとして管理され、起動時に初期化される。

カスタムプロバイダーの登録:
```typescript
@Module({
  providers: [
    {
      provide: 'CONFIG',
      useFactory: () => loadConfig(),
    },
  ],
})
```
```

**2. 役割による分類**

```markdown
# エンドユーザー向け
対象: プロダクトを使う人（非エンジニア含む）
必要な情報:
- 機能の使い方
- トラブルシューティング
- FAQ

例:
## 画像のアップロード方法

1. 「アップロード」ボタンをクリック
2. ファイルを選択（JPG、PNG形式、最大5MB）
3. 「送信」をクリック

エラーが出た場合:
- ファイルサイズが5MBを超えていないか確認
- 対応形式（JPG、PNG）か確認
```

```markdown
# 開発者向け
対象: コードを書く人
必要な情報:
- API仕様
- コード例
- 技術的な制約

例:
## 画像アップロードAPI

```http
POST /api/upload
Content-Type: multipart/form-data

file: (binary)
```

Response:
```json
{
  "url": "https://cdn.example.com/images/abc123.jpg",
  "size": 1024000,
  "format": "jpeg"
}
```

制約:
- 最大ファイルサイズ: 5MB
- 対応形式: image/jpeg, image/png
- レート制限: 100リクエスト/時
```
```

```markdown
# 運用担当者向け
対象: インフラ・デプロイを担当する人
必要な情報:
- デプロイ手順
- 監視ポイント
- バックアップ・復旧手順

例:
## デプロイ手順

1. ステージング環境でテスト
```bash
./deploy.sh staging
```

2. ヘルスチェック確認
```bash
curl https://staging.example.com/health
```

3. 本番デプロイ
```bash
./deploy.sh production
```

監視ポイント:
- CPU使用率（通常: 30%以下）
- メモリ使用率（通常: 60%以下）
- エラーレート（通常: 0.1%以下）
```

### 読者の目的による構成

**1. すぐに始めたい（Quick Start）**

```markdown
# Quick Start

5分でアプリを起動：

```bash
npm install
npm run dev
```

http://localhost:3000 にアクセス

詳細な説明は [セットアップガイド](./setup.md) を参照
```

**2. 全体を理解したい（Overview）**

```markdown
# Overview

## What is this?
リアルタイムチャットアプリケーション

## Key Features
- WebSocketによるリアルタイム通信
- E2E暗号化
- ファイル共有

## Architecture
- Frontend: React + TypeScript
- Backend: Node.js + Socket.io
- Database: PostgreSQL

詳細は [アーキテクチャドキュメント](./architecture.md) を参照
```

**3. 特定の機能を実装したい（Guides）**

```markdown
# How to Implement Authentication

## Step 1: Install dependencies
```bash
npm install jsonwebtoken bcrypt
```

## Step 2: Create auth service
```typescript
// src/auth/auth.service.ts
export class AuthService {
  async login(email: string, password: string) {
    // 実装
  }
}
```

## Step 3: Protect routes
```typescript
// src/middleware/auth.middleware.ts
export const authenticate = (req, res, next) => {
  // 実装
};
```

完全なコード例: [examples/auth](./examples/auth)
```

**4. トラブルを解決したい（Troubleshooting）**

```markdown
# Troubleshooting

## 目次
- [データベース接続エラー](#database-connection-error)
- [認証エラー](#authentication-error)
- [ビルドエラー](#build-error)

## Database Connection Error

### 症状
`Error: ECONNREFUSED`

### 解決方法
1. データベースが起動しているか確認
2. 接続情報（.env）を確認
3. ファイアウォール設定を確認

詳細: [データベースガイド](./database.md)
```

## 良いドキュメントの特徴

効果的なドキュメントには、共通する特徴があります。これらの特徴を意識することで、読みやすく、保守しやすいドキュメントを作成できます。

### 1. 明確な構成

**見出し階層が適切**

```markdown
# 最上位の見出し（1つだけ）

## 主要なセクション

### サブセクション

#### 詳細な説明
```

**目次がある（長いドキュメントの場合）**

```markdown
# API Documentation

## Table of Contents
- [Authentication](#authentication)
- [Endpoints](#endpoints)
  - [Users](#users)
  - [Posts](#posts)
- [Error Handling](#error-handling)
```

**論理的な順序**

```markdown
# セットアップガイド

## 1. 前提条件
必要なツールとバージョン

## 2. インストール
依存関係のインストール手順

## 3. 設定
環境変数・設定ファイルの編集

## 4. 起動
アプリケーションの起動方法

## 5. 動作確認
正しく動作しているか確認
```

### 2. 具体的な例

**抽象的な説明だけでなく、具体例を提示**

```markdown
❌ 悪い例:
環境変数を適切に設定してください。

✅ 良い例:
以下の環境変数を `.env` ファイルに設定してください:

```
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
JWT_SECRET=your-secret-key-here
PORT=3000
```
```

**実際に動作するコード例**

```markdown
❌ 悪い例:
適切なAPIコールを実装してください。

✅ 良い例:
```typescript
// ユーザー一覧を取得
const response = await fetch('https://api.example.com/users', {
  headers: {
    'Authorization': `Bearer ${accessToken}`,
    'Content-Type': 'application/json',
  }
});

const users = await response.json();
console.log(users);
```
```

### 3. 正確な情報

**情報源を明記**

```markdown
✅ 良い例:
Node.jsのイベントループは、シングルスレッドで非同期I/O処理を実現します。

参考: [Node.js公式ドキュメント - Event Loop](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
```

**一般的な傾向として記載**

```markdown
✅ 良い例:
一般的に、インデックスを適切に設定することで、クエリのパフォーマンスが向上する傾向があります。

ただし、実際の効果はデータ量、クエリの種類、テーブル構造により異なります。
```

**検証していないことは明記**

```markdown
✅ 良い例:
理論的には、キャッシュを導入することでデータベースへの負荷を軽減できます。

実際の効果は、キャッシュ戦略、データの更新頻度、アクセスパターンに依存します。
```

### 4. 検索可能性

**適切なキーワード**

```markdown
# データベースマイグレーション

タグ: database, migration, schema, prisma, typeorm

関連キーワード: スキーマ変更、テーブル作成、データ移行
```

**わかりやすい見出し**

```markdown
❌ 悪い例:
## その他

✅ 良い例:
## トラブルシューティング: よくあるエラーと解決方法
```

### 5. 保守性

**日付とバージョンの記載**

```markdown
# API Documentation

Version: 2.0.0
Last Updated: 2026-01-28

このドキュメントは API v2.0.0 に対応しています。
```

**変更履歴の記録**

```markdown
## Changelog

### 2026-01-28
- 認証エンドポイントの説明を追加
- エラーコードの表を更新

### 2026-01-15
- 初版作成
```

**TODO・FIXMEの明記**

```markdown
## Known Issues

- [ ] パフォーマンステストの結果を追加予定
- [ ] Swaggerドキュメントへのリンクを追加予定

TODO: エンドポイントのレート制限について記載
```

### 6. アクセシビリティ

**スクリーンショットに代替テキスト**

```markdown
![ログイン画面のスクリーンショット。メールアドレスとパスワードの入力フィールド、およびログインボタンが表示されている](./images/login.png)
```

**色だけに依存しない**

```markdown
❌ 悪い例:
赤いボタンをクリック

✅ 良い例:
「削除」ボタン（赤色）をクリック
```

**明確な言葉**

```markdown
❌ 悪い例:
ここをクリック

✅ 良い例:
[セットアップガイド](./setup.md) を参照
```

## ドキュメントのライフサイクル

ドキュメントは「一度書いたら終わり」ではありません。コードと同様に、継続的なメンテナンスが必要です。

### 1. 作成フェーズ

**タイミング**:
- プロジェクト開始時（README、アーキテクチャ）
- 機能実装時（API仕様書、コメント）
- 設計判断時（ADR）

**作成のポイント**:
- コードと同時に書く（後回しにしない）
- テンプレートを活用
- レビューを受ける

```markdown
# 新機能開発のワークフロー

1. 設計ドキュメント作成（ADR）
2. API仕様書の更新
3. コード実装
4. コメント・JSDocの記載
5. READMEの更新（必要に応じて）
6. コードレビュー + ドキュメントレビュー
```

### 2. レビューフェーズ

**レビューのポイント**:

```markdown
## ドキュメントレビューチェックリスト

### 正確性
- [ ] 情報が正確か
- [ ] コード例が動作するか
- [ ] リンクが有効か

### 完全性
- [ ] 必要な情報が網羅されているか
- [ ] エラーケースが記載されているか
- [ ] 前提条件が明記されているか

### 明確性
- [ ] 読者が理解できる表現か
- [ ] 専門用語に説明があるか
- [ ] 曖昧な表現がないか

### 構成
- [ ] 見出し階層が適切か
- [ ] 論理的な順序か
- [ ] 目次があるか（長文の場合）

### スタイル
- [ ] プロジェクトのスタイルガイドに準拠しているか
- [ ] トーンが一貫しているか
```

### 3. 更新フェーズ

**更新が必要なタイミング**:

```markdown
## ドキュメント更新のトリガー

### コード変更時（必須）
- APIの変更
- 機能の追加・削除
- 設定方法の変更
- 依存関係の更新

### 定期的な見直し（推奨）
- 四半期ごとにREADMEを見直し
- リンク切れのチェック
- 古い情報の更新

### フィードバック受領時
- ユーザーからの質問
- チームメンバーの指摘
- バグレポート

### バージョンアップ時
- メジャーバージョン: 全体的な見直し
- マイナーバージョン: 機能追加の反映
- パッチバージョン: バグ修正の反映
```

**更新の実践例**:

```markdown
# プルリクエスト時のドキュメント更新

## Before (PR提出前)
- コードのみ変更
- ドキュメントは古い情報のまま

## After (適切な更新)
```
feat: add user export feature

- 実装: ユーザーデータのCSVエクスポート機能
- API: GET /api/users/export を追加
- ドキュメント更新:
  - API仕様書にエンドポイントを追加
  - README の機能一覧に記載
  - CHANGELOG に追加
```
```

### 4. 廃止フェーズ

**非推奨の扱い**:

```markdown
## Deprecated Features

### /api/v1/auth (Deprecated)

⚠️ **非推奨**: このエンドポイントは2026-06-30に削除予定です。
新しいエンドポイント `/api/v2/auth` を使用してください。

#### 移行ガイド

Before (v1):
```http
POST /api/v1/auth
Content-Type: application/json

{
  "username": "john",
  "password": "secret"
}
```

After (v2):
```http
POST /api/v2/auth
Content-Type: application/json

{
  "email": "john@example.com",
  "password": "secret"
}
```

変更点:
- `username` → `email` に変更
- レスポンスに `refreshToken` が追加
```

**削除時の対応**:

```markdown
## Removed Features (v3.0.0)

### /api/v1/auth

このエンドポイントは v3.0.0 で削除されました。

代替手段:
- `/api/v2/auth` を使用

過去のドキュメント:
- v2.x のドキュメントは [docs/archive/v2](./archive/v2) を参照
```

## ドキュメントの配置戦略

ドキュメントをどこに配置するかは、アクセシビリティと保守性に大きく影響します。

### リポジトリ内の配置

**推奨構成**:

```
project/
├── README.md              # プロジェクトの入り口
├── CHANGELOG.md           # 変更履歴
├── CONTRIBUTING.md        # 貢献ガイド（OSSの場合）
├── LICENSE                # ライセンス
├── docs/                  # 詳細なドキュメント
│   ├── api/               # API仕様書
│   │   ├── overview.md
│   │   ├── authentication.md
│   │   ├── endpoints/
│   │   │   ├── users.md
│   │   │   └── posts.md
│   │   └── openapi.yaml
│   ├── architecture/      # アーキテクチャドキュメント
│   │   ├── overview.md
│   │   ├── components.md
│   │   └── diagrams/
│   ├── adr/               # Architecture Decision Records
│   │   ├── 001-database-selection.md
│   │   ├── 002-authentication-method.md
│   │   └── template.md
│   ├── guides/            # 実装ガイド
│   │   ├── setup.md
│   │   ├── deployment.md
│   │   └── troubleshooting.md
│   └── images/            # スクリーンショット・図
│       └── architecture-diagram.png
├── src/
│   └── (ソースコード + インラインコメント)
└── examples/              # サンプルコード
    ├── basic-usage/
    └── advanced-usage/
```

**配置の原則**:

```markdown
## ドキュメントの配置ルール

### README.md
場所: プロジェクトルート
目的: プロジェクトの入り口
内容:
- プロジェクト概要
- クイックスタート
- 詳細ドキュメントへのリンク

### docs/
場所: プロジェクトルート直下
目的: 詳細なドキュメント
分類:
- api: API仕様書
- architecture: システム設計
- adr: 設計判断の記録
- guides: 実装・運用ガイド

### インラインコメント
場所: ソースコード内
目的: コードの意図を説明
内容:
- なぜこの実装を選んだか
- 複雑なロジックの説明
- 外部ドキュメントへのリンク

### examples/
場所: プロジェクトルート直下
目的: 実際に動くサンプル
内容:
- 基本的な使い方
- 高度な使い方
- 各サンプルにREADME.md
```

### コードと別リポジトリにする場合

**メリット**:
- ドキュメントとコードのリリースサイクルを分離できる
- 非エンジニアがドキュメントを編集しやすい
- 大規模なドキュメントサイトを構築できる

**デメリット**:
- ドキュメントとコードの同期が難しい
- 更新の手間が増える
- バージョン管理が複雑

**適用例**:

```markdown
## ドキュメントサイト（別リポジトリ）

### ユースケース
- ユーザー向けの大規模なドキュメントサイト
- 複数プロジェクトの統合ドキュメント
- 非エンジニアが編集する必要がある

### 構成例
docs-site/
├── content/
│   ├── getting-started/
│   ├── api-reference/
│   ├── guides/
│   └── examples/
├── static/
│   └── images/
└── docusaurus.config.js  # または他のドキュメントツール

### 同期戦略
- コードリポジトリのCIからドキュメント生成
- 定期的に同期スクリプトを実行
- OpenAPI仕様から自動生成
```

## よくある間違いと対策

ドキュメント作成において、よくある間違いを認識し、避けることが重要です。

### 1. 「後で書く」という罠

```markdown
❌ よくあるパターン:
1. 機能を実装
2. 「ドキュメントは後で書こう」
3. 次の機能の実装へ
4. ドキュメントは永遠に書かれない

✅ 正しいアプローチ:
1. 設計ドキュメント作成（必要に応じて）
2. API仕様書の更新
3. 機能実装 + インラインコメント
4. READMEの更新
5. コードレビュー + ドキュメントレビュー
```

**対策**:

```markdown
## チーム内ルール

### プルリクエストの定義
「完成」の定義にドキュメント更新を含める

#### チェックリスト
- [ ] コードが動作する
- [ ] テストが通る
- [ ] ドキュメントが更新されている
  - [ ] API仕様書（該当する場合）
  - [ ] README（該当する場合）
  - [ ] CHANGELOG
  - [ ] インラインコメント

### CIチェック
ドキュメントの整合性をCIで確認
- リンク切れチェック
- コード例の動作確認
- スペルチェック
```

### 2. 曖昧な表現

```markdown
❌ 悪い例:
適切に設定してください。
いい感じに実装してください。
よしなにやってください。

✅ 良い例:
以下の環境変数を `.env` ファイルに設定してください:
- DATABASE_URL: PostgreSQLの接続URL
- JWT_SECRET: 32文字以上のランダムな文字列
- PORT: アプリケーションのポート番号（デフォルト: 3000）
```

### 3. 古い情報のまま放置

```markdown
❌ 問題:
README には古いセットアップ手順が記載されている
実際のコードは大幅に変更されている
ユーザーが混乱する

✅ 対策:
# セットアップ手順（v2.0.0以降）

⚠️ v1.x をお使いの場合は [v1のドキュメント](./archive/v1-setup.md) を参照

## インストール（v2.0.0）
```bash
# v2.0.0 から依存関係が変更されました
npm install --legacy-peer-deps
```
```

**定期的な見直しの仕組み**:

```markdown
## ドキュメントメンテナンス

### 四半期ごとのレビュー
- Q1 (1-3月): アーキテクチャドキュメントの見直し
- Q2 (4-6月): API仕様書の見直し
- Q3 (7-9月): README・セットアップガイドの見直し
- Q4 (10-12月): 全体的な整合性チェック

### 担当者
- Tech Lead: アーキテクチャドキュメント
- Backend Lead: API仕様書
- DevOps: デプロイ・インフラドキュメント
```

### 4. What と How ばかりで Why がない

```markdown
❌ 悪い例（Whatのみ）:
```typescript
// ユーザーを削除
await prisma.user.delete({ where: { id } });
```

❌ 悪い例（Howのみ）:
```typescript
// Prismaの delete メソッドを呼び出す
await prisma.user.delete({ where: { id } });
```

✅ 良い例（Whyを説明）:
```typescript
// GDPR対応: 削除リクエストから90日経過したユーザーを完全削除
// 参照: docs/gdpr-compliance.md
await prisma.user.delete({ where: { id } });
```
```

### 5. 専門用語だらけで読めない

```markdown
❌ 悪い例:
インジェクションコンテナにプロバイダーを登録し、
DIによってシングルトンインスタンスを解決します。

✅ 良い例:
依存性注入（Dependency Injection, DI）により、
各サービスを自動的に生成・管理します。

#### 依存性注入とは
サービスの生成を手動で行うのではなく、
フレームワークが自動的に必要なサービスを渡してくれる仕組みです。

例:
```typescript
// 手動（DIなし）
const database = new Database();
const userRepository = new UserRepository(database);
const userService = new UserService(userRepository);

// 依存性注入（DIあり）
// フレームワークが自動的に必要なサービスを渡してくれる
@Injectable()
class UserService {
  constructor(private userRepository: UserRepository) {}
}
```
```

## ドキュメント品質の評価基準

効果的なドキュメントかどうかを評価するための基準を定義しておくことが重要です。

### 定量的な指標

```markdown
## ドキュメント品質のメトリクス

### カバレッジ
- [ ] 全APIエンドポイントに仕様書がある
- [ ] 全パブリック関数にJSDoc/TSDocがある
- [ ] 重要な設計判断にADRがある

### 最新性
- [ ] 最終更新日が3ヶ月以内
- [ ] リンク切れがない
- [ ] コード例が最新バージョンで動作する

### 完全性
- [ ] セットアップ手順が完全（実際に実行できる）
- [ ] トラブルシューティングがある
- [ ] 前提条件が明記されている
```

### 定性的な指標

```markdown
## ドキュメントレビューの観点

### 新規メンバーテスト
新しいチームメンバーが:
- [ ] 1日でローカル環境を構築できるか
- [ ] 1週間でプロジェクトの全体像を理解できるか
- [ ] ドキュメントだけで基本的なタスクを完了できるか

### ユーザビリティ
- [ ] 目的の情報に3クリック以内でたどり着けるか
- [ ] 検索しやすいか（適切なキーワード）
- [ ] 目次・インデックスが整備されているか

### 理解しやすさ
- [ ] 専門用語に説明があるか
- [ ] 図・例が豊富か
- [ ] 段階的に説明されているか（簡単→複雑）
```

### フィードバックの収集

```markdown
## ドキュメントフィードバックの仕組み

### GitHub Issues
ラベル `documentation` でドキュメント関連の Issue を管理

テンプレート:
```
## ドキュメントの種類
- [ ] README
- [ ] API仕様書
- [ ] セットアップガイド
- [ ] その他

## 問題点
何がわかりにくかったか、何が足りないか

## 提案
どう改善すべきか
```

### アンケート
四半期ごとにチーム内アンケート:
- 最も役立ったドキュメント
- 最も改善が必要なドキュメント
- 新しく必要なドキュメント
```

## この章のまとめ

ドキュメント作成の原則として、以下のポイントを学びました：

### ドキュメントの種類
1. **README**: プロジェクトの入り口
2. **API仕様書**: APIの使い方を明確に
3. **アーキテクチャドキュメント**: システムの構造を可視化
4. **ADR**: 設計判断の記録
5. **コードコメント**: コードの意図を説明
6. **ドキュメントコメント**: 自動生成可能な形式
7. **トラブルシューティングガイド**: 問題解決の手順
8. **CHANGELOG**: バージョンごとの変更記録

### 良いドキュメントの特徴
- **明確な構成**: 見出し階層、目次、論理的な順序
- **具体的な例**: 実際に動作するコード、設定例
- **正確な情報**: 情報源の明記、検証された内容
- **検索可能性**: 適切なキーワード、わかりやすい見出し
- **保守性**: 日付とバージョン、変更履歴、TODO管理
- **アクセシビリティ**: 代替テキスト、明確な言葉

### 読者を意識する
- 技術レベル（初心者、中級者、上級者）
- 役割（エンドユーザー、開発者、運用担当者）
- 目的（すぐ始めたい、全体を理解したい、特定の機能を実装したい、トラブルを解決したい）

### ライフサイクル管理
1. **作成**: コードと同時に書く
2. **レビュー**: チェックリストで確認
3. **更新**: コード変更時、定期的な見直し
4. **廃止**: 非推奨の扱い、削除時の対応

### よくある間違い
- 「後で書く」という罠
- 曖昧な表現
- 古い情報のまま放置
- Whyがない（WhatとHowだけ）
- 専門用語だらけ

技術ドキュメントは、プロジェクトの成功に不可欠な要素です。コードと同様に、継続的にメンテナンスし、品質を向上させる必要があります。次章では、ドキュメント作成において最も重要な原則である「誠実性と正確性」について深く掘り下げます。

## チェックリスト

この章で学んだ内容を確認しましょう：

- [ ] ドキュメントの種類（README、API仕様書、ADR等）とそれぞれの目的を理解している
- [ ] 読者の技術レベル・役割・目的を意識してドキュメントを構成できる
- [ ] 良いドキュメントの特徴（明確な構成、具体例、正確性、検索可能性、保守性）を理解している
- [ ] ドキュメントのライフサイクル（作成、レビュー、更新、廃止）を理解している
- [ ] よくある間違い（後回し、曖昧な表現、Whyの欠如等）を避けられる
- [ ] ドキュメントの配置戦略（リポジトリ内の構成）を理解している
- [ ] ドキュメント品質を評価する基準（定量的・定性的指標）を知っている

## 次のステップ

次の第2章「誠実性と正確性」では、技術ドキュメントにおいて最も重要な原則を学びます：

- なぜ誠実性が重要なのか
- 実測データ・検証結果の正しい扱い方
- よくある間違いパターンとその対策
- Before/Afterの正しい書き方
- 読者の信頼を得るための具体的な方法

誠実で正確なドキュメントこそが、読者の信頼を得る唯一の方法です。次章で詳しく学びましょう。

---

**参考リンク**:
- [Keep a Changelog](https://keepachangelog.com/) - CHANGELOGの標準フォーマット
- [Semantic Versioning](https://semver.org/) - バージョン管理の基準
- [C4 Model](https://c4model.com/) - アーキテクチャ図の抽象度モデル
- [ADR GitHub](https://adr.github.io/) - Architecture Decision Recordsのリソース
