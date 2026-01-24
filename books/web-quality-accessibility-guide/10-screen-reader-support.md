# スクリーンリーダー対応 - 音声で理解できるようにする

## スクリーンリーダーとは

スクリーンリーダーは、画面の内容を音声で読み上げるソフトウェアです。視覚障害のあるユーザーが主に使用します。

### 主要なスクリーンリーダー

- **NVDA（Windows）**: 無料、オープンソース
- **JAWS（Windows）**: 有料、高機能
- **VoiceOver（macOS/iOS）**: 標準搭載
- **TalkBack（Android）**: 標準搭載
- **ORCA（Linux）**: 無料、オープンソース

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

## テスト方法

### VoiceOverでのテスト

1. Command + F5 で起動
2. VO + → で要素を順番に確認
3. VO + U でローターを開き、見出し・リンク・フォーム要素を確認
4. 全てのコンテンツが適切に読み上げられるか確認

### NVDAでのテスト

1. Control + Alt + N で起動
2. ↓ で要素を順番に確認
3. NVDA + F7 で要素リストを確認
4. H キーで見出しジャンプ

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
