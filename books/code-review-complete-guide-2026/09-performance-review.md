---
title: "第9章 パフォーマンスレビュー"
---

# 第9章 パフォーマンスレビュー

本章ではパフォーマンスレビューについて、実践的な観点から詳しく解説します。パフォーマンスの問題は、初期段階では気づきにくいものの、システムの成長とともに深刻な影響を及ぼす可能性があります。コードレビューの段階で適切にチェックすることで、将来的なパフォーマンス問題を未然に防ぐことができます。

## 概要

パフォーマンスレビューは、コードの実行効率、メモリ使用量、スケーラビリティの観点から品質を評価するプロセスです。本章では、アルゴリズム複雑度の評価、データベースクエリの最適化、フロントエンドのレンダリング最適化など、具体的なチェックポイントとコード例を通じて、効果的なレビュー方法を学びます。

## アルゴリズム複雑度の評価

### Big O記法による計算量の確認

コードレビューでは、実装されたアルゴリズムの時間計算量と空間計算量を評価することが重要です。特にループ処理やデータ構造の選択は、パフォーマンスに大きな影響を与えます。

```typescript
// 悪い例: O(n²) の複雑度
function findDuplicates(arr: number[]): number[] {
  const duplicates: number[] = [];
  for (let i = 0; i < arr.length; i++) {
    for (let j = i + 1; j < arr.length; j++) {
      if (arr[i] === arr[j] && !duplicates.includes(arr[i])) {
        duplicates.push(arr[i]);
      }
    }
  }
  return duplicates;
}

// 良い例: O(n) の複雑度
function findDuplicatesOptimized(arr: number[]): number[] {
  const seen = new Set<number>();
  const duplicates = new Set<number>();

  for (const num of arr) {
    if (seen.has(num)) {
      duplicates.add(num);
    } else {
      seen.add(num);
    }
  }

  return Array.from(duplicates);
}
```

上記の例では、Setを活用することで計算量をO(n²)からO(n)に改善しています。レビュー時には、このようなデータ構造の選択が適切かどうかを確認しましょう。

### 不要な計算の排除

```python
# 悪い例: 毎回計算を実行
def calculate_total_price(items):
    total = 0
    for item in items:
        # 毎回税率を計算している
        tax_rate = get_tax_rate()  # データベースアクセスを伴う
        total += item.price * (1 + tax_rate)
    return total

# 良い例: 計算を一度だけ実行
def calculate_total_price_optimized(items):
    tax_rate = get_tax_rate()  # 一度だけ取得
    total = sum(item.price * (1 + tax_rate) for item in items)
    return total
```

ループ内で不変な値を繰り返し計算することは、パフォーマンスの無駄です。レビュー時には、ループ不変式をループ外に移動できないか確認しましょう。

## データベースクエリの最適化

### N+1問題の検出

N+1問題は、データベースアクセスにおける典型的なパフォーマンス問題です。一覧取得後に個別レコードを取得するループが発生すると、クエリ数が指数的に増加します。

```typescript
// 悪い例: N+1問題が発生
async function getUsersWithPosts() {
  const users = await db.user.findMany();

  // ユーザーごとにクエリが発行される
  const usersWithPosts = await Promise.all(
    users.map(async (user) => ({
      ...user,
      posts: await db.post.findMany({ where: { userId: user.id } })
    }))
  );

  return usersWithPosts;
}

// 良い例: JOINまたはincludeで一度に取得
async function getUsersWithPostsOptimized() {
  const users = await db.user.findMany({
    include: {
      posts: true
    }
  });

  return users;
}
```

ORMを使用している場合、eager loadingの機能を活用することで、N+1問題を解決できます。レビュー時には、データベースアクセスのパターンを確認しましょう。

### インデックスの活用確認

```sql
-- 悪い例: インデックスが使用されない
SELECT * FROM orders
WHERE YEAR(created_at) = 2024;

-- 良い例: インデックスが使用される
SELECT * FROM orders
WHERE created_at >= '2024-01-01'
  AND created_at < '2025-01-01';
```

関数をカラムに適用すると、インデックスが使用されない場合があります。レビュー時には、WHERE句やJOIN条件でのカラムの使い方を確認しましょう。

## フロントエンドのパフォーマンス最適化

### React コンポーネントの最適化

```typescript
// 悪い例: 不要な再レンダリングが発生
function UserList({ users }: { users: User[] }) {
  return (
    <div>
      {users.map((user) => (
        <UserCard
          key={user.id}
          user={user}
          onEdit={() => console.log('Edit', user.id)}
        />
      ))}
    </div>
  );
}

// 良い例: React.memoとuseCallbackで最適化
const UserCard = React.memo(({ user, onEdit }: { user: User; onEdit: () => void }) => (
  <div onClick={onEdit}>
    <h3>{user.name}</h3>
    <p>{user.email}</p>
  </div>
));

function UserListOptimized({ users }: { users: User[] }) {
  const handleEdit = useCallback((userId: string) => {
    console.log('Edit', userId);
  }, []);

  return (
    <div>
      {users.map((user) => (
        <UserCard
          key={user.id}
          user={user}
          onEdit={() => handleEdit(user.id)}
        />
      ))}
    </div>
  );
}
```

