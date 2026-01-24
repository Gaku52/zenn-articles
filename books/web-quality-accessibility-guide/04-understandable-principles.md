# Understandable（理解可能） - 情報と操作を理解できるようにする

## Understandableとは

Understandable（理解可能）は、WCAG 2.1の第3原則です。情報とユーザーインターフェースの操作が、ユーザーにとって理解できることを求めます。

テキストの可読性、予測可能な動作、入力エラーの回避と修正が重要です。

## ガイドライン3.1: 読みやすさ

### 達成基準3.1.1 ページの言語（Level A）

**要件**: ページの主要言語がプログラム的に決定できる。

#### 実装例: HTML言語属性

```tsx
// ✅ 良い例: 日本語ページ
<html lang="ja">
  <head>
    <title>Web品質・アクセシビリティ完全ガイド</title>
  </head>
  <body>
    {/* コンテンツ */}
  </body>
</html>

// ✅ 良い例: 英語ページ
<html lang="en">
  <head>
    <title>Web Quality and Accessibility Guide</title>
  </head>
  <body>
    {/* Content */}
  </body>
</html>
```

### 達成基準3.1.2 一部分の言語（Level AA）

**要件**: 他の言語で書かれた部分がプログラム的に決定できる。

#### 実装例: 部分的な言語指定

```tsx
// ✅ 良い例: 日本語ページ内の英語部分
<p>
  この技術は<span lang="en">WCAG 2.1</span>に準拠しています。
</p>

<article lang="ja">
  <h1>技術ブログ</h1>
  <p>今日のトピックは...</p>

  <blockquote lang="en">
    <p>Accessibility is essential for developers and organizations...</p>
    <cite>W3C</cite>
  </blockquote>
</article>
```

## ガイドライン3.2: 予測可能

### 達成基準3.2.1 フォーカス時（Level A）

**要件**: コンポーネントがフォーカスを受け取ったときに、コンテキストの変化が起こらない。

#### 実装例: フォーカス時の動作

```tsx
// ✅ 良い例: フォーカスだけでは変化しない
<select
  value={value}
  onChange={(e) => setValue(e.target.value)}
  aria-label="国を選択"
>
  <option value="">選択してください</option>
  <option value="jp">日本</option>
  <option value="us">アメリカ</option>
</select>

// ❌ 悪い例: フォーカスで自動送信
<select
  onFocus={() => {
    // フォーカスだけで送信されてしまう
    submitForm()
  }}
>
  {/* options */}
</select>
```

### 達成基準3.2.2 入力時（Level A）

**要件**: 設定変更時に、ユーザーに事前に知らせない限り、自動的にコンテキストの変化が起こらない。

#### 実装例: 明示的な送信

```tsx
// ✅ 良い例: 明示的な送信ボタン
<form onSubmit={handleSubmit}>
  <label htmlFor="country">国</label>
  <select
    id="country"
    value={country}
    onChange={(e) => setCountry(e.target.value)}
  >
    <option value="">選択してください</option>
    <option value="jp">日本</option>
    <option value="us">アメリカ</option>
  </select>
  <button type="submit">送信</button>
</form>

// ❌ 悪い例: 選択で自動送信
<select
  onChange={(e) => {
    setCountry(e.target.value)
    submitForm() // 自動送信
  }}
>
  {/* options */}
</select>
```

### 達成基準3.2.3 一貫したナビゲーション（Level AA）

**要件**: 繰り返されるナビゲーションメカニズムが一貫した順序で表示される。

#### 実装例: 一貫したナビゲーション

```tsx
// components/Header.tsx - 全ページで共通
export function Header() {
  return (
    <header>
      <nav aria-label="メインナビゲーション">
        <ul className="flex gap-4">
          <li><a href="/">ホーム</a></li>
          <li><a href="/products">製品</a></li>
          <li><a href="/about">会社概要</a></li>
          <li><a href="/contact">お問い合わせ</a></li>
        </ul>
      </nav>
    </header>
  )
}

// 全てのページで同じ順序で表示
```

