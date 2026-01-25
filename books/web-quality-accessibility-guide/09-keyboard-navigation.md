# キーボードナビゲーション - 全ての機能を操作可能にする

## キーボードナビゲーションの重要性

マウスやタッチスクリーンを使用できないユーザーにとって、キーボード操作は必須です。WCAG 2.1では、全ての機能がキーボードで操作可能であることが要求されています（達成基準2.1.1、Level A）。

### キーボードのみを使用するユーザー

- 運動障害のあるユーザー
- スクリーンリーダーユーザー
- キーボードショートカットを好むパワーユーザー
- マウスが故障した場合の一時的なユーザー

## 基本的なキーボード操作

### Tab キー

- フォーカスを次の要素に移動
- フォーカス可能な要素: button、a、input、select、textarea、tabindex="0"

### Shift + Tab

- フォーカスを前の要素に移動

### Enter キー

- リンクやボタンをアクティベート
- フォームを送信

### Space キー

- ボタンをアクティベート
- チェックボックスをトグル
- スクロール（フォーカスがない場合）

### Arrow キー

- ラジオボタン、タブ、リストボックス等のナビゲーション
- スクロール

### Escape キー

- モーダルやドロップダウンを閉じる

## 実装パターン

### パターン1: カスタムボタン

```tsx
function CustomButton({ onClick, children }: { onClick: () => void, children: React.ReactNode }) {
  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault()
      onClick()
    }
  }

  return (
    <div
      role="button"
      tabIndex={0}
      onClick={onClick}
      onKeyDown={handleKeyDown}
      className="px-4 py-2 bg-blue-600 text-white rounded cursor-pointer"
    >
      {children}
    </div>
  )
}
```

### パターン2: フォーカストラップ（モーダル）

前章のModal コンポーネント参照。

### パターン3: Arrow キーナビゲーション

```tsx
'use client'

import { useState } from 'react'

export function RadioGroup({ options }: { options: string[] }) {
  const [selected, setSelected] = useState(0)

  return (
    <div role="radiogroup" aria-label="オプション選択">
      {options.map((option, index) => (
        <label key={option} className="flex items-center gap-2">
          <input
            type="radio"
            name="option"
            checked={selected === index}
            onChange={() => setSelected(index)}
            onKeyDown={(e) => {
              if (e.key === 'ArrowDown' || e.key === 'ArrowRight') {
                e.preventDefault()
                setSelected((selected + 1) % options.length)
              } else if (e.key === 'ArrowUp' || e.key === 'ArrowLeft') {
                e.preventDefault()
                setSelected((selected - 1 + options.length) % options.length)
              }
            }}
          />
          {option}
        </label>
      ))}
    </div>
  )
}
```

### パターン4: スキップリンク

```tsx
// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja">
      <body>
        <a
          href="#main"
          className="sr-only focus:not-sr-only focus:absolute focus:top-4 focus:left-4 focus:z-50 focus:px-4 focus:py-2 focus:bg-blue-600 focus:text-white"
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

## フォーカス管理

フォーカス管理は、キーボードナビゲーションの核心です。適切なフォーカス順序、視認性の高いフォーカスインジケーター、動的なフォーカス移動が重要です。

### フォーカス順序の原則

WCAG 2.1 達成基準2.4.3（Level A）では、意味のある順序でフォーカスが移動することが要求されています[^wcag-focus-order]。

[^wcag-focus-order]: [Understanding Success Criterion 2.4.3: Focus Order - W3C](https://www.w3.org/WAI/WCAG21/Understanding/focus-order.html)

**フォーカス順序の決定要因**:
1. DOM順序（HTMLの記述順）
2. tabindexの値（可能な限り避ける）
3. CSSのpositionやfloat（視覚的順序と論理的順序の乖離に注意）

```html
<!-- ✅ 良い例: DOM順序が論理的 -->
<form>
  <label for="name">名前</label>
  <input id="name" type="text" />

  <label for="email">メールアドレス</label>
  <input id="email" type="email" />

  <button type="submit">送信</button>
</form>

<!-- ❌ 悪い例: tabindexで順序を制御 -->
<form>
  <input tabindex="2" id="email" type="email" />
  <input tabindex="1" id="name" type="text" />
  <button tabindex="3" type="submit">送信</button>
