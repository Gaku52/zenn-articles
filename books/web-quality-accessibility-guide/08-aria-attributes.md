# ARIA属性 - スクリーンリーダー対応の強化

## ARIAとは

ARIA（Accessible Rich Internet Applications）は、HTMLだけでは表現できないリッチなUIコンポーネントのアクセシビリティを向上させるための仕様です。

WAI-ARIA 1.2が最新版で、W3Cが策定しています。

### ARIAの目的

- HTMLの意味を補完・拡張する
- 動的なコンテンツ変更をスクリーンリーダーに伝える
- カスタムコンポーネントに意味を付与する

## ARIAの5つのルール

### ルール1: 可能な限りネイティブHTML要素を使用する

```tsx
// ✅ 良い例: ネイティブbutton
<button onClick={handleClick}>クリック</button>

// ❌ 悪い例: 不要なARIA
<div role="button" onClick={handleClick}>クリック</div>
```

### ルール2: ネイティブ要素の意味を変更しない

```tsx
// ❌ 悪い例: buttonの意味を変更
<button role="heading">見出し</button>

// ✅ 良い例: 適切な要素を使用
<h2>
  <button onClick={toggleSection}>見出し</button>
</h2>
```

### ルール3: 全てのインタラクティブ要素はキーボード操作可能にする

```tsx
// ✅ 良い例: キーボード操作をサポート
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
```

### ルール4: role="presentation"またはaria-hidden="true"を持つ要素にフォーカスを当てない

```tsx
// ✅ 良い例: 装飾的な要素
<div aria-hidden="true">★★★</div>

// ❌ 悪い例: hiddenだがフォーカス可能
<button aria-hidden="true">クリック</button>
```

### ルール5: 全てのインタラクティブ要素にアクセシブルな名前を付ける

```tsx
// ✅ 良い例: aria-labelで名前を提供
<button aria-label="メニューを開く">
  <MenuIcon />
</button>
```

## ARIAの種類

ARIAは大きく3つに分類されます：

1. **role**: 要素の役割を定義
2. **property**: 要素の性質を定義（aria-label、aria-labelledby等）
3. **state**: 要素の状態を定義（aria-expanded、aria-checked等）

## 主要なrole属性

### ランドマークrole

HTMLのセマンティック要素がある場合は、そちらを優先します。

```tsx
// ✅ ネイティブ要素（推奨）
<header>
<nav>
<main>
<aside>
<footer>

// 以下は、ネイティブ要素が使えない場合のみ
<div role="banner"> {/* header相当 */}
<div role="navigation"> {/* nav相当 */}
<div role="main"> {/* main相当 */}
<div role="complementary"> {/* aside相当 */}
<div role="contentinfo"> {/* footer相当 */}
```

### ウィジェットrole

カスタムコンポーネントに意味を付与します。

#### button

```tsx
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
```

#### checkbox

```tsx
'use client'

import { useState } from 'react'

export function CustomCheckbox({ label }: { label: string }) {
  const [checked, setChecked] = useState(false)

  return (
    <div
      role="checkbox"
      aria-checked={checked}
      tabIndex={0}
      onClick={() => setChecked(!checked)}
      onKeyDown={(e) => {
        if (e.key === ' ' || e.key === 'Enter') {
          e.preventDefault()
          setChecked(!checked)
        }
      }}
      className={`inline-flex items-center gap-2 cursor-pointer`}
    >
      <span className={`w-5 h-5 border-2 flex items-center justify-center ${
        checked ? 'bg-blue-600 border-blue-600' : 'border-gray-300'
      }`}>
        {checked && <span className="text-white">✓</span>}
      </span>
      {label}
    </div>
  )
}
```

#### dialog

```tsx
'use client'

import { useEffect, useRef } from 'react'

interface DialogProps {
  isOpen: boolean
  onClose: () => void
  title: string
  children: React.ReactNode
}

export function Dialog({ isOpen, onClose, title, children }: DialogProps) {
  const dialogRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    if (isOpen) {
      dialogRef.current?.focus()
    }
  }, [isOpen])

  if (!isOpen) return null

  return (
    <div
      className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center"
      onClick={onClose}
    >
      <div
        ref={dialogRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby="dialog-title"
        tabIndex={-1}
        className="bg-white p-6 rounded-lg"
        onClick={(e) => e.stopPropagation()}
      >
        <h2 id="dialog-title" className="text-xl font-bold mb-4">
          {title}
        </h2>
        {children}
        <button onClick={onClose} className="mt-4 px-4 py-2 bg-gray-200 rounded">
          閉じる
        </button>
      </div>
    </div>
  )
}
```

#### tab、tablist、tabpanel

