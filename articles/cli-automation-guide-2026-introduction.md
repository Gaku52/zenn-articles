---
title: "Node.js/Python/Go完全対応のCLI開発本を書きました"
emoji: "💻"
type: "tech"
topics: ["cli", "nodejs", "python", "go", "自動化"]
published: true
---

## ある日の出来事

金曜の午後、チームのSlackにこんなメッセージが流れてきました。

> 「デプロイ手順、もう一回教えて」
> 「あ、前回と変わってるから。READMEの5番の手順、envファイルの設定追加して」
> 「え、どのenv？本番？ステージング？」

15分後、デプロイは失敗。原因は環境変数の設定ミスでした。

**「これ、CLIツールにしたら1コマンドで終わるのに」** と思ったことは何度もある。でも、いざ作ろうとすると壁にぶつかります。

- Node.js、Python、Go...どの言語がいいのか？
- Commander? Click? Cobra? フレームワークが多すぎて選べない
- 引数パースやエラーハンドリング、どう設計すればいいのか？
- 作ったツール、チームにどうやって配布するのか？

各言語のドキュメントを個別に読み漁って、それぞれの流儀を覚えて...というのは現実的ではありません。

そこで、**Node.js・Python・Goの3言語でCLI開発を体系的に学べる本**を書きました。

この記事では、本の技術的な内容の一部として**言語選定・フレームワーク選び・設計パターン・よくある失敗**を解説します。記事だけでも実務に役立つはずです。

---

## 🔍 どの言語でCLIを作るべきか？

CLI開発で最初にぶつかる壁が「言語選び」です。用途によって最適な言語が変わるので、判断基準を整理します。

| 観点 | Node.js | Python | Go |
|------|---------|--------|----|
| **起動速度** | やや遅い（V8起動） | やや遅い（インタプリタ） | 高速（ネイティブバイナリ） |
| **配布しやすさ** | npm publish（要Node.js） | PyPI publish（要Python） | シングルバイナリ（依存なし） |
| **エコシステム** | npm（フロントエンド連携◎） | pip（データ処理・AI連携◎） | go modules（インフラ連携◎） |
| **学習コスト** | 低い（JSを知っていれば） | 低い（直感的な文法） | 中程度（型システムの理解） |
| **型安全性** | TypeScriptで◎ | 型ヒントで○ | 標準で◎ |
| **向いている用途** | フロントエンド関連ツール | データ処理・スクリプト | インフラ・DevOpsツール |

### 判断のポイント

**Node.jsを選ぶべきとき：**
- チームがJavaScript/TypeScriptメインで開発している
- フロントエンドのビルドツールやLinter系のツールを作りたい
- npm経由で手軽に配布したい
- `npx`で即実行可能な形で配布したい

**Pythonを選ぶべきとき：**
- データ処理やAPI連携が主な用途
- 既存のPythonライブラリ（pandas、requestsなど）を活用したい
- プロトタイプを素早く作りたい
- AI/ML関連のツールを作りたい

**Goを選ぶべきとき：**
- ユーザーの環境にランタイムをインストールさせたくない
- kubectl、terraform、gh のようなDevOpsツールを目指す
- 高速な起動と実行速度が求められる
- クロスコンパイルして複数OS向けにバイナリを配布したい

### 実務での選び方

理論的な比較も大事ですが、実務では**「チームの主要言語に合わせる」**のが最もバランスの良い選択です。保守する人が読める言語でないと、作った本人が異動した瞬間にメンテ不能になります。

ただし例外があります。**配布先が社外やOSS**の場合は、ランタイム不要のGoが大きなアドバンテージを持ちます。「まずGoをインストールしてください」と言わなくて済むのは、ユーザー体験として圧倒的に優れています。

---

## 🎯 フレームワーク選定ガイド

言語が決まったら、次はフレームワーク選びです。ここも迷いやすいポイントなので、各言語の主要フレームワークの使い分けと実装パターンを解説します。

### Node.js: Commander.js + Inquirer.js

この2つは**競合ではなく、組み合わせて使うもの**です。

- **Commander.js** — コマンドとオプションの定義（`mycli build --watch`のようなインターフェース）
- **Inquirer.js** — 対話型プロンプト（ユーザーに選択肢を出して入力を受け取る）

