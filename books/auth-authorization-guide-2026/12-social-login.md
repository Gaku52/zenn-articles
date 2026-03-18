---
title: "ソーシャルログイン"
---

## この章で学ぶこと

ソーシャルログインは、Google や GitHub、Apple といった外部プロバイダーの認証基盤を活用して、ユーザーに簡単で安全なログイン体験を提供する仕組みです。この章では、主要 3 プロバイダーの設定方法から Next.js + NextAuth.js（Auth.js v5）による実装、アカウントリンクの設計まで、実践的に解説します。

- OAuth 2.0 / OpenID Connect とソーシャルログインの関係を理解する
- Google・GitHub・Apple 各プロバイダーの設定手順と固有の注意点を把握する
- アカウントリンクを安全に実装する方法を学ぶ
- Next.js + NextAuth.js での具体的な実装パターンを身につける

## ソーシャルログインの全体像

### なぜソーシャルログインが必要なのか

ソーシャルログインを導入すると、ユーザー側・開発者側の双方にメリットがあります。ユーザーはパスワードを覚える必要がなくなり、ワンクリックでログインできます。開発者はパスワードハッシュの管理やメール検証、2FA / MFA をプロバイダーに委任できます。

業界データでは、ソーシャルログインの導入によりサインアップ率が 20〜50% 向上するとされています。Google が最も利用率が高く（約 60%）、Apple は iOS ユーザーで急速に普及しています。

### OAuth 2.0 Authorization Code Flow の流れ

ソーシャルログインは OAuth 2.0 の Authorization Code Flow（PKCE 付き）で動作します。ユーザーがログインボタンをクリックすると、プロバイダーの認可エンドポイントにリダイレクトされ、同意後に Authorization Code がコールバック URL に返されます。アプリはサーバー側で Code をアクセストークンに交換し、取得したユーザー情報でセッションを作成します。

PKCE（Proof Key for Code Exchange）は Authorization Code の横取り攻撃を防ぐ仕組みで、Auth.js が自動的に処理してくれます。

### OpenID Connect と OAuth 2.0 の違い

プロバイダーによって採用しているプロトコルが異なります。

| 項目 | OAuth 2.0 | OpenID Connect |
|------|-----------|----------------|
| 目的 | 認可（API アクセス） | 認証（ユーザー特定） |
| 返却トークン | Access Token | ID Token + Access Token |
| ユーザー情報 | API で取得 | ID Token に含まれる |
| 採用プロバイダー | GitHub | Google, Apple |

GitHub は純粋な OAuth 2.0 のみ対応のため、ID Token が返されません。ユーザー情報は `/user` API で取得します。

## Google ログインの設定

### Google Cloud Console での準備

Google ログインを実装するには、Google Cloud Console で OAuth 同意画面と認証情報を設定します。

