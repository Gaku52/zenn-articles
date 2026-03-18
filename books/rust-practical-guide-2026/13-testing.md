---
title: "テスト戦略"
---

# テスト戦略

> 書いたコードに自信を持つために — Rustのテストエコシステムを使いこなす

## この章で学ぶこと

ユニットテスト、インテグレーションテスト、プロパティベーステスト、ベンチマーク、CI/CDでの自動化をカバーします。

---

## ユニットテストの基本

Rustでは`#[test]`属性をつけた関数がテストになります。`#[cfg(test)]`で囲むと本番ビルドから除外されます。

```rust
pub fn add(a: i64, b: i64) -> i64 { a + b }

pub fn divide(a: f64, b: f64) -> Result<f64, &'static str> {
    if b == 0.0 { Err("ゼロ除算エラー") } else { Ok(a / b) }
}

fn clamp(value: i64, min: i64, max: i64) -> i64 {
    value.max(min).min(max)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
        assert_eq!(add(-1, 1), 0);
    }

    #[test]
    fn test_divide() {
        assert!((divide(10.0, 3.0).unwrap() - 3.333).abs() < 0.001);
        assert!(divide(1.0, 0.0).is_err());
    }

    // プライベート関数もテスト可能
    #[test]
    fn test_clamp() {
        assert_eq!(clamp(5, 0, 10), 5);
        assert_eq!(clamp(-1, 0, 10), 0);
    }
}
```

`use super::*;`で親モジュールの関数（プライベート含む）にアクセスできます。**Rustではプライベート関数もユニットテストで直接検証できる**のが特徴です。

---

## assertマクロ群

| マクロ | 用途 | 失敗時の出力 |
|---|---|---|
| `assert!(expr)` | 真偽値の検証 | `"assertion failed"` |
| `assert_eq!(a, b)` | 等値比較 | leftとrightの値を表示 |
| `assert_ne!(a, b)` | 非等値比較 | leftとrightが同じであることを表示 |

いずれもカスタムメッセージを追加できます。浮動小数点は差分で判定します。

```rust
assert!(x > 0, "x は正数であるべき: x = {}", x);
assert_eq!(vec![1, 2, 3], vec![1, 2, 3], "ベクタが一致しない");
// 浮動小数点はassert_eq!ではなく差分で判定
assert!((0.1 + 0.2 - 0.3).abs() < 1e-10);
```

---

## #[should_panic]とResultを返すテスト

パニックを期待するテストには`#[should_panic]`を使います。`expected`でメッセージの一部を指定できます。

```rust
#[test]
#[should_panic(expected = "index out of bounds")]
fn test_index_panic() {
    let v = vec![1, 2, 3];
    let _ = v[5];
}
```

`?`演算子を使いたい場合は`Result`を返す形にします。

```rust
#[test]
fn test_parse() -> Result<(), Box<dyn std::error::Error>> {
    let value: i32 = "42".parse()?;
    assert_eq!(value, 42);
    Ok(())
}
```

`#[ignore]`属性で通常スキップし、`cargo test -- --ignored`で明示実行できます。

---

## テストの構成 — tests/ディレクトリ

Rustのテストは**ユニットテスト**（`src/`内）、**インテグレーションテスト**（`tests/`）、**ドキュメントテスト**（`///`内コードブロック）の3種類です。

```
my-project/
├── src/lib.rs          # ユニットテスト（#[cfg(test)]）
├── tests/
│   ├── common/mod.rs   # 共通ヘルパー（テスト実行されない）
│   └── api_test.rs     # 独立テストクレート
└── benches/bench.rs    # ベンチマーク
```

`tests/`内は**外部クレートとして扱われる**ため、公開APIのみテスト可能です。共通ヘルパーは`tests/common/mod.rs`に配置します。