```javascript
// Commander.jsでコマンド定義 + Inquirer.jsで対話型プロンプト
import { Command } from 'commander';
import inquirer from 'inquirer';

const program = new Command();

program
  .command('init')
  .description('プロジェクトを初期化')
  .option('-n, --name <name>', 'プロジェクト名')
  .action(async (options) => {
    // オプションが未指定なら対話で補完する
    const answers = await inquirer.prompt([
      {
        type: 'input',
        name: 'name',
        message: 'プロジェクト名？',
        when: !options.name,  // --nameが未指定のときだけ質問
        validate: (input) => input.length > 0 || '名前は必須です'
      },
      {
        type: 'list',
        name: 'template',
        message: 'テンプレートは？',
        choices: ['react', 'vue', 'svelte']
      },
      {
        type: 'confirm',
        name: 'typescript',
        message: 'TypeScriptを使いますか？',
        default: true
      }
    ]);

    const projectName = options.name || answers.name;
    console.log(`${projectName} を ${answers.template} で作成中...`);
  });

program.parse();
```

ポイントは**`when`プロパティ**です。コマンドラインオプションで指定済みなら質問をスキップし、未指定なら対話で補完する。これにより、**スクリプトからの自動実行（全オプション指定）と、人間の対話的な利用の両方に対応**できます。

### Python: Click vs Typer

こちらは**同じ目的のフレームワーク**で、どちらか一方を選びます。

| 比較 | Click | Typer |
|------|-------|-------|
| 設計思想 | デコレータベース | 型ヒントベース |
| Python要件 | 3.7+ | 3.7+（型ヒント活用は3.10+推奨） |
| 内部実装 | — | Click上に構築 |
| 自動補完 | プラグイン必要 | 組み込み |
| エラー表示 | シンプル | Rich統合でカラフル |
| 向いている場面 | 大規模・複雑なCLI | モダンでシンプルなCLI |

**Clickの実装例：**

```python
import click

@click.group()
def cli():
    """プロジェクト管理ツール"""
    pass

@cli.command()
@click.argument('name')
@click.option('--template', type=click.Choice(['react', 'vue', 'svelte']),
              default='react', help='テンプレート')
@click.option('--typescript/--no-typescript', default=True,
              help='TypeScriptを使用するか')
def init(name, template, typescript):
    """新しいプロジェクトを作成"""
    lang = 'TypeScript' if typescript else 'JavaScript'
    click.echo(f'{name} を {template} ({lang}) で作成中...')

    # プログレスバー付きの処理
    with click.progressbar(range(100), label='セットアップ中') as bar:
        for item in bar:
            pass  # 実際のセットアップ処理

    click.secho('✓ 作成完了！', fg='green', bold=True)

if __name__ == '__main__':
    cli()
```

**Typerの実装例：**

```python
import typer
from typing import Optional
from enum import Enum

app = typer.Typer(help="プロジェクト管理ツール")

class Template(str, Enum):
    react = "react"
    vue = "vue"
    svelte = "svelte"

@app.command()
def init(
    name: str = typer.Argument(..., help="プロジェクト名"),
    template: Template = typer.Option(Template.react, help="テンプレート"),
    typescript: bool = typer.Option(True, help="TypeScriptを使用するか"),
):
    """新しいプロジェクトを作成"""
    lang = 'TypeScript' if typescript else 'JavaScript'
    typer.echo(f'{name} を {template.value} ({lang}) で作成中...')
    typer.secho('✓ 作成完了！', fg=typer.colors.GREEN, bold=True)

if __name__ == '__main__':
    app()
```

Typerは**関数シグネチャがそのままCLIインターフェースになる**のが最大の特徴です。型ヒントを書くだけで引数の型チェックやヘルプメッセージが自動生成されます。

**迷ったらTyper**がおすすめです。内部でClickを使っているので、Clickの機能も必要に応じて呼び出せます。新規プロジェクトならTyperでシンプルに始めて、複雑になったらClickの機能を組み合わせるのが現実的な戦略です。

### Go: Cobraがデファクト

Goでは**Cobra**がデファクトスタンダードです。kubectl、gh（GitHub CLI）、hugo、terraformなど、有名ツールの多くがCobraで作られています。

