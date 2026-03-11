---
title: "関数"
---

# Chapter 05: 関数

> 複数戻り値、名前付き戻り値、クロージャ — Go関数の全て

## この章で学べること

- ✅ 関数の定義と呼び出し
- ✅ 複数戻り値（Goの最大の特徴の一つ）
- ✅ 名前付き戻り値
- ✅ 可変長引数
- ✅ 関数は第一級オブジェクト
- ✅ クロージャ
- ✅ init 関数

---

## 関数の基本

```go
// 引数なし、戻り値なし
func greet() {
    fmt.Println("Hello!")
}

// 引数あり、戻り値あり
func add(a int, b int) int {
    return a + b
}

// 同じ型の引数は省略できる
func add(a, b int) int {
    return a + b
}

func main() {
    greet()
    result := add(3, 5)
    fmt.Println(result)  // 8
}
```

---

## 複数戻り値

Goの関数は**複数の値を返せます**。これはGoの最も重要な特徴の一つです。

```go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil
}

func main() {
    result, err := divide(10, 3)
    if err != nil {
        fmt.Println("エラー:", err)
        return
    }
    fmt.Printf("結果: %.2f\n", result)  // 結果: 3.33
}
```

### なぜ複数戻り値なのか

他の言語では例外（try/catch）でエラーを処理しますが、Goでは**戻り値としてエラーを返す**のが標準です。

```go
// Go のエラー処理パターン
value, err := someFunction()
if err != nil {
    // エラー処理
    return err
}
// 正常処理
```

このパターンはGoのコードで何百回も書くことになります。冗長に感じるかもしれませんが、「エラーが起きうる場所」が一目瞭然になる利点があります。

### 不要な戻り値を無視する

```go
// エラーだけ必要な場合
_, err := divide(10, 0)

// 結果だけ必要な場合（エラーを無視）
result, _ := divide(10, 3)
```

---

## 名前付き戻り値

戻り値に名前を付けることができます。

```go
func split(sum int) (x, y int) {
    x = sum * 4 / 9
    y = sum - x
    return  // naked return: x, y が自動で返される
}

func main() {
    x, y := split(17)
    fmt.Println(x, y)  // 7 10
}
```

名前付き戻り値は関数のドキュメントとして役立ちますが、`return` で何を返しているか分かりにくくなることもあります。**短い関数でのみ使う**のが一般的です。

---

## 可変長引数

```go
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

func main() {
    fmt.Println(sum(1, 2, 3))       // 6
    fmt.Println(sum(1, 2, 3, 4, 5)) // 15

    // スライスを展開して渡す
    numbers := []int{10, 20, 30}
    fmt.Println(sum(numbers...))     // 60
}
```

可変長引数は **`...型`** で宣言します。関数内ではスライスとして扱えます。`fmt.Println` も可変長引数の関数です。

---

## 関数は第一級オブジェクト

Goでは関数を変数に代入したり、引数として渡したりできます。

### 関数を変数に代入

```go
func main() {
    // 関数を変数に代入
    double := func(n int) int {
        return n * 2
    }

    fmt.Println(double(5))  // 10
}
```

### 関数を引数に渡す

```go
func apply(nums []int, fn func(int) int) []int {
    result := make([]int, len(nums))
    for i, n := range nums {
        result[i] = fn(n)
    }
    return result
}

func main() {
    numbers := []int{1, 2, 3, 4, 5}

    doubled := apply(numbers, func(n int) int {
        return n * 2
    })
    fmt.Println(doubled)  // [2 4 6 8 10]

    squared := apply(numbers, func(n int) int {
        return n * n
    })
    fmt.Println(squared)  // [1 4 9 16 25]
}
```

---

## クロージャ

クロージャは外側の変数を「覚えている」関数です。

```go
func counter() func() int {
    count := 0
    return func() int {
        count++  // 外側の count を参照・更新できる
        return count
    }
}

func main() {
    c := counter()
    fmt.Println(c())  // 1
    fmt.Println(c())  // 2
    fmt.Println(c())  // 3

    // 別のカウンターは独立
    c2 := counter()
    fmt.Println(c2())  // 1
}
```

### 実用的なクロージャの例

```go
// ミドルウェアパターン
func withLogging(fn func(string) string) func(string) string {
    return func(input string) string {
        fmt.Printf("入力: %s\n", input)
        result := fn(input)
        fmt.Printf("出力: %s\n", result)
        return result
    }
}

func main() {
    upper := withLogging(strings.ToUpper)
    upper("hello")
    // 入力: hello
    // 出力: HELLO
}
```

---

## init 関数

`init` 関数はパッケージが読み込まれたときに自動実行される特殊な関数です。

```go
package main

import "fmt"

var startTime time.Time

func init() {
    // main() より先に実行される
    startTime = time.Now()
    fmt.Println("初期化完了")
}

func main() {
    fmt.Println("メイン処理開始")
    fmt.Println("開始時刻:", startTime)
}
// 出力:
// 初期化完了
// メイン処理開始
// 開始時刻: 2026-03-11 ...
```

`init` は引数も戻り値もなく、明示的に呼び出すことはできません。主にパッケージの初期化処理に使います。ただし多用するとプログラムの流れが追いにくくなるため、必要最小限にとどめましょう。

---

## やってみよう

1. 2つの整数を受け取り、大きい方を返す `max(a, b int) int` 関数を書いてみましょう
2. 可変長引数を受け取り、平均値を返す `average(nums ...float64) float64` 関数を書いてみましょう
3. クロージャを使って、呼ぶたびにフィボナッチ数列の次の値を返す関数を作ってみましょう

---

次の章では、配列・スライス・マップというGoのデータ構造を学びます。
