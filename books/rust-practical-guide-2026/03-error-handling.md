---
title: "エラーハンドリング"
---

## Rustにtry-catchがない理由

多くの言語では `try-catch` による例外処理が一般的ですが、Rustは意図的に例外機構を排除しています。代わりに `Result<T, E>` と `Option<T>` という2つの列挙型で、エラーや値の不在を**型として**表現します。

この設計には明確な理由があります。

- **明示性**: 関数のシグネチャを見るだけで、エラーが発生しうるかが分かります
- **網羅性**: コンパイラがエラー処理の漏れを検出してくれます
- **パフォーマンス**: 例外のスタック巻き戻しコストがありません
- **合成可能性**: `?` 演算子やコンビネータでエラー処理を簡潔に連鎖できます

Rustの3つのエラー分類を整理すると、次のようになります。

| 分類 | 型/機構 | 例 |
|:-----|:--------|:---|
| 回復不能エラー | `panic!()` | 配列の範囲外アクセス、バグ |
| 回復可能エラー | `Result<T, E>` | ファイル未発見、パースエラー |
| 値の不在 | `Option<T>` | 検索結果なし、設定項目なし |

「エラーが起きるかもしれない」という情報が型に埋め込まれているため、処理を忘れるとコンパイルが通りません。これが、実行時に突然 `NullPointerException` が飛んでくる言語との決定的な違いです。

## Option\<T\> の使い方

`Option<T>` は「値があるかもしれないし、ないかもしれない」を表す型です。他の言語でいう `null` や `nil` の代わりに使います。

```rust
fn find_user(id: u64) -> Option<String> {
    match id {
        1 => Some(String::from("田中")),
        2 => Some(String::from("鈴木")),
        _ => None,
    }
}

fn main() {
    // パターンマッチで安全に取り出す
    match find_user(1) {
        Some(name) => println!("ユーザー: {}", name),
        None => println!("見つかりません"),
    }
    // if let で Some の場合だけ処理
    if let Some(name) = find_user(2) {
        println!("ユーザー: {}", name);
    }
    // unwrap_or でデフォルト値を指定
    let name = find_user(99).unwrap_or(String::from("不明"));
    println!("ユーザー: {}", name);
}
```

`Option` にはコンビネータと呼ばれる便利なメソッドが多数あります。

```rust
// map: Some の中身を変換します
let doubled = Some(21).map(|x| x * 2); // Some(42)

// and_then: Option を返す処理を連鎖します（flatMap 相当）
let parsed = Some("42").and_then(|s| s.parse::<i32>().ok()).map(|n| n * 2); // Some(84)

// filter: 条件を満たさなければ None にします
let even = Some(4).filter(|x| x % 2 == 0); // Some(4)
let odd  = Some(3).filter(|x| x % 2 == 0); // None
```

ネストした `Option` を `?` 演算子で簡潔に扱うパターンも実務で頻出します。

```rust
fn get_db_url(config: &Config) -> Option<String> {
    let db = config.database.as_ref()?;  // None なら即 None を返す
    let host = db.host.as_ref()?;
    let port = db.port?;
    Some(format!("postgres://{}:{}/mydb", host, port))
}
```

## Result\<T, E\> の使い方

`Result<T, E>` は「成功(`Ok`)か失敗(`Err`)か」を表す型です。ファイル操作やネットワーク通信など、失敗しうる処理の戻り値に使います。

```rust
use std::fs;
use std::io;

fn read_username() -> Result<String, io::Error> {
    let content = fs::read_to_string("username.txt")?;
    Ok(content.trim().to_string())
}

fn main() {
    match read_username() {
        Ok(name) => println!("ユーザー名: {}", name),
        Err(e) => println!("エラー: {}", e),
    }
}
```

`Result` にもコンビネータがあります。`map` は `Ok` の中身を変換し、`map_err` は `Err` の中身を変換します。複数の `Result` を `collect` で一括集約することもできます。

```rust
let result = "21".parse::<i32>().map(|n| n * 2); // Ok(42)

let strings = vec!["1", "2", "3"];
let numbers: Result<Vec<i32>, _> = strings.iter().map(|s| s.parse::<i32>()).collect();
// Ok([1, 2, 3])  ※1つでもエラーがあれば全体が Err になります
```

## ? 演算子による伝播

`?` 演算子はRustのエラーハンドリングの要です。`Result` が `Err` のとき、現在の関数から即座にそのエラーを返します。