Cobraを使いこなすポイントは**Viper**（設定管理ライブラリ）との統合です。コマンドラインフラグ、環境変数、設定ファイルの優先順位を統一的に管理できます。

```go
package cmd

import (
    "fmt"
    "github.com/spf13/cobra"
    "github.com/spf13/viper"
)

var (
    cfgFile  string
    template string
    useTS    bool
)

// rootCmd はルートコマンド
var rootCmd = &cobra.Command{
    Use:   "mytool",
    Short: "プロジェクト管理ツール",
    Long:  "プロジェクトの作成、ビルド、デプロイを管理するCLIツール",
}

// initCmd は init サブコマンド
var initCmd = &cobra.Command{
    Use:   "init [name]",
    Short: "新しいプロジェクトを作成",
    Args:  cobra.ExactArgs(1),
    RunE: func(cmd *cobra.Command, args []string) error {
        name := args[0]
        lang := "JavaScript"
        if useTS {
            lang = "TypeScript"
        }
        fmt.Printf("%s を %s (%s) で作成中...\n", name, template, lang)
        return nil
    },
}

func init() {
    cobra.OnInitialize(initConfig)

    // グローバルフラグ
    rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "",
        "設定ファイルのパス（デフォルト: $HOME/.mytool.yaml）")

    // init コマンドのフラグ
    initCmd.Flags().StringVarP(&template, "template", "t", "react",
        "テンプレート（react, vue, svelte）")
    initCmd.Flags().BoolVar(&useTS, "typescript", true,
        "TypeScriptを使用するか")

    // Viperとバインド（環境変数 MYTOOL_TEMPLATE でも設定可能に）
    viper.BindPFlag("template", initCmd.Flags().Lookup("template"))

    rootCmd.AddCommand(initCmd)
}
```

Cobraの特徴は**サブコマンドの構造化**が得意なことです。`mytool init`、`mytool build`、`mytool deploy`のように、Gitライクなサブコマンド構造を自然に定義できます。

---

## 🏗️ CLIツールのアーキテクチャ

フレームワークの使い方を覚えたら、次に考えるべきは**ツール全体の構造**です。小さなスクリプトならフラットに書けばいいですが、チームで使うツールには設計が必要です。

### 推奨するディレクトリ構造（Node.jsの例）

```
my-cli/
├── src/
│   ├── commands/        # 各コマンドの実装
│   │   ├── init.ts
│   │   ├── build.ts
│   │   └── deploy.ts
│   ├── lib/             # 共通ロジック
│   │   ├── config.ts    # 設定管理
│   │   ├── logger.ts    # ログ出力
│   │   └── errors.ts    # エラー定義
│   └── index.ts         # エントリーポイント
├── tests/
│   ├── commands/
│   └── lib/
├── package.json
└── tsconfig.json
```

ポイントは**コマンドの実装とビジネスロジックを分離**することです。`commands/`はCLIインターフェース（引数パース、出力フォーマット）だけを担当し、実際のロジックは`lib/`に置く。これにより：

- コマンドのテストとロジックのテストを分離できる
- 将来的にAPIやWeb UIからも同じロジックを呼べる
- 新しいコマンドの追加が容易

### 設定の優先順位

CLIツールでは、設定を複数のソースから読み込むことがあります。**優先順位を明確に定義**しておかないと、「なぜこの値になるのか」がデバッグ困難になります。

```
1. コマンドラインフラグ（最高優先）    --port 8080
2. 環境変数                          MYTOOL_PORT=8080
3. プロジェクトの設定ファイル          .mytoolrc.yaml
4. ユーザーの設定ファイル              ~/.config/mytool/config.yaml
5. デフォルト値（最低優先）            port: 3000
```

この優先順位はCobraのViper統合が自動で処理してくれますが、Node.jsやPythonでは自分で実装する必要があります。本書Part 1でこの実装パターンを3言語で解説しています。

---

## ⚠️ CLI設計でよくある5つの失敗

フレームワークを使いこなせても、設計を間違えるとユーザーに使ってもらえません。よく見かける失敗パターンと、その解決策を紹介します。

### 失敗1: ヘルプメッセージが不親切

