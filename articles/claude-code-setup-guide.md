---
title: "Claude Codeを5分で始める方法"
emoji: "🚀"
type: "tech"
topics: ["claude", "ai", "cli", "devtools"]
published: false
---

## はじめに

Claude Codeは、ターミナル上で動くClaudeのCLIツール。コードを書きながらClaudeに質問したり、バグ修正を依頼したり、コードレビューを頼んだりできる。

導入が簡単だったので手順をメモしておく。Homebrewなら1コマンドで終わる。

## 準備

macOS 10.15以降、ネット環境、Claude ProかMaxプラン。これだけ。メモリは4GBあれば動く。

## インストール

Homebrewがあるならこれだけ。

```bash
brew install --cask claude-code
```

Homebrewを使っていない場合は公式スクリプトで。

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

どちらも一瞬で終わる。

## 起動

プロジェクトに移動して`claude`と打つだけ。

```bash
cd your-project
claude
```

初回だけログイン画面が出るので、Claude ProかMaxのアカウントでログイン。これで使える。

## 実際に使ってみる

起動したら、普通に話しかけるだけ。

```
> このコードの問題点を教えて
```

ファイルを指定して質問もできる。

```
> src/index.tsのバグを修正して
```

コードレビューも頼める。

```
> 最後のコミットをレビューして
```

会話しながらコードを直していける。VSCodeやCursorみたいな感覚だけど、ターミナルで完結する。

## 動作確認

不安なら確認してみる。

```bash
claude doctor
```

エラーが出なければOK。

## アップデート

Homebrewなら普通にアップデート。

```bash
brew upgrade claude-code
```

スクリプトでインストールした場合は自動更新が効いているので放っておいて良い。

## よくあるエラー

### ログインできない

`claude doctor`を実行してみる。ネットワークの問題かプランの契約状況を確認。

```bash
claude doctor
```

### コマンドが見つからない

パスが通っていない可能性がある。ターミナルを再起動してみる。

```bash
# ターミナルを再起動
exit
```

それでもダメなら再インストール。

```bash
brew reinstall claude-code
```

## まとめ

Homebrewなら1コマンドで導入完了。プロジェクトで`claude`を叩けばすぐ使える。

ターミナルから離れずにコードレビューやバグ修正ができるのは思ったより便利。特に、エラーメッセージをそのまま貼り付けて「これ直して」と言えるのが良い。

気になる人は試してみてほしい。Claude ProかMaxに入っているなら追加料金なしで使える。