```rust
use std::fs::File;
use std::io::{self, Read};

// ? 演算子なしだと match の連続で冗長になります
fn read_file_verbose(path: &str) -> Result<String, io::Error> {
    let file = match File::open(path) { Ok(f) => f, Err(e) => return Err(e) };
    let mut buf = String::new();
    match file.read_to_string(&mut buf) { Ok(_) => Ok(buf), Err(e) => Err(e) }
}

// ? 演算子ありなら同じ処理がこれだけで済みます
fn read_file_concise(path: &str) -> Result<String, io::Error> {
    let mut file = File::open(path)?;
    let mut buf = String::new();
    file.read_to_string(&mut buf)?;
    Ok(buf)
}
```

`?` は内部で `From::from(e)` を呼ぶため、`From` トレイトが実装されていれば異なるエラー型間の自動変換も行います。

## カスタムエラー型の定義

複数のエラー型を扱う場合、カスタムエラー型を `enum` で定義します。`Display`、`Error`、`From` の3つのトレイトを手動で実装する必要があります。

```rust
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
enum AppError {
    Io(io::Error),
    Parse(ParseIntError),
    Validation(String),
}

impl std::fmt::Display for AppError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            AppError::Io(e) => write!(f, "IOエラー: {}", e),
            AppError::Parse(e) => write!(f, "パースエラー: {}", e),
            AppError::Validation(msg) => write!(f, "検証エラー: {}", msg),
        }
    }
}

impl std::error::Error for AppError {}

impl From<io::Error> for AppError {
    fn from(e: io::Error) -> Self { AppError::Io(e) }
}
impl From<ParseIntError> for AppError {
    fn from(e: ParseIntError) -> Self { AppError::Parse(e) }
}

fn load_port(path: &str) -> Result<u32, AppError> {
    let content = std::fs::read_to_string(path)?; // io::Error → AppError
    let port: u32 = content.trim().parse()?;       // ParseIntError → AppError
    if port < 1024 {
        return Err(AppError::Validation(format!("ポート {} は予約済み", port)));
    }
    Ok(port)
}
```

この手動実装はかなり面倒です。この問題を解決するのが `thiserror` クレートです。

## thiserror クレートによるエラー定義

