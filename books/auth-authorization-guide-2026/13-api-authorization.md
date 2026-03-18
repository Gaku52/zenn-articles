---
title: "API認可とフロントエンド認可"
---

## この章で学ぶこと

この章では、APIレベルの認可とフロントエンドでの権限制御を扱います。

- APIキーの仕組みと安全な管理方法
- Bearer Tokenを使ったAPI認証
- ミドルウェアパターンによる認可の適用
- フロントエンドでの権限チェックとUI表示制御

## 1. APIキーによる認証

APIキーは、APIを利用するクライアントを識別するための文字列です。主にサーバー間通信やサードパーティ連携で使用されます。

通常、**プレフィックス**を付けて用途を明確にします。

```
sk_live_aBcDeFgHiJk...   ← 本番用 Secret Key
sk_test_aBcDeFgHiJk...   ← テスト用 Secret Key
pk_live_aBcDeFgHiJk...   ← 本番用 Public Key（クライアント向け）
```

### APIキーの生成と保存

安全に管理するためには、**平文をDBに保存しない**ことが鉄則です。SHA-256でハッシュ化した値を保存し、検証時にはリクエストのキーをハッシュ化して比較します。

```typescript
import crypto from "crypto";

function generateApiKey(prefix: string) {
  const randomPart = crypto.randomBytes(24).toString("base64url");
  const key = `${prefix}${randomPart}`;
  const hash = crypto.createHash("sha256").update(key).digest("hex");
  const lastFour = randomPart.slice(-4);
  return { key, hash, lastFour };
}

// key: ユーザーに1回だけ表示する
// hash: DBに保存する
// lastFour: 一覧画面で「sk_live_****abcd」のように表示する
```

ポイントは3つです。

1. **生成時にのみ平文を表示する** - 以降は取得不可にします
2. **ハッシュ値で保存する** - 漏洩してもキーが復元できません
3. **プレフィックスと末尾4文字は平文で保存する** - 管理画面での識別用です

ローテーション時は古いキーを即時無効化せず、**24時間のグレースピリオド**（猶予期間）を設けるのがベストプラクティスです。

## 2. Bearer Token認証

Bearer Token は OAuth 2.0 で標準化されたトークン送信方式です。`Authorization` ヘッダーにトークンを付与します。

```
GET /api/articles HTTP/1.1
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

「Bearer」は「持参人」を意味します。トークンを**持っている人が権限を持つ**という考え方のため、トークン漏洩には注意が必要です。

```typescript
import jwt from "jsonwebtoken";

async function verifyBearerToken(authHeader: string | undefined) {
  if (!authHeader?.startsWith("Bearer ")) {
    throw new Error("Authorization ヘッダーが不正です");
  }
  const token = authHeader.slice(7);
  try {
    return jwt.verify(token, process.env.JWT_SECRET!) as {
      userId: string; role: string; scopes: string[];
    };
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      throw new Error("トークンの有効期限が切れています");
    }
    throw new Error("不正なトークンです");
  }
}
```

### APIキーとBearer Tokenの使い分け

| 項目 | APIキー | Bearer Token (JWT) |
|------|---------|-------------------|
| 主な用途 | サーバー間通信、外部連携 | ユーザー代理のアクセス |
| 有効期間 | 長期（手動ローテーション） | 短期（自動更新） |
| 取り消し | DB上で即時無効化 | 短い有効期限で自然失効 |
| 適したケース | CI/CD、バッチ処理 | SPA、モバイルアプリ |

## 3. ミドルウェアパターンによるAPI認可

### 認証ミドルウェア

Express のミドルウェアパターンで認証をチェックします。

```typescript
async function authMiddleware(req: Request, res: Response, next: Function) {
  const apiKey = req.headers["x-api-key"] as string;
  const authHeader = req.headers.authorization;

  if (apiKey) {
    const hash = crypto.createHash("sha256").update(apiKey).digest("hex");
    const keyData = await db.apiKey.findUnique({ where: { keyHash: hash } });
    if (!keyData || keyData.revokedAt) {
      return res.status(401).json({ error: "無効なAPIキーです" });
    }
    req.auth = { userId: keyData.userId, scopes: keyData.scopes };
  } else if (authHeader?.startsWith("Bearer ")) {
    req.auth = await verifyBearerToken(authHeader);
  } else {
    return res.status(401).json({ error: "認証情報が必要です" });
  }
  next();
}
```

### スコープ検証ミドルウェア

認証の次にスコープ（権限の範囲）を検証します。

```typescript
function requireScope(...requiredScopes: string[]) {
  return (req: Request, res: Response, next: Function) => {
    const userScopes = new Set(req.auth?.scopes || []);
    if (userScopes.has("admin")) return next();

    const hasRequired = requiredScopes.every((s) => userScopes.has(s));
    if (!hasRequired) {
      return res.status(403).json({
        error: "insufficient_scope",
        required: requiredScopes,
        granted: Array.from(userScopes),
      });
    }
    next();
  };
}
```

### ミドルウェアの組み合わせ

ルート定義で複数のミドルウェアを段階的に適用します。

```typescript
app.get("/api/articles",
  authMiddleware, requireScope("articles:read"), listArticles);

