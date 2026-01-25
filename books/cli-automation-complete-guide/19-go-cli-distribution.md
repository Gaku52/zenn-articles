---
title: "Go CLIの配布"
---

# Go CLIの配布

Goで開発したCLIツールは、単一バイナリとして配布できるため、ユーザーへの提供が非常に簡単です。本章では、クロスコンパイル、GoReleaser、Homebrew、GitHub Releasesなど、Go CLIツールの配布方法を学びます。

## Goのクロスコンパイル

Goの最大の特徴の一つが、異なるプラットフォーム向けのバイナリを単一の開発環境から簡単にビルドできることです。

### 基本的なクロスコンパイル

GOOS（OS）とGOARCH（アーキテクチャ）環境変数を指定してビルドします。

```bash
# Linux (amd64)
GOOS=linux GOARCH=amd64 go build -o mycli-linux-amd64 cmd/mycli/main.go

# Linux (arm64)
GOOS=linux GOARCH=arm64 go build -o mycli-linux-arm64 cmd/mycli/main.go

# macOS (Intel)
GOOS=darwin GOARCH=amd64 go build -o mycli-darwin-amd64 cmd/mycli/main.go

# macOS (Apple Silicon)
GOOS=darwin GOARCH=arm64 go build -o mycli-darwin-arm64 cmd/mycli/main.go

# Windows (amd64)
GOOS=windows GOARCH=amd64 go build -o mycli-windows-amd64.exe cmd/mycli/main.go

# Windows (arm64)
GOOS=windows GOARCH=arm64 go build -o mycli-windows-arm64.exe cmd/mycli/main.go
```

### サポートされているプラットフォーム

```bash
# 利用可能なOS/アーキテクチャの組み合わせを確認
go tool dist list
```

主要なプラットフォームは以下の通りです。

- `linux/amd64`, `linux/arm64`, `linux/386`
- `darwin/amd64`, `darwin/arm64`
- `windows/amd64`, `windows/386`, `windows/arm64`
- `freebsd/amd64`, `openbsd/amd64`

## バイナリサイズの最適化

デフォルトのビルドには、デバッグ情報やシンボルテーブルが含まれます。リリース用ビルドでは、これらを削除してサイズを削減します。

### ldflagsによる最適化

```bash
# デバッグ情報とシンボルテーブルの削除
go build -ldflags="-s -w" -o mycli cmd/mycli/main.go

# オプションの説明:
# -s: シンボルテーブルを削除
# -w: DWARFデバッグ情報を削除
```

### バージョン情報の埋め込み

ビルド時にバージョン情報を埋め込むことができます。

```go
// version/version.go
package version

var (
    // これらの変数はビルド時に-ldflagsで設定される
    Version   = "dev"
    GitCommit = "none"
    BuildDate = "unknown"
)

func GetVersion() string {
    return Version
}

func GetFullVersion() string {
    return fmt.Sprintf("%s (commit: %s, built: %s)", Version, GitCommit, BuildDate)
}
```

ビルドコマンド:

```bash
# バージョン情報を埋め込んでビルド
go build -ldflags="-s -w \
    -X github.com/username/my-cli-tool/version.Version=1.0.0 \
    -X github.com/username/my-cli-tool/version.GitCommit=$(git rev-parse --short HEAD) \
    -X github.com/username/my-cli-tool/version.BuildDate=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
    -o mycli cmd/mycli/main.go
```

### UPXによる圧縮

UPX（Ultimate Packer for eXecutables）を使用してバイナリをさらに圧縮できます。

```bash
# UPXのインストール
brew install upx  # macOS
sudo apt install upx  # Ubuntu/Debian

# バイナリの圧縮
upx --best --lzma mycli

# 圧縮前後のサイズ比較
ls -lh mycli
```

注意: 圧縮されたバイナリは、一部のウイルススキャナで誤検知される可能性があります。

## Makefileによるビルド自動化

複数プラットフォーム向けのビルドを自動化します。

