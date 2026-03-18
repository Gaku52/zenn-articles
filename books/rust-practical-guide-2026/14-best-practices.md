---
title: "Cargoとベストプラクティス"
---

本書の最終章では、Rust プロジェクトを実務品質で運用するためのベストプラクティスを学びます。Cargo の詳細設定からCI/CD、パフォーマンス最適化、そして本書の先にある学習テーマまでを網羅します。

## 1. Cargo.toml の詳細設定

### パッケージメタデータと依存関係

```toml
[package]
name = "my-lib"
version = "0.1.0"
edition = "2021"
rust-version = "1.75"           # MSRV（最低サポートバージョン）
description = "A short description"
license = "MIT OR Apache-2.0"
repository = "https://github.com/user/my-lib"
keywords = ["async", "web"]     # 最大5個

[dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1", features = ["full"], optional = true }

[dev-dependencies]              # テスト・ベンチマーク専用
criterion = { version = "0.5", features = ["html_reports"] }

[build-dependencies]            # build.rs 専用
cc = "1"
```

`dependencies` と `dev-dependencies` の区別は必須です。テスト用クレートを `dependencies` に入れると、利用者に不要な依存が伝播します。

### Feature フラグとプロファイル

```toml
[features]
default = ["json"]
json = ["dep:serde_json"]        # dep: でFeature名とクレート名を分離
async = ["dep:tokio"]
full = ["json", "async"]

[profile.release]
opt-level = 3                    # 最大最適化
lto = "thin"                    # リンク時最適化
strip = true                    # シンボル除去
codegen-units = 1               # 単一コンパイル単位
panic = "abort"                 # パニック時即終了

[profile.dev.package."*"]
opt-level = 2                   # 依存クレートだけ最適化（実行速度改善）
```

`[profile.dev.package."*"]` は実用的なテクニックです。自分のコードは高速コンパイルのままで、依存クレートだけ最適化するため開発時の実行速度が改善します。

## 2. ワークスペースによるモノレポ管理

プロジェクトが成長したら、ワークスペースで複数クレートを統一管理します。

```toml
# ルート Cargo.toml
[workspace]
members = ["crates/core", "crates/cli", "crates/server"]
resolver = "2"

[workspace.dependencies]        # バージョンを一元管理
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
anyhow = "1"

[workspace.package]
version = "0.1.0"
edition = "2021"
```

各クレートでは `workspace = true` で継承します。

```toml
# crates/cli/Cargo.toml
[package]
name = "project-cli"
version.workspace = true
edition.workspace = true

[dependencies]
project-core = { path = "../core" }
tokio.workspace = true
```

クレート間は**上位層から下位層への一方向依存**を守ります。循環依存はコンパイルエラーになるため、Rust では自然に強制されます。

```bash
cargo build -p project-cli      # 特定クレートのみビルド
cargo test --workspace           # 全クレートのテスト実行
```

## 3. cargo clippy による静的解析

`clippy` は 700 以上の lint ルールを持つ公式リンターです。

```bash
cargo clippy --all-targets --all-features     # 全対象チェック
cargo clippy -- -D warnings                   # 警告をエラーに（CI向け）
cargo clippy --fix                            # 自動修正
```

典型的な検出例を示します。

```rust
// NG: 手動ループ
let mut count = 0;
for item in &items { if item.is_active() { count += 1; } }

// OK: イテレータメソッド
let count = items.iter().filter(|i| i.is_active()).count();
```

`Cargo.toml` でプロジェクト全体の lint レベルを設定できます。

```toml
[lints.clippy]
pedantic = { level = "warn", priority = -1 }
unwrap_used = "deny"
must_use_candidate = "allow"
```

## 4. cargo fmt によるフォーマット統一

```bash
cargo fmt               # フォーマット実行
cargo fmt -- --check    # チェックのみ（CI向け）
```

`rustfmt.toml` をリポジトリに配置してルールを統一します。

```toml
edition = "2021"
max_width = 100
imports_granularity = "Crate"
group_imports = "StdExternalCrate"
reorder_imports = true
```

`group_imports = "StdExternalCrate"` で `use` 文が標準ライブラリ / 外部クレート / 自クレートの順に自動整理されます。

## 5. ドキュメントコメントと cargo doc

`///` ドキュメントコメントは Markdown で記述でき、コード例はそのままテストとして実行されます。

```rust
/// 2つの数値を加算します。
///
/// # Examples
/// ```
/// assert_eq!(my_crate::add(2, 3), 5);
/// ```
///
/// # Panics
/// デバッグビルドでオーバーフロー時にパニックします。
pub fn add(a: i32, b: i32) -> i32 { a + b }
```

ドキュメントでよく使うセクションは `# Examples`（使用例）、`# Errors`（エラー条件）、`# Panics`（パニック条件）、`# Safety`（unsafe の安全性条件）の4つです。

```bash
cargo doc --open                               # 生成してブラウザで開く
RUSTDOCFLAGS="-D warnings" cargo doc --no-deps # 壊れたリンクの検出
cargo test --doc                               # ドキュメントテストのみ
```

## 6. クレート公開（crates.io）

公開前に以下のチェックを行います。

```bash
cargo fmt -- --check && cargo clippy --all-features -- -D warnings
cargo test --all-features && cargo test --no-default-features
RUSTDOCFLAGS="-D warnings" cargo doc --all-features --no-deps
cargo audit                     # セキュリティ監査
cargo publish --dry-run         # ドライラン
```

確認が通ったら `cargo login` (初回のみ) して `cargo publish` で公開します。ワークスペースでは依存される側から順に公開し、インデックス更新（約30秒）を待ちます。

