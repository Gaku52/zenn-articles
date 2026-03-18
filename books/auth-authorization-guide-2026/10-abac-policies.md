---
title: "ABAC とポリシーベース認可"
---

## ABAC とは何か

ABAC（Attribute-Based Access Control）は、アクセス制御の判断を「属性」の評価に基づいて行うモデルです。前章で扱った RBAC がユーザーに割り当てられた「ロール」で権限を決めるのに対し、ABAC ではユーザー・リソース・アクション・環境という 4 種類の属性を動的に組み合わせて、アクセスの可否を判断します。

ABAC の最大の強みは、事前にロールを網羅的に定義する必要がない点です。「東京オフィスの部長以上が、営業時間内に、自部署の機密文書を閲覧できる」といった複合的な条件をポリシーとして自然に記述できます。

```
RBAC と ABAC の考え方の違い:

  RBAC:
    admin は全記事を編集できる
    → ロール単位の粗い制御
    → ロールが増えると管理が大変（ロール爆発）

  ABAC:
    記事の作者は自分の記事を編集できる
    同じ部署のマネージャーは部下の評価を閲覧できる
    営業時間内のみデータをエクスポートできる
    → 属性の組み合わせで細かい制御が可能
```

---

## 4 つの属性カテゴリ

ABAC では、アクセス制御の判断材料を 4 つのカテゴリに分類します。それぞれの属性が「誰が」「何に」「何を」「どのような状況で」を表現します。

### Subject（主体）属性

アクセスを要求するユーザーやシステムに関する情報です。

- ユーザー ID、ロール、部署、役職
- メールアドレス、入社日、保有資格
- 所属グループ、マネージャーフラグ、セキュリティクリアランス

### Resource（リソース）属性

アクセス対象となるデータやオブジェクトの情報です。

- リソース ID、タイプ、作成者
- ステータス（下書き・公開・アーカイブ）、機密レベル
- 所有組織、作成日、公開フラグ

### Action（操作）属性

ユーザーが実行しようとしている操作の種類です。

- 基本操作: `read`、`create`、`update`、`delete`
- 業務操作: `publish`、`approve`、`export`、`archive`

### Environment（環境）属性

アクセス時の状況・コンテキスト情報です。

- 時刻、曜日、IP アドレス
- デバイス種別、ネットワーク種別（社内/VPN/公共）
- MFA 認証ステータス、リスクスコア

### ポリシーの記述例

これら 4 つの属性を組み合わせると、以下のようなポリシーを表現できます。

```
ポリシー 1: リソースオーナーシップ
  IF subject.role == "editor"
  AND resource.type == "article"
  AND resource.author == subject.id
  AND action == "update"
  THEN ALLOW

ポリシー 2: 部署スコープ + 時間制限
  IF subject.department == resource.department
  AND subject.role == "manager"
  AND action == "read"
  AND environment.time BETWEEN "09:00" AND "18:00"
  THEN ALLOW

ポリシー 3: セキュリティクリアランス + ネットワーク制限
  IF subject.clearance >= resource.classification
  AND environment.network == "corporate"
  AND environment.mfa_verified == true
  THEN ALLOW
```

---

## NIST ABAC 参照アーキテクチャ

NIST SP 800-162 では、ABAC のシステムを 4 つのコンポーネントに分離して設計することを推奨しています。

```
NIST ABAC 参照アーキテクチャ:

  PEP (Policy Enforcement Point)
    → アクセス要求をインターセプトする
    → PDP の決定結果を実施する（許可/拒否）
    → アプリのミドルウェアや API ゲートウェイが担当

  PDP (Policy Decision Point)
    → ポリシーを評価してアクセス可否を決定する
    → 属性情報を PIP から取得する
    → ポリシーエンジン（OPA、Cedar、CASL 等）が担当

  PAP (Policy Administration Point)
    → ポリシーの作成・管理・配布を行う
    → 管理者がポリシーを定義するインターフェース

  PIP (Policy Information Point)
    → 属性情報を提供する
    → ユーザー DB、リソース DB、外部サービス等
    → 環境情報（時刻、IP、デバイス情報）の取得
```

このアーキテクチャの要点は「判断するところ（PDP）」と「実施するところ（PEP）」を分離することです。これにより、ポリシーの変更がアプリケーションコードに影響しない設計が実現できます。

---

## RBAC vs ABAC 比較

RBAC と ABAC にはそれぞれ得意分野があります。どちらか一方だけが正解ということはなく、要件に応じて選択・併用するのが実務の定石です。

| 項目 | RBAC | ABAC |
|------|------|------|
| 制御の基盤 | ロール | 属性の組み合わせ |
| 制御の粒度 | ロール単位（粗い） | 属性単位（細かい） |
| 柔軟性 | 低〜中 | 高 |
| 導入の複雑性 | 低 | 中〜高 |
| 管理コスト | 低（ロール少数時） | 中 |
| スケーラビリティ課題 | ロール爆発 | ポリシー複雑化 |
| 適したユースケース | 組織の役割が明確な場合 | リソース所有者制御、条件付きアクセス |