```bash
# ❌ ダメな例 — 使い方が全く分からない
$ mytool --help
Usage: mytool [options]

# ✅ 良い例 — コマンド一覧、説明、具体例がある
$ mytool --help
Usage: mytool <command> [options]

プロジェクトの作成、ビルド、デプロイを管理するCLIツール

Commands:
  init <name>   新しいプロジェクトを作成
  build         プロジェクトをビルド
  deploy        本番環境にデプロイ

Options:
  -v, --verbose  詳細なログを出力
  -h, --help     ヘルプを表示
  --version      バージョンを表示

Examples:
  $ mytool init my-project --template react
  $ mytool build --watch
  $ mytool deploy --env production
```

ヘルプメッセージは**CLIのUI**です。`Examples`セクションがあるだけで使いやすさが大きく変わります。ユーザーは「まずコピペして動かしたい」ので、動く具体例を見せるのが最も効果的です。

### 失敗2: エラーメッセージが何も教えてくれない

```bash
# ❌ ダメな例 — 何をすればいいのか分からない
Error: invalid argument

# ✅ 良い例 — 原因と解決策を提示
Error: --port must be a number between 1 and 65535 (got "abc")

  Hint: Try 'mytool serve --port 3000'
  Docs: https://mytool.dev/docs/serve
```

**何が間違いで、どうすれば直るか**を1つのメッセージで伝えるのが良いエラー設計です。余裕があれば`Hint:`や`Docs:`も付けると、ユーザーが自力で解決しやすくなります。

### 失敗3: 終了コードを無視している

CLIツールはシェルスクリプトやCI/CDパイプラインから呼ばれます。終了コードを正しく返さないと、**異常終了なのにパイプラインが続行してしまう**危険があります。

```javascript
// Node.js — 終了コードの使い分け
process.exit(0); // 成功
process.exit(1); // 一般的なエラー
process.exit(2); // 引数エラー（慣例）
```

```python
# Python
import sys
sys.exit(0)  # 成功
sys.exit(1)  # エラー
```

```go
// Go
os.Exit(0) // 成功
os.Exit(1) // エラー
```

特にCI/CDパイプラインで使われるツールでは、終了コードの正確さが**自動化の信頼性**に直結します。

### 失敗4: 出力がパイプに対応していない

CLIツールの出力は、人間が読むだけでなく`grep`や`jq`でパイプ処理されることがあります。

```bash
# ❌ ダメな例 — 装飾が邪魔でパイプ処理できない
$ mytool list
🎉 Found 3 projects!
  ★ my-app (react)
  ★ api-server (express)
  ★ cli-tool (commander)

# ✅ 良い例 — TTY判定で出力を切り替える
$ mytool list          # 人間向け（カラー・アイコン付き）
  ✓ my-app (react)
  ✓ api-server (express)
  ✓ cli-tool (commander)

$ mytool list | grep react  # パイプ時（プレーンテキスト）
my-app react
```

```javascript
// Node.js — TTY判定で出力形式を切り替える
const isTTY = process.stdout.isTTY;

function formatOutput(projects) {
  if (isTTY) {
    // 人間向け: 色やアイコン付き
    return projects.map(p => `  ✓ ${p.name} (${p.template})`).join('\n');
  } else {
    // パイプ向け: プレーンテキスト（タブ区切り）
    return projects.map(p => `${p.name}\t${p.template}`).join('\n');
  }
}
```

UNIXのCLI文化では、**「1つのことをうまくやり、他のツールと組み合わせられる」**のが良いツールの条件です。

### 失敗5: 破壊的操作に確認がない

```bash
# ❌ ダメな例 — 即座に全削除される
$ mytool clean
Deleted 347 files.

# ✅ 良い例 — 確認プロンプトとドライランを用意
$ mytool clean
This will delete 347 files in ./dist and ./cache.
Are you sure? (y/N): n
Aborted.

$ mytool clean --dry-run
Would delete 347 files:
  ./dist/bundle.js
  ./dist/index.html
  ./cache/...
```

`--dry-run`フラグと確認プロンプトは、ユーザーの信頼を得るために必須です。特に`rm`や`delete`を含むコマンドでは、**デフォルトが安全側**になるよう設計します。

---

## 🧪 CLIツールのテスト戦略

CLIツールのテストは通常のアプリケーションとは少し異なります。入力が「コマンドライン引数」で、出力が「標準出力」「終了コード」「ファイルシステムの変化」だからです。

### テストすべき3つの層