公開クレートは SemVer を厳守します。`PATCH` はバグ修正、`MINOR` は後方互換な機能追加、`MAJOR` は破壊的変更です。`0.x.y` の間はすべて破壊的変更扱いになります。

## 7. GitHub Actions での CI 設定

```yaml
name: CI
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
env:
  CARGO_TERM_COLOR: always
  RUSTFLAGS: "-D warnings"

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with: { components: "clippy, rustfmt" }
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt --all -- --check
      - run: cargo clippy --workspace --all-targets --all-features
      - run: cargo test --workspace --all-features

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: rustsec/audit-check@v2
        with: { token: "${{ secrets.GITHUB_TOKEN }}" }
```

`Swatinem/rust-cache@v2` でビルドキャッシュが効き CI 時間を大幅に短縮できます。`cargo-deny` を加えればライセンス・脆弱性の自動チェックも可能です。

## 8. プロジェクト構成のベストプラクティス

```
my-project/
├── Cargo.toml            # ワークスペースルート
├── rustfmt.toml / deny.toml / rust-toolchain.toml
├── .github/workflows/ci.yml
├── crates/
│   ├── app/              # バイナリクレート
│   ├── core/             # ドメインロジック（外部依存最小）
│   ├── api/              # Web API
│   └── shared/           # 共通ユーティリティ
├── tests/                # 統合テスト
└── benches/              # ベンチマーク
```

`rust-toolchain.toml` でチーム全体の Rust バージョンを固定します。

```toml
[toolchain]
channel = "1.78.0"
components = ["rustfmt", "clippy", "rust-analyzer"]
```

**やってはいけないこと**: ライブラリで `Cargo.lock` を commit する（利用者と競合）、`*` ワイルドカードバージョン、Feature の過剰分割、循環依存。

## 9. パフォーマンス最適化の基本

**鉄則: 推測するな、計測せよ。**

```bash
cargo build --timings            # ビルド時間分析
cargo tree --duplicates          # 重複依存の検出
cargo bloat --release --crates   # バイナリサイズ分析
```

ビルド時間短縮にはリンカ変更が効果的です。`.cargo/config.toml` で `mold`(Linux) や `lld`(macOS) を指定するだけで数十%短縮されることがあります。

実行時パフォーマンスのコツは3つです。(1) 引数は `String` ではなく `&str` で受ける、(2) `Vec::with_capacity` で事前確保する、(3) イテレータで中間割り当てを避ける。

```rust
// Cow で必要な時だけ所有権を取得
fn normalize(s: &str) -> Cow<'_, str> {
    if s.contains(char::is_uppercase) {
        Cow::Owned(s.to_lowercase())
    } else {
        Cow::Borrowed(s)  // 割り当てなし
    }
}
```

## 10. 次のステップ

本書で所有権・ライフタイム・トレイトの土台を固めたら、以下の発展テーマに進めます。

| テーマ | 概要 | 主要ツール/リソース |
|---|---|---|
| **WebAssembly** | ブラウザで動く高速アプリ | wasm-pack, Leptos, Yew, WASI |
| **組み込み開発** | `no_std` でマイコン制御 | Embassy, RTIC, probe-rs |
| **unsafe 深掘り** | FFI・カスタムアロケータ | bindgen, Miri, The Rustonomicon |
| **高度な非同期** | gRPC・ミドルウェア設計 | Tower, tonic, tokio-console |
| **マクロ** | derive / attribute マクロ自作 | proc-macro, syn, quote |
| **ゲーム開発** | ECS ベースの開発 | Bevy |

Rust は「システムプログラミングからアプリケーション開発まで」をカバーする言語です。本書で身につけた知識は、どの分野に進んでも土台として活き続けます。

---

## まとめ

| 項目 | 要点 |
|---|---|
| Cargo.toml | メタデータ・依存・features・プロファイルを適切に設定する |
| ワークスペース | `workspace.dependencies` でバージョンを一元管理する |
| cargo clippy | 700+ の lint でミスを防止。CI で必ず実行する |
| cargo fmt | `rustfmt.toml` でルールを統一し、フォーマット議論を排除する |
| ドキュメント | `///` + `# Examples` を書き、`cargo test --doc` で品質を保つ |
| クレート公開 | SemVer 厳守、`cargo publish --dry-run` で事前確認する |
| CI/CD | GitHub Actions で fmt / clippy / test / audit を自動化する |
| プロジェクト構成 | 依存は一方向、`rust-toolchain.toml` でバージョン固定する |
| パフォーマンス | 計測が先。Clone 削減・イテレータ活用・リンカ最適化 |

## やってみよう！

**演習1: CI ワークフローの構築**
自分の Rust プロジェクトに GitHub Actions を設定してください。`cargo fmt --check`、`cargo clippy -- -D warnings`、`cargo test --workspace`、`Swatinem/rust-cache@v2` を含めます。

**演習2: ワークスペース分割**
単一クレートのプロジェクトをワークスペースに分割してください。最低3クレート（`core`, `app`, `shared`）に分け、`workspace.dependencies` でバージョンを統一してみましょう。

**演習3: ドキュメントテスト**
主要な公開関数すべてに `///` コメントと `# Examples` を追加し、`cargo test --doc` で全テストが通ることを確認してください。

**演習4: パフォーマンス計測**
`cargo build --timings` でビルド時間を計測し、最も時間のかかる依存を特定してください。リンカ変更や `[profile.dev.package."*"]` 設定で改善幅を測定しましょう。
