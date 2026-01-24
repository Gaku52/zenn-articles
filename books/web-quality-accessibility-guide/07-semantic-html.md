# セマンティックHTML - 意味のあるマークアップ

## セマンティックHTMLとは

セマンティック（semantic）とは「意味のある」という意味です。セマンティックHTMLは、コンテンツの意味や構造を適切に表現するHTMLを指します。

divやspanだけでマークアップするのではなく、header、nav、main、article、section等の意味を持つ要素を使用することで、ブラウザ、スクリーンリーダー、検索エンジンがコンテンツを正しく理解できるようになります。

## なぜセマンティックHTMLが重要か

### 1. アクセシビリティの向上

スクリーンリーダーは、セマンティックHTMLを使って文書構造を理解し、ユーザーに伝えます。

```tsx
// ✅ 良い例: スクリーンリーダーが「ナビゲーション」と認識
<nav>
  <ul>
    <li><a href="/">ホーム</a></li>
    <li><a href="/products">製品</a></li>
  </ul>
</nav>

// ❌ 悪い例: 単なるdivとして認識
<div className="navigation">
  <div>
    <div><a href="/">ホーム</a></div>
    <div><a href="/products">製品</a></div>
  </div>
</div>
```

### 2. SEOの向上

検索エンジンは、セマンティックHTMLを使ってページの構造と重要度を理解します。

- h1: ページの最も重要な見出し
- article: 独立したコンテンツ
- time: 日付・時刻情報

### 3. 保守性の向上

意味のあるHTML要素を使うことで、コードが読みやすく、理解しやすくなります。

### 4. 機能の自動取得

ブラウザは、セマンティックHTMLに基づいて自動的に機能を提供します。

- button要素: キーボード操作、フォーカス管理
- form要素: Enter キーでの送信
- nav要素: スクリーンリーダーのランドマークナビゲーション

## 主要なセマンティック要素

### ランドマーク要素

ランドマーク要素は、ページの主要なセクションを定義します。

```tsx
<body>
  <header>
    {/* サイトヘッダー */}
    <h1>サイト名</h1>
    <nav aria-label="メインナビゲーション">
      {/* グローバルナビゲーション */}
    </nav>
  </header>

  <main>
    {/* メインコンテンツ */}
    <article>
      <h2>記事タイトル</h2>
      <p>記事本文...</p>
    </article>

    <aside>
      {/* サイドバー、関連情報 */}
    </aside>
  </main>

  <footer>
    {/* フッター */}
  </footer>
</body>
```

#### header

ページまたはセクションのヘッダーを表します。

```tsx
// ✅ ページヘッダー
<header>
  <h1>Web品質・アクセシビリティ完全ガイド</h1>
  <nav>{/* ナビゲーション */}</nav>
</header>

// ✅ 記事ヘッダー
<article>
  <header>
    <h2>記事タイトル</h2>
    <p>著者: 山田太郎 | 公開日: 2026年1月24日</p>
  </header>
  <p>記事本文...</p>
</article>
```

#### nav

ナビゲーションリンクのセクションを表します。

```tsx
// ✅ 良い例: 複数のnavにaria-labelで区別
<header>
  <nav aria-label="メインナビゲーション">
    <ul>
      <li><a href="/">ホーム</a></li>
      <li><a href="/products">製品</a></li>
      <li><a href="/about">会社概要</a></li>
    </ul>
  </nav>
</header>

<aside>
  <nav aria-label="記事内ナビゲーション">
    <h2>目次</h2>
    <ol>
      <li><a href="#section1">セクション1</a></li>
      <li><a href="#section2">セクション2</a></li>
    </ol>
  </nav>
</aside>

<footer>
  <nav aria-label="フッターナビゲーション">
    <ul>
      <li><a href="/privacy">プライバシーポリシー</a></li>
      <li><a href="/terms">利用規約</a></li>
    </ul>
  </nav>
</footer>
```

#### main

ページのメインコンテンツを表します。**ページに1つだけ**。

```tsx
// ✅ 良い例
<body>
  <header>{/* ヘッダー */}</header>
  <main>
    <h1>ページタイトル</h1>
    <p>メインコンテンツ...</p>
  </main>
  <footer>{/* フッター */}</footer>
</body>

// ❌ 悪い例: 複数のmain
<body>
  <main>{/* メイン1 */}</main>
  <main>{/* メイン2 */}</main>
</body>
```

#### article

独立したコンテンツを表します（ブログ記事、ニュース記事、フォーラム投稿等）。

