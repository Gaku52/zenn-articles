---
title: "RBAC（ロールベースアクセス制御）"
---

## RBAC とは何か

RBAC（Role-Based Access Control）は、ユーザーに直接権限を割り当てるのではなく、**ロール（役割）** を介して権限を管理するアクセス制御モデルです。現在最も広く採用されている認可パターンであり、AWS IAM、GitHub Organizations、Google Workspace など主要サービスの基盤となっています。

RBAC の核心は「**人ではなく役割に権限を紐付ける**」という考え方です。新しいメンバーが加わったとき、個別に権限を設定するのではなく「editor ロールを割り当てる」だけで適切な権限が付与されます。

## ロール・パーミッション・ユーザーの関係

RBAC は 3 つの構成要素で成り立っています。

```
  ┌──────────┐    ┌──────────┐    ┌──────────────┐
  │  User    │───→│  Role    │───→│  Permission  │
  │ ユーザー  │ N:M│  ロール   │ N:M│  パーミッション │
  └──────────┘    └──────────┘    └──────────────┘
```

- **User（ユーザー）**: システムを利用する人やサービスアカウントです
- **Role（ロール）**: 「編集者」「管理者」のような役割の単位です
- **Permission（パーミッション）**: 「記事を作成できる」「ユーザーを削除できる」のような個別の操作権限です

ユーザーは 1 つ以上のロールを持ち、ロールは 1 つ以上のパーミッションを持ちます。ユーザーはロール経由で間接的に権限を取得します。

### なぜ User に直接 Permission を紐付けないのか

ユーザー数が増えると、個別の権限管理は破綻します。RBAC では「editor」というロールを定義しておけば、新しいユーザーにはロールを割り当てるだけで済みます。権限を変更したい場合もロールの定義を変えれば、そのロールを持つ全ユーザーに反映されます。

### パーミッションの命名規則

パーミッションは `resource:action` 形式で統一するのがベストプラクティスです。

```
articles:read     — 記事の閲覧
articles:create   — 記事の作成
articles:update   — 記事の編集
articles:delete   — 記事の削除
articles:publish  — 記事の公開
users:read        — ユーザー一覧の閲覧
users:invite      — ユーザーの招待
org:settings      — 組織設定の変更
```

リソース名は複数形（articles, users）、アクションは CRUD ベースにドメイン固有の操作（publish, approve）を加えます。`can_edit_articles` のような冗長な命名や、数値 ID での管理は避けてください。

## RBAC のレベル（NIST 定義）

NIST（米国国立標準技術研究所）は RBAC を 4 つのレベルに分類しています。

| レベル | 名称 | 特徴 |
|--------|------|------|
| RBAC0 | 基本 RBAC | ユーザー、ロール、パーミッションの基本構造 |
| RBAC1 | 階層ロール | ロール間の継承関係を追加 |
| RBAC2 | 制約付き RBAC | 相互排他や最大ロール数などの制約を追加 |
| RBAC3 | 統合 RBAC | RBAC1 + RBAC2 の両方を統合 |

小規模なアプリケーションでは RBAC0 で十分です。中規模では RBAC1（階層ロール）が推奨されます。エンタープライズ向けでは RBAC3 を採用するケースが多いです。

## ロール階層（RBAC1）

ロール階層を導入すると、上位ロールが下位ロールの権限を自動的に継承します。これにより権限の重複定義を排除できます。

```
super_admin
  └── admin
        └── publisher
              └── editor
                    └── viewer
```

この例では `admin` ロールは `publisher` の全権限を継承し、さらに `publisher` は `editor` の権限を、`editor` は `viewer` の権限を継承します。

### TypeScript でのロール定義

```typescript
// パーミッション定義
const PERMISSIONS = {
  'articles:read': '記事の閲覧',
  'articles:create': '記事の作成',
  'articles:update': '記事の編集',
  'articles:delete': '記事の削除',
  'articles:publish': '記事の公開',
  'users:read': 'ユーザー一覧',
  'users:create': 'ユーザー作成',
  'users:update': 'ユーザー編集',
  'users:delete': 'ユーザー削除',
  'org:settings': '組織設定',
} as const;

type Permission = keyof typeof PERMISSIONS;

// ロール定義（階層付き）
interface RoleConfig {
  description: string;
  permissions: Permission[];
  inherits: string[];
}

const ROLES: Record<string, RoleConfig> = {
  viewer: {
    description: '閲覧のみ',
    permissions: ['articles:read'],
    inherits: [],
  },
  editor: {
    description: '記事の作成・編集',
    permissions: ['articles:create', 'articles:update'],
    inherits: ['viewer'],
  },
  publisher: {
    description: '記事の公開・削除',
    permissions: ['articles:publish', 'articles:delete'],
    inherits: ['editor'],
  },
  admin: {
    description: 'ユーザー・組織管理',
    permissions: ['users:read', 'users:create', 'users:update',
                  'users:delete', 'org:settings'],
    inherits: ['publisher'],
  },
};
```

