---
title: "第10章 セキュリティレビュー"
---

# 第10章 セキュリティレビュー

本章ではセキュリティレビューについて、実践的な観点から詳しく解説します。セキュリティの脆弱性は、システムとユーザーに深刻な被害をもたらす可能性があるため、コードレビューの段階で適切にチェックすることが極めて重要です。

## 概要

セキュリティレビューは、コードに潜む脆弱性を発見し、セキュアなシステムを構築するための重要なプロセスです。本章では、OWASP Top 10の観点から、SQLインジェクション、XSS、CSRF、認証・認可、機密情報の管理など、具体的なチェックポイントとコード例を通じて、効果的なレビュー方法を学びます。

## OWASP Top 10の観点

OWASP（Open Web Application Security Project）が公開しているTop 10は、Webアプリケーションにおける最も重大なセキュリティリスクをまとめたものです。コードレビューでは、これらのリスクを念頭に置いてチェックすることが推奨されます。

主要な項目:
- インジェクション（SQLインジェクション、コマンドインジェクション等）
- 認証の不備
- 機密データの露出
- XML外部エンティティ（XXE）
- アクセス制御の不備
- セキュリティ設定のミス
- クロスサイトスクリプティング（XSS）
- 安全でないデシリアライゼーション
- 既知の脆弱性を持つコンポーネントの使用
- 不十分なログ記録と監視

## SQLインジェクション対策

### プレースホルダーとパラメータ化クエリの使用

SQLインジェクションは、ユーザー入力を適切にエスケープせずにSQL文に組み込むことで発生します。必ずプレースホルダーまたはパラメータ化クエリを使用しましょう。

```python
# 悪い例: SQLインジェクションの脆弱性あり
def get_user_by_id(user_id):
    query = f"SELECT * FROM users WHERE id = {user_id}"
    return db.execute(query)

# ユーザーが "1 OR 1=1" を入力すると、全ユーザーのデータが取得される

# 良い例: パラメータ化クエリを使用
def get_user_by_id_safe(user_id):
    query = "SELECT * FROM users WHERE id = ?"
    return db.execute(query, (user_id,))
```

```typescript
// 悪い例: 文字列結合でクエリを構築
async function getUserByEmail(email: string) {
  const query = `SELECT * FROM users WHERE email = '${email}'`;
  return await db.query(query);
}

// 良い例: ORMまたはパラメータ化クエリを使用
async function getUserByEmailSafe(email: string) {
  return await db.user.findUnique({
    where: { email }
  });
}
```

レビュー時には、生のSQL文字列を検索し、パラメータ化クエリが使用されているか確認しましょう。

### ストアドプロシージャの活用

```sql
-- ストアドプロシージャの例
CREATE PROCEDURE GetUserById
    @UserId INT
AS
BEGIN
    SELECT * FROM Users WHERE Id = @UserId;
END
```

ストアドプロシージャを使用することで、SQLインジェクションのリスクをさらに低減できます。

## クロスサイトスクリプティング（XSS）対策

### 入力値のサニタイズとエスケープ

XSSは、悪意のあるスクリプトがWebページに挿入され、他のユーザーのブラウザで実行される脆弱性です。ユーザー入力を表示する際は、必ず適切にエスケープしましょう。

```typescript
// 悪い例: ユーザー入力をそのままHTMLに挿入
function displayUserComment(comment: string) {
  document.getElementById('comment').innerHTML = comment;
}
// ユーザーが "<script>alert('XSS')</script>" を入力すると実行される

// 良い例: textContentを使用してエスケープ
function displayUserCommentSafe(comment: string) {
  document.getElementById('comment').textContent = comment;
}

// Reactの場合（JSXは自動的にエスケープされる）
function CommentComponent({ comment }: { comment: string }) {
  return <div>{comment}</div>;
}

// HTMLを含める必要がある場合はDOMPurifyでサニタイズ
import DOMPurify from 'dompurify';

function CommentWithHTML({ htmlContent }: { htmlContent: string }) {
  const sanitized = DOMPurify.sanitize(htmlContent);
  return <div dangerouslySetInnerHTML={{ __html: sanitized }} />;
}
```