</form>
```

### フォーカスインジケーター

WCAG 2.1 達成基準2.4.7（Level AA）では、視認性の高いフォーカスインジケーターが要求されています[^wcag-focus-visible]。

[^wcag-focus-visible]: [Understanding Success Criterion 2.4.7: Focus Visible - W3C](https://www.w3.org/WAI/WCAG21/Understanding/focus-visible.html)

```css
/* ✅ 視認性の高いフォーカススタイル */
a:focus-visible,
button:focus-visible,
input:focus-visible {
  outline: 3px solid #0066cc;
  outline-offset: 2px;
}

/* ✅ ダークモード対応 */
@media (prefers-color-scheme: dark) {
  a:focus-visible,
  button:focus-visible,
  input:focus-visible {
    outline-color: #4da6ff;
  }
}

/* Tailwind CSS */
.focus-visible:outline-none
.focus-visible:ring-2
.focus-visible:ring-blue-600
.focus-visible:ring-offset-2
```

**フォーカスインジケーターのベストプラクティス**:
- コントラスト比3:1以上（WCAG 2.1 達成基準1.4.11）
- outline-offset を使用して視認性を向上
- :focus-visible を使用（マウスクリック時は表示しない）
- outline: none は単独で使用しない（代替スタイルを提供）

### プログラム的なフォーカス移動

動的なコンテンツ変更時に、適切にフォーカスを移動することが重要です。

```tsx
'use client'

import { useRef, useEffect } from 'react'

export function SearchResults({ query, results }: { query: string, results: any[] }) {
  const headingRef = useRef<HTMLHeadingElement>(null)

  useEffect(() => {
    // 検索結果が表示されたら、見出しにフォーカス
    if (results.length > 0 && headingRef.current) {
      headingRef.current.focus()
    }
  }, [results])

  return (
    <section>
      <h2 ref={headingRef} tabIndex={-1}>
        「{query}」の検索結果: {results.length}件
      </h2>
      <ul>
        {results.map((result) => (
          <li key={result.id}>{result.title}</li>
        ))}
      </ul>
    </section>
  )
}
```

### tabIndex の使用

tabindexは、フォーカス可能性と順序を制御する属性です。

| 値 | 意味 | 使用例 |
|---|-----|-------|
| **0** | フォーカス可能、自然な順序 | カスタムボタン、ダイアログの見出し |
| **-1** | プログラム的にフォーカス可能、Tabでは到達不可 | 検索結果の見出し、エラーメッセージ |
| **1以上** | 使用しない（順序が複雑になる） | ❌ 非推奨 |

```tsx
// ✅ 良い例: tabIndex=0でフォーカス可能に
<div role="button" tabIndex={0} onClick={handleClick}>
  カスタムボタン
</div>

// ✅ 良い例: tabIndex=-1でプログラム的にフォーカス
<h2 ref={headingRef} tabIndex={-1}>
  エラーが発生しました
</h2>

// ❌ 悪い例: tabIndex=1以上は使用しない
<input tabIndex={2} />
```

## ショートカットキーの実装

WCAG 2.1 達成基準2.1.4（Level A）では、文字キーのショートカットは無効化または再割り当てが可能である必要があります[^wcag-character-key]。

[^wcag-character-key]: [Understanding Success Criterion 2.1.4: Character Key Shortcuts - W3C](https://www.w3.org/WAI/WCAG21/Understanding/character-key-shortcuts.html)

### 推奨されるショートカットキー

単一の文字キーではなく、修飾キー（Ctrl、Alt、Meta）との組み合わせを使用します。

```tsx
'use client'

import { useEffect } from 'react'