1. [Google Cloud Console](https://console.cloud.google.com) でプロジェクトを作成する
2. 「APIs & Services > OAuth consent screen」でアプリ情報を設定する
3. スコープに `openid`、`email`、`profile` を追加する
4. 「APIs & Services > Credentials」で OAuth client ID を作成する
5. リダイレクト URI に `http://localhost:3000/api/auth/callback/google`（開発用）を設定する
6. Client ID と Client Secret を取得する

:::message
テストモードでは最大 100 人のテストユーザーまでです。本番公開には Google による審査が必要で、プライバシーポリシーと利用規約の URL が求められます。
:::

### Auth.js v5 でのプロバイダー設定

```ts
// auth.ts
import Google from "next-auth/providers/google";

export const googleProvider = Google({
  clientId: process.env.GOOGLE_CLIENT_ID!,
  clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
  authorization: {
    params: {
      scope: "openid email profile",
      prompt: "consent",
      access_type: "offline",
    },
  },
  profile(profile) {
    return {
      id: profile.sub,
      name: profile.name,
      email: profile.email,
      image: profile.picture,
    };
  },
});
```

`prompt: "consent"` と `access_type: "offline"` を指定することで、Refresh Token を確実に取得できます。これを省略すると、Access Token の期限切れ時にユーザーが再ログインを求められます。

## GitHub ログインの設定

### GitHub OAuth App の作成

GitHub では Settings > Developer settings > OAuth Apps から OAuth App を作成します。

1. Application name にアプリ名を入力する
2. Authorization callback URL に `http://localhost:3000/api/auth/callback/github` を設定する
3. Client ID と Client Secret を取得する

:::message alert
GitHub OAuth App はコールバック URL を 1 つしか設定できません。開発・ステージング・本番で別々の OAuth App が必要です。
:::

ログイン目的であれば、スコープは `read:user user:email` で十分です。

### Auth.js v5 での設定とメール取得の注意点

GitHub ではユーザーがメールを非公開に設定している場合、`profile.email` が `null` になります。`/user/emails` API を使って取得する必要があります。

```ts
// auth.ts
import GitHub from "next-auth/providers/github";

export const githubProvider = GitHub({
  clientId: process.env.GITHUB_CLIENT_ID!,
  clientSecret: process.env.GITHUB_CLIENT_SECRET!,
  authorization: {
    params: {
      scope: "read:user user:email",
    },
  },
  profile(profile) {
    return {
      id: String(profile.id),
      name: profile.name || profile.login,
      email: profile.email,
      image: profile.avatar_url,
    };
  },
});
```

メールが `null` の場合に `/user/emails` API で補完する処理を `signIn` コールバックに追加します。

```ts
callbacks: {
  async signIn({ user, account }) {
    if (account?.provider === "github" && !user.email) {
      const res = await fetch("https://api.github.com/user/emails", {
        headers: {
          Authorization: `Bearer ${account.access_token}`,
          Accept: "application/vnd.github+json",
        },
      });
      if (res.ok) {
        const emails = await res.json();
        const primary = emails.find(
          (e: any) => e.primary && e.verified
        );
        if (primary) user.email = primary.email;
      }
    }
    return true;
  },
},
```

## Apple ログインの設定

### Apple Developer での準備

Apple ログインは他のプロバイダーと比較して設定が複雑です。

**前提条件:**

- Apple Developer Program（年額 $99）への加入が必須
- Web 向けには Services ID が必要
- localhost は Return URL に使用できないため、開発時は ngrok 等のトンネルが必要

**設定手順:**

1. Identifiers > App IDs で Sign in with Apple の Capability を有効化する
2. Identifiers > Services IDs で Web 用の Services ID を作成する
3. Keys から Sign in with Apple 用の Key を作成し、`.p8` ファイルをダウンロードする

:::message alert
`.p8` ファイルは一度しかダウンロードできません。紛失した場合は Key を再作成する必要があります。
:::

### Auth.js v5 での設定

Apple では `clientSecret` を JWT 形式で動的に生成する必要があります。

```ts
// auth.ts
import Apple from "next-auth/providers/apple";
import * as jose from "jose";

async function generateAppleClientSecret(): Promise<string> {
  const privateKeyPem = process.env.APPLE_PRIVATE_KEY!.replace(/\\n/g, "\n");
  const privateKey = await jose.importPKCS8(privateKeyPem, "ES256");

  return new jose.SignJWT({})
    .setProtectedHeader({ alg: "ES256", kid: process.env.APPLE_KEY_ID! })
    .setIssuedAt()
    .setExpirationTime("180d")
    .setAudience("https://appleid.apple.com")
    .setIssuer(process.env.APPLE_TEAM_ID!)
    .setSubject(process.env.APPLE_CLIENT_ID!)
    .sign(privateKey);
}

export const appleProvider = Apple({
  clientId: process.env.APPLE_CLIENT_ID!,
  clientSecret: await generateAppleClientSecret(),
});
```

### Apple 固有の注意点

Apple ログインには他のプロバイダーにはない特殊な挙動があります。

**name と email は初回認可時のみ返却されます。** 2 回目以降のログインでは `sub`（ユーザー識別子）しか返されません。初回ログイン時に必ず DB へ保存してください。

**Private Email Relay:** ユーザーが「メールを非公開」を選択すると、`xxxxx@privaterelay.appleid.com` という中継アドレスが返されます。このアドレスにメールを送るには、Apple Developer で送信元ドメインを登録し、SPF / DKIM を設定する必要があります。

## Next.js + NextAuth.js 統合実装

ここまでの 3 プロバイダーを統合した実装例を示します。

### auth.ts の全体構成

```ts
// auth.ts
import NextAuth from "next-auth";
import { PrismaAdapter } from "@auth/prisma-adapter";
import { prisma } from "@/lib/prisma";
import Google from "next-auth/providers/google";
import GitHub from "next-auth/providers/github";
import Apple from "next-auth/providers/apple";

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(prisma),
  providers: [
    Google({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
      authorization: {
        params: { prompt: "consent", access_type: "offline" },
      },
    }),
    GitHub({
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
      authorization: { params: { scope: "read:user user:email" } },
    }),
    Apple({
      clientId: process.env.APPLE_CLIENT_ID!,
      clientSecret: process.env.APPLE_CLIENT_SECRET!,
    }),
  ],
  callbacks: {
    async session({ session, user }) {
      session.user.id = user.id;
      return session;
    },
  },
});
```

### ログインページの実装

```tsx
// app/login/page.tsx
import { signIn } from "@/auth";

export default function LoginPage() {
  return (
    <div className="max-w-md mx-auto mt-20 space-y-3">
      <h1 className="text-2xl font-bold text-center mb-8">ログイン</h1>

      <form action={async () => {
        "use server";
        await signIn("google", { redirectTo: "/dashboard" });
      }}>
        <button type="submit" className="w-full border rounded-lg py-3">
          Google で続ける
        </button>
      </form>

      <form action={async () => {
        "use server";
        await signIn("github", { redirectTo: "/dashboard" });
      }}>
        <button type="submit" className="w-full bg-gray-800 text-white rounded-lg py-3">
          GitHub で続ける
        </button>
      </form>

      <form action={async () => {
        "use server";
        await signIn("apple", { redirectTo: "/dashboard" });
      }}>
        <button type="submit" className="w-full bg-black text-white rounded-lg py-3">
          Apple で続ける
        </button>
      </form>
    </div>
  );
}
```

### API ルートの設定

```ts
// app/api/auth/[...nextauth]/route.ts
import { handlers } from "@/auth";
export const { GET, POST } = handlers;
```

## アカウントリンク

### なぜアカウントリンクが必要なのか

あるユーザーが Google（`alice@gmail.com`）でサインアップした後、同じメールアドレスの GitHub アカウントでログインしようとするケースを考えます。このとき、2 つのアカウントを同一ユーザーとして紐づける仕組みが「アカウントリンク」です。

### 自動リンクのセキュリティリスク

メールアドレスが一致するだけで自動リンクするのは危険です。攻撃者が被害者のメールアドレスを知っていれば、メール検証なしでアカウントを作成できるプロバイダーを使ってアカウントを乗っ取れます。

**安全なアカウントリンクの原則:** `email_verified` が `true` の場合のみ自動リンクを許可します。

```ts
function checkEmailVerification(provider: string, profile?: any): boolean {
  switch (provider) {
    case "google":
      return profile?.email_verified === true;
    case "github":
      // user:email スコープで取得したメールは検証済み
      return true;
    case "apple":
      return profile?.email_verified === true;
    default:
      return false;
  }
}
```

### 手動リンクの設定画面

ユーザーが設定画面からプロバイダーを追加・解除できるようにします。重要なのは、最後のログイン方法を解除できないようにするガード処理です。

```ts
// アカウント解除時のガード
const accountCount = await prisma.account.count({
  where: { userId: session.user.id },
});
const hasPassword = !!(await prisma.user.findUnique({
  where: { id: session.user.id },
  select: { password: true },
}))?.password;

if (accountCount <= 1 && !hasPassword) {
  return { error: "他にログイン方法がないため解除できません" };
}
```

## プロバイダー比較表

| 項目 | Google | GitHub | Apple |
|------|--------|--------|-------|
| プロトコル | OIDC | OAuth 2.0 | OIDC |
| ID Token | あり | なし | あり |
| email 取得 | 常に取得可 | null の場合あり | 初回のみ |
| Refresh Token | あり | なし（OAuth App） | あり |
| アバター | あり | あり | なし |
| 費用 | 無料 | 無料 | $99/年 |
| localhost 対応 | 可 | 可 | 不可 |

## よくあるトラブルと対処法

| 問題 | 原因 | 対処法 |
|------|------|--------|
| `OAuthAccountNotLinked` | 同じメールが別プロバイダーで登録済み | アカウントリンク機能を実装する |
| `redirect_uri_mismatch` | コールバック URL の不一致 | プロバイダー設定を確認する（末尾スラッシュに注意） |
| GitHub の email が null | ユーザーがメールを非公開に設定 | `/user/emails` API で取得する |
| Apple の name が null | 2 回目以降のログイン | 初回ログイン時に必ず DB へ保存する |
| Token 期限切れ | Refresh Token 未取得 | Google: `prompt=consent` + `access_type=offline` を設定する |

## まとめ

| 項目 | ポイント |
|------|---------|
| プロトコル | Google / Apple は OIDC、GitHub は OAuth 2.0 |
| Google 設定 | `prompt=consent` + `access_type=offline` で Refresh Token を取得する |
| GitHub 設定 | email が null になる場合があるため `/user/emails` API で補完する |
| Apple 設定 | name / email は初回のみ返却される。clientSecret は JWT で動的生成する |
| アカウントリンク | `email_verified` が true の場合のみ自動リンクを許可する |
| セキュリティ | state + PKCE は Auth.js が自動処理する。Client Secret はサーバー側のみで使用する |
| UX | 「...で続ける」形式のボタン、わかりやすいエラーメッセージを提供する |

## やってみよう！

以下のステップで、実際にソーシャルログインを体験してみましょう。

1. **Google ログインを実装する:** Google Cloud Console で OAuth App を作成し、Auth.js の Google プロバイダーを設定してください。`localhost:3000` でログイン・ログアウトが動作することを確認します。

2. **GitHub ログインを追加する:** GitHub OAuth App を作成し、2 つ目のプロバイダーとして追加してください。メールが非公開のアカウントでテストし、`/user/emails` API による補完が動作するか確認します。

3. **アカウントリンクを試す:** 同じメールアドレスの Google アカウントと GitHub アカウントで順番にログインし、`OAuthAccountNotLinked` エラーが出ることを確認してください。そのうえで、`email_verified` チェック付きの自動リンク処理を実装してみましょう。
