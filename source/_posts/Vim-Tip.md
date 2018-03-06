---
layout: post 
title: Vim Tip
subtitle: 记录日常中使用的 Vim 命令，经常更新
author: 帕帕
date: 2017-09-05 15:30:56 +0800
categories: tip
tag: tip
---

## 1、替换第n1行到第n2行的内容

```Vim
:n1,n2/origin/replace/g
```

## 2、替换整个文件的内容

```Vim
:%s/origin/replace/g
```

## 3、移动n1-n2行(包括n1,n2)到n3行之下

```Vim
n1,n2 m n3     
```

## 4、复制n1-n2行(包括n1,n2)到n3行之下

```Vim
:n1,n2 co n3
```

## 5、删除文件的空行

```Vim
:g/^$/d
```

## 6、在文本中插入一个1到100的序列（来自池老师[《说，谁才是最帅的编程工具？》](http://mp.weixin.qq.com/s?__biz=MjM5ODQ2MDIyMA==&mid=2650712546&idx=1&sn=c4db99547b75d6001b3cfaa6cbc0e715&scene=1&srcid=0805j7ny3Ua1WufWDEpnhwOG#rd)）

```Vim
:r!seq 100
```

## 7、在当前的每一行文字前面增加“序号. ”（来自池老师[《说，谁才是最帅的编程工具？》](http://mp.weixin.qq.com/s?__biz=MjM5ODQ2MDIyMA==&mid=2650712546&idx=1&sn=c4db99547b75d6001b3cfaa6cbc0e715&scene=1&srcid=0805j7ny3Ua1WufWDEpnhwOG#rd)）

```Vim
:let i=1 | g /^/ s//\=i.". "/ | let i+=1
```

## 8、当前目录下（包括子文件夹）所有后缀为 java 的文件中的 apache 替换成 eclipse，那么在当前目录下依次执行如下命令：（来自池老师[《说，谁才是最帅的编程工具？》](http://mp.weixin.qq.com/s?__biz=MjM5ODQ2MDIyMA==&mid=2650712546&idx=1&sn=c4db99547b75d6001b3cfaa6cbc0e715&scene=1&srcid=0805j7ny3Ua1WufWDEpnhwOG#rd)）

```Vim
vim
:n **/*.java
:argdo %s/apache/eclipse/ge | update 
```