```tsx
'use client'

import { useState } from 'react'

interface Tab {
  id: string
  label: string
  content: React.ReactNode
}

export function Tabs({ tabs }: { tabs: Tab[] }) {
  const [activeTab, setActiveTab] = useState(0)

  return (
    <div>
      <div role="tablist" className="flex border-b">
        {tabs.map((tab, index) => (
          <button
            key={tab.id}
            role="tab"
            aria-selected={activeTab === index}
            aria-controls={`panel-${tab.id}`}
            id={`tab-${tab.id}`}
            tabIndex={activeTab === index ? 0 : -1}
            onClick={() => setActiveTab(index)}
            onKeyDown={(e) => {
              if (e.key === 'ArrowRight') {
                setActiveTab((activeTab + 1) % tabs.length)
              } else if (e.key === 'ArrowLeft') {
                setActiveTab((activeTab - 1 + tabs.length) % tabs.length)
              }
            }}
            className={`px-4 py-2 ${
              activeTab === index
                ? 'border-b-2 border-blue-600 text-blue-600'
                : 'text-gray-600'
            }`}
          >
            {tab.label}
          </button>
        ))}
      </div>

      {tabs.map((tab, index) => (
        <div
          key={tab.id}
          role="tabpanel"
          id={`panel-${tab.id}`}
          aria-labelledby={`tab-${tab.id}`}
          hidden={activeTab !== index}
          className="p-4"
        >
          {tab.content}
        </div>
      ))}
    </div>
  )
}
```

## 主要なARIAプロパティ

### aria-label

要素にラベルを提供します。視覚的なテキストがない場合に使用します。

```tsx
// ✅ アイコンボタン
<button aria-label="検索">
  <SearchIcon />
</button>

// ✅ ナビゲーションの区別
<nav aria-label="メインナビゲーション">
  {/* リンク */}
</nav>
<nav aria-label="フッターナビゲーション">
  {/* リンク */}
</nav>

// ❌ 不要なaria-label
<button aria-label="送信">送信</button>
{/* テキストがあるので不要 */}
```

### aria-labelledby

既存の要素を参照してラベルを提供します。

```tsx
// ✅ ダイアログのタイトル
<div role="dialog" aria-labelledby="dialog-title">
  <h2 id="dialog-title">確認</h2>
  <p>本当に削除しますか?</p>
</div>

// ✅ フォームセクション
<section aria-labelledby="personal-info">
  <h2 id="personal-info">個人情報</h2>
  <form>{/* フォーム要素 */}</form>
</section>

// ✅ 複数要素の組み合わせ
<div role="region" aria-labelledby="title subtitle">
  <h2 id="title">記事タイトル</h2>
  <p id="subtitle">サブタイトル</p>
  <p>本文...</p>
</div>
```

### aria-describedby

要素の詳細な説明を提供します。

```tsx
// ✅ フォームのヘルプテキスト
<div>
  <label htmlFor="password">パスワード</label>
  <input
    id="password"
    type="password"
    aria-describedby="password-help"
  />
  <p id="password-help" className="text-sm text-gray-600">
    8文字以上、大文字・小文字・数字を含む必要があります
  </p>
</div>

// ✅ エラーメッセージ
<div>
  <label htmlFor="email">メールアドレス</label>
  <input
    id="email"
    type="email"
    aria-invalid="true"
    aria-describedby="email-error"
  />
  <p id="email-error" className="text-red-600">
    メールアドレスの形式が正しくありません
  </p>
</div>
```

### aria-hidden

スクリーンリーダーから要素を隠します。装飾的な要素に使用します。

```tsx
// ✅ 装飾的なアイコン
<div>
  <span aria-hidden="true">★</span>
  <span className="sr-only">評価: 5つ星</span>
</div>

// ✅ 重複した情報を隠す
<button>
  削除
  <TrashIcon aria-hidden="true" />
</button>

// ❌ 悪い例: 重要な情報を隠す
<div aria-hidden="true">
  重要なお知らせ
</div>
```

## 主要なARIAステート

### aria-expanded

展開可能な要素の状態を示します。

```tsx
'use client'

import { useState } from 'react'

export function Accordion({ title, content }: { title: string, content: string }) {
  const [isExpanded, setIsExpanded] = useState(false)

  return (
    <div>
      <h3>
        <button
          aria-expanded={isExpanded}
          aria-controls="panel"
          onClick={() => setIsExpanded(!isExpanded)}
          className="w-full text-left px-4 py-2 bg-gray-100"
        >
          {title}
          <span aria-hidden="true">{isExpanded ? '▲' : '▼'}</span>
        </button>
      </h3>
      <div
        id="panel"
        hidden={!isExpanded}
        className="p-4"
      >
        {content}
      </div>
    </div>
  )
}
```

### aria-pressed

トグルボタンの状態を示します。

```tsx
'use client'

import { useState } from 'react'

export function ToggleButton() {
  const [isPressed, setIsPressed] = useState(false)

  return (
    <button
      aria-pressed={isPressed}
      onClick={() => setIsPressed(!isPressed)}
      className={`px-4 py-2 rounded ${
        isPressed ? 'bg-blue-600 text-white' : 'bg-gray-200'
      }`}
    >
      {isPressed ? 'オン' : 'オフ'}
    </button>
  )
}
```

### aria-checked

チェックボックスやラジオボタンの状態を示します。

```tsx
// ネイティブ要素は自動的にaria-checkedを持つ
<input type="checkbox" checked />

// カスタム要素の場合
<div
  role="checkbox"
  aria-checked={checked}
  tabIndex={0}
  onClick={() => setChecked(!checked)}
>
  {checked ? '☑' : '☐'} チェックボックス
</div>
```

