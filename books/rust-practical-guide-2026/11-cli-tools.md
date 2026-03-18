---
title: "CLIツール開発"
---

# Chapter 11: CLIツール開発

> clap, indicatif, walkdir -- Rustで本格CLIツールを構築する

Rustはシングルバイナリで配布でき、起動が高速で、メモリ安全性も保証されます。ripgrep、fd、batなど著名CLIツールの多くがRustで書かれています。この章では、clapによる引数解析からクロスコンパイルまで、実用的なCLI開発の全工程を学びます。

---

## clapクレートによる引数パーサー（derive API）

`clap`はRust CLIの業界標準です。derive APIなら構造体定義だけで型安全な引数解析を実現できます。

```rust
use clap::{Parser, ValueEnum};
use std::path::PathBuf;

/// ファイル検索ツール
#[derive(Parser, Debug)]
#[command(name = "findrs", version, about)]
struct Cli {
    /// 検索パターン（正規表現）
    pattern: String,

    /// 検索対象ディレクトリ
    #[arg(short, long, default_value = ".")]
    dir: PathBuf,

    /// 最大結果数
    #[arg(short = 'n', long, default_value_t = 100)]
    max_results: usize,

    /// 詳細出力を有効にする
    #[arg(short, long)]
    verbose: bool,

    /// 出力フォーマット
    #[arg(short, long, value_enum, default_value_t = Format::Text)]
    format: Format,
}

#[derive(ValueEnum, Clone, Debug)]
enum Format { Text, Json, Csv }

fn main() {
    let cli = Cli::parse();
    println!("検索: '{}' in {:?}", cli.pattern, cli.dir);
}
// $ findrs "TODO" --dir src/ -n 50 --verbose
```

`#[arg(...)]`属性で短縮名、デフォルト値、バリデーションを制御します。`value_parser`でポート番号の範囲チェックやファイル存在確認など、独自の検証も追加できます。

---

## サブコマンドの定義

`git commit`のようなサブコマンドは`#[derive(Subcommand)]`でenumとして定義します。

```rust
use clap::{Parser, Subcommand};
use std::path::PathBuf;

#[derive(Parser)]
#[command(name = "filetool", version, about = "ファイル管理ツール")]
struct Cli {
    #[arg(short, long, global = true)]
    verbose: bool,

    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// ファイルを検索する
    Search {
        pattern: String,
        #[arg(short, long, default_value = ".")]
        dir: PathBuf,
        #[arg(short, long)]
        extension: Option<String>,
    },
    /// ディレクトリを分析する
    Analyze {
        #[arg(default_value = ".")]
        path: PathBuf,
        #[arg(short, long)]
        depth: Option<usize>,
    },
}

fn main() {
    let cli = Cli::parse();
    match &cli.command {
        Commands::Search { pattern, dir, extension } => {
            println!("検索: '{}' in {:?}", pattern, dir);
        }
        Commands::Analyze { path, depth } => {
            println!("分析: {:?} (深度: {:?})", path, depth);
        }
    }
}
// $ filetool search "TODO" --dir src/ --extension rs
```

`global = true`を指定したフラグは全サブコマンドで有効になります。`--verbose`のようなツール全体設定に便利です。

---

## 環境変数と設定ファイルの読み込み

CLIツールでは「引数 > 環境変数 > 設定ファイル > デフォルト値」の優先順位で設定を解決します。clapは`#[arg(env = "...")]`で環境変数フォールバックに対応しています。

```rust
use clap::Parser;

#[derive(Parser)]
struct Cli {
    #[arg(long, env = "MY_API_KEY")]
    api_key: String,

    #[arg(long, env = "RUST_LOG", default_value = "info")]
    log_level: String,
}
// $ MY_API_KEY=secret myapp --log-level debug
```

設定ファイルは`directories`クレート（XDG準拠）と`toml`で管理します。macOSでは`~/Library/Application Support/`、Linuxでは`~/.config/`に自動的にマッピングされます。

---

## 標準入出力とパイプ処理

Unix哲学に従い、パイプ対応のCLIツールを作りましょう。