### ロール爆発問題

RBAC が ABAC を必要とする代表的な理由が「ロール爆発」です。

```
ロール爆発の例:

  基本ロール:         admin, editor, viewer → 3 ロール
  部署を掛け合わせ:    3 x 3部署 = 9 ロール
  地域を掛け合わせ:    9 x 3地域 = 27 ロール
  プロジェクトを追加:  27 x N = 指数関数的に増加

  ABAC なら:
    ポリシー: "同じ部署 AND 同じ地域 AND ロールが editor 以上"
    → ロール数は 3 のまま。属性の組み合わせで動的に判定
```

実務では「基本は RBAC、細かい制御に ABAC を追加」というハイブリッドが一般的です。

---

## ポリシーエンジンの選択肢

ABAC を実装するためのポリシーエンジンには、いくつかの代表的な選択肢があります。

### CASL（JavaScript/TypeScript）

JavaScript/TypeScript 向けのポリシーライブラリです。MongoDB 互換のクエリ構文で条件を記述でき、フロントエンドとバックエンドで同じポリシー定義を共有できます。

```typescript
import { AbilityBuilder, createMongoAbility, subject } from '@casl/ability';

function defineAbilityFor(user) {
  const { can, cannot, build } = new AbilityBuilder(createMongoAbility);

  if (user.role === 'admin') {
    can('manage', 'Article');
    can('update', 'User', { orgId: user.orgId }); // 同じ組織のみ
    cannot('delete', 'User');
  }

  if (user.role === 'editor') {
    can('read', 'Article');
    can('create', 'Article');
    can('update', 'Article', { authorId: user.id }); // 自分の記事のみ
    can('delete', 'Article', { authorId: user.id });
  }

  if (user.role === 'viewer') {
    can('read', 'Article', { status: 'published' }); // 公開記事のみ
  }

  return build();
}

// 使用例
const ability = defineAbilityFor({ id: 'user_1', role: 'editor', orgId: 'org_1' });
ability.can('update', subject('Article', { authorId: 'user_1' })); // true
ability.can('update', subject('Article', { authorId: 'user_2' })); // false
```

### OPA / Rego（マイクロサービス向け）

CNCF 卒業プロジェクトの汎用ポリシーエンジンです。独自言語 Rego でポリシーを記述し、サイドカーや REST API としてデプロイします。

```rego
package authz
default allow = false

allow {
  input.action == "read"
  input.resource.status == "published"
}

allow {
  input.action == "update"
  input.resource.author_id == input.subject.id
}
```

### Cedar（AWS 環境向け）

AWS が開発した型安全なポリシー言語です。形式検証が可能で、Amazon Verified Permissions と連携できます。

```
permit(
  principal in Group::"editors",
  action == Action::"update",
  resource
) when { resource.author == principal && resource.status != "archived" };
```

### ポリシーエンジン比較表

| 項目 | CASL | OPA/Rego | Cedar | Casbin |
|------|------|----------|-------|--------|
| 言語 | JavaScript/TS | Rego（独自） | Cedar（独自） | 設定ファイル |
| 実行環境 | ブラウザ/Node.js | Go バイナリ | Rust バイナリ | 多言語 SDK |
| 学習コスト | 低 | 中〜高 | 中 | 低〜中 |
| フロントエンド対応 | あり | なし | なし | なし |
| DB 統合 | Prisma 等 | なし | なし | 限定的 |
| 形式検証 | なし | 限定的 | あり | なし |
| 適したユースケース | Web アプリ | K8s/マイクロサービス | AWS 連携 | 多言語環境 |

---

## 実装パターン

### パターン 1: リソースオーナーシップ

最も一般的なパターンです。作成者が自分のリソースを操作できるようにします。

```typescript
can('update', 'Article', { authorId: user.id });
can('delete', 'Article', { authorId: user.id });
```

### パターン 2: 組織スコープ

マルチテナント SaaS の基本パターンです。同じ組織のリソースのみアクセスを許可します。

```typescript
can('read', 'Article', { orgId: user.orgId });
can('update', 'User', { orgId: user.orgId });
```

### パターン 3: ステータスベース

リソースの状態に応じて操作を制限します。

```typescript
can('update', 'Article', { status: { $ne: 'archived' } });
cannot('delete', 'Article', { status: 'published' });
```

### パターン 4: 環境属性による制限

時間帯やネットワークなど環境条件を加味します。