export function KeyboardShortcuts() {
  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      // Ctrl/Cmd + K: 検索を開く
      if ((e.ctrlKey || e.metaKey) && e.key === 'k') {
        e.preventDefault()
        // 検索モーダルを開く処理
        console.log('検索を開く')
      }

      // Ctrl/Cmd + /: ヘルプを開く
      if ((e.ctrlKey || e.metaKey) && e.key === '/') {
        e.preventDefault()
        // ヘルプモーダルを開く処理
        console.log('ヘルプを開く')
      }

      // Escape: モーダルを閉じる
      if (e.key === 'Escape') {
        // モーダルを閉じる処理
        console.log('モーダルを閉じる')
      }
    }

    window.addEventListener('keydown', handleKeyDown)
    return () => window.removeEventListener('keydown', handleKeyDown)
  }, [])

  return null
}
```

### ショートカットキーの表示

ユーザーに利用可能なショートカットキーを表示します。

```tsx
export function ShortcutHelp() {
  return (
    <div role="dialog" aria-labelledby="shortcut-title">
      <h2 id="shortcut-title">キーボードショートカット</h2>
      <dl>
        <dt>
          <kbd>Ctrl</kbd> + <kbd>K</kbd>
        </dt>
        <dd>検索を開く</dd>

        <dt>
          <kbd>Ctrl</kbd> + <kbd>/</kbd>
        </dt>
        <dd>ヘルプを開く</dd>

        <dt>
          <kbd>Escape</kbd>
        </dt>
        <dd>モーダルを閉じる</dd>

        <dt>
          <kbd>Tab</kbd>
        </dt>
        <dd>次の要素へ移動</dd>

        <dt>
          <kbd>Shift</kbd> + <kbd>Tab</kbd>
        </dt>
        <dd>前の要素へ移動</dd>
      </dl>
    </div>
  )
}
```

### kbdタグのスタイリング

```css
kbd {
  display: inline-block;
  padding: 2px 6px;
  font-family: 'SF Mono', 'Monaco', 'Consolas', monospace;
  font-size: 0.875rem;
  color: #333;
  background-color: #f5f5f5;
  border: 1px solid #ccc;
  border-radius: 4px;
  box-shadow: 0 1px 0 rgba(0, 0, 0, 0.2);
}

@media (prefers-color-scheme: dark) {
  kbd {
    color: #e0e0e0;
    background-color: #2a2a2a;
    border-color: #555;
  }
}
```

## 複雑なウィジェットのキーボード操作

WAI-ARIA Authoring Practices Guide（APG）では、複雑なウィジェットのキーボード操作パターンが定義されています[^aria-patterns]。

[^aria-patterns]: [WAI-ARIA Authoring Practices Guide - W3C](https://www.w3.org/WAI/ARIA/apg/patterns/)

### タブコンポーネント

```tsx
'use client'

import { useState, useRef } from 'react'

const tabs = ['プロフィール', '設定', '通知']

export function Tabs() {
  const [selectedIndex, setSelectedIndex] = useState(0)
  const tabRefs = useRef<(HTMLButtonElement | null)[]>([])

  const handleKeyDown = (e: React.KeyboardEvent, index: number) => {
    let newIndex = index

    if (e.key === 'ArrowRight') {
      newIndex = (index + 1) % tabs.length
    } else if (e.key === 'ArrowLeft') {
      newIndex = (index - 1 + tabs.length) % tabs.length
    } else if (e.key === 'Home') {
      newIndex = 0
    } else if (e.key === 'End') {
      newIndex = tabs.length - 1
    } else {
      return
    }

    e.preventDefault()
    setSelectedIndex(newIndex)
    tabRefs.current[newIndex]?.focus()
  }

  return (
    <div>
      <div role="tablist" aria-label="アカウント設定">
        {tabs.map((tab, index) => (
          <button
            key={tab}
            ref={(el) => (tabRefs.current[index] = el)}
            role="tab"
            aria-selected={selectedIndex === index}
            aria-controls={`panel-${index}`}
            id={`tab-${index}`}
            tabIndex={selectedIndex === index ? 0 : -1}
            onClick={() => setSelectedIndex(index)}
            onKeyDown={(e) => handleKeyDown(e, index)}
          >
            {tab}
          </button>
        ))}
      </div>

      {tabs.map((tab, index) => (
        <div
          key={tab}
          role="tabpanel"
          id={`panel-${index}`}
          aria-labelledby={`tab-${index}`}
          hidden={selectedIndex !== index}
          tabIndex={0}
        >
          {tab}のコンテンツ
        </div>
      ))}
    </div>
  )
}
```

**タブのキーボード操作**:
- **ArrowRight**: 次のタブに移動
- **ArrowLeft**: 前のタブに移動
- **Home**: 最初のタブに移動
- **End**: 最後のタブに移動

## まとめ

キーボードナビゲーションは、アクセシビリティの基本です。全ての機能がキーボードで操作可能であることを確認してください。

### チェックリスト

- Tabキーで全要素に到達できる
- Enter/Spaceで全ボタンが動作する
- モーダルにフォーカストラップがある
- Escapeでモーダルが閉じる
- フォーカスインジケーターが視認できる

## 次のステップ

次章では、スクリーンリーダー対応について解説します。
