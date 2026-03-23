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

ジェネリクス（Go 1.18+）は、同じロジックを異なる型に適用する仕組みです。型パラメータの構文は `[T constraint]` で、`T` が型パラメータ名、`constraint` が T に課す条件です。

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

制約は型パラメータに課す「型が満たすべき条件」です。

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
// Number かつ Stringer を満たす型に限定する例
type PrintableNumber interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 | ~float32 | ~float64
    String() string
}
// 使用例: type Score int に func (s Score) String() string を定義すれば適合する
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

:::details 解答例を見る

```go
package main

import "fmt"

// Unique は重複を除去したスライスを返す。
// 制約を comparable にすることで、mapのキーとして使える型に限定する。
// 元のスライスの順序は保持される（最初に出現した順）。
func Unique[T comparable](s []T) []T {
	seen := make(map[T]struct{}, len(s)) // struct{} はメモリゼロ
	result := make([]T, 0, len(s))
	for _, v := range s {
		if _, exists := seen[v]; !exists {
			seen[v] = struct{}{}
			result = append(result, v)
		}
	}
	return result
}

func main() {
	// int で動作確認
	nums := Unique([]int{1, 2, 3, 2, 1, 4})
	fmt.Println(nums) // [1 2 3 4]

	// string で動作確認（同じ関数が型推論で使える）
	words := Unique([]string{"a", "b", "a", "c", "b"})
	fmt.Println(words) // [a b c]

	// 空スライス
	empty := Unique([]int{})
	fmt.Println(empty) // []

	// 全て同じ値
	same := Unique([]int{5, 5, 5})
	fmt.Println(same) // [5]
}

// 出力例:
// [1 2 3 4]
// [a b c]
// []
// [5]
```

ポイント: `map[T]struct{}` はGoで集合（set）を表現する慣用パターン。`struct{}` はサイズ0なので `map[T]bool` よりメモリ効率が良い。制約に `comparable` を使うことで、`==` 演算が可能な型（int, string, struct等）のみに限定され、mapのキーとして安全に使える。

:::

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

:::details 解答例を見る

```go
package main

import "fmt"

// GroupBy はスライスの要素をキー関数の戻り値でグルーピングする。
// T any: 要素の型は何でもよい（メソッドも演算も不要）。
// K comparable: キーはmapのキーになるため comparable が必要。
func GroupBy[T any, K comparable](s []T, keyFn func(T) K) map[K][]T {
	result := make(map[K][]T)
	for _, v := range s {
		key := keyFn(v) // キー関数で要素からキーを抽出
		result[key] = append(result[key], v)
	}
	return result
}

type User struct {
	Name string
	Role string
}

func main() {
	users := []User{
		{"Alice", "admin"},
		{"Bob", "user"},
		{"Charlie", "admin"},
		{"Dave", "user"},
	}

	// ロール別にグルーピング
	byRole := GroupBy(users, func(u User) string { return u.Role })
	for role, members := range byRole {
		fmt.Printf("%s: %v\n", role, members)
	}
	// 出力例:
	// admin: [{Alice admin} {Charlie admin}]
	// user: [{Bob user} {Dave user}]

	fmt.Println("---")

	// int スライスを偶奇でグルーピング（異なる型でも同じ関数が使える）
	nums := []int{1, 2, 3, 4, 5, 6}
	byParity := GroupBy(nums, func(n int) string {
		if n%2 == 0 {
			return "even"
		}
		return "odd"
	})
	fmt.Println(byParity)
	// 出力例: map[even:[2 4 6] odd:[1 3 5]]
}
```

ポイント: 2つの型パラメータ `[T any, K comparable]` を使い分けるのがこの問題の核心。要素 `T` は制約不要（`any`）だが、キー `K` はmapのキーになるため `comparable` が必要。キー抽出ロジックを関数引数で受け取ることで、どんなグルーピング条件にも対応できる汎用的な関数になる。

:::

### 練習3: ジェネリックな Result 型

エラーまたは値を持つ `Result[T]` 型と、`Ok`/`Err` コンストラクタ、`Unwrap()`/`UnwrapOr(def T)` メソッドを実装してみよう。

```go
// Ok(42).Unwrap()                        => 42, nil
// Err[int](errors.New("x")).UnwrapOr(0)  => 0
```

:::details 解答例を見る

```go
package main

import (
	"errors"
	"fmt"
)

// Result はエラーまたは値を持つジェネリック型。
// 成功（Ok）と失敗（Err）を1つの型で表現する。
type Result[T any] struct {
	value T
	err   error
	ok    bool // 成功かどうかを明示的に持つ（ゼロ値と区別するため）
}

// Ok は成功の Result を返すコンストラクタ
func Ok[T any](value T) Result[T] {
	return Result[T]{value: value, ok: true}
}

// Err は失敗の Result を返すコンストラクタ。
// 型パラメータは推論できないため、Err[int](...) のように明示する。
func Err[T any](err error) Result[T] {
	return Result[T]{err: err, ok: false}
}

// IsOk は成功かどうかを返す
func (r Result[T]) IsOk() bool {
	return r.ok
}

// Unwrap は成功なら値を、失敗ならエラーを返す
func (r Result[T]) Unwrap() (T, error) {
	if r.ok {
		return r.value, nil
	}
	var zero T
	return zero, r.err
}

// UnwrapOr は成功なら値を、失敗ならデフォルト値を返す
func (r Result[T]) UnwrapOr(def T) T {
	if r.ok {
		return r.value
	}
	return def
}

// Map は成功の場合に値を変換する。失敗の場合はそのままエラーを伝播する。
func Map[T, U any](r Result[T], f func(T) U) Result[U] {
	if r.ok {
		return Ok(f(r.value))
	}
	return Err[U](r.err)
}

func main() {
	// Ok の場合
	r1 := Ok(42)
	val, err := r1.Unwrap()
	fmt.Printf("Unwrap: %d, %v\n", val, err)
	// 出力: Unwrap: 42, <nil>

	// Err の場合
	r2 := Err[int](errors.New("something went wrong"))
	fmt.Printf("IsOk: %v\n", r2.IsOk())
	// 出力: IsOk: false

	fmt.Printf("UnwrapOr: %d\n", r2.UnwrapOr(0))
	// 出力: UnwrapOr: 0

	// Map で値を変換
	r3 := Ok(10)
	r4 := Map(r3, func(n int) string {
		return fmt.Sprintf("value=%d", n)
	})
	s, _ := r4.Unwrap()
	fmt.Printf("Map result: %s\n", s)
	// 出力: Map result: value=10

	// Err に Map を適用してもエラーが伝播する
	r5 := Err[int](errors.New("fail"))
	r6 := Map(r5, func(n int) string {
		return fmt.Sprintf("value=%d", n) // この関数は呼ばれない
	})
	_, err = r6.Unwrap()
	fmt.Printf("Map on Err: %v\n", err)
	// 出力: Map on Err: fail
}
```

ポイント: `Result[T]` のようなジェネリック型では、成功/失敗を `ok bool` フィールドで明示的に管理する。ゼロ値（例: `int` の `0`）が正当な成功値の場合があるため、`value` がゼロ値かどうかでは判定できない。`Err[int](...)` のように型パラメータを明示する必要がある場面は、型推論の限界を理解する良い例。`Map` 関数はメソッドではなく関数として定義する（Goではメソッドに新しい型パラメータを追加できないため）。

:::