```rust
// tests/common/mod.rs
pub struct TestContext { pub temp_dir: tempfile::TempDir }
impl TestContext {
    pub fn new() -> Self {
        TestContext { temp_dir: tempfile::tempdir().unwrap() }
    }
    pub fn write_file(&self, name: &str, content: &str) {
        std::fs::write(self.temp_dir.path().join(name), content).unwrap();
    }
}

// tests/integration_test.rs
mod common;

#[test]
fn test_workflow() {
    let ctx = common::TestContext::new();
    ctx.write_file("input.txt", "10\n20\n30");
    let sum: i64 = std::fs::read_to_string(ctx.temp_dir.path().join("input.txt"))
        .unwrap().lines().filter_map(|l| l.parse::<i64>().ok()).sum();
    assert_eq!(sum, 60);
}
```

:::message
`tests/`内のファイルはそれぞれ独立バイナリとしてコンパイルされます。ファイル数が多い場合は`tests/integration.rs`に`mod`で束ねるとビルドが速くなります。
:::

---

## テストフィクスチャとセットアップ

ビルダーパターンのフィクスチャで重複を減らせます。`rstest`を使えばパラメータ化テストも簡潔です。

```rust
struct TestFixture { service: UserService }
impl TestFixture {
    fn new() -> Self {
        TestFixture { service: UserService::new(MockDatabase::new()) }
    }
    fn with_users(self, count: usize) -> Self {
        for i in 0..count {
            self.service.create_user(&format!("user{}@example.com", i));
        }
        self
    }
}

#[test]
fn test_list_users() {
    let f = TestFixture::new().with_users(5);
    assert_eq!(f.service.list_users().len(), 5);
}
```

```rust
use rstest::rstest;

#[rstest]
#[case("", true)]
#[case("hello", false)]
fn test_is_blank(#[case] input: &str, #[case] expected: bool) {
    assert_eq!(input.trim().is_empty(), expected);
}
```

---

## モックとテストダブル（mockall）

traitで依存を定義し、`#[automock]`でモックを自動生成します。`expect_*`で呼び出し回数・引数・戻り値を指定できます。

```rust
use mockall::{automock, predicate::*};

#[automock]
trait UserRepository {
    fn find_by_id(&self, id: u64) -> Option<User>;
    fn save(&self, user: &User) -> Result<(), String>;
}

#[test]
fn test_register_user() {
    let mut mock = MockUserRepository::new();
    mock.expect_find_by_id()
        .with(eq(42)).times(1).returning(|_| None);
    mock.expect_save()
        .times(1).returning(|_| Ok(()));

    let service = UserService::new(mock);
    assert!(service.register(42, "new_user").is_ok());
}
```

テストダブルは用途に応じて使い分けましょう。

| 種類 | 用途 | 例 |
|---|---|---|
| **Stub** | 固定値を返す | 常に`Ok(())`を返す |
| **Mock** | 呼び出しを検証 | `expect_save().times(1)` |
| **Fake** | 簡易実装 | インメモリDB |

---

## プロパティベーステスト（proptest）

proptestはランダム入力を大量に生成し、「どんな入力でも成り立つ性質」を検証します。失敗時は反例を自動最小化（shrinking）します。

```rust
use proptest::prelude::*;

fn insertion_sort(mut arr: Vec<i32>) -> Vec<i32> {
    for i in 1..arr.len() {
        let key = arr[i];
        let mut j = i;
        while j > 0 && arr[j - 1] > key { arr[j] = arr[j - 1]; j -= 1; }
        arr[j] = key;
    }
    arr
}

proptest! {
    #[test]
    fn sort_is_ordered(v in prop::collection::vec(any::<i32>(), 0..100)) {
        let sorted = insertion_sort(v);
        for w in sorted.windows(2) { prop_assert!(w[0] <= w[1]); }
    }

    #[test]
    fn sort_preserves_length(v in prop::collection::vec(any::<i32>(), 0..100)) {
        prop_assert_eq!(v.len(), insertion_sort(v.clone()).len());
    }
}
```

失敗した入力は`proptest-regressions/`に保存され、次回テストで自動再実行されます。このディレクトリはGitにコミットしてください。

---

## ベンチマーク（criterion）

criterionは統計的なパフォーマンス計測を行います。`benches/`に配置し、`Cargo.toml`で`harness = false`を指定します。

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion, BenchmarkId};