### 権限の再帰的な解決

ロール階層を正しく処理するには、再帰的に親ロールの権限を収集する必要があります。循環参照を防止するために `visited` セットを使用します。

```typescript
function resolvePermissions(
  role: string,
  visited = new Set<string>()
): Set<Permission> {
  if (visited.has(role)) return new Set(); // 循環参照防止
  visited.add(role);

  const config = ROLES[role];
  if (!config) return new Set();

  const permissions = new Set<Permission>(config.permissions);

  // 親ロールの権限を再帰的に収集
  for (const parentRole of config.inherits) {
    for (const perm of resolvePermissions(parentRole, visited)) {
      permissions.add(perm);
    }
  }

  return permissions;
}

function hasPermission(userRole: string, permission: Permission): boolean {
  return resolvePermissions(userRole).has(permission);
}

// 動作確認
hasPermission('editor', 'articles:read');    // true（viewer から継承）
hasPermission('editor', 'articles:create');  // true（直接の権限）
hasPermission('editor', 'articles:publish'); // false（publisher 以上）
```

## DB スキーマ設計

RBAC のデータベース設計には、正規化と非正規化の 2 つのアプローチがあります。

### 非正規化アプローチ（小〜中規模向け）

ユーザーテーブルに `role` カラムを持たせ、権限定義はコードで管理します。クエリがシンプルでパフォーマンスも良好です。ただし権限変更にはコード修正とデプロイが必要です。

```sql
CREATE TABLE users (
  id         VARCHAR(36) PRIMARY KEY,
  email      VARCHAR(255) UNIQUE NOT NULL,
  name       VARCHAR(100) NOT NULL,
  role       VARCHAR(50) NOT NULL DEFAULT 'viewer',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 正規化アプローチ（大規模向け）

5 つのテーブルで構成し、管理画面からの動的な権限変更に対応します。

```sql
-- ロールテーブル（階層対応）
CREATE TABLE roles (
  id          VARCHAR(36) PRIMARY KEY,
  name        VARCHAR(50) UNIQUE NOT NULL,
  description TEXT,
  parent_id   VARCHAR(36) REFERENCES roles(id),
  created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- パーミッションテーブル
CREATE TABLE permissions (
  id          VARCHAR(36) PRIMARY KEY,
  name        VARCHAR(100) UNIQUE NOT NULL,
  description TEXT,
  created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ユーザーとロールの中間テーブル
CREATE TABLE user_roles (
  user_id     VARCHAR(36) REFERENCES users(id) ON DELETE CASCADE,
  role_id     VARCHAR(36) REFERENCES roles(id) ON DELETE CASCADE,
  assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  assigned_by VARCHAR(36),
  PRIMARY KEY (user_id, role_id)
);

-- ロールとパーミッションの中間テーブル
CREATE TABLE role_permissions (
  role_id       VARCHAR(36) REFERENCES roles(id) ON DELETE CASCADE,
  permission_id VARCHAR(36) REFERENCES permissions(id) ON DELETE CASCADE,
  PRIMARY KEY (role_id, permission_id)
);
```

この設計では `user_roles` と `role_permissions` の 2 つの中間テーブルにより、ユーザーとパーミッションが多対多で柔軟に紐付きます。`roles` テーブルの `parent_id` により階層構造も表現できます。

### DB からユーザーの全権限を取得

```typescript
async function getUserPermissions(userId: string): Promise<Set<string>> {
  // Prisma で階層込みの権限を取得
  const user = await prisma.user.findUnique({
    where: { id: userId },
    include: {
      roles: {
        include: {
          role: {
            include: {
              permissions: { include: { permission: true } },
              parent: {
                include: {
                  permissions: { include: { permission: true } },
                },
              },
            },
          },
        },
      },
    },
  });

  const permissions = new Set<string>();

  function collectPermissions(role: any) {
    role.permissions.forEach(({ permission }: any) => {
      permissions.add(permission.name);
    });
    if (role.parent) collectPermissions(role.parent);
  }

  user?.roles.forEach(({ role }) => collectPermissions(role));
  return permissions;
}
```

## ミドルウェア実装

### Express ミドルウェア

権限チェックをミドルウェアとして実装すると、ルート定義時に宣言的に権限を指定できます。

```typescript
import { Request, Response, NextFunction } from 'express';

// 全ての必須権限を持っているかチェック
function requirePermission(...required: string[]) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const user = req.user;
    if (!user) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const userPermissions = await getUserPermissions(user.id);
    const hasAll = required.every((p) => userPermissions.has(p));

    if (!hasAll) {
      return res.status(403).json({
        error: 'Insufficient permissions',
        required,
      });
    }

    next();
  };
}

