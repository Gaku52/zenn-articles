---
title: 'Ubuntuのデフォルトユーザーを変更する'
excerpt: 'WSLでデフォルトユーザーを変更したいとき、スムーズに設定できなかったため記録を残しておきます。'
coverImage: '/assets/blog/Ubuntu/Ubuntu.png'
date: '2022-11-01'
ogImage:
  url: '/assets/blog/Ubuntu/Ubuntu.png'
tags:
  - 'Ubuntu'
  - 'Linux'
  - 'WSL2'
---

## Ubuntuのデフォルトユーザーを変更する

WindowsでWSL2を導入した(Ubuntu22.04 LTS)とき、初期設定ではWindowsターミナルからUbuntuを毎回起動する度に
rootユーザーでされてしまう。rootは管理者権限のあるユーザーであるためセキュリティ上あまり使いたくない。  

そこで新規ユーザーを作成した後に、デフォルトユーザー(ターミナル起動時のユーザー)を変更するのが一般的である。  

今回は、解決方法の1つである「Powershellからのコマンド入力」を方法を記載しておきます。  

### 解決方法

全体的な流れ  

1.Windows PowerShell（Ubuntu以外のシェル）を実行する。  
2.以下のコマンドラインをバージョンに合わせて入力する。(※注意点：ubuntu2204はバージョンによって変わる。)  
3.ターミナルでUbuntuを起動すると、設定した＜ユーザー名＞での起動が確認される。

Windows PowerShellを実行後、以下コマンド

```tsx
ubuntu2204 config --default-user <ユーザー名>
```

### エラー調査

注意点として私が誤ってしまった点として、Ubuntuを開いて上記コマンドを入力しても  
以下のように表示されます。  

下記エラーの意味は「指定されたコマンドが見つからない。」という意味です。  
今回は、「Windowsターミナル」でUbuntuを開いたときのデフォルトユーザー設定ですので  
Windows側のシェルからコマンド入力する必要がある。ということでしょう。  

```tsx
ubuntu2204: command not found
```

Windows PowerShell（もしくは他のシェル）でコマンド入力をすると設定が完了されます。  

### 参考サイト様

[解説サイト様①「WSL2のデフォルトログインユーザーをrootから変更したい」](https://creepfablic.site/2022/06/28/wsl2-default-login-user-change/)  
[解説サイト様②「WSL2のUbuntuのデフォルトユーザーを変更する」](https://onl.la/WHcR1yd)  
[解説サイト様③「WSL で デフォルトユーザ を変更する方法」](https://devlights.hatenablog.com/entry/2021/05/29/070000)  
