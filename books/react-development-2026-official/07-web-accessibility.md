---
title: "Webアクセシビリティ実践ガイド"
---

# Chapter 7: Webアクセシビリティ実践ガイド

> 誰もが使えるWebアプリケーションを実現する

## この章で学べること

この章では、Webアクセシビリティの実践的な実装方法を学びます。

- ✅ WCAG 2.1準拠の実装方法
- ✅ セマンティックHTMLとARIA属性
- ✅ キーボード操作への対応
- ✅ スクリーンリーダー対応
- ✅ アクセシビリティテストの実践

**前提知識**: Chapter 01-06の内容、HTMLの基礎

**所要時間**: 60-80分

---

## 目次

1. [アクセシビリティとは](#1-アクセシビリティとは)
   - WCAG 2.1の4原則
   - 準拠レベル
   - なぜ重要か
2. [セマンティックHTMLの実践](#2-セマンティックhtmlの実践)
   - 適切なHTML要素の選択
   - ランドマークの使用
3. [ARIA属性の正しい使い方](#3-aria属性の正しい使い方)
   - role属性
   - aria-label / aria-labelledby
   - aria-live（動的コンテンツ）
4. [キーボード操作への対応](#4-キーボード操作への対応)
   - フォーカス管理
   - カスタムコンポーネントの実装
5. [色とコントラスト](#5-色とコントラスト)
   - コントラスト比の基準
   - 色だけに依存しない設計
6. [アクセシビリティテスト](#6-アクセシビリティテスト)
   - 自動テスト（axe, Lighthouse）
   - 手動テスト

---

## 1. アクセシビリティとは

### WCAG 2.1の4原則（POUR）

| 原則 | 説明 | 例 |
|------|------|-----|
| **Perceivable**（知覚可能） | 情報が知覚できる形で提示される | 画像の代替テキスト、動画の字幕 |
| **Operable**（操作可能） | UIが操作可能 | キーボード操作、十分な時間 |
| **Understandable**（理解可能） | 情報とUIが理解できる | 明確なエラーメッセージ、予測可能な動作 |
| **Robust**（堅牢） | 多様な技術で解釈できる | 正しいHTML、支援技術との互換性 |

### 準拠レベル

| レベル | 説明 | 法的要件 |
|--------|------|----------|
| **A** | 最低限 | 多くの国で必須 |
| **AA** | 推奨 | 米国、EU、日本等で必須 |
| **AAA** | 最高 | 一部の公共サービス |

**推奨**: **Level AA準拠**を目指す

### なぜ重要か

**1. 法的要件**
- 米国: Section 508、ADA
- EU: European Accessibility Act
- 日本: 障害者差別解消法

**2. ビジネス価値**
- 全人口の15%以上が何らかの障害を持つ（WHO調査）
- アクセシブルなサイトはSEOにも有利

**3. すべてのユーザーに恩恵**
- 一時的な障害（骨折、眼精疲労）
- 状況的制約（騒がしい環境、明るい屋外）
- 高齢者の利便性向上

---

## 2. セマンティックHTMLの実践

### 適切なHTML要素の選択

```tsx
// ❌ 悪い例: divとspanだけ
<div onClick={handleClick}>クリックしてください</div>
<div>
  <div>記事タイトル</div>
  <div>記事本文...</div>
</div>

// ✅ 良い例: セマンティックHTML
<button onClick={handleClick}>クリックしてください</button>
<article>
  <h2>記事タイトル</h2>
  <p>記事本文...</p>
</article>
```

### ランドマーク（Landmark）の使用

```tsx
export function Layout({ children }: { children: React.ReactNode }) {
  return (
    <div>
      {/* ヘッダー */}
      <header>
        <nav aria-label="メインナビゲーション">
          <a href="/">ホーム</a>
          <a href="/about">会社概要</a>
        </nav>
      </header>

      {/* メインコンテンツ */}
      <main>
        {children}
      </main>

      {/* サイドバー */}
      <aside aria-label="関連情報">
        <h3>関連記事</h3>
        {/* ... */}
      </aside>

      {/* フッター */}
      <footer>
        <p>&copy; 2026 Company</p>
      </footer>
    </div>
  )
}
```

**効果**: スクリーンリーダーユーザーが素早くセクションに移動できる

### フォーム要素のラベル

```tsx
// ❌ 悪い例: ラベルなし
<input type="text" placeholder="名前" />

// ✅ 良い例: label要素で紐付け
<div>
  <label htmlFor="name">名前</label>
  <input id="name" type="text" />
</div>

// ✅ 良い例: aria-labelを使用
<input type="text" aria-label="名前" />
```

---

## 3. ARIA属性の正しい使い方

### role属性

```tsx
// ✅ 良い例: カスタムボタンにrole指定
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
  クリック
</div>

// ⚠️ 注意: できるだけネイティブ要素を使う
<button onClick={handleClick}>クリック</button>
```

### aria-label / aria-labelledby

```tsx
// パターン1: aria-label（シンプル）
<button aria-label="メニューを閉じる">
  <CloseIcon />
</button>

// パターン2: aria-labelledby（複数要素から）
<dialog aria-labelledby="dialog-title" aria-describedby="dialog-desc">
  <h2 id="dialog-title">確認</h2>
  <p id="dialog-desc">本当に削除しますか？</p>
  <button>削除</button>
  <button>キャンセル</button>
</dialog>
```

### aria-live（動的コンテンツ）

```tsx
// 通知コンポーネント
export function Toast({ message }: { message: string }) {
  return (
    <div
      role="status"
      aria-live="polite"  // 'polite' | 'assertive' | 'off'
      aria-atomic="true"
      className="toast"
    >
      {message}
    </div>
  )
}

// 使用例
function App() {
  const [toast, setToast] = useState('')

  const handleSave = async () => {
    await save()
    setToast('保存しました') // スクリーンリーダーが読み上げる
  }

  return (
    <>
      <button onClick={handleSave}>保存</button>
      {toast && <Toast message={toast} />}
    </>
  )
}
```

**aria-liveの値**:
- `polite`: 現在の読み上げが終わってから通知
- `assertive`: すぐに通知（緊急時のみ）
- `off`: 通知しない

---

## 4. キーボード操作への対応

### フォーカス管理

#### パターン1: モーダルのフォーカストラップ

```tsx
'use client'

import { useEffect, useRef } from 'react'

interface ModalProps {
  isOpen: boolean
  onClose: () => void
  children: React.ReactNode
}

export function Modal({ isOpen, onClose, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null)
  const previousFocusRef = useRef<HTMLElement | null>(null)

  useEffect(() => {
    if (isOpen) {
      // モーダルを開く時
      previousFocusRef.current = document.activeElement as HTMLElement
      modalRef.current?.focus()

      // Escキーで閉じる
      const handleEscape = (e: KeyboardEvent) => {
        if (e.key === 'Escape') onClose()
      }
      document.addEventListener('keydown', handleEscape)

      return () => {
        document.removeEventListener('keydown', handleEscape)
        // フォーカスを元に戻す
        previousFocusRef.current?.focus()
      }
    }
  }, [isOpen, onClose])

  if (!isOpen) return null

  return (
    <div
      ref={modalRef}
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
      tabIndex={-1}
      className="modal"
    >
      <h2 id="modal-title">タイトル</h2>
      {children}
      <button onClick={onClose}>閉じる</button>
    </div>
  )
}
```

#### パターン2: タブUI

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

  const handleKeyDown = (e: React.KeyboardEvent, index: number) => {
    let newIndex = index

    switch (e.key) {
      case 'ArrowRight':
        newIndex = (index + 1) % tabs.length
        break
      case 'ArrowLeft':
        newIndex = (index - 1 + tabs.length) % tabs.length
        break
      case 'Home':
        newIndex = 0
        break
      case 'End':
        newIndex = tabs.length - 1
        break
      default:
        return
    }

    e.preventDefault()
    setActiveTab(newIndex)
  }

  return (
    <div>
      {/* タブリスト */}
      <div role="tablist" aria-label="タブ">
        {tabs.map((tab, index) => (
          <button
            key={tab.id}
            role="tab"
            aria-selected={activeTab === index}
            aria-controls={`panel-${tab.id}`}
            id={`tab-${tab.id}`}
            tabIndex={activeTab === index ? 0 : -1}
            onClick={() => setActiveTab(index)}
            onKeyDown={(e) => handleKeyDown(e, index)}
          >
            {tab.label}
          </button>
        ))}
      </div>

      {/* タブパネル */}
      {tabs.map((tab, index) => (
        <div
          key={tab.id}
          role="tabpanel"
          id={`panel-${tab.id}`}
          aria-labelledby={`tab-${tab.id}`}
          hidden={activeTab !== index}
        >
          {tab.content}
        </div>
      ))}
    </div>
  )
}
```

### スキップリンク

```tsx
// app/layout.tsx
export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {/* スキップリンク */}
        <a href="#main-content" className="skip-link">
          メインコンテンツへスキップ
        </a>

        <header>...</header>

        <main id="main-content">
          {children}
        </main>
      </body>
    </html>
  )
}
```

```css
/* globals.css */
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  background: #000;
  color: #fff;
  padding: 8px;
  text-decoration: none;
  z-index: 100;
}

.skip-link:focus {
  top: 0;
}
```

---

## 5. 色とコントラスト

### コントラスト比の基準

**WCAG AA基準**:
- 通常のテキスト: 4.5:1以上
- 大きなテキスト（18pt以上 or 14pt太字）: 3:1以上
- UIコンポーネント: 3:1以上

```tsx
// ❌ 悪い例: コントラスト不足（2:1）
<button style={{ background: '#999', color: '#bbb' }}>
  クリック
</button>

// ✅ 良い例: 十分なコントラスト（7:1）
<button style={{ background: '#000', color: '#fff' }}>
  クリック
</button>
```

### 色だけに依存しない

```tsx
// ❌ 悪い例: 色だけで区別
<p style={{ color: 'red' }}>エラー</p>
<p style={{ color: 'green' }}>成功</p>

// ✅ 良い例: アイコンとテキストで明示
<p style={{ color: 'red' }}>
  <ErrorIcon /> エラー: 入力内容を確認してください
</p>
<p style={{ color: 'green' }}>
  <SuccessIcon /> 成功: 保存しました
</p>
```

### ダークモード対応

```tsx
'use client'

import { useEffect, useState } from 'react'

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light')

  useEffect(() => {
    // ユーザー設定を尊重
    const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches
    setTheme(prefersDark ? 'dark' : 'light')
  }, [])

  return (
    <div data-theme={theme}>
      {children}
    </div>
  )
}
```

```css
/* globals.css */
[data-theme='light'] {
  --bg: #ffffff;
  --text: #000000;
}

[data-theme='dark'] {
  --bg: #1a1a1a;
  --text: #ffffff;
}

body {
  background: var(--bg);
  color: var(--text);
}
```

---

## 6. アクセシビリティテスト

### 自動テスト: axe-core

```bash
npm install --save-dev @axe-core/react
```

```tsx
// app/layout.tsx（開発環境のみ）
import { useEffect } from 'react'

export default function RootLayout({ children }) {
  useEffect(() => {
    if (process.env.NODE_ENV === 'development') {
      import('@axe-core/react').then((axe) => {
        axe.default(React, ReactDOM, 1000)
      })
    }
  }, [])

  return <html>{children}</html>
}
```

### 自動テスト: Lighthouse

```bash
# Chrome DevToolsのLighthouseタブで実行
# または CLI
npm install -g lighthouse
lighthouse https://your-site.com --only-categories=accessibility
```

### 手動テスト

#### 1. キーボードナビゲーション

```bash
# チェック項目
- Tab: 全ての要素にフォーカスできるか
- Enter/Space: ボタン・リンクが動作するか
- Esc: モーダル・ドロップダウンが閉じるか
- 矢印キー: タブ・リストが操作できるか
```

#### 2. スクリーンリーダーテスト

**macOS**:
```bash
# VoiceOver起動: Cmd + F5
# 基本操作:
- Ctrl + Option + 矢印: 要素間移動
- Ctrl + Option + Space: クリック
- Ctrl + Option + U: ランドマーク一覧
```

**Windows**:
```bash
# NVDA（無料）をインストール
# https://www.nvaccess.org/
```

#### 3. コントラストチェック

```bash
# Chrome DevTools
1. 要素を検査
2. Computed → Accessibility
3. Contrast ratio を確認
```

### Jest + React Testing Library

```bash
npm install --save-dev jest-axe
```

```typescript
// __tests__/Button.test.tsx
import { render } from '@testing-library/react'
import { axe, toHaveNoViolations } from 'jest-axe'
import { Button } from '@/components/Button'

expect.extend(toHaveNoViolations)

describe('Button accessibility', () => {
  it('should not have accessibility violations', async () => {
    const { container } = render(<Button>Click me</Button>)
    const results = await axe(container)
    expect(results).toHaveNoViolations()
  })

  it('should be keyboard accessible', () => {
    const handleClick = jest.fn()
    const { getByRole } = render(<Button onClick={handleClick}>Click</Button>)

    const button = getByRole('button')
    button.focus()

    // Enter key
    fireEvent.keyDown(button, { key: 'Enter' })
    expect(handleClick).toHaveBeenCalledTimes(1)
  })
})
```

---

## まとめ

この章では、Webアクセシビリティの実践を学びました。

**重要ポイント**:
- ✅ WCAG AA準拠を目指す（法的要件・ビジネス価値）
- ✅ セマンティックHTMLを優先（divより適切な要素）
- ✅ ARIA属性は補助的に使用（ネイティブ要素が優先）
- ✅ キーボード操作を完全サポート
- ✅ コントラスト比4.5:1以上を確保
- ✅ 自動テスト（axe, Lighthouse）+ 手動テスト

**次章予告**: Chapter 8では、プロジェクトアーキテクチャ設計を学びます。

---

**参考リンク**:
- [WCAG 2.1](https://www.w3.org/WAI/WCAG21/quickref/)
- [WAI-ARIA Authoring Practices](https://www.w3.org/WAI/ARIA/apg/)
- [axe DevTools](https://www.deque.com/axe/devtools/)