// ルート定義での使用例
app.get('/api/articles',
  requirePermission('articles:read'),
  getArticles
);

app.post('/api/articles',
  requirePermission('articles:create'),
  createArticle
);

app.delete('/api/articles/:id',
  requirePermission('articles:delete'),
  deleteArticle
);
```

### Next.js Server Actions での権限チェック

```typescript
import { auth } from '@/auth';
import { redirect } from 'next/navigation';

async function authorize(...required: string[]) {
  const session = await auth();
  if (!session) redirect('/login');

  const permissions = resolvePermissions(session.user.role);
  const hasAll = required.every((p) => permissions.has(p as Permission));

  if (!hasAll) {
    throw new Error(`権限が不足しています: ${required.join(', ')}`);
  }
  return session;
}

// Server Action での使用
'use server';

async function publishArticle(articleId: string) {
  const session = await authorize('articles:publish');

  await prisma.article.update({
    where: { id: articleId },
    data: { status: 'published', publishedBy: session.user.id },
  });

  revalidatePath('/articles');
}
```

## よくあるアンチパターン

RBAC の実装で陥りがちな失敗パターンを紹介します。

### 1. ロール名のハードコーディング

```typescript
// NG: ロール名で分岐
if (user.role === 'admin' || user.role === 'super_admin') {
  // 削除を許可
}

// OK: パーミッションで分岐
if (userPermissions.has('articles:delete')) {
  // 削除を許可
}
```

ロール名をコードに埋め込むと、ロールの追加や再編成のたびにコード修正が必要になります。パーミッション名でチェックすれば、ロール構成が変わってもコードの修正は不要です。

### 2. フロントエンドだけの権限チェック

フロントエンドでボタンを非表示にするだけでは、API を直接叩かれると突破されます。必ずバックエンド API 側でもミドルウェアによる権限チェックを行ってください。

### 3. デフォルト許可の設計

「拒否リストに載っていなければ許可」という設計は、新しいリソースを追加した際に権限漏れが発生します。**デフォルト拒否**（許可リストに載っているもののみアクセス可能）を原則としてください。

## まとめ

| 項目 | ポイント |
|------|---------|
| 基本構造 | User → Role → Permission の間接参照 |
| パーミッション命名 | `resource:action` 形式で統一する |
| ロール階層 | 継承により権限の重複定義を排除（RBAC1） |
| DB 設計（小規模） | users テーブルに role カラムを追加 |
| DB 設計（大規模） | 5 テーブル正規化 + 中間テーブル |
| ミドルウェア | `requirePermission()` で宣言的にチェック |
| デフォルト方針 | 拒否がデフォルト（最小権限の原則） |
| アンチパターン | ロール名ハードコード、フロントのみチェック |

## やってみよう！

- [ ] 自分のアプリケーションに必要なロールを 3〜5 個洗い出し、それぞれに紐付くパーミッションを `resource:action` 形式でリストアップしてみましょう
- [ ] `resolvePermissions` 関数を実装し、ロール階層が正しく解決されることをテストコードで確認してみましょう
- [ ] Express または Next.js で `requirePermission` ミドルウェアを実装し、権限のないユーザーが 403 を受け取ることを確認してみましょう
- [ ] 正規化版の DB スキーマを実際に作成し、ユーザーの全権限を取得する SQL クエリを書いてみましょう
