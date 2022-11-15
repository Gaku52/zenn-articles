---
title: 'WSL2でNext.jsアプリの変更時にHOT Reloadが遅いとき'
excerpt: 'WSL2でWEBアプリ開発時において、HOTリロードが遅いときの状況と原因について記します。'
coverImage: '/assets/blog/Ubuntu/Ubuntu.png'
date: '2022-11-02'
ogImage:
  url: '/assets/blog/Ubuntu/Ubuntu.png'
tags:
  - 'Ubuntu'
  - 'Linux'
  - 'WSL2'
---

## WSL2でNext.jsアプリの変更時にHOT Reloadが遅いとき

WSL2 ＋ Docker の実行環境で Next.js プロジェクトを実行環境としたとき、変更時にリアルタイム更新(HOT Reload)が遅い現象が起こりました。

実際どれくらい遅いというと、ローカル開発と比べて 3~5 秒遅いと思われます。(体感)

せっかく Docker の環境で開発したいと思っても、これではローカルの方がいいかな...となってしまいます。

そこで、今回は状況分析から解決策実施まで行いました。

### 状況と原因

状況として WLS2 の mnt/c から Windows 上のディレクトリへアクセスしていたのですが、それが原因のようです。

データ更新の際、毎回 Windows へアクセスしているため時間がかかっているとのこと。

### 解決方法

コマンド操作でディレクトリを、WSL2 の home ディレクトリ上へコピーする。  
Ubuntu 側で直接 WEB アプリ開発環境を構築することもでき、HOT Reload 速度も向上します。

### 参考サイト様

[解説サイト様 ①「Docker で Next.js の開発環境を作ったがローカルで開発することにした」](https://blog.kapiecii.com/posts/2022/07/24/docker-and-nextjs/)  
[解説サイト様 ②「(続き)Windows+WSL2 で React の開発環境を作った」](https://onl.la/Dwj9qFg)