### 達成基準3.2.4 一貫した識別性（Level AA）

**要件**: 同じ機能を持つコンポーネントは一貫して識別される。

#### 実装例: 一貫したアイコンとラベル

```tsx
// ✅ 良い例: 検索機能が常に同じ
<button aria-label="検索">
  <SearchIcon />
  検索
</button>

// 別のページでも同じ
<button aria-label="検索">
  <SearchIcon />
  検索
</button>

// ❌ 悪い例: ページごとに異なる
// ページ1
<button aria-label="検索">
  <SearchIcon />
  探す
</button>

// ページ2
<button aria-label="サーチ">
  <MagnifyingGlassIcon />
  サーチ
</button>
```

## ガイドライン3.3: 入力支援

### 達成基準3.3.1 エラーの特定（Level A）

**要件**: 入力エラーが自動的に検出され、エラー箇所がテキストで説明される。

#### 実装例: エラーメッセージ

```tsx
'use client'

import { useState } from 'react'

export function ContactForm() {
  const [errors, setErrors] = useState<Record<string, string>>({})

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault()
    const formData = new FormData(e.currentTarget)
    const newErrors: Record<string, string> = {}

    const email = formData.get('email') as string
    if (!email) {
      newErrors.email = 'メールアドレスを入力してください'
    } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      newErrors.email = 'メールアドレスの形式が正しくありません'
    }

    if (Object.keys(newErrors).length > 0) {
      setErrors(newErrors)
      return
    }

    // 送信処理
  }

  return (
    <form onSubmit={handleSubmit}>
      {Object.keys(errors).length > 0 && (
        <div
          role="alert"
          aria-live="assertive"
          className="p-4 mb-4 bg-red-100 border border-red-400 rounded"
        >
          <h2 className="font-bold">入力エラーがあります</h2>
          <ul>
            {Object.entries(errors).map(([field, message]) => (
              <li key={field}>{message}</li>
            ))}
          </ul>
        </div>
      )}

      <div>
        <label htmlFor="email">
          メールアドレス
          <span className="text-red-600" aria-label="必須">*</span>
        </label>
        <input
          id="email"
          name="email"
          type="email"
          required
          aria-required="true"
          aria-invalid={errors.email ? 'true' : 'false'}
          aria-describedby={errors.email ? 'email-error' : undefined}
          className={errors.email ? 'border-red-600' : ''}
        />
        {errors.email && (
          <p id="email-error" className="text-red-600 text-sm mt-1">
            {errors.email}
          </p>
        )}
      </div>

      <button type="submit">送信</button>
    </form>
  )
}
```

### 達成基準3.3.2 ラベル又は説明（Level A）

**要件**: ユーザー入力が必要な場合、ラベルまたは説明が提供される。

#### 実装例: フォームのラベルと説明

```tsx
<form>
  <div>
    <label htmlFor="password">
      パスワード
      <span className="text-red-600" aria-label="必須">*</span>
    </label>
    <input
      id="password"
      type="password"
      required
      aria-required="true"
      aria-describedby="password-help"
    />
    <p id="password-help" className="text-sm text-gray-600 mt-1">
      8文字以上、大文字・小文字・数字を含む必要があります
    </p>
  </div>

  <fieldset>
    <legend>連絡方法（少なくとも1つ選択してください）</legend>
    <label>
      <input type="checkbox" name="contact" value="email" />
      メール
    </label>
    <label>
      <input type="checkbox" name="contact" value="phone" />
      電話
    </label>
  </fieldset>

  <button type="submit">送信</button>
</form>
```

### 達成基準3.3.3 エラー修正の提案（Level AA）

**要件**: エラーが検出され、修正案が分かる場合、ユーザーに提案する。

#### 実装例: エラー修正の提案

