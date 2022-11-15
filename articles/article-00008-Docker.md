---
title: 'Dockerの学習'
excerpt: 'Dockerの基礎用語理解をし、コマンド操作からでローカル環境での実行・本番環境へのデプロイまでの方法を学習する。'
coverImage: '/assets/blog/docker-photo/Docker.png'
date: '2022-10-23'
ogImage:
  url: '/assets/blog/docker-photo/Docker.png'
tags:
  - 'Docker'
  - 'Linux'
  - 'Ubuntu'
---

## Dockerの学習

Dockerの学習をするにあたって、学習し始めたばかりの段階においては苦戦したため、Dockerの用語理解が必要だと思いました。  
そこで、今回の学習では「Dockerにおける基礎知識」の学習をしていきます。

### Dockerとは？

Dockerとは、コンテナ仮想化を用いてアプリケーション開発・配置・実行するためのオープンプラットフォームです。

メリットとしては、  
①コンテナの起動・停止を軽量で高速に行うことができる。  
②OS（Mac,Windows）によって生じる環境構築時に起こりやすいエラーがあるが、コンテナをコピーすることで全く同じ環境で実行し環境構築の手間を省くことができる。  