```tsx
// ✅ ブログ記事
<article>
  <header>
    <h2>WCAG 2.1準拠のベストプラクティス</h2>
    <p>
      <time datetime="2026-01-24">2026年1月24日</time>
      著者: <span>山田太郎</span>
    </p>
  </header>

  <section>
    <h3>はじめに</h3>
    <p>本記事では...</p>
  </section>

  <section>
    <h3>実装方法</h3>
    <p>実装方法について...</p>
  </section>

  <footer>
    <p>タグ: <a href="/tags/accessibility">アクセシビリティ</a></p>
  </footer>
</article>

// ✅ ニュース一覧
<div>
  <h2>最新ニュース</h2>
  <article>
    <h3>ニュース1</h3>
    <p>概要...</p>
  </article>
  <article>
    <h3>ニュース2</h3>
    <p>概要...</p>
  </article>
</div>
```

#### section

テーマに基づいたコンテンツのグループを表します。

```tsx
// ✅ 良い例: 各sectionに見出しがある
<article>
  <h1>製品紹介</h1>

  <section>
    <h2>機能</h2>
    <p>主な機能...</p>
  </section>

  <section>
    <h2>価格</h2>
    <p>価格プラン...</p>
  </section>

  <section>
    <h2>サポート</h2>
    <p>サポート情報...</p>
  </section>
</article>

// ❌ 悪い例: 見出しのないsection
<section>
  <p>何かのコンテンツ...</p>
</section>
```

#### aside

メインコンテンツと間接的に関連するコンテンツを表します（サイドバー、広告、関連リンク等）。

```tsx
// ✅ サイドバー
<main>
  <article>
    <h1>記事タイトル</h1>
    <p>記事本文...</p>
  </article>

  <aside>
    <h2>関連記事</h2>
    <ul>
      <li><a href="/article1">関連記事1</a></li>
      <li><a href="/article2">関連記事2</a></li>
    </ul>
  </aside>
</main>

// ✅ 記事内の補足情報
<article>
  <h1>記事タイトル</h1>
  <p>本文...</p>

  <aside>
    <h2>用語解説</h2>
    <p>この記事で使用する専門用語の解説...</p>
  </aside>

  <p>本文の続き...</p>
</article>
```

#### footer

ページまたはセクションのフッターを表します。

```tsx
// ✅ ページフッター
<footer>
  <p>&copy; 2026 会社名</p>
  <nav aria-label="フッターナビゲーション">
    <a href="/privacy">プライバシーポリシー</a>
    <a href="/terms">利用規約</a>
  </nav>
</footer>

// ✅ 記事フッター
<article>
  <h2>記事タイトル</h2>
  <p>本文...</p>
  <footer>
    <p>著者: 山田太郎</p>
    <p>公開日: <time datetime="2026-01-24">2026年1月24日</time></p>
  </footer>
</article>
```

### 見出し要素（h1〜h6）

見出しは文書の階層構造を表します。**スキップせずに順番に使用**することが重要です。

```tsx
// ✅ 良い例: 階層的な見出し
<h1>ページタイトル</h1>
  <h2>セクション1</h2>
    <h3>サブセクション1.1</h3>
    <h3>サブセクション1.2</h3>
  <h2>セクション2</h2>
    <h3>サブセクション2.1</h3>

// ❌ 悪い例: h2をスキップ
<h1>ページタイトル</h1>
  <h3>セクション1</h3> <!-- h2をスキップ -->

// ❌ 悪い例: 見た目だけで選択
<h1>大きな見出し</h1>
<h4>小さな見出し</h4> <!-- 見た目だけで選んでいる -->
```

#### 実装例: Next.jsでの見出し管理

```tsx
// components/Heading.tsx
type HeadingLevel = 1 | 2 | 3 | 4 | 5 | 6

interface HeadingProps {
  level: HeadingLevel
  children: React.ReactNode
  className?: string
}

export function Heading({ level, children, className }: HeadingProps) {
  const Tag = `h${level}` as const

  return <Tag className={className}>{children}</Tag>
}

// 使用例
<Heading level={1}>ページタイトル</Heading>
<Heading level={2}>セクション</Heading>
<Heading level={3}>サブセクション</Heading>
```

### リスト要素

#### ul（順序なしリスト）

