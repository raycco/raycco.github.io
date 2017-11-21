---
title: GitHub搭建个人的博客
tags: [hexo, github]
categories: 搭建博客
---
最近突然间发现，自己过去看过的东西，没过多久就忘得一干二净，每当要用时，又得从头来一遍，真是好记性不如烂笔头，所以有了写笔记的想法，GitHub上可以方便记录自己一切想记录的，于是就在GitHub上开始写笔记，希望自己不要把知识忘得太快。本文记录在Windows环境下使用Hexo搭建GitHub博客的过程。
## 安装Node.js
在nodejs官网下载对应的版本安装
下载地址：[https://nodejs.org/en/download](https://nodejs.org/en/download)

## 安装Git
下载地址：[https://git-for-windows.github.io](https://git-for-windows.github.io)

## 创建GitHub账户
进入GitHub主页[https://github.com/](https://github.com/)，依次输入用户名、邮箱、密码，然后点击注册，按默认点击“Finish sign up”。然后进行邮箱验证。

## 创建GitHub仓库
点击“New repository”，新建一个仓库，仓库名为“[yourname].github.io”，这样https://[yourname].github.io 就是你的博客地址了。默认这仓库只有master分支，新建一个hexo分支。

## 配置Hexo
接下来的命令都在Git Bash中执行。

在自己喜欢的位置新建一个blog文件夹，在这个文件夹下打开Git Bash，因为npm是国外服务器，可能执行比较慢，可以使用淘宝镜像，命令如下：

    $ npm install -g cnpm --registry=https://registry.npm.taobao.org
执行成功后使用淘宝NPM安装Hexo

    $ cnpm install -g hexo-cli
    $ cnmp install hexo --save
    $ hexo -v
    hexo: 3.4.0
    hexo-cli: 1.0.4
    os: Windows_NT 6.1.7600 win32 ia32
    http_parser: 2.7.0
    node: 8.7.0
    v8: 6.1.534.42
    uv: 1.15.0
    zlib: 1.2.11
    ares: 1.10.1-DEV
    modules: 57
    nghttp2: 1.25.0
    openssl: 1.0.2l
    icu: 59.1
    unicode: 9.0
    cldr: 31.0.1
    tz: 2017b
到这里hexo已经安装好了

## 配置ssh key

    ssh-keygen -t rsa -C "Github的注册邮箱地址"
在C:\Users\yourname\\.ssh下会得到密钥id\_rsa和id\_rsa.pub，用nodepad++打开id_rsa.pub复制全文，打开[https://github.com/settings/ssh](https://github.com/settings/ssh)，Add SSH key，粘贴进去保存。 

测试是否配置成功

    $ ssh -T git@github.com
    Hi [yourname]! You've successfully authenticated, but GitHub does not 
    provide shell access.


## 配置git信息

    $ git config --global user.name "你的用户名"
    $ git config --global user.email "你的邮箱"

## 使用Hexo管理博客
### 初始化博客
    $ hexo init <nodejs-hexo> //初始化nodejs-hexo（文件夹名随意）
    $ git clone -b hexo https://github.com/[yourname]/[yourname].github.io
将[yourname].github.io文件夹下的\.git文件夹拷贝到nodejs-hexo文件夹。

    $ cnpm install //安装生成器
    $ hexo server //运行（Ctrl + C停止运行）
在浏览器输入localhost:4000，这样就可以在本地看到博客了。
### 配置博客

_config.yml中配置基本信息

    title: #博客标题
    subtitle: #博客副标题
    description: #博客描述
    author: #博客作者
    language: zh-Hans
    timezone: Asia/Shanghai
_config.yml中配置主题

    theme: next

_config.yml中配置部署

    deploy:
      type: git
      repo: https://github.com/[yourname]/[yourname].github.io
      branch: master
注意：这里的设置冒号后面必须有空格

### 发布博客

    $ hexo new "博客名" //增加新文章
    $ cnpm install hexo-deployer-git --save //安装hexo git插件
    $ git add .
    $ git commit -m "message"
    $ git push origin hexo
    $ hexo generate //生成静态文件
    $ hexo deploy   //部署

### 换PC管理博客
    $ git clone -b hexo https://github.com/[yourname]/[yourname].github.io
在[yourname].github.io中从新安装hexo，就可以写博客及发布博客了。

### 参考
Next主题配置参考：[http://theme-next.iissnan.com/theme-settings.html](http://theme-next.iissnan.com/theme-settings.html)