```rust
use std::io::{self, BufRead, Write, IsTerminal, BufWriter};

fn main() -> anyhow::Result<()> {
    let stdin = io::stdin();
    let stdout = io::stdout();
    let mut out = BufWriter::new(stdout.lock());

    if stdin.is_terminal() {
        eprintln!("テキストを入力してください（Ctrl+D で終了）:");
    }

    for line in stdin.lock().lines() {
        let line = line?;
        writeln!(out, "{}", line.to_uppercase())?;
    }
    out.flush()?;
    Ok(())
}
// $ echo "hello" | myapp          ← パイプ入力
// $ myapp < input.txt > output.txt  ← リダイレクト
```

重要なポイントは3つです。`is_terminal()`でTTYかパイプかを判定すること、`BufWriter`で出力をバッファリングすること（大量出力で10-100倍高速化）、そしてエラーは`eprintln!`（stderr）に出力してパイプ時の混入を防ぐことです。

---

## プログレスバーと色付き出力（indicatif, colored）

```rust
use colored::Colorize;
use indicatif::{ProgressBar, ProgressStyle};

fn main() {
    // カラー出力
    println!("{}", "成功!".green().bold());
    println!("{}", "警告: ディスク容量低下".yellow());
    println!("{}", "エラー: ファイル破損".red().bold());

    // プログレスバー
    let pb = ProgressBar::new(100);
    pb.set_style(ProgressStyle::with_template(
        "{spinner:.green} [{bar:40.cyan/blue}] {pos}/{len} ({eta})"
    ).unwrap().progress_chars("=>-"));

    for i in 0..100 {
        pb.set_position(i);
        std::thread::sleep(std::time::Duration::from_millis(20));
    }
    pb.finish_with_message("完了!");
}
```

パイプ先ではカラーコードが不要です。`NO_COLOR`環境変数とTTY判定を冒頭で設定しましょう。

```rust
fn setup_color() {
    if std::env::var("NO_COLOR").is_ok() || !std::io::stdout().is_terminal() {
        colored::control::set_override(false);
    }
}
```

---

## ファイル操作（std::fs, walkdir）

`walkdir`クレートで再帰的なディレクトリ走査を効率的に行えます。

```rust
use std::path::Path;
use walkdir::WalkDir;

fn search_files(dir: &Path, ext: Option<&str>, max_depth: Option<usize>) -> Vec<walkdir::DirEntry> {
    let mut walker = WalkDir::new(dir);
    if let Some(d) = max_depth { walker = walker.max_depth(d); }

    walker.into_iter()
        .filter_map(|e| e.ok())          // 権限エラーをスキップ
        .filter(|e| e.file_type().is_file())
        .filter(|e| match ext {
            Some(x) => e.path().extension().map_or(false, |ex| ex == x),
            None => true,
        })
        .collect()
}
```

`filter_map(|e| e.ok())`でアクセス権限エラーを安全にスキップできます。シンボリックリンクのループ検出や深度制限も組み込まれています。

---

## クロスコンパイルと配布

`cross`はDockerコンテナ内でビルドするため、ターゲットのツールチェインを個別にインストールする必要がありません。

```bash
cargo install cross
cross build --release --target x86_64-unknown-linux-musl   # Linux（静的リンク）
cross build --release --target x86_64-pc-windows-gnu        # Windows
cross build --release --target aarch64-unknown-linux-gnu    # ARM Linux
```

バイナリサイズの最適化には、`Cargo.toml`の`[profile.release]`で`strip = true`（30-50%削減）、`lto = true`（10-20%削減）、`panic = "abort"`（5-10%削減）を設定します。

配布の自動化には`cargo-dist`が便利です。`cargo dist init`で初期化し、タグプッシュすればGitHub Actionsがマルチプラットフォームのバイナリをビルドしてリリースします。Homebrewのtapやシェルインストーラーも自動生成されます。

---

## 実践プロジェクト: ファイル検索ツールの構築

ここまでの知識を統合した`findrs`の実装です。