レビュー時には、`innerHTML`、`dangerouslySetInnerHTML`、`eval`などの危険な関数の使用を確認し、適切なサニタイズが行われているかチェックしましょう。

### Content Security Policy（CSP）の設定

```typescript
// Express.jsでのCSP設定例
import helmet from 'helmet';

app.use(
  helmet.contentSecurityPolicy({
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
    },
  })
);
```

CSPを適切に設定することで、XSS攻撃の影響を軽減できます。

## クロスサイトリクエストフォージェリ（CSRF）対策

### CSRFトークンの実装

CSRFは、ユーザーの意図しないリクエストを攻撃者が強制的に実行させる攻撃です。CSRFトークンを使用して対策しましょう。

```typescript
// Express.jsでのCSRF対策例
import csrf from 'csurf';

const csrfProtection = csrf({ cookie: true });

app.get('/form', csrfProtection, (req, res) => {
  res.render('form', { csrfToken: req.csrfToken() });
});

app.post('/process', csrfProtection, (req, res) => {
  // CSRFトークンが検証される
  res.send('Data is being processed');
});
```

```typescript
// フロントエンドでのトークン送信
async function submitForm(data: FormData) {
  const csrfToken = document.querySelector<HTMLInputElement>(
    'input[name="_csrf"]'
  )?.value;

  const response = await fetch('/api/process', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'CSRF-Token': csrfToken || '',
    },
    body: JSON.stringify(data),
  });

  return response.json();
}
```

レビュー時には、状態を変更するエンドポイント（POST、PUT、DELETE）にCSRF対策が実装されているか確認しましょう。

### SameSite Cookie属性の設定

```typescript
// Cookieの設定例
app.use(session({
  secret: process.env.SESSION_SECRET,
  cookie: {
    httpOnly: true,
    secure: true, // HTTPS通信時のみ
    sameSite: 'strict', // CSRF対策
    maxAge: 24 * 60 * 60 * 1000 // 24時間
  }
}));
```

SameSite属性を設定することで、クロスサイトでのCookie送信を制限できます。

## 認証・認可の実装確認

### パスワードの適切なハッシュ化

```typescript
// 悪い例: プレーンテキストまたは弱いハッシュ関数
import crypto from 'crypto';

function hashPasswordWeak(password: string): string {
  return crypto.createHash('md5').update(password).digest('hex');
}

// 良い例: bcryptまたはargon2を使用
import bcrypt from 'bcrypt';

async function hashPasswordStrong(password: string): Promise<string> {
  const saltRounds = 10;
  return await bcrypt.hash(password, saltRounds);
}

async function verifyPassword(
  password: string,
  hash: string
): Promise<boolean> {
  return await bcrypt.compare(password, hash);
}
```

パスワードは必ず適切なハッシュ関数（bcrypt、argon2、scrypt等）を使用してハッシュ化しましょう。MD5やSHA-1は脆弱なため使用を避けます。

### JWTの適切な実装

```typescript
// 悪い例: 秘密鍵がハードコード
const token = jwt.sign({ userId: user.id }, 'secret123', { expiresIn: '1h' });

// 良い例: 環境変数から秘密鍵を取得
const token = jwt.sign(
  { userId: user.id },
  process.env.JWT_SECRET as string,
  {
    expiresIn: '1h',
    algorithm: 'HS256'
  }
);

// トークンの検証
function verifyToken(token: string) {
  try {
    return jwt.verify(token, process.env.JWT_SECRET as string);
  } catch (error) {
    throw new Error('Invalid token');
  }
}
```

レビュー時には、トークンの有効期限設定、秘密鍵の管理、署名アルゴリズムの選択が適切か確認しましょう。

### ロールベースのアクセス制御

