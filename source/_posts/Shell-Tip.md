---
title: Shell Tip
author: 帕帕
date: 2017-09-05 15:30:56 +0800
categories: 技术
tags: [tip] 
---

> 记录日常中用到的一些 Bash 脚本，经常更新

## Tip 1 : 修改文件里面的内容
早上产品有一个小需求就是把工程中的所有网页的标题修改为黑米流量通，可以使用以下命令来实现

```sh
$ find . -name '*.html' -print0 | xargs -0 sed -i '' -e 's/<title>.*<\/title>/<title>黑米流量通<\/title>/g'
```

* `find`          查找命令，可以用 man find 查看更多的信息
* `.`             代表当前目录
* `-name`         find 命令的参数，表示要查找的文件名
* `-print0`       是一种不换行的输出格式，以 ASCII NUL 字符（也就是\0）作为分隔符。上面的例子可能是 `a.html\0b.html\0c.html`
* `|`             这是一个管道符，表示把前面命令的输出作为后面命令的输入
* `xargs`         是用来构造输入参数，并且循环执行每一个参数
* `-0`            表示让 xargs 使用 ASCII NUL 来分隔参数。上面的例子将被分隔成 `a.html` `b.html` `c.html` 三个参数依次执行
* `sed`           这是一个流编辑器，如果传的是文件名会把文件内容读入内存，如果只是普通字符串就会把字符串读入内存
* `-i`            表示要把原来的文件内容做一次备份，后面的 `''` 是表示要备份的文件名字，如果没有文件名字就表示不需要备份
* `-e`            表示后面的字符串是一个命令，需要被执行
* `s/old/new/g`   这个是用来替换字符串的命令

## Tip 2 : 查找文件的内容
把匹配的文件内容的相关文件列出来

```sh
$ find . -name '*.html' -print0 | xargs -0 grep 'PATTERN'
```

## Tip 3 : 解决 Homebrew 的权限问题
查看 Homebrew 的所有权

```sh
$ ls -al `which brew`
```

把 Homebrew 的用户和分组修改为 root 和 wheel

```sh
$ sudo chown root:wheel `which brew`
```

最后还原 Homebrew 的权限（安全）

```sh
$ sudo chown : `chown brew`
```

## Tip 4 : 利用 Shell 生成生成 ICON

```sh
#!/bin/sh
#此脚本是用来生成 iPhone 和 iPad 所需 icon 的不同尺寸的，最好是准备一张 1024x1024 的 Icon 图片


filename="icon.png"

dirname="icon"

name_array=("Icon-20.png" "Icon-20@2x.png" "Icon-20@3x.png"
"Icon-29.png" "Icon-29@2x.png" "Icon-29@3x.png"
"Icon-40.png" "Icon-40@2x.png" "Icon-40@3x.png"
"Icon-60@2x.png" "Icon-60@3x.png"
"Icon-76.png" "Icon-76@2x.png"
"Icon-83.5@2x.png")
size_array=("20" "40" "80"
"29" "58" "87"
"40" "80" "120"
"120" "180"
"76" "152"
"167")

mkdir $dirname

for ((i=0;i<${#name_array[@]};++i)); do
    m_dir=$dirname/${name_array[i]}
    cp $filename $m_dir
    sips -Z ${size_array[i]} $m_dir
done
```

## Tip5 : 使用 Python 共享当前目录

利用下面的命令可以暂时开启一个端口号为 8000 的 HTTP 服务，其他人只需要在浏览器输入 `http://ip-address:8000` 即可浏览共享目录下的文件

```sh
$ python -m SimpleHTTPServer
```


## Tip6 : 加密和解密文件

* 加密

```sh
$ tar czf - {SRC_DIR} | openssl des3 -salt -k "{KEY}" -out {DIST_PACKAGE}.tar.gz
```

示例：

目录名 `paris_code`，秘钥 `meta#com`，输出包 `paris_code_20161008.tar.gz`

```sh
$ tar czf - paris_code | openssl des3 -salt -k "meta#com" -out paris_code_20161008.tar.gz
```

* 解密

第一步：获取代码压缩文件包

下载地址 `http://XXXX.com/paris_code_20161008.tar.gz`

第二步：解密文件（OS X / Linux only）

在 Terminal 进入压缩文件包同级目录，输入以下命令：

```sh
$ openssl des3 -d -k "meta#com" -salt -in paris_code_20161008.tar.gz | tar xzf -
```

## Tip7: iOS 打包命令