```rust
use anyhow::{Context, Result};
use clap::Parser;
use colored::Colorize;
use indicatif::{ProgressBar, ProgressStyle};
use regex::Regex;
use std::io::{IsTerminal, Write, BufWriter};
use std::path::PathBuf;
use walkdir::WalkDir;

#[derive(Parser)]
#[command(name = "findrs", version, about = "高速ファイル検索ツール")]
struct Cli {
    pattern: String,
    #[arg(short, long, default_value = ".")]
    dir: PathBuf,
    #[arg(short = 'n', long, default_value_t = 100)]
    max_results: usize,
    #[arg(short, long)]
    extension: Option<String>,
    #[arg(short, long)]
    verbose: bool,
}

fn main() -> Result<()> {
    let cli = Cli::parse();
    if !std::io::stdout().is_terminal() { colored::control::set_override(false); }

    let regex = Regex::new(&cli.pattern)
        .context(format!("無効な正規表現: '{}'", cli.pattern))?;

    let entries: Vec<_> = WalkDir::new(&cli.dir).into_iter()
        .filter_map(|e| e.ok())
        .filter(|e| e.file_type().is_file())
        .filter(|e| match &cli.extension {
            Some(ext) => e.path().extension().map_or(false, |x| x == ext.as_str()),
            None => true,
        }).collect();

    let pb = std::io::stderr().is_terminal().then(|| {
        let pb = ProgressBar::new(entries.len() as u64);
        pb.set_style(ProgressStyle::with_template(
            "{spinner:.green} [{bar:30}] {pos}/{len} 検索中..."
        ).unwrap());
        pb
    });

    let stdout = std::io::stdout();
    let mut out = BufWriter::new(stdout.lock());
    let mut count = 0;

    for entry in &entries {
        if let Some(ref pb) = pb { pb.inc(1); }
        let path = entry.path();
        let content = match std::fs::read_to_string(path) {
            Ok(c) => c,
            Err(_) => continue,
        };
        for (i, line) in content.lines().enumerate() {
            if regex.is_match(line) {
                writeln!(out, "{}:{}: {}",
                    path.display().to_string().cyan(),
                    (i + 1).to_string().yellow(), line)?;
                count += 1;
                if count >= cli.max_results { break; }
            }
        }
        if count >= cli.max_results { break; }
    }

    if let Some(pb) = pb { pb.finish_and_clear(); }
    out.flush()?;
    eprintln!("{} {} 件マッチ", "結果:".green().bold(), count);
    Ok(())
}
```

```bash
$ findrs "TODO|FIXME" --dir src/ --extension rs
$ findrs "fn\s+\w+" -n 50 --verbose
$ findrs "use std" --dir src/ | wc -l    # パイプで後続処理へ
```

---

## まとめ

| 概念 | 要点 |
|------|------|
| clap derive | `#[derive(Parser)]`で型安全なCLI定義。`value_enum`でenum引数、`value_parser`でバリデーション |
| サブコマンド | `#[derive(Subcommand)]`でenumベース定義。`global = true`で共通フラグを全コマンドに適用 |
| 環境変数 | `#[arg(env = "...")]`で環境変数フォールバック。優先順位: 引数 > 環境変数 > デフォルト |
| 設定ファイル | directories（XDG準拠）+ tomlでOS横断の設定管理 |
| パイプ対応 | `is_terminal()`でTTY判定。BufWriterで大量出力を高速化。エラーはstderrへ |
| カラー・進捗 | colored + indicatifでUX向上。NO_COLOR規約を尊重すること |
| walkdir | 再帰ディレクトリ走査。権限エラーのスキップ、深度制限、ループ検出 |
| クロスコンパイル | crossでDockerベースのクロスビルド。musl静的リンクで環境非依存バイナリ |
| バイナリ最適化 | strip + lto + panic=abortで30-60%のサイズ削減 |
| 配布自動化 | cargo-distでタグプッシュ時にマルチプラットフォーム自動リリース |

---

## やってみよう！

### 演習1: clapでワードカウントツール

`wordcount`コマンドを作ってみましょう。

- 引数でファイルパスを受け取る（`Vec<PathBuf>`で複数指定可能）
- `--lines`フラグで行数のみ、`--words`で単語数のみ表示
- フラグなしの場合は行数・単語数・バイト数をすべて表示
- ファイル省略時はstdinから読み取り（パイプ対応）

### 演習2: サブコマンド付き文字列変換ツール

`textool`を作り、`upper`/`lower`/`count`サブコマンドを実装しましょう。

- `textool upper` -- 大文字変換、`textool lower` -- 小文字変換
- `textool count` -- 行数・単語数・文字数を表示
- `$ echo "Hello" | textool upper` で `HELLO` と出力されることを確認

### 演習3: プログレスバー付きディレクトリハッシュ計算

ディレクトリ内ファイルのSHA256を計算するツールを作ってみましょう。

- walkdirでファイルを再帰走査し、indicatifでプログレスバーを表示
- `--extension`で対象拡張子をフィルタ
- crossでLinux向けにクロスコンパイルしてみましょう
