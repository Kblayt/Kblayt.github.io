---
title: Git提交项目笔记
date: 2022-10-22 15:21:46
categories:
    - 项目管理工具
tags:
    - git
    - gitee
    - github
---
1、打开想要提交或者克隆的项目文件夹

2、进入该文件夹打开Git Bash Here

3、初始化用户，告诉git你是谁

- git config --global user.name “你的名字或昵称”

- git config --global user.email “你的邮箱”

4、在该文件夹中执行下面命令，完成初始化

- git init

- git remote add origin <项目地址> //注:项目地址形式为:https://gitee.com/xxx/xxx.git或者 git@gitee.com:xxx/xxx.git

- 如果提交到github上则为： https://<Personal access tokens(自己github上的tokens)>@github.com/Kblayt/Kblayt.github.io.git

5、若想克隆，只需要执行命令

- git clone <项目地址> //注:项目地址形式为:https://gitee.com/xxx/xxx.git或者 git@gitee.com:xxx/xxx.git

6、进入已经初始化好的或者克隆项目的目录然后执行：

从服务器下更新项目，因为已经clone过，所以不需要再更新

- git pull origin master

7、做一些修改后执行如下命令完成一次完整的提交

- git add .

- git commit -m “安装教程测试”

- git push origin master

注意：提交命令有两个，git push origin master（正常提交）和git push origin master -f（强制提交，强制提交可能会把之前的commit注释信息，不会改变修改的代码，慎用），都是提交到master分支 其他常见git命令

- 查看所有分支 ：git branch -a

- 切换到某一分支：git checkout 分支名称

- 合并分支：git merge 原分支 目标分支