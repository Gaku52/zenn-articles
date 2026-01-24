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

### フォーカスインジケーター

```css
/* ✅ 視認性の高いフォーカススタイル */
a:focus-visible,
button:focus-visible,
input:focus-visible {
  outline: 3px solid #0066cc;
  outline-offset: 2px;
}

/* Tailwind CSS */
.focus-visible:outline-none
.focus-visible:ring-2
.focus-visible:ring-blue-600
.focus-visible:ring-offset-2
```

### tabIndex の使用

- **0**: フォーカス可能、自然な順序
- **-1**: プログラム的にフォーカス可能、Tab では到達不可
- **1以上**: 使用しない（順序が複雑になる）

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
