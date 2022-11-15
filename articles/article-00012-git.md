---
title: 'GitHubに接続するhttpsとsshの方式について'
excerpt: 'GitHubのリポートリポジトリと接続する際の方式についての調査を行います。'
coverImage: '/assets/blog/github/github.png'
date: '2022-11-04'
ogImage:
  url: '/assets/blog/github/github.png'
tags:
  - 'Git'
  - 'GitHub'
  - 'https'
  - 'ssh'
---

## GitHubに接続するhttpsとsshの方式について

GitHub のリモートリポジトリへ接続は、今や Git が開発時に必須ですので把握しておきたい所になると思います。  
今回は、https 接続と ssh 接続それぞれの特徴を記しておきます。

### https について

GitHub が推奨しているのが https 接続です。  
リポジトリの https のデフォルトの URL 形式を用いて、主に Web 上で通信が行われる。

[解説サイト様 ①「【解説】【GitHub】HTTPS と SSH の違い」](https://zenn.dev/nameless_sn/articles/the_differences_between_https_and_ssh)

### SSH について

はじめに GitHub に ssh 接続する際は、公開鍵・秘密鍵の生成します。  
そして GitHub 上に公開鍵を設定することで接続ができるようにます。

特徴  
・アカウント情報だけでリポジトリに書き込めるので、どこからでも簡単にアクセスできる。  
・HTTPS はすべてのファイアウォールで開かれている点が良い。

[解説サイト様 ①「GitHub に SSH 接続する方法！(キーの作成から push まで解説)」](https://codelikes.com/github-ssh-connection/)

### 考察

「セキュリティ、速度、設定が簡単か、参考記事が多いか」など、私はどちらも設定してみたが HTTPS の方が楽であると感じました。  
HTTPS であれば 作業ディレクトリ>.git>config の config ファイルにリポジトリの URL を設定するだけのため非常に簡単だと思います。  