```makefile
# Makefile
.PHONY: build build-all clean install test lint

# 変数定義
BINARY_NAME=mycli
VERSION?=$(shell git describe --tags --always --dirty)
GIT_COMMIT=$(shell git rev-parse --short HEAD)
BUILD_DATE=$(shell date -u +%Y-%m-%dT%H:%M:%SZ)
LDFLAGS=-s -w \
    -X github.com/username/my-cli-tool/version.Version=$(VERSION) \
    -X github.com/username/my-cli-tool/version.GitCommit=$(GIT_COMMIT) \
    -X github.com/username/my-cli-tool/version.BuildDate=$(BUILD_DATE)

# ローカルビルド
build:
	go build -ldflags="$(LDFLAGS)" -o $(BINARY_NAME) cmd/mycli/main.go

# インストール
install: build
	install -d $(GOPATH)/bin
	install -m 755 $(BINARY_NAME) $(GOPATH)/bin/

# 全プラットフォーム向けビルド
build-all:
	mkdir -p dist

	# Linux
	GOOS=linux GOARCH=amd64 go build -ldflags="$(LDFLAGS)" -o dist/$(BINARY_NAME)-linux-amd64 cmd/mycli/main.go
	GOOS=linux GOARCH=arm64 go build -ldflags="$(LDFLAGS)" -o dist/$(BINARY_NAME)-linux-arm64 cmd/mycli/main.go

	# macOS
	GOOS=darwin GOARCH=amd64 go build -ldflags="$(LDFLAGS)" -o dist/$(BINARY_NAME)-darwin-amd64 cmd/mycli/main.go
	GOOS=darwin GOARCH=arm64 go build -ldflags="$(LDFLAGS)" -o dist/$(BINARY_NAME)-darwin-arm64 cmd/mycli/main.go

	# Windows
	GOOS=windows GOARCH=amd64 go build -ldflags="$(LDFLAGS)" -o dist/$(BINARY_NAME)-windows-amd64.exe cmd/mycli/main.go
	GOOS=windows GOARCH=arm64 go build -ldflags="$(LDFLAGS)" -o dist/$(BINARY_NAME)-windows-arm64.exe cmd/mycli/main.go

# アーカイブ作成
archive: build-all
	cd dist && \
	tar czf $(BINARY_NAME)-linux-amd64.tar.gz $(BINARY_NAME)-linux-amd64 && \
	tar czf $(BINARY_NAME)-linux-arm64.tar.gz $(BINARY_NAME)-linux-arm64 && \
	tar czf $(BINARY_NAME)-darwin-amd64.tar.gz $(BINARY_NAME)-darwin-amd64 && \
	tar czf $(BINARY_NAME)-darwin-arm64.tar.gz $(BINARY_NAME)-darwin-arm64 && \
	zip $(BINARY_NAME)-windows-amd64.zip $(BINARY_NAME)-windows-amd64.exe && \
	zip $(BINARY_NAME)-windows-arm64.zip $(BINARY_NAME)-windows-arm64.exe

# チェックサム生成
checksum: archive
	cd dist && shasum -a 256 *.tar.gz *.zip > checksums.txt

# テスト
test:
	go test -v -race ./...

# リント
lint:
	go vet ./...
	staticcheck ./...

# クリーン
clean:
	rm -f $(BINARY_NAME)
	rm -rf dist/
```

使用例:

```bash
# ローカルビルド
make build

# 全プラットフォーム向けビルド
make build-all

# アーカイブとチェックサム作成
make archive checksum

# クリーン
make clean
```

## GoReleaserによる自動リリース

GoReleaserは、Go向けの自動リリースツールで、ビルド、アーカイブ、GitHub Releasesへの公開を自動化します。

### GoReleaserのインストール

```bash
# macOS
brew install goreleaser

# Linux
go install github.com/goreleaser/goreleaser@latest
```

### .goreleaser.yamlの設定

```yaml
# .goreleaser.yaml
version: 2

before:
  hooks:
    - go mod tidy
    - go test ./...

builds:
  - id: mycli
    main: ./cmd/mycli
    binary: mycli
    env:
      - CGO_ENABLED=0
    goos:
      - linux
      - darwin
      - windows
    goarch:
      - amd64
      - arm64
    ldflags:
      - -s -w
      - -X github.com/username/my-cli-tool/version.Version={{.Version}}
      - -X github.com/username/my-cli-tool/version.GitCommit={{.Commit}}
      - -X github.com/username/my-cli-tool/version.BuildDate={{.Date}}

archives:
  - id: mycli
    format: tar.gz
    name_template: "{{ .ProjectName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}"
    format_overrides:
      - goos: windows
        format: zip
    files:
      - README.md
      - LICENSE
      - CHANGELOG.md

checksum:
  name_template: "checksums.txt"
  algorithm: sha256

changelog:
  sort: asc
  filters:
    exclude:
      - "^docs:"
      - "^test:"
      - "^chore:"

release:
  github:
    owner: username
    name: my-cli-tool
  draft: false
  prerelease: auto
  name_template: "{{.ProjectName}} v{{.Version}}"
  header: |
    ## What's Changed
  footer: |
    ## Installation

    ### macOS
    ```bash
    brew install username/tap/mycli
    ```

    ### Linux
    ```bash
    wget https://github.com/username/my-cli-tool/releases/download/v{{.Version}}/mycli_{{.Version}}_linux_amd64.tar.gz
    tar xzf mycli_{{.Version}}_linux_amd64.tar.gz
    sudo mv mycli /usr/local/bin/
    ```

# Homebrew Tap
brews:
  - repository:
      owner: username
      name: homebrew-tap
    name: mycli
    homepage: "https://github.com/username/my-cli-tool"
    description: "A powerful CLI tool for project management"
    license: "MIT"
    dependencies:
      - name: git
    install: |
      bin.install "mycli"
    test: |
      system "#{bin}/mycli", "--version"
```