```tsx
'use client'

import { useState } from 'react'

export function EmailForm() {
  const [email, setEmail] = useState('')
  const [suggestion, setSuggestion] = useState('')

  const commonDomains = ['gmail.com', 'yahoo.co.jp', 'outlook.com']

  const handleEmailChange = (value: string) => {
    setEmail(value)

    // 簡単なドメイン提案
    if (value.includes('@')) {
      const [, domain] = value.split('@')
      const suggestion = commonDomains.find(d => d.startsWith(domain))
      if (suggestion && suggestion !== domain) {
        setSuggestion(value.split('@')[0] + '@' + suggestion)
      } else {
        setSuggestion('')
      }
    }
  }

  return (
    <div>
      <label htmlFor="email">メールアドレス</label>
      <input
        id="email"
        type="email"
        value={email}
        onChange={(e) => handleEmailChange(e.target.value)}
        aria-describedby={suggestion ? 'email-suggestion' : undefined}
      />
      {suggestion && (
        <div id="email-suggestion" className="mt-2 text-sm text-blue-600">
          もしかして: {suggestion}?
          <button
            type="button"
            onClick={() => {
              setEmail(suggestion)
              setSuggestion('')
            }}
            className="ml-2 underline"
          >
            これを使用
          </button>
        </div>
      )}
    </div>
  )
}
```

### 達成基準3.3.4 エラー回避（Level AA）

**要件**: 法的義務、金融取引、データ削除などを伴うページでは、送信を取り消せる、確認できる、またはチェックできる。

#### 実装例: 削除の確認

```tsx
'use client'

import { useState } from 'react'

export function DeleteButton({ itemName, onDelete }: { itemName: string, onDelete: () => void }) {
  const [showConfirm, setShowConfirm] = useState(false)

  return (
    <>
      <button
        onClick={() => setShowConfirm(true)}
        className="px-4 py-2 bg-red-600 text-white rounded"
      >
        削除
      </button>

      {showConfirm && (
        <div
          role="dialog"
          aria-modal="true"
          aria-labelledby="confirm-title"
          className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center"
        >
          <div className="bg-white p-6 rounded-lg">
            <h2 id="confirm-title" className="text-xl font-bold mb-4">
              削除の確認
            </h2>
            <p className="mb-4">
              本当に「{itemName}」を削除しますか？
              この操作は取り消せません。
            </p>
            <div className="flex gap-2">
              <button
                onClick={() => setShowConfirm(false)}
                className="px-4 py-2 bg-gray-200 rounded"
              >
                キャンセル
              </button>
              <button
                onClick={() => {
                  onDelete()
                  setShowConfirm(false)
                }}
                className="px-4 py-2 bg-red-600 text-white rounded"
              >
                削除する
              </button>
            </div>
          </div>
        </div>
      )}
    </>
  )
}
```

#### 実装例: フォーム送信の確認

```tsx
'use client'

import { useState } from 'react'

export function OrderForm() {
  const [step, setStep] = useState<'input' | 'confirm' | 'complete'>('input')
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    address: '',
  })

  const handleSubmit = async () => {
    // 送信処理
    await submitOrder(formData)
    setStep('complete')
  }

  if (step === 'confirm') {
    return (
      <div>
        <h2>注文内容の確認</h2>
        <dl>
          <dt>お名前</dt>
          <dd>{formData.name}</dd>
          <dt>メールアドレス</dt>
          <dd>{formData.email}</dd>
          <dt>配送先住所</dt>
          <dd>{formData.address}</dd>
        </dl>
        <div className="flex gap-2 mt-4">
          <button
            onClick={() => setStep('input')}
            className="px-4 py-2 bg-gray-200 rounded"
          >
            修正する
          </button>
          <button
            onClick={handleSubmit}
            className="px-4 py-2 bg-blue-600 text-white rounded"
          >
            この内容で注文する
          </button>
        </div>
      </div>
    )
  }

  if (step === 'complete') {
    return (
      <div role="alert" aria-live="polite">
        <h2>注文が完了しました</h2>
        <p>ご注文ありがとうございました。</p>
      </div>
    )
  }

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault()
        setStep('confirm')
      }}
    >
      {/* フォーム入力 */}
      <div>
        <label htmlFor="name">お名前</label>
        <input
          id="name"
          value={formData.name}
          onChange={(e) => setFormData({ ...formData, name: e.target.value })}
          required
        />
      </div>
      <button type="submit">確認画面へ</button>
    </form>
  )
}

async function submitOrder(data: any) {
  // 送信処理
}
```

