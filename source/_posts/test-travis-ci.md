---
title: Travis-ci 构建自动化博客
date: 2018-03-28 23:08:00
tags:
	- Travis
	- 前端
---
## 序言
昨天与同事讨论Hexo博客，了解到了一个工具叫 Travis CI。即兴之下，查阅使用方法，尝试使用并写下此文。

以往本人的部署方式是：
hexo使用的部署命令是把生成好的静态文件上传到仓库中的，所以，在其他电脑上同步下来的只是生成后的静态文件而已，不是源码。

到这里就会想到能不能这样：将源码同步到远程仓库后，可以实现自动生成部署呢？
答案是肯定的，可以通过Travis CI来实现。

本文介绍 Travis CI 的 Linux/Mac 系统下的基本用法。
用好它，能提高效率，还能使开发流程更稳健、专业。而且，它对于开源项目（如github个人项目）是免费的。

>说明：
文中出现的命令，Windows用户注意，命令前面有 $ 的表示在Git Bash中执行，没有的在CMD命令窗口执行。
Linux和Mac系统在终端下不区分。
文末将会提及关于用户使用Windows系统通过 Travis CI 解密证书时报错问题。


## 使用准备
Travis CI 只支持 Github，不支持其他代码托管服务。这意味着，你必须满足以下条件，才能使用 Travis CI。