```tsx
// ✅ ナビゲーションメニュー
<nav>
  <ul>
    <li><a href="/">ホーム</a></li>
    <li><a href="/products">製品</a></li>
    <li><a href="/about">会社概要</a></li>
  </ul>
</nav>

// ✅ 機能一覧
<section>
  <h2>主な機能</h2>
  <ul>
    <li>自動バックアップ</li>
    <li>リアルタイム同期</li>
    <li>高度なセキュリティ</li>
  </ul>
</section>
```

#### ol（順序ありリスト）

```tsx
// ✅ 手順
<section>
  <h2>セットアップ手順</h2>
  <ol>
    <li>アカウントを作成</li>
    <li>プロフィールを設定</li>
    <li>初期設定を完了</li>
  </ol>
</section>

// ✅ ランキング
<section>
  <h2>人気記事トップ3</h2>
  <ol>
    <li>WCAG 2.1準拠ガイド</li>
    <li>セマンティックHTML入門</li>
    <li>ARIA属性の使い方</li>
  </ol>
</section>
```

#### dl（説明リスト）

```tsx
// ✅ 用語と定義
<dl>
  <dt>WCAG</dt>
  <dd>Web Content Accessibility Guidelinesの略。Webアクセシビリティの国際標準。</dd>

  <dt>ARIA</dt>
  <dd>Accessible Rich Internet Applicationsの略。スクリーンリーダー対応のための属性。</dd>
</dl>

// ✅ 製品仕様
<dl>
  <dt>サイズ</dt>
  <dd>幅: 200mm</dd>
  <dd>高さ: 100mm</dd>

  <dt>重量</dt>
  <dd>500g</dd>

  <dt>カラー</dt>
  <dd>ブラック、ホワイト、シルバー</dd>
</dl>
```

### テーブル要素

データテーブルには、適切なマークアップが必要です。

```tsx
// ✅ 良い例: アクセシブルなテーブル
<table>
  <caption>2025年度 四半期別売上実績</caption>
  <thead>
    <tr>
      <th scope="col">四半期</th>
      <th scope="col">売上（万円）</th>
      <th scope="col">前年比</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">Q1</th>
      <td>1,000</td>
      <td>+20%</td>
    </tr>
    <tr>
      <th scope="row">Q2</th>
      <td>1,200</td>
      <td>+25%</td>
    </tr>
    <tr>
      <th scope="row">Q3</th>
      <td>1,500</td>
      <td>+30%</td>
    </tr>
    <tr>
      <th scope="row">Q4</th>
      <td>2,000</td>
      <td>+35%</td>
    </tr>
  </tbody>
  <tfoot>
    <tr>
      <th scope="row">合計</th>
      <td>5,700</td>
      <td>+27.5%</td>
    </tr>
  </tfoot>
</table>

// ✅ 複雑なテーブル
<table>
  <caption>製品比較表</caption>
  <thead>
    <tr>
      <th scope="col">機能</th>
      <th scope="col">無料プラン</th>
      <th scope="col">プロプラン</th>
      <th scope="col">エンタープライズ</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">ストレージ</th>
      <td>10GB</td>
      <td>100GB</td>
      <td>無制限</td>
    </tr>
    <tr>
      <th scope="row">ユーザー数</th>
      <td>1名</td>
      <td>10名</td>
      <td>無制限</td>
    </tr>
  </tbody>
</table>
```

### フォーム要素

フォームには、適切なラベルと構造が必要です。

```tsx
// ✅ 良い例: アクセシブルなフォーム
<form>
  {/* テキスト入力 */}
  <div>
    <label htmlFor="name">
      氏名
      <span className="text-red-600" aria-label="必須">*</span>
    </label>
    <input
      id="name"
      type="text"
      required
      aria-required="true"
    />
  </div>

  {/* ラジオボタングループ */}
  <fieldset>
    <legend>性別</legend>
    <label>
      <input type="radio" name="gender" value="male" />
      男性
    </label>
    <label>
      <input type="radio" name="gender" value="female" />
      女性
    </label>
    <label>
      <input type="radio" name="gender" value="other" />
      その他
    </label>
  </fieldset>

  {/* チェックボックスグループ */}
  <fieldset>
    <legend>興味のある分野（複数選択可）</legend>
    <label>
      <input type="checkbox" name="interest" value="frontend" />
      フロントエンド
    </label>
    <label>
      <input type="checkbox" name="interest" value="backend" />
      バックエンド
    </label>
    <label>
      <input type="checkbox" name="interest" value="design" />
      デザイン
    </label>
  </fieldset>

  {/* セレクトボックス */}
  <div>
    <label htmlFor="country">国</label>
    <select id="country" required>
      <option value="">選択してください</option>
      <option value="jp">日本</option>
      <option value="us">アメリカ</option>
      <option value="uk">イギリス</option>
    </select>
  </div>

  {/* テキストエリア */}
  <div>
    <label htmlFor="message">メッセージ</label>
    <textarea
      id="message"
      rows={5}
      aria-describedby="message-help"
    />
    <p id="message-help" className="text-sm text-gray-600">
      500文字以内で入力してください
    </p>
  </div>

  <button type="submit">送信</button>
</form>
```

