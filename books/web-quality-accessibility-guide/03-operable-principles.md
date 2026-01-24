# Operable（操作可能） - UIを操作できるようにする

## Operableとは

Operable（操作可能）は、WCAG 2.1の第2原則です。ユーザーインターフェースのコンポーネントとナビゲーションが、様々な入力方法で操作できることを求めます。

特にキーボード操作、十分な操作時間の提供、発作を引き起こさないデザイン、ナビゲーションの容易さが重要です。

## ガイドライン2.1: キーボードアクセシブル

### 達成基準2.1.1 キーボード（Level A）

**要件**: 全ての機能がキーボードで操作可能である。

#### 実装例: キーボード操作可能なボタン

```tsx
// ✅ 良い例: ネイティブボタンは自動的にキーボード操作可能
<button onClick={handleClick}>
  クリック
</button>

// ✅ 良い例: カスタム要素にキーボードサポート追加
<div
  role="button"
  tabIndex={0}
  onClick={handleClick}
  onKeyDown={(e) => {
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault()
      handleClick()
    }
  }}
>
  カスタムボタン
</div>

// ❌ 悪い例: キーボード操作不可
<div onClick={handleClick}>
  クリック
</div>
```

#### 実装例: ドロップダウンメニュー

```tsx
'use client'

import { useState, useRef, useEffect } from 'react'

export function Dropdown() {
  const [isOpen, setIsOpen] = useState(false)
  const [focusedIndex, setFocusedIndex] = useState(0)
  const menuRef = useRef<HTMLDivElement>(null)

  const items = ['オプション1', 'オプション2', 'オプション3']

  const handleKeyDown = (e: React.KeyboardEvent) => {
    switch (e.key) {
      case 'Enter':
      case ' ':
        e.preventDefault()
        setIsOpen(!isOpen)
        break
      case 'ArrowDown':
        e.preventDefault()
        if (isOpen) {
          setFocusedIndex((prev) => (prev + 1) % items.length)
        } else {
          setIsOpen(true)
        }
        break
      case 'ArrowUp':
        e.preventDefault()
        if (isOpen) {
          setFocusedIndex((prev) => (prev - 1 + items.length) % items.length)
        }
        break
      case 'Escape':
        setIsOpen(false)
        break
    }
  }

  return (
    <div className="relative">
      <button
        onClick={() => setIsOpen(!isOpen)}
        onKeyDown={handleKeyDown}
        aria-haspopup="true"
        aria-expanded={isOpen}
      >
        メニュー
      </button>
      {isOpen && (
        <div
          ref={menuRef}
          role="menu"
          className="absolute mt-2 bg-white border rounded shadow-lg"
        >
          {items.map((item, index) => (
            <button
              key={item}
              role="menuitem"
              className={`block w-full px-4 py-2 text-left ${
                index === focusedIndex ? 'bg-blue-100' : ''
              }`}
              onClick={() => {
                console.log(item)
                setIsOpen(false)
              }}
            >
              {item}
            </button>
          ))}
        </div>
      )}
    </div>
  )
}
```

### 達成基準2.1.2 キーボードトラップなし（Level A）

**要件**: キーボードフォーカスが特定の要素に閉じ込められない。

#### 実装例: モーダルのフォーカストラップ

```tsx
'use client'

import { useEffect, useRef } from 'react'
import { createPortal } from 'react-dom'

interface ModalProps {
  isOpen: boolean
  onClose: () => void
  title: string
  children: React.ReactNode
}

export function Modal({ isOpen, onClose, title, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    if (!isOpen) return

    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        onClose()
        return
      }

      if (e.key === 'Tab') {
        const focusableElements = modalRef.current?.querySelectorAll<HTMLElement>(
          'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
        )
        if (!focusableElements || focusableElements.length === 0) return

        const firstElement = focusableElements[0]
        const lastElement = focusableElements[focusableElements.length - 1]

        if (e.shiftKey && document.activeElement === firstElement) {
          e.preventDefault()
          lastElement.focus()
        } else if (!e.shiftKey && document.activeElement === lastElement) {
          e.preventDefault()
          firstElement.focus()
        }
      }
    }

    document.addEventListener('keydown', handleKeyDown)
    return () => document.removeEventListener('keydown', handleKeyDown)
  }, [isOpen, onClose])

  if (!isOpen) return null

  return createPortal(
    <div
      className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50"
      onClick={onClose}
    >
      <div
        ref={modalRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby="modal-title"
        className="bg-white p-6 rounded-lg max-w-md w-full"
        onClick={(e) => e.stopPropagation()}
      >
        <h2 id="modal-title" className="text-2xl font-bold mb-4">
          {title}
        </h2>
        {children}
        <button
          onClick={onClose}
          className="mt-4 px-4 py-2 bg-gray-200 rounded hover:bg-gray-300"
        >
          閉じる
        </button>
      </div>
    </div>,
    document.body
  )
}
```

