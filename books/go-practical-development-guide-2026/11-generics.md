---
title: "ジェネリクス"
---

# Chapter 11: ジェネリクス

> 型パラメータと制約 — 型安全な汎用コード

## この章で学ぶこと

1. **型パラメータ**の構文とジェネリック関数の書き方
2. **制約** — `any`, `comparable`, `cmp.Ordered`, カスタム制約
3. **ジェネリック型** — 型安全なデータ構造の定義
4. **実践パターン** — スライス操作、標準ライブラリ活用
5. **使いどころと避けどころ** — ジェネリクス vs インターフェース

---

## 1. 型パラメータの基本

ジェネリクス（Go 1.18+）は、同じロジックを異なる型に適用する仕組みだ。型パラメータの構文は `[T constraint]` で、`T` が型パラメータ名、`constraint` が T に課す条件。

```go
package main

import (
    "cmp"
    "fmt"
)

// 導入前: MaxInt, MaxFloat, MaxString... と型ごとに複製していた
// 導入後: 一つの関数で全ての順序付き型に対応
func Max[T cmp.Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}

func Clamp[T cmp.Ordered](val, lo, hi T) T {
    return Max(lo, min(val, hi)) // min は Go 1.21+ のビルトイン
}

func main() {
    fmt.Println(Max(3, 7))              // 7
    fmt.Println(Max(3.14, 2.71))        // 3.14
    fmt.Println(Max("apple", "banana")) // banana
    fmt.Println(Clamp(150, 0, 100))     // 100
}
```

**型推論**により、`Max[int](3, 7)` と明示しなくても引数から `T = int` と自動推論される。異なる型を混ぜると（`Max(3, 7.0)`）コンパイルエラーになる。

:::message
Go 1.21 以降、`min` と `max` はビルトイン関数として使えます。単純な最大・最小には自作関数よりビルトインを使いましょう。
:::

---

## 2. 制約（Constraints）

制約は型パラメータに課す「型が満たすべき条件」だ。

| 制約 | 許容される型 | 使える演算 | 用途 |
|------|------------|-----------|------|
| `any` | 全ての型 | なし | コンテナ、ラッパー |
| `comparable` | 比較可能な型 | `==`, `!=` | mapのキー、重複排除 |
| `cmp.Ordered` | 順序付き型 | `<`, `>`, `<=`, `>=` | ソート、最大最小 |

インターフェースで独自の制約も作れる。

```go
// 型集合ベース: チルダ(~)で「基底型がintの全ての型」を含める
type Number interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
    ~float32 | ~float64
}

type Score int // ~int に合致する（チルダなしの int だと不一致）

func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}

// メソッドベース: 特定のメソッドを要求する制約
type Validatable interface {
    Validate() error
}

func SaveAll[T Validatable](items []T) error {
    for i, item := range items {
        if err := item.Validate(); err != nil {
            return fmt.Errorf("item[%d]: %w", i, err)
        }
    }
    return nil
}

// 複合制約: 型集合 + メソッドの組み合わせも可能
type OrderedStringer interface {
    cmp.Ordered
    String() string
}
```

**チルダ `~` のポイント**: `int` と書くと `int` 型のみ。`~int` と書くと `type Score int` のような派生型も含まれる。

---

## 3. ジェネリック型

ジェネリック型で型安全なデータ構造を定義できる。

```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T // ゼロ値の取得方法
        return zero, false
    }
    last := len(s.items) - 1
    item := s.items[last]
    s.items = s.items[:last]
    return item, true
}

func (s *Stack[T]) Len() int { return len(s.items) }

func main() {
    s := &Stack[int]{}
    s.Push(1)
    s.Push(2)
    val, _ := s.Pop() // 2
}
```

**メソッドの制限**: メソッドに新しい型パラメータは追加できない。型定義で宣言した型パラメータのみ使える。

```go
// NG: メソッドに新しい型パラメータ U を追加
// func (s *Stack[T]) Map[U any](f func(T) U) *Stack[U] { ... }

// OK: 関数として定義する
func MapStack[T, U any](s *Stack[T], f func(T) U) *Stack[U] {
    result := &Stack[U]{}
    for _, item := range s.items {
        result.Push(f(item))
    }
    return result
}
```

---

## 4. 実践パターン

### スライス操作ユーティリティ

```go
func Map[T, U any](s []T, f func(T) U) []U {
    result := make([]U, len(s))
    for i, v := range s {
        result[i] = f(v)
    }
    return result
}

func Filter[T any](s []T, pred func(T) bool) []T {
    var result []T
    for _, v := range s {
        if pred(v) {
            result = append(result, v)
        }
    }
    return result
}

func Reduce[T, U any](s []T, init U, f func(U, T) U) U {
    acc := init
    for _, v := range s {
        acc = f(acc, v)
    }
    return acc
}

// 使用例
nums := []int{1, 2, 3, 4, 5}
doubled := Map(nums, func(n int) int { return n * 2 })       // [2,4,6,8,10]
evens := Filter(nums, func(n int) bool { return n%2 == 0 })  // [2,4]
sum := Reduce(nums, 0, func(acc, n int) int { return acc+n }) // 15
```

### 標準ライブラリの活用

自作する前に `slices`、`maps`、`cmp` パッケージを確認しよう。