fn fib_iter(n: u64) -> u64 {
    let (mut a, mut b) = (0u64, 1u64);
    for _ in 0..n { let t = b; b = a + b; a = t; }
    a
}

fn bench(c: &mut Criterion) {
    let mut group = c.benchmark_group("fibonacci");
    for n in [10, 20, 30] {
        group.bench_with_input(BenchmarkId::new("iterative", n), &n,
            |b, &n| b.iter(|| fib_iter(black_box(n))));
    }
    group.finish();
}

criterion_group!(benches, bench);
criterion_main!(benches);
```

```toml
[[bench]]
name = "fib_bench"
harness = false

[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }
```

`black_box()`でコンパイラの最適化を防ぎ、正確な計測を実現します。結果は`target/criterion/report/index.html`で確認できます。

---

## cargo testのオプションとフィルタリング

| コマンド | 説明 |
|---|---|
| `cargo test` | 全テスト実行 |
| `cargo test --lib` | ユニットテストのみ |
| `cargo test --test integration` | 特定インテグレーションテスト |
| `cargo test --doc` | ドキュメントテスト |
| `cargo test parse` | 名前に`parse`を含むテスト |
| `cargo test -- --nocapture` | `println!`出力を表示 |
| `cargo test -- --test-threads=1` | シリアル実行 |
| `cargo test -- --ignored` | `#[ignore]`テストのみ |

高速テストランナー`cargo-nextest`を使うとCI実行が大幅に高速化します。

```bash
cargo install cargo-nextest
cargo nextest run                # 高速並列実行
cargo nextest run --retries 2    # 失敗時リトライ
```

---

## CI/CDでのテスト自動化

GitHub Actionsでのテスト・カバレッジ自動化の構成例です。

```yaml
name: Test
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo nextest run --workspace
      - run: cargo install cargo-llvm-cov
      - run: cargo llvm-cov --lcov --output-path lcov.info
      - uses: codecov/codecov-action@v4
        with:
          files: lcov.info
```

カバレッジ閾値を設定すれば品質を強制できます。

```bash
cargo llvm-cov --html                 # HTMLレポート生成
cargo llvm-cov --fail-under-lines 80  # 80%未満でCI失敗
```

---

## まとめ

| 項目 | 要点 |
|---|---|
| `#[test]` + `#[cfg(test)]` | ユニットテストの基本。プライベート関数もテスト可能 |
| `assert!` / `assert_eq!` / `assert_ne!` | 3つの基本assertマクロ。カスタムメッセージ対応 |
| `tests/`ディレクトリ | インテグレーションテスト。公開APIのみ検証 |
| `#[should_panic]` | パニックを期待するテスト。`expected`でメッセージ指定 |
| `Result`を返すテスト | `?`演算子が使える。`Err`返却で失敗 |
| フィクスチャ / rstest | ビルダーパターンやパラメータ化でセットアップ共通化 |
| mockall | `#[automock]`で自動モック生成。呼び出しを検証 |
| proptest | ランダム入力で性質を検証。反例を自動最小化 |
| criterion | 統計的ベンチマーク。`black_box`で最適化防止 |
| cargo-nextest | 高速並列テストランナー。CI/CDに最適 |
| cargo-llvm-cov | カバレッジ計測。閾値チェックでCI連携 |

## やってみよう！

**演習1**: 文字列の母音数を返す`count_vowels`関数を実装し、正常系・空文字列・大文字混在のテストを`assert_eq!`で書いてください。

**演習2**: `tests/common/mod.rs`に一時ファイル作成ヘルパーを定義し、インテグレーションテストから利用する構成を作ってください。

**演習3**: `trait Storage { fn get(&self, key: &str) -> Option<String>; fn set(&self, key: &str, value: &str) -> Result<(), String>; }`に`#[automock]`を付けてモックテストを書いてください。

**演習4**: `fn reverse(s: &str) -> String`を実装し、proptestで「2回reverseすると元に戻る」「長さが変わらない」を検証してください。
