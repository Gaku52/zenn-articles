---
title: "データ構造"
---

# Chapter 06: データ構造

> 配列、スライス、マップ — Goのデータ管理の基本

## この章で学べること

- ✅ 配列（固定長）
- ✅ スライス（可変長 — Goで最も使うデータ構造）
- ✅ スライスの内部構造と注意点
- ✅ マップ（キーと値のペア）
- ✅ make と new の違い

---

## 配列

配列は**固定長**のデータ構造です。宣言時にサイズが決まり、後から変更できません。

```go
// 配列の宣言
var numbers [5]int                       // [0 0 0 0 0]
colors := [3]string{"Red", "Green", "Blue"}
auto := [...]int{1, 2, 3, 4, 5}         // [...] でサイズを自動推論

fmt.Println(len(colors))  // 3
fmt.Println(colors[0])    // Red
```

**実務では配列をほとんど使いません。** 代わりにスライスを使います。配列はサイズが型の一部であり（`[3]int` と `[5]int` は別の型）、関数に渡すときにコピーが発生するためです。

---

## スライス

スライスは**可変長**の配列です。Goで最も頻繁に使うデータ構造です。

### 基本操作

```go
// スライスの作成
fruits := []string{"Apple", "Banana", "Cherry"}  // 配列と違い [...] や数値なし
numbers := []int{1, 2, 3, 4, 5}

// 空のスライス
var empty []int          // nil スライス（長さ0、容量0）
also := []int{}          // 空スライス（長さ0、容量0、非nil）
made := make([]int, 5)   // [0 0 0 0 0]（長さ5、容量5）

// アクセス
fmt.Println(fruits[0])    // Apple
fmt.Println(fruits[1:3])  // [Banana Cherry]（スライシング）

// 長さと容量
fmt.Println(len(fruits))  // 3（現在の要素数）
fmt.Println(cap(fruits))  // 3（内部配列のサイズ）
```

### append — 要素の追加

```go
fruits := []string{"Apple", "Banana"}

// 要素を追加（元のスライスは変更されない）
fruits = append(fruits, "Cherry")
fmt.Println(fruits)  // [Apple Banana Cherry]

// 複数の要素を一度に追加
fruits = append(fruits, "Date", "Elderberry")

// スライス同士の結合
more := []string{"Fig", "Grape"}
fruits = append(fruits, more...)  // ... で展開
```

> **重要**: `append` は**新しいスライスを返す**ため、必ず `fruits = append(fruits, ...)` のように再代入してください。

### スライシング

```go
nums := []int{0, 1, 2, 3, 4, 5}

fmt.Println(nums[1:4])  // [1 2 3]  （インデックス1から3まで）
fmt.Println(nums[:3])   // [0 1 2]  （先頭から2まで）
fmt.Println(nums[3:])   // [3 4 5]  （3から末尾まで）
fmt.Println(nums[:])    // [0 1 2 3 4 5]  （全範囲、ただし内部配列は共有）
```

### スライスの内部構造

スライスは3つの要素を持つ**ヘッダ構造体**です：

```
スライスヘッダ（変数代入でコピーされる）
┌──────────────────┐
│ ptr ─────────────┼──→ [ A ][ B ][ C ][ _ ][ _ ]  ← 内部配列（ヒープ上）
│ len = 3          │       ↑
│ cap = 5          │       この配列の場所を指している
└──────────────────┘
```

「スライスを別の変数に代入する」とは、**このヘッダをコピーする**ことです。
内部配列のポインタも一緒にコピーされるため、**同じ配列を2つのスライスが指す**状態になります。

### 注意: スライスのコピーは内部配列を共有する

> スライスは「参照型」ではありません。Goには参照渡しは存在せず、スライスも例外ではありません。ただし、ヘッダの中に配列へのポインタが入っているため、コピーしても同じ配列を指し続けます。

```go
original := []int{1, 2, 3}
copied := original  // ヘッダのコピー（内部配列は共有）

copied[0] = 999
fmt.Println(original)  // [999 2 3]  ← 元のスライスも変わる！
fmt.Println(copied)    // [999 2 3]
```

**図解:**
```
代入前:
original: ptr─→[1][2][3]
                ↑
代入後:
original: ptr─→[1][2][3]
                ↑
copied:   ptr──┘           ← 同じ配列を指している

copied[0] = 999 の後:
original: ptr─→[999][2][3]
                ↑
copied:   ptr──┘
```

### 独立したコピーを作る

```go
// copy 関数を使う
independent := make([]int, len(original))
copy(independent, original)

// または Go 1.21+
independent := slices.Clone(original)

independent[0] = 0
fmt.Println(original)    // [999 2 3]  ← 変わらない
fmt.Println(independent) // [0 2 3]
```

### append はなぜ呼び出し元に反映されないのか

