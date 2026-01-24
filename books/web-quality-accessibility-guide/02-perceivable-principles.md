# Perceivable（知覚可能） - 情報を認識できるようにする

## Perceivableとは

Perceivable（知覚可能）は、WCAG 2.1の第1原則です。ユーザーが情報とユーザーインターフェースのコンポーネントを認識できる形で提示する必要があります。

視覚、聴覚、触覚など、ユーザーの少なくとも1つの感覚で情報を受け取れるようにすることが求められます。

## ガイドライン1.1: テキストによる代替

### 達成基準1.1.1 非テキストコンテンツ（Level A）

**要件**: 全ての非テキストコンテンツに、テキストによる代替を提供する。

#### 実装例: 画像のalt属性

```tsx
// ✅ 良い例
<img
  src="/product.jpg"
  alt="青いTシャツ、サイズM、綿100%"
/>

// ✅ 装飾的な画像
<img
  src="/decorative-line.svg"
  alt=""
  role="presentation"
/>

// ❌ 悪い例（altがない）
<img src="/product.jpg" />
```

#### 実装例: アイコンボタン

```tsx
// ✅ 良い例
<button aria-label="メニューを開く">
  <MenuIcon />
</button>

// ❌ 悪い例（ラベルがない）
<button>
  <MenuIcon />
</button>
```

#### 実装例: キャプチャ画像

```tsx
// ✅ 良い例: 複雑な情報を含む画像
<figure>
  <img
    src="/sales-chart.png"
    alt="2025年の売上推移グラフ。1月100万円から12月500万円まで右肩上がり"
  />
  <figcaption>
    2025年売上推移: 1月から12月まで継続的に増加し、年間成長率400%
  </figcaption>
</figure>
```

## ガイドライン1.2: 時間依存メディア

### 達成基準1.2.1 音声のみ及び映像のみ（Level A）

**要件**: 音声のみ、または映像のみのコンテンツに代替を提供する。

#### 実装例: 音声ファイル

```tsx
<div>
  <audio controls src="/podcast.mp3">
    お使いのブラウザは音声要素をサポートしていません。
  </audio>
  <details>
    <summary>トランスクリプト</summary>
    <p>
      こんにちは。今日のトピックは...
    </p>
  </details>
</div>
```

### 達成基準1.2.2 キャプション（Level A）

**要件**: 動画に字幕を提供する。

#### 実装例: 動画の字幕

```tsx
<video controls>
  <source src="/video.mp4" type="video/mp4" />
  <track
    kind="captions"
    src="/captions-ja.vtt"
    srclang="ja"
    label="日本語"
    default
  />
  <track
    kind="captions"
    src="/captions-en.vtt"
    srclang="en"
    label="English"
  />
</video>
```

#### VTTファイルの例

```
WEBVTT

00:00:00.000 --> 00:00:03.000
こんにちは。今日のトピックは
Webアクセシビリティです。

00:00:03.500 --> 00:00:07.000
WCAG 2.1について解説します。
```

### 達成基準1.2.3 音声解説またはメディアの代替（Level A）

**要件**: 視覚情報を音声で説明する、またはトランスクリプトを提供する。

### 達成基準1.2.5 音声解説（Level AA）

**要件**: 動画に音声解説を提供する。

## ガイドライン1.3: 適応可能

### 達成基準1.3.1 情報及び関係性（Level A）

**要件**: 構造と関係性をプログラム的に決定できる、またはテキストで提供する。

#### 実装例: セマンティックHTML

```tsx
// ✅ 良い例: セマンティックな見出し構造
<article>
  <h1>記事タイトル</h1>
  <section>
    <h2>セクション1</h2>
    <h3>サブセクション1.1</h3>
    <p>コンテンツ...</p>
  </section>
  <section>
    <h2>セクション2</h2>
    <p>コンテンツ...</p>
  </section>
</article>

// ❌ 悪い例: divとスタイルだけ
<div>
  <div class="big-text">記事タイトル</div>
  <div class="medium-text">セクション1</div>
  <p>コンテンツ...</p>
</div>
```

#### 実装例: データテーブル