Reactコンポーネントでは、React.memoやuseCallback、useMemoを適切に使用することで、不要な再レンダリングを防ぎます。ただし、過度な最適化は逆効果になる場合もあるため、パフォーマンス計測を行いながら適用することが推奨されます。

### 遅延ローディングとコード分割

```typescript
// 悪い例: すべてのコンポーネントを即座にインポート
import Dashboard from './Dashboard';
import Settings from './Settings';
import Reports from './Reports';

function App() {
  const [page, setPage] = useState('dashboard');

  return (
    <div>
      {page === 'dashboard' && <Dashboard />}
      {page === 'settings' && <Settings />}
      {page === 'reports' && <Reports />}
    </div>
  );
}

// 良い例: 動的インポートで遅延ローディング
const Dashboard = lazy(() => import('./Dashboard'));
const Settings = lazy(() => import('./Settings'));
const Reports = lazy(() => import('./Reports'));

function AppOptimized() {
  const [page, setPage] = useState('dashboard');

  return (
    <Suspense fallback={<div>Loading...</div>}>
      <div>
        {page === 'dashboard' && <Dashboard />}
        {page === 'settings' && <Settings />}
        {page === 'reports' && <Reports />}
      </div>
    </Suspense>
  );
}
```

コード分割を活用することで、初期ロード時のバンドルサイズを削減できます。特にSPA（Single Page Application）では、この手法が効果的です。

## メモリリークの検出

### イベントリスナーの適切な管理

```typescript
// 悪い例: イベントリスナーが解除されない
function BadComponent() {
  useEffect(() => {
    const handleResize = () => {
      console.log('Window resized');
    };

    window.addEventListener('resize', handleResize);
    // クリーンアップ関数がない
  }, []);

  return <div>Content</div>;
}

// 良い例: クリーンアップ関数でリスナーを解除
function GoodComponent() {
  useEffect(() => {
    const handleResize = () => {
      console.log('Window resized');
    };

    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []);

  return <div>Content</div>;
}
```

イベントリスナーやタイマーは、適切に解除しないとメモリリークの原因になります。レビュー時には、useEffectのクリーンアップ関数が適切に実装されているか確認しましょう。

## キャッシュ戦略のレビュー

### メモ化の適切な使用

```typescript
// 悪い例: 毎回複雑な計算を実行
function ExpensiveComponent({ data }: { data: number[] }) {
  const result = data.reduce((acc, val) => acc + Math.sqrt(val), 0);

  return <div>{result}</div>;
}

// 良い例: useMemoで計算結果をキャッシュ
function OptimizedComponent({ data }: { data: number[] }) {
  const result = useMemo(
    () => data.reduce((acc, val) => acc + Math.sqrt(val), 0),
    [data]
  );

  return <div>{result}</div>;
}
```

計算コストの高い処理は、useMemoを使ってキャッシュすることで、パフォーマンスを改善できます。ただし、依存配列の設定を誤ると、バグの原因になるため注意が必要です。

## パフォーマンスレビューチェックリスト

レビュー時に確認すべき項目をチェックリスト形式でまとめます。

- [ ] アルゴリズムの時間計算量は適切か（O(n²)以上の処理がないか）
- [ ] ループ内で不要な計算や関数呼び出しがないか
- [ ] データベースクエリでN+1問題が発生していないか
- [ ] WHERE句やJOIN条件でインデックスが使用できるか
- [ ] Reactコンポーネントで不要な再レンダリングが発生していないか
- [ ] イベントリスナーやタイマーが適切にクリーンアップされているか
- [ ] 大きなバンドルサイズのコンポーネントがコード分割されているか
- [ ] 計算コストの高い処理がメモ化されているか
- [ ] キャッシュの有効期限や無効化戦略が適切に設計されているか
- [ ] 不要なメモリ確保やオブジェクト生成がないか

## まとめ

本章では、パフォーマンスレビューにおける重要なポイントを解説しました。

主なポイント:
- Big O記法を使ったアルゴリズム複雑度の評価
- N+1問題などデータベースクエリの最適化
- Reactコンポーネントの再レンダリング最適化
- メモリリークの検出と予防
- 適切なキャッシュ戦略の実装

パフォーマンス最適化は、早すぎる最適化を避けつつ、明らかな非効率を排除するバランスが重要です。レビュー時には、実装の複雑さとパフォーマンス改善のトレードオフを考慮しながら、フィードバックを提供しましょう。

## 参考文献

- [Big O Notation - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Glossary/Big_O_notation)
- [React Performance Optimization - React Official Documentation](https://react.dev/learn/render-and-commit)
- [Database Query Optimization - PostgreSQL Documentation](https://www.postgresql.org/docs/current/performance-tips.html)
- [Web Performance - Google Developers](https://developers.google.com/web/fundamentals/performance/why-performance-matters)
- [JavaScript Performance - V8 Dev](https://v8.dev/blog/cost-of-javascript-2019)