### リリースの実行

```bash
# ローカルテスト（ビルドのみ、公開しない）
goreleaser release --snapshot --clean

# リリース実行（タグをプッシュした後）
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0
goreleaser release --clean
```

## GitHub Actionsでの自動リリース

CI/CDパイプラインでリリースプロセスを自動化します。

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - "v*"

permissions:
  contents: write

jobs:
  goreleaser:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.21"

      - name: Run tests
        run: go test -v ./...

      - name: Run GoReleaser
        uses: goreleaser/goreleaser-action@v5
        with:
          distribution: goreleaser
          version: latest
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

タグをプッシュすると、自動的にリリースが作成されます。

```bash
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0
```

## Homebrewでの配布

macOSユーザー向けに、Homebrewでの配布が推奨されます。

### Homebrew Tap作成

```bash
# Tapリポジトリの作成
mkdir homebrew-tap
cd homebrew-tap
git init

# Formula作成
cat > mycli.rb << 'EOF'
class Mycli < Formula
  desc "A powerful CLI tool for project management"
  homepage "https://github.com/username/my-cli-tool"
  version "1.0.0"

  on_macos do
    if Hardware::CPU.arm?
      url "https://github.com/username/my-cli-tool/releases/download/v1.0.0/mycli_1.0.0_darwin_arm64.tar.gz"
      sha256 "YOUR_SHA256_HERE"
    else
      url "https://github.com/username/my-cli-tool/releases/download/v1.0.0/mycli_1.0.0_darwin_amd64.tar.gz"
      sha256 "YOUR_SHA256_HERE"
    end
  end

  on_linux do
    if Hardware::CPU.arm? && Hardware::CPU.is_64_bit?
      url "https://github.com/username/my-cli-tool/releases/download/v1.0.0/mycli_1.0.0_linux_arm64.tar.gz"
      sha256 "YOUR_SHA256_HERE"
    else
      url "https://github.com/username/my-cli-tool/releases/download/v1.0.0/mycli_1.0.0_linux_amd64.tar.gz"
      sha256 "YOUR_SHA256_HERE"
    end
  end

  def install
    bin.install "mycli"
  end

  test do
    system "#{bin}/mycli", "--version"
  end
end
EOF

# Tapの公開
git add .
git commit -m "Add mycli formula"
git remote add origin https://github.com/username/homebrew-tap.git
git push -u origin main
```

### インストール方法

```bash
# Tapの追加
brew tap username/tap

# インストール
brew install mycli

# 更新
brew upgrade mycli
```

## go installによる配布

Goユーザー向けには、`go install`コマンドでの配布も有効です。

### 前提条件

- リポジトリにgo.modファイルが存在する
- main.goがモジュールルートまたはcmd/配下に存在する

### インストールコマンド

```bash
# 最新版をインストール
go install github.com/username/my-cli-tool/cmd/mycli@latest

# 特定バージョンをインストール
go install github.com/username/my-cli-tool/cmd/mycli@v1.0.0

# 特定コミットをインストール
go install github.com/username/my-cli-tool/cmd/mycli@abc123
```

README.mdに記載する例:

```markdown
## Installation

### Using Go

```bash
go install github.com/username/my-cli-tool/cmd/mycli@latest
```

### Using Homebrew (macOS)

```bash
brew install username/tap/mycli
```

### Binary Release

Download the latest release from [GitHub Releases](https://github.com/username/my-cli-tool/releases).
```

## Go CLI配布チェックリスト

効果的な配布のために、以下を確認しましょう。

- [ ] クロスコンパイルで主要プラットフォームをサポートしている
- [ ] ldflagsでバイナリサイズを最適化している
- [ ] バージョン情報をビルド時に埋め込んでいる
- [ ] Makefileでビルドプロセスを自動化している
- [ ] GoReleaserで自動リリースを設定している
- [ ] GitHub Actionsでリリースを自動化している
- [ ] Homebrew Formulaを提供している
- [ ] チェックサムファイルを生成している
- [ ] READMEにインストール方法を明記している
- [ ] CHANGELOGを管理している

## まとめ

Goのクロスコンパイル機能と単一バイナリという特性により、CLIツールの配布は非常に簡単です。GoReleaserとGitHub Actionsを組み合わせることで、タグをプッシュするだけで複数プラットフォーム向けのバイナリを自動的にビルド・公開できます。Homebrewでの配布により、macOSユーザーへの提供も容易になります。

## 参考文献

- [Go Build Constraints](https://pkg.go.dev/cmd/go#hdr-Build_constraints)
- [GoReleaser公式ドキュメント](https://goreleaser.com/)
- [Homebrew Formula Cookbook](https://docs.brew.sh/Formula-Cookbook)
- [GitHub Actions - Go](https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-go)
- [UPX - Ultimate Packer for eXecutables](https://upx.github.io/)