```go
func addElement(s []int) {
    s = append(s, 100)  // ヘッダのコピーを変更しているだけ
    fmt.Println(s)      // [1 2 3 100]
}

func main() {
    nums := []int{1, 2, 3}
    addElement(nums)
    fmt.Println(nums)   // [1 2 3] ← 変わっていない！
}
```

`append` はスライスヘッダ（特に `len`）を更新しますが、関数に渡されるのはヘッダのコピーです。コピーを更新しても元のヘッダには影響しません。

スライス自体を関数内で変更したい場合はポインタを渡します（詳しくは次章）。

### make でスライスを作る

```go
// make([]型, 長さ, 容量)
s := make([]int, 3, 10)
fmt.Println(len(s))  // 3
fmt.Println(cap(s))  // 10

// 容量を指定すると append 時のメモリ再確保を減らせる
// 要素数が事前に分かっている場合に有効
users := make([]User, 0, 100)  // 長さ0、容量100
```

### スライスの削除パターン

```go
s := []int{1, 2, 3, 4, 5}

// i 番目の要素を削除（順序を維持）
i := 2
s = append(s[:i], s[i+1:]...)
fmt.Println(s)  // [1 2 4 5]

// Go 1.21+ なら slices パッケージ
s = slices.Delete(s, i, i+1)
```

---

## マップ

マップは**キーと値のペア**を格納するデータ構造です。他の言語の辞書（dict）やハッシュマップに相当します。

### 基本操作

```go
// マップの作成
ages := map[string]int{
    "Alice": 30,
    "Bob":   25,
    "Carol": 28,
}

// make で作成
scores := make(map[string]int)

// 値の設定
scores["Math"] = 95
scores["English"] = 88

// 値の取得
age := ages["Alice"]
fmt.Println(age)  // 30

// 存在しないキーはゼロ値を返す
unknown := ages["Nobody"]
fmt.Println(unknown)  // 0（int のゼロ値）
```

### 存在チェック（comma ok パターン）

```go
age, ok := ages["Alice"]
if ok {
    fmt.Printf("Alice は %d 歳\n", age)
} else {
    fmt.Println("Alice は見つかりません")
}

// よく使う短縮形
if age, ok := ages["Bob"]; ok {
    fmt.Printf("Bob は %d 歳\n", age)
}
```

この `value, ok` パターンはGoの至る所で使われる重要なイディオムです。

### 削除

```go
delete(ages, "Bob")
```

### イテレーション

```go
for name, age := range ages {
    fmt.Printf("%s: %d歳\n", name, age)
}

// キーだけ
for name := range ages {
    fmt.Println(name)
}
```

> **注意**: マップのイテレーション順序は**毎回ランダム**です。順序が必要な場合はキーをソートしてから使います。

### マップも内部でポインタを持つ

スライスと同様に、マップも内部でデータへのポインタを持ちます。
マップを別の変数に代入したり関数に渡したりしても、内部データは共有されます。

```go
original := map[string]int{"Alice": 30}
copied := original  // 内部データを共有

copied["Alice"] = 99
fmt.Println(original["Alice"])  // 99 ← 元も変わる

func addEntry(m map[string]int) {
    m["new"] = 42  // 呼び出し元にも反映される
}
```

> スライスもマップも「参照型」ではなく、「内部にポインタを持つ値」です。どちらも値渡しですが、そのポインタを通して同じデータにアクセスできます。

### マップのゼロ値は nil

```go
var m map[string]int  // nil マップ
// m["key"] = 1       // パニック！ nil マップへの書き込みはランタイムエラー

m = make(map[string]int)  // 初期化すればOK
m["key"] = 1               // OK
```

---

## make と new

| 関数 | 用途 | 返す型 | 使用頻度 |
|------|------|--------|---------|
| `make` | スライス、マップ、チャネルの初期化 | 値そのもの | 高い |
| `new` | 任意の型のゼロ値へのポインタ | ポインタ | 低い |

```go
// make: スライス、マップ、チャネル専用
s := make([]int, 5)        // []int
m := make(map[string]int)  // map[string]int

// new: ゼロ値へのポインタを返す
p := new(int)    // *int（値は0）
fmt.Println(*p)  // 0
```

実務では `make` を頻繁に使い、`new` はほとんど使いません。ポインタが必要な場合は `&` 演算子（次章で学習）を使うのが一般的です。

---

## やってみよう

1. 文字列のスライスに5つの果物を入れ、`range` で全て表示してみましょう
2. `map[string]int` で5教科の点数を管理し、合計点と平均点を計算してみましょう
3. スライスのコピーの挙動を確認してみましょう（コピーを変更して元が変わるか）
4. `comma ok` パターンを使って、マップに存在しないキーを安全にチェックしてみましょう

---

次の章では、ポインタとメモリモデルを学びます。スライスとマップの内部構造の理解がさらに深まります。
