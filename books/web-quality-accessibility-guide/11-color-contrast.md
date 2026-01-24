# カラーコントラスト - 視認性を確保する

## カラーコントラストの重要性

十分なカラーコントラストは、以下のユーザーにとって重要です：

- ロービジョンのユーザー
- 色覚多様性のあるユーザー
- 明るい環境でデバイスを使用するユーザー
- 加齢による視力低下のあるユーザー

## WCAG 2.1の基準

### 達成基準1.4.3 コントラスト（Level AA）

- **通常テキスト**: 最低4.5:1
- **大きなテキスト**: 最低3:1（18pt以上または14pt太字以上）

### 達成基準1.4.6 コントラスト（Level AAA）

- **通常テキスト**: 最低7:1
- **大きなテキスト**: 最低4.5:1

### 達成基準1.4.11 非テキストコントラスト（Level AA）

- **UI要素**: 最低3:1（ボタン、フォーム、フォーカスインジケーター等）
- **グラフィック**: 最低3:1（アイコン、グラフ等）

## コントラスト比の計算

コントラスト比は、以下の式で計算されます：

```
(L1 + 0.05) / (L2 + 0.05)
```

L1: 明るい色の相対輝度
L2: 暗い色の相対輝度

実際には、ツールを使用して確認します。

## チェックツール

### Chrome DevTools

1. 要素を検査
2. Styles パネルでカラーピッカーを開く
3. コントラスト比が表示される

### WebAIM Contrast Checker

https://webaim.org/resources/contrastchecker/

- 前景色と背景色を入力
- コントラスト比を確認
- AA、AAAへの適合を確認

### Lighthouse

Chrome DevTools > Lighthouse > Accessibility

## 実装例

### テキストのコントラスト

```css
/* ✅ 良い例: 十分なコントラスト */
.text {
  color: #333333; /* コントラスト比 12.6:1 */
  background: #ffffff;
}

.text-on-blue {
  color: #ffffff; /* コントラスト比 4.5:1 */
  background: #0066cc;
}

/* ❌ 悪い例: 不十分なコントラスト */
.text-bad {
  color: #999999; /* コントラスト比 2.8:1 */
  background: #ffffff;
}
```

### ボタンのコントラスト

```css
/* ✅ 良い例 */
button {
  background: #0066cc;
  color: #ffffff; /* コントラスト比 4.5:1 */
  border: 2px solid #004080; /* UI要素のコントラスト比 3.5:1 */
}

button:focus-visible {
  outline: 3px solid #ff6600; /* フォーカスインジケーター 4.2:1 */
  outline-offset: 2px;
}
```

### フォームのコントラスト

```css
/* ✅ 良い例 */
input {
  border: 2px solid #666666; /* コントラスト比 5.7:1 */
  color: #333333;
  background: #ffffff;
}

input:focus {
  border-color: #0066cc; /* コントラスト比 3.5:1 */
  outline: none;
}
```

## 色だけで情報を伝えない

WCAG 2.1 達成基準1.4.1により、色だけで情報を伝えてはいけません。

```tsx
// ❌ 悪い例: 色だけ
<p className="text-red-600">エラー</p>
<p className="text-green-600">成功</p>

// ✅ 良い例: アイコンとテキスト
<div className="flex items-center gap-2 text-red-600">
  <XCircleIcon />
  <span>エラー: 入力内容を確認してください</span>
</div>

<div className="flex items-center gap-2 text-green-600">
  <CheckCircleIcon />
  <span>送信に成功しました</span>
</div>
```

## カラーパレット設計

アクセシビリティを考慮したカラーパレット設計：

```typescript
// colors.ts
export const colors = {
  // プライマリカラー（AAA準拠）
  primary: {
    bg: '#0066cc',
    text: '#ffffff', // コントラスト比 4.5:1
  },
  // セカンダリカラー
  secondary: {
    bg: '#666666',
    text: '#ffffff', // コントラスト比 5.7:1
  },
  // ステータスカラー
  success: {
    bg: '#28a745',
    text: '#ffffff', // コントラスト比 3.4:1（大きなテキストのみ）
  },
  error: {
    bg: '#dc3545',
    text: '#ffffff', // コントラスト比 4.5:1
  },
  // テキストカラー
  text: {
    primary: '#333333', // コントラスト比 12.6:1
    secondary: '#666666', // コントラスト比 5.7:1
    disabled: '#999999', // 無効時は例外（コントラスト要件なし）
  },
}
```

## ダークモード対応

```css
/* ライトモード */
:root {
  --color-bg: #ffffff;
  --color-text: #333333; /* コントラスト比 12.6:1 */
  --color-primary: #0066cc;
}

/* ダークモード */
@media (prefers-color-scheme: dark) {
  :root {
    --color-bg: #1a1a1a;
    --color-text: #e0e0e0; /* コントラスト比 11.6:1 */
    --color-primary: #4da6ff;
  }
}
```

## まとめ

カラーコントラストは、視認性を確保するための重要な要素です。

### チェックリスト

- 通常テキスト: 4.5:1以上
- 大きなテキスト: 3:1以上
- UI要素: 3:1以上
- 色だけで情報を伝えていない
- ツールで確認済み

## 次のステップ

次章では、アクセシビリティテストについて解説します。