app.post("/api/articles",
  authMiddleware, requireScope("articles:write"), createArticle);

app.delete("/api/articles/:id",
  authMiddleware, requireScope("articles:delete"),
  requireOwnership("article"), deleteArticle);
```

`requireOwnership` はリソースの所有者かどうかを確認するミドルウェアです。外部向けAPIでは、権限不足の場合にリソースの存在を隠すため `404` を返すのが一般的です。

### HTTPステータスコードの使い分け

| コード | 意味 | 使いどころ |
|--------|------|-----------|
| `401` | 認証エラー（未認証） | トークン未提供、トークン無効 |
| `403` | 認可エラー（権限不足） | スコープ不足、ロール不足 |
| `404` | セキュリティ目的の403代替 | リソースの存在を隠したい場合 |
| `429` | レート制限超過 | Retry-Afterヘッダーを付与 |

## 4. フロントエンドでの権限チェック

### 大前提: フロントエンドの認可はUXのため

フロントエンドの認可は**表示制御によるUX向上**が目的です。セキュリティの最終防衛線ではありません。DevToolsでボタンの`disabled`を外したり、`curl`で直接APIを叩けばバイパスできます。**必ずバックエンド側でも認可チェックを行ってください**。

### AuthContextで権限を管理する

React の Context API で権限情報をアプリ全体に共有します。

```typescript
"use client";
import { createContext, useContext, useCallback, type ReactNode } from "react";
import { useSession } from "next-auth/react";
import { useQuery } from "@tanstack/react-query";

interface AuthContextValue {
  user: { id: string; role: string } | null;
  can: (permission: string) => boolean;
  hasRole: (role: string) => boolean;
}

const AuthContext = createContext<AuthContextValue>({
  user: null, can: () => false, hasRole: () => false,
});

export function AuthProvider({ children }: { children: ReactNode }) {
  const { data: session } = useSession();
  const { data: permissions = new Set<string>() } = useQuery({
    queryKey: ["permissions", session?.user?.id],
    queryFn: async () => {
      const res = await fetch("/api/auth/permissions");
      const data = await res.json();
      return new Set<string>(data.permissions);
    },
    enabled: !!session,
    staleTime: 5 * 60 * 1000,
  });

  const can = useCallback(
    (permission: string) => {
      if (session?.user?.role === "admin") return true;
      return permissions.has(permission);
    },
    [session?.user?.role, permissions]
  );
  const hasRole = useCallback(
    (role: string) => session?.user?.role === role,
    [session?.user?.role]
  );

  return (
    <AuthContext.Provider value={{ user: session?.user ?? null, can, hasRole }}>
      {children}
    </AuthContext.Provider>
  );
}

export const useAuth = () => useContext(AuthContext);
```

### Authorizedコンポーネントで宣言的にUI制御

```tsx
"use client";
import { useAuth } from "@/lib/auth-context";
import type { ReactNode } from "react";

