---
title: "CSRF対策"
---

## CSRF攻撃とは何か

CSRF（Cross-Site Request Forgery）は、認証済みユーザーのブラウザを悪用して、意図しないリクエストを送信させる攻撃です。日本語では「クロスサイトリクエストフォージェリ」と呼ばれます。

攻撃者は、ユーザーがログイン中のサービスに対して、ユーザーの知らないうちに操作（送金、パスワード変更、メールアドレス変更など）を実行させます。

## CSRF攻撃の仕組み

### 基本的な攻撃フロー

CSRF攻撃は以下の流れで成立します。

1. ユーザーが `bank.com` にログインし、セッションCookieが有効な状態になります
2. 攻撃者が `evil.com` に悪意のあるHTMLを仕込みます
3. ユーザーが `evil.com` を訪問すると、隠しフォームが自動送信されます
4. ブラウザは `bank.com` へのリクエストにCookieを自動付与します
5. `bank.com` はユーザーからの正規リクエストと判断し、処理を実行します

```html
<!-- evil.com に設置された攻撃用HTML -->
<html>
<body onload="document.forms[0].submit()">
  <form action="https://bank.com/api/transfer" method="POST">
    <input type="hidden" name="to" value="attacker-account" />
    <input type="hidden" name="amount" value="1000000" />
  </form>
</body>
</html>
```

この攻撃が成功する根本的な原因は、ブラウザがクロスサイトリクエストでもCookieを自動送信するという仕様にあります。

### 攻撃が成立する3つの条件

CSRF攻撃は、以下の3つの条件がすべて揃った場合に成立します。

| 条件 | 説明 |
|------|------|
| Cookieベースの認証 | セッションCookieが自動送信される |
| 予測可能なリクエスト構造 | パラメータ名・値が推測可能で、ランダムトークンが不要 |
| 状態変更を行う操作 | 送金・パスワード変更・メールアドレス変更など |

### 攻撃の種類

CSRFにはいくつかの攻撃パターンがあります。

- **POST CSRF** -- 隠しフォームの自動送信による最も一般的な攻撃です
- **GET CSRF** -- `<img src="bank.com/transfer?to=evil">` のようにGETリクエストで状態変更するAPIが狙われます
- **Login CSRF** -- 攻撃者のアカウントでユーザーをログインさせ、入力した情報を窃取します
- **JSON CSRF** -- `enctype="text/plain"` を悪用してJSONリクエストを偽装します

## Synchronizer Token Pattern

最も確実なCSRF防御パターンが「Synchronizer Token Pattern（同期トークンパターン）」です。

### 仕組み

1. サーバーがランダムなトークンを生成し、セッションに紐づけて保存します
2. フォームの `hidden` フィールドにトークンを埋め込みます
3. POST送信時にサーバーがトークンを検証します
4. 攻撃者はトークンの値を知ることができないため、攻撃が失敗します

### 実装例（Express）

```typescript
import crypto from "crypto";
import { Request, Response, NextFunction } from "express";

// トークン生成（暗号論的に安全なランダム値）
function generateCSRFToken(): string {
  return crypto.randomBytes(32).toString("hex");
}

// タイミング攻撃を防ぐ安全な比較
function safeCompare(a: string, b: string): boolean {
  if (a.length !== b.length) return false;
  return crypto.timingSafeEqual(Buffer.from(a), Buffer.from(b));
}

// CSRFミドルウェア
function csrfProtection() {
  return async (req: Request, res: Response, next: NextFunction) => {
    // GET/HEAD/OPTIONSはスキップ
    if (["GET", "HEAD", "OPTIONS"].includes(req.method)) {
      const sessionId = req.session?.id;
      if (sessionId) {
        let token = req.session.csrfToken;
        if (!token) {
          token = generateCSRFToken();
          req.session.csrfToken = token;
        }
        res.locals.csrfToken = token;
      }
      return next();
    }

    // POST/PUT/DELETE等の検証
    const sessionId = req.session?.id;
    if (!sessionId) {
      return res.status(403).json({ error: "No session" });
    }

    const token =
      (req.headers["x-csrf-token"] as string) ||
      req.body?._csrf;

    const storedToken = req.session.csrfToken;

    if (!token || !storedToken || !safeCompare(token, storedToken)) {
      return res.status(403).json({
        error: "Invalid CSRF token",
      });
    }

    next();
  };
}
```

`crypto.timingSafeEqual()` を使用しているのは、通常の `===` 比較がタイミング攻撃に対して脆弱だからです。通常の比較は最初の不一致文字で処理を終了するため、レスポンス時間からトークンを推測される恐れがあります。

### Per-Session Token と Per-Request Token

トークンの更新戦略には2種類があります。

| 戦略 | メリット | デメリット |
|------|---------|----------|
| Per-Session Token | 実装がシンプル、「戻る」ボタンで問題が起きない | トークン漏洩時の影響が大きい |
| Per-Request Token | セキュリティが高い、漏洩時の影響が限定的 | 「戻る」ボタンでフォーム再送信が失敗する |