```typescript
// ミドルウェアでの権限チェック
function requireRole(allowedRoles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    const user = req.user;

    if (!user) {
      return res.status(401).json({ error: 'Unauthorized' });
    }

    if (!allowedRoles.includes(user.role)) {
      return res.status(403).json({ error: 'Forbidden' });
    }

    next();
  };
}

// 使用例
app.delete('/api/users/:id', requireRole(['admin']), async (req, res) => {
  // 管理者のみが実行可能
  await deleteUser(req.params.id);
  res.json({ success: true });
});
```

認可のチェックは、必ずサーバー側で行いましょう。クライアント側のみの制御は容易に回避されます。

## 機密情報のハードコード検出

### 環境変数の使用

```typescript
// 悪い例: APIキーがハードコード
const apiKey = 'sk_live_1234567890abcdef';
const dbPassword = 'mypassword123';

fetch('https://api.example.com/data', {
  headers: { 'Authorization': `Bearer ${apiKey}` }
});

// 良い例: 環境変数から取得
const apiKey = process.env.API_KEY;
const dbPassword = process.env.DB_PASSWORD;

if (!apiKey || !dbPassword) {
  throw new Error('Required environment variables are not set');
}

fetch('https://api.example.com/data', {
  headers: { 'Authorization': `Bearer ${apiKey}` }
});
```

レビュー時には、APIキー、パスワード、秘密鍵などがハードコードされていないか確認しましょう。gitleaksやtruffleHogなどのツールを活用することも有効です。

### .gitignoreの適切な設定

```gitignore
# 環境変数ファイル
.env
.env.local
.env.production

# 認証情報
credentials.json
secrets.yaml

# データベースファイル
*.sqlite
*.db
```

機密情報を含むファイルが誤ってコミットされないよう、.gitignoreに追加しましょう。

## 依存関係の脆弱性チェック

### npm auditの活用

```bash
# 脆弱性のスキャン
npm audit

# 修正可能な脆弱性の自動修正
npm audit fix

# package-lock.jsonの更新を含む修正
npm audit fix --force
```

レビュー時には、package.jsonやpackage-lock.jsonの変更があった場合、npm auditの結果を確認しましょう。

### Dependabotの設定

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```

Dependabotを設定することで、依存関係の脆弱性を自動的に検出し、プルリクエストを作成できます。

## セキュリティレビューチェックリスト

レビュー時に確認すべき項目をチェックリスト形式でまとめます。

- [ ] ユーザー入力がSQL文に直接組み込まれていないか
- [ ] パラメータ化クエリまたはORMが使用されているか
- [ ] ユーザー入力が適切にエスケープ・サニタイズされているか
- [ ] CSRFトークンが状態変更エンドポイントに実装されているか
- [ ] パスワードが適切なハッシュ関数（bcrypt等）でハッシュ化されているか
- [ ] JWTの秘密鍵が環境変数から取得されているか
- [ ] 認可チェックがサーバー側で実装されているか
- [ ] APIキーやパスワードがハードコードされていないか
- [ ] 機密情報を含むファイルが.gitignoreに追加されているか
- [ ] 依存関係に既知の脆弱性がないか（npm audit等で確認）
- [ ] HTTPSが使用され、Cookie属性（Secure、HttpOnly、SameSite）が適切に設定されているか
- [ ] エラーメッセージに機密情報が含まれていないか

## まとめ

本章では、セキュリティレビューにおける重要なポイントを解説しました。

主なポイント:
- OWASP Top 10を意識した脆弱性チェック
- SQLインジェクション、XSS、CSRFなどの主要な攻撃への対策
- 適切な認証・認可の実装確認
- 機密情報のハードコード検出と環境変数の使用
- 依存関係の脆弱性管理

セキュリティは、一度の対策で完結するものではなく、継続的な改善が必要です。コードレビューの段階で脆弱性を発見し、修正することで、セキュアなシステムを構築しましょう。

## 参考文献

- [OWASP Top Ten](https://owasp.org/www-project-top-ten/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [MDN Web Security](https://developer.mozilla.org/en-US/docs/Web/Security)
- [Node.js Security Best Practices](https://nodejs.org/en/docs/guides/security/)
- [GitHub Security Best Practices](https://docs.github.com/en/code-security)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