### その他の重要な要素

#### figure と figcaption

画像や図表にキャプションを付ける際に使用します。

```tsx
// ✅ 画像とキャプション
<figure>
  <img
    src="/chart.png"
    alt="2025年売上推移グラフ"
  />
  <figcaption>
    図1: 2025年の月別売上推移。右肩上がりの傾向が見られる。
  </figcaption>
</figure>

// ✅ コードブロックとキャプション
<figure>
  <pre><code>{`
function greet(name: string) {
  return \`Hello, \${name}!\`;
}
  `}</code></pre>
  <figcaption>リスト1: TypeScriptの関数定義</figcaption>
</figure>
```

#### time

日付・時刻を表します。

```tsx
// ✅ 日付
<time datetime="2026-01-24">2026年1月24日</time>

// ✅ 時刻
<time datetime="14:30">午後2時30分</time>

// ✅ 日時
<time datetime="2026-01-24T14:30:00+09:00">
  2026年1月24日 14:30
</time>

// ✅ 期間
<p>
  イベント期間:
  <time datetime="2026-02-01">2月1日</time>
  〜
  <time datetime="2026-02-03">2月3日</time>
</p>
```

#### address

連絡先情報を表します。

```tsx
// ✅ 記事の著者情報
<article>
  <h1>記事タイトル</h1>
  <p>記事本文...</p>
  <footer>
    <address>
      著者: <a href="mailto:yamada@example.com">山田太郎</a>
    </address>
  </footer>
</article>

// ✅ 会社の連絡先
<footer>
  <address>
    株式会社Example<br />
    〒100-0001<br />
    東京都千代田区千代田1-1<br />
    電話: <a href="tel:+81-3-1234-5678">03-1234-5678</a><br />
    メール: <a href="mailto:info@example.com">info@example.com</a>
  </address>
</footer>
```

## divとspanの使い分け

divとspanは汎用的な要素ですが、**意味を持つ要素がある場合はそちらを優先**します。

```tsx
// ❌ 悪い例: divだらけ
<div>
  <div>ページタイトル</div>
  <div>
    <div>ナビゲーション</div>
    <div>リンク1</div>
    <div>リンク2</div>
  </div>
  <div>メインコンテンツ</div>
</div>

// ✅ 良い例: セマンティックHTML
<header>
  <h1>ページタイトル</h1>
  <nav>
    <ul>
      <li><a href="/link1">リンク1</a></li>
      <li><a href="/link2">リンク2</a></li>
    </ul>
  </nav>
</header>
<main>
  <p>メインコンテンツ</p>
</main>
```

### divを使うべき場合

- レイアウト目的のコンテナ
- スタイリング目的のラッパー
- 他に適切な要素がない場合

```tsx
// ✅ レイアウトコンテナ
<div className="container mx-auto px-4">
  <article>
    <h1>記事タイトル</h1>
    <p>本文...</p>
  </article>
</div>

// ✅ グリッドレイアウト
<div className="grid grid-cols-3 gap-4">
  <div>カード1</div>
  <div>カード2</div>
  <div>カード3</div>
</div>
```

## まとめ

セマンティックHTMLは、アクセシビリティ、SEO、保守性を向上させる基盤です。主なポイント：

- ランドマーク要素でページ構造を定義
- 見出し階層を守る（h1→h2→h3、スキップしない）
- リストに適切な要素を使用（ul、ol、dl）
- テーブルにcaption、th、scopeを使用
- フォームにlabel、fieldset、legendを使用
- divとspanは、適切な要素がない場合のみ使用

### チェックリスト

- ページにmain、header、footer等のランドマークがある
- 見出し階層が論理的
- ナビゲーションにnav要素を使用
- リストにul/ol要素を使用
- テーブルにcaptionとthがある
- フォームに全てのlabelがある

## 次のステップ

次章では、ARIA属性について詳しく解説します。セマンティックHTMLで表現できない部分を補完する方法を学びます。