**1. ユニットテスト：ビジネスロジック**

コマンドの実装から切り離された純粋なロジックをテストします。

```javascript
// lib/config.ts のテスト
describe('mergeConfig', () => {
  it('コマンドラインフラグが環境変数より優先される', () => {
    const result = mergeConfig(
      { port: 8080 },           // フラグ
      { port: 3000, host: '0.0.0.0' }  // 環境変数
    );
    expect(result.port).toBe(8080);
    expect(result.host).toBe('0.0.0.0');
  });
});
```

**2. インテグレーションテスト：コマンド実行**

実際にコマンドをプロセスとして実行し、出力と終了コードを検証します。

```javascript
import { execSync } from 'child_process';

describe('mytool init', () => {
  it('プロジェクトが正しく作成される', () => {
    const output = execSync('node ./bin/mytool init test-project --template react')
      .toString();
    expect(output).toContain('test-project を react で作成中');
  });

  it('引数なしでエラー終了する', () => {
    expect(() => {
      execSync('node ./bin/mytool init', { stdio: 'pipe' });
    }).toThrow();
  });
});
```

**3. スナップショットテスト：ヘルプメッセージ**

ヘルプメッセージが意図せず変更されていないことを検証します。

```javascript
it('ヘルプメッセージが変更されていない', () => {
  const output = execSync('node ./bin/mytool --help').toString();
  expect(output).toMatchSnapshot();
});
```

本書のPart 2-4では、各言語のテストフレームワーク（Jest、pytest、go test）を使ったCLI固有のテストパターンを詳しく解説しています。

---

## 📦 3言語の配布方法を比較する

CLIツールは「作って終わり」ではなく、ユーザーの手元に届けてこそ意味があります。3言語で配布の仕組みが大きく異なるので、比較します。

### Node.js: npm publish

```bash
# ユーザーのインストール方法
npm install -g mytool    # グローバルインストール
npx mytool init my-app   # インストール不要で即実行
```

**メリット：** `npx`で即実行できるので、ユーザーのハードルが低い
**デメリット：** Node.jsランタイムが必要

### Python: PyPI publish

```bash
# ユーザーのインストール方法
pip install mytool       # pip経由
pipx install mytool      # 隔離環境にインストール（推奨）
```

**メリット：** `pipx`を使えば仮想環境が自動管理される
**デメリット：** Pythonランタイムが必要、バージョン問題が起きやすい

### Go: シングルバイナリ配布

```bash
# ユーザーのインストール方法
go install github.com/user/mytool@latest   # go install
brew install mytool                         # Homebrew
# または GitHub Releases からバイナリをダウンロード
```

**メリット：** 依存なしのシングルバイナリ、クロスコンパイルで全OS対応
**デメリット：** バイナリサイズが大きくなりがち

### 配布で見落としがちなポイント

どの言語でも共通して重要なのが**バージョニング**です。CLIツールは破壊的変更がユーザーのスクリプトを壊す可能性があるため、**セマンティックバージョニング**を厳守し、`--version`フラグで常にバージョンを確認できるようにしておきましょう。

本書のPart 2-4では、各言語でのpublish手順、Homebrew formulae作成、GitHub Actionsでの自動リリースまで、配布の全工程を解説しています。

---

## 📖 本書で、さらに深く学べること

ここまでの内容は本のエッセンスの一部です。本書（全25章）では、これらの基礎の先にある**実装・テスト・配布・CI/CD統合**まで一貫して解説しています。

**Part 1: CLI設計・アーキテクチャ（4章）**
UNIXフィロソフィー、12 Factor CLI Apps、プラグインアーキテクチャ、設定ファイル戦略（YAML/TOML/JSON）の使い分けと優先順位設計

**Part 2: Node.js CLI開発（6章）**
Commander.js + Inquirer.jsの実践的な組み合わせ、Jestでのテスト戦略、npm publishから`npx`で即実行可能なパッケージ配布まで

**Part 3: Python CLI開発（6章）**
Click/Typerの実装パターン、pytestでのテスト、PyPI publish、pipx対応、Rich/tqdmを使ったリッチな出力

**Part 4: Go CLI開発（4章）**
Cobra + Viper統合、`go install`でのインストール、Homebrew formulae作成、GitHub Releasesでのクロスコンパイル配布

