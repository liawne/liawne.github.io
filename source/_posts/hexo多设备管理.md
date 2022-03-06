---
title: hexo多设备管理
date: 2022-03-01 22:13:11
tags:
- record
categories: 随笔
description: "在新设备上准备hexo发布文章的配置过程"
---
`github+hexo+next搭建个人博客`已经完成,使用一段时间后,发现存在多处编辑提交的需求.
在网上查询相关方案后,决定采取广大网友建议的方式: 将平时需要修改的内容存储在hexo分支,hexo基于更改内容生成的文件提交到master分支
下列内容记录在raspberry上配置实现hexo编辑提交内容的过程
<!--more-->

### 1.安装基础包

在新机器上安装基础包，当前机器为raspberry，以该机器为例说明
raspberry安装系统是"Raspbian GNU/Linux 10 (buster)"，基于debian的发行，用如下命令安装
```bash
# 安装nodejs、npm、git
sudo apt install nodejs npm git
```

### 2.配置git及github

安装好基础包后，要配置github上**个人资料/settings/SSH and GPG keys/SSH keys**,用于提交代码使用
```bash
# 在raspberry上,使用pi用户生成公私钥对
PI $ ssh-keygen -t rsa -P ''
Generating public/private rsa key pair.
Enter file in which to save the key (/home/pi/.ssh/id_rsa): Your identification has been saved in /home/pi/.ssh/id_rsa.
Your public key has been saved in /home/pi/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:5C2iM9maPY/n+2iDwmodQJOjxTq2ZJkwv0xBy4EG68E pi@raspberrypi
The keys randomart image is:
+---[RSA 2048]----+
|o+o.             |
|=+Oo             |
|+E=+    .        |
|=*=    o .       |
|+=.o  . S .      |
| .o .+ . .       |
|   o=...         |
|  . +*o.+.       |
| ...o.o*=+.      |
+----[SHA256]-----+

# pi用户家目录下.ssh文件夹下新生成了id_rsa/id_rsa.pub文件
PI $ ls ~/.ssh -lrt
total 16
-rw-r--r-- 1 pi pi  444 Feb 27 21:28 known_hosts
-rw------- 1 pi pi 1366 Feb 28 13:06 authorized_keys
-rw-r--r-- 1 pi pi  396 Mar  1 21:30 id_rsa.pub
-rw------- 1 pi pi 1823 Mar  1 21:30 id_rsa

# 复制id_rsa.pub内容,在github上SSH keys界面上点击New SSH key,添加复制的公钥内容并命名
PI $ cat ~/.ssh/id_rsa.pub 
ssh-rsa AAAAB3NzaCxxxx.....
```

配置本地git
```bash
# 配置user.name和user.email
PI $ git config --global user.email "you@example.com"
PI $ git config --global user.name "Your Name"
```


### 3.npm安装hexo相关包

配置好github **SSH keys**后,下载**yourname.github.io**库到本地
```bash
# 进入相应的目录,下载hexo分支代码; 此处在家目录下新建了projects目录,进入projects目录进行操作
PI $ mkdir ~/projects && cd ~/projects

# $yourname填写自己实际的名称
PI $ git clone git@github.com:$yourname/$yourname.github.io.git
```

下载完成后,进入**yourname.github.io**目录,安装npm相关包
```bash
# 安装hexo
PI $ cd $yourname.github.io && npm install hexo

# 当前目录下已经包含package.json文件,直接安装相关包
PI $ npm install

# hexo同步至github需要使用的包
PI $ npm install hexo-deployer-git

# 安装完成后,需要将当前目录下node_modules/.bin加入到PATH中
PI $ echo 'export PATH=$PATH:/home/pi/projects/$yourname.github.io/node_modules/.bin' >> ~/.bashrc 
PI $ source ~/.bashrc
```

### 4.测试发布新文章

上述内容配置完成后,在新机器上测试新文章发布
```bash
# 新增文章
PI $ hexo n test

# 编辑source/_post/test.md内容,可以在typora等markdown写作软件上先完成内容后再粘贴
PI $ vim source/_post/test.md

# 生成内容
PI $ hexo g

# 测试新文章内容,hexo s后访问localhost:4000查看文章效果
PI $ hexo s

# 内容核对完成,发布到github yourname.github.io的master分支上
PI $ hexo d
```