```tsx
// ✅ 良い例: アクセシブルなテーブル
<table>
  <caption>2025年度 売上実績</caption>
  <thead>
    <tr>
      <th scope="col">月</th>
      <th scope="col">売上（万円）</th>
      <th scope="col">前年比</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">1月</th>
      <td>100</td>
      <td>+20%</td>
    </tr>
    <tr>
      <th scope="row">2月</th>
      <td>150</td>
      <td>+30%</td>
    </tr>
  </tbody>
</table>
```

#### 実装例: フォームのラベル

```tsx
// ✅ 良い例: ラベルと入力の関連付け
<form>
  <div>
    <label htmlFor="username">ユーザー名</label>
    <input
      id="username"
      type="text"
      required
      aria-required="true"
    />
  </div>

  <fieldset>
    <legend>通知設定</legend>
    <label>
      <input type="checkbox" name="email" />
      メール通知を受け取る
    </label>
    <label>
      <input type="checkbox" name="sms" />
      SMS通知を受け取る
    </label>
  </fieldset>
</form>
```

### 達成基準1.3.2 意味のある順序（Level A）

**要件**: コンテンツの順序が意味を持つ場合、正しい読み上げ順序をプログラム的に決定できる。

#### 実装例: DOMの順序

```tsx
// ✅ 良い例: 論理的な順序
<main>
  <h1>ページタイトル</h1>
  <nav aria-label="パンくずリスト">
    <ol>
      <li><a href="/">ホーム</a></li>
      <li><a href="/products">製品</a></li>
      <li aria-current="page">製品詳細</li>
    </ol>
  </nav>
  <article>
    <h2>製品概要</h2>
    <p>製品の説明...</p>
  </article>
  <aside>
    <h2>関連製品</h2>
    <ul>...</ul>
  </aside>
</main>

// ❌ 悪い例: CSSで視覚的順序を変更
<div style="display: flex; flex-direction: column-reverse;">
  <div>2番目に見える</div>
  <div>1番目に見える</div>
</div>
```

### 達成基準1.3.3 感覚的な特徴（Level A）

**要件**: 色、形、サイズ、位置だけでなく、テキストでも指示を提供する。

#### 実装例: 指示の提供

```tsx
// ✅ 良い例: 複数の手がかり
<form>
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
  <p id="password-help" className="text-sm text-gray-600">
    8文字以上、大文字・小文字・数字を含む必要があります
  </p>
</form>

// ❌ 悪い例: 色だけで伝える
<p style="color: red;">
  エラーがあります
</p>
```

### 達成基準1.3.4 表示の向き（Level AA）

**要件**: 縦向き、横向きのどちらでもコンテンツが利用できる。

#### 実装例: レスポンシブデザイン

```css
/* ✅ 良い例: 両方の向きに対応 */
@media (orientation: portrait) {
  .container {
    flex-direction: column;
  }
}

@media (orientation: landscape) {
  .container {
    flex-direction: row;
  }
}

/* ❌ 悪い例: 向きを固定 */
@media (orientation: portrait) {
  body {
    transform: rotate(90deg);
  }
}
```

### 達成基準1.3.5 入力目的の特定（Level AA）

**要件**: フォーム入力の目的を特定できるようにする。

#### 実装例: autocomplete属性

```tsx
// ✅ 良い例: autocomplete属性
<form>
  <label htmlFor="name">氏名</label>
  <input
    id="name"
    type="text"
    autoComplete="name"
  />

  <label htmlFor="email">メールアドレス</label>
  <input
    id="email"
    type="email"
    autoComplete="email"
  />

  <label htmlFor="tel">電話番号</label>
  <input
    id="tel"
    type="tel"
    autoComplete="tel"
  />

  <label htmlFor="address">住所</label>
  <input
    id="address"
    type="text"
    autoComplete="street-address"
  />
</form>
```

## ガイドライン1.4: 判別可能

### 達成基準1.4.1 色の使用（Level A）

**要件**: 色だけで情報を伝えない。

#### 実装例: 複数の手がかり

```tsx
// ✅ 良い例: 色とアイコン、テキスト
<div className="flex items-center gap-2 text-green-700">
  <CheckCircleIcon />
  <span>送信成功</span>
</div>

<div className="flex items-center gap-2 text-red-700">
  <XCircleIcon />
  <span>エラー: 入力内容を確認してください</span>
</div>

// ❌ 悪い例: 色だけ
<div className="bg-green-500 p-2">
  OK
</div>
```

### 達成基準1.4.3 コントラスト（Level AA）

