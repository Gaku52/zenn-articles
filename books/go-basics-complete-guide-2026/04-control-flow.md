---
title: "制御フロー"
---

# Chapter 04: 制御フロー

> if, for, switch, defer — Goの制御構文はシンプルで統一的

## この章で学べること

- ✅ if / else による条件分岐
- ✅ for文（Goに while はない）
- ✅ switch文（break不要の賢い分岐）
- ✅ defer（関数終了時の後処理）

---

## なぜGoの制御構文はシンプルなのか

多くの言語には `for`、`while`、`do-while`、`foreach` など複数のループ構文があります。Goには **`for` だけ**です。条件分岐も `if` と `switch` だけ。三項演算子もありません。

「1つの正しい書き方」を提供することで、チーム開発での認知負荷を下げる — これがGoの設計思想です。

---

## 条件分岐: if / else

### 基本構文

```go
score := 85

if score >= 90 {
    fmt.Println("優秀")
} else if score >= 70 {
    fmt.Println("良好")
} else if score >= 50 {
    fmt.Println("合格")
} else {
    fmt.Println("不合格")
}
// 出力: 良好
```

> **注意**: Goでは条件式に**括弧 `()` は不要**です。波括弧 `{}` は**必須**です。

### if の初期化文

Goの `if` 文は、条件の前に**短い文**を1つ置けます。変数のスコープを絞れるのが利点です。

```go
// 他の言語での書き方
err := doSomething()
if err != nil {
    // err を処理
}
// err がこの後もスコープに残ってしまう

// Goらしい書き方
if err := doSomething(); err != nil {
    fmt.Println("エラー:", err)
    return
}
// err はここではアクセスできない（スコープ外）
```

このパターンはGoのコードで非常に頻繁に使われます。特にエラー処理で活躍します。

---

## ループ: for

### 基本の for ループ（C言語スタイル）

```go
for i := 0; i < 5; i++ {
    fmt.Println(i)
}
// 出力: 0, 1, 2, 3, 4
```

### while スタイル（条件だけ）

```go
count := 0
for count < 5 {
    fmt.Println(count)
    count++
}
```

Goに `while` キーワードはありません。`for` に条件だけ書けば同じことができます。

### 無限ループ

```go
for {
    fmt.Println("永遠に実行")
    // break で抜ける
    break
}
```

### range によるイテレーション

コレクション（スライス、マップ、文字列、チャネル）のループには `range` を使います。

```go
// スライスのイテレーション
fruits := []string{"Apple", "Banana", "Cherry"}

for i, fruit := range fruits {
    fmt.Printf("%d: %s\n", i, fruit)
}
// 0: Apple
// 1: Banana
// 2: Cherry

// インデックスが不要な場合
for _, fruit := range fruits {
    fmt.Println(fruit)
}

// インデックスだけ必要な場合
for i := range fruits {
    fmt.Println(i)
}
```

### 文字列の range

```go
s := "Go言語"
for i, r := range s {
    fmt.Printf("byte=%d, char=%c\n", i, r)
}
// byte=0, char=G
// byte=1, char=o
// byte=2, char=言
// byte=5, char=語
```

`range` は文字列をバイト単位ではなく**rune（文字）単位**で処理します。インデックスはバイト位置です。

### break と continue

```go
// break: ループを抜ける
for i := 0; i < 10; i++ {
    if i == 5 {
        break
    }
    fmt.Println(i)
}
// 0, 1, 2, 3, 4

// continue: 次のイテレーションへ
for i := 0; i < 10; i++ {
    if i%2 == 0 {
        continue  // 偶数をスキップ
    }
    fmt.Println(i)
}
// 1, 3, 5, 7, 9
```

### ラベル付きbreak

ネストしたループから一気に抜けたい場合：

```go
outer:
    for i := 0; i < 3; i++ {
        for j := 0; j < 3; j++ {
            if i == 1 && j == 1 {
                break outer  // 外側のループごと抜ける
            }
            fmt.Printf("i=%d, j=%d\n", i, j)
        }
    }
```

---

## 分岐: switch

### 基本構文

```go
day := "Monday"

switch day {
case "Monday":
    fmt.Println("週の始まり")
case "Friday":
    fmt.Println("もうすぐ週末")
case "Saturday", "Sunday":  // 複数の値をまとめられる
    fmt.Println("休日")
default:
    fmt.Println("平日")
}
```

> **重要**: Goの switch は**自動で break**します。C言語のようにフォールスルーしません。

### 式なしの switch（if-else チェーンの代替）

```go
score := 85

switch {
case score >= 90:
    fmt.Println("A")
case score >= 80:
    fmt.Println("B")
case score >= 70:
    fmt.Println("C")
default:
    fmt.Println("F")
}
// 出力: B
```

この形式は複数条件の分岐を見やすく書けるため、長い if-else チェーンの代わりによく使われます。

### 初期化文付き switch

```go
switch os := runtime.GOOS; os {
case "darwin":
    fmt.Println("macOS")
case "linux":
    fmt.Println("Linux")
default:
    fmt.Printf("その他: %s\n", os)
}
```

### fallthrough（明示的なフォールスルー）

```go
switch 3 {
case 3:
    fmt.Println("3です")
    fallthrough  // 次の case も実行する
case 4:
    fmt.Println("3 または 4 です")
}
// 出力:
// 3です
// 3 または 4 です
```

`fallthrough` は実務ではほとんど使いません。存在を知っていれば十分です。

---

## defer — 後処理の予約

`defer` は「この関数が終わるときに実行してね」という予約です。

### 基本的な使い方

```go
func main() {
    fmt.Println("開始")
    defer fmt.Println("終了")  // 関数の最後に実行される
    fmt.Println("処理中")
}
// 出力:
// 開始
// 処理中
// 終了
```

### 典型的な用途: リソースのクリーンアップ

```go
func readFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close()  // この関数を抜けるとき必ず閉じる

    // ファイルの読み取り処理...
    // どこで return しても f.Close() が呼ばれる
    return nil
}
```

`defer` の利点は「リソースを開いたすぐ後にクリーンアップを書ける」ことです。後の方で `Close()` を書き忘れる心配がありません。

### defer の実行順序（LIFO）

複数の `defer` は**後入れ先出し（LIFO）**で実行されます。

```go
func main() {
    defer fmt.Println("1st")
    defer fmt.Println("2nd")
    defer fmt.Println("3rd")
}
// 出力:
// 3rd
// 2nd
// 1st
```

---

## やってみよう

1. FizzBuzz を書いてみましょう（1-30で、3の倍数は"Fizz"、5の倍数は"Buzz"、両方の倍数は"FizzBuzz"）
2. 式なしの switch を使って BMI の判定を書いてみましょう
3. `range` を使って文字列 `"Hello, 世界"` を1文字ずつ表示してみましょう
4. `defer` を使って「開始→処理中→終了」の順で表示するプログラムを書いてみましょう

---

次の章では、関数の定義と活用を学びます。