>- 拥有 GitHub 帐号
>- 该帐号下面有一个项目
>- 该项目里面有可运行的代码
> 这里主要讲述 Travis CI，搭建博客请参考[hexo+github 搭建个人博客](https://www.jianshu.com/p/f4cc5866946b)

上述条件都准备好，就可以开始使用 Travis CI 了。

（1）访问官方网站 [travis-ci.org](https://travis-ci.org/)，点击右上角的个人头像，使用 Github 账户登陆 Travis CI。
![](./uploads/travis_ci_01.png '登陆')

（2）Travis 会列出 Github 上面你的所有仓库，以及你所属于的组织。此时，选择你需要 Travis 帮你构建的仓库，打开仓库旁边的开关。一旦激活了一个仓库，Travis 会监听这个仓库的所有变化。

在右上角你的账户名点击进入 Profile，在Repositories tab页点击Sync now同步你的github项目。
![](./uploads/travis_ci_02.png '激活并监听仓库')


## 配置Travis
（1）新建配置文件
```bash
$ cd 博客项目文件夹根目录
$ touch .travis.yml
```

（2）在项目的根目录建立 **.travis** 文件夹
```bash
$ mkdir .travis
```

（3）复制ssh文件 **id_rsa** 到 **.travis** 目录下
```bash
$ cd 博客项目文件夹根目录/.travis
```

（4）在 **.travis** 目录下创建 **ssh_config** 文件
```bash
$ cp ~/.ssh/id_rsa_blog .travis/
```

编辑 ssh_config，输入以下信息
```
Host github.com
User git
StrictHostKeyChecking no
IdentityFile ~/.ssh/id_rsa
IdentitiesOnly yes
```

（5）travis 登录
根据提示输入github的用户名和密码
```bash
$ cd 博客项目文件夹根目录/.travis
$ travis login --auto
We need your GitHub login to identify you.
This information will not be sent to Travis CI, only to api.github.com.
The password will not be displayed.

Try running with --github-token or --auto if you don't want to enter your password anyway.

Username: a298003154
Password for a298003154: ************
Successfully logged in as a298003154!
```

（6）对ssh的私钥进行加密
```
travis encrypt-file .travis/id_rsa_blog .travis/ --add
```
此操作会生成加密之后的秘钥文件 **id_rsa.enc** ，删除 **id_rsa** 密钥文件(私钥不能随便泄露)。
同时在终端上会输出类似如下信息：
```
openssl aes-256-cbc -K $encrypted_xxxxxxxxxxx_key -iv $encrypted_xxxxxxxxxxx_iv
```
这是用于 **id_rsa.enc** 解密的信息，保存上面 xxxxxxxxxxx 的信息，后面会在 **.travis.yml** 配置文件会用到。

## 编辑配置文件
（1）Travis配置文件
打开Travis配置文件 **.travis.yml**，添加如下信息：
```
	language: node_js
	node_js:
	- "4"  # nodejs的版本
	branches:
	only:
	- dev  # 设置自动化部署的分支
	before_install:
	- export TZ='Asia/Shanghai'  # 设置时区
	- npm install -g hexo
	- npm install -g hexo-cli
	# 将xxxxxxxxxxx替换上面生成的内容
	# 这里面的文件路径可根据自己的情况进行修改
	# 解密id_rsa_blog.enc 输出到.ssh/文件夹下，命名为id_rsa
	- openssl aes-256-cbc -K $encrypted_xxxxxxxxxxx_key -iv $encrypted_xxxxxxxxxxx_iv -in .travis/id_rsa.enc -out ~/.ssh/id_rsa -d
	# 设置id_rsa文件权限
	- chmod 600 ~/.ssh/id_rsa
	# 添加ssh密钥
	- eval $(ssh-agent)
	- ssh-add ~/.ssh/id_rsa
	# 添加ssh配置文件
	- cp .travis/ssh_config ~/.ssh/config
	# 设置github账户信息
	- git config --global user.name "github" #设置github用户名
	- git config --global user.email "github@qq.com" #设置github用户邮箱
	# 安装依赖组件
	install:
	- npm install
	# 执行的命令
	script:
	- hexo clean && hexo g -d
```
（2）Hexo配置文件
打开Hexo配置文件 **_config.yml**，编辑如下信息：
```
deploy:
  type: git
  repo: git@github.com:a298003154/a298003154.github.io.git
  branch: master
```

## 更新/发布博客项目
（1）初始化本地仓库
切换到项目根目录下，删除原来部署时产生的.git文件夹。
```bash
$ git init
```
（2）提交推送4步走
提交本地修改，关联远程仓库，推送至github仓库。
```bash
$ # 添加文件
$ git add .
$ # 提交修改
$ git commit -m "test travis"
$ # 将github仓库改为自己的
$ git remote add origin git@github.com:a298003154/a298003154.github.io.git
$ # 推送至远程仓库
$ git push -u origin dev
```
push本地的代码至远程仓库之后，在 [travis-ci.org](https://travis-ci.org/) 后台查看相关情况。
下面是成功的结果：
![](./uploads/travis_ci_03.png '发布成功')

## 报错问题
1.Windows用户可能会出现如下错误:
```
The command "openssl aes-256-cbc -K $encrypted_xxxxxxxxxxx_key -iv $encrypted_xxxxxxxxxxx_iv -in .travis/id_rsa.enc -out ~/.ssh/id_rsa -d" failed and exited with 1 during.
```
原因是travis在执行加密操作生成的加密信息位数不对。之后在ubuntu/linux系统中操作就一切正常。

如果不想安装虚拟机，还有另外一种方式实现。

2.所有的配置文件是yaml格式，空格一定要注意。

<!--
http://www.ruanyifeng.com/blog/2017/12/travis_ci_tutorial.html
持续集成服务 Travis CI 教程

https://travis-ci.com/
travis官网

https://www.jianshu.com/p/7f05b452fd3a
手把手教从零开始在GitHub上使用Hexo搭建博客教程(三)-使用Travis自动部署Hexo(1)

https://www.jianshu.com/p/fff7b3384f46
手把手教从零开始在GitHub上使用Hexo搭建博客教程(四)-使用Travis自动部署Hexo(2)

https://blog.csdn.net/u012373815/article/details/53574002
hexo＋Travis-ci＋github构建自动化博客

https://www.jianshu.com/p/3dafd38f3733
Travis-CI解密证书时报错问题

https://segmentfault.com/a/1190000005804780
用travis和git hook搞个一键部署

https://www.v2ex.com/t/170462
使用 Travis CI 自动部署 Hexo -->