## 実践的なパターン

### パターン1: リアルタイムバリデーション

```tsx
'use client'

import { useState, useEffect } from 'react'

export function RealtimeValidation() {
  const [username, setUsername] = useState('')
  const [isChecking, setIsChecking] = useState(false)
  const [isAvailable, setIsAvailable] = useState<boolean | null>(null)

  useEffect(() => {
    if (username.length < 3) {
      setIsAvailable(null)
      return
    }

    const timer = setTimeout(async () => {
      setIsChecking(true)
      // APIで確認（実際の実装）
      const available = await checkUsername(username)
      setIsAvailable(available)
      setIsChecking(false)
    }, 500)

    return () => clearTimeout(timer)
  }, [username])

  return (
    <div>
      <label htmlFor="username">ユーザー名</label>
      <input
        id="username"
        value={username}
        onChange={(e) => setUsername(e.target.value)}
        aria-describedby="username-status"
        aria-invalid={isAvailable === false ? 'true' : 'false'}
      />
      <div id="username-status" aria-live="polite">
        {isChecking && '確認中...'}
        {isAvailable === true && (
          <span className="text-green-600">✓ 使用可能です</span>
        )}
        {isAvailable === false && (
          <span className="text-red-600">× すでに使用されています</span>
        )}
      </div>
    </div>
  )
}

async function checkUsername(username: string): Promise<boolean> {
  // 実際のAPI呼び出し
  return new Promise((resolve) => {
    setTimeout(() => resolve(Math.random() > 0.5), 500)
  })
}
```

### パターン2: 段階的な開示

```tsx
'use client'

import { useState } from 'react'

export function ProgressiveDisclosure() {
  const [showAdvanced, setShowAdvanced] = useState(false)

  return (
    <form>
      <div>
        <label htmlFor="name">名前</label>
        <input id="name" type="text" required />
      </div>

      <div>
        <label htmlFor="email">メール</label>
        <input id="email" type="email" required />
      </div>

      <button
        type="button"
        onClick={() => setShowAdvanced(!showAdvanced)}
        aria-expanded={showAdvanced}
        aria-controls="advanced-options"
      >
        {showAdvanced ? '詳細オプションを隠す' : '詳細オプションを表示'}
      </button>

      {showAdvanced && (
        <div id="advanced-options">
          <div>
            <label htmlFor="company">会社名</label>
            <input id="company" type="text" />
          </div>
          <div>
            <label htmlFor="department">部署</label>
            <input id="department" type="text" />
          </div>
        </div>
      )}

      <button type="submit">送信</button>
    </form>
  )
}
```

## まとめ

Understandable（理解可能）原則は、情報とUIの操作が理解できることを求めます。主なポイント：

- ページと部分的な言語を指定
- 予測可能な動作
- 一貫したナビゲーションと識別
- エラーの特定と修正の提案
- 重要な操作には確認ステップ

### チェックリスト

- html要素にlang属性がある
- フォームに明確なラベルと説明がある
- エラーメッセージが具体的
- 削除などの重要な操作に確認ダイアログがある
- ナビゲーションが全ページで一貫している

## 次のステップ

次章では、WCAG 2.1の第4原則「Robust（堅牢）」について解説します。