`thiserror` は derive マクロで `Display`、`Error`、`From` を自動生成します。主にライブラリのエラー型定義に使います。

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum DatabaseError {
    #[error("接続エラー: {0}")]
    ConnectionFailed(String),

    #[error("クエリエラー: {query}")]
    QueryFailed { query: String, #[source] source: std::io::Error },

    #[error("レコードが見つかりません: ID={id}")]
    NotFound { id: u64 },

    #[error(transparent)]
    Io(#[from] std::io::Error),
}
```

各アトリビュートの役割は次の通りです。

- `#[error("...")]` -- `Display` を自動実装します
- `#[from]` -- `From` トレイトを自動実装し、`?` での変換を可能にします
- `#[source]` -- `Error::source()` を自動実装し、エラーチェーンを構成します
- `#[error(transparent)]` -- 内部エラーのメッセージをそのまま表示します

## anyhow クレートによるアプリケーションエラー

`anyhow` はアプリケーション層でのエラー処理を簡潔にするクレートです。具体的なエラー型を定義せず、すべてのエラーを `anyhow::Error` に集約します。

```rust
use anyhow::{Context, Result, bail, ensure};

fn load_config(path: &str) -> Result<Config> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("設定ファイル '{}' を読み込めません", path))?;

    let lines: Vec<&str> = content.lines().collect();
    ensure!(lines.len() >= 2, "設定ファイルには少なくとも2行必要です");

    let host = lines[0].trim().to_string();
    if host.is_empty() { bail!("ホストが指定されていません"); }

    let port: u16 = lines[1].trim().parse().context("ポート番号のパースに失敗")?;
    Ok(Config { host, port })
}
```

- **`context()` / `with_context()`** -- エラーに文脈情報を付加します
- **`bail!`** -- 即座に `Err` を返すマクロです
- **`ensure!`** -- 条件を満たさなければ `Err` を返すマクロです

## thiserror vs anyhow の使い分け

この2つは競合するものではなく、使う場面が異なります。

| 特性 | thiserror | anyhow |
|:-----|:----------|:-------|
| 目的 | ライブラリのエラー型定義 | アプリのエラー処理 |
| エラー型 | 具体的な enum | 型消去された `anyhow::Error` |
| パターンマッチ | そのまま可能 | `downcast` が必要 |
| 推奨場面 | 公開API、ライブラリ | CLI、Webサーバー、バイナリ |

実務で最も多いのは**ハイブリッドアプローチ**です。ライブラリ層は `thiserror` で具体的なエラー型を定義し、アプリケーション層は `anyhow` でそれらを集約します。

```rust
// ライブラリ層: thiserror で具体的なエラー型
mod db {
    use thiserror::Error;
    #[derive(Debug, Error)]
    pub enum DbError {
        #[error("接続エラー: {0}")] Connection(String),
        #[error("レコード未発見: {0}")] NotFound(u64),
    }
    pub fn find_user(id: u64) -> Result<String, DbError> {
        match id { 0 => Err(DbError::NotFound(id)), _ => Ok(format!("user_{}", id)) }
    }
}

// アプリケーション層: anyhow でエラーを集約
fn get_user_name(id: u64) -> anyhow::Result<String> {
    let user = db::find_user(id)
        .with_context(|| format!("ユーザーID {} の取得に失敗", id))?;
    Ok(user)
}
```

## panic! の適切な使いどころ

`panic!` はプログラムを即座に中断します。回復可能なエラーには使わず、以下の場面に限定してください。

1. **プログラムの不変条件が破れた場合**（バグの検出）
2. **テストコード**（`assert!`、`unwrap`）
3. **プロトタイプ段階**（後で `Result` に置き換える前提）
4. **回復不可能な初期化エラー**（`main` の冒頭のみ）

```rust
// panic! が適切な例: バグの検出
fn divide(a: f64, b: f64) -> f64 {
    assert!(b != 0.0, "ゼロ除算はバグです");
    a / b
}

// panic! が不適切な例: ユーザー入力のバリデーション
fn parse_port_bad(s: &str) -> u16 { s.parse().unwrap() } // BAD
fn parse_port_good(s: &str) -> Result<u16, std::num::ParseIntError> { s.parse() } // GOOD
```

## 実務パターン: エラー設計のベストプラクティス

### エラーの握りつぶしを避ける

```rust
// BAD: エラーを無視しています
let _ = std::fs::write("data.txt", data);

// GOOD: エラーを伝播します
std::fs::write("data.txt", data)?;

// GOOD: 重要でない場合はログして続行
if let Err(e) = std::fs::write("data.txt", data) {
    eprintln!("警告: データの保存に失敗: {}", e);
}
```

### 文脈のあるエラーメッセージを書く

`with_context()` を使って「何をしようとして、何が起きたか」が分かるメッセージを付けましょう。

```rust
// BAD: 生のエラーだけ
let content = std::fs::read_to_string(&path)?;

// GOOD: 文脈付き
let content = std::fs::read_to_string(&path)
    .with_context(|| format!("ユーザーID {} のファイル '{}' を読み込めません", id, path))?;
```

### レイヤードアーキテクチャでのエラー変換

実務のプロジェクトでは、レイヤーごとにエラー型を定義し上位レイヤーで変換するのが一般的です。

```
インフラ層  → thiserror (InfraError: DB接続、ネットワーク)
    ↓ From
ドメイン層  → thiserror (DomainError: ユーザー未発見、残高不足)
    ↓ context()
アプリ層    → anyhow (文脈付きエラーメッセージ)
```

## まとめ

| 概念 | 要点 |
|:-----|:-----|
| Option\<T\> | 値の有無を型で表現します。`None` は正常な不在です |
| Result\<T, E\> | 成功/失敗を型で表現します。全エラーパスが明示されます |
| ? 演算子 | `Err`/`None` を早期リターンする構文糖衣です |
| From トレイト | `?` でのエラー型自動変換の仕組みです |
| カスタムエラー型 | 複数のエラーを enum で統合します |
| thiserror | ライブラリ向け。derive マクロでボイラープレートを削減します |
| anyhow | アプリ向け。`context()` でエラーチェーンを構成します |
| panic! | 回復不能なバグのみに使います |
| ハイブリッド設計 | ライブラリ層は thiserror、アプリ層は anyhow が定石です |

## やってみよう！

### 演習1: Option のコンビネータ

ユーザーIDから名前を検索し、見つかった名前の文字数を返す `user_name_length` 関数を実装してください。`find_user` を呼び、`map` で文字数に変換します。

```rust
fn find_user(id: u64) -> Option<String> {
    match id { 1 => Some("Alice".into()), 2 => Some("Bob".into()), _ => None }
}

fn user_name_length(id: u64) -> Option<usize> {
    // TODO: find_user と map を使って実装してください
    todo!()
}
```

### 演習2: thiserror でカスタムエラー型

`thiserror` を使って `ConfigError`（Io / Parse / Validation の3バリアント）を定義し、ファイルを読み込んでポート番号（1024以上）を返す `load_port` 関数を実装してください。

### 演習3: anyhow でエラーチェーン

JSON ファイルからユーザー一覧を読み込み、特定のユーザーを検索する関数を `anyhow` で実装してください。`context()` で「何をしようとしていたか」が分かるメッセージを付けることを意識しましょう。