**Part 5: スクリプト自動化（5章）**
Shell/Node.js/Pythonスクリプトの実践パターン、GitHub Actions/GitLab CIとの統合、冪等性のあるデプロイスクリプト設計

記事で紹介した「どの言語を選ぶか」「どのフレームワークを使うか」「どう設計するか」が分かったら、**次は実際に手を動かして作る段階**です。本書はそこをカバーしています。

---

## 💡 こんな人におすすめ

### ✅ こんな方に最適

- **複数の言語でCLIツールを作りたい開発者** — 同じ機能を3言語で比較しながら学べる
- **チーム内の作業を自動化したいエンジニア** — 設計原則から配布方法まで一気通貫
- **OSSのCLIツールを公開したい方** — npm/PyPI/Homebrew配布の手順を完全解説
- **フレームワーク選定に迷っている方** — 各フレームワークの特徴と使い分けを詳しく比較
- **「なんとなく動くスクリプト」を卒業したい方** — 設計・テスト・配布まで体系的に

### ⚠️ こんな方には向いていません

- プログラミング初心者（各言語の基礎を学んでから）
- GUIアプリケーション開発を学びたい方
- 1言語だけで十分な方（単言語の専門書の方が深い場合があります）

---

## 📊 こんな場面で使える

### 「社内のデプロイ作業、毎回手動でやってるんだけど...」

開発チームでありがちな悩みです。本書のPart 1で設計原則を学び、チームの主要言語（Part 2-4）で実装、Part 5でCI/CDに組み込めば、**属人化していた作業をツールとして標準化**できます。

### 「自作ツールをOSSとして公開したいけど、配布の仕方が分からない」

CLIツールは作って終わりではなく、**使ってもらえる形で配布**してこそ価値があります。npm publish、PyPI publish、Homebrew formulae作成、GitHub Releasesの自動化まで、各言語の配布フローを実際の手順で解説しています。

### 「フロントエンドはNode.js、バックエンドはPython、インフラはGo — 全部CLIツール作りたい」

マルチ言語環境では、それぞれの環境に最適なCLIツールが欲しくなります。この本は**同じ設計思想で3言語を横断的に学べる**ので、言語が変わっても一貫した品質のツールが作れます。

---

## 🙋 よくある質問

**Q: プログラミング初心者でも読めますか？**
A: 各言語の基礎（関数、クラス、パッケージ管理）を理解している中級者向けです。初心者の方は各言語の入門書を先に読むことをおすすめします。

**Q: 1言語だけ学びたい場合は？**
A: 1言語のみなら、その言語専門の本の方が詳しいかもしれません。ただし、複数言語を比較することで「なぜその言語ではこう書くのか」がより深く理解できます。それがこの本の強みです。

**Q: サンプルコードはありますか？**
A: 全てのチャプターに実装例が含まれています。3言語を並べて比較しているので、言語間の違いが直感的に分かります。

**Q: 最新のフレームワークバージョンに対応していますか？**
A: 2026年時点の各フレームワークの安定バージョンに基づいて執筆しています。

**Q: 実務で使えますか？**
A: はい。実務での自動化、社内ツール開発、OSS公開まで全てカバーしています。設計パターンからCI/CD統合まで、プロダクションで必要な知識を網羅しています。

---

## 📘 読んでみる

**CLI開発完全ガイド 2026 - Commander.js/Click/Cobraで作るコマンドラインツール**

全25章。設計原則から実装・テスト・配布・CI/CD統合まで、3言語でCLI開発を体系的に学べる1冊です。

👉 **[Zennで読む](https://zenn.dev/gaku1234/books/cli-automation-complete-guide)**（500円）

---

## 📚 関連リソース

**[iOS開発応用編 2026](https://zenn.dev/gaku1234/books/ios-advanced-guide-2026)**
Xcodeプロジェクト設定、セキュリティ、データ永続化

**[Claude Codeガイド: モダンフロントエンド開発 2026](https://zenn.dev/gaku1234/books/claude-code-frontend-guide-2026)**
React、Next.js、TypeScriptの実践ガイド

**[claude-code-skills（GitHub）](https://github.com/Gaku52/claude-code-skills)**
25個の技術Skills、無料公開

---

質問・フィードバック大歓迎です！Zennのコメント欄でお待ちしています。