### 達成基準2.1.4 文字キーのショートカット（Level A）

**要件**: 文字キーのみのショートカットがある場合、無効化、再割り当て、またはフォーカス時のみ有効にできる。

## ガイドライン2.2: 十分な時間

### 達成基準2.2.1 タイミング調整可能（Level A）

**要件**: 時間制限がある場合、ユーザーが調整、延長、または無効化できる。

#### 実装例: セッションタイムアウトの警告

```tsx
'use client'

import { useState, useEffect } from 'react'

export function SessionTimeout() {
  const [timeLeft, setTimeLeft] = useState(300) // 5分
  const [showWarning, setShowWarning] = useState(false)

  useEffect(() => {
    const timer = setInterval(() => {
      setTimeLeft((prev) => {
        if (prev <= 60 && !showWarning) {
          setShowWarning(true)
        }
        return prev - 1
      })
    }, 1000)

    return () => clearInterval(timer)
  }, [showWarning])

  const extendSession = () => {
    setTimeLeft(300)
    setShowWarning(false)
  }

  if (!showWarning) return null

  return (
    <div
      role="alert"
      aria-live="assertive"
      className="fixed top-4 right-4 bg-yellow-100 border border-yellow-400 p-4 rounded"
    >
      <p className="font-bold">セッションがまもなく終了します</p>
      <p>残り時間: {timeLeft}秒</p>
      <button
        onClick={extendSession}
        className="mt-2 px-4 py-2 bg-blue-600 text-white rounded"
      >
        セッションを延長
      </button>
    </div>
  )
}
```

### 達成基準2.2.2 一時停止、停止、非表示（Level A）

**要件**: 自動的に動くコンテンツを一時停止、停止、または非表示にできる。

#### 実装例: カルーセル

```tsx
'use client'

import { useState, useEffect } from 'react'

export function Carousel({ items }: { items: string[] }) {
  const [currentIndex, setCurrentIndex] = useState(0)
  const [isPaused, setIsPaused] = useState(false)

  useEffect(() => {
    if (isPaused) return

    const interval = setInterval(() => {
      setCurrentIndex((prev) => (prev + 1) % items.length)
    }, 5000)

    return () => clearInterval(interval)
  }, [isPaused, items.length])

  return (
    <div className="relative">
      <div className="overflow-hidden">
        <div className="flex transition-transform duration-500">
          {items.map((item, index) => (
            <div
              key={index}
              className={`w-full flex-shrink-0 ${
                index === currentIndex ? 'block' : 'hidden'
              }`}
            >
              {item}
            </div>
          ))}
        </div>
      </div>

      <div className="flex gap-2 mt-4">
        <button
          onClick={() => setIsPaused(!isPaused)}
          aria-label={isPaused ? '再生' : '一時停止'}
          className="px-4 py-2 bg-gray-200 rounded"
        >
          {isPaused ? '▶' : '⏸'}
        </button>
        <button
          onClick={() => setCurrentIndex((prev) => (prev - 1 + items.length) % items.length)}
          aria-label="前へ"
          className="px-4 py-2 bg-gray-200 rounded"
        >
          ←
        </button>
        <button
          onClick={() => setCurrentIndex((prev) => (prev + 1) % items.length)}
          aria-label="次へ"
          className="px-4 py-2 bg-gray-200 rounded"
        >
          →
        </button>
      </div>

      <div className="mt-2 text-sm text-gray-600">
        {currentIndex + 1} / {items.length}
      </div>
    </div>
  )
}
```

## ガイドライン2.3: 発作と身体的反応

### 達成基準2.3.1 3回の閃光、又は閾値以下（Level A）

**要件**: 1秒間に3回以上点滅するコンテンツがない。

#### 実装例: アニメーションの制限

