# Go CLI配布

本章では、Go CLIツールのビルドと配布方法を学びます。

## クロスコンパイル

```bash
# Linux
GOOS=linux GOARCH=amd64 go build -o mycli-linux-amd64

# macOS (Intel)
GOOS=darwin GOARCH=amd64 go build -o mycli-darwin-amd64

# macOS (Apple Silicon)
GOOS=darwin GOARCH=arm64 go build -o mycli-darwin-arm64

# Windows
GOOS=windows GOARCH=amd64 go build -o mycli-windows-amd64.exe
```

## サイズ最適化

```bash
go build -ldflags="-s -w" -o mycli
```

## Homebrew Formula

```ruby
class Mycli < Formula
  desc "A powerful CLI tool"
  homepage "https://github.com/username/my-cli-tool"
  url "https://github.com/username/my-cli-tool/archive/v1.0.0.tar.gz"
  sha256 "..."

  depends_on "go" => :build

  def install
    system "go", "build", *std_go_args
  end

  test do
    system "#{bin}/mycli", "--version"
  end
end
```

## まとめ

Goのクロスコンパイル機能により、複数プラットフォーム向けのバイナリを簡単に配布できます。