export function Authorized({ permission, role, fallback = null, children }: {
  permission?: string; role?: string;
  fallback?: ReactNode; children: ReactNode;
}) {
  const { can, hasRole } = useAuth();
  if (permission && !can(permission)) return <>{fallback}</>;
  if (role && !hasRole(role)) return <>{fallback}</>;
  return <>{children}</>;
}
```

使用例です。

```tsx
function ArticleCard({ article }) {
  return (
    <div>
      <h2>{article.title}</h2>
      <Authorized permission="articles:update">
        <button>編集</button>
      </Authorized>
      <Authorized permission="articles:delete"
        fallback={<button disabled>削除</button>}>
        <button onClick={() => deleteArticle(article.id)}>削除</button>
      </Authorized>
      <Authorized role="admin">
        <button>監査ログ</button>
      </Authorized>
    </div>
  );
}
```

### ナビゲーションの権限フィルタリング

サイドバーの項目を権限で再帰的にフィルタリングします。

```typescript
const navItems = [
  { label: "ダッシュボード", href: "/dashboard" },
  { label: "記事作成", href: "/articles/new", permission: "articles:create" },
  { label: "ユーザー管理", href: "/admin/users", role: "admin" },
];

function filterItems(items, can, hasRole) {
  return items
    .filter((item) => {
      if (item.permission && !can(item.permission)) return false;
      if (item.role && !hasRole(item.role)) return false;
      return true;
    })
    .map((item) => ({
      ...item,
      children: item.children ? filterItems(item.children, can, hasRole) : undefined,
    }))
    .filter((item) => !item.children || item.children.length > 0);
}
```

### Server Componentsでの権限判定（推奨）

Next.js の Server Components でサーバー側で権限を判定し、フラグを Client Component に渡す方式が最も安全です。

```tsx
// Server Component
export default async function ArticlePage({ params }) {
  const session = await auth();
  if (!session) redirect("/login");
  const article = await prisma.article.findUnique({ where: { id: params.id } });
  if (!article) notFound();

  const permissions = {
    canEdit: session.user.role === "admin" || article.authorId === session.user.id,
    canDelete: session.user.role === "admin",
  };

  return <ArticleActions articleId={article.id} {...permissions} />;
}

// Client Component
"use client";
function ArticleActions({ articleId, canEdit, canDelete }) {
  return (
    <div>
      {canEdit && <button>編集</button>}
      {canDelete && <button>削除</button>}
    </div>
  );
}
```

この方式の利点は、権限ロジックがサーバー側にあるため改ざん不可能であること、権限のないデータをクライアントに送信しないことです。

## まとめ

| 項目 | ポイント |
|------|---------|
| APIキー | SHA-256でハッシュ保存。プレフィックス付き。生成時のみ平文表示 |
| Bearer Token | Authorizationヘッダーで送信。JWTは有効期限と署名を検証 |
| ミドルウェア | 認証→スコープ検証→所有者チェックの順で段階的に適用 |
| HTTPステータス | 401（未認証）/403（権限不足）/404（存在隠蔽）を使い分ける |
| フロント認可 | UX最適化が目的。セキュリティはバックエンドで保証する |
| UI表示制御 | Authorizedコンポーネントで宣言的に制御する |
| Server Components | サーバーで権限判定してフラグをpropsで渡すのが最も安全 |

## やってみよう！

1. **APIキー管理を実装する** - `generateApiKey`関数を作成し、SHA-256でのハッシュ保存と検証の流れを確認してください。`sk_live_`プレフィックスで生成してみましょう。

2. **ミドルウェアチェーンを組む** - Expressで`authMiddleware` → `requireScope("articles:read")`の順にミドルウェアを適用し、不正なリクエストで`401`/`403`が返ることを確認してください。

3. **Authorizedコンポーネントを作る** - `<Authorized permission="articles:delete">`を実装し、権限に応じてボタンの表示・非表示が切り替わることを確認してください。`fallback`によるdisabled表示も試してみましょう。

4. **Server Componentで権限フラグを渡す** - Server Componentでロールと所有者を比較し、`canEdit`/`canDelete`フラグをClient Componentにpropsで渡すパターンを実装してみてください。
