---
layout: post 
title: Raspberry PI 基本使用
subtitle: 记录自己在 RaspberryPI 比较频繁使用的命令
author: 帕帕
date: 2018-03-05 22:04:56 +0800
categories: tip
tag: tip
---


## 1. 保证 Raspberry PI 能够在外网使用

```
// 在你的 Raspberry PI 上使用 autossh 来实现不掉线的反向代理：
autossh -M 5678 -fNR 2018:localhost:22 root@54.219.12.213
```

```
// 在你的外网服务器（比如上面例子中的：54.219.12.213）上可以使用 ssh 登录你的 Raspberry PI
ssh -p 2018 pi@127.0.0.1
```

