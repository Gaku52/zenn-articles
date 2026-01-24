# Robust（堅牢） - 様々な技術で解釈できるようにする

## Robustとは

Robust（堅牢）は、WCAG 2.1の第4原則です。コンテンツが、支援技術を含む様々なユーザーエージェントによって確実に解釈できることを求めます。

特に、正しいHTMLマークアップ、ARIA属性の適切な使用が重要です。

## ガイドライン4.1: 互換性

### 達成基準4.1.1 構文解析（Level A）

**要件**: マークアップ言語で実装されたコンテンツにおいて、要素には完全な開始タグ及び終了タグがあり、仕様に準じて入れ子になっており、重複した属性がない。

**注**: この基準はWCAG 2.2で廃止されました（HTML5の普及により、ブラウザが自動修正するため）。ただし、正しいHTMLマークアップは引き続き重要です。

#### 実装例: 正しいHTMLマークアップ

```tsx
// ✅ 良い例: 正しいマークアップ
<div>
  <p>段落1</p>
  <p>段落2</p>
</div>

<ul>
  <li>アイテム1</li>
  <li>アイテム2</li>
</ul>

// ❌ 悪い例: 不正なマークアップ
<div>
  <p>段落1
  <p>段落2</p>
</div>

<ul>
  <div>アイテム1</div> <!-- ul内にdivは不適切 -->
</ul>

// ❌ 悪い例: 重複した属性
<div id="main" id="content">
  {/* 重複したid */}
</div>
```

#### 実装例: バリデーションツールの使用

```bash
# HTMLバリデーション（開発時）
pnpm add -D html-validate

# .htmlvalidate.json
{
  "extends": ["html-validate:recommended"],
  "rules": {
    "no-duplicate-id": "error",
    "valid-id": "error",
    "element-permitted-content": "error"
  }
}
```

### 達成基準4.1.2 名前（name）、役割（role）及び値（value）（Level A）

**要件**: 全てのユーザーインターフェースコンポーネントについて、名前及び役割は、プログラムによる解釈が可能である。

#### 実装例: セマンティックHTML

```tsx
// ✅ 良い例: ネイティブ要素（自動的に名前と役割がある）
<button type="button">クリック</button>
<input type="checkbox" id="agree" />
<label htmlFor="agree">利用規約に同意する</label>

// ✅ 良い例: カスタム要素にroleとaria属性
<div
  role="button"
  tabIndex={0}
  aria-pressed={isPressed}
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

// ❌ 悪い例: roleや名前がない
<div onClick={handleClick}>
  クリック
</div>
```

#### 実装例: ARIAステート

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

#### 実装例: 複雑なウィジェット

```tsx
'use client'

import { useState } from 'react'

export function Accordion() {
  const [expandedIndex, setExpandedIndex] = useState<number | null>(null)

  const items = [
    { title: 'セクション1', content: 'コンテンツ1' },
    { title: 'セクション2', content: 'コンテンツ2' },
    { title: 'セクション3', content: 'コンテンツ3' },
  ]

  return (
    <div>
      {items.map((item, index) => (
        <div key={index}>
          <h3>
            <button
              aria-expanded={expandedIndex === index}
              aria-controls={`panel-${index}`}
              onClick={() =>
                setExpandedIndex(expandedIndex === index ? null : index)
              }
              className="w-full text-left px-4 py-2 bg-gray-100 hover:bg-gray-200"
            >
              {item.title}
            </button>
          </h3>
          <div
            id={`panel-${index}`}
            role="region"
            aria-labelledby={`button-${index}`}
            hidden={expandedIndex !== index}
            className="px-4 py-2"
          >
            {item.content}
          </div>
        </div>
      ))}
    </div>
  )
}
```

### 達成基準4.1.3 ステータスメッセージ（Level AA）

**要件**: ステータスメッセージがプログラム的に決定でき、フォーカスを受け取らずに支援技術に伝えられる。

#### 実装例: aria-liveリージョン