**要件**: テキストと背景のコントラスト比が最低4.5:1（大きなテキストは3:1）。

#### 実装例: 適切なコントラスト

```css
/* ✅ 良い例: 十分なコントラスト */
.text {
  color: #333333; /* コントラスト比 12.6:1 */
  background: #ffffff;
}

.large-text {
  color: #666666; /* コントラスト比 5.7:1 */
  background: #ffffff;
  font-size: 24px;
}

/* ❌ 悪い例: 不十分なコントラスト */
.text-bad {
  color: #999999; /* コントラスト比 2.8:1 */
  background: #ffffff;
}
```

### 達成基準1.4.4 テキストのサイズ変更（Level AA）

**要件**: 200%までテキストをサイズ変更しても、コンテンツや機能が失われない。

#### 実装例: 相対単位の使用

```css
/* ✅ 良い例: remやemを使用 */
body {
  font-size: 16px;
}

h1 {
  font-size: 2rem; /* 32px */
}

p {
  font-size: 1rem; /* 16px */
  line-height: 1.5;
}

/* ❌ 悪い例: 固定ピクセル値 */
p {
  font-size: 12px;
  width: 600px;
  overflow: hidden;
}
```

### 達成基準1.4.10 リフロー（Level AA）

**要件**: 320pxの幅でも水平スクロールなしでコンテンツが利用できる。

#### 実装例: レスポンシブデザイン

```css
/* ✅ 良い例: フレキシブルなレイアウト */
.container {
  max-width: 100%;
  padding: 1rem;
}

.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 1rem;
}

/* ❌ 悪い例: 固定幅 */
.container {
  width: 1200px;
}
```

### 達成基準1.4.11 非テキストコントラスト（Level AA）

**要件**: UI要素や重要なグラフィックのコントラスト比が最低3:1。

#### 実装例: ボタンのコントラスト

```css
/* ✅ 良い例: ボーダーとフォーカス */
button {
  background: #0066cc;
  color: #ffffff;
  border: 2px solid #004080; /* コントラスト比 3.5:1 */
}

button:focus-visible {
  outline: 3px solid #ff6600; /* コントラスト比 4.2:1 */
  outline-offset: 2px;
}
```

### 達成基準1.4.12 テキストの間隔（Level AA）

**要件**: 行間、段落間などを調整しても、コンテンツが失われない。

#### 実装例: 十分な間隔

```css
/* ✅ 良い例: 調整可能な間隔 */
p {
  line-height: 1.5;
  margin-bottom: 1em;
  letter-spacing: 0.05em;
  word-spacing: 0.1em;
}

/* テキスト間隔調整に対応 */
* {
  line-height: inherit !important;
}
```

### 達成基準1.4.13 ホバーまたはフォーカスで表示されるコンテンツ（Level AA）

**要件**: ツールチップなどが表示される際、閉じられる、ホバー可能、表示が継続する。

#### 実装例: アクセシブルなツールチップ

```tsx
'use client'

import { useState } from 'react'

export function Tooltip({ children, content }: { children: React.ReactNode, content: string }) {
  const [isVisible, setIsVisible] = useState(false)

  return (
    <div className="relative inline-block">
      <button
        onMouseEnter={() => setIsVisible(true)}
        onMouseLeave={() => setIsVisible(false)}
        onFocus={() => setIsVisible(true)}
        onBlur={() => setIsVisible(false)}
        aria-describedby="tooltip"
      >
        {children}
      </button>
      {isVisible && (
        <div
          id="tooltip"
          role="tooltip"
          className="absolute z-10 px-3 py-2 text-sm bg-gray-900 text-white rounded"
        >
          {content}
        </div>
      )}
    </div>
  )
}
```

## まとめ

Perceivable（知覚可能）原則は、ユーザーが情報を認識できるようにすることを求めます。主なポイント：

- 非テキストコンテンツにテキスト代替を提供
- 動画に字幕と音声解説を提供
- セマンティックHTMLで構造を表現
- 十分なカラーコントラスト
- レスポンシブでリフロー可能なデザイン

### チェックリスト

- 全ての画像にalt属性がある
- 動画に字幕がある
- セマンティックHTMLを使用している
- コントラスト比が4.5:1以上（通常テキスト）
- 320pxの幅でも水平スクロールなし

## 次のステップ

次章では、WCAG 2.1の第2原則「Operable（操作可能）」について解説します。
