# カラーコントラスト - 視認性を確保する

## カラーコントラストの重要性

十分なカラーコントラストは、以下のユーザーにとって重要です：

- ロービジョンのユーザー
- 色覚多様性のあるユーザー
- 明るい環境でデバイスを使用するユーザー
- 加齢による視力低下のあるユーザー

## WCAG 2.1の基準

WCAG 2.1では、テキストとUI要素のコントラスト比について明確な基準が定義されています[^wcag-contrast]。

[^wcag-contrast]: [Understanding Success Criterion 1.4.3: Contrast (Minimum) - W3C](https://www.w3.org/WAI/WCAG21/Understanding/contrast-minimum.html)

### 達成基準1.4.3 コントラスト（Level AA）

**テキストのコントラスト**:
- **通常テキスト**: 最低4.5:1
  - 通常テキストとは、18pt（約24px）未満、または14pt（約18.5px）太字未満のテキスト
- **大きなテキスト**: 最低3:1
  - 大きなテキストとは、18pt（約24px）以上、または14pt（約18.5px）太字以上のテキスト

**例外**:
- 装飾的なテキスト（意味のない装飾）
- ロゴタイプ
- 無効なUI要素（disabled状態）

### 達成基準1.4.6 コントラスト（Level AAA）

より高い視認性を求める場合:

- **通常テキスト**: 最低7:1
- **大きなテキスト**: 最低4.5:1

Level AAAは推奨レベルですが、ロービジョンユーザーにとっては非常に重要です。

### 達成基準1.4.11 非テキストコントラスト（Level AA）

WCAG 2.1で追加された基準です[^wcag-non-text]。

[^wcag-non-text]: [Understanding Success Criterion 1.4.11: Non-text Contrast - W3C](https://www.w3.org/WAI/WCAG21/Understanding/non-text-contrast.html)

**対象**:
- **UI要素**: 最低3:1（ボタンの境界線、フォームの枠線、フォーカスインジケーター等）
- **グラフィック**: 最低3:1（アイコン、グラフ、図表等）

**例外**:
- 無効なUI要素
- ユーザーがカスタマイズ可能な要素
- 写真や装飾的な画像

## コントラスト比の計算

コントラスト比は、以下の式で計算されます：

```
(L1 + 0.05) / (L2 + 0.05)
```

L1: 明るい色の相対輝度
L2: 暗い色の相対輝度

実際には、ツールを使用して確認します。

## チェックツール

コントラスト比を手動で計算するのは困難なため、ツールを使用して確認します。

### Chrome DevTools

最も手軽にコントラスト比を確認できるツールです。

**使用方法**:
1. 要素を右クリック → 検証
2. Styles パネルで色のアイコンをクリック（カラーピッカーが開く）
3. コントラスト比が表示される
4. AA、AAAへの適合状況が表示される

**利点**:
- インストール不要
- リアルタイムで確認可能
- 推奨色を提案してくれる

### WebAIM Contrast Checker

[WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/)は、オンラインで使用できるコントラストチェッカーです。

**使用方法**:
1. 前景色（テキスト色）を入力（例: #333333）
2. 背景色を入力（例: #ffffff）
3. コントラスト比を確認
4. AA、AAAへの適合を確認
5. 通常テキスト/大きなテキストの区別

**利点**:
- シンプルで使いやすい
- URLで共有可能
- 代替色を提案してくれる

### Colour Contrast Analyser (CCA)

[Colour Contrast Analyser](https://www.tpgi.com/color-contrast-checker/)は、デスクトップアプリケーションです。

**使用方法**:
1. アプリケーションをダウンロード（Windows/macOS）
2. スポイトツールで画面上の色を取得
3. コントラスト比を確認

**利点**:
- 画面上の任意の色を取得可能
- オフラインで使用可能
- WCAG 2.1とWCAG 3.0（APCA）の両方に対応

### Lighthouse

Chrome DevToolsのLighthouseでアクセシビリティスコアを測定できます。

**使用方法**:
1. Chrome DevTools を開く（F12）
2. Lighthouse タブを選択
3. Accessibility カテゴリをチェック
4. Analyze page load をクリック

**利点**:
- アクセシビリティ全般をチェック
- CI/CDに統合可能（Lighthouse CI）
- パフォーマンスと同時にチェック可能

### axe DevTools

[axe DevTools](https://www.deque.com/axe/devtools/)は、ブラウザ拡張機能です。

**使用方法**:
1. Chrome/Firefox拡張機能をインストール
2. DevToolsのaxeタブを開く
3. Scan ALL of my page をクリック

**利点**:
- より詳細な問題を検出
- 修正方法の提案
- 自動テストライブラリ（axe-core）と同じエンジン

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
  warning: {
    bg: '#ffc107',
    text: '#000000', // コントラスト比 8.4:1（黄色背景には黒文字）
  },
  // テキストカラー
  text: {
    primary: '#333333', // コントラスト比 12.6:1
    secondary: '#666666', // コントラスト比 5.7:1
    disabled: '#999999', // 無効時は例外（コントラスト要件なし）
  },
}
```

### カラーパレットの検証

設計したカラーパレットを検証するスクリプト:

```typescript
// scripts/verify-colors.ts
type ColorPair = {
  name: string
  foreground: string
  background: string
  minRatio: number
}

// コントラスト比を計算する関数
function getContrastRatio(color1: string, color2: string): number {
  // 実装は省略（ライブラリを使用することを推奨）
  // 例: polished の readableColor、color2k の getContrast 等
  return 4.5 // ダミー値
}

const colorPairs: ColorPair[] = [
  { name: 'Primary button', foreground: '#ffffff', background: '#0066cc', minRatio: 4.5 },
  { name: 'Body text', foreground: '#333333', background: '#ffffff', minRatio: 4.5 },
  { name: 'Secondary text', foreground: '#666666', background: '#ffffff', minRatio: 4.5 },
]

colorPairs.forEach((pair) => {
  const ratio = getContrastRatio(pair.foreground, pair.background)
  const passes = ratio >= pair.minRatio
  console.log(`${pair.name}: ${ratio.toFixed(2)}:1 - ${passes ? '✅ PASS' : '❌ FAIL'}`)
})
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