```tsx
'use client'

import { useState } from 'react'

export function SaveButton() {
  const [status, setStatus] = useState<'idle' | 'saving' | 'saved' | 'error'>('idle')

  const handleSave = async () => {
    setStatus('saving')
    try {
      await saveData()
      setStatus('saved')
      setTimeout(() => setStatus('idle'), 3000)
    } catch (error) {
      setStatus('error')
    }
  }

  return (
    <div>
      <button
        onClick={handleSave}
        disabled={status === 'saving'}
        className="px-4 py-2 bg-blue-600 text-white rounded disabled:opacity-50"
      >
        {status === 'saving' ? '保存中...' : '保存'}
      </button>

      {/* ステータスメッセージ */}
      <div
        role="status"
        aria-live="polite"
        aria-atomic="true"
        className="mt-2"
      >
        {status === 'saved' && (
          <span className="text-green-600">✓ 保存しました</span>
        )}
        {status === 'error' && (
          <span className="text-red-600">エラーが発生しました</span>
        )}
      </div>
    </div>
  )
}

async function saveData() {
  return new Promise((resolve) => setTimeout(resolve, 1000))
}
```

#### 実装例: トースト通知

```tsx
'use client'

import { createContext, useContext, useState, useCallback } from 'react'

type Toast = {
  id: number
  message: string
  type: 'success' | 'error' | 'info'
}

type ToastContextType = {
  addToast: (message: string, type: Toast['type']) => void
}

const ToastContext = createContext<ToastContextType | null>(null)

export function ToastProvider({ children }: { children: React.ReactNode }) {
  const [toasts, setToasts] = useState<Toast[]>([])

  const addToast = useCallback((message: string, type: Toast['type']) => {
    const id = Date.now()
    setToasts((prev) => [...prev, { id, message, type }])
    setTimeout(() => {
      setToasts((prev) => prev.filter((t) => t.id !== id))
    }, 5000)
  }, [])

  return (
    <ToastContext.Provider value={{ addToast }}>
      {children}
      <div
        aria-live="polite"
        aria-atomic="false"
        className="fixed top-4 right-4 z-50 space-y-2"
      >
        {toasts.map((toast) => (
          <div
            key={toast.id}
            role="status"
            className={`px-4 py-2 rounded shadow-lg ${
              toast.type === 'success'
                ? 'bg-green-100 text-green-800'
                : toast.type === 'error'
                ? 'bg-red-100 text-red-800'
                : 'bg-blue-100 text-blue-800'
            }`}
          >
            {toast.message}
          </div>
        ))}
      </div>
    </ToastContext.Provider>
  )
}

export function useToast() {
  const context = useContext(ToastContext)
  if (!context) {
    throw new Error('useToast must be used within ToastProvider')
  }
  return context
}
```

## ARIAの適切な使用

### ARIA使用の5つのルール

1. **可能な限りネイティブHTML要素を使用する**
2. **ネイティブ要素の意味を変更しない**
3. **全てのインタラクティブ要素はキーボード操作可能にする**
4. **role="presentation"またはaria-hidden="true"を持つ要素にフォーカスを当てない**
5. **全てのインタラクティブ要素にアクセシブルな名前を付ける**

#### 実装例: ルール1 - ネイティブHTML優先

```tsx
// ✅ 良い例: ネイティブHTML
<button onClick={handleClick}>クリック</button>
<a href="/page">リンク</a>
<input type="checkbox" />

// ❌ 悪い例: 不要なARIA
<div role="button" onClick={handleClick}>クリック</div>
<span role="link" onClick={navigate}>リンク</span>
<div role="checkbox" onClick={toggleCheck}></div>
```

#### 実装例: ルール2 - ネイティブの意味を変更しない

```tsx
// ❌ 悪い例: ボタンの意味を変更
<button role="heading">
  これは見出しではありません
</button>

// ✅ 良い例: 適切な要素を使用
<h2>
  <button onClick={toggleSection}>
    セクションタイトル
  </button>
</h2>
```

#### 実装例: ルール3 - キーボード操作可能

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

#### 実装例: ルール4 - hidden要素にフォーカスしない

```tsx
// ✅ 良い例: 装飾的な要素はaria-hidden
<div aria-hidden="true" className="decorative-icon">
  ★
</div>

// ❌ 悪い例: hiddenだがフォーカス可能
<button aria-hidden="true" onClick={handleClick}>
  クリック
</button>
```

#### 実装例: ルール5 - アクセシブルな名前

