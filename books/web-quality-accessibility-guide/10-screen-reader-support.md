---
title: "スクリーンリーダー対応 - 音声で理解できるようにする"
---

# スクリーンリーダー対応 - 音声で理解できるようにする

## スクリーンリーダーとは

スクリーンリーダーは、画面の内容を音声で読み上げるソフトウェアです。視覚障害のあるユーザーが主に使用します。

WebAIMの2023年調査によると、スクリーンリーダーユーザーの約87.6%が日常的にスクリーンリーダーを使用しています[^webaim-survey]。

[^webaim-survey]: [Screen Reader User Survey #10 Results - WebAIM](https://webaim.org/projects/screenreadersurvey10/)

### 主要なスクリーンリーダー

#### NVDA（Windows）

- **概要**: 無料、オープンソース
- **シェア**: Windowsユーザーの約41.6%が使用[^webaim-survey]
- **特徴**:
  - 無料で高機能
  - 定期的にアップデート
  - 多言語対応（日本語含む）
- **ダウンロード**: [NV Access](https://www.nvaccess.org/)

#### JAWS（Windows）

- **概要**: 有料、高機能
- **シェア**: Windowsユーザーの約53.7%が使用[^webaim-survey]
- **特徴**:
  - 最も歴史があり、高機能
  - 企業向けサポートが充実
  - 年間ライセンス約$90（2026年時点）
- **公式サイト**: [Freedom Scientific](https://www.freedomscientific.com/products/software/jaws/)

#### VoiceOver（macOS/iOS）

- **概要**: 標準搭載
- **シェア**: macOSユーザーの約70.5%が使用[^webaim-survey]
- **特徴**:
  - macOS/iOSに標準搭載
  - Safariとの親和性が高い
  - ローターによる要素ナビゲーション
- **起動方法**: Command + F5（macOS）

#### TalkBack（Android）

- **概要**: 標準搭載
- **特徴**:
  - Androidに標準搭載
  - タッチジェスチャーでの操作
  - Google Chromeとの親和性が高い

#### ORCA（Linux）

- **概要**: 無料、オープンソース
- **特徴**:
  - Linux用の主要スクリーンリーダー
  - GNOMEデスクトップ環境で利用可能

## スクリーンリーダーの基本操作

### VoiceOver（macOS）

- 起動/終了: Command + F5
- 次の要素: VO + →（VO = Control + Option）
- 前の要素: VO + ←
- クリック: VO + Space
- ローター: VO + U（見出し、リンク、フォーム要素等をリスト表示）

### NVDA（Windows）

- 起動/終了: Control + Alt + N
- 次の要素: ↓
- 前の要素: ↑
- クリック: Enter
- 要素リスト: NVDA + F7

## 実装のベストプラクティス

### 1. セマンティックHTMLの使用

スクリーンリーダーは、セマンティックHTMLを使って文書構造を理解します。

```tsx
// ✅ 良い例
<nav aria-label="メインナビゲーション">
  <ul>
    <li><a href="/">ホーム</a></li>
  </ul>
</nav>

// ❌ 悪い例
<div class="navigation">
  <div><a href="/">ホーム</a></div>
</div>
```

### 2. alt属性の適切な使用

```tsx
// ✅ 情報を含む画像
<img src="/product.jpg" alt="青いTシャツ、サイズM" />

// ✅ 装飾的な画像
<img src="/decoration.svg" alt="" role="presentation" />

// ✅ リンク内の画像
<a href="/products/123">
  <img src="/product.jpg" alt="青いTシャツの詳細を見る" />
</a>
```

### 3. フォームのラベル

```tsx
// ✅ 明確なラベル
<label htmlFor="email">メールアドレス</label>
<input id="email" type="email" required aria-required="true" />

// ✅ ヘルプテキスト
<label htmlFor="password">パスワード</label>
<input
  id="password"
  type="password"
  aria-describedby="password-help"
/>
<p id="password-help">8文字以上、大文字・小文字・数字を含む</p>
```

### 4. ライブリージョン

動的な変更をスクリーンリーダーに通知します。

```tsx
// ✅ ステータスメッセージ
<div role="status" aria-live="polite">
  {message}
</div>

// ✅ エラーメッセージ
<div role="alert" aria-live="assertive">
  {error}
</div>
```

## ARIA属性の活用

ARIA（Accessible Rich Internet Applications）属性は、スクリーンリーダーに追加情報を提供します[^aria-spec]。

[^aria-spec]: [WAI-ARIA 1.2 Specification - W3C](https://www.w3.org/TR/wai-aria-1.2/)

### aria-label と aria-labelledby

```tsx
// ✅ aria-label: 要素に直接ラベルを付ける
<button aria-label="閉じる">
  <XIcon />
</button>

// ✅ aria-labelledby: 他の要素を参照してラベルを付ける
<div role="dialog" aria-labelledby="dialog-title" aria-describedby="dialog-desc">
  <h2 id="dialog-title">確認</h2>
  <p id="dialog-desc">この操作を実行してもよろしいですか？</p>
  <button>はい</button>
  <button>いいえ</button>
</div>
```

### aria-describedby

追加の説明情報を提供します。

```tsx
<label htmlFor="password">パスワード</label>
<input
  id="password"
  type="password"
  aria-describedby="password-requirements"
  aria-invalid={hasError}
  aria-errormessage={hasError ? "password-error" : undefined}
/>
<p id="password-requirements">
  8文字以上、大文字・小文字・数字を含む
</p>
{hasError && (
  <p id="password-error" role="alert">
    パスワードが要件を満たしていません
  </p>
)}
```

### aria-live

動的な変更をスクリーンリーダーに通知します。

| 値 | 意味 | 使用例 |
|---|-----|-------|
| **off** | 通知しない（デフォルト） | - |
| **polite** | ユーザーの操作が完了してから通知 | ステータスメッセージ、検索結果 |
| **assertive** | すぐに通知（現在の読み上げを中断） | エラーメッセージ、緊急通知 |

```tsx
'use client'

import { useState } from 'react'

export function SearchForm() {
  const [status, setStatus] = useState('')

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setStatus('検索中...')
    // 検索処理
    await new Promise((resolve) => setTimeout(resolve, 1000))
    setStatus('検索が完了しました。10件の結果が見つかりました。')
  }

  return (
    <form onSubmit={handleSubmit}>
      <input type="search" aria-label="検索" />
      <button type="submit">検索</button>

      {/* polite: ユーザーの操作後に読み上げ */}
      <div role="status" aria-live="polite" aria-atomic="true">
        {status}
      </div>
    </form>
  )
}
```

### aria-hidden

スクリーンリーダーから要素を隠します。装飾的な要素に使用します。

```tsx
// ✅ 装飾的なアイコン
<button>
  <CheckIcon aria-hidden="true" />
  送信
</button>

// ❌ 悪い例: 重要な情報を隠す
<p aria-hidden="true">重要なお知らせ</p>
```

## テスト方法

スクリーンリーダーでのテストは、アクセシビリティ検証の重要な部分です。自動テストでは検出できない問題を発見できます。

### VoiceOverでのテスト（macOS）

**基本操作**:
1. Command + F5 で起動
2. VO + → で要素を順番に確認（VO = Control + Option）
3. VO + U でローターを開き、見出し・リンク・フォーム要素を確認
4. 全てのコンテンツが適切に読み上げられるか確認

**具体的なテスト項目**:
- [ ] 見出し構造が正しい（h1からh6が論理的）
- [ ] 全ての画像にalt属性がある
- [ ] フォームラベルが正しく関連付けられている
- [ ] ボタンの役割が明確
- [ ] リンクの目的地が理解できる
- [ ] ライブリージョンが適切に読み上げられる

**ローターの使い方**:
- VO + U → 見出しモード: H キーで見出しをジャンプ
- VO + U → リンクモード: L キーでリンクをジャンプ
- VO + U → フォームコントロールモード: F キーでフォーム要素をジャンプ

### NVDAでのテスト（Windows）

**基本操作**:
1. Control + Alt + N で起動
2. ↓ で要素を順番に確認
3. NVDA + F7 で要素リストを確認
4. H キーで見出しジャンプ

**ブラウズモードのショートカット**:
- **H**: 次の見出し
- **Shift + H**: 前の見出し
- **1〜6**: 特定のレベルの見出しにジャンプ（例: 2キーでh2）
- **K**: 次のリンク
- **E**: 次の編集フィールド
- **B**: 次のボタン
- **L**: 次のリスト

**音声確認のポイント**:
- ページタイトルが最初に読み上げられるか
- ランドマーク（navigation、main、contentinfo等）が正しく読み上げられるか
- フォーカス移動時に要素の種類（ボタン、リンク等）が読み上げられるか

### JAWSでのテスト（Windows）

**基本操作**:
1. JAWSを起動（通常は自動起動）
2. ↓ で要素を順番に確認
3. Insert + F6 で見出しリストを表示
4. Insert + F7 でリンクリストを表示

**バーチャルカーソルのショートカット**:
- NVDAとほぼ同じキー配置
- H: 見出し、K: リンク、B: ボタン等

### テスト時の注意点

1. **複数のスクリーンリーダーでテスト**: 少なくともVoiceOver（macOS）とNVDA（Windows）で確認
2. **実際のユーザーと同じ環境**: ブラウザとスクリーンリーダーの組み合わせに注意
3. **音声のみで理解できるか**: 画面を見ずにタスクを完了できるか確認
4. **自動テストと併用**: axe DevToolsやLighthouseと組み合わせる

## まとめ

スクリーンリーダー対応は、セマンティックHTML、適切なラベル、ARIA属性の組み合わせです。

### チェックリスト

- セマンティックHTMLを使用
- 全ての画像にalt属性
- フォームに明確なラベル
- ライブリージョンで動的変更を通知
- スクリーンリーダーでテスト済み

## 次のステップ

次章では、カラーコントラストについて解説します。