一般的なWebアプリでは **Per-Session Token** が推奨されます。金融系・高セキュリティが求められる場合は **Per-Request Token** を検討してください。

## Double Submit Cookie

サーバー側に状態を持たずにCSRF防御を実現するパターンです。

### 仕組み

1. サーバーがランダムなトークンをCookie（`httpOnly: false`）に設定します
2. クライアントのJavaScriptがCookieからトークンを読み取ります
3. リクエスト時にCookieの値をカスタムヘッダー（`X-CSRF-Token`）にも設定します
4. サーバーがCookieの値とヘッダーの値が一致するかを検証します

攻撃者のサイトからは同一オリジンポリシーにより対象サイトのCookieを読み取ることができないため、ヘッダーに正しい値を設定できず攻撃が失敗します。

### 実装例（Express）

```typescript
import crypto from "crypto";
import { Request, Response, NextFunction } from "express";

function doubleSubmitCSRF(secret: string) {
  // HMAC署名付きトークンを生成
  function createSignedToken(): string {
    const value = crypto.randomBytes(32).toString("hex");
    const signature = crypto
      .createHmac("sha256", secret)
      .update(value)
      .digest("hex");
    return `${value}.${signature}`;
  }

  // 署名を検証
  function verifySignature(token: string): boolean {
    const [value, signature] = token.split(".");
    if (!value || !signature) return false;
    const expected = crypto
      .createHmac("sha256", secret)
      .update(value)
      .digest("hex");
    return crypto.timingSafeEqual(
      Buffer.from(signature),
      Buffer.from(expected)
    );
  }

  return (req: Request, res: Response, next: NextFunction) => {
    if (["GET", "HEAD", "OPTIONS"].includes(req.method)) {
      if (!req.cookies["csrf-token"]) {
        const token = createSignedToken();
        res.cookie("csrf-token", token, {
          httpOnly: false, // JSから読み取り可能にする
          secure: process.env.NODE_ENV === "production",
          sameSite: "lax",
          path: "/",
        });
      }
      return next();
    }

    const cookieToken = req.cookies["csrf-token"];
    const headerToken = req.headers["x-csrf-token"] as string;

    if (!cookieToken || !headerToken) {
      return res.status(403).json({ error: "Missing CSRF token" });
    }

    if (!verifySignature(cookieToken)) {
      return res.status(403).json({ error: "Invalid token signature" });
    }

    if (
      !crypto.timingSafeEqual(
        Buffer.from(cookieToken),
        Buffer.from(headerToken)
      )
    ) {
      return res.status(403).json({ error: "CSRF token mismatch" });
    }

    next();
  };
}
```

### なぜHMAC署名が必要なのか

署名なしのDouble Submit Cookieには弱点があります。攻撃者がサブドメイン（`evil.sub.example.com`）を制御している場合、`Set-Cookie: csrf=attacker-value; domain=.example.com` でCookieを上書きし、ヘッダーにも同じ値を設定することで検証を通過できてしまいます。

HMAC署名を付与すると、攻撃者はサーバーの秘密鍵（`secret`）を知らないため正しい署名を生成できません。Cookie Injection攻撃を防ぐことができます。

## SameSite Cookie

SameSite属性は、ブラウザレベルでクロスサイトリクエスト時のCookie送信を制御する仕組みです。追加のコード実装なしでCSRF攻撃の大部分を防御できます。

### 3つの値

| 値 | 動作 | 用途 |
|----|------|------|
| `Strict` | クロスサイトリクエストでCookieを一切送信しない | 管理画面など高セキュリティ領域 |
| `Lax` | トップレベルGETナビゲーションのみCookie送信 | 一般的なWebアプリ（推奨） |
| `None` | すべてのクロスサイトリクエストでCookie送信（`Secure`必須） | サードパーティCookieが必要な場合 |

### リクエスト種別ごとのCookie送信

| リクエスト種別 | Strict | Lax | None |
|--------------|--------|-----|------|
| `<a href>` リンククリック | 送信しない | 送信する | 送信する |
| `<form method="GET">` | 送信しない | 送信する | 送信する |
| `<form method="POST">` | 送信しない | 送信しない | 送信する |
| `<img src>` / `<iframe>` | 送信しない | 送信しない | 送信する |
| `fetch()` / `XMLHttpRequest` | 送信しない | 送信しない | 送信する |

`SameSite=Lax` を設定すれば、POST CSRFの主要な攻撃パターンを防御できます。

### SameSite Cookieの推奨設定

```typescript
import { CookieOptions } from "express";

// 一般的なセッションCookie
const sessionCookieOptions: CookieOptions = {
  httpOnly: true,
  secure: true,
  sameSite: "lax",
  path: "/",
  maxAge: 30 * 24 * 60 * 60 * 1000, // 30日
};

// 管理画面用（高セキュリティ）
const adminCookieOptions: CookieOptions = {
  httpOnly: true,
  secure: true,
  sameSite: "strict",
  path: "/admin",
  maxAge: 4 * 60 * 60 * 1000, // 4時間
};
```