```css
/* ✅ 良い例: 緩やかなアニメーション */
.fade-in {
  animation: fadeIn 1s ease-in;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

/* ユーザーがアニメーション削減を希望している場合 */
@media (prefers-reduced-motion: reduce) {
  .fade-in {
    animation: none;
  }
}

/* ❌ 悪い例: 高速点滅 */
.blink {
  animation: blink 0.2s infinite;
}

@keyframes blink {
  0%, 50% { opacity: 1; }
  51%, 100% { opacity: 0; }
}
```

## ガイドライン2.4: ナビゲーション可能

### 達成基準2.4.1 ブロックスキップ（Level A）

**要件**: 繰り返されるコンテンツブロックをスキップできる。

#### 実装例: スキップリンク

```tsx
// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja">
      <body>
        <a
          href="#main"
          className="sr-only focus:not-sr-only focus:absolute focus:top-4 focus:left-4 focus:z-50 focus:px-4 focus:py-2 focus:bg-blue-600 focus:text-white focus:rounded"
        >
          メインコンテンツへスキップ
        </a>
        <header>
          <nav>{/* ナビゲーション */}</nav>
        </header>
        <main id="main" tabIndex={-1}>
          {children}
        </main>
      </body>
    </html>
  )
}
```

### 達成基準2.4.2 ページタイトル（Level A）

**要件**: ページに内容を説明するタイトルがある。

#### 実装例: Next.jsのメタデータ

```tsx
// app/products/[id]/page.tsx
import { Metadata } from 'next'

export async function generateMetadata({ params }: { params: { id: string } }): Promise<Metadata> {
  const product = await getProduct(params.id)

  return {
    title: `${product.name} - 製品詳細 | ストア名`,
    description: product.description,
  }
}

export default function ProductPage({ params }: { params: { id: string } }) {
  // ページコンテンツ
}
```

### 達成基準2.4.3 フォーカス順序（Level A）

**要件**: フォーカス順序が論理的である。

#### 実装例: 論理的なDOM順序

```tsx
// ✅ 良い例: DOM順序が論理的
<form>
  <div>
    <label htmlFor="name">氏名</label>
    <input id="name" type="text" />
  </div>
  <div>
    <label htmlFor="email">メール</label>
    <input id="email" type="email" />
  </div>
  <div>
    <label htmlFor="phone">電話</label>
    <input id="phone" type="tel" />
  </div>
  <button type="submit">送信</button>
</form>

// ❌ 悪い例: tabIndexで順序を変更
<div>
  <input tabIndex={3} />
  <input tabIndex={1} />
  <input tabIndex={2} />
</div>
```

### 達成基準2.4.4 リンクの目的（Level A）

**要件**: リンクの目的がリンクテキストから判断できる。

#### 実装例: 明確なリンクテキスト

```tsx
// ✅ 良い例: 明確なリンクテキスト
<a href="/products/123">
  青いTシャツの詳細を見る
</a>

// ✅ 良い例: aria-labelで補足
<a
  href="/products/123"
  aria-label="青いTシャツの詳細を見る"
>
  詳細
</a>

// ❌ 悪い例: 不明確なリンクテキスト
<a href="/products/123">
  こちら
</a>
<a href="/products/123">
  詳細
</a>
```

### 達成基準2.4.5 複数の手段（Level AA）

**要件**: ウェブページを見つけるための複数の手段がある。

#### 実装例: 検索とナビゲーション

```tsx
<header>
  <nav aria-label="メインナビゲーション">
    <ul>
      <li><a href="/">ホーム</a></li>
      <li><a href="/products">製品</a></li>
      <li><a href="/about">会社概要</a></li>
    </ul>
  </nav>

  <form role="search">
    <label htmlFor="search">サイト内検索</label>
    <input id="search" type="search" />
    <button type="submit">検索</button>
  </form>
</header>

<footer>
  <nav aria-label="サイトマップ">
    <h2>サイトマップ</h2>
    <ul>
      <li><a href="/products">製品一覧</a></li>
      <li><a href="/contact">お問い合わせ</a></li>
      <li><a href="/sitemap">サイトマップ</a></li>
    </ul>
  </nav>
</footer>
```

### 達成基準2.4.6 見出し及びラベル（Level AA）

**要件**: 見出しとラベルがトピックや目的を説明している。

#### 実装例: 明確な見出しとラベル