```sh
echo "----------------"
echo "Begin Build!"
PROJECT_NAME="orbit"
BUILD_DATE="$(date +'%Y%m%d')"
BUNDLE_ID="com.meta.paris"
cd ${WORKSPACE}

#/usr/local/bin/npm install

if [ -d "${WORKSPACE}/build" ]; then 
    if ls ${WORKSPACE}/build/**/*.ipa 1> /dev/null 2>&1; then
        rm -rf ${WORKSPACE}/build/**/*.ipa; 
    fi;
    if ls ${WORKSPACE}/build/**/*.xcarchive 1> /dev/null 2>&1; then
        rm -rf ${WORKSPACE}/build/**/*.xcarchive; 
    fi;
else 
    mkdir ${WORKSPACE}/build; 
fi;

echo "计算今天的 Build Version"
if [ -d "${WORKSPACE}/build/${BUILD_DATE}" ]; then 
   #如果不加上面的 if, Jenkins 无法直接执行下面的命令❓
	BUILD_DATE_COUNT=$(ls ${WORKSPACE}/build | grep "^${BUILD_DATE}" -c)
    if [ ${BUILD_DATE_COUNT} -lt 10 ]; then
        BUILD_DATE_COUNT="0${BUILD_DATE_COUNT}"
    fi;
	BUILD_VERSION="${BUILD_DATE}${BUILD_DATE_COUNT}"
else 
  	BUILD_VERSION=${BUILD_DATE}
fi;
echo "今天的 Build Version 是 ${BUILD_VERSION}"

if [ -d "${WORKSPACE}/build/${BUILD_VERSION}" ]; then 
    rm -rf ${WORKSPACE}/build/${BUILD_VERSION}; 
fi;
mkdir ${WORKSPACE}/build/${BUILD_VERSION};

if [ -d "${WORKSPACE}/Enterprise.plist" ]; then
    rm ${WORKSPACE}/Enterprise.plist; 
fi;

#http://www.matrixprojects.net/p/xcodebuild-export-options-plist/
Enterprise='<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
        <key>teamID</key>
        <string></string>
        <key>method</key>
        <string>app-store</string>
        <key>uploadSymbols</key>
        <true/>
        <key>uploadBitcode</key>
        <false/>
</dict>
</plist>'
echo ${Enterprise} > ${WORKSPACE}/Enterprise.plist

sed -i '' 's/ProvisioningStyle = Automatic;/ProvisioningStyle = Manual;/g' \
${WORKSPACE}/${PROJECT_NAME}.xcodeproj/project.pbxproj

sed -i '' 's/DEVELOPMENT_TEAM = .*;/DEVELOPMENT_TEAM = "";/g' \
${WORKSPACE}/${PROJECT_NAME}.xcodeproj/project.pbxproj

#动态生成 Build Version
sed -i '' "/<key>CFBundleVersion<\/key>/{N;s/<string>.*<\/string>/<string>${BUILD_VERSION}<\/string>/g;}" \
${WORKSPACE}/${PROJECT_NAME}/${PROJECT_NAME}-Info.plist

xcodebuild -workspace ${WORKSPACE}/${PROJECT_NAME}.xcworkspace \
-scheme ${PROJECT_NAME} -sdk iphoneos \
build CODE_SIGN_IDENTITY="iPhone Distribution: Beijing PS Technology Co., Ltd." \
PROVISIONING_PROFILE="" \
-configuration Release clean archive \
-archivePath ${WORKSPACE}/build/${BUILD_VERSION}/${PROJECT_NAME}.xcarchive

xcodebuild -exportArchive -exportOptionsPlist ${WORKSPACE}/Enterprise.plist \
-archivePath ${WORKSPACE}/build/${BUILD_VERSION}/${PROJECT_NAME}.xcarchive \
-exportPath ${WORKSPACE}/build/${BUILD_VERSION}/

echo "----------------"
echo "Build successfully!"


echo "Begin Upload to itunes..."
#Use [shenzhen](https://github.com/nomad/shenzhen) to upload the ipa file to itunes connect.
/usr/local/bin/ipa distribute:itunesconnect -f ${WORKSPACE}/build/${BUILD_VERSION}/${PROJECT_NAME}.ipa -a YourAppleID -p YourPassword -i ${BUNDLE_ID} --upload
echo "Upload successfully!"
```


## Tip8: 重置 iOS 模拟器

相信各位在做 iOS 开发的同学都会碰到模拟器上各种神奇的现象，通过重置 iOS 模拟器基本上可以解决大部分问题：

```Sh
// 退出当前的所有模拟器
$ osascript -e 'tell application "iOS Simulator" to quit'
$ osascript -e 'tell application "Simulator" to quit'

// 清掉之前使用模拟器产生的所有内容
$ xcrun simctl erase all
```