```typescript
function defineAbilityWithEnv(user, env) {
  const { can, cannot, build } = new AbilityBuilder(createMongoAbility);

  can('read', 'Article', { orgId: user.orgId });

  // 営業時間内のみエクスポート可能
  const hour = env.time.getHours();
  if (hour >= 9 && hour < 18) {
    can('export', 'Article', { orgId: user.orgId });
  }

  // 社内ネットワークからのみ機密データにアクセス可能
  if (env.networkType === 'corporate') {
    can('read', 'Article', { classification: 'confidential' });
  }

  // MFA 認証済みの場合のみ管理操作を許可
  if (env.isMfaVerified) {
    can('delete', 'Article', { authorId: user.id });
  }

  return build();
}
```

### パターン 5: RBAC + ABAC ハイブリッド

実務で最も推奨されるパターンです。3 層に分けてアクセス制御を行います。

```
リクエスト処理の流れ:

  Request
    → 第 1 層: RBAC チェック（ミドルウェア）    「この API にアクセスできるか？」
    → 第 2 層: ABAC チェック（サービス層）       「この特定のリソースを操作できるか？」
    → 第 3 層: ビジネスルール（ドメイン層）       「この操作はビジネス上許可されているか？」
  Response
```

```typescript
// 第 1 層: RBAC ミドルウェア
function requireRole(...roles) {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient role' });
    }
    next();
  };
}

// 第 2 層 + 第 3 層: サービス層
class ArticleService {
  async update(user, articleId, data) {
    const ability = defineAbilityFor(user);
    const article = await prisma.article.findUnique({ where: { id: articleId } });
    if (!article) throw new NotFoundError('Article not found');

    // ABAC チェック
    ForbiddenError.from(ability).throwUnlessCan(
      'update', subject('Article', article)
    );

    // ビジネスルールチェック
    if (article.status === 'published' && !user.permissions?.includes('edit_published')) {
      throw new Error('公開済み記事の編集には特別な権限が必要です');
    }

    return prisma.article.update({ where: { id: articleId }, data });
  }
}

// API ルート
app.put('/api/articles/:id',
  requireRole('editor', 'admin'),
  async (req, res) => {
    const result = await articleService.update(req.user, req.params.id, req.body);
    res.json(result);
  }
);
```

---

## よくあるアンチパターン

ABAC の導入で陥りやすい落とし穴を押さえておきましょう。

| アンチパターン | 問題点 | 対策 |
|--------------|--------|------|
| フロントのみの権限制御 | DevTools で回避可能 | バックエンドでも必ず検証する |
| ハードコードされた `if (role === 'admin')` | ロール追加のたびにコード修正が必要 | ポリシーを一箇所で管理する |
| 過度に複雑なポリシー | デバッグが困難、パフォーマンス低下 | ポリシーを階層化・モジュール化する |
| ポリシーのテスト不足 | 変更時に意図しない権限漏れが発生 | マトリクステストで全組み合わせを検証する |
| 監査ログの欠如 | インシデント時の調査が不可能 | アクセス決定を必ずログに記録する |

---

## まとめ

| 項目 | ポイント |
|------|---------|
| ABAC の本質 | 属性（主体・リソース・操作・環境）の組み合わせでアクセスを制御します |
| RBAC との違い | RBAC はロール単位、ABAC は属性単位。ABAC のほうが粒度が細かいです |
| ロール爆発 | RBAC だけでは条件の掛け合わせでロール数が指数的に増加します |
| NIST アーキテクチャ | PEP（実施）・PDP（判断）・PAP（管理）・PIP（情報提供）の 4 層です |
| ポリシーエンジン | Web アプリには CASL、マイクロサービスには OPA、AWS には Cedar が適しています |
| 実装の定石 | RBAC + ABAC のハイブリッド（第 1 層 RBAC、第 2 層 ABAC、第 3 層ビジネスルール） |
| フロント権限制御 | UX 改善のため。セキュリティはバックエンドで担保します |
| テスト | ロール x アクション x リソースのマトリクステストで網羅的に検証します |
| 監査ログ | 誰が・いつ・何に・どうアクセスしたかを記録し、コンプライアンスに対応します |

---

## やってみよう！

1. **CASL でポリシーを定義してみよう** --- `@casl/ability` をインストールし、`viewer`（公開記事のみ閲覧可）、`editor`（自分の記事のみ編集可）、`admin`（同じ組織の全リソース管理可）の 3 ロールを定義してください。`ability.can()` で期待通りの結果が返ることを確認しましょう。

2. **ロール爆発を体験してみよう** --- 「3 ロール x 3 部署 x 3 地域」の RBAC を実際にコードで書き出してみてください。次に同じ要件を ABAC（属性の組み合わせ）で書き直し、コード量と管理のしやすさを比較してみましょう。

3. **ハイブリッド認可を組んでみよう** --- Express の API エンドポイントに、第 1 層（RBAC ミドルウェア）と第 2 層（CASL による ABAC チェック）を組み合わせた認可処理を実装してください。「editor ロールだが自分の記事でない場合は 403 を返す」ことをテストで確認しましょう。
