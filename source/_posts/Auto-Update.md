---
layout: post 
title: Auto Update
subtitle: ""
author: 帕帕
date: 2017-09-05 15:30:56 +0800
categories: tip
tag: tip
---

```sh
#!/bin/sh
# 此脚本是用来自动更新 Mac 上的一些软件或配置

echo "Begin gem update ..."
gem update
echo "End gem update!"

#echo "Begin gem install cocoapods ..."
#gem install cocoapods
#echo "End gem install cocoapods!"

echo "\n"
echo "Begin pod setup ..."
pod setup
echo "End pod setup!"

echo "\n"
echo "Begin brew update ..."
brew update
echo "End brew update"
```