### SameSiteの限界

SameSite Cookieだけでは完全な防御にはなりません。

- **GETでの状態変更**: `GET /delete-account` のようなAPIは `Lax` でも攻撃可能です
- **サブドメイン間**: `app.example.com` と `evil.example.com` は同一サイトと判定されます
- **古いブラウザ**: SameSiteをサポートしないブラウザが一部存在します

そのため、SameSite Cookieは他の防御パターンと組み合わせて使用することが推奨されます。

## CSRF対策の実装例

### Next.js Server Actionsの場合

Next.js App RouterのServer Actionsは、フレームワークが自動的にCSRF保護を行います。内部でOriginヘッダーの検証が実施されるため、追加の実装は不要です。

```typescript
// app/actions/article.ts
"use server";

import { auth } from "@/auth";
import { revalidatePath } from "next/cache";

export async function createArticle(formData: FormData) {
  // Server Actionsは自動的にCSRF保護される
  const session = await auth();
  if (!session) throw new Error("Unauthorized");

  const title = formData.get("title") as string;
  const content = formData.get("content") as string;

  await prisma.article.create({
    data: { title, content, authorId: session.user.id },
  });

  revalidatePath("/articles");
}
```

### Next.js API Routesの場合

API Routesでは手動でCSRF対策を行う必要があります。Middlewareを使ったOriginヘッダー検証が効果的です。

```typescript
// middleware.ts
import { NextRequest, NextResponse } from "next/server";

export function middleware(request: NextRequest) {
  if (
    request.nextUrl.pathname.startsWith("/api/") &&
    !["GET", "HEAD", "OPTIONS"].includes(request.method)
  ) {
    const origin = request.headers.get("origin");
    const host = request.headers.get("host");

    if (origin) {
      const originUrl = new URL(origin);
      const expectedHost = host?.split(":")[0];
      if (originUrl.hostname !== expectedHost) {
        return NextResponse.json(
          { error: "CSRF validation failed" },
          { status: 403 }
        );
      }
    } else {
      // OriginもRefererもない場合は拒否
      const referer = request.headers.get("referer");
      if (!referer) {
        return NextResponse.json(
          { error: "Missing Origin header" },
          { status: 403 }
        );
      }
    }
  }
  return NextResponse.next();
}
```

### SPA（React）での実装

Cookie認証を使うSPAの場合は、Double Submit Cookieパターンが適しています。

```typescript
// CSRFトークンを自動付与するfetchラッパー
function csrfFetch(url: string, options: RequestInit = {}): Promise<Response> {
  const csrfToken = document.cookie
    .split("; ")
    .find((row) => row.startsWith("csrf-token="))
    ?.split("=")[1];

  return fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      "X-CSRF-Token": csrfToken || "",
    },
    credentials: "same-origin",
  });
}
```

### 多層防御の設計

実際のアプリケーションでは、複数の防御レイヤーを組み合わせることが重要です。一般的なWebアプリであれば **SameSite=Lax + Originヘッダー検証** で十分です。金融系・医療系では、さらにCSRFトークンを追加してください。

避けるべきアンチパターンとして、GETで状態変更する設計（`<img>` タグで攻撃可能）、`Math.random()` によるトークン生成（暗号的に不安全）、CSRFトークンのURL埋め込み（ログやRefererで漏洩）が挙げられます。

## まとめ

| 防御方法 | セキュリティ | 実装コスト | サーバー状態 | SPA適合性 | 推奨用途 |
|---------|------------|-----------|------------|----------|---------|
| SameSite=Lax | 高 | 最小 | 不要 | 自動適用 | 全アプリで必須 |
| Synchronizer Token | 最高 | 中 | 必要（Redis等） | 要工夫 | 高セキュリティ要件 |
| Double Submit Cookie | 高 | 低 | 不要 | 適合 | SPA + Cookie認証 |
| Signed Double Submit | 最高 | 中 | 不要 | 適合 | サブドメインリスクあり |
| Originヘッダー検証 | 中 | 低 | 不要 | 適合 | 補助的な防御 |
| Custom Header | 高 | 低 | 不要 | 最適 | APIのみのサービス |

CSRF攻撃は「ブラウザがCookieを自動送信する」という仕様を悪用するものです。防御の基本は「正規のユーザー操作であることを証明する仕組み」を追加することにあります。まずは `SameSite=Lax` を設定し、必要に応じてトークンベースの防御を追加してください。

## やってみよう！

1. Expressで簡単なフォームアプリを作成し、SameSite属性なしのCookieでCSRF攻撃が成立することを確認してみましょう
2. `SameSite=Lax` を設定して、POSTによるCSRF攻撃がブロックされることをDevToolsのNetworkタブで確認してみましょう
3. Double Submit Cookieパターンを実装し、トークンなしのリクエストが403で拒否されることをテストしてみましょう
4. Next.js App RouterのServer Actionsを使い、異なるOriginからのリクエストが自動的にブロックされることを確認してみましょう