### aria-selected

選択可能な要素（タブ、リストアイテム等）の状態を示します。

```tsx
// タブの例（前述のTabsコンポーネント参照）
<button
  role="tab"
  aria-selected={activeTab === index}
>
  タブ{index + 1}
</button>

// リストボックス
<div role="listbox">
  {items.map((item, index) => (
    <div
      key={item.id}
      role="option"
      aria-selected={selectedIndex === index}
      onClick={() => setSelectedIndex(index)}
    >
      {item.label}
    </div>
  ))}
</div>
```

## ライブリージョン（aria-live）

動的に変化するコンテンツをスクリーンリーダーに通知します。

### aria-live="polite"

ユーザーの操作を中断せずに通知します。ほとんどの場合、こちらを使用します。

```tsx
// ✅ ステータスメッセージ
<div
  role="status"
  aria-live="polite"
  aria-atomic="true"
>
  {statusMessage}
</div>

// 使用例
const [status, setStatus] = useState('')

const handleSave = async () => {
  setStatus('保存中...')
  await saveData()
  setStatus('保存しました')
}
```

### aria-live="assertive"

即座に通知します。エラーメッセージなど、重要な情報のみに使用します。

```tsx
// ✅ エラーメッセージ
<div
  role="alert"
  aria-live="assertive"
  className="text-red-600"
>
  {errorMessage}
</div>

// 使用例
const [error, setError] = useState('')

const handleSubmit = async () => {
  try {
    await submitForm()
  } catch (err) {
    setError('送信に失敗しました。もう一度お試しください。')
  }
}
```

### aria-atomic

ライブリージョン全体を読み上げるか、変更部分のみ読み上げるかを指定します。

```tsx
// ✅ 全体を読み上げ
<div
  role="status"
  aria-live="polite"
  aria-atomic="true"
>
  {count}件の新着メッセージがあります
</div>

// ✅ 変更部分のみ読み上げ
<div
  aria-live="polite"
  aria-atomic="false"
>
  <p>合計: <span>{total}</span>円</p>
</div>
```

## ARIAの実践例

### 例1: アクセシブルなモーダル

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
  const previousActiveElement = useRef<HTMLElement | null>(null)

  useEffect(() => {
    if (isOpen) {
      previousActiveElement.current = document.activeElement as HTMLElement
      modalRef.current?.focus()

      const handleKeyDown = (e: KeyboardEvent) => {
        if (e.key === 'Escape') {
          onClose()
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
    } else {
      previousActiveElement.current?.focus()
    }
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
        tabIndex={-1}
        className="bg-white p-6 rounded-lg max-w-md w-full"
        onClick={(e) => e.stopPropagation()}
      >
        <h2 id="modal-title" className="text-2xl font-bold mb-4">
          {title}
        </h2>
        {children}
        <button
          onClick={onClose}
          className="mt-4 px-4 py-2 bg-gray-200 rounded"
        >
          閉じる
        </button>
      </div>
    </div>,
    document.body
  )
}
```

### 例2: アクセシブルなドロップダウンメニュー

```tsx
'use client'

import { useState, useRef, useEffect } from 'react'

export function Dropdown({ label, items }: { label: string, items: string[] }) {
  const [isOpen, setIsOpen] = useState(false)
  const [focusedIndex, setFocusedIndex] = useState(0)
  const buttonRef = useRef<HTMLButtonElement>(null)

  useEffect(() => {
    if (!isOpen) {
      setFocusedIndex(0)
    }
  }, [isOpen])

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
        buttonRef.current?.focus()
        break
    }
  }

  return (
    <div className="relative">
      <button
        ref={buttonRef}
        onClick={() => setIsOpen(!isOpen)}
        onKeyDown={handleKeyDown}
        aria-haspopup="true"
        aria-expanded={isOpen}
        className="px-4 py-2 bg-white border rounded"
      >
        {label}
      </button>

      {isOpen && (
        <div
          role="menu"
          className="absolute mt-2 bg-white border rounded shadow-lg"
        >
          {items.map((item, index) => (
            <button
              key={item}
              role="menuitem"
              tabIndex={-1}
              className={`block w-full text-left px-4 py-2 hover:bg-gray-100 ${
                index === focusedIndex ? 'bg-blue-100' : ''
              }`}
              onClick={() => {
                console.log(item)
                setIsOpen(false)
                buttonRef.current?.focus()
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

## まとめ

ARIA属性は、HTMLだけでは表現できないリッチなUIのアクセシビリティを向上させます。主なポイント：

- 可能な限りネイティブHTML要素を使用
- ARIAの5つのルールを守る
- role、property、stateを適切に使用
- aria-liveで動的変更を通知
- カスタムコンポーネントにキーボード操作を実装

### チェックリスト

- ネイティブHTML要素を優先している
- カスタムコンポーネントに適切なroleがある
- インタラクティブ要素がキーボード操作可能
- ステート（aria-expanded等）が正しく更新される
- ライブリージョンで動的変更を通知している

## 次のステップ

次章では、キーボードナビゲーションの実装について詳しく解説します。