```tsx
// ✅ 良い例: アイコンボタンにラベル
<button aria-label="メニューを開く">
  <MenuIcon />
</button>

// ✅ 良い例: 視覚的なラベルとaria-labelledby
<div role="dialog" aria-labelledby="dialog-title">
  <h2 id="dialog-title">確認</h2>
  {/* コンテンツ */}
</div>
```

## よくあるARIAの間違い

### 間違い1: 不要なaria-label

```tsx
// ❌ 悪い例: すでにテキストがあるのにaria-label
<button aria-label="送信">送信</button>

// ✅ 良い例: テキストだけで十分
<button>送信</button>

// ✅ 良い例: アイコンのみの場合はaria-label必要
<button aria-label="送信">
  <SendIcon />
</button>
```

### 間違い2: role="button"とtype="button"の混同

```tsx
// ❌ 悪い例: divにrole="button"は避けるべき
<div role="button" onClick={handleClick}>
  クリック
</div>

// ✅ 良い例: ネイティブbutton要素を使用
<button type="button" onClick={handleClick}>
  クリック
</button>
```

### 間違い3: aria-hiddenとvisibility/displayの混同

```tsx
// ❌ 悪い例: 視覚的には見えるがスクリーンリーダーで隠れる
<div aria-hidden="true" className="visible">
  重要な情報
</div>

// ✅ 良い例: 装飾的な要素のみhidden
<div aria-hidden="true" className="decorative">
  ★★★
</div>
```

## テストと検証

### 自動テストツール

```bash
# axe-core: アクセシビリティ自動テスト
pnpm add -D @axe-core/react

# ESLintプラグイン
pnpm add -D eslint-plugin-jsx-a11y
```

```tsx
// app/layout.tsx（開発環境のみ）
if (process.env.NODE_ENV !== 'production') {
  import('@axe-core/react').then((axe) => {
    axe.default(React, ReactDOM, 1000)
  })
}
```

```json
// .eslintrc.json
{
  "extends": [
    "next/core-web-vitals",
    "plugin:jsx-a11y/recommended"
  ],
  "rules": {
    "jsx-a11y/alt-text": "error",
    "jsx-a11y/aria-props": "error",
    "jsx-a11y/aria-proptypes": "error",
    "jsx-a11y/aria-unsupported-elements": "error",
    "jsx-a11y/role-has-required-aria-props": "error"
  }
}
```

### 手動テスト

1. **スクリーンリーダーテスト**
   - macOS: VoiceOver (Command + F5)
   - Windows: NVDA（無料）、JAWS（有料）

2. **キーボード操作テスト**
   - Tab: フォーカス移動
   - Shift + Tab: 逆方向のフォーカス移動
   - Enter: アクティベート
   - Space: チェックボックス、ボタン
   - Arrow keys: ラジオボタン、リスト

3. **ブラウザDevTools**
   - Chrome DevTools > Accessibility Tree
   - Firefox DevTools > Accessibility Inspector

## 実践的なチェックリスト

### HTML/ARIA検証

- すべてのインタラクティブ要素にアクセシブルな名前がある
- カスタムコンポーネントに適切なroleがある
- aria-expanded、aria-pressed等のステートが正しく更新される
- aria-liveリージョンでステータス変更を通知している
- aria-hiddenとtabIndex="-1"を適切に使用している

### キーボードアクセシビリティ

- Tabキーで全ての要素に到達できる
- Enter/Spaceで全てのボタンが動作する
- Escapeでモーダルが閉じる
- Arrow keysでナビゲーション（該当する場合）
- フォーカスが視覚的に確認できる

### スクリーンリーダー対応

- 画像にalt属性がある
- フォームにラベルがある
- 見出し構造が論理的
- ランドマークが適切に使用されている
- ライブリージョンで動的変更を通知している

## まとめ

Robust（堅牢）原則は、様々な支援技術で確実に解釈できることを求めます。主なポイント：

- 正しいHTMLマークアップ
- 適切なARIA属性の使用
- 名前、役割、ステートのプログラム的な提供
- ステータスメッセージの適切な通知
- 自動テストと手動テストの組み合わせ

### チェックリスト

- HTML validatorでエラーがない
- ESLintのa11yルールをパスしている
- スクリーンリーダーで全てのコンテンツが読み上げられる
- キーボードだけで操作できる
- axe DevToolsでエラーがない

## 次のステップ

次章では、WCAG 2.1準拠のための包括的なチェックリストを提供します。