```tsx
// ✅ 良い例: 明確な見出し
<article>
  <h1>Web品質・アクセシビリティ完全ガイド 2026</h1>
  <section>
    <h2>WCAG 2.1準拠ガイド</h2>
    <h3>Perceivable（知覚可能）</h3>
    <p>...</p>
  </section>
</article>

// ✅ 良い例: 明確なフォームラベル
<form>
  <label htmlFor="email">
    メールアドレス（ログインに使用）
  </label>
  <input id="email" type="email" />

  <label htmlFor="password">
    パスワード（8文字以上）
  </label>
  <input id="password" type="password" />
</form>
```

### 達成基準2.4.7 フォーカスの可視化（Level AA）

**要件**: フォーカスインジケーターが視覚的に確認できる。

#### 実装例: カスタムフォーカススタイル

```css
/* ✅ 良い例: 視認性の高いフォーカス */
a:focus-visible,
button:focus-visible,
input:focus-visible {
  outline: 3px solid #0066cc;
  outline-offset: 2px;
}

/* Tailwind CSSの場合 */
.focus-visible:outline-none
.focus-visible:ring-2
.focus-visible:ring-blue-600
.focus-visible:ring-offset-2

/* ❌ 悪い例: フォーカスを消す */
button:focus {
  outline: none;
}
```

### 達成基準2.5.1 ポインタジェスチャー（Level A）

**要件**: 複雑なジェスチャに代替手段を提供する。

#### 実装例: スワイプ操作の代替

```tsx
'use client'

import { useState } from 'react'

export function ImageGallery({ images }: { images: string[] }) {
  const [currentIndex, setCurrentIndex] = useState(0)

  // スワイプ操作の実装は省略

  return (
    <div>
      <img src={images[currentIndex]} alt={`画像 ${currentIndex + 1}`} />

      {/* 代替手段: ボタン */}
      <div className="flex gap-2 mt-4">
        <button
          onClick={() => setCurrentIndex((prev) => Math.max(0, prev - 1))}
          disabled={currentIndex === 0}
        >
          前へ
        </button>
        <button
          onClick={() => setCurrentIndex((prev) => Math.min(images.length - 1, prev + 1))}
          disabled={currentIndex === images.length - 1}
        >
          次へ
        </button>
      </div>
    </div>
  )
}
```

### 達成基準2.5.2 ポインタのキャンセル（Level A）

**要件**: ポインタ操作がキャンセル可能である。

#### 実装例: クリック操作のキャンセル

```tsx
'use client'

export function CancelableButton() {
  const handleMouseDown = (e: React.MouseEvent) => {
    // mousedownでは何もしない
  }

  const handleClick = (e: React.MouseEvent) => {
    // clickで実行（mouseupで発火）
    console.log('クリックされました')
  }

  return (
    <button
      onMouseDown={handleMouseDown}
      onClick={handleClick}
      className="px-4 py-2 bg-blue-600 text-white rounded"
    >
      クリック（マウスを離すまでキャンセル可能）
    </button>
  )
}
```

### 達成基準2.5.3 ラベルを含む名前（Level A）

**要件**: 視覚的なラベルとプログラム的な名前が一致する。

#### 実装例: ラベルの一致

```tsx
// ✅ 良い例: ラベルとaria-labelが一致
<button aria-label="検索">
  検索
</button>

// ✅ 良い例: 視覚的なテキストを含む
<button aria-label="サイト内を検索">
  検索
</button>

// ❌ 悪い例: 一致しない
<button aria-label="送信">
  検索
</button>
```

### 達成基準2.5.4 動きによる起動（Level A）

**要件**: デバイスの動きによる機能に代替手段を提供する。

## まとめ

Operable（操作可能）原則は、様々な入力方法でUIを操作できることを求めます。主なポイント：

- 全ての機能がキーボードで操作可能
- フォーカストラップを避ける
- 時間制限に調整機能を提供
- 論理的なフォーカス順序
- 視認可能なフォーカスインジケーター

### チェックリスト

- キーボードだけで全ての機能が操作できる
- モーダルでフォーカストラップが実装されている
- スキップリンクがある
- フォーカスインジケーターが視認できる
- 自動再生コンテンツに停止ボタンがある

## 次のステップ

次章では、WCAG 2.1の第3原則「Understandable（理解可能）」について解説します。