```go
import ("cmp"; "slices")

nums := []int{3, 1, 4, 1, 5}
slices.Sort(nums)                                    // [1,1,3,4,5]
slices.Contains([]string{"a", "b", "c"}, "b")        // true
slices.Max(nums)                                     // 5

// カスタムソート
type User struct { Name string; Age int }
users := []User{{"Charlie", 30}, {"Alice", 25}, {"Bob", 35}}
slices.SortFunc(users, func(a, b User) int {
    return cmp.Compare(a.Age, b.Age)
})
// [{Alice 25}, {Charlie 30}, {Bob 35}]

// cmp.Or: 最初の非ゼロ値を返す（デフォルト値パターン）
name := cmp.Or(userName, "anonymous")
```

### リポジトリパターン

制約にインターフェースを使い、ドメイン固有のジェネリック型を作る例。

```go
type Entity interface { GetID() string }

type Repository[T Entity] interface {
    FindByID(id string) (T, error)
    Save(entity T) error
}

// インメモリ実装
type InMemoryRepo[T Entity] struct { store map[string]T }

func NewInMemoryRepo[T Entity]() *InMemoryRepo[T] {
    return &InMemoryRepo[T]{store: make(map[string]T)}
}

func (r *InMemoryRepo[T]) Save(e T) error { r.store[e.GetID()] = e; return nil }

func (r *InMemoryRepo[T]) FindByID(id string) (T, error) {
    if e, ok := r.store[id]; ok {
        return e, nil
    }
    var zero T
    return zero, fmt.Errorf("not found: %s", id)
}

// User が GetID() を実装していれば Repository[User] として使える
type User struct { ID, Name string }
func (u User) GetID() string { return u.ID }
```

---

## 5. 使いどころと避けどころ

### 判断基準

| 場面 | 推奨 | 理由 |
|------|------|------|
| コレクション操作（Map, Filter） | ジェネリクス | 同じアルゴリズムを全ての型に適用 |
| データ構造（Stack, Queue） | ジェネリクス | 型安全なコンテナ |
| DB接続の抽象化 | インターフェース | 実装が異なる |
| HTTPハンドラ | インターフェース | http.Handler パターン |
| `fmt.Println` 的な関数 | `any` 引数 | ジェネリクス不要 |

**原則**: 同じアルゴリズムを異なる型に適用 → ジェネリクス。異なる実装を同じ振る舞いに抽象化 → インターフェース。

### アンチパターン: 不要なジェネリクス化

```go
// NG: any しか使わないなら interface{} で十分
func PrintValue[T any](v T) { fmt.Println(v) }

// OK
func PrintValue(v any) { fmt.Println(v) }
```

**2つ以上の具体型で使われるか？** を自問しよう。1つの型にしか使われないなら具体型を直接使う。

### アンチパターン: 制約が複雑すぎる

```go
// NG: インラインで長い制約を書く
type Store[K comparable, V interface {
    ~int | ~string; fmt.Stringer
}] struct { data map[K]V }

// OK: 制約を分離して名前をつける
type ValueType interface {
    ~int | ~string
    fmt.Stringer
}
type Store[K comparable, V ValueType] struct { data map[K]V }
```

---

## まとめ

| 概念 | 要点 |
|------|------|
| 型パラメータ | `[T constraint]` 構文で宣言。引数から型推論されるため明示不要な場合が多い |
| 制約（Constraints） | `any`（全型）、`comparable`（`==`可）、`cmp.Ordered`（順序比較可）の3段階が基本 |
| チルダ `~` | `~int` で `type Score int` のような派生型も許容。なしだと厳密一致のみ |
| ジェネリック型 | `Stack[T any]` のように型安全なデータ構造を定義。メソッドに新しい型パラメータは追加不可 |
| 実践パターン | `Map`/`Filter`/`Reduce` などのスライス操作、リポジトリパターンで効果的 |
| 標準ライブラリ | `slices`・`maps`・`cmp` パッケージを優先活用。自作前に確認する |
| ジェネリクス vs インターフェース | 同じアルゴリズムを異なる型に適用→ジェネリクス、異なる実装を抽象化→インターフェース |
| アンチパターン | `any` しか使わないなら `any` 引数で十分。1つの型にしか使わないなら具体型を直接使う |

---

## やってみよう！

### 練習1: ジェネリックな Unique 関数

重複を除去する `Unique` 関数を実装してみよう。

```go
// ヒント: seen マップで既出チェック。制約は comparable。
func Unique[T comparable](s []T) []T {
    // TODO: 実装してみよう
}

// テスト
// Unique([]int{1, 2, 3, 2, 1, 4}) => [1, 2, 3, 4]
// Unique([]string{"a", "b", "a"}) => ["a", "b"]
```

### 練習2: ジェネリックな GroupBy 関数

スライスの要素をキー関数でグルーピングする関数を実装してみよう。

```go
// ヒント: T any と K comparable の2つの型パラメータが必要
func GroupBy[T any, K comparable](s []T, keyFn func(T) K) map[K][]T {
    // TODO: 実装してみよう
}

// テスト
type User struct { Name, Role string }
users := []User{{"Alice", "admin"}, {"Bob", "user"}, {"Charlie", "admin"}}
byRole := GroupBy(users, func(u User) string { return u.Role })
// map["admin":[{Alice admin} {Charlie admin}] "user":[{Bob user}]]
```

### 練習3: ジェネリックな Result 型

エラーまたは値を持つ `Result[T]` 型と、`Ok`/`Err` コンストラクタ、`Unwrap()`/`UnwrapOr(def T)` メソッドを実装してみよう。

```go
// Ok(42).Unwrap()                        => 42, nil
// Err[int](errors.New("x")).UnwrapOr(0)  => 0
```
