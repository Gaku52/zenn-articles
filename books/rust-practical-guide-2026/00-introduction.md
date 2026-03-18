---
title: "Rustとは -- なぜ今Rustなのか"
---

## なぜ今Rustなのか

2022年にLinuxカーネルがRustを公式サポートし、2024年にはホワイトハウスがメモリ安全な言語の採用を推奨しました。AWS、Google、Microsoft、Cloudflareがインフラの中核にRustを採用しています。

これは一時的なトレンドではありません。ソフトウェアの信頼性に対する要求が高まる中、**コンパイル時にバグを潰す**というRustのアプローチが、産業界で本格的に評価されているのです。

---

## 1. Rustの3つの柱

Rustは「安全性」「速度」「並行性」の三拍子を同時に達成します。従来これらはトレードオフでしたが、所有権システムという独自のアプローチで解決しています。

### 安全性 -- コンパイラが守る

nullポインタ参照、ダングリングポインタ、バッファオーバーフローなど、C/C++で長年問題だったバグの約70%（Microsoft Research調査）がコンパイル時に防止されます。

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1; // s1の所有権がs2にムーブ
    // println!("{}", s1); // コンパイルエラー！s1はもう使えない
    println!("{}", s2);    // OK
}
```

### 速度 -- ゼロコスト抽象化

GCがないため一時停止が発生しません。高レベルな書き方でもC/C++同等の性能を発揮します。

```rust
let sum: i32 = (0..1000)
    .filter(|x| x % 2 == 0)
    .map(|x| x * x)
    .sum();
// コンパイラが手書きのforループと同等の機械語に最適化
```

### 並行性 -- 恐れなき並行処理

データ競合をコンパイル時に検出します。Goの`-race`フラグは実行時検出ですが、Rustでは**そもそもコンパイルが通りません**。

---

## 2. 他言語との比較

| 観点 | JS / Python | Go | Rust |
|------|------------|----|------|
| メモリ管理 | GCが自動回収 | GCが自動回収 | 所有権（GCなし） |
| null | `null`/`None` | `nil` | `Option<T>` |
| エラー処理 | 例外 | `error`戻り値 | `Result<T, E>` |
| 実行速度 | 遅い | 速い | 非常に速い |
| バイナリ配布 | ランタイム必要 | シングルバイナリ | シングルバイナリ |
| 学習曲線 | 緩やか | 緩やか | 急峻（習得後は高生産性） |

### GCがないのになぜ安全？

各値に「所有者」が1つだけ存在し、所有者がスコープを抜けると自動でメモリ解放されます。コンパイラが借用規則を静的に検証するため、GCなしでメモリ安全を実現しています。

```rust
fn main() {
    {
        let s = String::from("hello"); // sがメモリを所有
        println!("{}", s);
    } // sがスコープを抜ける → 自動解放（Drop）
}
```

- **Python経験者向け**: `with`文のリソース管理を全ての値に自動適用するイメージです
- **Go経験者向け**: `defer`で閉じ忘れを防ぐ必要がありません

---

## 3. 開発環境セットアップ

### rustupのインストール

```bash
# macOS / Linux
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env

# バージョン確認
rustc --version
cargo --version
```

Windowsの場合は [rustup.rs](https://rustup.rs/) からインストーラをダウンロードしてください。

### 推奨ツールの追加

```bash
rustup component add clippy    # 静的解析リンター
rustup component add rustfmt   # コードフォーマッタ
```

### エディタ

**VS Code** + `rust-analyzer` 拡張が最も推奨です。リアルタイム型推論、インラインエラー表示、コード補完が使えます。

---

## 4. Cargoの基本

CargoはRustのビルドシステム兼パッケージマネージャです。npm（Node.js）やpip+poetry（Python）に相当しますが、ビルド・テスト・ドキュメント生成まで統一的に扱えます。

### 主要コマンド

```bash
cargo new my_project         # プロジェクト作成
cargo new my_lib --lib       # ライブラリ作成
cargo build                  # デバッグビルド
cargo build --release        # リリースビルド（最適化あり）
cargo run                    # ビルドして実行
cargo check                  # コンパイルチェックのみ（高速）
cargo test                   # テスト実行
cargo clippy                 # 静的解析
cargo fmt                    # フォーマット
```

### 使い分けのポイント

| コマンド | 用途 | 使用頻度 |
|---------|------|---------|
| `cargo check` | 型エラー確認。バイナリを作らないので高速 | 非常に高い |
| `cargo clippy` | ベストプラクティスの提案 | PR前に必須 |
| `cargo test` | テスト実行 | CI必須 |
| `cargo fmt` | フォーマット統一 | コミット前 |

### Cargo.tomlの構成

```toml
[package]
name = "my_project"
version = "0.1.0"
edition = "2024"

[dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1", features = ["full"] }

[dev-dependencies]
assert_cmd = "2.0"

[profile.release]
opt-level = 3
lto = true
strip = true
```

---

## 5. Hello Worldと基本構文

```bash
cargo new hello_rust && cd hello_rust
```

### Hello World

```rust
fn main() {
    println!("Hello, Rust!");
}
```

`println!`末尾の`!`はマクロを示します。フォーマット文字列がコンパイル時に検証されます。

### 変数とイミュータビリティ

```rust
fn main() {
    let x = 5;           // 不変（デフォルト）
    // x = 6;            // コンパイルエラー！
    let mut y = 10;      // mutで可変宣言
    y += 1;
    println!("x={}, y={}", x, y);

    // シャドウイング：型も変更可能
    let x = x + 1;
    let x = x.to_string();  // i32 → String
    println!("x = {}", x);
}
```

### 関数と戻り値

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b  // セミコロンなし = 式として返却
}

fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err("ゼロ除算".to_string())
    } else {
        Ok(a / b)
    }
}

fn main() {
    println!("3 + 4 = {}", add(3, 4));
    match divide(10.0, 3.0) {
        Ok(v) => println!("10 / 3 = {:.2}", v),
        Err(e) => eprintln!("エラー: {}", e),
    }
}
```

### Option / Result / パターンマッチング

```rust
fn main() {
    // Option型：nullの代わり
    let name: Option<&str> = Some("Rust");
    if let Some(n) = name {
        println!("名前: {}", n);
    }

    // Result型：例外の代わり
    let number: Result<i32, _> = "42".parse();
    match number {
        Ok(n) => println!("パース成功: {}", n),
        Err(e) => println!("パース失敗: {}", e),
    }
}
```

---

## 6. Rust 2024 editionの位置づけ

Rustは「edition」で後方互換性を保ちつつ進化します。異なるeditionのクレート同士も相互依存可能です。

| Edition | 主な変更点 |
|---------|----------|
| 2015 | 初期安定版 |
| 2018 | async/await準備、モジュールパス簡略化 |
| 2021 | クロージャ部分キャプチャ、`IntoIterator` for arrays |
| **2024** | **`unsafe_op_in_unsafe_fn`デフォルト化、RPIT改善** |

本書は2024 editionを前提とします。`Cargo.toml`で`edition = "2024"`を指定してください。

---

## 7. 対象読者と学習ロードマップ

本書はJS/TS、Python、Go、Java等の経験者がRustを実践で使えるようになることを目的としています。システムプログラミングの知識は不要です。所有権やライフタイムに挫折した方も歓迎します。

### 学習ロードマップ

```
第1部：基礎（Ch.1-4）  所有権・型・エラー・コレクション
第2部：実践（Ch.5-8）  設計・モジュール・テスト・async
第3部：応用（Ch.9-12） CLI開発・Web API・DB連携・CI/CD
```

第1部の所有権と借用（Chapter 1）は全ての基盤です。必ず最初に読んでください。

---

## まとめ

| 項目 | 要点 |
|------|------|
| 3つの柱 | 安全性・速度・並行性を同時達成 |
| メモリ管理 | 所有権システム（GCなし、実行時オーバーヘッドなし） |
| GC言語との違い | コンパイル時にメモリ安全性を保証 |
| ツールチェーン | `rustup` + `cargo` + `rust-analyzer` |
| Cargoの基本 | `new`/`build`/`run`/`test`/`clippy`/`fmt` |
| 2024 edition | unsafe境界の厳密化、後方互換性は維持 |
| 学習のコツ | エラーメッセージを丁寧に読む。最初はcloneを恐れない |

---

## やってみよう！

### 演習1：環境構築の確認

```bash
rustc --version
cargo --version
cargo clippy --version
cargo fmt --version
```

全てのコマンドが動作すればセットアップ完了です。

### 演習2：最初のプロジェクト

以下のプログラムを作成し、`cargo run`で実行してください。

```rust
fn greet(name: &str, language: &str) -> String {
    match language {
        "ja" => format!("こんにちは、{}さん！", name),
        "en" => format!("Hello, {}!", name),
        "es" => format!("Hola, {}!", name),
        _    => format!("Hi, {}!", name),
    }
}

fn main() {
    let languages = vec!["ja", "en", "es", "fr"];
    for lang in &languages {
        println!("{}", greet("Rust", lang));
    }
}
```

`cargo clippy`で警告が出ないこと、`cargo fmt`で整形されることも確認しましょう。

### 演習3：テストを書いてみる

演習2の`greet`関数にテストを追加してください。

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_greet_japanese() {
        assert_eq!(greet("太郎", "ja"), "こんにちは、太郎さん！");
    }

    #[test]
    fn test_greet_english() {
        assert_eq!(greet("Alice", "en"), "Hello, Alice!");
    }

    #[test]
    fn test_greet_unknown() {
        assert_eq!(greet("Bob", "xx"), "Hi, Bob!");
    }
}
```

`cargo test`で全テストが通ることを確認しましょう。テストが標準装備なのはRustの大きな魅力です。

---

次章では、Rustの最も重要な概念である**所有権と借用**を詳しく学びます。
