---
title: "useState と useEffect の深い理解"
---

# Chapter 1: useState と useEffect の深い理解

## この章で学べること

この章では、Reactの最も基本的で重要な2つのHooks、`useState`と`useEffect`を深く理解します。

- ✅ useStateのバッチ更新と関数型更新の仕組み
- ✅ useEffectの実行タイミングと依存配列の正しい理解
- ✅ クリーンアップ関数の必要性と使い方
- ✅ 実務で即使える実践例5つ

**前提知識**: Reactの基礎（JSX、コンポーネント）、useState/useEffectを使ったことがある

**所要時間**: 30-40分

---

## 1. useState の深い理解

### 1.1 useState の基本的な使い方

まず、基本的な使い方をおさらいしましょう。

```typescript
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  );
}
```

これは誰でも知っている基本的な使い方です。しかし、実務ではもっと深い理解が必要になります。

---

### 1.2 バッチ更新の仕組み

**重要**: Reactは複数の状態更新を自動的にバッチ処理します。

```typescript
function BatchExample() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  const handleClick = () => {
    // これら2つの更新は1回のレンダリングにまとめられる
    setCount(count + 1);
    setFlag(!flag);
    // ここでは再レンダリングは1回だけ
  };

  console.log('レンダリング実行'); // クリックごとに1回だけ表示される

  return (
    <div>
      <p>Count: {count}</p>
      <p>Flag: {flag ? 'ON' : 'OFF'}</p>
      <button onClick={handleClick}>更新</button>
    </div>
  );
}
```

**なぜこれが重要か？**

（TODO: バッチ更新のメリットを説明）

---

### 1.3 関数型更新（Functional Update）

**問題**: 以下のコードは期待通りに動作しません。

```typescript
function Problem() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1); // count = 0
    setCount(count + 1); // count = 0（まだ更新されていない）
    setCount(count + 1); // count = 0（まだ更新されていない）
    // 結果: count = 1（期待は3だが...）
  };

  return <button onClick={handleClick}>+3</button>;
}
```

**解決策**: 関数型更新を使う

```typescript
function Solution() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(prev => prev + 1); // prev = 0, 次は 1
    setCount(prev => prev + 1); // prev = 1, 次は 2
    setCount(prev => prev + 1); // prev = 2, 次は 3
    // 結果: count = 3 ✅
  };

  return <button onClick={handleClick}>+3</button>;
}
```

**ルール**: 前の状態に依存する更新は、必ず関数型更新を使う。

---

### 1.4 オブジェクト・配列の更新

**重要**: オブジェクトや配列は、新しい参照を作成して更新する必要があります。

```typescript
// ❌ 間違い: 直接変更
const handleAdd = () => {
  user.name = 'New Name'; // これは動作しない
  setUser(user);
};

// ✅ 正解: 新しいオブジェクトを作成
const handleAdd = () => {
  setUser({ ...user, name: 'New Name' });
};

// 配列の場合
const [items, setItems] = useState<string[]>([]);

// ❌ 間違い
items.push('new'); // これは動作しない
setItems(items);

// ✅ 正解
setItems([...items, 'new']);
```

（TODO: なぜ新しい参照が必要か、詳しく説明）

---

### 1.5 遅延初期化（Lazy Initialization）

**問題**: 初期値の計算が重い場合

```typescript
// ❌ これは毎回計算される（無駄）
const [value, setValue] = useState(expensiveCalculation());
```

**解決策**: 関数を渡す

```typescript
// ✅ 初回レンダリング時のみ計算される
const [value, setValue] = useState(() => expensiveCalculation());
```

（TODO: 実際の例を示す）

---

## 2. useEffect の深い理解

### 2.1 useEffect の基本

```typescript
useEffect(() => {
  // 副作用の処理
  console.log('実行されました');
}, [依存配列]);
```

**3つの使い方**:

1. **依存配列なし**: 毎回実行
2. **空の依存配列 `[]`**: マウント時のみ実行
3. **依存配列あり `[count]`**: countが変わったときのみ実行

（TODO: それぞれの使い分けを詳しく説明）

---

### 2.2 実行タイミングの理解

```typescript
function EffectTiming() {
  const [count, setCount] = useState(0);

  console.log('1. レンダリング開始');

  useEffect(() => {
    console.log('3. useEffect実行（レンダリング後）');
  });

  console.log('2. レンダリング中');

  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}
```

**実行順序**:
1. レンダリング開始
2. レンダリング中
3. useEffect実行（DOM更新後）

（TODO: useLayoutEffectとの違いを説明）

---

### 2.3 クリーンアップ関数

**なぜ必要か？**: メモリリークを防ぐため

```typescript
useEffect(() => {
  // セットアップ
  const timer = setInterval(() => {
    console.log('Tick');
  }, 1000);

  // クリーンアップ（コンポーネントがアンマウントされる前に実行）
  return () => {
    clearInterval(timer);
  };
}, []);
```

**クリーンアップが必要な例**:
- タイマー（setInterval, setTimeout）
- イベントリスナー（addEventListener）
- WebSocket接続
- サブスクリプション

（TODO: 各例を詳しく説明）

---

### 2.4 依存配列の正しい理解

**ルール**: useEffect内で使用するすべての値を依存配列に含める

```typescript
// ❌ 間違い: countを依存配列に含めていない
useEffect(() => {
  console.log(count); // ESLintが警告を出す
}, []);

// ✅ 正解
useEffect(() => {
  console.log(count);
}, [count]);
```

**よくある間違い**:

（TODO: よくある間違いと解決策を説明）

---

## 3. 実践例

### 例1: フォーム入力の管理

```typescript
function FormExample() {
  const [formData, setFormData] = useState({
    name: '',
    email: ''
  });

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const { name, value } = e.target;
    setFormData(prev => ({
      ...prev,
      [name]: value
    }));
  };

  return (
    <form>
      <input
        name="name"
        value={formData.name}
        onChange={handleChange}
        placeholder="名前"
      />
      <input
        name="email"
        value={formData.email}
        onChange={handleChange}
        placeholder="メール"
      />
    </form>
  );
}
```

（TODO: 詳しい解説を追加）

---

### 例2: APIからのデータ取得

```typescript
function DataFetching() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchData = async () => {
      try {
        setLoading(true);
        const response = await fetch('https://api.example.com/data');
        const json = await response.json();
        setData(json);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchData();
  }, []); // マウント時のみ実行

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  return <div>{JSON.stringify(data)}</div>;
}
```

（TODO: 詳しい解説を追加）

---

### 例3: タイマー・インターバル

（TODO: 実装例を追加）

---

### 例4: イベントリスナーの登録/解除

（TODO: 実装例を追加）

---

### 例5: ローカルストレージとの同期

（TODO: 実装例を追加）

---

## まとめ

この章では、`useState`と`useEffect`について以下を学びました：

- ✅ useStateのバッチ更新と関数型更新
- ✅ useEffectの実行タイミングとクリーンアップ
- ✅ 依存配列の正しい理解
- ✅ 実務で使える実践例5つ

**次の章では**: カスタムHooksの設計パターンを学び、再利用可能なロジックを抽出する方法を習得します。

---

## 参考リンク

- [React公式ドキュメント - useState](https://react.dev/reference/react/useState)
- [React公式ドキュメント - useEffect](https://react.dev/reference/react/useEffect)

---

**執筆メモ（公開時に削除）**:
- [ ] バッチ更新のメリットを詳しく説明
- [ ] useLayoutEffectとの違いを追加
- [ ] 例3-5の実装を追加
- [ ] 各セクションに図解を追加（可能なら）
- [ ] CodeSandboxのデモリンクを追加
- [ ] 実際のコードで動作確認
